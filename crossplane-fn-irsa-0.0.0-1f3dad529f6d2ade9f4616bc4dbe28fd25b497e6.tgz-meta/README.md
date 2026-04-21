# crossplane-fn-irsa

A [Crossplane Composition Function](https://docs.crossplane.io/latest/concepts/composition-functions/) that automates the setup of **IAM Roles for Service Accounts (IRSA)** infrastructure in AWS. IRSA enables Kubernetes service accounts to assume AWS IAM roles, allowing pods to access AWS services without embedding static credentials.

## How the Composition Works

The IRSA composition uses a **3-step pipeline** that combines a Go-based discovery function, a KCL resource renderer, and an auto-ready detector:

```
IRSAClaim (user input)
        |
        v
┌───────────────────────────────────┐
│  Step 1: irsa-discovery           │  Go function (this repo)
│  - Discover Route53 hosted zone   │  Queries AWS APIs to find existing
│  - Discover CloudFront distro     │  resources and generates OIDC files.
│  - Generate OIDC discovery doc    │  Patches results into XR status.
│  - Generate JWKS keys file        │
└───────────────┬───────────────────┘
                |
                v
┌───────────────────────────────────┐
│  Step 2: render-resources         │  function-kcl
│  - Reads XR spec + status         │  Renders all AWS managed resources
│  - Produces Crossplane MRs        │  using the discovered values from
│  - Handles region differences     │  step 1 as input.
└───────────────┬───────────────────┘
                |
                v
┌───────────────────────────────────┐
│  Step 3: auto-ready               │  function-auto-ready
│  - Detects composed resource      │  Marks the XR as ready when all
│    readiness                      │  composed resources are ready.
└───────────────────────────────────┘
```

### Step 1: Discovery (this function)

The Go function runs first and performs AWS API calls to discover existing infrastructure and generate OIDC-related files. It patches results into the XR's `status` fields so that step 2 can use them.

**For standard AWS regions**, the function:
1. Queries **Route53** to find the hosted zone matching the cluster domain
2. Queries **CloudFront** to find an existing distribution matching `irsa.<domain>`
3. Generates the **OIDC discovery document** (`.well-known/openid-configuration`) pointing to `https://irsa.<domain>`
4. Reads the cluster's service account signing key and generates a **JWKS file** (`keys.json`)

**For China regions** (`cn-north-1`, `cn-northwest-1`), the function:
1. Queries **IAM** for existing OpenID Connect providers
2. Generates the **OIDC discovery document** pointing to `https://s3.<region>.amazonaws.com.cn/<bucket>`
3. Generates the **JWKS file** from the service account signing key

### Step 2: Resource Rendering (function-kcl)

The KCL script reads the XR spec and the status fields populated by step 1, then renders the appropriate set of AWS managed resources.

**Standard AWS regions** produce:

| Resource | Kind | Purpose |
|----------|------|---------|
| OIDC Provider | `iam.aws.upbound.io/OpenIDConnectProvider` | Enables EKS-style pod identity |
| S3 Bucket | `s3.aws.upbound.io/Bucket` | Stores OIDC discovery and JWKS files |
| S3 Discovery Object | `s3.aws.upbound.io/BucketObject` | `.well-known/openid-configuration` |
| S3 Keys Object | `s3.aws.upbound.io/BucketObject` | `keys.json` (JWKS) |
| S3 Bucket Policy | `s3.aws.upbound.io/BucketPolicy` | Grants CloudFront OAI read access |
| CloudFront OAI | `cloudfront.aws.upbound.io/OriginAccessIdentity` | Origin access for S3 |
| ACM Certificate | `acm.aws.upbound.io/Certificate` | TLS cert for `irsa.<domain>` |
| Route53 Validation Record | `route53.aws.upbound.io/Record` | DNS validation for ACM cert |
| CloudFront Distribution | `cloudfront.aws.upbound.io/Distribution` | Serves OIDC files via HTTPS |
| Route53 CNAME | `route53.aws.upbound.io/Record` | Points `irsa.<domain>` to CloudFront |

**China regions** produce a smaller set (no CloudFront/ACM/Route53):

| Resource | Kind | Purpose |
|----------|------|---------|
| OIDC Provider | `iam.aws.upbound.io/OpenIDConnectProvider` | Enables pod identity |
| S3 Bucket | `s3.aws.upbound.io/Bucket` | Stores OIDC files |
| S3 Discovery Object | `s3.aws.upbound.io/BucketObject` | `.well-known/openid-configuration` |
| S3 Keys Object | `s3.aws.upbound.io/BucketObject` | `keys.json` (JWKS) |
| S3 Bucket Policy | `s3.aws.upbound.io/BucketPolicy` | Public read access |
| S3 Public Access Block | `s3.aws.upbound.io/BucketPublicAccessBlock` | Allows public access |
| S3 Ownership Controls | `s3.aws.upbound.io/BucketOwnershipControls` | Sets object ownership |

## Composite Resource Definition

The function defines an `IRSA` composite resource (with claim kind `IRSAClaim`):

```yaml
apiVersion: crossplane.giantswarm.io/v1
kind: IRSA
spec:
  name: string              # Base name for resources (required)
  bucketName: string         # S3 bucket name for OIDC files (required)
  domain: string             # Cluster domain (required for non-China regions)
  providerConfigRef: string  # AWS ProviderConfig name (required)
  region: string             # AWS region (required, default: us-east-1)
  tags: object               # Tags applied to all resources (optional)
```

## Function Input

The composition configures this function with an `Input` resource that maps XR fields to the function's internal references:

```yaml
input:
  apiVersion: irsa.fn.giantswarm.io
  kind: Input
  spec:
    domainRef: spec.domain                                      # Where to read the domain
    regionRef: spec.region                                      # Where to read the region
    providerConfigRef: spec.providerConfigRef                   # Where to read the provider config
    s3BucketNameRef: spec.bucketName                            # Where to read the bucket name
    route53HostedZonePatchToRef: status.importResources.route53ZoneId  # Where to patch the zone ID
    s3KeysPatchToRef: status.s3Keys                             # Where to patch the JWKS file
    s3DiscoveryPatchToRef: status.s3Discovery                   # Where to patch the discovery doc
```

## Examples

### Standard AWS region

```yaml
apiVersion: crossplane.giantswarm.io/v1
kind: IRSAClaim
metadata:
  name: mycluster
  namespace: org-giantswarm
spec:
  compositionRef:
    name: irsa-composition
  name: mycluster
  bucketName: 242036376510-g8s-mycluster-oidc-pod-identity-v3
  domain: mycluster.gaws.gigantic.io
  providerConfigRef: mycluster
  region: eu-west-2
```

This creates the full IRSA stack: S3 bucket with OIDC files, CloudFront distribution serving `irsa.mycluster.gaws.gigantic.io`, ACM certificate, Route53 records, and an IAM OIDC provider.

### China region

```yaml
apiVersion: crossplane.giantswarm.io/v1
kind: IRSAClaim
metadata:
  name: mycluster
  namespace: org-giantswarm
spec:
  compositionRef:
    name: irsa-composition
  name: mycluster
  bucketName: 306934455918-g8s-mycluster-oidc-pod-identity-v3
  providerConfigRef: mycluster
  region: cn-north-1
```

Note that `domain` is omitted for China regions. The OIDC issuer URL uses the S3 endpoint directly (`https://s3.cn-north-1.amazonaws.com.cn/<bucket>`), so no CloudFront, ACM, or Route53 resources are created.

## Architecture Diagram

```
                     Standard AWS Regions
                     ════════════════════

  Pod (with SA) ──► EKS/IRSA ──► IAM OIDC Provider
                                        │
                                        │ validates tokens via
                                        ▼
                               ┌─── irsa.<domain> ───┐
                               │    (Route53 CNAME)   │
                               └─────────┬────────────┘
                                         │
                                         ▼
                               ┌─── CloudFront ──────┐
                               │   (ACM TLS cert)    │
                               └─────────┬───────────┘
                                         │
                                         ▼
                               ┌─── S3 Bucket ───────┐
                               │  /.well-known/       │
                               │    openid-config     │
                               │  /keys.json (JWKS)   │
                               └──────────────────────┘


                     China Regions
                     ═════════════

  Pod (with SA) ──► EKS/IRSA ──► IAM OIDC Provider
                                        │
                                        │ validates tokens via
                                        ▼
                               ┌─── S3 Bucket ───────┐
                               │  (public access)     │
                               │  /.well-known/       │
                               │    openid-config     │
                               │  /keys.json (JWKS)   │
                               └──────────────────────┘
```

## Development

### Building

```bash
make build
```

When editing the `input` types, run code generation:

```bash
go generate ./...
```
