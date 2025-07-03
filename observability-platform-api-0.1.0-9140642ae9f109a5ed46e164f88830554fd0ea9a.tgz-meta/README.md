[![CircleCI](https://dl.circleci.com/status-badge/img/gh/giantswarm/observability-platform-api/tree/main.svg?style=svg)](https://dl.circleci.com/status-badge/redirect/gh/giantswarm/observability-platform-api/tree/main)


# Observability Platform API

Giant Swarm provides an Observability Platform API as the main entry point to its Observability Platform platform.

**What is this app?**

This app contains a set of ingresses to access the Observability Platform API components. This app is made to be deployed by Giant Swarm close to the Observability Platform components to allow customer access to the platform components from outside their management clusters (e.g. self-hosted Grafana, external log shipper)

## Installing

There are several ways to install this app onto a workload cluster.

- [Using GitOps to instantiate the App](https://docs.giantswarm.io/tutorials/continuous-deployment/apps/add-appcr/)
- By creating an [App resource](https://docs.giantswarm.io/reference/platform-api/crd/apps.application.giantswarm.io) using the platform API as explained in [Getting started with App Platform](https://docs.giantswarm.io/tutorials/fleet-management/app-platform/).
