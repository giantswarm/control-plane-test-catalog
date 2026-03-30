# PagerDuty Configuration

> ⚠️ **SECURITY NOTICE**: This repository contains customer-sensitive information including team member details, email addresses, and PagerDuty configuration data. This repository must remain private and should not be made public under any circumstances.

Terraform configuration for managing PagerDuty services, schedules, and escalation policies for multi-team alerting from Alertmanager instances.

## Overview

This repository configures PagerDuty to route alerts from multiple Alertmanager instances to appropriate on-call teams.

## Quick Start

### Prerequisites

- Terraform >= 1.13.0 (see [terraform installation documentation](https://developer.hashicorp.com/terraform/install))
- PagerDuty account with API access
- PagerDuty API token (in pagerduty `My Profile` => `User Settings`)

### Initial Setup

1. Clone the repository

2. Export your pagerduty token as environment variable
   ```bash
   export TF_VAR_pagerduty_token="your-api-token-here"
   `````

3. Initialize Terraform:
   ```bash
   terraform init
   ```

4. Review the plan:
   ```bash
   terraform plan
   ```

5. Apply the configuration:
   ```bash
   terraform apply
   ```

### Notes

⚠️ This repo includes the *terraform state* file, `terraform.tfstate`. This file is updated each time a `terraform apply` command performs actual changes to the state. Please always run the `terraform apply` command in the main branch and create a pull request to update the `terraform.tfstate` file afterwards.
It also includes our configuration, `values.auto.tfvars`, even though it should probably be in another config repo.

## Alertmanager Integration

See [SETUP_INSTRUCTIONS.md](SETUP_INSTRUCTIONS.md) for detailed steps on integrating Alertmanager instances with PagerDuty services.

## Urgent Email Integration

This repository includes a special **urgent email service** that provides emergency alerting to all teams simultaneously.

### Key Features

- **Email-to-PagerDuty Integration**: Forwards emails from Google Groups to PagerDuty incidents
- **Multi-Team Paging**: Automatically pages all team schedules with staggered escalation (1min, 2min delays)
- **Smart Filtering**: Suppresses email replies, unauthorized domains, and auto-replies
- **Service Orchestration**: Uses modern PagerDuty service orchestration instead of deprecated event rules

### Usage

1. **Integration Details**: see [SETUP_INSTRUCTIONS.md](SETUP_INSTRUCTIONS.md)
2. **Email Address**: The PagerDuty email address is available via `terraform output urgent_email_address`
3. **Google Group Setup**: Configure your Google Group to forward emails to the PagerDuty email address
4. **Testing**: Send test emails to verify filtering and escalation behavior

### Authorized Domains

The urgent service accepts emails from authorized domains configured in the `allowed_urgent_domains` list in `urgent.tf`:
- @giantswarm.io (primary)
- Various customer domains (@adidas-group.com, @telekom.de, @vodafone.com, etc.)
- Partner and vendor domains

To add new domains, edit the `allowed_urgent_domains` list in `urgent.tf`. Use `terraform output urgent_allowed_domains` to see the current list.

Emails from unauthorized domains, replies (Re:), forwards (Fwd:), and auto-replies are automatically suppressed.

> **⚠️ PagerDuty Plan Limitation**: Email filtering and suppression features require a higher PagerDuty tier plan. Currently, these rules are configured but not active - all emails will create incidents until the account is upgraded to a plan that supports Alert Suppression.

## Creating a new installation (Management Cluster)

- Add your installation to the `alertmanager_instances` list in `values.auto.tfvars`
- Apply your new terraform configuration
- Run `terraform output -json integration_keys_by_instance | jq '."cluster-graveler"'` and keep the output for later use
-
- In the customer's configs repository, edit `installations/[installation]/apps/observability-operator/secret-values.yaml.patch` (warning: you have to decrypt / encrypt this file)
- Add this under `.alerting`:
    ```
    cronitorHeartbeatPingKey: [cronitor_ping]
    cronitorHeartbeatManagementKey: [cronitor_management]
    teams:
        - name: atlas
          pagerdutyToken: [atlas_key]
        - name: cabbage
          pagerdutyToken: [cabbage_key]
        - name: honeybadger
          pagerdutyToken: [honeybadger_key]
        - name: phoenix
          pagerdutyToken: [phoenix_key]
        - name: rocket
          pagerdutyToken: [rocket_key]
        - name: se
          pagerdutyToken: [se_key]
        - name: shield
          pagerdutyToken: [honeybadger_key]
        - name: tenet
          pagerdutyToken: [phoenix_key]
    ```
- Replace the `[team_key]` placeholders with the keys from the `terraform output` command
- Concerning the `[cronitor_xxx]` placeholders, follow the [cronitor documentation](https://intranet.giantswarm.io/docs/support-and-ops/installation-setup-guide/heartbeat-monitoring-setup/).

## Architecture

See [ARCHITECTURE.md](ARCHITECTURE.md) for detailed architecture overview.
