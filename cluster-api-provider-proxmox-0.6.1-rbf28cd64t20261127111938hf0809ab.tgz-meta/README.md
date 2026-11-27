[![CircleCI](https://circleci.com/gh/giantswarm/cluster-api-provider-proxmox-app.svg?style=shield)](https://circleci.com/gh/giantswarm/cluster-api-provider-proxmox-app)

# cluster-api-provider-proxmox-app

[Cluster API Provider proxmox}(https://github.com/ionos-cloud/cluster-api-provider-proxmox) - packaged as a Giant Swarm app.

This repository is primary used to import the upstream Cluster API Provider proxmox manifests into Giant Swarm's own app catalog.

Content of the `/helm` directory will be bundled, released and pushed to the `app-catalog` via [`architect`](https://github.com/giantswarm/architect). This happens automatically and is done by [this](.circleci/config.yml) `circleCI` configuration.

> To keep it quite easy to update the manifest from upstream, we don't change the fetched manifests directly. All Giant Swarm specific adjustments get applied via `kustomize`.

## Usage

How to work within this repository?

This repository uses Vendir to sync upstream updates. To update the repo, set the upstream chart version in `vendir.yml`, then run `sync/sync.sh`.

This is run automatically whenever Renovate opens a PR which bumps the version in `vendir.yml`

# Useful links

* [proxmox doc](https://github.com/ionos-cloud/cluster-api-provider-proxmox/tree/main/docs)
