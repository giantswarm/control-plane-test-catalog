[![CircleCI](https://dl.circleci.com/status-badge/img/gh/giantswarm/kyverno-crds/tree/main.svg?style=svg)](https://dl.circleci.com/status-badge/redirect/gh/giantswarm/kyverno-crds/tree/main)

# kyverno-crds chart

Giant Swarm offers a kyverno-crds App which can be installed in workload clusters.

The `kyverno-crds` App includes the necessary CRDs to run Kyverno in workload and managament clusters.

## CRDs included in the chart

The chart includes CRDs from the `kyverno.io`, `reports.kyverno.io` and `wgpolicyk8s.io` groups.

```
.
├── kyverno.io
│   ├── kyverno.io_admissionreports.yaml
│   ├── kyverno.io_backgroundscanreports.yaml
│   ├── kyverno.io_cleanuppolicies.yaml
│   ├── kyverno.io_clusteradmissionreports.yaml
│   ├── kyverno.io_clusterbackgroundscanreports.yaml
│   ├── kyverno.io_clustercleanuppolicies.yaml
│   ├── kyverno.io_clusterpolicies.yaml
│   ├── kyverno.io_globalcontextentries.yaml
│   ├── kyverno.io_policies.yaml
│   ├── kyverno.io_policyexceptions.yaml
│   └── kyverno.io_updaterequests.yaml
├── reports.kyverno.io
│   ├── reports.kyverno.io_clusterephemeralreports.yaml
│   └── reports.kyverno.io_ephemeralreports.yaml
└── wgpolicyk8s.io
    ├── wgpolicyk8s.io_clusterpolicyreports.yaml
    └── wgpolicyk8s.io_policyreports.yaml
```

## Configuration

You can configure the CRDs you want to install with the following values:

```yaml
crds:
    groups:

    # -- Install CRDs in group `kyverno.io`
    # -- This field can be overwritten by setting crds.labels in the parent chart
    kyverno:
        admissionreports: true
        backgroundscanreports: true
        cleanuppolicies: true
        clusteradmissionreports: true
        clusterbackgroundscanreports: true
        clustercleanuppolicies: true
        clusterpolicies: true
        globalcontextentries: true
        policies: true
        policyexceptions: true
        updaterequests: true

    # -- Install CRDs in group `reports.kyverno.io`
    # -- This field can be overwritten by setting crds.labels in the parent chart
    reports:
        clusterephemeralreports: true
        ephemeralreports: true

    # -- Install CRDs in group `wgpolicyk8s.io`
    # -- This field can be overwritten by setting crds.labels in the parent chart
    wgpolicyk8s:
        clusterpolicyreports: true
        policyreports: true
```

## Credit

- [`kyverno`][kyverno-upstream]

[kyverno-upstream]: https://github.com/kyverno/kyverno
