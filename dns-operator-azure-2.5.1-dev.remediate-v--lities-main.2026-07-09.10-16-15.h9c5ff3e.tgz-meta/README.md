[![CircleCI](https://circleci.com/gh/giantswarm/dns-operator-azure.svg?style=shield)](https://circleci.com/gh/giantswarm/dns-operator-azure) [![Docker Repository on Quay](https://quay.io/repository/giantswarm/dns-operator-azure/status "Docker Repository on Quay")](https://quay.io/repository/giantswarm/dns-operator-azure)

# dns-operator-azure

`dns-operator-azure` is an operator to create necessary DNS records on Azure DNS for CAPI workload clusters. It 
supports both CAPZ and non-CAPZ workload clusters.

## Use cases

### CAPZ workload clusters

`cluster-api-provider-azure` takes care of the most of the DNS management for CAPZ WCs but there are some shortcomings:

- It only uses `Private DNS Zone` for private clusters for the cluster itself, but we want to expose all DNS records 
  to the public so that users/clients can access clusters via networking solutions such as VPNs. We plan to add the feature to upstream, but it is not there yet. See https://github.com/giantswarm/roadmap/issues/3374
- It doesn't provide DNS features for MC-to-WC api connection when WC is private. We are creating private links and private endpoints 
  so that MC can access the WC api via FQDN but via a specific IP in MC Vnet. See the whole flow in https://github.com/giantswarm/azure-private-endpoint-operator/blob/main/README.md
- When the management cluster is fully private, WCs cannot access the MC endpoints by default. We use private links and 
  private endpoints here again so that monitoring clients in WCs can access the ingress endpoints in MC. See the whole flow in https://github.com/giantswarm/azure-private-endpoint-operator/blob/main/README.md

### Non-CAPZ workload clusters

We manage non-CAPZ workload clusters in CAPZ MCs too. We call this concept as `multi-provider` setup. In this case, 
`dns-operator-azure` helps us to manage DNS records of workload clusters on Azure DNS.

Supported non-CAPZ providers:
- CAPV WCs (Vsphere)

#### Tagging Resource Groups for Non-CAPZ workload clusters

For CAPZ workload clusters, the resource group of the DNS zone is managed by CAPZ.
CAPZ allows for tagging the resource group of the DNS zone with additional tags set in the `AzureCluster` resource.
For non-CAPZ workload clusters, the resource group of the DNS zone is managed by `dns-operator-azure`.
The `dns-operator-azure` supports tagging the resource group of the DNS zone via prefixed annotations on the non-CAPZ workload cluster's Infrastructure Cluster resource.

Annotations prefixed by `azure-resourcegroup-tag.` will be used to tag the resource group of the DNS zone.
Note that the prefix `azure-resourcegroup-tag.` will be stripped from the annotation key when tagging the resource group.

## Expected Behavior

### Public DNS Zone <wc_name>.<base_domain> in <wc_name> resource group

***When to create***

- When the WC is public CAPZ, created by CAPZ.
- When the WC is private CAPZ, created by dns-operator-azure.
- When the WC is non-CAPZ, created by dns-operator-azure.
- Used by everyone to access the WC.
- MC can be public or private. It doesn't matter.

***A records***
- `api` and `apiserver` should refer 
  - Public CAPZ: IP of `AzureCluster.Spec.NetworkSpec.APIServerLB.FrontendIPs[0].PublicIP.Name` LB in Azure
  - Private CAPZ: `AzureCluster.Spec.NetworkSpec.APIServerLB.FrontendIPs[0].PrivateIPAddress`
  - Non-CAPZ WCs: `Cluster.Spec.ControlPlaneEndpoint.Host`

- `ingress` should refer the IP of k8s service in the WC if exists. 
  - CAPZ Clusters: Managed by `external-dns` (deployed by default-apps-azure)
  - Non-CAPZ Clusters: Managed by `dns-operator-azure`
  
***CNAME records***
- `*` should refer `ingress.<wc_name>.<base_domain>`

***NS records in basedomain***

dns-operator-azure is supposed to create an `NS` record in the `<baseDomain>` zone to refer the `<wc_name>.<base_domain>` zone.

### Public DNS Zone <mc_name>.<base_domain> in <mc_name> resource group
Since MC is also a workload cluster in itself, this entry is supposed to exist as above.
 
### Private DNS Zone <wc_name>.<base_domain> in <wc_name> resource group 
- Created only for private CAPZ workload clusters by CAPZ.
- Used by the WC itself.

***A records***
- `apiserver` should refer `AzureCluster.Spec.NetworkSpec.APIServerLB.FrontendIPs[0].PrivateIPAddress`
- `ingress` should refer the IP of k8s service in the WC if exists. Managed by `external-dns` (deployed by default-apps-azure)
 
### Private DNS Zone <wc_name>.<base_domain> in <mc_name> resource group 

- Used by the MC to access the WC api.
- Created for private CAPZ workload clusters by dns-operator-azure.
- MC can be public or private. It doesn't matter.
- Not created for non-CAPZ workload clusters.


***A records***
- `apiserver` should refer the private endpoint in the MC VNet that points the private link of WC api.

###  Private DNS Zone <mc_name>.<base_domain> in <mc_name> resource group

Since MC is also a workload cluster in itself, this entry is supposed to exist as above if MC is a private cluster..

###  Private DNS Zone <mc_name>.<base_domain> in <wc_name> resource group  

- Used by the WC to access the MC ingress endpoints.
- Created for private CAPZ management clusters by dns-operator-azure.
- WC can be public or private. It doesn't matter.
- Not created for non-CAPZ workload clusters.
- The ingress record refers the private endpoint in the WC VNet that points the private link of MC ingress endpoints.

***A records***
- `ingress` should refer the private endpoint in the WC VNet that points the private link of MC ingress.

***CNAME records***
- `*` should refer `ingress.<mc_name>.<base_domain>`. 

## Bastion concept

We switched to `teleport` to access cluster nodes, so we stopped deploying bastion nodes. This feature is no 
supported anymore. Last supported version was `1.3.4`. 

## Cluster Deletion

On `Cluster` deletion, `CAPZ` deletes the entire `resourceGroup` where the `<clustername>` specific DNS zone exists 
as well. For that reason on deletion only the `NS` record in the `<baseDomain>` must be handled by the operator.


## Configuration of the operator

`dns-operator-azure` expect an existing DNS Zone which is used as `baseDomain` (e.g. `kubernetes.my-company.io`).
There is no need to have this DNS Zone in the same subscription where `dns-operator-azure` is running.

To act on this DNS Zone, the name and the resource group must be defined by `-base-domain` and `-base-domain-resource-group` flag.
The subscription where this DNS Zone exist must be defined by setting the `AZURE_SUBSCRIPTION_ID` environment variable.

## Azure AuthN/AuthZ

To make `dns-operator-azure` work on the `baseDomain` DNS Zone you have to create an application in `Azure ActiveDirectory`. This application need the `DNS Zone Contributor` role applied for to the `baseDomain` DNS Zone.

The application secrets must be defined by setting the `AZURE_CLIENT_ID`, `AZURE_TENANT_ID` and `AZURE_CLIENT_SECRET` environment variables.

To make `dns-operator-azure` work on the `clustername` DNS zone it's only required that the Kubernetes `serviceAccount` is able to get the referenced `AzureClusterIdentity` from the `AzureCluster`. With these information the `dns-operator-azure` creates an internal Azure client to interact with the cluster specific Azure resources.

To enable `dns-operator-azure` to create Azure DNS zones and records for non-Azure workload clusters, the operator needs to know credentials and details of the destination Azure subscription, where the DNS zone and records for the non-Azure workload clusters are supposed to be stored.
By default, the `dns-operator-azure` uses credentials referenced in the management cluster's `AzureCluster` resource.
In case this behaviour is acceptable, there is no need to provide any additional configuration to the operator.
The default behaviour can be overridden by providing an explicit reference to a specific `AzureClusterIdentity` resource in the `--azure-identity-ref-name` and `--azure-identity-ref-namespace` flags.
Additional details about the subscription need to be provided by setting the `CLUSTER_AZURE_CLIENT_ID`, `CLUSTER_AZURE_TENANT_ID`, `CLUSTER_AZURE_SUBSCRIPTION_ID` and `CLUSTER_AZURE_LOCATION` (the Azure Location of the DNS records for the non-Azure workload clusters) flags.
