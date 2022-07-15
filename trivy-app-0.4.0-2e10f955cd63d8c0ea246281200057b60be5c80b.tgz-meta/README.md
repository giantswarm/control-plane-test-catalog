[![CircleCI](https://circleci.com/gh/giantswarm/trivy-app-app.svg?style=shield)](https://circleci.com/gh/giantswarm/trivy-app-app)

# trivy-app chart

Giant Swarm offers a trivy App which can be installed in workload clusters.
Here we define the trivy-app chart with its templates and default configuration.

**What is this app?**
**Why did we add it?**
**Who can use it?**

## Installing

There are several ways to install this app onto a workload cluster.

- [Using our web interface][app-ui]
- By creating an [App resource][app-crd] in the management cluster as explained in [Getting started with App Platform][app-getting-started].

## Configuring

### values.yaml
**This is an example of a values file you could upload using our web interface.**
```
# values.yaml

```

### Sample App CR and ConfigMap for the management cluster
If you have access to the Kubernetes API on the management cluster, you could create
the App CR and ConfigMap directly.

Here is an example that would install the app to
workload cluster `abc12`:

```
# appCR.yaml

```

```
# user-values-configmap.yaml


```

See our [full reference page on how to configure applications][app-config] for more details.

## Compatibility

This app has been tested to work with the following workload cluster release versions:

*

## Limitations

Some apps have restrictions on how they can be deployed.
Not following these limitations will most likely result in a broken deployment.

*

## Credit

* https://github.com/aquasecurity/trivy

[app-config]: https://docs.giantswarm.io/app-platform/app-configuration/
[app-crd]: https://docs.giantswarm.io/ui-api/management-api/crd/apps.application.giantswarm.io/
[app-getting-started]: https://docs.giantswarm.io/app-platform/getting-started/
[app-ui]: https://docs.giantswarm.io/ui-api/web/app-platform/#installing-an-app
