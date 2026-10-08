[![CircleCI](https://dl.circleci.com/status-badge/img/gh/giantswarm/aws-vpc-operator/tree/main.svg?style=svg)](https://dl.circleci.com/status-badge/redirect/gh/giantswarm/aws-vpc-operator/tree/main)

# aws-vpc-operator

Operator for managing AWS VPCs for Cluster API AWS workload clusters.

We use this operator to reconcile AWSClusters using the private VPC mode, because in those cases we need a custom VPC, different from what CAPA would create.
Clusters choose the private VPC mode using this annotation on the `AWSCluster` CR:

```yaml
aws.giantswarm.io/vpc-mode: private
```
