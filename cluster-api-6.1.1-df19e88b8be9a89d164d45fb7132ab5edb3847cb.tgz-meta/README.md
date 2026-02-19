[![CircleCI](https://dl.circleci.com/status-badge/img/gh/giantswarm/cluster-api-app.svg?style=svg)](https://dl.circleci.com/status-badge/redirect/gh/giantswarm/cluster-api-app)

# cluster-api-app

This is a meta app that provides deployment packaging for Cluster API components.

## Prerequisites

To get all the `make` targets running

* [`kubectl`](https://github.com/kubernetes/kubectl) in version `>= v1.27.0` is required
* [`yq`](https://github.com/mikefarah/yq) is required

## How it works

The `make generate` target transfers the upstream Cluster API components into a Giant Swarm specific Helm chart. Besides there are some other changes required to make them fit into our stack.

To make all the changes transparent and reproducible, `kubectl kustomize` is used to apply patches.

The following notable commands & scripts are triggered in `make generate`:

1. [`hack/fetch-manifest.sh`](hack/fetch-manifest.sh): Fetches the Cluster API components for the version specified in `helm/cluster-api/values.yaml`.
1. `kubectl kustomize config/helm --output helm/cluster-api/templates`: Generates kustomized Helm templates from upstream Cluster API components.
1. [`hack/move-crds.sh`](hack/move-crds.sh): Moves all the CRDs into the `helm/cluster-api/charts/crd-install/files` directory. They are later used in the CRD install job.
1. [`hack/generate-patches.sh`](hack/generate-patches.sh): Extracts the upstream Cluster API CRDs into `kustomize` patches in `helm/cluster-api/charts/crd-install/files`.
1. [`hack/remove-braces.sh`](hack/remove-braces.sh): Removes braces breaking Helm templates from CRDs.

## Upgrading Cluster API

See the [`README.md`](https://github.com/giantswarm/cluster-api/blob/main/README.md) of our Cluster API fork for testing and releasing changes.

It is important to run `make generate` so that the templates, CRDs and patches are regenerated using the new version of Cluster API.

**NOTE:** When new webhooks are added upstream, we need to manually add them to the relevant patches.
