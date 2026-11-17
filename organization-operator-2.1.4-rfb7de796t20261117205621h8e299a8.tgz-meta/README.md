[![CircleCI](https://circleci.com/gh/giantswarm/organization-operator.svg?&style=shield)](https://circleci.com/gh/giantswarm/organization-operator)

# organization-operator

`organization-operator` runs in every Giant Swarm management cluster and reconciles the cluster-scoped [`Organization`](https://docs.giantswarm.io/reference/platform-api/crd/organizations.security.giantswarm.io/) custom resource.
For each `Organization` it creates and owns a namespace named `org-<name>`, labelled with `giantswarm.io/organization`, which is where that organization's clusters, apps and access control live; the namespace is deleted again when the `Organization` is removed.
Organizations are the tenancy boundary of the Giant Swarm platform, so this operator is the first step of [multi-tenancy](https://docs.giantswarm.io/overview/fleet-management/multi-tenancy/) on a management cluster.

## Installing

The operator is installed on every management cluster through [management-cluster-bases](https://github.com/giantswarm/management-cluster-bases) and is not intended to be installed by customers.
The `Organization` CRD is defined in this repository under `config/crd/bases` but is installed separately from the chart; once both are present, organizations can be created for example with [`kubectl gs template organization`](https://docs.giantswarm.io/reference/kubectl-gs/template-organization/).

## Configuring

The defaults are suitable for Giant Swarm management clusters.
Metrics scraping is enabled with TLS and authentication; the example below turns off the certificate creation for a cluster without cert-manager.

```yaml
# values.yaml
serviceMonitor:
  enabled: true
  tls:
    enabled: false
  scrapeAuth:
    enabled: false
```

See our [full reference on how to configure apps](https://docs.giantswarm.io/tutorials/fleet-management/app-platform/app-configuration/) for more details.

## Credit

- Built with [Kubebuilder](https://book.kubebuilder.io/) and [controller-runtime](https://github.com/kubernetes-sigs/controller-runtime).
