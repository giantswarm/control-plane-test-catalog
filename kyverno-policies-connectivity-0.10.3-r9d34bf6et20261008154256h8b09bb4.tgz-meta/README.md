# kyverno-policies-connectivity

This repository contains Kyverno policies which Giant Swarm uses for managing connectivity in our clusters.

## Repository structure

We implement an app according to the [general Giant Swarm app platform](https://docs.giantswarm.io/app-platform/) which relies on Helm for application management.

The `policies` folder contains the policies which are then escaped to be compliant with helm specific syntax.
We use `[[` and  `]]` delimiters to handle cases where variables should be managed by helm.

The `hack` folder contains scripts which are used during local development and in CI,
like `template.sh` for `make generate` and `chainsaw-extra-resources.sh` for `make install-extras`.

## Development

There are only very few prerequisites for local testing:
1. `make` has to be installed
2. `kubectl` has to be installed
3. `kind` has to be installed
4. `helm` has to be installed
5. `docker` has to be running for `kind`
6. `curl` and `sed` have to be installed
7. [chainsaw](https://kyverno.github.io/chainsaw/) has to be installed

To only generate the policies in the `helm` folder structure:
```bash
make generate
```

### Running tests

Create a `kind` cluster, install Kyverno and the policies, then run the tests:
```bash
make kind-create install-kyverno install-extras install-policies
chainsaw test ./tests/chainsaw
```

`make kind-create` creates a cluster named `chainsaw-kyverno-cluster` and switches your current `kubectl` context to it.
The Kubernetes and Kyverno versions match CI. Override them with `KUBERNETES_VERSION` and `KYVERNO_VERSION`.

After changing a policy, run `make generate install-policies` before running the tests again.

`make kind-create` and `make install-kyverno` only work on a fresh cluster.
To start over, delete the cluster and run all four targets again.

Delete the cluster when you are done:
```bash
kind delete cluster --name chainsaw-kyverno-cluster
```

### Adding tests

Tests are [Chainsaw](https://kyverno.github.io/chainsaw/) tests in [`tests/chainsaw`](tests/chainsaw).
Each test has its own folder with a `chainsaw-test.yaml`.
Reusable steps, like checking that a policy is ready, live in [`tests/chainsaw/_steps-templates`](tests/chainsaw/_steps-templates).
See [`check-policy-ready`](tests/chainsaw/check-policy-ready/chainsaw-test.yaml) for an example.

