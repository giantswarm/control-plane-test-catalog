[![CircleCI](https://dl.circleci.com/status-badge/img/gh/giantswarm/observability-platform-api/tree/main.svg?style=svg)](https://dl.circleci.com/status-badge/redirect/gh/giantswarm/observability-platform-api/tree/main)

# observability-platform-api

## Purpose

The **observability-platform-api** provides the external access layer for Giant Swarm's Observability Platform, managing Gateway API HTTPRoutes that enable secure access to observability services from external sources. This app handles all external routing, authentication, and access control for the platform's APIs.

### What this app is for:

- **External API Access**: Secure HTTP/HTTPS endpoints for external systems to interact with observability services
- **Authentication Gateway**: OIDC JWT authentication for all external requests, with the tenant taken from the `X-Scope-OrgID` header
- **Multi-Service Routing**: Unified domain with path-based routing to different observability backends
- **Access Control**: Envoy Gateway `SecurityPolicy` (JWT) on every route, and read routes limited to read methods. Tenants are not isolated, see [Tenancy](#tenancy)

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

The observability-platform-api creates `HTTPRoute` and `GRPCRoute` resources under a unified domain for direct access to observability backends. Each route is secured by an Envoy Gateway `SecurityPolicy` (JWT) and enforces the `X-Scope-OrgID` tenant header.

The table below describes the JWT domain. When `basicAuth.enabled` is set, the Mimir and Loki read rows are additionally served on `basicAuth.hostname` behind Basic Auth.

Read routes match the HTTP method as well as the path. Other methods get a `404` from Envoy. Each entry of `loki.read.paths` and `mimir.read.paths` is either a path string, which allows `GET` only, or `{path, methods}`:

- Loki: `GET` and `POST` on the query, label, series, index, `format_query`, `detected_*` and `patterns` endpoints. `GET` only on `/loki/api/v1/rules`, `/loki/api/v1/tail`, `/loki/api/v1/status/buildinfo` and `/loki/api/v1/drilldown-limits`. Creating or deleting Loki rules is not possible through the API.
- Mimir: `GET` and `POST` on every read path, except `/prometheus/api/v1/rules`, which is `GET` only.

A list can hold at most 64 path and method pairs. Each pair is rendered twice (with and without `X-Scope-OrgID`) and Gateway API allows 128 matches per route.

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
│             │ /loki/api/v1/tail                                            │                                  │               │
│             │ /loki/api/v1/format_query                                    │                                  │               │
│             │ /loki/api/v1/index/stats                                     │                                  │               │
│             │ /loki/api/v1/index/volume                                    │                                  │               │
│             │ /loki/api/v1/index/volume_range                              │                                  │               │
│             │ /loki/api/v1/detected_fields                                 │                                  │               │
│             │ /loki/api/v1/patterns                                        │                                  │               │
│             │ /loki/api/v1/detected_field                                  │                                  │               │
│             │ /loki/api/v1/status/buildinfo                                │                                  │               │
│             │ /loki/api/v1/drilldown-limits                                │                                  │               │
│ HTTPS       │ /loki/api/v1/push                                            │ Logs / Loki                      │ Write         │
│ HTTPS       │ /otlp/v1/logs                                                │ Logs / Loki (OTLP HTTP)          │ Write         │
│ HTTPS       │ /prometheus/api/v1/query                                     │ Metrics / Mimir                  │ Read          │
│             │ /prometheus/api/v1/query_range                               │                                  │               │
│             │ /prometheus/api/v1/query_exemplars                           │                                  │               │
│             │ /prometheus/api/v1/labels                                    │                                  │               │
│             │ /prometheus/api/v1/label                                     │                                  │               │
│             │ /prometheus/api/v1/series                                    │                                  │               │
│             │ /prometheus/api/v1/rules                                     │                                  │               │
│             │ /prometheus/api/v1/status                                    │                                  │               │
│             │ /prometheus/api/v1/metadata                                  │                                  │               │
│             │ /prometheus/api/v1/detected_labels                           │                                  │               │
│ HTTPS       │ /prometheus/api/v1/push  (rewritten → /api/v1/push)         │ Metrics / Mimir                  │ Write         │
│ HTTPS       │ /otlp/v1/metrics                                             │ Metrics / Mimir (OTLP)           │ Write         │
│ HTTPS       │ /tempo/api/echo                                              │ Traces / Tempo                   │ Read          │
│             │ /tempo/api/status/buildinfo                                  │                                  │               │
│             │ /tempo/api/metrics/query_range                               │                                  │               │
│             │ /tempo/api/search                                            │                                  │               │
│             │ /tempo/api/search/tags                                       │                                  │               │
│             │ /tempo/api/v2/search                                         │                                  │               │
│             │ /tempo/api/v2/search/tags                                    │                                  │               │
│             │ /tempo/api/v2/search/tag/{tag}/values                        │                                  │               │
│             │ /tempo/api/traces                                            │                                  │               │
│             │ /tempo/api/v2/traces  (all rewritten, /tempo prefix removed) │                                  │               │
│ gRPC (+TLS) │ /tempopb                                                     │ Traces / Tempo                   │ Read          │
│             │ ├── /tempopb.StreamingQuerier.SearchTagsV2                   │                                  │               │
│             │ ├── /tempopb.StreamingQuerier.MetricsQueryRange              │                                  │               │
│             │ └── ...                                                      │                                  │               │
│ HTTPS       │ /v1/traces                                                   │ Traces / Tempo (OTLP HTTP)       │ Write         │
│ gRPC (+TLS) │ opentelemetry.proto.collector.trace.v1.TraceService          │ Traces / Tempo (OTLP gRPC)       │ Write         │
└─────────────┴──────────────────────────────────────────────────────────────┴──────────────────────────────────┴───────────────┘
```

### Authentication

All routes (read and write) require JWT Bearer token authentication via Envoy Gateway `SecurityPolicy`.

All routes additionally enforce that the `X-Scope-OrgID` header is present and non-empty:

- HTTP routes: a request with a valid credential but no `X-Scope-OrgID` gets `401` with the `text/plain` body `Missing X-Scope-OrgID header`. A missing or invalid JWT also gets `401`, with the body `Jwt is missing` (or the validation error) and a `www-authenticate` header, so the body tells the two cases apart.
- gRPC routes (Tempo): a `GRPCRoute` cannot return a custom response, so a request without `X-Scope-OrgID` matches no route. The client gets `grpc-status: 12` (`UNIMPLEMENTED`) with no message, and Envoy logs `route_not_found`.

Optionally, the Mimir and Loki *read* paths can additionally be exposed on a second hostname behind HTTP Basic Auth, for clients that cannot obtain an OIDC token — see [Basic Auth read routes](docs/basic-auth.md). This is disabled by default.

#### Configuring JWT providers

Set `auth.jwt.providers` to the list of trusted OIDC issuers. JWT validation is done inline by Envoy Gateway against the issuer's JWKS endpoint — no external auth service required.

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

Multiple OIDC providers are supported — tokens from any configured issuer are accepted. This handles both human users (OIDC sessions via Dex, Azure AD, etc.) and applications (Azure AD service principals, any OIDC-compliant IdP). Helm template strings are supported in all fields.

### Tenancy

The API authenticates requests. It does not isolate tenants:

- The JWT is validated, but it is not tied to a tenant. The tenant is whatever the client sends in `X-Scope-OrgID`, and it reaches the backend unchanged.
- So any valid token can read and write any tenant of Loki, Mimir and Tempo. The same is true of the Basic Auth routes.
- A piped header such as `X-Scope-OrgID: a|b` reaches Loki. When the Loki read target runs with `-querier.multi-tenant-queries-enabled`, one query returns data from several tenants. Mimir does the same when `-tenant-federation.enabled` is set.

Only give tokens to clients that may access every tenant on the installation.

### Known limitations

- Loki's Prometheus-compatible rules API (`/prometheus/api/v1/rules` and `/prometheus/api/v1/alerts` on Loki) is not exposed. When Loki and Mimir share a hostname (the default), `/prometheus/api/v1/rules` goes to Mimir. So Grafana cannot list Loki-managed alert rules through the API. Use `/loki/api/v1/rules` to list Loki rules.

## Architecture Notes

### Multi-Route Design

This app creates multiple `HTTPRoute` and `GRPCRoute` resources (one per service per direction) rather than a single route because:

**Benefits:**
- **Granular Control**: Each service can have independent configuration and lifecycle
- **Namespace Isolation**: Different backend services live in different namespaces (`loki`, `mimir`, `tempo`)
- **Feature Flags**: Individual routes can be enabled/disabled based on cluster capabilities
- **Security Boundaries**: Each service can be independently enabled or disabled without affecting others

**Template structure** — templates are organised per service under `templates/loki/`, `templates/mimir/`, and `templates/tempo/`. Each directory contains:
- `route-read.yaml` — HTTP read `HTTPRoute`
- `route-write.yaml` — HTTP write `HTTPRoute`
- `route-grpc.yaml` — gRPC `GRPCRoute`(s): Tempo read + OTLP write
- `securitypolicy.yaml` — one `SecurityPolicy` per route for Loki and Mimir (single SP covers all HTTP routes); Tempo requires one SP per route because each `GRPCRoute` must have its own `SecurityPolicy`
- `filters.yaml` — shared `HTTPRouteFilter` resources (headers-check and path rewrite where applicable)

**Operational Considerations:**

- All routes share the same hostname and `X-Scope-OrgID` enforcement
- JWT providers (`auth.jwt.providers`) are shared across all services
- JWT validation is done inline by Envoy Gateway — no external auth service required
- gRPC routes (`GRPCRoute`) do not support `HTTPRouteFilter` via `ExtensionRef`, so missing `X-Scope-OrgID` on gRPC requests results in a no-route rejection (`UNIMPLEMENTED`) rather than a strict 401

## Configuration & Deployment

**All configuration is managed through [shared-configs](https://github.com/giantswarm/shared-configs)** - this repository provides the base templates that are populated by the shared-configs system during deployment.

- **Target Environment**: Management clusters only (not workload clusters)
- **Deployment Method**: Automatically via Giant Swarm platform management
- **Configuration Source**: Templates in this repo + values from shared-configs
- **Feature Control**: Conditional route creation based on cluster capabilities (`loki.enabled`, `mimir.enabled`, `tempo.enabled`)

## Documentation & Resources

### User Documentation

- [**Basic Auth read routes**](docs/basic-auth.md) - Optional Basic Auth hostname for Mimir and Loki reads: configuration, credential format, rotation
- [**Data Import/Export Guide**](https://docs.giantswarm.io/overview/observability/data-management/data-import-export/) - Public API documentation and usage examples
- [**Intranet Documentation**](https://intranet.giantswarm.io/docs/observability/gateway/) - Internal operational guides

### Related Repositories

- [**shared-configs**](https://github.com/giantswarm/shared-configs) - Central configuration management system

### Project Information

- [**Implementation Roadmap**](https://github.com/giantswarm/roadmap/issues/3568) - Original project scope and requirements
- **Team**: Atlas (@giantswarm/team-atlas)
- **Status**: Production deployment on management clusters
