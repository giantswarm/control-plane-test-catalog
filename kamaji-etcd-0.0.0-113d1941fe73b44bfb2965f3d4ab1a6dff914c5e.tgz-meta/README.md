[![CircleCI](https://dl.circleci.com/status-badge/img/gh/giantswarm/kamaji-etcd-app/tree/main.svg?style=svg)](https://dl.circleci.com/status-badge/redirect/gh/giantswarm/kamaji-etcd-app/tree/main)
[![OpenSSF Scorecard](https://api.securityscorecards.dev/projects/github.com/giantswarm/kamaji-etcd-app/badge)](https://securityscorecards.dev/viewer/?uri=github.com/giantswarm/kamaji-etcd-app)

[Guide about how to manage an app on Giant Swarm](https://handbook.giantswarm.io/docs/dev-and-releng/app-developer-processes/adding_app_to_appcatalog/)

# kamaji-etcd chart

Giant Swarm offers a kamaji-etcd App which can be installed in clusters.
Here, we define the kamaji-etcd chart with its templates and default configuration.

This chart is synced from the upstream [clastix/kamaji-etcd](https://github.com/clastix/kamaji-etcd) repository using vendir.

**What is this app?**

kamaji-etcd provides a multi-tenant etcd cluster for use with Kamaji, the Kubernetes control plane manager.

**Why did we add it?**

To support Kamaji deployments that require a dedicated etcd datastore.

**Who can use it?**

Anyone deploying Kamaji control planes that need etcd as a datastore.

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

## Updating the Chart

This chart is synced from the upstream [clastix/kamaji-etcd](https://github.com/clastix/kamaji-etcd) repository using [vendir](https://carvel.dev/vendir/).
Then, custom Giant Swarm patches are applied to adapt the chart to our requirements.

To update to the latest version, see [sync/README.md](sync/README.md).

The vendir configuration will pull the chart from `charts/kamaji-etcd/` in the upstream repository.

## Compatibility

This app has been tested to work with the following workload cluster release versions:

- _add release version_

## Limitations

Some apps have restrictions on how they can be deployed.
Not following these limitations will most likely result in a broken deployment.

- _add limitation_

## Credit

This chart is sourced from the upstream [clastix/kamaji-etcd](https://github.com/clastix/kamaji-etcd) Helm chart repository and maintained by the Clastix team.
