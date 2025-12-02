[![CircleCI](https://circleci.com/gh/giantswarm/cluster-autoscaler-app.svg?style=shield)](https://circleci.com/gh/giantswarm/cluster-autoscaler-app)

# Cluster Autoscaler

Giant Swarm offers a `cluster-autoscaler-app-bundle` Managed App which can be installed in tenant clusters.

Here we define the `cluster-autoscaler-app-bundle`, `cluster-autoscaler-app` charts with their templates and default configuration.

## Architecture

This repository contains two Helm charts:

- `helm/cluster-autoscaler-app-bundle/`: Main chart installed on the management cluster, contains the workload cluster chart and the required AWS IAM role.
- `helm/cluster-autoscaler-app/`: Workload cluster chart that contains the actual Cluster Autoscaler setup.

Users only need to install the bundle chart on the management cluster, which in turn will deploy the workload cluster chart.

## Deployment

* Managed by [app-operator].
* Production releases are stored in the [default-catalog] and [control-plane-catalog].
* WIP releases are stored in the [default-test-catalog] and [control-plane-test-catalog].

## Installation

Install the chart on the management cluster using an App CR:

```yaml
apiVersion: application.giantswarm.io/v1alpha1
kind: App
metadata:
  name: coyote-cluster-autoscaler-app-bundle
  namespace: org-acme
spec:
  catalog: default
  config:
    configMap:
      name: coyote-cluster-values
      namespace: org-acme
  kubeConfig:
    inCluster: true
  name: cluster-autoscaler-app-bundle
  namespace: org-acme
  version: 1.34.1-1
```

## Release Process

* Ensure CHANGELOG.md is up to date.
* Create a new branch with name `release#vx.x.x-gsx`. Automation will create a release PR.
  * Make sure the major, minor and patch version matches with upstream.
* Merging the release PR will push a new Git tag and trigger a new tarball to be pushed to the [default-catalog] and [control-plane-catalog].
* Update [cluster-operator] with the new version.

[app-operator]: https://github.com/giantswarm/app-operator
[cluster-operator]: https://github.com/giantswarm/cluster-operator
[default-catalog]: https://github.com/giantswarm/default-catalog
[default-test-catalog]: https://github.com/giantswarm/default-test-catalog
[control-plane-catalog]: https://github.com/giantswarm/control-plane-catalog
[control-plane-test-catalog]: https://github.com/giantswarm/control-plane-test-catalog
