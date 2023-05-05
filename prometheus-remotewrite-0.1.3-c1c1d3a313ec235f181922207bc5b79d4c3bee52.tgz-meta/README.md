[![CircleCI](https://dl.circleci.com/status-badge/img/gh/giantswarm/prometheus-remotewrite/tree/main.svg?style=svg&circle-token=a8dd07e3519331cc58fdc3bcd57d36d9702c5f59)](https://dl.circleci.com/status-badge/redirect/gh/giantswarm/prometheus-remotewrite/tree/main)

# prometheus-remotewrite chart

This application provides a way to configure all Giant Swarm Prometheus servers with multiple remote write destinations.

## Installing

This application is only meant to be installed on management clusters. This is achieved via our usual `opsctl deploy` tooling command.

See details : https://intranet.giantswarm.io/docs/dev-and-releng/how-to-deploy-to-a-management-cluster/#deploying-a-unique-app

## Configuring

### values.yaml

**This is an example of a values file you could upload using our web interface.**

```yaml
# values.yaml
remoteWrite:
  grafanaCloud:
    url: http://some-prometheus.endpoint
    username: foo
    apiKey: bar
```

The current configuration for different management clusters is held into the config repository. See :

https://github.com/giantswarm/config/blob/main/default/apps/prometheus-remotewrite/configmap-values.yaml.template

https://github.com/giantswarm/config/blob/main/installations/gauss/apps/prometheus-remotewrite/secret-values.yaml.patch

See the [full reference on how to configure apps](https://docs.giantswarm.io/app-platform/app-configuration/) for more details.

## Architecture

This application contains remote write resources which definition lives in [prometheus-meta-operator][prometheus-meta-operator]. This definition is based on the upstream [Prometheus.remoteWrite][prometheus remotewrite spec] definition. Details of definition and implementation can be found in the [roadmap][roadmap] issue.

More details about Prometheus remote write feature :

- [prometheus remotewrite configuration](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#remote_write)
- [prometheus remotewrite tuning](https://prometheus.io/docs/practices/remote_write/)

[prometheus remotewrite spec]: https://github.com/prometheus-operator/prometheus-operator/blob/main/Documentation/api.md#remotewritespec
[prometheus-meta-operator]: https://github.com/giantswarm/prometheus-meta-operator#remotewrite-crs
[roadmap]: https://github.com/giantswarm/roadmap/issues/496
