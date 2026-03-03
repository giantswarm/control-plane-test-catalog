[![CircleCI](https://dl.circleci.com/status-badge/img/gh/giantswarm/observability-platform-api/tree/main.svg?style=svg)](https://dl.circleci.com/status-badge/redirect/gh/giantswarm/observability-platform-api/tree/main)

# observability-platform-api

## Purpose

The **observability-platform-api** provides the external access layer for Giant Swarm's Observability Platform, managing Gateway API HTTPRoutes that enable secure access to observability services from external sources. This app handles all external routing, authentication, and access control for the platform's APIs.

### What this app is for:

- **External API Access**: Secure HTTP/HTTPS endpoints for external systems to interact with observability services
- **Authentication Gateway**: OIDC-based authentication and tenant routing for all external requests
- **Multi-Service Routing**: Unified domain with path-based routing to different observability backends
- **Access Control**: Enforcement of tenant isolation via Envoy Gateway `SecurityPolicy` for all external access

## Place in Observability Platform

The **observability-platform-api** serves as the external gateway of Giant Swarm's Observability Platform, providing direct access to observability services.

**Complete Platform Components:**

- **observability-platform-api** (this repo) → External access control and routing
- **Loki, Mimir, Tempo** → Storage backends for logs, metrics, and traces

All configuration is managed centrally through [shared-configs](https://github.com/giantswarm/shared-configs) templates, ensuring consistent deployment across all Giant Swarm installations.

## Technical Implementation

This repository contains the Helm chart and configuration templates for creating and managing Gateway API `HTTPRoute` resources (via Envoy Gateway) that expose observability platform APIs to external users.

## Technical Architecture

### Route Management

The observability-platform-api creates separate `HTTPRoute` resources under a unified domain for direct access to observability backends. Each route is secured by an Envoy Gateway `SecurityPolicy` (JWT validation) and enforces the `X-Scope-OrgID` tenant header.

```
┌───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                              Domain: https://observability.<codename>.<base-domain>                                           │
├─────────────┬──────────────────────────────────────────────────────────────┬──────────────────────────────────┬───────────────┤
│ Protocol    │ Path                                                         │ Data type / backend              │ Type          │
├─────────────┼──────────────────────────────────────────────────────────────┼──────────────────────────────────┼───────────────┤
│ HTTPS       │ /loki/api/v1/query                                           │ Logs / Loki                      │ Read          │
│             │ /loki/api/v1/query_range                                     │                                  │               │
│             │ /loki/api/v1/labels                                          │                                  │               │
│             │ /loki/api/v1/label                                           │                                  │               │
│             │ /loki/api/v1/series                                          │                                  │               │
│             │ /loki/api/v1/index                                           │                                  │               │
│             │ /loki/api/v1/rules                                           │                                  │               │
│             │ /loki/api/v1/detected_labels                                 │                                  │               │
│ HTTPS       │ /loki/api/v1/push                                            │ Logs / Loki                      │ Write         │
│ HTTPS       │ /prometheus/api/v1/query                                     │ Metrics / Mimir                  │ Read          │
│             │ /prometheus/api/v1/query_range                               │                                  │               │
│             │ /prometheus/api/v1/query_exemplars                           │                                  │               │
│             │ /prometheus/api/v1/labels                                    │                                  │               │
│             │ /prometheus/api/v1/label                                     │                                  │               │
│             │ /prometheus/api/v1/rules                                     │                                  │               │
│             │ /prometheus/api/v1/status                                    │                                  │               │
│             │ /prometheus/api/v1/metadata                                  │                                  │               │
│             │ /prometheus/api/v1/detected_labels                           │                                  │               │
│ HTTPS       │ /prometheus/api/v1/push  (rewritten → /api/v1/push)         │ Metrics / Mimir                  │ Write         │
│ HTTPS       │ /tempo/api/echo                                              │ Traces / Tempo                   │ Read          │
│             │ /tempo/api/status/buildinfo                                  │                                  │               │
│             │ /tempo/api/metrics/query_range                               │                                  │               │
│             │ /tempo/api/search                                            │                                  │               │
│             │ /tempo/api/v2/search                                         │                                  │               │
│             │ /tempo/api/traces                                            │                                  │               │
│             │ /tempo/api/v2/traces  (all rewritten, /tempo prefix removed) │                                  │               │
│ gRPC (+TLS) │ /tempopb                                                     │ Traces / Tempo                   │ Read          │
│             │ ├── /tempopb.StreamingQuerier.SearchTagsV2                   │                                  │               │
│             │ ├── /tempopb.StreamingQuerier.MetricsQueryRange              │                                  │               │
│             │ └── ...                                                      │                                  │               │
│ HTTPS       │ /v1/traces                                                   │ OpenTelemetry Traces / Tempo     │ Write         │
└─────────────┴──────────────────────────────────────────────────────────────┴──────────────────────────────────┴───────────────┘
```

### Authentication

All routes (read and write) use Envoy Gateway's native JWT validation via `SecurityPolicy.jwt`. This validates Bearer tokens directly against the JWKS endpoint of each configured issuer, with no external auth service required.

Multiple OIDC providers are supported — tokens from any configured issuer are accepted. This handles both human users (OIDC sessions via Dex, Azure AD, etc.) and applications (Azure AD service principals, any OIDC-compliant IdP).

All routes additionally enforce that the `X-Scope-OrgID` header is present and non-empty — requests missing it receive a `401`.

#### Configuring JWT providers

Set `auth.jwt.providers` to the list of trusted OIDC issuers. At least one provider is required when any service is enabled:

```yaml
auth:
  jwt:
    providers:
    - name: dex
      issuer: "https://dex.mycluster.example.com"
      remoteJWKS:
        uri: "https://dex.mycluster.example.com/keys"
    - name: azure-ad
      issuer: "https://login.microsoftonline.com/<tenant-id>/v2.0"
      remoteJWKS:
        uri: "https://login.microsoftonline.com/<tenant-id>/discovery/v2.0/keys"
```

Each provider entry maps directly to an Envoy Gateway JWT provider. Tokens are validated against the issuer's JWKS endpoint; any token from a listed issuer is accepted. Helm template strings are supported in all fields.

## Architecture Notes

### Multi-Route Design

This app creates multiple `HTTPRoute` resources rather than a single route because:

**Benefits:**
- **Granular Control**: Each service can have independent configuration and lifecycle
- **Namespace Isolation**: Different backend services live in different namespaces (`loki`, `mimir`, `tempo`)
- **Feature Flags**: Individual routes can be enabled/disabled based on cluster capabilities
- **Security Boundaries**: Each service can be independently enabled or disabled without affecting others

**Operational Considerations:**

- All routes share the same hostname, `X-Scope-OrgID` enforcement, and JWT provider list
- JWT validation is done inline by Envoy Gateway — no external auth service required

## Configuration & Deployment

**All configuration is managed through [shared-configs](https://github.com/giantswarm/shared-configs)** - this repository provides the base templates that are populated by the shared-configs system during deployment.

- **Target Environment**: Management clusters only (not workload clusters)
- **Deployment Method**: Automatically via Giant Swarm platform management
- **Configuration Source**: Templates in this repo + values from shared-configs
- **Feature Control**: Conditional route creation based on cluster capabilities (`loki.enabled`, `mimir.enabled`, `tempo.enabled`)

## Documentation & Resources

### User Documentation

- [**Data Import/Export Guide**](https://docs.giantswarm.io/overview/observability/data-management/data-import-export/) - Public API documentation and usage examples
- [**Intranet Documentation**](https://intranet.giantswarm.io/docs/observability/gateway/) - Internal operational guides

### Related Repositories

- [**shared-configs**](https://github.com/giantswarm/shared-configs) - Central configuration management system

### Project Information

- [**Implementation Roadmap**](https://github.com/giantswarm/roadmap/issues/3568) - Original project scope and requirements
- **Team**: Atlas (@giantswarm/team-atlas)
- **Status**: Production deployment on management clusters
