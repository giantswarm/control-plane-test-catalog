[![CircleCI](https://circleci.com/gh/giantswarm/ingress-nginx-app.svg?style=shield)](https://circleci.com/gh/giantswarm/ingress-nginx-app)

# Ingress NGINX Controller

This app installs the [Ingress NGINX Controller](https://github.com/kubernetes/ingress-nginx) into your cluster.

Its job is to satisfy external requests to services running in the cluster. See the [Kubernetes Ingress documentation](https://kubernetes.io/docs/concepts/services-networking/ingress) for a higher level overview.

## Prerequisites

- Kubernetes version >= v1.20.0
- Ingress API version >= `networking.k8s.io/v1` (`extensions/v1beta1` & `networking.k8s.io/v1beta1` are no longer supported)

Additionally, you'll have to make sure your `Ingress` resources are using the `.spec.ingressClassName` field, matching the name of your `IngressClass` resource (default: `nginx`).

In case it is not possible to update all `Ingress` resources to specify the `.spec.ingressClassName`, you can add the `--watch-ingress-without-class` argument using the `controller.extraArgs`. Please make sure there is no other Ingress Controller doing so deployed to your cluster.

If you are installing [multiple Ingress NGINX Controllers](https://docs.giantswarm.io/advanced/ingress/multi-nginx-ic), each needs to have an unique `IngressClass` resource. You can use the values structure below:

```
controller:
  ingressClassResource:
    name: nginx-internal
    controllerValue: k8s.io/ingress-nginx-internal
```

**Note:** It is not possible to change the `controller.ingressClassResource` values after installation, except you are also changing `controller.ingressClassResource.name`. If you need to change these values, you will need to uninstall the app first.

## Installation

There are two ways to install this app into your cluster:

1. [Using our web interface](https://docs.giantswarm.io/ui-api/web/app-platform#installing-an-app).
2. Directly creating the [`App` resource](https://docs.giantswarm.io/use-the-api/management-api/crd/apps.application.giantswarm.io) in the management cluster.

### Sample values files for the web interface

This is an example of the values file you could upload using our web interface:

```
controller:
  config:
    error-log-level: info
```

### Sample App CR and ConfigMap for the management cluster

If you have access to the Kubernetes API of the management cluster, you can create the `ConfigMap` & `App` resources directly. These sample values deploy the `ingress-nginx` app to your cluster `abc12`:

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: ingress-nginx-user-values
  namespace: abc12
data:
  values: |
    controller:
      config:
        error-log-level: info
---
apiVersion: application.giantswarm.io/v1alpha1
kind: App
metadata:
  name: ingress-nginx
  namespace: abc12
spec:
  catalog: giantswarm
  name: ingress-nginx
  version: 3.0.0
  userConfig:
    configMap:
      name: ingress-nginx-user-values
      namespace: abc12
  namespace: kube-system
```

If you feel like any of the configuration values need to be encrypted at rest, you can also provide a secret. For this app we don't think there are any configuration values that need to be encrypted.

See our [full reference page on how to configure applications](https://docs.giantswarm.io/app-platform/app-configuration) for more details.

## Upgrading

TODO

## Configuration

All configuration options are documented in the [`values.yaml`](https://github.com/giantswarm/ingress-nginx-app/blob/main/helm/ingress-nginx/values.yaml) file.

### Internal Ingress NGINX Controller

This chart contains a template for an additional [internal service](https://github.com/giantswarm/ingress-nginx-app/blob/main/helm/ingress-nginx/templates/controller-service-internal.yaml) to cover the use case of having separate external & internal services with a single Ingress NGINX Controller instance.

Valid configuration options are as follows:

**External only (default)**

This is the default behavior. No additional configuration required.

**Internal only**

This configures the [default service](https://github.com/giantswarm/ingress-nginx-app/blob/main/helm/ingress-nginx/templates/controller-service.yaml) to create an internal facing load balancer:

 ```yaml
controller:
  service:
    public: false
 ```

**External & internal**

This enables the additional [internal service](https://github.com/giantswarm/ingress-nginx-app/blob/main/helm/ingress-nginx/templates/controller-service-internal.yaml) and creates an internal facing load balancer.

```yaml
controller:
  service:
    internal:
      enabled: true
```

### AWS & PROXY protocol

Since version v2.17.0, this chart has the `use-proxy-procotol` enabled by default when installed in AWS.

To disable this behavior, it's still possible to set `controller.config.use-proxy-protocol` to `"false"` in the values.

## For developers

### Local installation

To install the chart locally:

```bash
$ git clone https://github.com/giantswarm/ingress-nginx-app.git
$ cd ingress-nginx-app
$ helm install helm/ingress-nginx
```

Provide a custom `values.yaml`:

```bash
$ helm install helm/ingress-nginx --values values.yaml
```

### Release process

* Ensure CHANGELOG.md is up to date.
* Create a new branch with name `release#vx.x.x`. Automation will create a release PR.
* Merging the release PR will push a new Git tag and trigger a new tarball to be pushed to the [giantswarm-catalog].
