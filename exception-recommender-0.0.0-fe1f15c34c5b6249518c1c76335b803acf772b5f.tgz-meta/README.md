# exception-recommender chart

The `exception-recommender` Helm chart creates PolicyExceptionDrafts that can be used as a base model for Kyverno PolicyExceptions. The Drafts are created from PolicyReports for the "Pod Security Standard" Policy categories. 

## Installing

There are several ways to install this app onto a workload cluster.

- [Using GitOps to instantiate the App](https://docs.giantswarm.io/advanced/gitops/apps/)
- [Using our web interface](https://docs.giantswarm.io/platform-overview/web-interface/app-platform/#installing-an-app).
- By creating an [App resource](https://docs.giantswarm.io/use-the-api/management-api/crd/apps.application.giantswarm.io/) in the management cluster as explained in [Getting started with App Platform](https://docs.giantswarm.io/getting-started/app-platform/).

## Configuring

### values.yaml

**This is an example of a values file you could upload using our web interface.**

```yaml
recommender:
  # Install PolicyExceptionDrafts on the default namespace
  destinationNamespace: "default"
```

See our [full reference on how to configure apps](https://docs.giantswarm.io/getting-started/app-platform/app-configuration/) for more details.
