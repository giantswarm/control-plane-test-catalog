[![CircleCI](https://circleci.com/gh/giantswarm/trivy-operator-app.svg?style=shield)](https://circleci.com/gh/giantswarm/trivy-operator-app)

# Trivy Operator

Giant Swarm offers an app for Aqua Security's [Trivy Operator][trivy-operator-upstream], which can be installed in workload clusters. It is part of our [managed security solution][managed-security], but can also be installed independently.

Trivy Operator is an in-cluster component which runs vulnerability scans, Kubernetes CIS and NSA benchmarks, and other types of policy and best practice validation scans using [Trivy][trivy-aqua]. It replaces a previous project, called [Starboard][starboard], which has now been fully deprecated along with our [Starboard App][starboard-app].

The results of these scans are saved in the cluster in the form of Kubernetes custom resources named `VulnerabilityReport`, `ConfigAuditReport`, and other types of reports.

Results of each scan type can be retrieved from the cluster, for example using `kubectl`:

```shell
$ kubectl get vulnerabilityreports
NAMESPACE   NAME    REPOSITORY   TAG   SCANNER   AGE
...
```

You can also export the data from these reports to Prometheus to use in alerts and Grafana dashboards using our [`starboard-exporter`][starboard-exporter].

This repository contains our packaging and Giant Swarm-specific configuration of the upstream charts.

## Installing

There are several ways to install this app onto a workload cluster.

- [Using our web interface][app-ui].
- By creating an [App resource][app-crd] in the management cluster as explained in [Getting started with App Platform][app-getting-started].

### Scanning Backend

To perform vulnerability scans and produce reports, Trivy Operator needs a vulnerability scanner.

If deploying Trivy Operator as part of our managed security bundle, a Trivy server will be automatically installed for you.

If installing Trivy Operator independently, we recommend first installing our [Trivy app][trivy-app] in your cluster to serve as the vulnerability scanner for Trivy Operator, or using another existing Trivy instance.

Alternatively, you can configure Trivy Operator to use Standalone mode, which creates a new instance of the Trivy scanner per-scan. This is very inefficient and can lead to throttling by the backing vulnerability database. To do it anyway, set `trivy.mode` to `Standalone` in `values.yaml`.

In either case, please note that the Trivy version set by `trivy.imageRef` must be the same version as your Trivy backend (even if the actual image is not the same), as Trivy Operator uses that value internally to determine the API format to use for Trivy.

### Sample App CR and ConfigMap for the management cluster

If you have access to the Kubernetes API on the management cluster, you could create
the App CR and ConfigMap directly.

See our [full reference on how to configure apps][app-config] for more details.

## Credit

- [`trivy-operator`][trivy-operator-upstream]

[app-config]: https://docs.giantswarm.io/app-platform/app-configuration/
[app-crd]: https://docs.giantswarm.io/ui-api/management-api/crd/apps.application.giantswarm.io/
[app-getting-started]: https://docs.giantswarm.io/app-platform/getting-started/
[app-ui]: https://docs.giantswarm.io/ui-api/web/app-platform/#installing-an-app
[managed-security]: https://docs.giantswarm.io/app-platform/apps/security/
[starboard]: https://github.com/aquasecurity/starboard
[starboard-app]: https://github.com/giantswarm/starboard-app
[starboard-exporter]: https://github.com/giantswarm/starboard-exporter
[trivy-app]: https://github.com/giantswarm/trivy-app/
[trivy-aqua]: https://github.com/aquasecurity/trivy
[trivy-operator-upstream]: https://github.com/aquasecurity/trivy-operator
