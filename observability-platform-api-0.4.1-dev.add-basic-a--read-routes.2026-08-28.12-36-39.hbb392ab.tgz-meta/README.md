[![CircleCI](https://dl.circleci.com/status-badge/img/gh/giantswarm/observability-platform-api/tree/main.svg?style=svg)](https://dl.circleci.com/status-badge/redirect/gh/giantswarm/observability-platform-api/tree/main)

# observability-platform-api

## Purpose

The **observability-platform-api** provides the external access layer for Giant Swarm's Observability Platform, managing Gateway API HTTPRoutes that enable secure access to observability services from external sources. This app handles all external routing, authentication, and access control for the platform's APIs.

### What this app is for:

- **External API Access**: Secure HTTP/HTTPS endpoints for external systems to interact with observability services
- **Authentication Gateway**: OIDC JWT authentication and tenant routing for all external requests
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

The observability-platform-api creates `HTTPRoute` and `GRPCRoute` resources under a unified domain for direct access to observability backends. Each route is secured by an Envoy Gateway `SecurityPolicy` (JWT) and enforces the `X-Scope-OrgID` tenant header.

The table below describes the JWT domain. When `basicAuth.enabled` is set, the Mimir and Loki read rows are additionally served on `basicAuth.hostname` behind Basic Auth.

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
│ HTTPS       │ /loki/api/v1/push                                            │ Logs / Loki                      │ Write         │
│ HTTPS       │ /otlp/v1/logs                                                │ Logs / Loki (OTLP HTTP)          │ Write         │
│ gRPC (+TLS) │ opentelemetry.proto.collector.logs.v1.LogsService            │ Logs / Loki (OTLP gRPC)          │ Write         │
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

All routes additionally enforce that the `X-Scope-OrgID` header is present and non-empty — requests missing it receive a `401`.

Optionally, the Mimir and Loki *read* paths can additionally be exposed on a second hostname behind HTTP Basic Auth, for clients that cannot obtain an OIDC token — see [Basic Auth read routes](#basic-auth-read-routes-opt-in). This is disabled by default.

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

### Basic Auth read routes (opt-in)

Some clients cannot present an OIDC token. The case this exists for is an
external, customer-managed Grafana: its core Prometheus and Loki datasources
have no OAuth2 client-credentials support, and *Forward OAuth identity* only
covers interactive queries — alerting rules evaluate without a user session, so
they have no token to forward.

For those clients, `basicAuth.enabled` exposes the **Mimir and Loki read paths**
on a **separate hostname**, authenticated with HTTP Basic Auth instead of a JWT.
Write paths and Tempo are not exposed. The JWT routes are untouched.

```yaml
basicAuth:
  enabled: true
  hostname: "observability-basicauth.<codename>.<base-domain>"
  parentRefs:
  - name: giantswarm-default
    namespace: envoy-gateway-system
  usersSecret:
    name: observability-basicauth-users
    namespace: monitoring   # default
```

This renders, per enabled service, an `HTTPRoute` on `basicAuth.hostname`
reusing that service's existing `read.paths` / `read.backendService` /
`read.backendPort`, plus a `SecurityPolicy` carrying only `basicAuth`, plus a
headers-check `HTTPRouteFilter`. A single `ReferenceGrant` in
`usersSecret.namespace` lets both SecurityPolicies read the one Secret.

`X-Scope-OrgID` is enforced exactly as on the JWT routes: present and non-empty,
or `401`. Basic Auth authenticates the caller; it does not scope them to a
tenant — the same is true of the JWT routes today.

> **Prerequisite:** `basicAuth.hostname` must be covered by a listener on the
> parent Gateway (and by its TLS certificate). If the Gateway uses exact-host
> listeners rather than a wildcard, add the hostname there first.

#### Why a separate hostname

Envoy Gateway runs the `jwt` and `basicAuth` filters of a `SecurityPolicy`
sequentially, with AND semantics: a route configured with both rejects every
request, whichever credential it carries — a JWT gets `Expected 'Basic'
authentication scheme`, Basic credentials get `Jwt is missing`. See
[envoyproxy/gateway#8491](https://github.com/envoyproxy/gateway/issues/8491).

Separate routes on a separate hostname is the workaround: each route carries
exactly one auth method, so the two never meet in one filter chain.

#### Basic Auth credential format

The chart does **not** create, template or manage the users Secret. It is
provisioned and rotated out of band by whoever owns the credentials, so no
password or password hash passes through this chart's values or through Giant
Swarm configuration.

The chart renders the routes based on the *values* alone — it never looks the
Secret up in the cluster — so the Secret and the chart can be applied in either
order, and creating the Secret afterwards needs no redeploy:

- If the Secret is missing when the chart deploys, Envoy Gateway cannot resolve
  the `SecurityPolicy` and sets a **500 direct response** on the affected routes.
  It fails closed: the routes are never briefly unauthenticated.
- Envoy Gateway watches Secrets and indexes `basicAuth.users` references, so it
  picks the Secret up on its own once created. The same applies to later edits,
  which is what makes [rotation](#rotation) a Secret-only operation.

The Secret must contain a `.htpasswd` key, one `user:hash` entry per line:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: observability-basicauth-users
  namespace: monitoring
type: Opaque
stringData:
  .htpasswd: |
    acme-grafana-1:{SHA}3WBs1Ju70GtMJgb8JEn4+4eXV4Y=
```

**Only the `{SHA}` scheme is supported.** Modern `htpasswd` defaults to bcrypt,
which Envoy rejects — pass `-s` explicitly:

```bash
# prompts for the password (keeps it out of shell history)
htpasswd -ns acme-grafana-1

# or, without apache2-utils:
printf '%s' '<password>' | openssl sha1 -binary | openssl base64
# prefix the result with {SHA} and the username
```

Conventions worth keeping to:

- **Use a long random password.** `{SHA}` is unsalted SHA-1, so a weak password
  is cheap to recover from the hash. 32+ random characters, not a passphrase.
- **Name the user after the consumer and its generation** — `acme-grafana-1`,
  not `grafana`. The suffix is what makes rotation possible.
- CRLF line endings are normalised, so a file authored on Windows is fine.

#### Rotation

The `.htpasswd` key holds a *list*, which is the rotation mechanism. There is no
coordinated cutover and no involvement from Giant Swarm:

1. Add a second entry with a new username and password. Both now work.
2. Switch the client (e.g. the Grafana datasource) to the new credential. If
   anything goes wrong, the old one is still live — switch back.
3. Delete the old entry.

Giant Swarm cannot read, recover or reset these credentials. If they are lost,
the holder replaces the hash themselves.

Revocation does not depend on the credential holder: emptying the `.htpasswd`
key, or setting `basicAuth.enabled: false`, removes access without touching the
JWT routes.

#### Troubleshooting

A malformed Secret surfaces on the `SecurityPolicy` status rather than in the
chart, since the chart never validates its contents:

```bash
kubectl get securitypolicy -n mimir <release>-mimir-basicauth -o yaml
```

| Symptom | Cause |
|---|---|
| Every request gets `500` | The `SecurityPolicy` did not translate — see its status for which of the two below it is. Envoy Gateway fails closed rather than serving the routes unauthenticated |
| `secret <ns>/<name> does not exist` | Secret missing, or `ReferenceGrant` not applied |
| `secret <ns>/<name> must contain a non-empty ".htpasswd" key` | Wrong key name, or empty value |
| Policy accepted, but every request gets `401` | Hash is not `{SHA}` (bcrypt/MD5), or the password does not match |
| `401` with a valid credential | `X-Scope-OrgID` header missing or empty |

`500` means the policy is broken; `401` means the policy is working and the
credential is not. That distinction is the fastest way to tell a Kubernetes-level
problem from a credential-level one.

If `basicAuth.enabled` is set but `hostname`, `usersSecret.name` or
`usersSecret.namespace` is empty *in the values*, nothing is rendered at all —
the same convention the JWT routes follow with an empty `auth.jwt.providers`.
This is a values check, not a cluster lookup: a configured but non-existent
Secret still renders the routes, and produces the `500` above.

#### Removing Basic Auth (maintainers)

> This subsection is for maintainers of this chart. It is not part of operating
> or using the Basic Auth routes — if you are configuring an installation or
> managing credentials, stop at the previous subsection.

This is a stopgap, to be removed once
[envoyproxy/gateway#8491](https://github.com/envoyproxy/gateway/issues/8491)
lands and a single route can accept either credential. The upstream Envoy half
is already merged ([envoyproxy/envoy#43911](https://github.com/envoyproxy/envoy/pull/43911),
adding `allow_missing` to the basic_auth filter); what remains is the Envoy
Gateway API for it.

The feature is deliberately self-contained — no existing template references it.
To remove it: delete `templates/basicauth/`, delete the `basicAuth` block from
`values.yaml` and `values.schema.json`, and drop the hostname from the parent
Gateway. To migrate instead of remove, move `basicAuth.users` into the
per-service `SecurityPolicy` alongside `jwt`.

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
- `route-grpc.yaml` — gRPC `GRPCRoute`(s): Loki OTLP write; Tempo read + OTLP write
- `securitypolicy.yaml` — one `SecurityPolicy` per route for Loki and Mimir (single SP covers all HTTP routes); Tempo requires one SP per route because each `GRPCRoute` must have its own `SecurityPolicy`
- `filters.yaml` — shared `HTTPRouteFilter` resources (headers-check and path rewrite where applicable)

**Operational Considerations:**

- All routes share the same hostname and `X-Scope-OrgID` enforcement
- JWT providers (`auth.jwt.providers`) are shared across all services
- JWT validation is done inline by Envoy Gateway — no external auth service required
- gRPC routes (`GRPCRoute`) do not support `HTTPRouteFilter` via `ExtensionRef`, so missing `X-Scope-OrgID` on gRPC requests results in a no-route rejection rather than a strict 401

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
