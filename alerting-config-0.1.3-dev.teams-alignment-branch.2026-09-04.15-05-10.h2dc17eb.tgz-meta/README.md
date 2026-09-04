# alerting-config

> ⚠️ **SECURITY NOTICE**: This repository contains customer-sensitive information including team names, PagerDuty tokens and Slack credentials. This repository must remain private.

Giant Swarm's Alertmanager configuration, shipped as an app and deployed per installation.

The chart renders a Kubernetes Secret labelled `observability.giantswarm.io/kind: alertmanager-config`, which [observability-operator](https://github.com/giantswarm/observability-operator) picks up and pushes to Mimir Alertmanager. See [docs/alertmanager.md](https://github.com/giantswarm/observability-operator/blob/main/docs/alertmanager.md) for the contract.

This configuration used to live in the operator's own chart. It moved here so the operator carries no Giant Swarm alerting specifics and any user can supply their own routing.

## Configuration

Non-secret values come from konfigure via `shared-configs`. Secrets are per-installation, SOPS-encrypted into each `<customer>-configs` repository.

## Installing

Deployed as an App CR on the management cluster, ordered after `observability-operator` so the operator's validating webhook is serving when the Secret is created.
