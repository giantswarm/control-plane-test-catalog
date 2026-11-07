[![CircleCI](https://circleci.com/gh/giantswarm/trivy-app.svg?style=shield)](https://circleci.com/gh/giantswarm/trivy-app)

# trivy-app

Trivy is a comprehensive security scanner supporting detection of several types of security issues across various types of target resources.

### Targets:

* Container Image
* Filesystem
* Git repository (remote)
* Kubernetes cluster or resource

### Scanners:

* OS packages and software dependencies in use (SBOM)
* Known vulnerabilities (CVEs)
* IaC misconfigurations
* Sensitive information and secrets

Read more in the [Trivy documentation](https://trivy.dev/).

## Installing

The recommended way to install this app onto a workload cluster is a Flux `HelmRelease`:

- [Deploying an application via a Flux HelmRelease](https://docs.giantswarm.io/tutorials/fleet-management/app-platform/deploy-app-helmrelease/)
- [Adding a HelmRelease via GitOps](https://docs.giantswarm.io/tutorials/continuous-deployment/helm-releases/add-helmrelease/)

## Configuring

### values.yaml

This is an example of a values file that increases the size of the Trivy cache volume and its memory limit.
The upstream chart is a dependency, so its values are nested under `trivy`.
See [`values.yaml`](helm/trivy/values.yaml) and the [upstream chart values](helm/trivy/charts/trivy/values.yaml) for all options.

```yaml
# values.yaml
trivy:
  persistence:
    size: 10Gi
  resources:
    limits:
      memory: 2Gi
```

### Deploying with kubectl-gs

You can use the [official Giant Swarm kubectl plug-in](https://github.com/giantswarm/kubectl-gs/) to create the
Flux `OCIRepository` and `HelmRelease` in the management cluster.

Here is an example that would install the app to workload cluster `abc123` of organization `example`:

```shell
kubectl gs deploy chart \
  --chart-name trivy \
  --version 0.18.0 \
  --organization example \
  --target-cluster abc123 \
  --target-namespace trivy \
  --values-file values.yaml
```

Add `--dry-run` to print the manifests without applying them.

See the [`kubectl gs deploy chart` reference](https://docs.giantswarm.io/reference/kubectl-gs/deploy-chart/) for all options.

## Development

### Upstream chart

The upstream chart from [aquasecurity/trivy](https://github.com/aquasecurity/trivy/tree/main/helm) is vendored into
`helm/trivy/charts/trivy` with [vendir](https://carvel.dev/vendir/) (see [`vendir.yml`](vendir.yml)).
Run `make update-chart` to sync it and update the chart dependencies.

## Credit

* https://github.com/aquasecurity/trivy
