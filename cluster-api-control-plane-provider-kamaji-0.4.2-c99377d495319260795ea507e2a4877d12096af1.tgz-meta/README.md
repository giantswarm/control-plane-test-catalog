[![CircleCI](https://dl.circleci.com/status-badge/img/gh/giantswarm/cluster-api-control-plane-provider-kamaji-app/tree/main.svg?style=svg)](https://dl.circleci.com/status-badge/redirect/gh/giantswarm/cluster-api-control-plane-provider-kamaji-app/tree/main)
[![OpenSSF Scorecard](https://api.securityscorecards.dev/projects/github.com/giantswarm/cluster-api-control-plane-provider-kamaji-app/badge)](https://securityscorecards.dev/viewer/?uri=github.com/giantswarm/cluster-api-control-plane-provider-kamaji-app)

# cluster-api-control-plane-provider-kamaji-app chart

This repository contains a Helm chart for the cluster-api-control-plane-provider-kamaji-app App by Giant Swarm. The Kamaji controller it deploys is a Cluster API Control Plane Provider for Kamaji, which enables the hosting of Control Planes for Workload Clusters inside Giant Swarm Management Clusters.

## Updating

> [!WARNING]
> DO NOT merge Renovate PRs without following the instructions below.
>
> Failure to do so will not update the chart itself.

### Renovate updates

1. Check out the PR created by Renovate.
2. Run `sync/sync.sh` which will retrieve the asset from the upstream release, customize it for Giant Swarm, and update the ch  art files.

### Manual updates

1. Update the `tag` field in [`vendir.yml`](vendir.yml) to the desired upstream version.
2. Run `sync/sync.sh` which will retrieve the asset from the upstream release, customize it for Giant Swarm, and update the chart files.

## Credit

- https://github.com/clastix/cluster-api-control-plane-provider-kamaji
