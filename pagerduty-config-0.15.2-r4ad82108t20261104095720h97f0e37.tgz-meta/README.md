# PagerDuty Configuration

> ⚠️ **SECURITY NOTICE**: This repository contains customer-sensitive information including team member details, email addresses, and PagerDuty configuration data. This repository must remain private.

PagerDuty configuration managed via [Crossplane](https://crossplane.io/) and deployed as a Helm chart on the **gazelle** cluster. [Flux](https://fluxcd.io/) applies changes automatically when PRs merge to main.

## Overview

This repo manages most PagerDuty resources — schedules, escalation policies, services, and integrations — for routing alerts from Alertmanager instances to the correct on-call teams.
Note: users are automagically managed via SCIM.

See [ARCHITECTURE.md](ARCHITECTURE.md) for a full technical overview.

## How it works

1. Configuration lives in [`helm/pagerduty-config/values.yaml`](helm/pagerduty-config/values.yaml)
2. Open a PR with your change — CI runs `helm lint` and `helm template`
3. On release, Flux picks up the change and applies it to the gazelle cluster
4. Crossplane reconciles the PagerDuty resources


## Common operations

See [QUICKSTART.md](QUICKSTART.md) for step-by-step guides:

### Get integration keys

```bash
# All keys (JSON)
./scripts/get-integration-keys.sh --format json

# One team across all clusters
./scripts/get-integration-keys.sh --team atlas

# One cluster across all teams
./scripts/get-integration-keys.sh --cluster cluster-agama

# One team, one cluster
./scripts/get-integration-keys.sh --team atlas --cluster cluster-agama
```

### Urgent email service

The urgent service pages all team schedules simultaneously when an email arrives at the Google Group address. The escalation policy is generated automatically from `teams:` in `values.yaml`. Manage allowed senders via `urgent.allowedDomains`.

See [SETUP_INSTRUCTIONS.md](SETUP_INSTRUCTIONS.md) for the full setup guide.

## Testing

```bash
# Render all templates locally
make helm-template

# Lint the chart
make helm-lint
```

See [scripts/README.md](scripts/README.md) for test alert scripts that let you trigger and resolve test incidents via Alertmanager or directly via the PagerDuty Events API.

## Architecture

See [ARCHITECTURE.md](ARCHITECTURE.md).
