[![CircleCI](https://dl.circleci.com/status-badge/img/gh/giantswarm/gateway-api-bundle/tree/main.svg?style=svg)](https://dl.circleci.com/status-badge/redirect/gh/giantswarm/gateway-api-bundle/tree/main)

[Read me after cloning this template (GS staff only)](https://handbook.giantswarm.io/docs/dev-and-releng/app-developer-processes/adding_app_to_appcatalog/)

# gateway-api-bundle chart

Giant Swarm offers a gateway-api-bundle App which can be installed in workload clusters.
Here, we define the gateway-api-bundle chart with its templates and default configuration.

**What is this app?**

**Why did we add it?**

**Who can use it?**

## Installing

There are several ways to install this app onto a workload cluster.

- [Using GitOps to instantiate the App](https://docs.giantswarm.io/tutorials/continuous-deployment/apps/add-appcr/)
- By creating an [App resource](https://docs.giantswarm.io/reference/platform-api/crd/apps.application.giantswarm.io) using the platform API as explained in [Getting started with App Platform](https://docs.giantswarm.io/tutorials/fleet-management/app-platform/).

## Configuring

### values.yaml

**This is an example of a values file you could upload using our web interface.**

```yaml
# values.yaml

```

### Sample App CR and ConfigMap for the management cluster

If you have access to the Kubernetes API on the management cluster, you could create the App CR and ConfigMap directly.

Here is an example that would install the app to workload cluster `abc12`:

```yaml
# appCR.yaml

```

```yaml
# user-values-configmap.yaml

```

See our [full reference on how to configure apps](https://docs.giantswarm.io/tutorials/fleet-management/app-platform/app-configuration/) for more details.

## Compatibility

| Gateway API Bundle | Gateway API CRD App | Envoy Gateway App |
| --- | --- | --- |
| 1.8.x | 1.6.x (Gateway API v1.4.x) | 1.4.x (EnvoyGateway 1.6.x) |
| 1.7.x | 1.4.x (Gateway API v1.3.0) | 1.3.x (EnvoyGateway 1.5.x) |
| 1.2.x - 1.6.x | 1.4.x (Gateway API v1.3.0) | 1.2.x (EnvoyGateway 1.5.x) |
| 1.1.x | 1.4.x (Gateway API v1.3.0) | 1.1.x (EnvoyGateway 1.5.x) |
| 1.0.x | 1.4.x (Gateway API v1.3.0) | 1.0.x (EnvoyGateway 1.4.x) |
| 0.5.x | 1.3.0 (Gateway API v1.2.1) | 0.3.0 (EnvoyGateway 1.3.x) |

## Limitations

Some apps have restrictions on how they can be deployed.
Not following these limitations will most likely result in a broken deployment.

- _add limitation_

