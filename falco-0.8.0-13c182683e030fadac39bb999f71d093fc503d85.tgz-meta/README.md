[![CircleCI](https://circleci.com/gh/giantswarm/falco-app.svg?style=shield)](https://circleci.com/gh/giantswarm/falco-app)

# falco chart

Giant Swarm offers a [falco](https://falco.org/) App which can be installed in workload clusters.
Here we define the falco chart with its templates and default configuration.

Falco is a host-based intrusion detection system which watches and checks Linux syscalls against a predefined list of rules. Anomalous activity (as defined by the rules) triggers a Falco event, which can be used to alert responders or take automated remediation actions.

## Installing

There are several ways to install this app onto a workload cluster.

- [Using our web interface][app-ui]
- By creating an [App resource][app-crd] in the management cluster as explained in [Getting started with App Platform][app-getting-started].

## Configuring

**Note: There are currently known compatibility issues when using the Falco kernel module with Flatcar kernel version 5.10.77-flatcar and later. The ebpf driver must be used instead (see the sample `user-values-configmap.yaml` below).**

### values.yaml

**This is an example of a values file you could upload using our web interface.**

```yaml
# values.yaml

global:
  registry: quay.io

falco:
  podSecurityPolicy:
    create: true
  falco:
    grpc:
      enabled: true
    grpcOutput:
      enabled: true
  customRules:
    {}
    # Example:
    #
    # rules-traefik.yaml: |-
    #   [ rule body ]


falco-exporter:
  podSecurityPolicy:
    create: true

falcosidekick:

```

#### Falco Configurations

Please see the below page for configurable values.
[Falco Configuration](helm/falco-app/charts/falco#configuration)

#### Falco Exporter Configurations

Please see the below page for configurable values.
[Falco Exporter Configuration](helm/falco-app/charts/falco-exporter#configuration)

#### Falco sidekick Configurations

Please see the below page for configurable values.
[Falco sidekick Configuration](helm/falco-app/charts/falcosidekick#configuration)

### Sample App CR and ConfigMap for the management cluster

If you have access to the Kubernetes API on the management cluster, you could create
the App CR and ConfigMap directly.

You can provide additional configuration via a ConfigMap or the web interface.

```yaml
# user-values-configmap.yaml
# To use the ebpf driver instead of the Falco kernel module:
falco:
  ebpf:
    enabled: "true"

```

See our [full reference page on how to configure applications][app-config] for more details.

## Credit

* https://github.com/falcosecurity/charts

[app-config]: https://docs.giantswarm.io/app-platform/app-configuration/
[app-crd]: https://docs.giantswarm.io/ui-api/management-api/crd/apps.application.giantswarm.io/
[app-getting-started]: https://docs.giantswarm.io/app-platform/getting-started/
[app-ui]: https://docs.giantswarm.io/ui-api/web/app-platform/#installing-an-app
