# muster-skill-workflows

Helm chart packaging the [muster](https://github.com/giantswarm/muster) `Workflow` custom resources that
back Claude Code skills in [giantswarm/claude-code](https://github.com/giantswarm/claude-code). Each
**skill-support workflow** replaces a sequence of Muster tool calls a skill used to spell out in prose with
one call, `workflow_<name>`, that returns only a shaped answer.

Workflows are authored and verified with the `gs-base:optimize-skill-for-muster` skill, whose references hold
the conventions (naming, labels, output envelope, description template, verification recipe).

## Layout

- `helm/muster-skill-workflows/workflows/` -- one `muster.giantswarm.io/v1alpha1` `Workflow` manifest per
  file, named by the question it answers.
- `helm/muster-skill-workflows/templates/workflows.yaml` -- emits the manifests verbatim via `.Files.Glob`.
  They live outside `templates/` because their specs contain muster's own Go-template expressions, which Helm
  must not render.
- `scenarios/` -- `muster test` scenarios with mock MCP servers, required for every write workflow.
- `hack/validate-workflows.py` -- CRD byte limits (spec 1000, arg/step/sub-step 500, including `forEach`
  bodies and `onFailure` handlers) and the read/write guard; `hack/test-validate-workflows.sh` proves the
  guard on fixtures. Both run in CI.

The manifests carry no namespace; the release installs them into its namespace (`agent-platform`).

## Conventions in short

- Labels `klaus-lab.giantswarm.io/category: skill-support` and
  `klaus-lab.giantswarm.io/source-skill: <plugin>.<skill>`; annotation `klaus-lab.giantswarm.io/consumers`
  lists every calling skill, `klaus-lab.giantswarm.io/manual-fallback` the fallback file that mirrors the
  steps.
- A workflow is read-only or a write workflow, never both. Write workflows declare `onFailure` rollback
  steps; the human confirmation lives in the consuming skill.
- Every workflow declares an `output` template, so raw step results never reach the caller.

## Why a separate repository

[muster-runbooks](https://github.com/giantswarm/muster-runbooks) holds the alert-triage runbooks. This
repository is a team outside Bumblebee shipping its own workflow chart into the shared gateway -- the same
path a customer would take to ship their own workflows.

## Adding or changing a workflow

1. Add or edit the manifest; run `python3 hack/validate-workflows.py`.
2. Open a PR with the measurement table from the live round-trip (turns and bytes, manual versus workflow).
3. On merge, the Auto Release workflow tags a patch release and CircleCI pushes the chart to
   `oci://gsociprivate.azurecr.io/charts/giantswarm/muster-skill-workflows` (private).

## Deployment

Deployed on the gazelle and graveler management clusters via Flux (`OCIRepository` with a semver range plus
`HelmRelease`) into the `agent-platform` namespace; see the `muster-skill-workflows` extra in
giantswarm-management-clusters. The `Workflow` CRD ships with the `muster-crds` chart.
