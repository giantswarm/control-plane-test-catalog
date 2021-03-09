[![CircleCI](https://circleci.com/gh/giantswarm/azure-ad-pod-identity-app.svg?style=shield)](https://circleci.com/gh/giantswarm/azure-ad-pod-identity-app)

-----

**This readme documents an App on the Giant Swarm App Platform**

Read more in: [What is this repo?](#what-is-this-repo)

----

# Azure AD Pod Identity App

This App installs the [Azure AD Pod Identity] onto your tenant cluster.

Azure AD Pod Identity enables Kubernetes applications to access cloud resources securely with Azure Active Directory.

Using Kubernetes primitives, administrators configure identities and bindings to match pods. Then without any code modifications, your containerized applications can leverage any resource in the cloud that depends on Azure Active Directory as an identity provider.

**Table of Contents:**

- [Azure AD Pod Identity App](#azure-ad-pod-identity-app)
- [Installing](#installing)
  - [Sample values files for the web interface and API](#sample-values-files-for-the-web-interface-and-api)
  - [Sample App CR and ConfigMap for the Control Plane](#sample-app-cr-and-configmap-for-the-control-plane)
  - [Important note about required cluster level config](#important-note-about-required-cluster-level-config)
- [Configuration Options](#configuration-options)
- [Limitations](#limitations)
- [For developers](#for-developers)
  - [Installing the Chart locally](#installing-the-chart-locally)
  - [Release Process](#release-process)
- [What is this repo?](#what-is-this-repo)

## Installing

There are 3 ways to install this app onto a tenant cluster.

1. [Using our web interface](https://docs.giantswarm.io/reference/web-interface/app-catalog/)
2. [Using our API](https://docs.giantswarm.io/api/#operation/createClusterAppV5)
3. Directly creating the App custom resource on the Control Plane.

### Required parameters

This application cannot be installed without configuring the following required parameters:

| Parameter                        | Description                                                                                                                                    |
| -------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| `adminsecret.cloud`              | Azure cloud environment name. Should be one of: AzurePublicCloud, AzureUSGovernmentCloud, AzureChinaCloud, AzureGermanCloud                    |
| `adminsecret.subscriptionID`     | Azure subscription ID where the cluster is located                                                                                             |
| `adminsecret.resourceGroup`      | Azure resource group of the cluster                                                                                                            |
| `adminsecret.tenantID`           | Azure service principal tenantID                                                                                                               |
| `adminsecret.clientID`           | Azure service principal clientID                                                                                                               |
| `adminsecret.clientSecret`       | Azure service principal clientSecret                                                                                                           |

For more configuration options, see [Configuration Options](#configuration-options)

### Sample values files for the web interface and API

This is an example of the values file you could upload using our web interface.

```
# values.yaml

adminsecret:
  cloud: AzurePublicCloud
```

If you are not using the web interface, our (deprecated) API takes the same
structure but formatted as JSON:

```
# values.json

{
  "adminsecret": {
    "cloud": "AzurePublicCloud"
  }
}
```

### Sample App CR and ConfigMap for the Control Plane

If you have access to the Kubernetes API on the Control Plane, you could create
the App CR and ConfigMap directly.

Here is an example that would install the azure-ad-pod-identity-app to
tenant cluster `abc12`:

```
# appCR.yaml
apiVersion: application.giantswarm.io/v1alpha1
kind: App
metadata:
  labels:
    app-operator.giantswarm.io/version: 1.0.0
  name: azure-ad-pod-identity-app

  # Tenant cluster resources live in a namespace with the same ID as the
  # tenant cluster.
  namespace: abc12

spec:
  name: azure-ad-pod-identity-app
  namespace: azure-system
  catalog: giantswarm-playground
  version: 0.1.0

  userConfig:
    configMap:
      name: azure-ad-pod-identity-app-user-values
      namespace: abc12
    secret:
      name: ""
      namespace: ""

  kubeConfig:
    context:
      name: abc12-kubeconfig
    inCluster: false
    secret:
      name: abc12-kubeconfig
      namespace: abc12
```

```
# user-values-configmap.yaml

apiVersion: v1
kind: ConfigMap

metadata:
  name: azure-ad-pod-identity-app-user-values
  namespace: abc12

data:
  values: |
    adminsecret:
      cloud: AzurePublicCloud
```

If you feel like any of the configuration values need to be encrypted at rest,
you can also provide a secret. 

It is a convention to call the user level configmap `{app-name}-user-values`.
So in this case we called the ConfigMap `azure-ad-pod-identity-app-user-values`

If you place these files in a folder called `foldername`, you could use the
command: `kubectl apply foldername`, to deploy this app to a tenant cluster
with id `abc12`.

See our [full reference page on how to configure applications](https://docs.giantswarm.io/reference/app-configuration/) for more details.

## Configuration Options

All configuration options are documented in the [values.yaml](/helm/azure-ad-pod-identity-app/values.yaml) file.

## Limitations

Some of our apps have certain restrictions on how they can be deployed.
Not following these limitations will most likely result in a broken deployment.

- This app _must not_ be installed more than once.

## For developers

### Installing the Chart locally

To install the chart locally:

```bash
$ git clone https://github.com/giantswarm/azure-ad-pod-identity-app.git
$ cd azure-ad-pod-identity-app
$ helm install helm/azure-ad-pod-identity-app
```

! Beware that you need to provide Azure Credentials for the application to run.

Provide a custom `values.yaml`:

```bash
$ helm install azure-ad-pod-identity-app -f values.yaml
```

### Release Process

* Ensure CHANGELOG.md is up to date.
* Create a new GitHub release with the version e.g. `v0.1.0` and link the
changelog entry.
* This will push a new git tag and trigger a new tarball to be pushed to the
[giantswarm-playground-catalog].
* Test and verify the Azure AD Pod Identity App release across supported environments in a new or existing WIP platform release.

## What is this repo?

This repo contains a helm chart for the [Giant Swarm App Platform].
While it is _just a Helm chart_, there might be some Giant Swarm App Platform
specific values in the templates.

It is available as an `App` in the `giantswarm-playground-catalog` and `giantswarm-playground-test-catalog`.

[app-operator]: https://github.com/giantswarm/app-operator
[giantswarm-playground-catalog]: https://github.com/giantswarm/giantswarm-playground-catalog
[giantswarm-playground-test-catalog]: https://github.com/giantswarm/giantswarm-playground-test-catalog
[Azure AD Pod Identity]: https://github.com/Azure/aad-pod-identity
[Giant Swarm App Platform]: https://docs.giantswarm.io/basics/app-catalog/