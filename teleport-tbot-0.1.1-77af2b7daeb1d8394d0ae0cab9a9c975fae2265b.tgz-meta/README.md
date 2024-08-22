# teleport-tbot app

Helm chart for the teleport-tbot app running in Giant Swarm clusters.

`teleport-tbot` is an agent designed to use [Teleport Machine ID](https://goteleport.com/docs/enroll-resources/machine-id/getting-started/) to provide machines with an identity that can authenticate to a Teleport cluster. This enables secure access to registered resources such as Kubernetes clusters and more.

## How it works?

Currently, teleport-tbot only works with `kubernetes` join method. To set it up, follow these steps:
1. Create provision token for the bot, [see example here](https://github.com/giantswarm/teleport-fleet/blob/main/kubernetes/shared/templates/bot-glippy-token.yaml)
2. Create bot user pointing to above token and roles (e.g: teleport-operator, teleport-tbot).
3. In values.yaml, set `teleport.tokenName` to above bot token name.

## Use cases at Giant Swarm

- **Cluster E2E Test Suites**: Used to verify cluster registration with the Teleport cluster.
- **Access to Private E2E Clusters**: Used to access private E2E cluster using tbot generated kubeconfig.
- **Integration with [teleport-operator](https://github.com/giantswarm/teleport-operator)**: Used for join token management for cluster and node registration.
