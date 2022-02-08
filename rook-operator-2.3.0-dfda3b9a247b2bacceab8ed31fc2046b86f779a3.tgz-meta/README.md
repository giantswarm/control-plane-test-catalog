[![CircleCI](https://circleci.com/gh/giantswarm/rook-operator-app.svg?style=shield)](https://circleci.com/gh/giantswarm/rook-operator-app)

# rook-operator chart

This App can be used in several different deployment scenarios, but its main use is
to expose storage in a KVM-based Management Cluster to customer workloads which are
running in a Workload Cluster.

If you wish to make use of this App in your Giant Swarm clusters, please talk to your
Account Engineer who will guide you through the requirements.

**What is this app?**

This app deploys the [rook operator](https://github.com/rook/rook/) to Giant Swarm clusters
(both Management and Workload clusters).

**Why did we add it?**

Using Rook to provide Ceph storage abstracts storage provisioning away from the underlying
infrastructure and improves the end-user experience when interacting with storage.

**Who can use it?**

Anyone may use this app, however it is directly intended to be used in Giant Swarm on-premise
clusters, and as such it is customised specifically for this purpose.

## Limitations

Some apps have restrictions on how they can be deployed.
Not following these limitations will most likely result in a broken deployment.

* This App will only be available for KVM on-premise clusters.
* The cluster namespace (usually `rook-ceph`) **must** exist before the App is installed.

## Credit

* https://github.com/rook/rook/
