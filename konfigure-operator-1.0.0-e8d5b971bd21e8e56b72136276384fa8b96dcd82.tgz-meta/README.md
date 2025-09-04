# konfigure-operator

testing...
testing...
testing...

Generate configurations based on the Giant Swarm configuration system.

This operator is a thin wrapper around the configuration generator tool [konfigure](https://github.com/giantswarm/konfigure).

## Description

The purpose of this operator is to provide a way to generate ConfigMaps and Secrets used as configuration input for
any interfaces. The result manifest can be mounted to Giant Swarm App CRs, Flux HelmReleases or anything else.

## Supported configurations

### Konfiguration

This CRD is used to generate configurations for the schema-based Generalized Configuration System.

The source for the generation is fetched from a Flux GitRepository that should point to a repository that contains
the configuration code conforming to the referenced schema.

Encryption support consists of SOPS. The operator automatically fetches SOPS keys from the cluster that has the
`konfigure.giantswarm.io/data: sops-keys` label.

Generated configurations are applied to a destination namespace. The `.metadata.name` of these resources will be
generated according to the following rules:

- the core of it is the iteration name, the keys in the `.spec.targets.iterations` map
- if `.spec.destination.naming.prefix` is present, it will be prepended to the iteration name with a `-` character
  - this value must start and end with an alphanumeric character, can contain `-` characters within
  - must be no longer than 64 characters
- if `.spec.destination.naming.suffix` is present, it will be appended to the iteration name with a `-` character
  - this value must start and end with an alphanumeric character, can contain `-` characters within
  - must be no longer than 64 characters
- if `.spec.destination.naming.useSeparator` is set to false, the `-` character will be omitted from gluing the prefix
  and the suffix from the iteration name

 The reason for these restrictions is that ConfigMap and Secret `.metadata.name` field must follow:
 https://kubernetes.io/docs/concepts/overview/working-with-objects/names/#dns-subdomain-names.


Here is an example:

```yaml
apiVersion: konfigure.giantswarm.io/v1alpha1
kind: Konfiguration
metadata:
  name: example-1
  namespace: default
spec:
  targets:
    schema:
      reference:
        name: konfigurationschema-example
        namespace: default
    defaults:
      variables:
        - name: installation
          value: golem
    iterations:
      app-operator:
        variables:
          - name: app
            value: app-operator
      chart-operator:
        variables:
          - name: app
            value: chart-operator
      dex-operator:
        variables:
          - name: app
            value: dex-operator
  destination:
    namespace: default
    naming:
      suffix: example-1
  reconciliation:
    interval: 5m
    retryInterval: 1m
  sources:
    flux:
      gitRepository:
        name: "giantswarm-config"
        namespace: "flux-giantswarm"
```
#### Breakdown and explanation

This section contains explanation for using `Konfiguration` CRs to generate configs.

##### .spec.targets

This section describes what to render.

The `.schema` field contains information on the schema to use to interpret the referenced git repository and to use to
render the listed targets. The `.reference` field must point to an existing `KonfigurationSchema` CR, for example:

```yaml
apiVersion: konfigure.giantswarm.io/v1alpha1
kind: KonfigurationSchema
metadata:
  name: konfigurationschema-example
  namespace: default
spec:
  raw:
    remote:
      url: https://raw.githubusercontent.com/giantswarm/konfiguration-schemas/refs/heads/main/schemas/management-cluster-configuration/schema.yaml
```

The `.raw.remote.url` field should point to an accessible Generalized Configuration System schema file.

Read more on how schemas for the Generalized Configuration systems work in the
[konfigure](https://github.com/giantswarm/konfigure/blob/main/README.md) repository.

Alternatively, a `KonfigurationSchema` can provide the full contents of the schema under `.spec.raw.content`.

A schema can take variables to make complex layers and structures and decide on which paths in the tree to render
based on values of those variables. The `.defaults.variables` field contains `.name` and `.value` pairs that will
be used in each target to render. Individual iterations can optionally provide overrides for these given defaults.

The `.iterations` field is a map of targets to render. Each key of the map acts as a unique name for that iteration.
This name will be used throughout the reconciliation loop to render the resulting manifests and refer back to the
iteration in case of errors or any other cases.

Each iteration has a `.variables` field that can provide further variables to the schema-based rendering process. These
variables will be merged on top of the default variables and passed down to `konfigure` along with the schema and the
fetched source to render the desired targets.

##### .destination

This section contains information on where to apply the generated manifests and how to name them.

Please note that there is collision detection implemented within the operator logic to avoid managing a single ConfigMap
or Secret by multiple configuration rendering CRs. Also, if a generated manifest overwrites an existing manifest
not considered to be managed by the operator, apply will fail stating that the target already exists.

Resources managed by the operator are considered exclusive to the operator. Any changes to them will be overwritten by
consecutive reconciliations.

All generated resources will also be applied the following labels:

```yaml
# A ConfigMap or Secret is considered to be managed by the operator if this label and key pair is present on it
configuration.giantswarm.io/generated-by: konfigure-operator

# The following labels are used to identify the config CR that was used to generate the manifest
configuration.giantswarm.io/ownerApiGroup: konfigure.giantswarm.io
configuration.giantswarm.io/ownerApiVersion: v1alpha1
configuration.giantswarm.io/ownerKind: Konfiguration
configuration.giantswarm.io/ownerName: gauss-konfiguration
configuration.giantswarm.io/ownerNamespace: giantswarm

# This label is used to present which revision on the source was used to generate this manifest
configuration.giantswarm.io/revision: 38be874bfa3d627bf70366bd3ae43ff9dcfb4fcf
```

Apps fail to render or apply will simply be skipped over and will be presented as a failure on the CR status without
blocking the generation of other matched apps. Any failure being present will mark the whole CR as failed in the Ready
condition.

> ⚠️ Please note that "transactions" are not supported at the moment, meaning that if configmap and a secret are generated
> successfully, we do not try to atomically apply both of them to the server. First the config map is applied, then the
> secret. If let's say the secret apply fails, we do not revert the config map. Such a scenario will mark the iteration
> as failed tho in `.status.failures`.

##### .reconciliation

This section controls when the next reconciliation should kick in for successful reconciliations, meaning all matched
apps were correctly generated and applied. This is controlled by `.interval`. Failure re-scheduling can be configured
with `.retryInterval`. Both accept Go duration formats, see: https://pkg.go.dev/time.

##### .sources

This section contains information on the source that should be used to generate the configurations.

Currently on Flux GitRepository sources are supported that should point to assembled Giant Swarm config repositories.

The `.flux.gitRepository` section should contain the `.name` and `.namespace` of the Flux GitRepository resource to fetch.

###### Temporarily disabling reconciliation of generated config maps and secret

The label `configuration.giantswarm.io/reconcile` - not supported as annotation - can be added with value `disabled` to
generated config maps and secrets to disable `konfigure-operator` updating them on each reconciliation.

```yaml
configuration.giantswarm.io/reconcile: disabled
```

Disabled resources will be marked on the `Konfiguration` CR's `.status.disabled` field. The `.name` field of each object
references the iteration name.

```yaml
status:
  disabled:
  - name: konfigure-operator
    kind: ConfigMap
    target:
      name: konfigure-operator-konfiguration
      namespace: giantswarm
  - name: konfigure-operator
    kind: Secret
    target:
      name: konfigure-operator-konfiguration
      namespace: giantswarm
```

#### Status

Example status can be used to diagnose issues with the configuration CR.

Here is an example of a failure scenario:

```yaml
status:
  conditions:
    - lastTransitionTime: "2025-03-12T15:06:07Z"
      message: 'Attempted revision: c8f73a3b5ad0ddaad337d78f4e49ea8eae49d2a7'
      observedGeneration: 4
      reason: ReconciliationFailed
      status: "False"
      type: Ready
  failed:
    - name: app-operator
      message: secrets "app-operator-example" already exists
    - name: aws-operator
      message: 'failed to render template from "default/apps/aws-operator/configmap-values.yaml.template":
      template: main:75:37: executing "main" at <.workloadCluster.ssh.ssoPublicKey>:
      map has no entry for key "ssoPublicKey"'
  lastAppliedRevision: 9eb2f00e201df4f9d2b1e3a15e870e2b911726ab
  lastAttemptedRevision: c8f73a3b5ad0ddaad337d78f4e49ea8eae49d2a7
  lastReconciledAt: "2025-03-12T15:06:07.572012492Z"
  observedGeneration: 4
```

The `.failed` section contains a list of apps with their name and a message that describes where the process for it failed.
The `.name` field of each object references the iteration name.

We only support the `Ready` condition at the moment. If all iterations rendered and applied fine, it will be
marked as `ReconciliationSucceeded`.

If there are any failures, the CR will be marked as `ReconciliationFailed `and will be retried indefinitely
each `.spec.reconciliation.retryInterval`.

It can happen that the generation fails before starting even. For example when the SOPS keys cannot be fetched or
the source cannot be downloaded, e.g. because it does not exist or the `source-controller` URL is invalid or inaccessible.
In such scenarios the `Ready` condition will be marked as `SetupFailed`.

The `.lastAppliedRevision` field contains the revision of the source when all matched CRs were last successfully generated
and applied. The `.lastAttemptedRevision` is the source revision used during the last reconciliation of the resource that
occurred at `.lastReconciledAt` and at generation `.observedGeneration`.

> ℹ️ Please note, that currently the operator is not subscribed to event of Flux `source-controller`. An update on the
> source will not trigger a reconciliation. The intervals purely depend on `.spec.reconciliation` of the CR and the
> outcome of the last reconciliation loop.

## Development

To generate CRDs, run the following commands:

```shell
make api

# Recommended to match the generated code to static checks performed on CI
goimports -local github.com/giantswarm/konfigure-operator -w .
```

Finally, please stage and commit the changes.
