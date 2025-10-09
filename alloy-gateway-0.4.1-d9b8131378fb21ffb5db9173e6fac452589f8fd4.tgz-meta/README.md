[![CircleCI](https://dl.circleci.com/status-badge/img/gh/giantswarm/alloy-gateway-app/tree/main.svg?style=svg)](https://dl.circleci.com/status-badge/redirect/gh/giantswarm/alloy-gateway-app/tree/main)

# alloy-gateway-app

## Purpose

The **alloy-gateway-app** enables Giant Swarm customers to send observability data from external sources into the platform's centralized observability platform. This app is specifically designed for data ingestion from sources **outside** of Giant Swarm managed Kubernetes clusters.

## Place in Observability Platform

The **alloy-gateway-app** is a core component of Giant Swarm's Observability Platform API ecosystem, working in tandem with the [observability-platform-api](https://github.com/giantswarm/observability-platform-api) to provide secure external data ingestion capabilities.

**Complete Platform Components:**

- **alloy-gateway-app** (this repo) → Data processing and forwarding
- [**observability-platform-api**](https://github.com/giantswarm/observability-platform-api) → External access control and routing
- **Loki, Mimir, Tempo** → Storage backends for logs, metrics, and traces

All configuration is managed centrally through [shared-configs](https://github.com/giantswarm/shared-configs) templates, ensuring consistent deployment across all Giant Swarm installations.

## Technical Implementation

This repository contains the Helm chart and configuration templates for deploying Grafana Alloy in gateway mode to accept external observability data securely.

## Configuration & Deployment

**All configuration is managed through [shared-configs](https://github.com/giantswarm/shared-configs)** - this repository provides the base templates that are populated by the shared-configs system during deployment.

- **Target Environment**: Management clusters only (not workload clusters)
- **Deployment Method**: Automatically via Giant Swarm platform management
- **Configuration Source**: Templates in this repo + values from shared-configs

## Documentation & Resources

### User Documentation

- [**Data Import/Export Guide**](https://docs.giantswarm.io/overview/observability/data-management/data-import-export/) - Public API documentation and usage examples
- [**Intranet Documentation**](https://intranet.giantswarm.io/docs/observability/gateway/) - Internal operational guides

### Related Repositories

- [**observability-platform-api**](https://github.com/giantswarm/observability-platform-api) - Ingress management and external access control
- [**shared-configs**](https://github.com/giantswarm/shared-configs) - Central configuration management system
- [**alloy-app**](https://github.com/giantswarm/alloy-app) - Base Grafana Alloy Helm chart

### Project Information

- [**Implementation Roadmap**](https://github.com/giantswarm/roadmap/issues/3568) - Original project scope and requirements
- **Team**: Atlas (@giantswarm/team-atlas)
- **Status**: Production deployment on management clusters
