[![CircleCI](https://circleci.com/gh/giantswarm/security-pack-helper.svg?style=shield)](https://circleci.com/gh/giantswarm/security-pack-helper)

# security-pack-helper chart

The Giant Swarm platform includes a number of security capabilities supported by other open-source projects in the Kubernetes / cloud-native ecosystem.
We try to stay as close as possible to the official, standard implementations of each tool we use behind the scenes, but it is sometimes necessary for us to support modifications or temporary "glue," for example to mitigate known issues while waiting for new releases.

This repo contains an operator which we run alongside our other security components to apply custom logic to their behaviors when necessary.
