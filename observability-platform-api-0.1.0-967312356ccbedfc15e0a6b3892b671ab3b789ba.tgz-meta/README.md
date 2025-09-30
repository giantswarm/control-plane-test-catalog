[![CircleCI](https://dl.circleci.com/status-badge/img/gh/giantswarm/observability-platform-api/tree/main.svg?style=svg)](https://dl.circleci.com/status-badge/redirect/gh/giantswarm/observability-platform-api/tree/main)

# observability-platform-api

The **observability-platform-api** provides the ingress layer for Giant Swarm's Observability Platform, enabling secure access to observability services from external sources. It manages multiple ingresses that provide both data ingestion and querying capabilities through a unified API endpoint.

## Overview

This application creates and manages the NGINX ingresses that expose the observability platform's APIs to external users. It works in conjunction with the [alloy-gateway-app](https://github.com/giantswarm/alloy-gateway-app) to provide a complete external data ingestion and querying solution.

### Key Features

- **🔒 Secure Access**: OIDC-based authentication with configurable identity providers
- **📊 Multi-Service Ingress**: Manages ingresses for Loki, Mimir, Tempo, and Gateway services
- **🏢 Multi-Tenant Routing**: Enforces tenant isolation via HTTP headers
- **🌐 External Integration**: Enables access from self-hosted Grafana and external log shippers
- **⚡ High Availability**: Built on NGINX ingress with TLS termination

## Architecture

The observability-platform-api creates multiple ingresses under a single domain:

```
https://observability.<codename>.<base-domain>
├── /loki/api/v1/push          → Gateway Ingress → alloy-gateway (port 3100)
├── /v1/traces                 → Gateway OTLP    → alloy-gateway (port 4318)
├── /loki/api/v1/query*        → Loki Ingress    → loki-gateway (port 80)
├── /prometheus/api/v1/*       → Mimir Ingress   → mimir-gateway (port 80)
└── /tempo/api/*               → Tempo Ingress   → tempo-gateway (port 80)
```

### Ingress Configuration

#### 1. Gateway Ingress (Log Data Ingestion)

- **Purpose**: External log data ingestion
- **Target**: `observability-gateway-alloy:3100` (in `monitoring` namespace)
- **Paths**: `/loki/api/v1/push`
- **Use Case**: External log shippers, custom applications

#### 2. Gateway OTLP Ingress (Trace Data Ingestion)

- **Purpose**: External trace data ingestion via OpenTelemetry Protocol
- **Target**: `observability-gateway-alloy:4318` (in `monitoring` namespace)
- **Paths**: `/v1/traces`
- **Use Case**: OpenTelemetry collectors, tracing-enabled applications

#### 3. Loki Ingress (Log Querying)

- **Purpose**: Log data querying and retrieval
- **Target**: `loki-gateway:80` (in `loki` namespace)
- **Paths**: `/loki/api/v1/query`, `/loki/api/v1/query_range`, `/loki/api/v1/labels`, etc.
- **Use Case**: External Grafana instances, custom dashboards

#### 4. Mimir Ingress (Metrics Querying)

- **Purpose**: Metrics data querying and retrieval
- **Target**: `mimir-gateway:80` (in `mimir` namespace)
- **Paths**: `/prometheus/api/v1/query`, `/prometheus/api/v1/query_range`, `/prometheus/api/v1/labels`, etc.
- **Use Case**: External Grafana instances, monitoring tools

#### 5. Tempo Ingress (Trace Querying)

- **Purpose**: Trace data querying and retrieval
- **Target**: `tempo-gateway:80` (in `tempo` namespace)
- **Paths**: `/tempo/api/search`, `/tempo/api/traces`, `/tempo/api/v2/search`, etc.
- **Use Case**: External Grafana instances, tracing analysis tools

## Authentication & Security

### OIDC Integration

All ingresses are protected by external authentication:

```yaml
nginx.ingress.kubernetes.io/auth-url: https://dex.<codename>.<base>/userinfo
```

**Custom OIDC Provider:**

```yaml
observabilityPlatformApi:
  authUrl: "https://your-custom-oidc-provider.com/userinfo"
```

### Multi-Tenant Access Control

**Required Headers:**

- `Authorization: Bearer <oidc-token>` - Valid JWT token from identity provider
- `X-Scope-OrgId: <tenant-name>` - Target tenant identifier (mandatory)

**Security Enforcement:**

```nginx
# Ensure requests have the X-Scope-OrgID header set
if ($http_x_scope_orgid = "") {
  return 401;
}
add_header X-Scope-OrgID $http_x_scope_orgid;
```

## Configuration

### Template Variables

Configuration is managed via template files in [shared-configs](https://github.com/giantswarm/shared-configs):

| Variable | Purpose | Example |
|----------|---------|---------|
| `.codename` | Installation codename | `golem`, `ghost` |
| `.base` | Base domain | `giantswarm.io` |
| `.ingress.tls.letsencrypt` | Enable automatic TLS | `true` |
| `.managementCluster.clusterIssuer` | TLS certificate issuer | `letsencrypt-prod` |
| `.observabilityPlatformApi.authUrl` | Custom OIDC endpoint | Custom URL or default to Dex |

### Example Configuration

```yaml
ingresses:
  - name: gateway
    namespace: monitoring
    service:
      name: observability-gateway-alloy
      port: 3100
    hosts:
      - host: observability.{{ .codename }}.{{ .base }}
        paths:
        - path: /loki/api/v1/push
          pathType: ImplementationSpecific
    annotations:
      nginx.ingress.kubernetes.io/auth-url: {{ .observabilityPlatformApi.authUrl | default "https://dex.{{ .codename }}.{{ .base }}/userinfo" }}
```

## Usage Examples

### External Grafana Configuration

**Data Source Setup:**

```yaml
datasources:
  - name: External Loki
    type: loki
    url: https://observability.<codename>.<base-domain>
    jsonData:
      httpHeaderName1: X-Scope-OrgId
    secureJsonData:
      httpHeaderValue1: your-tenant-name
```

### Programmatic Access

**Send Traces:**

```bash
curl -X POST \
     -H "Authorization: Bearer $OIDC_TOKEN" \
     -H "X-Scope-OrgId: your-tenant" \
     -H "Content-Type: application/json" \
     -d @traces-payload.json \
     "https://observability.<codename>.<base>/v1/traces"
```

**Query Logs:**

```bash
curl -H "Authorization: Bearer $OIDC_TOKEN" \
     -H "X-Scope-OrgId: your-tenant" \
     "https://observability.<codename>.<base>/loki/api/v1/query_range?query={job=\"your-job\"}"
```

**Query Metrics:**

```bash
curl -H "Authorization: Bearer $OIDC_TOKEN" \
     -H "X-Scope-OrgId: your-tenant" \
     "https://observability.<codename>.<base>/prometheus/api/v1/query?query=up"
```

**Query Traces:**

```bash
curl -H "Authorization: Bearer $OIDC_TOKEN" \
     -H "X-Scope-OrgId: your-tenant" \
     "https://observability.<codename>.<base>/tempo/api/search?tags=service.name=your-service"
```

## Deployment Information

- **Target Environment**: Management clusters only
- **Installation**: Automatically deployed by Giant Swarm platform management
- **Dependencies**: Requires Loki, Mimir, and alloy-gateway services to be running
- **Configuration Location**: [shared-configs/default/apps/observability-platform-api](https://github.com/giantswarm/shared-configs/tree/main/default/apps/observability-platform-api)

## Related Components

### Core Dependencies
- [**alloy-gateway-app**](https://github.com/giantswarm/alloy-gateway-app) - Data ingestion gateway
- **loki-gateway** - Log storage and querying service
- **mimir-gateway** - Metrics storage and querying service
- **tempo-gateway** - Trace storage and querying service

### Documentation

- [**Data Import/Export Guide**](https://docs.giantswarm.io/overview/observability/data-management/data-import-export/) - Complete API usage documentation
- [**Multi-Tenancy Setup**](https://docs.giantswarm.io/overview/observability/configuration/multi-tenancy/) - Tenant configuration guide

## Installing

**Note:** This application is deployed automatically on management clusters and is not intended for manual installation.

For development or testing purposes:

- [Using GitOps to instantiate the App](https://docs.giantswarm.io/tutorials/continuous-deployment/apps/add-appcr/)
- By creating an [App resource](https://docs.giantswarm.io/reference/platform-api/crd/apps.application.giantswarm.io) using the platform API

---

**Need help with configuration?** Contact your Giant Swarm account team for assistance with OIDC setup and ingress configuration.
