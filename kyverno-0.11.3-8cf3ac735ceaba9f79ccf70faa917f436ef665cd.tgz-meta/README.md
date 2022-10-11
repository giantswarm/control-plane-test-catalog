[![CircleCI](https://circleci.com/gh/giantswarm/kyverno-app.svg?style=shield)](https://circleci.com/gh/giantswarm/kyverno-app)

# kyverno chart

Giant Swarm offers a kyverno App which can be installed in workload clusters.

Kyverno is an admission controller which offers policy enforcement as a validating or mutating webhook.

Here we define the kyverno chart with its templates and default configuration.

## Installing

There are 3 ways to install this app onto a workload cluster.

1. [Using our web interface](https://docs.giantswarm.io/ui-api/web/app-platform/#installing-an-app)
2. [Using our API](https://docs.giantswarm.io/api/#operation/createClusterAppV5)
3. Directly creating the [App custom resource](https://docs.giantswarm.io/ui-api/management-api/crd/apps.application.giantswarm.io/) on the management cluster.

## Configuring

#### Kyverno Configurations

Please see the below page for configurable values.
[Kyverno Configuration](helm/kyverno/#configuration)


See our [full reference page on how to configure applications](https://docs.giantswarm.io/app-platform/app-configuration/) for more details.

## Credit

* https://github.com/kyverno/kyverno
