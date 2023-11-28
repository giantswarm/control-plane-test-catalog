[![CircleCI](https://dl.circleci.com/status-badge/img/gh/giantswarm/kyverno-policy-operator/tree/main.svg?style=svg)](https://dl.circleci.com/status-badge/redirect/gh/giantswarm/kyverno-policy-operator/tree/main)

# kyverno-policy-operator chart

Giant Swarm offers a kyverno-policy-operator App which can be installed in workload clusters.
Here we define the kyverno-policy-operator chart with its templates and default configuration.

Kyverno Policy Operator reconciles the Giant Swarm PolicyException instances and creates the necessary Kyverno PolicyExceptions objects.

A Giant Swarm PolicyException consists of a list of Policies and Targets to be excluded from the Kyverno Policy Engine. Having a Giant Swarm PolicyException will ensure that workloads targeted by that policy won't be blocked during Admission time by Kyverno.

### Sample Giant Swarm PolicyException:

In some cases, it may be necessary to exempt specific resources from the enforcement of Kyverno policies, such as `disallow-privilege-escalation` and `require-run-as-nonroot`. To achieve this, you can create a Giant Swarm PolicyException. Below is an example of how to exclude the `my-custom-operator` Deployment in the `default` namespace from these policies:

```yaml
apiVersion: policy.giantswarm.io/v1alpha1
kind: PolicyException
metadata:
  name: my-custom-operator
  namespace: policy-exceptions
spec:
  policies:
  - disallow-privilege-escalation
  - require-run-as-nonroot
  targets:
  - kind: Deployment
    names:
    - my-custom-operator
    namespaces:
    - default
```

This Policy Exception configuration will be detected by the Kyverno Policy Operator, which will create a corresponding Kyverno Policy Exception resource:

```yaml
apiVersion: kyverno.io/v2alpha1
kind: PolicyException
metadata:
  labels:
    app.kubernetes.io/managed-by: kyverno-policy-operator
  name: my-custom-operator
  namespace: policy-exceptions
(...)
spec:
  background: false
  exceptions:
  - policyName: require-run-as-nonroot
    ruleNames:
    - run-as-non-root
    - autogen-run-as-non-root
    - autogen-cronjob-run-as-non-root
  - policyName: disallow-privilege-escalation
    ruleNames:
    - privilege-escalation
    - autogen-privilege-escalation
    - autogen-cronjob-privilege-escalation
  match:
    any:
    - resources:
        kinds:
        - Deployment
        - ReplicaSet
        - Pod
        names:
        - my-custom-operator*
        namespaces:
        - default
```

## Installing

There are several ways to install this app onto a workload cluster.

- [Using GitOps to instantiate the App](https://docs.giantswarm.io/advanced/gitops/apps/)
- [Using our web interface](https://docs.giantswarm.io/platform-overview/web-interface/app-platform/#installing-an-app).
- By creating an [App resource](https://docs.giantswarm.io/use-the-api/management-api/crd/apps.application.giantswarm.io/) in the management cluster as explained in [Getting started with App Platform](https://docs.giantswarm.io/getting-started/app-platform/).

## Configuring

### values.yaml

**This is an example of a values file you could upload using our web interface.**

```yaml
# Set the PolicyExceptions destination namespace
policyOperator:
  destinationNamespace: ""
```

### Sample App CR and ConfigMap for the management cluster

If you have access to the Kubernetes API on the management cluster, you could create
the App CR and ConfigMap directly.

Here is an example that would install the app to
workload cluster `abc12`:

```yaml
# appCR.yaml
apiVersion: application.giantswarm.io/v1alpha1
kind: App
metadata:
  name: kyverno-policy-operator
  namespace: demo01
spec:
  catalog: giantswarm-playground-test
  config:
    configMap:
      name: demo01-cluster-values
      namespace: demo01
  name: kyverno-policy-operator
  namespace: kyverno-policy-operator
  version: 0.0.1
```

```yaml
# user-values-configmap.yaml
policyOperator:
  destinationNamespace: "policy-exceptions"
```

See our [full reference on how to configure apps](https://docs.giantswarm.io/getting-started/app-platform/app-configuration/) for more details.

## Compatibility

This app has been tested to work with the following workload cluster release versions:

- v19.1.0

## Limitations

This App needs Kyverno App [v0.15+](https://github.com/giantswarm/kyverno-app) to be installed in the cluster. 
