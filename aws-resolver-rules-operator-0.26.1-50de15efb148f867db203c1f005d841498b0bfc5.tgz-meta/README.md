# Controllers

## Resolver rules controller

This controller reconciles `AWSClusters`. It only reconciles `AWSClusters` using the private DNS mode, set by the `aws.giantswarm.io/dns-mode` annotation.
It creates a Resolver Rule for the workload cluster k8s API endpoint and associates it with the DNS Server VPC.
That way, other clients using the DNS Server will be able to resolve the workload cluster k8s API endpoint.

## Unpause controller

This controller reconciles `AWSClusters`. Our AWSClusters start paused in some situations, and we use this controller to unpause the clusters when certain conditions are met.
It will unpause the `Cluster` and `AWSCluster` when `VPC` and `Subnet` conditions are marked as ready.
It also requires the `ResolverRules` condition to be ready if using the private dns mode.

## DNS controller

This controller reconciles `Clusters`. It creates a hosted zone that could be a private or a public one depending on the DNS mode of the workload cluster.
The mode is selected using the `aws.giantswarm.io/dns-mode` annotation on the `AWSCluster` CR. It also creates three DNS records in the hosted zone:

- `api`: a dns record of type `A` pointing to the control plane Load Balancer
- `*`: a CNAME pointing to the `ingress.$basedomain` record

When the mode is public, it creates a record set in the parent's hosted zone so that dns delegation works.

When the mode is private, the hosted zone is associated with a list of VPCs that can be specified using the `aws.giantswarm.io/dns-assign-additional-vpc` annotation
on the `AWSCluster` CR.

When a workload cluster is deleted, the hosted zone is deleted, together with the delegation on the parent zone.

# Testing

You can run all tests with

```
make test-all
```

## Unit tests

You can run only the unit tests with

```
make test-unit
```

## Integration tests

You can run only the integration tests with

```
make test-integration
```

This project uses [`LocalStack`](https://github.com/localstack/localstack) for integration tests. Remember that you can use the `aws` cli targetting the local [`LocalStack`](https://github.com/localstack/localstack) environment with
```
$ aws --endpoint=http://localhost:4566 route53 list-hosted-zones-by-name
```

## AWS account setup

To run the integration and acceptance tests against an AWS account you need to create an IAM role. To do that, first source the `aws-resolver-rules-operator-test-secrets.sh` file from LastPass and then run the following commands. Note that you also need to target the account you are setting up when using the aws cli and update the desired WC or MC account env var in the LastPass secret afterwards.

```shell
source aws-resolver-rules-operator-test-secrets.sh
aws iam create-policy --policy-name tests-aws-resolver-rules-operator --policy-document file://tests/assets/test-role-policy.json

policy_file=$(mktemp)
envsubst <tests/assets/test-role-trust-policy.json >$policy_file
aws iam create-role --role-name tests-aws-resolver-rules-operator --assume-role-policy-document file://$policy_file
aws iam attach-role-policy --role-name ttests-aws-resolver-rules-operator --policy-arn arn:aws:iam::$AWS_ACCOUNT:policy/tests-aws-resolver-rules-operator
rm $policy_file
```
