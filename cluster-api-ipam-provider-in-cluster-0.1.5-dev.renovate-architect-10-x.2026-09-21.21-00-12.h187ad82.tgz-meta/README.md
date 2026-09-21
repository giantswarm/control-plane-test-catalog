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

### apply new `kustomize` changes to the charts

1. if not already done, run `make fetch-upstream-manifest` (only has to be done once)
   > upstream manifest will be stored in [`config/kustomize/origin`](config/kustomize/origin)
1. write your desired changes as kustomize patches in [config/kustomize]
1. run `make apply-kustomize-patches` to apply the changes\n
   > this will generate a patched version under [`config/kustomize/tmp`](config/kustomize/tmp)
1. once you're done, run `make release-manifests` to move all relevant files into the [`helm/cluster-api-ipam-provider-in-cluster`](helm/cluster-api-ipam-provider-in-cluster) folder

### update to a newer CAIP release

1. edit the value of `TAG_TO_SYNC` in [the Makefile](Makefile.custom.mk) to the desired commit to pin in the source repo.
2. run `make all`
3. also make sure the tag of the container image in the default values.yaml is updated if you want to use the newly released thing

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
