[![CircleCI](https://circleci.com/gh/giantswarm/tunnelport.svg?&style=shield)](https://circleci.com/gh/giantswarm/tunnelport)

# tunnelport

A Kubernetes operator that wraps Teleport's `tbot` + a `Service` behind a single
`RemoteApp` CR, so workloads on a consumer management cluster can reach a
Teleport-exposed app as if it were a local `Service` — no Teleport SDK in caller
code.

## What it does

A platform engineer writes one `RemoteApp`. The operator renders a `tbot`
`Deployment` (which dials Teleport and exposes an `application-tunnel` on a
local port) plus a `ClusterIP` `Service` pointing at it. Workloads `curl
http://<remoteapp-name>:<port>` and traffic is tunneled to the app behind
Teleport.

```
  Consumer MC                          Central MC          Producer MC
  ┌──────────────────────────────┐     ┌──────────┐        ┌────────────────┐
  │                              │     │ Teleport │        │ teleport-kube- │
  │  caller pod                  │     │  proxy   │        │   agent (app   │
  │     │                        │     │   +      │        │     mode)      │
  │     │ http://payments:8080   │     │  auth    │        │       │        │
  │     ▼                        │     └────▲─────┘        │       ▼        │
  │  Service (ClusterIP) ──┐     │          │              │  backend app   │
  │                        │     │          │ mTLS         └────────────────┘
  │                        ▼     │          │ tunnel               ▲
  │                   tbot pod ──┼──────────┘──────────────────────┘
  │                   (rendered  │
  │                    by this   │
  │                    operator) │
  └──────────────────────────────┘
            ▲
            │ owns Deployment + Service + ConfigMap
            │
       RemoteApp CR  (access.giantswarm.io/v1alpha1)
         spec:
           appName: payments
           port: 8080
           proxyAddr: teleport.example:443
```

One `tbot` Deployment per `RemoteApp` (per-app blast-radius isolation). The
operator renders a per-CR `ServiceAccount`; tbot joins Central via the
`kubernetes` join method, presenting that SA's projected JWT against a
per-`RemoteApp` `TeleportProvisionToken` on Central (`kubernetes.type:
static_jwks`, per ADR 0006). No token Secrets to rotate, no operator-side
credential plumbing. See [`CONTEXT.md`](./CONTEXT.md) for the full design.

## Scope

`RemoteApp` covers Teleport **Application Service** apps (TCP/HTTP) only.
Database, Kubernetes, and SSH access are out of scope.
