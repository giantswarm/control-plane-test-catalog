[![CircleCI](https://circleci.com/gh/giantswarm/falco-app.svg?style=shield)](https://circleci.com/gh/giantswarm/falco-app)

# falco chart

Giant Swarm offers a [falco](https://falco.org/) App which can be installed in workload clusters.
Here we define the falco chart with its templates and default configuration.

Falco is a host-based intrusion detection system which watches and checks Linux syscalls against a predefined list of rules. Anomalous activity (as defined by the rules) triggers a Falco event, which can be used to alert responders or take automated remediation actions.

## Installing

The recommended way to install this app onto a workload cluster is a Flux `HelmRelease`:

- [Deploying an application via a Flux HelmRelease](https://docs.giantswarm.io/tutorials/fleet-management/app-platform/deploy-app-helmrelease/)
- [Adding a HelmRelease via GitOps](https://docs.giantswarm.io/tutorials/continuous-deployment/helm-releases/add-helmrelease/)

## Configuring

### values.yaml

This is an example of a values file that adds a custom Falco rule.
The upstream charts are dependencies, so their values are nested under `falco`, `falcosidekick` and `k8s-metacollector`.
See [`values.yaml`](helm/falco/values.yaml) for the defaults of this app.

```yaml
# values.yaml
falco:
  customRules:
    rules-custom.yaml: |-
      - rule: Terminal shell in container (custom)
        desc: A shell with an attached terminal was spawned in a container.
        condition: spawned_process and container and proc.name in (bash, sh) and proc.tty != 0
        output: "Shell in container (user=%user.name container=%container.name cmd=%proc.cmdline)"
        priority: NOTICE
```

Falco uses the modern eBPF probe by default (`falco.driver.kind: modern_ebpf`).

Please see the upstream charts for all configurable values:

- [Falco](helm/falco/charts/falco/README.md#values)
- [Falcosidekick](helm/falco/charts/falcosidekick/README.md)
- [k8s-metacollector](helm/falco/charts/k8s-metacollector/README.md)

### Deploying with kubectl-gs

You can use the [official Giant Swarm kubectl plug-in](https://github.com/giantswarm/kubectl-gs/) to create the
Flux `OCIRepository` and `HelmRelease` in the management cluster.

Here is an example that would install the app to workload cluster `abc123` of organization `example`:

```shell
kubectl gs deploy chart \
  --chart-name falco \
  --version 0.14.0 \
  --organization example \
  --target-cluster abc123 \
  --target-namespace falco \
  --values-file values.yaml
```

Add `--dry-run` to print the manifests without applying them.

See the [`kubectl gs deploy chart` reference](https://docs.giantswarm.io/reference/kubectl-gs/deploy-chart/) for all options.

## Development

### Upstream charts

The upstream charts from [falcosecurity/charts](https://github.com/falcosecurity/charts) are vendored into
`helm/falco/charts` with [vendir](https://carvel.dev/vendir/) (see [`vendir.yml`](vendir.yml)).
Run `make update-chart` to sync them and update the chart dependencies.

## Credit

* https://github.com/falcosecurity/charts
