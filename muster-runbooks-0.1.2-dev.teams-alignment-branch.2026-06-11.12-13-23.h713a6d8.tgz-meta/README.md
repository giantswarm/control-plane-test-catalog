# muster-runbooks

Helm chart packaging the [muster](https://github.com/giantswarm/muster)
`Workflow` custom resources that implement alert triage runbooks for the
Giant Swarm agentic platform.

Each workflow in `helm/muster-runbooks/workflows/` encodes the read-only
investigation steps for one alert (Prometheus queries, Kubernetes object
inspection, event collection) and is executed by agents through the muster
MCP aggregator.

## Layout

- `helm/muster-runbooks/workflows/` -- one YAML manifest per alert runbook
  (`muster.giantswarm.io/v1alpha1` `Workflow`).
- `helm/muster-runbooks/templates/workflows.yaml` -- emits the workflow
  manifests verbatim via `.Files.Glob`. The manifests are deliberately kept
  outside `templates/` because their spec fields contain muster's own
  Go-template expressions, which must not be rendered by Helm.

The manifests carry no namespace; resources are installed into the Helm
release namespace.

## Adding or changing a workflow

1. Add or edit the manifest in `helm/muster-runbooks/workflows/`.
2. Open a PR. On merge, the Auto Release workflow tags a new patch release
   and CircleCI pushes the chart to
   `oci://gsociprivate.azurecr.io/charts/giantswarm/muster-runbooks`
   (this is a private repository, so the chart is treated as private).

## Deployment

Deployed on the gazelle management cluster via Flux (`OCIRepository` +
`HelmRelease` with semver range, auto-tracking the latest release) into the
`agentic-platform` namespace. See the `muster-runbooks` extra in the
giantswarm-management-clusters repository.

## Prerequisites

The `Workflow` CRD (`muster.giantswarm.io`) must already be present on the
cluster -- on the agentic platform it ships with the `agentic-platform-crds`
chart.
