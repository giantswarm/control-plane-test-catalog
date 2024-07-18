[![CircleCI](https://dl.circleci.com/status-badge/img/gh/giantswarm/observability-bundle/tree/main.svg?style=svg)](https://dl.circleci.com/status-badge/redirect/gh/giantswarm/observability-bundle/tree/main)

# observability-bundle chart

The observability bundle provides all necessary components to enable observability capabilities in a workload cluster.

## Apps

* alloy-app
* prometheus-operator-crd
* prometheus-operator-app
* prometheus-agent
* promtail-app
* grafana-agent-app

## Installing

There are several ways to install this app onto a workload cluster.

- [Using our web interface](https://docs.giantswarm.io/ui-api/web/app-platform/#installing-an-app).
- By creating an [App resource](https://docs.giantswarm.io/ui-api/management-api/crd/apps.application.giantswarm.io/) in the management cluster as explained in [Getting started with App Platform](https://docs.giantswarm.io/app-platform/getting-started/).

## Upgrading

See our [upgrade guide](./docs/upgrade.md).

## Configuring

### values.yaml

for each app you can use `userConfig` to supply values
or `extraConfigs` as secret or configmap

