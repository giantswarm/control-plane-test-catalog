[![CircleCI](https://dl.circleci.com/status-badge/img/gh/giantswarm/alloy-gateway-app/tree/main.svg?style=svg)](https://dl.circleci.com/status-badge/redirect/gh/giantswarm/alloy-gateway-app/tree/main)

# alloy-gateway-app

The **alloy-gateway** is a secure ingestion gateway that enables customers to send observability data (logs, traces, and metrics) from external sources to Giant Swarm's Observability Platform. It extends beyond cluster-managed data collection to support external systems, SaaS applications, and hybrid infrastructure scenarios.

## Overview

This component serves as the secure entry point for external observability data into Giant Swarm's platform. It provides authenticated and multi-tenant data ingestion capabilities while maintaining the same security standards as internal platform components.

### Key Features

- **🔒 Secure Authentication**: OIDC-based authentication with multi-tenant access control
- **📊 Multi-Protocol Support**: 
  - Logs via Loki-compatible API
  - Traces via OpenTelemetry Protocol (OTLP) HTTP endpoint
  - Metrics support (planned)
- **🌐 External Integration**: Purpose-built for data from sources outside Giant Swarm managed clusters
- **⚡ High Performance**: Built on Grafana Alloy for efficient data processing and forwarding
- **🏢 Multi-Tenant**: Tenant isolation ensures secure data separation

## Architecture

The alloy-gateway operates as part of the broader [Observability Platform API](https://docs.giantswarm.io/overview/observability/data-management/data-import-export/) ecosystem:

```
External Sources → HTTPS/TLS → alloy-gateway → Internal Backends
     ↓                           ↓                    ↓
• SaaS Apps              • OIDC Auth           • Loki (logs)
• Legacy Systems         • Multi-tenant        • Tempo (traces)  
• Custom Apps            • Data validation     • Mimir (metrics)
```

**Technical Architecture:**
1. **Ingress Layer**: NGINX ingress with external authentication via Dex (or custom OIDC provider)
2. **Gateway Service**: Grafana Alloy with dual endpoints:
   - `loki.source.api` listening on port 3100 (logs)
   - `otelcol.receiver.otlp` listening on port 4318 (traces)
3. **Internal Forwarding**: 
   - Logs → `loki-gateway.loki.svc/loki/api/v1/push`
   - Traces → `tempo-gateway.tempo.svc` (OTLP HTTP)
4. **Multi-tenancy**: Enforced via mandatory `X-Scope-OrgID` header validation and preservation

**Authentication Flow:**
1. Client sends data with Bearer token (`Authorization` header)
2. NGINX ingress validates OIDC token with Dex (default: `https://dex.<codename>.<base>/userinfo`)
3. Tenant access verified via required `X-Scope-OrgId` header
4. Data forwarded to Alloy gateway, then to internal Loki

The gateway is exposed via ingresses managed by the [observability-platform-api](https://github.com/giantswarm/observability-platform-api) component.

## API Endpoints

All observability endpoints are accessible at:
`https://observability.<codename>.<base-domain>` 

Where:
- `<codename>` is your Giant Swarm installation's codename
- `<base-domain>` is your installation's base domain

### ✅ Data Ingestion (Push)

| Data Type | Endpoint | Purpose | Status |
|-----------|----------|---------|---------|
| **Logs** | `/loki/api/v1/push` | Send logs to the platform | Production |
| **Traces** | `/v1/traces` | Send traces to the platform | Production |

### ✅ Data Querying (Pull)

| Data Type | Endpoints | Purpose | Status |
|-----------|-----------|---------|---------|
| **Logs** | `/loki/api/v1/query*` | Query and retrieve logs | Production |
| **Metrics** | `/prometheus/api/v1/*` | Query and retrieve metrics | Production |
| **Traces** | `/tempo/api/*` | Query and retrieve traces | Production |

**Available query endpoints:**
- **Logs**: `/loki/api/v1/query`, `/loki/api/v1/query_range`, `/loki/api/v1/labels`, `/loki/api/v1/series`, etc.
- **Metrics**: `/prometheus/api/v1/query`, `/prometheus/api/v1/query_range`, `/prometheus/api/v1/labels`, `/prometheus/api/v1/rules`, etc.
- **Traces**: `/tempo/api/search`, `/tempo/api/traces`, `/tempo/api/v2/search`, `/tempo/api/v2/traces`, etc.

### 🚧 In Development

| Data Type | Endpoint | Purpose | Status |
|-----------|----------|---------|---------|
| **Metrics Push** | `/prometheus/api/v1/push` | Send metrics to the platform | Planned |

## Getting Started

### Prerequisites

1. **OIDC Configuration**: Work with Giant Swarm to configure your identity provider
2. **Tenant Setup**: Create or identify your target tenant in the platform
3. **Valid Credentials**: Obtain an OIDC bearer token from your identity provider
4. **Network Access**: Ensure connectivity to `https://observability.<codename>.<base-domain>`

### Authentication Requirements

All requests must include:

- **Authorization Header**: `Authorization: Bearer <your-oidc-token>`
- **Tenant Header**: `X-Scope-OrgId: <your-tenant-name>`
- **Content-Type**: `application/json` (for JSON payloads)

### Usage Examples

#### Sending Logs

```bash
curl -X POST \
     -H "Authorization: Bearer $OIDC_TOKEN" \
     -H "X-Scope-OrgId: your-tenant" \
     -H "Content-Type: application/json" \
     -d @logs-payload.json \
     "https://observability.<codename>.<base-domain>/loki/api/v1/push"
```

**Example payload** (`logs-payload.json`):
```json
{
  "streams": [
    {
      "stream": {
        "job": "external-service",
        "level": "info",
        "environment": "production"
      },
      "values": [
        ["1640995200000000000", "Application started successfully"],
        ["1640995201000000000", "Processing user request"]
      ]
    }
  ]
}
```

#### Sending Traces

```bash
curl -X POST \
     -H "Authorization: Bearer $OIDC_TOKEN" \
     -H "X-Scope-OrgId: your-tenant" \
     -H "Content-Type: application/json" \
     -d @traces-payload.json \
     "https://observability.<codename>.<base-domain>/v1/traces"
```

**Example OTLP traces payload** (`traces-payload.json`):
```json
{
  "resourceSpans": [
    {
      "resource": {
        "attributes": [
          {"key": "service.name", "value": {"stringValue": "external-service"}},
          {"key": "service.version", "value": {"stringValue": "1.0.0"}}
        ]
      },
      "scopeSpans": [
        {
          "scope": {"name": "external-tracer"},
          "spans": [
            {
              "traceId": "5B8EFFF798038103D269B633813FC60C",
              "spanId": "EEE19B7EC3C1B174",
              "name": "external-operation",
              "startTimeUnixNano": "1640995200000000000",
              "endTimeUnixNano": "1640995201000000000",
              "attributes": [
                {"key": "operation.type", "value": {"stringValue": "http_request"}}
              ]
            }
          ]
        }
      ]
    }
  ]
}
```

For detailed payload formats and advanced usage, see the [Data Import/Export Guide](https://docs.giantswarm.io/overview/observability/data-management/data-import-export/).

## Security & Compliance

The alloy-gateway maintains the same security standards as Giant Swarm's internal observability platform:

- **🔐 End-to-End Encryption**: All data transfer uses TLS 1.2+ encryption
- **🎯 Identity-Based Access**: Integration with your organization's OIDC provider
- **🏢 Multi-Tenant Isolation**: Secure data separation at the tenant level
- **📋 Audit Trails**: All data access and ingestion requests are logged
- **🚫 Zero Trust**: No data access without valid authentication and tenant authorization

## Integration Options

### Supported Data Shippers

**For Logs:**
- **Grafana Alloy**: Native support with Loki remote write
- **Vector**: Loki sink with authentication headers
- **Fluent Bit**: Loki output plugin with custom headers
- **Fluentd**: Loki plugin with authentication configuration
- **Custom Applications**: Any HTTP client supporting bearer token authentication

**For Traces:**
- **OpenTelemetry Collector**: OTLP HTTP exporter
- **Grafana Alloy**: OTLP HTTP exporter configuration
- **Jaeger Agent**: OTLP HTTP endpoint support
- **Custom Applications**: OpenTelemetry SDKs with OTLP HTTP exporter

### Example Integration Configurations

<details>
<summary>Grafana Alloy - Logs Configuration</summary>

```alloy
loki.write "external" {
  endpoint {
    url = "https://observability.<codename>.<base-domain>/loki/api/v1/push"
    headers = {
      "Authorization" = "Bearer " + env("OIDC_TOKEN"),
      "X-Scope-OrgId" = "your-tenant",
    }
  }
}
```
</details>

<details>
<summary>Grafana Alloy - Traces Configuration</summary>

```alloy
otelcol.exporter.otlphttp "external" {
  client {
    endpoint = "https://observability.<codename>.<base-domain>"
    headers = {
      "Authorization" = "Bearer " + env("OIDC_TOKEN"),
      "X-Scope-OrgId" = "your-tenant",
    }
  }
}

otelcol.processor.batch "default" {
  output {
    traces = [otelcol.exporter.otlphttp.external.input]
  }
}
```
</details>

<details>
<summary>OpenTelemetry Collector Configuration</summary>

```yaml
exporters:
  otlphttp:
    endpoint: https://observability.<codename>.<base-domain>
    headers:
      authorization: "Bearer ${OIDC_TOKEN}"
      x-scope-orgid: "your-tenant"

service:
  pipelines:
    traces:
      exporters: [otlphttp]
```
</details>

<details>
<summary>Vector Configuration</summary>

```toml
[sinks.loki]
type = "loki"
inputs = ["your_source"]
endpoint = "https://observability.<codename>.<base-domain>"
auth.strategy = "bearer"
auth.token = "${OIDC_TOKEN}"
headers.X-Scope-OrgId = "your-tenant"
```
</details>

## Deployment Information

- **Target Environment**: Management clusters only
- **Installation**: Automatically deployed by Giant Swarm platform management
- **Dependencies**: Requires [observability-platform-api](https://github.com/giantswarm/observability-platform-api) for ingress configuration
- **Base Technology**: Built on [Grafana Alloy](https://github.com/giantswarm/alloy-app)

⚠️ **Note**: This application is **not intended for workload clusters** and is specifically designed for management cluster deployment.

## Related Resources

### Components
- [**observability-platform-api**](https://github.com/giantswarm/observability-platform-api) - Ingress layer and API definitions
- [**alloy-app**](https://github.com/giantswarm/alloy-app) - Base Grafana Alloy application

### Documentation  
- [**Data Import/Export Guide**](https://docs.giantswarm.io/overview/observability/data-management/data-import-export/) - Complete API documentation
- [**Multi-Tenancy Setup**](https://docs.giantswarm.io/overview/observability/configuration/multi-tenancy/) - Tenant configuration guide
- [**Authentication Setup**](https://docs.giantswarm.io/overview/observability/data-management/data-import-export/#authentication-and-access-control) - OIDC configuration details

### Support
- [**Roadmap Issue**](https://github.com/giantswarm/roadmap/issues/3568) - Original implementation roadmap
- **Contact**: Reach out via your designated support channels for authentication setup assistance

---

**Need help with setup?** Contact your Giant Swarm account team to configure OIDC authentication and tenant access for your external data sources.
