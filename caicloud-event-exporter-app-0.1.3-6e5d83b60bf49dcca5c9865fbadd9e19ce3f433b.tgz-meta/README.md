# caicloud-event-exporter-app

[caicloud/event_exporter](https://github.com/caicloud/event_exporter) packaged
as a Giant Swarm app. It exports Kubernetes events as Prometheus metrics.
[prometheus-rules](https://github.com/giantswarm/prometheus-rules) depends on
this exporter to provide metrics for Crossplane alerts.

## Updating the app

To release a new version or introduce changes, please refer to
[CONTRIBUTING.md](CONTRIBUTING.md). Upstream YAML file is a base for the chart,
with all the extras and patches added by
[kustomization.yaml](hack/kustomization.yaml) in `hack/` directory.

## Metrics Overview

> Excerpt from upstream README.md

1. `kube_event_count` Count of kubernetes event that was seen for the last hour. The metric value is the same as the count property of `Event` object in the cluster.
   ```
   kube_event_count{involved_object_kind="Deployment",involved_object_name="event-exporter",involved_object_namespace="default",name="event-exporter.1640452bd04fc7bf",namespace="default",reason="ScalingReplicaSet",source="/deployment-controller",type="Normal"} 1
   ```
2. `kube_event_unique_events_total` Total number of kubernetes unique event that happened for the last hour.
   ```
   kube_event_unique_events_total{involved_object_kind="Deployment",involved_object_name="event-exporter",involved_object_namespace="default",name="event-exporter.1640452bd04fc7bf",namespace="default",reason="ScalingReplicaSet",source="/deployment-controller",type="Normal"} 1
   ```
3. `event_exporter_version`Information of the event exporter that was built
   ```
   event_exporter_build_info{branch="v1.0",build_date="2020-10-22T10:11:29Z",build_user="Caicloud Authors",go_version="go1.13.15",version="v1.0.0"} 1
   ```
