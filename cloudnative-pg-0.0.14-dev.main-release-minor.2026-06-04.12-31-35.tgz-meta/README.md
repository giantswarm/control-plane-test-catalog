[![CircleCI](https://dl.circleci.com/status-badge/img/gh/giantswarm/cloudnative-pg-app/tree/main.svg?style=svg)](https://dl.circleci.com/status-badge/redirect/gh/giantswarm/cloudnative-pg-app/tree/main)
[![OpenSSF Scorecard](https://api.securityscorecards.dev/projects/github.com/giantswarm/edgedb-app/badge)](https://securityscorecards.dev/viewer/?uri=github.com/giantswarm/edgedb-app)

# cloudnative-pg chart

This repository contains a Giant Swarm App for installing CloudNative PostgreSQL (a.k.a. CloudNative PG or `cnpg`) in a Giant Swarm cluster.

**Why did we add it?**

Several capabilities of the Giant Swarm security platform benefit from using PostgreSQL as a database. CloudNative PostgreSQL is an operator which manages the lifecycle of PostgreSQL databases deployed within Kubernetes clusters. This app packages the `cnpg` operator for use by our platform.

**Who can use it?**

The official CNPG charts are well-maintained and should be preferred over this repository. This chart is slightly opinionated for Giant Swarm's use case and intended for our internal use, but others are welcome to use it if they find it helpful.

Note to Giant Swarm customers: CNPG is used internally in our platform, and not currently offered as a managed app. You are welcome to use this chart if it suits your needs, but it will not be monitored and support will be on a best-effort basis.

## Installing

There are several ways to install this app onto a workload cluster.

- [Using GitOps to instantiate the App](https://docs.giantswarm.io/advanced/gitops/apps/)
- [Using our web interface](https://docs.giantswarm.io/platform-overview/web-interface/app-platform/#installing-an-app).
- By creating an [App resource](https://docs.giantswarm.io/use-the-api/management-api/crd/apps.application.giantswarm.io/) in the management cluster as explained in [Getting started with App Platform](https://docs.giantswarm.io/getting-started/app-platform/).

## Configuring

See our [full reference on how to configure apps](https://docs.giantswarm.io/getting-started/app-platform/app-configuration/) for more details.

## Credit

- <https://github.com/cloudnative-pg/charts>
- <https://github.com/cloudnative-pg/cloudnative-pg>
