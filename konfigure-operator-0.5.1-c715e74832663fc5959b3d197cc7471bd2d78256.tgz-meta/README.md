# konfigure-operator

Generate configurations based on the Giant Swarm configuration system.

This operator is a thin wrapper around the configuration generator tool [konfigure](https://github.com/giantswarm/konfigure).

## Description

The purpose of this operator is to provide a way to generate ConfigMaps and Secrets used as configuration input for
any interfaces. The result manifest can be mounted to Giant Swarm App CRs, Flux HelmReleases or anything else.

## Supported configurations

### ManagementClusterConfiguration

This CRD is used to generate management cluster app configurations.

Based on a set of filters, a set of applications are selected for generation for the given management cluster.

The source for the generation is fetched from a Flux GitRepository that should point to an assembled Giant Swarm
configuration repository.

Encryption support is consisted of SOPS. The operator automatically fetches SOPS keys from the cluster that has the
`konfigure.giantswarm.io/data: sops-keys` label.

Generated configurations are applied to a destination namespace. The `.metadata.name` of these resources will be
generated according to the following rules:

- the core of it is the app name as is present in e.g. `shared-configs` folder `default/apps`
- if `.spec.destination.naming.prefix` is present, it will be prepended to the app name with a `-` character
  - this value must start and end with an alphanumeric character, can contain `-` characters within
  - must be no longer than 64 characters
- if `.spec.destination.naming.suffix` is present, it will be appended to the app name with a `-` character
  - this value must start and end with an alphanumeric character, can contain `-` characters within
  - must be no longer than 64 characters
- if `.spec.destination.naming.useSeparator` is set to false, the `-` character will be omitted from gluing the prefix
  and the suffix from the app name

 The reason for these restrictions is that ConfigMap and Secret `.metadata.name` field must follow:
 https://kubernetes.io/docs/concepts/overview/working-with-objects/names/#dns-subdomain-names.


Here is an example:

```yaml
apiVersion: konfigure.giantswarm.io/v1alpha1
kind: ManagementClusterConfiguration
metadata:
  name: example-1
  namespace: default
spec:
  configuration:
    applications:
      includes:
        exactMatchers:
          - app-operator
          - chart-operator
        regexMatchers:
          - kyverno.*
      excludes:
        exactMatchers:
          - kyverno-policies-dx
        regexMatchers: []
    cluster:
      name: golem
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
      service:
        url: "source-controller.flux-giantswarm.svc"
```
#### Breakdown and explanation

This section contains explanation for using `ManagementClusterConfiguration` CRs to generate configs.

##### .spec.configuration

The `.applications` section contains information on which application to render configuration for:

- both the `.includes` and the `.excludes` section contains exact and regex matchers.
  - the regex matchers work with regular expressions following: https://github.com/google/re2/wiki/Syntax
  - the filtering logic works in the following way:
    - list all possible apps in the fetched source (e.g. all folders in `shared-configs` folder: `default/apps`)
    - add all apps from the full set that matches any `.spec.configuration.applications.includes.exactMatchers` to the result set
    - add all apps from the full set that matches any of the `.spec.configuration.applications.includes.regexMatchers`
    - remove all apps from the result set that matches any `.spec.configuration.applications.excludes.exactMatchers`
    - remove all apps from the result set that matches any `.spec.configuration.applications.excludes.regexMatchers`
    - the result set is finished
- app listed in `.includes.exactMatchers` that are not present in all possible apps, will be listed on the CR status as a miss

The `.cluster` section contains information on which management cluster overrides should be applied on top of the default
configuration. A folder named `.cluster.name` must exist in the assembled source under the `installations` folder. Under that,
the `apps` folder must contain the app specific folder with their respective configuration override templates.

##### .destination

This section contains information on where to apply the generated manifests and how to name them.

Please note that there is collision detection implemented within the operator logic to avoid managing a single ConfigMap
or Secret by multiple configuration rendering CRs. Also, if a generated manifest would overwrite an existing manifest
that is not considered to be managed by the operator, apply will fail stating that the target already exists.

Resources managed by the operator are considered exclusive to the operator. Any changes to them will be overwritten by
consecutive reconciliations.

All generated resources will also be applied the following labels:

```yaml
# A ConfigMap or Secret is considered to be managed by the operator if this label and key pair is present on it
configuration.giantswarm.io/generated-by: konfigure-operator

# The following labels are used to identify the config CR that was used to generate the manifest
configuration.giantswarm.io/ownerApiGroup: konfigure.giantswarm.io
configuration.giantswarm.io/ownerApiVersion: v1alpha1
configuration.giantswarm.io/ownerKind: ManagementClusterConfiguration
configuration.giantswarm.io/ownerName: gauss-configuration
configuration.giantswarm.io/ownerNamespace: giantswarm

# This label is used to present which revision on the source was used to generate this manifest
configuration.giantswarm.io/revision: 38be874bfa3d627bf70366bd3ae43ff9dcfb4fcf
```

Apps fail to render or apply will simply be skipped over and will be presented as a failure on the CR status without
blocking the generation of other matched apps. Any failure being present will mark the whole CR as failed in the Ready
condition.

> ⚠️ Please note that "transactions" are not supported at the moment, meaning that if a configmap and a secret is generated
> successfully, we do not try to atomically apply both of them to the server. First the config map is applied, then the
> secret. If let's say the secret apply fails, we do not revert the config map. Such scenario will mark the app as failed
> tho in `.status.failures`.

##### .reconciliation

This section controls when the next reconciliation should kick in for successful reconciliations, meaning all matched
apps were correctly generated and applied. This is controlled by `.interval`. Failure re-scheduling can be configured
with `.retryInterval`. Both accept Go duration formats, see: https://pkg.go.dev/time.

##### .sources

This section contains information on the source that should be used to generate the configurations.

Currently on Flux GitRepository sources are supported that should point to assembled Giant Swarm config repositories.

The `.flux.gitRepository` section should contain the `.name` and `.namespace` of the Flux GitRepository resource to fetch.

The `.flux.service.url` field can point to any Flux `source-contoller` service URL. If not set, it is defaulted to the
standard Flux deployment: `source-controller.flux.svc`.

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
  failures:
    - appName: app-operator
      message: secrets "app-operator-example" already exists
    - appName: aws-operator
      message: 'failed to render template from "default/apps/aws-operator/configmap-values.yaml.template":
      template: main:75:37: executing "main" at <.workloadCluster.ssh.ssoPublicKey>:
      map has no entry for key "ssoPublicKey"'
  lastAppliedRevision: 9eb2f00e201df4f9d2b1e3a15e870e2b911726ab
  lastAttemptedRevision: c8f73a3b5ad0ddaad337d78f4e49ea8eae49d2a7
  lastReconciledAt: "2025-03-12T15:06:07.572012492Z"
  misses:
    - no-such-operator
  observedGeneration: 4
```

The `.failures` section contains a list of apps with their name and a message that describes where the process for it failed.

The `.misses` section is informative on the app names that were listed in `.includes.exactMatchers` that were not matched
against any of the possible apps. This is merely use to indicate if maybe a type is in an app name or the config repo does
not have that app, in order to avoid silently not generating configs for an expected app. Misses will not mark the CR
as failed.

We only support the `Ready` condition at the moment. If all app configurations generated and applied fine, it will be
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

In order to generate CRDs, run the following commands:

```shell
make api

# Recommended to match the generated code to static checks performed on CI
goimports -local github.com/giantswarm/konfigure-operator -w .
```

Finally, please stage and commit the changes.
