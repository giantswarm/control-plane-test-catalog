[![CircleCI](https://dl.circleci.com/status-badge/img/gh/giantswarm/cluster-api-provider-azure-app/tree/main.svg?style=svg)](https://dl.circleci.com/status-badge/redirect/gh/giantswarm/cluster-api-provider-azure-app/tree/main)

# cluster-api-provider-azure

Cluster API Azure controller packaged as a Giant Swarm app.

### Multi-tenancy, aka multi-account, aka Bring Your Own Credentials (BYOC)

In addition to using default credentials which use the management cluster's
Azure subscription, you can deploy your clusters to other Azure subscriptions.

> Note: Giant Swarm Azure management clusters are already ready for creating
> workload clusters in other subscriptions. This was done by creating necessary
> CRs so you can use them and create clusters for existing organizations, and
> the cluster's will be deployed to the organization's Azure subscription.

How to use other, non-default, credentials for your workload clusters? You need
AzureClusterIdentity and Secret where Azure credentials are stored.

AzureClusterIdentity object has the following:
- Azure tenant ID
- Service principal client ID
- Reference to Secret which holds the service principal client secret

```yaml
apiVersion: infrastructure.cluster.x-k8s.io/v1alpha3
kind: AzureClusterIdentity
metadata:
  name: my-sp-identity
  namespace: hello
spec:
  type: ServicePrincipal
  tenantID: $TENANT_ID
  clientID: $CLIENT_ID
  clientSecret:
    name: my-secret-name
    namespace: hello
---
apiVersion: v1
kind: Secret
metadata:
  name: my-secret-name
  namespace: hello
type: Opaque
data:
  clientSecret: $CLIENT_SECRET
```

Then in your AzureCluster object, set `Spec.IdentityRef` like this:

```yaml
apiVersion: infrastructure.cluster.x-k8s.io/v1alpha3
kind: AzureCluster
metadata:
  name: my-cluster
  namespace: hello
spec:
  identityRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1alpha3
    kind: AzureClusterIdentity
    name: my-sp-identity
    namespace: hello
  # ...
```

When you are creating a new AzureCluster, check the management cluster to see
if it already has AzureClusterIdentity objects created: 

```yaml
$ kubectl get AzureClusterIdentity -A
NAMESPACE    NAME                     AGE
giantswarm   org-credential-default   31d
giantswarm   org-credential-ym2rbz    31d
```

You can use these to create workload clusters.

For more details [here](https://github.com/kubernetes-sigs/cluster-api-provider-azure/blob/v0.4.13/docs/proposals/20200720-single-controller-multitenancy.md)
you can find CAPZ (v0.4.13) proposal for the multi-tenancy.
