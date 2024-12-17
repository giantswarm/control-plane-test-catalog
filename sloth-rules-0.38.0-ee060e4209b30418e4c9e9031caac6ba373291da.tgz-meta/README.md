[![CircleCI](https://dl.circleci.com/status-badge/img/gh/giantswarm/sloth-rules/tree/main.svg?style=svg&circle-token=dd0a8499874cd79f634d33b9d52b6ad912093741)](https://dl.circleci.com/status-badge/redirect/gh/giantswarm/sloth-rules/tree/main)

# SLO

This repository contains the SLO definition for Giant Swarm services.

## How to create a SLO

- Locate your team's directory within the `areas` subfolders (if missing, please add it). Example: `areas/platform/teams/phoenix`.
- Create a new directory to hold the SLO definitions for the service you want to monitor or use the existing one.
- Create a README.md file in the new direcory describing your SLO. You can use https://github.com/dastergon/sreworkbook-templates-md/blob/master/slo-document.md and other services in this repo as a template.
- Create an `slos` directory inside the service directory created in the previous step.
- Add one or more `.yaml` files holding the following data:

```yaml
---
description: "Description of that this SLO is for"
alertName: "NameOfTheAlertAsItWillAppearInOpsgenie"
objective: 99.5 # SLO value expressed as a percentage
errorQuery: rate(vector(0)[{{.window}}]) by (cluster_id) # query that returns the number of error occourences. Always use the `by (cluster_id)` clause if this metric is related to k8s clusters and use `{{.window}}` placeholder for time intervals. 
totalQuery: rate(vector(0)[{{.window}}]) by (cluster_id) # query that returns the total number of events (errors and successes). Always use the `by (cluster_id)` clause if this metric is related to k8s clusters and use `{{.window}}` placeholder for time intervals.
alertLabels:
  cancel_if_wathever: "true"
annotations:
  opsrecipe: whatever/
# To deploy and run the rules only on some specific clusters (can also be set to workload_cluster)
cluster_type: management_cluster
```

:warning: Please add the `cancel_if_outside_working_hours: "true"` label to your SLO at least for the 1st week after you released it to make sure it works as intended and doesn't page too much. :warning:

Finally run `make generate` to generate your SLO CRs. Output will be in the `output` directory.

### Limtiations

- Labels cannot be changed currently, as this will heavily disrupt all SLO rules evaluation (see https://github.com/giantswarm/roadmap/issues/2579).
  You should rename the alert when you need to change labels for one.

  If you have changed a label, you will have to silence the recordingrule for 30 days (until old labels are purged). See [an example silence PR](https://github.com/giantswarm/silences/pull/1179).

## SLO definition guidelines

- The resulting prometheusRules generated from your SLO will use the following query : `errorQuery/totalQuery` so keep in mind that **the result of your errorQuery divided by your totalQuery must be a decimal value comprised between 0 and 1**.
- Both `errorQuery` and `totalQuery` must use the `[{{.window}}]` interval. This means that if you need your `totalQuery` to return a static value (e.g 1), you will have to create a query based on a metric returning a static value.
Instead of having `vector(1)` you can for example use `sum(rate(kube_pod_container_info{container=~"whatever"}[{{.window}}]))`
- You can and should **add an opsrecipe annotation** to your SLO.
- Always test your SLO on a testing installation before creating a new repo release to make sure it pages right when the conditions are met.
- You can create a SLO withouth alerts by setting `pageAlert.disable` and/or `ticketAlert.disable` as `true`. By default, both alerts are created.

## Architecture

The following graph show how the SLOs are deployed and used in a MC :

![image](images/sloth-architecture.png)

`sloth-rules` is deployed as an app in the MC and generates the `prometheusservicelevel` (PSL) CRDs corresponding to the SLOs defined in the `areas` folder. From there, `sloth` deployed as a pod by the [sloth-app](https://github.com/giantswarm/sloth-app) app acts as an operator and generates `prometheusRules` from those PSL objects which are thus usable by prometheus.

Whenever someone creates a SLO, the end result is a `prometheusRule` which defines several recording rules, all using the same query : `errorQuery/totalQuery` but evaluated on different intervals (5m, 30m, 1hr...). The object also defines 2 alerting rules, one with `severity: page` which will page the oncall person on OpsGenie and the other one with `severity: ticket` which will only send a message in the team's alert channel on Slack.

For more details concerning `Sloth` architecture, please visit the [official website](https://sloth.dev/introduction/architecture/)

## Sloth-rules repo logic

When using `Sloth` out of the box, you have to define the PSL files yourself, which is not that hard, but can be tedious. To ease your SLO definition process, the following process happens to allow you to define SLOs as shown in the first section of this README :

![image](images/sloth-rules.png)
