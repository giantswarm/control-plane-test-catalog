[![CircleCI](https://circleci.com/gh/giantswarm/macropower-analytics-panel-server-app.svg?style=shield)](https://circleci.com/gh/giantswarm/macropower-analytics-panel-server-app)

# macropower-analytics-panel-server chart

Giant Swarm offers a macropower-analytics-panel-server App which can be installed in workload clusters.
Here we define the macropower-analytics-panel-server chart with its templates and default configuration.

**What is this app?**

This app is used to export grafana analytics information to prometheus.

**Why did we add it?**

We added this app because we want to add analytics to our management clusters grafana to know which dashboards are used the most internally and by customers.

**Who can use it?**

Every Grafana user.

## Installing

There are 3 ways to install this app onto a workload cluster.

1. [Using our web interface](https://docs.giantswarm.io/ui-api/web/app-platform/#installing-an-app)
2. [Using our API](https://docs.giantswarm.io/api/#operation/createClusterAppV5)
3. Directly creating the [App custom resource](https://docs.giantswarm.io/ui-api/management-api/crd/apps.application.giantswarm.io/) on the management cluster.

## Credit

* https://github.com/MacroPower/macropower-analytics-panel
