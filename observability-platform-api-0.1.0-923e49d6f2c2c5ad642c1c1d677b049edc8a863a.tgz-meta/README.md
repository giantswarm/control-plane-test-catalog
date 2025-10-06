[![CircleCI](https://dl.circleci.com/status-badge/img/gh/giantswarm/observability-platform-api/tree/main.svg?style=svg)](https://dl.circleci.com/status-badge/redirect/gh/giantswarm/observability-platform-api/tree/main)

# observability-platform-api

## Purpose

The **observability-platform-api** provides the external access layer for Giant Swarm's Observability Platform, managing NGINX ingresses that enable secure access to observability services from external sources. This app handles all external routing, authentication, and access control for the platform's APIs.

### What this app is for:

- **External API Access**: Secure HTTP/HTTPS endpoints for external systems to interact with observability services
- **Authentication Gateway**: OIDC-based authentication and tenant routing for all external requests
- **Multi-Service Routing**: Unified domain with path-based routing to different observability backends
- **Access Control**: Enforcement of tenant isolation and security policies for external access

## Place in Observability Platform

The **observability-platform-api** serves as the external gateway of Giant Swarm's Observability Platform, working in tandem with the [alloy-gateway-app](https://github.com/giantswarm/alloy-gateway-app) to provide complete external access capabilities.

**Complete Platform Components:**

- **observability-platform-api** (this repo) → External access control and routing
- [**alloy-gateway-app**](https://github.com/giantswarm/alloy-gateway-app) → Data processing and forwarding for ingestion
- **Loki, Mimir, Tempo** → Storage backends for logs, metrics, and traces

All configuration is managed centrally through [shared-configs](https://github.com/giantswarm/shared-configs) templates, ensuring consistent deployment across all Giant Swarm installations.

## Technical Implementation

This repository contains the Helm chart and configuration templates for creating and managing NGINX ingresses that expose observability platform APIs to external users.

## Technical Architecture

### Ingress Management

The observability-platform-api creates **6 separate ingresses** under a unified domain:

```
┌───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                              Domain: https://observability.<codename>.<base-domain>                                           │
├─────────────┬──────────────────────────────────────────────────────────────┬──────────────────────────────────┬───────────────┤
│ Protocol    │ Path                                                         │ Backend                          │ Type          │
├─────────────┼──────────────────────────────────────────────────────────────┼──────────────────────────────────┼───────────────┤
│ HTTP (OTLP) │ /v1/traces                                                   │ observability-gateway-alloy:4318 │ Write         │
│ HTTP        │ /loki/api/v1/push                                            │ observability-gateway-alloy:3100 │ Write         │
│ HTTP        │ /loki/api/v1/*                                               │ loki-gateway:80                  │ Read          │
│             │ ├── /loki/api/v1/query                                       │                                  │               │
│             │ ├── /loki/api/v1/labels                                      │                                  │               │
│             │ └── ...                                                      │                                  │               │
│ HTTP        │ /prometheus/api/v1/*                                         │ mimir-gateway:80                 │ Read          │
│             │ ├── /prometheus/api/v1/query                                 │                                  │               │
│             │ ├── /prometheus/api/v1/labels                                │                                  │               │
│             │ └── ...                                                      │                                  │               │
│ HTTP        │ /tempo/api/*                                                 │ tempo-query-frontend:3200        │ Read          │
│             │ ├── /tempo/api/v2/search                                     │                                  │               │
│             │ ├── /tempo/api/v2/traces                                     │                                  │               │
│             │ └── ...                                                      │                                  │               │
│ gRPC        │ /tempopb.*                                                   │ tempo-query-frontend:9095        │ Read/Write    │
│             │ ├── /tempopb.StreamingQuerier.SearchTagsV2                   │                                  │               │
│             │ ├── /tempopb.StreamingQuerier.MetricsQueryRange              │                                  │               │
│             │ └── ...                                                      │                                  │               │
└─────────────┴──────────────────────────────────────────────────────────────┴──────────────────────────────────┴───────────────┘
```

## Architecture Notes

### Multi-Ingress Design

This app creates multiple ingresses rather than a single ingress because:

**Benefits:**
- **Granular Control**: Each service can have independent configuration and lifecycle
- **Namespace Isolation**: Different backend services live in different namespaces
- **Feature Flags**: Individual ingresses can be enabled/disabled based on cluster capabilities
- **Security Boundaries**: Different authentication or access policies per service type

**Operational Considerations:**

- All ingresses share the same domain and authentication configuration
- Consistent header validation and tenant isolation across all endpoints
- Unified TLS certificate management for the shared domain

## Configuration & Deployment

**All configuration is managed through [shared-configs](https://github.com/giantswarm/shared-configs)** - this repository provides the base templates that are populated by the shared-configs system during deployment.

- **Target Environment**: Management clusters only (not workload clusters)
- **Deployment Method**: Automatically via Giant Swarm platform management
- **Configuration Source**: Templates in this repo + values from shared-configs
- **Feature Control**: Conditional ingress creation based on cluster capabilities

## Documentation & Resources

### User Documentation

- [**Data Import/Export Guide**](https://docs.giantswarm.io/overview/observability/data-management/data-import-export/) - Public API documentation and usage examples
- [**Intranet Documentation**](https://intranet.giantswarm.io/docs/observability/gateway/) - Internal operational guides

### Related Repositories

- [**alloy-gateway-app**](https://github.com/giantswarm/alloy-gateway-app) - Data ingestion gateway and processing
- [**shared-configs**](https://github.com/giantswarm/shared-configs) - Central configuration management system

### Project Information

- [**Implementation Roadmap**](https://github.com/giantswarm/roadmap/issues/3568) - Original project scope and requirements
- **Team**: Atlas (@giantswarm/team-atlas)
- **Status**: Production deployment on management clusters
