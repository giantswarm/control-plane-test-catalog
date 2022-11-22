[![CircleCI](https://circleci.com/gh/giantswarm/prometheus-operator-crd.svg?style=shield)](https://circleci.com/gh/giantswarm/prometheus-operator-crd)

# prometheus-operator-crd chart

Giant Swarm offers a prometheus-operator-crd App which can be installed in workload clusters.
Here we define the prometheus-operator-crd chart with its templates and default configuration.

**What is this app?**

This app is a dependency for [prometheus-operator-app](https://github.com/giantswarm/prometheus-operator-app), which install the required CustomResourceDefinitions.

**Why did we add it?**

CRD management does not comes out of the box, so we need a separate app/chart in order to be able to fully manage (install/upgrade/delete) the CRDs.

**Who can use it?**

Anyone can use this.

## Installing

There are 3 ways to install this app onto a workload cluster.

1. [Using our web interface](https://docs.giantswarm.io/ui-api/web/app-platform/#installing-an-app)
2. [Using our API](https://docs.giantswarm.io/api/#operation/createClusterAppV5)
3. Directly creating the [App custom resource](https://docs.giantswarm.io/ui-api/management-api/crd/apps.application.giantswarm.io/) on the management cluster.

### Sample App CR for the management cluster

If you have access to the Kubernetes API on the management cluster, you could create the App CR and ConfigMap directly.

Here is an example that would install the app to workload cluster `abc12`:

```
# appCR.yaml
apiVersion: application.giantswarm.io/v1alpha1
kind: App
metadata:
  name: prometheus-operator-crd
  namespace: abc12
spec:
  catalog: control-plane-test-catalog
  config:
    configMap:
      name: abc12-cluster-values
      namespace: abc12
  kubeConfig:
    context:
      name: abc12
    inCluster: false
    secret:
      name: abc12-kubeconfig
      namespace: abc12
  name: prometheus-operator-crd
  namespace: prometheus-operator-crd
  version: 0.1.0
```

See our [full reference page on how to configure applications](https://docs.giantswarm.io/app-platform/app-configuration/) for more details.

## Credits

* https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack

## Upgrade

In order to sync CRDs from upstream, run:

* Github action `crds-upgrade`

![Screen Shot 2022-10-25 at 2 31 59 PM](https://user-images.githubusercontent.com/15221272/197787276-047d67fa-0de4-478a-89e1-834e1316050a.png)


OR

* Run this script:

```bash
VERSION=32.4.0 hack/sync.sh
```
