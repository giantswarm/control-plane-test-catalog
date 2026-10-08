# kyverno-policies-connectivity

This repository contains Kyverno policies which Giant Swarm uses for managing connectivity in our clusters.

## Repository structure

We implement an app according to the [general Giant Swarm app platform](https://docs.giantswarm.io/app-platform/) which relies on Helm for application management.

The `policies` folder contains the policies which are then escaped to be compliant with helm specific syntax.
We use `[[` and  `]]` delimiters to handle cases where variables should be managed by helm.

The `hack` folder contains scripts which are used during local development and in CI.
These scripts enable us to easily set up a local testing environment.

## Development

There are only very few prerequisites for local testing:
1. `make` has to be installed
2. `kubectl` has to be installed
3. `kind` has to be installed
4. [chainsaw](https://kyverno.github.io/chainsaw/) has to be installed

To only generate the policies in the `helm` folder structure:
```bash
make generate
```

### Adding tests

Tests are [Chainsaw](https://kyverno.github.io/chainsaw/) tests in [`tests/chainsaw`](tests/chainsaw).
Each test has its own folder with a `chainsaw-test.yaml`.
Reusable steps, like checking that a policy is ready, live in [`tests/chainsaw/_steps-templates`](tests/chainsaw/_steps-templates).
See [`check-policy-ready`](tests/chainsaw/check-policy-ready/chainsaw-test.yaml) for an example.

### Tilt
You can use Tilt for fast feedback loops.

First create the local `kind` cluster
```shell
make kind-create
```

Then you just need to start `tilt`
```shell
make tilt-up
```
