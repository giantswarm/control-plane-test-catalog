[![CircleCI](https://dl.circleci.com/status-badge/img/gh/giantswarm/plugin-barman-cloud/tree/main.svg?style=svg)](https://dl.circleci.com/status-badge/redirect/gh/giantswarm/plugin-barman-cloud/tree/main)

[Guide about how to manage an app on Giant Swarm](https://handbook.giantswarm.io/docs/dev-and-releng/app-developer-processes/adding_app_to_appcatalog/)

# plugin-barman-cloud chart

Giant Swarm offers a plugin-barman-cloud App which can be installed in workload clusters.
Here, we define the plugin-barman-cloud chart with its templates and default configuration.

**What is this app?**

**Why did we add it?**

**Who can use it?**

## Installing

There are several ways to install this app onto a workload cluster.

- [Using GitOps to instantiate the App](https://docs.giantswarm.io/tutorials/continuous-deployment/apps/add-appcr/)
- By creating an [App resource](https://docs.giantswarm.io/reference/platform-api/crd/apps.application.giantswarm.io) using the platform API as explained in [Getting started with App Platform](https://docs.giantswarm.io/tutorials/fleet-management/app-platform/).

## Certificates

The plugin and the CloudNativePG operator talk over mTLS. With `pluginbarmancloud.enabled: true`,
the chart has cert-manager issue both certificates, `barman-cloud-server-tls` and
`barman-cloud-client-tls`, and cert-manager renews them. They come from a namespaced, self-signed
`Issuer` the chart creates in the release namespace (`pluginbarmancloud.certificate.createIssuer`).

That Issuer is named `letsencrypt-giantswarm`, but it is not Let's Encrypt: the name is historical.
Keep it. A new `issuerName` makes cert-manager reissue both certificates under a new CA.

The plugin therefore runs on any cluster with cert-manager, not only on Giant Swarm management
clusters. To use an issuer of your own instead, set `pluginbarmancloud.certificate.createIssuer: false`
and `pluginbarmancloud.certificate.issuerName` to a namespaced `Issuer` in the release namespace.

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

This app has been tested to work with the following workload cluster release versions:

- _add release version_

## Limitations

Some apps have restrictions on how they can be deployed.
Not following these limitations will most likely result in a broken deployment.

- _add limitation_

## Credit

- {APP HELM REPOSITORY}
