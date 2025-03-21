[![CircleCI](https://dl.circleci.com/status-badge/img/gh/giantswarm/observability-bundle/tree/main.svg?style=svg)](https://dl.circleci.com/status-badge/redirect/gh/giantswarm/observability-bundle/tree/main)

# observability-bundle chart

The observability bundle provides all necessary components to enable observability capabilities in a workload cluster.

## Apps

* alloy-app
* prometheus-operator-crd
* kube-prometheus-stack-app
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

#### Monitoring Custom Resources in Kube State Metrics (KSM)

You can enable various Custom Resources in KSM to generate metrics from them. The list of supported Custom Resources is in the `helm/observability-bundle/ksm-configurations` folder.
To enable a specific resource just add its name (matching the filename - without extension) in the `kubeStateMetricsCustomResources` Helm value. e.g.

```yaml
kubeStateMetricsCustomResources:
  - flux_gitrepository_v1
  - capi_machinedeployment_v1beta1
  - capi_kubeadmcontrolplane_v1beta1
```

The generated KSM configuration will be merged with the provided in `userConfig.kubePrometheusStack.configMap.values.kube-prometheus-stack.kube-state-metrics`.
