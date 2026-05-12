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
           tokenName: payments-bot
```

One `tbot` Deployment per `RemoteApp` (per-app blast-radius isolation). Under
the kubernetes-join model (ADR 0004) the operator renders a per-CR
`ServiceAccount`; the rendered tbot pod authenticates to Teleport using that
SA's projected JWT, validated against the `static_jwks` pinned on the
Teleport `ProvisionToken`. No static-token `Secret` is delivered to the
consumer cluster. See [`CONTEXT.md`](./CONTEXT.md) for the full design and
[`docs/adr/0004-kubernetes-join-method.md`](./docs/adr/0004-kubernetes-join-method.md)
for the join-method decision.

## Scope

`RemoteApp` covers Teleport **Application Service** apps (TCP/HTTP) only.
Database, Kubernetes, and SSH access are out of scope.
