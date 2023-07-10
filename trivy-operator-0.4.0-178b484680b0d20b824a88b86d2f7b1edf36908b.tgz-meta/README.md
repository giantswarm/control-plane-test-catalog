[![CircleCI](https://circleci.com/gh/giantswarm/trivy-operator-app.svg?style=shield)](https://circleci.com/gh/giantswarm/trivy-operator-app)

# trivy-operator chart

Giant Swarm offers a trivy-operator App which can be installed in workload clusters.
Here we define the trivy-operator chart with its templates and default configuration.

## Installing

There are several ways to install this app onto a workload cluster.

- [Using our web interface](https://docs.giantswarm.io/ui-api/web/app-platform/#installing-an-app).
- By creating an [App resource](https://docs.giantswarm.io/ui-api/management-api/crd/apps.application.giantswarm.io/) in the management cluster as explained in [Getting started with App Platform](https://docs.giantswarm.io/app-platform/getting-started/).

## Configuring

### values.yaml

**This is an example of a values file you could upload using our web interface.**

```yaml
# values.yaml

```

### Sample App CR and ConfigMap for the management cluster

If you have access to the Kubernetes API on the management cluster, you could create
the App CR and ConfigMap directly.

See our [full reference on how to configure apps](https://docs.giantswarm.io/app-platform/app-configuration/) for more details.

## Credit

- [`trivy-operator`][trivy-operator-upstream]

[trivy-operator-upstream]: https://github.com/aquasecurity/trivy-operator
