# teleport-tbot app

Helm chart for the teleport-tbot app running in Giant Swarm clusters for internal use only.

- **Cluster E2E Test Suites**: Used to verify cluster registration with the Teleport cluster.
- **Access to Private E2E Clusters**: Used to access private E2E cluster using tbot generated kubeconfig.
- **Integration with [teleport-operator](https://github.com/giantswarm/teleport-operator)**: Used for join token management for cluster and node registration.

## What is Teleport Bot?

`teleport-tbot` is an agent designed to use [Teleport Machine ID](https://goteleport.com/docs/enroll-resources/machine-id/getting-started/) to provide machines with an identity that can authenticate to a Teleport cluster. This enables secure access to registered resources such as Kubernetes clusters and more.

>[!IMPORTANT]
> Currently, this Helm chart is not general purpose and only works with `kubernetes` join method specific to Giant Swarm clusters. To set it up, follow these steps:
> 1. Create provision token for the bot, [see example here](https://github.com/giantswarm/teleport-fleet/blob/main/kubernetes/shared/templates/bot-glippy-token.yaml)
> 2. Create bot user pointing to above token and roles (e.g: teleport-operator, teleport-tbot).
> 3. In values.yaml, set `teleport.tokenName` to above bot token name and `enabled: true`.

## Release process

Please follow the standard Giant Swarm release process.

1. Merge you changes to `main` branch, ensuring the CHANGELOG is updated.
2. Create a release branch named `main#release#VERSION`, where VERSION can be `major`, `minor`, or `patch`.
3. Merge the Release PR.

## Credit
- https://github.com/gravitational/teleport
