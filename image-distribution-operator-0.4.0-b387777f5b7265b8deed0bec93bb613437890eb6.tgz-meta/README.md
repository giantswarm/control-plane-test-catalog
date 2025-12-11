# image-distribution-operator

This operator runs on a kubernetes management cluster.
Its purpose is to propagate node os images that are needed to create workload clusters to
the workload clusters respective image catalog.

## Description

The image-distribution-operator is made up of two controllers:

### `release-controller`
The `release-controller` watches release custom resources on the management cluster.
It will generate the os image name from each release and keep track of the images that are needed to create workload clusters.
For each image that is needed, it will create a `NodeImage` custom resource.
The list of releases using the image is stored in the `NodeImage` Status.
If a release is deleted, the `NodeImage` Status is updated to remove the release from the list.
If a `NodeImage` is no longer needed, it is deleted.

### `image-controller`
The `image-controller` watches `NodeImage` custom resources on the workload clusters.
For each `NodeImage` that is created, it will ensure that the image is available inside the provider image catalog.
The current state of the image is stored in the `NodeImage` Status. (e.g. `Available`, `Uploading`)
If the image is not available, the controller will attempt to upload the image to the provider image catalogs.
If the image is available, the controller will update the `NodeImage` Status to `Available`.
If the `NodeImage` is deleted, the controller will remove the image from the provider image catalogs.

### AWS S3 Client
The `image-controller` imports images from a public S3 bucket.
The bucket is specified inside the `values.yaml` file.

```yaml
s3:
  bucket: "my-bucket"
  region: "us-west-2"
```

### Vsphere Client
The `image-controller` can upload images to one or more locations inside a VCenter.
The VCenter credentials and locations are specified inside the `values.yaml` file.

```yaml
vsphere:
  credentials:
    username: "my-username"
    password: "my-password"
    vcenter: "my-vcenter"
  locations:
    location1:
      datacenter: "my-datacenter"
      datastore: "my-datastore"
      cluster: "my-cluster"
      folder: "my-folder"
    location2:
      datacenter: "another-datacenter"
      datastore: "another-datastore"
      cluster: "another-cluster"
      folder: "another-folder"
      resourcepool: "my-resourcepool" # Optional
      host: "my-host" # Optional - First host in the cluster by default
      network: "my-network" # Optional
      imagesuffix: "my-suffix" # Optional
```

### VMware Cloud Director Client
The `image-controller` can upload images to VMware Cloud Director (VCD) catalogs.
The VCD credentials and locations are specified inside the `values.yaml` file.

```yaml
vcd:
  downloadDir: "/tmp/images"
  credentials:
    url: "https://my-vcd-instance.example.com"
    username: "my-username"
    password: "my-password"
    org: "my-org"
    insecure: false
  locations:
    name: "my-location"
    org: "my-org"
    vdc: "my-vdc"
    catalog: "my-catalog"
```

## Getting Started

### Network requirements

#### Pull mode

The vSphere ESXi hosts must have access to the internet on port 443 (or 80, not preferred), this includes the vSphere firewall outgoing rules under `Hostname > Configure > System > Firewall > Outgoing > Edit > httpClient`.

#### Push mode

The IDO pod must have access to the IP of the vSphere ESXi hosts on port 443.

### Prerequisites
- go version v1.23.0+
- docker version 17.03+.
- kubectl version v1.11.3+.
- Access to a Kubernetes v1.11.3+ cluster.

### Keeping the helm chart up to date
The helm chart is generated using the kubebuilder plugin from the config directory.
After making changes to `config`, run `sync/sync.sh` to update the helm chart.
More information can be found in [this document](sync/README.md).

### To Deploy on the cluster
**Build and push your image to the location specified by `IMG`:**

```sh
make docker-build docker-push IMG=<some-registry>/image-distribution-operator:tag
```

**NOTE:** This image ought to be published in the personal registry you specified.
And it is required to have access to pull the image from the working environment.
Make sure you have the proper permission to the registry if the above commands don’t work.

**Install the CRDs into the cluster:**

```sh
make install
```

**Deploy the Manager to the cluster with the image specified by `IMG`:**

```sh
make deploy IMG=<some-registry>/image-distribution-operator:tag
```

> **NOTE**: If you encounter RBAC errors, you may need to grant yourself cluster-admin
privileges or be logged in as admin.

**Create instances of your solution**
You can apply the samples (examples) from the config/sample:

```sh
kubectl apply -k config/samples/
```

>**NOTE**: Ensure that the samples has default values to test it out.

### To Uninstall
**Delete the instances (CRs) from the cluster:**

```sh
kubectl delete -k config/samples/
```

**Delete the APIs(CRDs) from the cluster:**

```sh
make uninstall
```

**UnDeploy the controller from the cluster:**

```sh
make undeploy
```

## Project Distribution

Following the options to release and provide this solution to the users.

### By providing a bundle with all YAML files

1. Build the installer for the image built and published in the registry:

```sh
make build-installer IMG=<some-registry>/image-distribution-operator:tag
```

**NOTE:** The makefile target mentioned above generates an 'install.yaml'
file in the dist directory. This file contains all the resources built
with Kustomize, which are necessary to install this project without its
dependencies.

2. Using the installer

Users can just run 'kubectl apply -f <URL for YAML BUNDLE>' to install
the project, i.e.:

```sh
kubectl apply -f https://raw.githubusercontent.com/<org>/image-distribution-operator/<tag or branch>/dist/install.yaml
```

### By providing a Helm Chart

1. Build the chart using the optional helm plugin

```sh
kubebuilder edit --plugins=helm/v1-alpha
```

2. See that a chart was generated under 'dist/chart', and users
can obtain this solution from there.

**NOTE:** If you change the project, you need to update the Helm Chart
using the same command above to sync the latest changes. Furthermore,
if you create webhooks, you need to use the above command with
the '--force' flag and manually ensure that any custom configuration
previously added to 'dist/chart/values.yaml' or 'dist/chart/manager/manager.yaml'
is manually re-applied afterwards.

## Contributing
// TODO(user): Add detailed information on how you would like others to contribute to this project

**NOTE:** Run `make help` for more information on all potential `make` targets

More information can be found via the [Kubebuilder Documentation](https://book.kubebuilder.io/introduction.html)

## License

Copyright 2025.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

