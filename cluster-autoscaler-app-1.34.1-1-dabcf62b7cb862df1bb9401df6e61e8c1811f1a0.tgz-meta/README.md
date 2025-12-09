[![CircleCI](https://circleci.com/gh/giantswarm/cluster-autoscaler-app.svg?style=shield)](https://circleci.com/gh/giantswarm/cluster-autoscaler-app)

# Cluster Autoscaler

Helm chart for the Kubernetes [Cluster Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler) running in Giant Swarm tenant clusters.

## Deployment

* Managed by [app-operator].
* Production releases are stored in the [default-catalog].
* WIP releases are stored in the [default-test-catalog].

## Installation

To install the chart locally:

```bash
$ git clone https://github.com/giantswarm/cluster-autoscaler-app.git
$ cd cluster-autoscaler-app
$ helm install helm/cluster-autoscaler-app
```

Provide a custom `values.yaml`:

```bash
$ helm install helm/cluster-autoscaler-app --values values.yaml
```

## Release Process

* Ensure CHANGELOG.md is up to date.
* Create a new branch with name `release#vx.x.x-gsx`. Automation will create a release PR.
  * Make sure the major, minor and patch version matches with upstream.
* Merging the release PR will push a new Git tag and trigger a new tarball to be pushed to the [default-catalog].
* Update [cluster-operator] with the new version.

[app-operator]: https://github.com/giantswarm/app-operator
[cluster-operator]: https://github.com/giantswarm/cluster-operator
[default-catalog]: https://github.com/giantswarm/default-catalog
[default-test-catalog]: https://github.com/giantswarm/default-test-catalog
