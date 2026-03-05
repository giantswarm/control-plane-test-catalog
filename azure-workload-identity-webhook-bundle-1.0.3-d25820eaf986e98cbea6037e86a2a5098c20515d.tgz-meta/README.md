[![CircleCI](https://dl.circleci.com/status-badge/img/gh/giantswarm/azure-workload-identity-webhook-app/tree/main.svg?style=svg)](https://dl.circleci.com/status-badge/redirect/gh/giantswarm/azure-workload-identity-webhook-app/tree/main)

# azure-workload-identity-webhook chart

Giant Swarm offers an `azure-workload-identity-webhook-bundle` Managed App which can be installed in management clusters.
Here we define the `azure-workload-identity-webhook-bundle` and `azure-workload-identity-webhook` charts with their templates and default configuration.

## Key terminology

| Term | Where it lives | What it does |
|------|---------------|--------------|
| **BUNDLE-ONLY** | Management cluster only | Never forwarded to workload chart. Examples: `clusterID`, `ociRepositoryUrl` |
| **UPSTREAM** | Workload cluster, under `upstream:` key | Routed to the unmodified upstream subchart. Controls the actual application: image, replicas, resources, etc. |
| **EXTRAS** | Workload cluster, at top level (not under `upstream:`) | Consumed by GS extras templates: `networkPolicy`, `verticalPodAutoscaler`, `global.podSecurityStandards`, etc. |

## Architecture

```
Bundle values.yaml (management cluster)
    |
    v
_helpers.tpl (giantswarm.workloadValues)
  - Excludes bundle-only keys (ociRepositoryUrl, clusterID)
  - Combines split registry+repository into single repository path
  - Routes upstream values under upstream: key
  - Routes extras (networkPolicy, VPA, global) at top level
    |
    v
ConfigMap ({clusterID}-azure-workload-identity-webhook-config)
    |
    v
Flux HelmRelease (valuesFrom: ConfigMap)
    |
    v
Workload chart values (deep-merged with workload values.yaml defaults)
    |
    +-- upstream: {}  ->  upstream subchart (alias: upstream)
    +-- top-level     ->  GS extras templates (CiliumNetworkPolicy, VPA, PSS)
```

### Charts

| Chart | Location | Deployed to | Purpose |
|-------|----------|-------------|---------|
| **Bundle chart** | `helm/azure-workload-identity-webhook-bundle/` | Management cluster | Orchestrator. Creates Flux resources (OCIRepository + HelmRelease) and a ConfigMap with computed values. |
| **Workload chart** | `helm/azure-workload-identity-webhook/` | Target cluster (via Flux) | Wraps the **unmodified** upstream chart as a Helm dependency (alias `upstream`) and adds GS-specific extras templates. |

### Value flow

1. Bundle `values.yaml` contains GS image overrides in split `registry`+`repository` format
2. `giantswarm.combineImage` helper merges them into a single `repository` path (e.g., `gsoci.azurecr.io/giantswarm/webhook`)
3. `giantswarm.workloadValues` routes upstream values under `upstream:` key and extras at top level
4. ConfigMap carries the computed values to the Flux HelmRelease
5. Workload chart receives values and passes `upstream:` to the unmodified subchart

### Why this pattern

- **Unmodified upstream** — version bump is just `helm dependency update`
- **Separation of concerns** — Flux orchestration on MC, app on target cluster
- **Single App CR** — install bundle once on MC

## Installation

Create an App CR on the management cluster:

```yaml
apiVersion: application.giantswarm.io/v1alpha1
kind: App
metadata:
  name: azure-workload-identity-webhook-bundle
  namespace: org-example
spec:
  catalog: control-plane-catalog
  name: azure-workload-identity-webhook-bundle
  namespace: org-example
  version: ""
  config:
    configMap:
      name: azure-workload-identity-webhook-bundle-values
      namespace: org-example
```

## Testing

```bash
# Lint both charts
helm lint helm/azure-workload-identity-webhook/
helm lint helm/azure-workload-identity-webhook-bundle/

# Template workload chart
helm template test helm/azure-workload-identity-webhook/ --set upstream.azureTenantID=test

# Update upstream dependency
cd helm/azure-workload-identity-webhook/ && helm dependency update
```

## Credit

* [Azure Workload Identity](https://github.com/Azure/azure-workload-identity)
