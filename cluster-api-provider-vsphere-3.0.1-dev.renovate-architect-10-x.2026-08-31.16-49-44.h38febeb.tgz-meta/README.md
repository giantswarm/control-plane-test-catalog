[![CircleCI](https://circleci.com/gh/giantswarm/cluster-api-provider-vsphere-app.svg?style=shield)](https://circleci.com/gh/giantswarm/cluster-api-provider-vsphere-app)

# cluster-api-provider-vsphere-app

Cluster API Provider vSphere - packaged as a Giant Swarm app.

This repository is primary used to import the upstream Cluster API Provider vSphere manifests into Giant Swarm's own app catalog.

Content of the `/helm` directory will be bundled, released and pushed to the `app-catalog` via [`architect`](https://github.com/giantswarm/architect). This happens automatically and is done by [this](.circleci/config.yml) `circleCI` configuration.

> To keep it quite easy to update the manifest from upstream, we don't change the fetched manifests directly. All Giant Swarm specific adjustments get applied via `kustomize`.

## Usage

How to work within this repository?

### apply new `kustomize` changes to the charts

1. if not already done, run `make fetch-upstream-manifest` (only has to be done once)
   > upstream manifest will be stored in [`config/kustomize/origin`](config/kustomize/origin)
1. write your desired changes as kustomize patches in [config/kustomize]
1. run `make apply-kustomize-patches` to apply the changes\n
   > this will generate a patched version under [`config/kustomize/tmp`](config/kustomize/tmp)
1. once you're done, run `make release-manifests` to move all relevant files into the [`helm/cluster-api-provider-vsphere`](helm/cluster-api-provider-vsphere) folder

### update to a newer CAPVCD release

1. edit the value of `COMMIT_TO_SYNC` in [the Makefile](Makefile.custom.mk) to the desired commit to pin in the source repo.
2. run `make all`

# Useful links

* [vSphere doc](https://github.com/kubernetes-sigs/cluster-api-provider-vsphere/blob/main/docs/getting_started.md)
* internal vSphere fork - [`giantswarm/cluster-api-provider-vsphere`](https://github.com/giantswarm/cluster-api-provider-vsphere)