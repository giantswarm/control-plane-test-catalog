### Crossplane MC OIDC

A Crossplane Composition Function that automates the setup of OpenID Connect (OIDC) trust relationships between a Management Cluster (MC) and multiple Workload Clusters (WCs) across different AWS accounts.

## Architecture

This function implements a multi-stage pipeline architecture that integrates with existing Crossplane resources to establish cross-account OIDC trust relationships:

### Function Workflow

The function operates through the following sequence:

```
1. Discovery Phase (crossplane-fn-mcoidc)
   - List all ProviderConfig resources (aws.upbound.io/v1beta1)
   - List all AWSCluster and AWSManagedCluster resources (infrastructure.cluster.x-k8s.io/v1beta2)
   - Filter ProviderConfigs with matching AWSCluster or AWSManagedCluster resources
   - Extract and deduplicate AWS account IDs from roleARNs
   - Retrieve OIDC provider details from existing MC resources
    ↓
2. Resource Rendering (function-kcl)
   - Generate ProviderConfig resources for each discovered account
   - Generate OpenIDConnectProvider resources with cross-account ARNs
    ↓
3. Readiness Detection (function-auto-ready)
   - Monitor resource creation and readiness status
   - Update composite resource status when all resources are ready
```

## General Logic

### Account Discovery Logic

The function implements intelligent account discovery to identify target AWS accounts for OIDC setup:

1. **Resource Enumeration**: Lists all ProviderConfig resources in the cluster and matches them with corresponding AWSCluster or AWSManagedCluster resources
2. **Account Extraction**: Parses roleARNs from ProviderConfig specs to extract AWS account IDs using the format `arn:aws:iam::ACCOUNT_ID:role/ROLE_NAME`
3. **Filtering**: Excludes the Management Cluster's own account ID to prevent self-referential configurations
4. **Deduplication**: Maintains a unique set of account IDs to avoid creating multiple OIDC providers in the same account

### OIDC Provider Replication Logic

The function replicates OIDC provider configurations across multiple AWS accounts:

1. **Source Discovery**: Locates the existing OpenIDConnectProvider resource in the Management Cluster by matching the `crossplane.io/claim-name` label
2. **Configuration Extraction**: Retrieves essential OIDC provider properties:
   - `clientIdList`: List of allowed client IDs
   - `thumbprintList`: SSL certificate thumbprints for trust validation
   - `url`: OIDC provider endpoint URL
   - `arn`: Source ARN for cross-account reference transformation
3. **Cross-Account Transformation**: Generates target ARNs by replacing the source account ID with each target account ID
4. **Resource Generation**: Creates OpenIDConnectProvider resources in each target account with consistent configuration

### Resource Management Strategy

The function employs a declarative approach to resource management:

- **ProviderConfig Creation**: Generates dedicated ProviderConfig resources for each target AWS account using the naming pattern `${mc_name}-${account_id}`
- **External Name Annotation**: Uses Crossplane's external-name annotation to establish the correct ARN references for imported OIDC providers
- **Idempotent Operations**: Ensures that repeated executions produce consistent results without duplicating resources


### Setting up

You need to set the name in several places

- `go.mod` - change the name from `crossplane-fn-mcoidc` to something reflective of your function
- `input/v1beta1/input.go` - change `+groupName=template.fn.giantswarm.io` to something unique for your function
- `fn.go` - set `composedName` to the name of your function (I normally use basename $module)

### Building

When editing `input`, do not forget to run ``go generate ./...`` otherwise your input will be out of line.

### Some suggestions for your README

After you have created your new repository, you may want to add some of these badges to the top of your README.

- **CircleCI:** After enabling builds for this repo via [this link](https://circleci.com/setup-project/gh/giantswarm/crossplane-fn-mcoidc), you can find badge code on [this page](https://app.circleci.com/settings/project/github/giantswarm/crossplane-fn-mcoidc/status-badges).

- **Go reference:** use [this helper](https://pkg.go.dev/badge/) to create the markdown code.

- **Go report card:** enter the module name on the [front page](https://goreportcard.com/) and hit "Generate report". Then use this markdown code for your badge: `[![Go report card](https://goreportcard.com/badge/github.com/giantswarm/crossplane-fn-mcoidc)](https://goreportcard.com/report/github.com/giantswarm/crossplane-fn-mcoidc)`

- **Sourcegraph "used by N projects" badge**: for public Go repos only: `[![Sourcegraph](https://sourcegraph.com/github.com/giantswarm/crossplane-fn-mcoidc/-/badge.svg)](https://sourcegraph.com/github.com/giantswarm/crossplane-fn-mcoidc)`
