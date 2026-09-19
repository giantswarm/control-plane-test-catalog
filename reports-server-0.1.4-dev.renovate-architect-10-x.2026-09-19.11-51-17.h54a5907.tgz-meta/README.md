[![CircleCI](https://dl.circleci.com/status-badge/img/gh/giantswarm/reports-server-app/tree/main.svg?style=svg)](https://dl.circleci.com/status-badge/redirect/gh/giantswarm/reports-server-app/tree/main)
[![OpenSSF Scorecard](https://api.securityscorecards.dev/projects/github.com/giantswarm/reports-server-app/badge)](https://securityscorecards.dev/viewer/?uri=github.com/giantswarm/reports-server-app)

# Kyverno Report Server chart

Kyverno Reports Server is an extension API server which serves certain Kubernetes API server requests via API aggregation. It is used to support storing PolicyReports in an external database instead of in etcd. Removing PolicyReports from etcd [is desirable for a number of stability and performance reasons](https://github.com/kyverno/KDP/pull/51), but keeping them available through the Kubernetes API keeps them readily available for use by report producers/consumers as well as end users.

**Why did we add it?**

Giant Swarm uses this app to manage the lifecycle of the Kyverno Reports Server as part of our platform. This chart is slightly opinionated for Giant Swarm's use case. Others are welcome to use it, but the official charts should be considered definitive.

## Installing

There are several ways to install this app onto a workload cluster:

- [Using GitOps to instantiate the App](https://docs.giantswarm.io/advanced/gitops/apps/)
- [Using our web interface](https://docs.giantswarm.io/platform-overview/web-interface/app-platform/#installing-an-app).
- By creating an [App resource](https://docs.giantswarm.io/use-the-api/management-api/crd/apps.application.giantswarm.io/) in the management cluster as explained in [Getting started with App Platform](https://docs.giantswarm.io/getting-started/app-platform/).

## Configuring

See our [full reference on how to configure apps](https://docs.giantswarm.io/getting-started/app-platform/app-configuration/) for more details.

## Credit

- <https://github.com/kyverno/reports-server>
