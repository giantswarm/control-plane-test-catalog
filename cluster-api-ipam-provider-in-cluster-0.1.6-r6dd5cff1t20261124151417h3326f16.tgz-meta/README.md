[![CircleCI](https://circleci.com/gh/giantswarm/cluster-api-ipam-provider-in-cluster-app.svg?style=shield)](https://circleci.com/gh/giantswarm/cluster-api-ipam-provider-in-cluster-app)

# cluster-api-ipam-provider-in-cluster

Cluster API IPAM Provider In Cluster - packaged as a Giant Swarm app.

This repository is primary used to import the upstream Cluster API IPAM Provider In Cluster manifests into Giant Swarm's own app catalog.

Content of the `/helm` directory will be bundled, released and pushed to the `app-catalog` via [`architect`](https://github.com/giantswarm/architect). This happens automatically and is done by [this](.circleci/config.yml) `circleCI` configuration.

> To keep it quite easy to update the manifest from upstream, we don't change the fetched manifests directly. All Giant Swarm specific adjustments get applied via `kustomize`.

## Usage

This app contains a helm chart for [`cluster-api-ipam-provider-in-cluster`](https://github.com/kubernetes-sigs/cluster-api-ipam-provider-in-cluster).
The application itself assumes `IpAddress` and `IpAddressClaim` custom resource definitions to exist in the cluster. 
These are installed together with common CAPI CRDs.

The [`sync/sync.sh`](sync/sync.sh) script generates the chart. It needs `vendir`, `kustomize`, `yq`, `jq` and `helm`.

### apply new `kustomize` changes to the charts

1. Write your changes as kustomize patches in [`sync/patches/kustomize`](sync/patches/kustomize).
1. Run `./sync/sync.sh` to generate the chart in [`helm/cluster-api-ipam-provider-in-cluster`](helm/cluster-api-ipam-provider-in-cluster).

### update to a newer upstream release

Renovate updates the release tag in [`vendir.yml`](vendir.yml). The `Sync vendir changes` workflow then runs `./sync/sync.sh` on the Renovate branch.

1. Make sure that the image tag in [`sync/patches/values/manifests/values.yaml`](sync/patches/values/manifests/values.yaml) agrees with the new release. The sync does not change it.
1. Run `./sync/sync.sh` again if you change the image tag.

### deploy the latest version of app
```
k gs template app --catalog control-plane-test-catalog \
                  --name cluster-api-ipam-provider-in-cluster \
                  --target-namespace default \
                  --organization giantswarm \
                  --cluster-name foo \
                  --version <latest-released-version>-gitSHa | k apply -f -
```
# Useful links

* [Upstream repo](https://github.com/kubernetes-sigs/cluster-api-ipam-provider-in-cluster)
