# SLO

WARNING: this is a POC run by team phoenix. Do not use for production reasons.

This repository contains the SLO definition for Giant Swarm services.

## How to add a service

- Locate your team's directory within the `areas` subfolders (if missing, please add it). Example: `area/kaas/teams/phoenix`.
- Create a new directory to hold the SLO definitions for the service you want to monitor.
- Create a README.md file in the new direcory describing your SLO. You can use https://github.com/dastergon/sreworkbook-templates-md/blob/master/slo-document.md and other services in this repo as a template.
- Create an `slos` directory inside the service directory created in the previous step.
- Add one or more `.yaml` files holding the following data:

```yaml
---
description: "Description of that this SLO is for"
alertName: "NameOfTheAlertAsItWillAppearInOpsgenie"
objective: 99.5 # SLO value expressed as a percentage
errorQuery: rate(vector(0)[{{..window}}]) by (cluster_id) # query that returns the number of error occourences. Always use the `by (cluster_id)` clause if this metric is related to k8s clusters and use `{{.window}}` placeholder for time intervals. 
totalQuery: rate(vector(0)[{{..window}}]) by (cluster_id) # query that returns the total number of events (errors and successes). Always use the `by (cluster_id)` clause if this metric is related to k8s clusters and use `{{.window}}` placeholder for time intervals.
```

Finally run `make generate` to generate your SLO CRs. Output will be in the `output` directory.
