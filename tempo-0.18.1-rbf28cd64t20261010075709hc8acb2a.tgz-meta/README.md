# Tempo App

[![CircleCI](https://circleci.com/gh/giantswarm/tempo-app.svg?style=shield)](https://circleci.com/gh/giantswarm/tempo-app)

Giant Swarm offers Grafana Tempo as a [managed app](https://docs.giantswarm.io/changes/managed-apps/). This chart provides a distributed tempo setup based on this
[upstream chart](https://github.com/grafana/helm-charts/blob/main/charts/tempo-distributed).
It tunes some options from upstream to make the chart easier to deploy.

This chart is meant to be used with S3 compatible storage only. Access to the S3
storage must be ensured for the chart to work.
* Check [below](#deploying-on-aws) to see what configuration you need on the AWS side.
* or [below](#deploying-on-azure) to see what configuration you need on the Azure side.

**Table of Contents:**

- [Requirements](#requirements)
- [Install](#install)
- [Upgrading](#upgrading)
- [Configuration](#configuration)
- [Limitations](#limitations)
- [Links](#links)
- [Credit](#credits)

## Requirements

* You need to ensure that pods deployed can access S3 storage (as explained above).

## Install

There are several ways to install this app onto a workload cluster.

- [Using GitOps to instantiate the App](https://docs.giantswarm.io/advanced/gitops/#installing-managed-apps)
- [Using our web interface](https://docs.giantswarm.io/ui-api/web/app-platform/#installing-an-app).
- By creating an [App resource](https://docs.giantswarm.io/ui-api/management-api/crd/apps.application.giantswarm.io/) in the management cluster as explained in [Getting started with App Platform](https://docs.giantswarm.io/app-platform/getting-started/).

## Upgrading

### Upgrading an existing Release to a new major version

A major chart version change (like v0.5.0 -> v1.0.0) indicates that there is an incompatible breaking change needing manual actions.

Versions before v1.0.0 are not stable, and can even have breaking changes between "minor" versions. (like v0.5.0 -> v0.6.0)

#### Rollback

You can rollback to your previous Tempo version, and see your old traces.
However, because of multi-tenancy, seeing traces that were stored with the new version may require some config tweaking.

## Configuration

As this application is build upon the Grafana tempo upstream chart as a dependency, most of the values to override can be found [here](https://github.com/grafana/helm-charts/blob/main/charts/tempo-distributed/values.yaml).

Some samples can be found [here](./sample_configs/)

### General recommendations

The number of `replicas` in the [default values file](https://github.com/giantswarm/tempo-app/blob/main/helm/tempo/values.yaml) are generally considered safe.
If you reduce the number of `replicas` below the default recommended values, expect undefined behaviour and problems.

### Prepare config file

1. Create app config file
Grab the included [sample config file](https://github.com/giantswarm/tempo-app/blob/main/sample_configs/values-gs-aws.yaml)
or [azure sample config file](https://github.com/giantswarm/tempo-app/blob/main/sample_configs/values-gs-azure.yaml),
read the comments for options and adjust to your needs. To check all available
options, please consult the [full `values.yaml` file](https://github.com/giantswarm/tempo-app/blob/main/helm/tempo/values.yaml).

#### Multi-tenant setup

TODO: Not ready yet

### Deploying on AWS

The recommended deployment mode is using S3 storage mode. Assuming your cluster
has `IRSA` , `cert-manager` and `external-dns` included, you should be good to use
the instructions below to setup S3 bucket and the necessary permissions in your
AWS account.

Make sure to create this config for the *cluster* where you are deploying Tempo, and not at installation-level.

#### Prepare AWS S3 storage.

Create a new private S3 bucket based in the same region
as your instances. Ex. `gs-tempo-storage`.
* consider creating private VPC endpoint for S3 - traffic volume might be
  considerable and this might save you some money for the transfer fees,
* it is recommended to use S3 bucket class for frequent access (`S3 standard`),
* create a retention policy for the bucket.
* CLI procedure:
```bash
# prepare environment
export CLUSTER_NAME=zj88t
export REGION=eu-central-1
export INSTALLATION=gorilla
export BUCKET_NAME=gs-tempo-storage-"$CLUSTER_NAME" # must be globally unique
export AWS_PROFILE=gorilla-atlas # your AWS CLI profile
export TEMPO_POLICY="$BUCKET_NAME"-policy
export TEMPO_ROLE="$BUCKET_NAME"-role

# create bucket
aws --profile="$AWS_PROFILE" s3 mb s3://"$BUCKET_NAME" --region "$REGION"
```

#### Prepare AWS IAM policy.

Create an IAM Policy in IAM. If you want to use AWS WebUI, copy/paste the contents of `POLICY_DOC` variable.
```bash
# Create policy
POLICY_DOC='{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket",
                "s3:PutObject",
                "s3:GetObject",
                "s3:DeleteObject" ],
            "Resource": [
                "arn:aws:s3:::'"$BUCKET_NAME"'",
                "arn:aws:s3:::'"$BUCKET_NAME"'/*"
            ]
        },
        {
            "Sid": "VisualEditor1",
            "Effect": "Allow",
            "Action": [
                "s3:GetAccessPoint",
                "s3:GetAccountPublicAccessBlock",
                "s3:ListAccessPoints"
            ],
            "Resource": "*"
        }
    ]
}'
aws --profile="$AWS_PROFILE" iam create-policy --policy-name "$TEMPO_POLICY" --policy-document "$POLICY_DOC"
```

#### Prepare AWS IAM role

Giant Swarm uses IRSA (Iam Roles for Service Accounts) to allow pods to access S3 buckets' resources. For more details concerning IRSA, you can refer to the [official documentation](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) as well as to the [giant swarm one](https://docs.giantswarm.io/advanced/iam-roles-for-service-accounts).

This means that the role's `Trust Relationship` will be different that the one used for KIAM (cf above) :
```bash
PRINCIPAL_ARN="$(aws --profile="$AWS_PROFILE" iam get-role --role-name "$CLUSTER_NAME"-IAMManager-Role | sed -n 's/.*Arn.*"\(arn:.*\)".*/\1/p')"
ROLE_DOC='{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::'$PRINCIPAL_ARN':oidc-provider/irsa.'$CLUSTER_NAME'.k8s.'$INSTALLATION'.'$REGION'.aws.gigantic.io"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "irsa.'$CLUSTER_NAME'.k8s.'$INSTALLATION'.'$REGION'.aws.gigantic.io:sub": "system:serviceaccount:tempo:tempo"
                }
            }
        }
    ]
}'
```

#### Create role

Everything is now set to create the role :
```bash
aws --profile="$AWS_PROFILE" iam create-role --role-name "$TEMPO_ROLE" --assume-role-policy-document "$ROLE_DOC"
# Attach the policy to the role
TEMPO_POLICY_ARN="${PRINCIPAL_ARN%:role/*}:policy/$TEMPO_POLICY"
aws --profile="$AWS_PROFILE" iam attach-role-policy --policy-arn "$TEMPO_POLICY_ARN" --role-name "$TEMPO_ROLE"
```

* Store the role's arn in a variable for the next step :
```bash
TEMPO_ROLE_ARN="${PRINCIPAL_ARN%:role/*}:role/$TEMPO_ROLE"
```

#### Link IAM role to Kubernetes

Since IRSA is relying on the use of service accounts to grant access rights to the pods, you don't have to manually create the `tempo` namespace as you won't have to annotate it. Instead, you'll have to edit the Chart's values under the `tempo` section with the following :
```bash
serviceAccount:
  create: true
  name: tempo
  annotations:
    eks.amazonaws.com/role-arn: "$TEMPO_ROLE_ARN"
```

This way, all pods using the `tempo` service account will be able to access to the S3 bucket created earlier.

#### Install the app

* Fill in the values from previous step in your config (`values.yaml`) file:
  * role annotation for S3
  * cluster ID
  * node pool ID
  * and your custom setup

* Install the app using your values.
  Don't forget to use the same namespace as you prepared above for the installation.

### Deploying on Azure

#### Gather data

Find the 'Subscription name' (usually named after your installation) name and the 'Resource group' of your cluster (usually named after cluster id) inside your 'Azure subscription'
* list subscriptions:
```
az account list -otable
export SUBSCRIPTION_NAME="your subscription"
```
* list resource groups:
```
az group list --subscription "$SUBSCRIPTION_NAME" -otable
export RESOURCE_GROUP="your resource group"
```

#### object storage setup
1. Create 'Storage Account' on Azure ([How-to](https://docs.microsoft.com/en-us/azure/storage/common/storage-account-create)) ['Create storage account'](https://portal.azure.com/#create/Microsoft.StorageAccount)
    * 'Account kind' should be 'BlobStorage'
    * Example with Azure CLI:
```
# Chose your storage account name
export STORAGE_ACCOUNT_NAME="tempo$RESOURCE_GROUP"
# then create it
az storage account create \
     --subscription "$SUBSCRIPTION_NAME" \
     --name "$STORAGE_ACCOUNT_NAME" \
     --resource-group "$RESOURCE_GROUP" \
     --sku Standard_GRS \
     --encryption-services blob \
     --https-only true \
     --kind BlobStorage \
     --access-tier Hot
```
(It may be required to set the location using the `--location` flag.)

2. Create a 'Blob service' 'Container' in your storage account
    * Example on how to do it with Powershell in Azure portal:
```
export CONTAINER_NAME="$STORAGE_ACCOUNT_NAME"container
az storage container create \
     --subscription "$SUBSCRIPTION_NAME" \
     -n "$CONTAINER_NAME" \
     --public-access off \
     --account-name "$STORAGE_ACCOUNT_NAME"
```

3. Go to the 'Access keys' page of your 'Storage account'
    * Use the 'Storage account name' for `azure_storage.account_name`
    * Use the name of the 'Blob service' 'Container' for `azure_storage.blob_container_name`
    * Use one of the keys for `azure.storage_key`
    * With azure CLI
```
az storage account keys list \
     --subscription "$SUBSCRIPTION_NAME" \
     --account-name "$STORAGE_ACCOUNT_NAME" \
| jq -r '.[]|select(.keyName=="key1").value'
```

#### Install the app

* Fill in the values from previous step in your config (`values.yaml`) file:
  * cluster ID
  * node pool ID
  * and your custom setup

* Install the app using your values.

### Testing your deployment

// TODO document this

#### Ingesting data with ...

// TODO document this

## Limitations

The application and its default values have been tailored to work inside Giant Swarm clusters.
If you want to use it for any other scenario, know that you might need to adjust some values.

## Links

TODO: Add useful links

## Credit

This application is installing the upstream chart below with defaults to ensure it runs smoothly in Giant Swarm clusters.

* <https://github.com/grafana/helm-charts/blob/main/charts/tempo-distributed>
