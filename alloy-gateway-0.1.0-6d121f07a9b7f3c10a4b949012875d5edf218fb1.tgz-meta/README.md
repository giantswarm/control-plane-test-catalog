[![CircleCI](https://dl.circleci.com/status-badge/img/gh/giantswarm/alloy-gateway-app/tree/main.svg?style=svg)](https://dl.circleci.com/status-badge/redirect/gh/giantswarm/alloy-gateway-app/tree/main)

[Read me after cloning this template (GS staff only)](https://handbook.giantswarm.io/docs/dev-and-releng/app-developer-processes/adding_app_to_appcatalog/)

# alloy-gateway-app chart

Giant Swarm offers a alloy-gateway-app App which can be installed in workload clusters.
Here we define the alloy-gateway-app chart with its templates and default configuration.

**What is this app?**

This App is a work around used to deploy alloy as a gateway on management clusters until our CI tooling supports deploying an application with a custom name as this is mostly installing the alloy app with the name alloy-gateway.

**Why did we add it?**

Check out this to know more https://github.com/giantswarm/roadmap/issues/3568

**Who can use it?**

This application is not meant to be deployed on workload clusters.
