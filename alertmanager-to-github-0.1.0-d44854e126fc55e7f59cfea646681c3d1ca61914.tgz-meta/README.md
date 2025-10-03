[![CircleCI](https://dl.circleci.com/status-badge/img/gh/giantswarm/alertmanager-to-github/tree/main.svg?style=svg)](https://dl.circleci.com/status-badge/redirect/gh/giantswarm/alertmanager-to-github/tree/main)

[Guide about how to manage an app on Giant Swarm](https://handbook.giantswarm.io/docs/dev-and-releng/app-developer-processes/adding_app_to_appcatalog/)

# alertmanager-to-github App

Giant Swarm app for integrating Prometheus Alertmanager with GitHub Issues, based on [pfnet-research/alertmanager-to-github](https://github.com/pfnet-research/alertmanager-to-github).

## What is this app?

This app automatically creates and manages GitHub issues based on Prometheus alerts. When alerts with `severity: ticket` are fired, the app creates corresponding GitHub issues, updates them with alert changes, and closes them when alerts resolve.

## Key Features

- **Automatic issue creation** from Prometheus alerts
- **Issue lifecycle management** (creation, updates, closure)
- **Deduplication** using Alertmanager grouping keys
- **Customizable templates** for issue titles and descriptions
- **GitHub App authentication** with minimal required permissions
- **Webhook authentication** for secure Alertmanager integration

## Prerequisites

- Kubernetes cluster with Alertmanager deployed
- GitHub organization with admin access
- GitHub App configured with appropriate permissions

## GitHub App Setup

As per [github documentation](https://docs.github.com/en/apps/creating-github-apps/about-creating-github-apps/about-creating-github-apps):
> A GitHub App is a type of integration that you can build to interact with and extend the functionality of GitHub. 

You need to create such an integration so that alertmanager-to-github can communicate to github.

### 1. Create GitHub App

In your GitHub organization:

1. Go to `Settings → Developer settings → GitHub Apps`
2. Click "New GitHub App"
3. Configure basic information:
   - **Name**: `[Organization] Alertmanager Integration`
   - **Homepage URL**: Your organization's homepage
   - **Webhook URL**: Leave empty (not used)
   - **Webhook secret**: Leave empty

### 2. Set Permissions

Configure these permissions:

**Repository permissions:**

- `Issues`: Read & write
- `Metadata`: Read

**Organization permissions:** None  
**Account permissions:** None

### 3. Generate Private Key

1. In the GitHub App settings, scroll to "Private keys"
2. Click "Generate a private key"
3. Download the `.pem` file and store it securely

### 4. Install the App

1. Go to "Install App" in the GitHub App settings
2. Install on your organization
3. Select repositories where issues should be created
4. Note the **Installation ID** from the URL (e.g., `https://github.com/settings/installations/12345678`)

### 5. Collect Required Information

You'll need these values for configuration:
- **App ID**: Found in GitHub App settings (General tab)
- **Installation ID**: From the installation URL
- **Private Key**: Content of the downloaded `.pem` file
- **Repository**: Format `owner/repository` where issues will be created

## Alert Configuration

### Required Alert Labels

Alerts must include `severity: ticket` to trigger GitHub issue creation:

```yaml
groups:
- name: example.rules
  rules:
  - alert: ServiceDown
    expr: up{job="my-service"} == 0
    for: 5m
    labels:
      severity: ticket
      team: platform
      component: my-service
    annotations:
      summary: "Service {{ $labels.job }} is down"
      description: "Service has been down for more than 5 minutes"
      runbook_url: "https://runbooks.example.com/service-down"
```

### Alertmanager Routing

Configure Alertmanager to route ticketed alerts to this app:

```yaml
route:
  group_by: ['alertname', 'cluster_id', 'installation']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 12h
  receiver: 'default'
  routes:
  - match:
      severity: ticket
    receiver: 'github-integration'

receivers:
- name: 'github-integration'
  webhook_configs:
  - url: 'http://alertmanager-to-github.monitoring.svc.cluster.local:8080/webhook'
    send_resolved: true
    http_config:
      basic_auth:
        username: 'webhook'
        password: 'your-webhook-password'
```

## Installing

Install this app using Giant Swarm's App Platform:

### Using GitOps

Add to your GitOps repository:

```yaml
apiVersion: application.giantswarm.io/v1alpha1
kind: App
metadata:
  name: alertmanager-to-github
  namespace: monitoring
spec:
  catalog: control-plane-catalog
  name: alertmanager-to-github-app
  namespace: monitoring
  version: "1.2.0"
  config:
    configMap:
      name: alertmanager-to-github-config
      namespace: monitoring
  userConfig:
    secret:
      name: alertmanager-to-github-secrets
      namespace: monitoring
```

### Using kubectl

Create the App resource directly:

```bash
kubectl apply -f - <<EOF
apiVersion: application.giantswarm.io/v1alpha1
kind: App
metadata:
  name: alertmanager-to-github
  namespace: monitoring
spec:
  catalog: control-plane-catalog
  name: alertmanager-to-github-app
  namespace: monitoring
  version: "1.2.0"
  config:
    configMap:
      name: alertmanager-to-github-config
      namespace: monitoring
  userConfig:
    secret:
      name: alertmanager-to-github-secrets
      namespace: monitoring
EOF
```

## Configuration

### ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-to-github-config
  namespace: monitoring
data:
  config.yaml: |
    # GitHub repository for issues (required)
    repository: "myorg/alerts"

    # Issue template configuration
    template:
      title: "🚨 {{ .GroupLabels.alertname }}: {{ .GroupLabels.cluster_id | default \"unknown\" }}"
      body: |
        ## Alert Information

        | Field | Value |
        |-------|--------|
        | **Alert** | {{ .GroupLabels.alertname }} |
        | **Cluster** | {{ .GroupLabels.cluster_id | default "N/A" }} |
        | **Team** | {{ .GroupLabels.team | default "Unknown" }} |
        | **Component** | {{ .GroupLabels.component | default "N/A" }} |
        | **Severity** | {{ .GroupLabels.severity }} |

        ## Alert Details

        {{ range .Alerts }}
        ### {{ .Annotations.summary | default "No summary available" }}

        {{ .Annotations.description | default "No description available" }}

        {{- if .Annotations.runbook_url }}
        **📖 Runbook**: {{ .Annotations.runbook_url }}
        {{- end }}

        **⏰ Started**: {{ .StartsAt.Format "2006-01-02 15:04:05 UTC" }}
        {{- if .EndsAt }}
        **✅ Ended**: {{ .EndsAt.Format "2006-01-02 15:04:05 UTC" }}
        {{- end }}

        ---
        {{ end }}

        *This issue was automatically created by the alerting system.*

    # Webhook authentication (optional but recommended)
    auth:
      username: "webhook"
      password: "webhook-secret"

    # Logging configuration
    log:
      level: "info"
      format: "json"

    # Server configuration
    server:
      port: 8080
```

### Secrets

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-to-github-secrets
  namespace: monitoring
type: Opaque
stringData:
  github-app-id: "YOUR_GITHUB_APP_ID"
  github-installation-id: "YOUR_INSTALLATION_ID"
  github-private-key: |
    [Your GitHub App Private Key Content - paste the entire content from your downloaded .pem file here]
  webhook-password: "YOUR_SECURE_WEBHOOK_PASSWORD"
```

### values.yaml

For Helm-based configuration:

```yaml
# GitHub configuration
github:
  appId: "YOUR_GITHUB_APP_ID"
  installationId: "YOUR_INSTALLATION_ID"
  repository: "your-org/your-repo"

# Issue template customization
template:
  title: "🚨 {{ .GroupLabels.alertname }}: {{ .GroupLabels.cluster_id | default \"unknown\" }}"
  body: |
    ## Alert: {{ .GroupLabels.alertname }}

    {{ range .Alerts }}
    **Summary**: {{ .Annotations.summary }}
    **Description**: {{ .Annotations.description }}
    {{ end }}

# Webhook authentication
webhook:
  auth:
    enabled: true
    username: "webhook"

# Logging
logging:
  level: "info"
  format: "json"

# Resources
resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 50m
    memory: 64Mi
```

## Template Customization

The issue templates use Go template syntax with access to Alertmanager webhook data:

### Available Variables

- `.GroupLabels` - Labels used for grouping (map[string]string)
- `.CommonLabels` - Labels common to all alerts in the group (map[string]string)
- `.CommonAnnotations` - Annotations common to all alerts (map[string]string)
- `.ExternalURL` - Alertmanager external URL
- `.Alerts` - Array of individual alerts

### Alert Object Fields

Each alert in `.Alerts` contains:
- `.Status` - Alert status (firing/resolved)
- `.Labels` - Alert labels (map[string]string)
- `.Annotations` - Alert annotations (map[string]string)
- `.StartsAt` - Alert start time
- `.EndsAt` - Alert end time (for resolved alerts)
- `.GeneratorURL` - URL to the generating query

### Template Examples

**Simple template:**
```yaml
template:
  title: "Alert: {{ .GroupLabels.alertname }}"
  body: |
    {{ range .Alerts }}
    - {{ .Annotations.summary }}
    {{ end }}
```

**Detailed template with conditional logic:**
```yaml
template:
  title: "🚨 {{ .GroupLabels.alertname }}{{if .GroupLabels.cluster_id}} ({{ .GroupLabels.cluster_id }}){{end}}"
  body: |
    ## Status: {{ if eq .Status "firing" }}🔥 FIRING{{ else }}✅ RESOLVED{{ end }}

    {{- if .GroupLabels.team }}
    **Team**: {{ .GroupLabels.team }}
    {{- end }}
    {{- if .GroupLabels.component }}
    **Component**: {{ .GroupLabels.component }}
    {{- end }}

    ## Alerts ({{ len .Alerts }})

    {{ range .Alerts }}
    ### {{ .Annotations.summary | default "No summary" }}

    {{ .Annotations.description | default "No description" }}

    {{- if .Annotations.runbook_url }}
    [📖 Runbook]({{ .Annotations.runbook_url }})
    {{- end }}

    ---
    {{ end }}
```

## Credit

- **Upstream Project**: [pfnet-research/alertmanager-to-github](https://github.com/pfnet-research/alertmanager-to-github)
- **Maintainer**: Giant Swarm Team Atlas
