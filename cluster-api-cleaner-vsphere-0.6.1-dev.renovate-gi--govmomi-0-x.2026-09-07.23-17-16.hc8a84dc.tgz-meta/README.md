[![CircleCI](https://dl.circleci.com/status-badge/img/gh/giantswarm/cluster-api-cleaner-vsphere/tree/master.svg?style=svg)](https://dl.circleci.com/status-badge/redirect/gh/giantswarm/cluster-api-cleaner-vsphere/tree/master)
[![image@quay](https://quay.io/repository/giantswarm/cluster-api-cleaner-vsphere/status "image@quay")](https://quay.io/repository/giantswarm/cluster-api-cleaner-vsphere)
[![docker.io pulls](https://img.shields.io/docker/pulls/giantswarm/cluster-api-cleaner-vsphere.svg)](https://hub.docker.com/r/giantswarm/cluster-api-cleaner-vsphere)
[![License: MIT](https://img.shields.io/badge/License-Apache_2.0-yellow.svg)](https://opensource.org/licenses/Apache-2.0)

# cluster-api-cleaner-vsphere

A helper operator for CAPV to delete resources created by apps in workload clusters.

Currently, it only cleans Cloud Native Storage->Container Volumes.
