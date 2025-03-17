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

Read more in the (Trivy documentation)[https://aquasecurity.github.io/trivy/]

## Installing

There are several ways to install this app onto a workload cluster.

- [Using our web interface][app-ui]
- By creating an [App resource][app-crd] in the management cluster as explained in [Getting started with App Platform][app-getting-started].

## Configuring

### values.yaml
**This is an example of a values file you could upload using our web interface.**
```yaml
# values.yaml
trivy:
  modules:
    # Enable Trivy modules feature and install the spring4shell module
    enabled: true
    urls:
    - ghcr.io/aquasecurity/trivy-module-spring4shell
```

See our [full reference page on how to configure applications][app-config] for more details.

## Development

### Subtrees
This repo is configured to have a git subtree split folder helm/trivy from https://github.com/giantswarm/trivy-upstream at helm/trivy-app/charts/trivy/ in the local repository.

## Credit

* https://github.com/aquasecurity/trivy

[app-config]: https://docs.giantswarm.io/app-platform/app-configuration/
[app-crd]: https://docs.giantswarm.io/ui-api/management-api/crd/apps.application.giantswarm.io/
[app-getting-started]: https://docs.giantswarm.io/app-platform/getting-started/
[app-ui]: https://docs.giantswarm.io/ui-api/web/app-platform/#installing-an-app
