# azure-private-endpoint-operator

Azure Private Endpoint Operator is a Kubernetes operator that manages private endpoints in AzureCluster CRs.

### MC to WC api

Private workload clusters use internal load balancer for api server, which means the MC cannot access the WC api server by default. 

If the workload cluster is private, 
- We add private links to `AzureCluster` of workload cluster in `cluster-azure` chart. See https://github.com/giantswarm/cluster-azure/blob/9d08a6fe0596525026746ce1ffcd6704d3fa4479/helm/cluster-azure/templates/_azure_cluster.tpl#L73
- CAPZ creates the private link `<wc-name>-api-privatelink`
- This operator watches `AzureCluster` of workload cluster for private links and it injects private endpoints to `AzureCluster` CR of management clusters.
- CAPZ creates the private endpoints `<wc-name>-api-privatelink-privateendpoint` in MC's VNET.
- This operator also adds the annotation `azure-private-endpoint-operator.giantswarm.io/private-link-apiserver-ip` to `AzureCluster` of workload clusters.
- The annotation for IP is handled by `dns-operator-azure`. It adds the record to the private DNS zone with WC name and links it to the management clusters' VNET. 

### WC to MC ingress

Private management clusters use internal load balancer for api server and ingresses, which means the WC cannot access them by default.
WCs don't need to access MC's api server, but they need to access the ingresses of MCs because of monitoring tools.

- We create a private link `<mc-name>-ingress-privatelink` for the MC once while creating the MC.
- This operator watches `AzureCluster` of workload clusters. It injects private endpoints to `AzureCluster` CR of workload clusters.
- CAPZ creates the private endpoints `<wc-name>-to-<mc-name>-privatelink-privateendpoint` in WC's VNET.
- This operator also adds the annotation `azure-private-endpoint-operator.giantswarm.io/private-link-mc-ingress-ip` to `AzureCluster` of workload clusters.
- The annotation for IP is handled by `dns-operator-azure`. It adds the record to the private DNS zone with MC name and links it to the workload clusters' VNET.


## License

Copyright 2023.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

