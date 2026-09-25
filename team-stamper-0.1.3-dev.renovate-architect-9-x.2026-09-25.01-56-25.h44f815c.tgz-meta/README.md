[![CircleCI](https://dl.circleci.com/status-badge/img/gh/giantswarm/team-stamper/tree/main.svg?style=svg)](https://dl.circleci.com/status-badge/redirect/gh/giantswarm/team-stamper/tree/main)

# Team Stamper Controller

This controller is in charge of adding the `application.giantswarm.io/team` annotation to each HelmRelease CR refering to OCIRepository CR, providing information of apps  ownership. HelmRelease CRs that are not using OCIRepository CRs as app sources are not supported.

Value of the annotation is then translated by the Kube State Metrics into emitted metrics label, which is then used for routing app-related alerts to the appropriate teams.
