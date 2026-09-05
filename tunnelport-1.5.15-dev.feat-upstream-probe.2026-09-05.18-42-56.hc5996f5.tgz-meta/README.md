[![CircleCI](https://circleci.com/gh/giantswarm/tunnelport.svg?&style=shield)](https://circleci.com/gh/giantswarm/tunnelport)

# tunnelport

A Kubernetes operator that wraps Teleport's `tbot` + a `Service` behind a single
`RemoteApp` CR, so workloads on a consumer management cluster can reach a
Teleport-exposed app as if it were a local `Service` — no Teleport SDK in caller
code.

## How it works

A platform engineer writes one `RemoteApp`. The operator reconciles it into a
`ServiceAccount`, a `ConfigMap` carrying `tbot`'s rendered config, a
`Deployment` (one `tbot` container, one `ghostunnel` sidecar) and a
`ClusterIP` `Service`. Workloads on the consumer MC call
`https://<remoteapp-name>:8443` and traffic ends up at the backing app on a
producer MC.

Three concerns are independent and worth keeping separate: how packets flow,
how `tbot` proves who it is, and how the tunnel Service authenticates itself
to its callers.

### Networking

The `Service` resolves to the per-`RemoteApp` pod. Inside the pod,
`ghostunnel` accepts the inbound TLS connection and forwards plaintext over
`127.0.0.1:<spec.port>` to `tbot`'s `application-tunnel`. `tbot` holds an
open mTLS application tunnel to Teleport central, which relays the request
to a `teleport-kube-agent` (Application Service mode) on a producer MC, and
from there to the backing service.

```
  Consumer MC                            Central MC         Producer MC
  ┌────────────────────────────────┐    ┌──────────┐       ┌────────────────┐
  │                                │    │ Teleport │       │ teleport-kube- │
  │  caller pod                    │    │  proxy   │       │   agent        │
  │     │                          │    │   +      │       │   (app mode)   │
  │     │ https://payments:8443    │    │  auth    │       │       │        │
  │     ▼                          │    └────▲─────┘       │       ▼        │
  │  Service (ClusterIP, 8443)     │         │             │  backend app   │
  │     │                          │         │ mTLS        └────────────────┘
  │     ▼                          │         │ app tunnel        ▲
  │  RemoteApp pod ────────────────┼─────────┴───────────────────┘
  │   (tbot + ghostunnel sidecar)  │
  └────────────────────────────────┘
```

One `tbot` Deployment per `RemoteApp` — `tbot` is one identity per process,
so multi-tunnel-per-`tbot` is not on the table, and the per-pod boundary
keeps blast radius scoped to a single app. The `Service` stays `ClusterIP`
on purpose; `LoadBalancer` / `NodePort` would defeat the local-only framing.

### Auth: how tbot joins Teleport

`tbot` proves its identity to Teleport using the `kubernetes` join method.
For each `RemoteApp` the operator renders a dedicated `ServiceAccount` in
the CR's namespace, plus a projected `serviceAccountToken` volume whose
`audience` is the Teleport cluster name. `tbot` reads that JWT, presents it
on every join, and Teleport central's `ProvisionToken` validates the JWT
against the consumer MC's signing keys embedded inline (`static_jwks`).

```
  Consumer MC                                      Central MC
  ┌──────────────────────────────────────────┐    ┌────────────────────────┐
  │                                          │    │                        │
  │  ┌── RemoteApp pod ───────────────────┐  │    │  Teleport auth         │
  │  │  tbot                              │  │    │                        │
  │  │   reads projected SA JWT and       │  │    │  ProvisionToken        │
  │  │   presents it as join credential   ├──┼───►│    join: kubernetes    │
  │  │   (audience = clusterName)         │  │    │    trust: static_jwks  │
  │  └────────────────────────────────────┘  │    │    allow: <ns>:<name>  │
  │           ▲ uses                         │    │                        │
  │           │                              │    │  validates JWT against │
  │   ServiceAccount  (one per RemoteApp,    │    │  the consumer MC's     │
  │                    rendered by operator) │    │  JWKS embedded in the  │
  │                                          │    │  ProvisionToken        │
  └──────────────────────────────────────────┘    └────────────────────────┘
```

Consequences worth pulling out:

- No static-token `Secret` is ever delivered to the consumer cluster. The
  operator holds no `secrets` RBAC and watches no rotating object.
- The kubelet refreshes the projected JWT transparently; `tbot` consumes
  the rotated token on its next join. The operator does nothing on
  rotation.
- `static_jwks` is chosen because consumer-MC kube-apiservers are private
  in the GS topology, so the `in_cluster` mode (which would have Teleport
  reach the consumer's `tokenreviews` endpoint) is not viable. Exporting
  the consumer MC's JWKS to central is a one-off platform-team GitOps
  step, out of scope for this operator.
- The `ProvisionToken`'s `kubernetes.allow` list pins
  `<cr.namespace>:<cr.name>`, so the CR name and namespace must be agreed
  before central-side provisioning lands.

The Teleport cluster name and proxy address are operator-level chart values
(`teleport.clusterName`, `teleport.proxyAddr`), not CR fields. A given
consumer MC therefore hosts `RemoteApp`s that all target the same Teleport
cluster; multi-Teleport on one MC is an explicit non-goal (the answer is a
second operator install in its own namespace).

### TLS: how callers trust the tunnel

The `ghostunnel` sidecar terminates TLS on `8443` using a SPIFFE X.509-SVID
minted by `tbot`'s `workload-identity-x509` service, signed by Teleport
central's SPIFFE CA. The SVID and its private key live in an `emptyDir`
shared with the sidecar; `ghostunnel` watches the files and reloads on
rotation. Callers verify the SVID against a single trust-bundle `Secret`
(`tunnelport-spiffe-bundle`) materialised by a chart-managed singleton
`tunnelport-trust-bundle` Deployment.

```
  Consumer MC                                            Central MC
  ┌──────────────────────────────────────────────┐      ┌──────────────┐
  │                                              │      │   Teleport   │
  │  caller pod                                  │      │   SPIFFE CA  │
  │     │  https://payments:8443                 │      │              │
  │     │  (verifies server SVID against the     │      │  signs every │
  │     │   svid_bundle.pem file it mounts       │      │  SVID below  │
  │     ▼   from tunnelport-spiffe-bundle)       │      │              │
  │  Service (ClusterIP, tls/8443)               │      └──────▲───────┘
  │     │                                        │             │
  │     ▼                                        │             │
  │  ┌── RemoteApp pod ─────────────────┐        │             │
  │  │  ghostunnel  ── presents SVID    │        │             │
  │  │       ▲       from emptyDir      │        │             │
  │  │       │                          │        │             │
  │  │  tbot ── mints SVID via          │────────┼─────────────┤
  │  │          workload-identity-x509  │        │             │
  │  └──────────────────────────────────┘        │             │
  │                                              │             │
  │  ┌── tunnelport-trust-bundle ─────┐          │             │
  │  │   (chart singleton)            │          │             │
  │  │  tbot ─► Secret:               │──────────┼─────────────┘
  │  │   tunnelport-spiffe-bundle     │          │
  │  │   (svid_bundle.pem)            │          │
  │  └────────────────────────────────┘          │
  └──────────────────────────────────────────────┘
```

Why this shape:

- Every `tbot` pod on the MC already extends transitive trust to Teleport
  central's CA chain — the join contract makes that true. Reusing the same
  identity authority for the tunnel cert avoids standing up a parallel CA
  managed by cert-manager or a manual bootstrap process.
- An SVID carries a meaningful workload identity
  (`spiffe://<cluster>/bot/<bot-name>`) in addition to DNS SANs. Callers
  that want to verify the SPIFFE ID, not just the chain, get richer
  attestation for free.
- One bundle covers every `RemoteApp` on the MC because they're all signed
  by the same Teleport SPIFFE CA. Per-CR Secrets would carry identical
  bytes; the singleton tbot writes the bundle once and every consumer in
  the release namespace mounts it.

A plaintext `http/8080` port is still served on the `Service` alongside
`tls/8443` during migration; it's deprecated and a future change will drop
it once every known caller has moved.

### Observability: is the tunnel actually usable?

Everything a Kubernetes-native operator can passively observe about a
tunnel is a proxy for the question callers care about, and all of those
proxies can be green while the answer is no. In
[giantswarm/giantswarm#37521][gap] 40 tunnels served SVIDs whose
`dns_sans` had not followed a namespace rename: `tbot` joined, the SVID
was issued, `ghostunnel` bound `:8443`, the sidecar's `TCPSocket`
readiness probe connected — because a TCP connect never looks at a
certificate — and every caller failed hostname verification for two days.

So the operator asks the tunnel directly. Every
`verification.interval` (default `2m`, leader-elected so one replica
does it) it dials each `RemoteApp` whose tbot pods are Ready at
`<name>.<namespace>.svc.<clusterDomain>:<tls.port>` with `ServerName` set
to that FQDN, and verifies the served chain against the SPIFFE trust
bundle it mounts from `tunnelport-spiffe-bundle`. That is the same check
`curl --cacert` performs, which is to say the same check a caller
performs.

A verified handshake still stops at the consumer-side listener. In
[giantswarm/tunnelport#110][upstream] a Teleport app service whose
connection to its auth server had gone stale answered every new session
with `504 Gateway Timeout` for thirteen minutes; ghostunnel forwarded the
requests faithfully, and four RemoteApps stayed Ready and Verified
throughout. So on that same verified session the operator then sends one
request *through* the tunnel — `GET /`, or `spec.probe.path` — via tbot,
the Teleport proxy and the app service to the app, and looks at what
comes back.

The outcomes show up here:

| Where | What |
|---|---|
| `RemoteApp.status.conditions[TunnelVerified]` | `True` / `False` / `Unknown`, with the specific X.509 fault as the message. Also the `Verified` column of `kubectl get remoteapp`. |
| `tunnelport_remoteapp_tls_verification{remoteapp_name,remoteapp_namespace,result}` | `1` for the current result. Scraped via the chart's `PodMonitor` and alerted on by its `PrometheusRule`. |
| `RemoteApp.status.conditions[UpstreamReachable]` | `True` for any HTTP status but a gateway failure (200, a 401 from an OAuth resource server, 404 all count); `False` with reason `UpstreamUnreachable` for 502/503/504 or no response within `verification.upstream.timeout`; `Unknown` when no request was sent. The message carries the status, the probed URL and, while failing, the time of the last good probe. **Folds into `Ready`** and `status.ready`; also the `Upstream` column under `-o wide`. |
| `tunnelport_remoteapp_upstream_probe_status{remoteapp_name,remoteapp_namespace,result}` | The HTTP status the far end answered with (`0` for none), `result` one of `reachable` / `unreachable`. No series for tunnels that were not probed. |
| Kubernetes Events on the `RemoteApp` | `Warning UpstreamUnreachable` when the far end stops answering, `Normal UpstreamReachable` when it answers again. |

`result` on the TLS series is one of `verified`, `cert_invalid`
(connected, no verified TLS session), `unreachable` (nothing accepted a
connection) or `not_ready` (the tunnel makes no claim to be serving, so it
was not probed). `cert_invalid` and `unreachable` are deliberately
separate: they have different first moves, and collapsing them would put
the SAN-drift failure back in the same bucket as an ordinary outage.

Three properties worth knowing:

- **"Cannot verify" is never reported as "the certificate is bad,"** and
  "no request was sent" is never reported as "nobody answered". With no
  readable trust bundle the operator has no opinion on any certificate, so
  it publishes none and flips `tunnelport_tls_verification_available` to
  `0` — which `TunnelPortTLSVerificationUnavailable` alerts on. A tunnel
  whose handshake failed gets `UpstreamReachable=Unknown`, not `False`. A
  check that fails silently would be the same class of bug as the one it
  exists to find.
- **`TunnelVerified` does not feed `status.ready`; `UpstreamReachable`
  does.** `Ready=True` with `Verified=False` is a legitimate and highly
  actionable state with its own alert. A tunnel whose far end answers
  nothing but 504 is simply not usable, and `Ready` is where its consumers
  — muster's MCPServer, `kubectl get remoteapp` — look; during #110 they
  looked there and saw green. `Ready=False` with reason
  `UpstreamUnreachable` names the layer.
- **One connection per RemoteApp per round, spread out.** The request rides
  the TLS probe's session, and each round schedules its probes at random
  offsets within `verification.jitter` (default `30s`), so ~40 tunnels do
  not open 40 Teleport app sessions at the same instant every two minutes.

The mechanism needs no additional RBAC beyond `create`/`patch` on
`events.k8s.io` Events — it reads the `RemoteApp` and Pod lists it already
watches and the trust bundle from a mounted file, never a `Secret` through
the API server. Turn the whole thing off with `verification.enabled=false`,
or only the request through the tunnel with
`verification.upstream.enabled=false` (which also returns `Ready` to
join-level). It is off automatically when there is no bundle to verify
against: `trustBundle.enabled=false` and no
`verification.trustBundleSecretName`.

[gap]: https://github.com/giantswarm/giantswarm/issues/37521
[upstream]: https://github.com/giantswarm/tunnelport/issues/110

## Scope

`RemoteApp` covers Teleport **Application Service** apps (TCP/HTTP) only.
Database, Kubernetes, and SSH access are out of scope.
