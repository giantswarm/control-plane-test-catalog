# PagerDuty Configuration

> ⚠️ **SECURITY NOTICE**: This repository contains customer-sensitive information including team member details, email addresses, and PagerDuty configuration data. This repository must remain private.

PagerDuty configuration managed via [Crossplane](https://crossplane.io/) and deployed as a Helm chart on the **gazelle** cluster. [Flux](https://fluxcd.io/) applies changes automatically when PRs merge to main.

## Overview

This repo manages all PagerDuty resources — users, schedules, escalation policies, services, and integrations — for routing alerts from Alertmanager instances to the correct on-call teams.

See [ARCHITECTURE.md](ARCHITECTURE.md) for a full technical overview.

## How it works

1. Configuration lives in [`helm/pagerduty-config/values.yaml`](helm/pagerduty-config/values.yaml)
2. Open a PR with your change — CI runs `helm lint` and `helm template`
3. On release, Flux picks up the change and applies it to the gazelle cluster
4. Crossplane reconciles the PagerDuty resources

To apply manually:
```bash
helm upgrade --install pagerduty-config ./helm/pagerduty-config/ --namespace crossplane
```

## Common operations

Most changes follow a two-phase pattern: **create or find in PagerDuty → add the ID to `values.yaml`**.

### Add a user

Users are provisioned automatically via SCIM. Once the account exists in PagerDuty:

1. Find the user's PagerDuty ID in their profile URL (`/users/PXXXXXX`)
2. Add to `users:` in `values.yaml`:
   ```yaml
   users:
     alice:
       email: alice@giantswarm.io
       pdId: PXXXXXX
   ```
3. Open a PR

### Add a team and on-call schedule

1. Create the schedule and escalation policy in the PagerDuty UI
2. Copy their IDs from the browser URL
3. Add to `teams:` in `values.yaml`:
   ```yaml
   teams:
     newteam:
       policyId: PXXXXXX      # escalation policy ID
       area: platform          # area schedule for OOBH coverage (optional)
       schedules:
         newteam:
           scheduleId: PXXXXXX # primary schedule ID
           catchupId: PXXXXXX  # catchup schedule ID
           users: [alice, bob]
           restrictions: businessHours
   ```
4. Add the new schedule ID to `urgent.scheduleGroups` in `values.yaml` so the team is included in the urgent escalation policy (max 5 IDs per group — add a new group if needed)
5. Open a PR

### Add an Alertmanager instance (cluster)

No PagerDuty ID needed — Crossplane creates a fresh integration key for each team × cluster pair.

1. Add the cluster name to `alertmanagerInstances:` in `values.yaml`
2. Open a PR
3. Once Flux applies, retrieve the new integration keys:
   ```bash
   ./scripts/get-integration-keys.sh --cluster cluster-newcluster
   ```
4. Update the SOPS-encrypted Alertmanager config in the cluster's GitOps repo

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

The urgent service pages all team schedules simultaneously when an email arrives at the Google Group address. Manage it via `values.yaml`:

- `urgent.allowedDomains` — domains permitted to trigger incidents; others are suppressed
- `urgent.scheduleGroups` — groups of schedule IDs (max 5 per group, PagerDuty limit)

See [SETUP_INSTRUCTIONS.md](SETUP_INSTRUCTIONS.md) for the full setup guide.

## Testing

See [scripts/README.md](scripts/README.md) for test alert scripts that let you trigger and resolve test incidents via Alertmanager or directly via the PagerDuty Events API.

## Architecture

See [ARCHITECTURE.md](ARCHITECTURE.md).
