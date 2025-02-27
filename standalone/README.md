# Standalone Tool Documentation

## Overview

The `standalone.py` script simulates the [InstructLab](https://instructlab.ai/) workflow within a [Kubernetes](https://kubernetes.io/) environment,
replicating the functionality of a [KubeFlow Pipeline](https://github.com/kubeflow/pipelines). This allows for distributed training and evaluation of models
without relying on centralized orchestration tools like KubeFlow.

The `standalone.py` tool provides support for fetching generated SDG (Synthetic Data Generation) data from an object store that complies with the AWS S3 specification. While AWS S3 is supported, alternative object storage solutions such as Ceph, Nooba, and MinIO are also compatible.

## Requirements

Since the `standalone.py` script is designed to run within a Kubernetes environment, the following
requirements must be met:
* A Kubernetes cluster with the necessary resources to run the InstructLab workflow.
  * Nodes with GPUs available for training.
  * [KubeFlow training operator](https://github.com/kubeflow/training-operator) must be running and `PyTorchJob` CRD must be installed.
  * A [StorageClass](https://kubernetes.io/docs/concepts/storage/storage-classes/) that supports dynamic provisioning with [ReadWriteMany](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes) access mode.
* A Kubernetes configuration that allows access to the Kubernetes cluster.
  * Both cluster and in-cluster configurations are supported.
* SDG data generated and uploaded to an object store.

> [!NOTE]
> The script can be run outside of a Kubernetes environment or within a Kubernetes Job, but it requires a Kubernetes configuration file to access the cluster.

> [!TIP]
> Check the `show` command to display an example of a Kubernetes Job that runs the script. Use `./standalone.py show` to see the example.

## Features

* Run any part of the InstructLab workflow in a standalone environment independently or a full end-to-end workflow:
  * Fetch SDG data from an object store.
  * Train model.
  * Evaluate model.
  * Final model evaluation. (Not implemented yet)
  * Push the final model back to the object store. (Not implemented yet) -  same location as the SDG data.

The `standalone.py` script includes a main command to execute the full workflow, along with
subcommands to run individual parts of the workflow separately. To view all available commands, use
`./standalone.py --help`. Essentially, this is how the script operates to execute the end-to-end
workflow. The command will retrieve the SDG data from the object store and set up the necessary
resources for running the training and evaluation steps:

```bash
./standalone.py run --namespace my-namespace --sdg-object-store-secret sdg-data
```

Now let's say you only want to fetch the SDG data, you can use the `sdg-data-fetch` subcommand:

```bash
./standalone.py run --namespace my-namespace --sdg-object-store-secret sdg-data sdg-data-fetch
```

Other subcommands are available to run the training and evaluation steps:

```bash
./standalone.py run --namespace my-namespace train
./standalone.py run --namespace my-namespace evaluation
```

* Fetch SDG data from an object store.
  * Support for AWS S3 and compatible object storage solutions.
  * Configuration via CLI options, environment variables, or Kubernetes secrets.

> [!CAUTION]
> All the CLI options MUST be positioned after the `run` command and BEFORE any subcommands.

## Usage

The script requires information regarding the location and method for accessing the SDG data. This information can be provided in two main ways:

1. CLI Options or/and Environment Variables: Supply all necessary information via CLI options or environment variables.
2. Kubernetes Secret: Provide the name of a Kubernetes secret that contains all relevant details using the `--sdg-object-store-secret` option.

### CLI Options

* `--namespace`: The namespace in which the Kubernetes resources are located - **Required**
* `--storage-class`: The storage class to use for the PVCs - **Optional** - Default: cluster default storage class.
* `--nproc-per-node`: The number of processes to run per node - **Optional** - Default: 1.
* `--sdg-object-store-secret`: The name of the Kubernetes secret containing the SDG object store credentials.
* `--sdg-object-store-endpoint`: The endpoint of the object store. `SDG_OBJECT_STORE_ENDPOINT` environment variable can be used as well.
* `--sdg-object-store-bucket`: The bucket name in the object store. `SDG_OBJECT_STORE_BUCKET` environment variable can be used as well.
* `--sdg-object-store-access-key`: The access key for the object store. `SDG_OBJECT_STORE_ACCESS_KEY` environment variable can be used as well.
* `--sdg-object-store-secret-key`: The secret key for the object store. `SDG_OBJECT_STORE_SECRET_KEY` environment variable can be used as well.
* `--sdg-object-store-data-key`: The key for the SDG data in the object store. e.g., `sdg.tar.gz`. `SDG_OBJECT_STORE_DATA_KEY` environment variable can be used as well.
* `--sdg-object-store-verify-tls`: Whether to verify TLS for the object store endpoint (default:
  true). `SDG_OBJECT_STORE_VERIFY_TLS` environment variable can be used as well.
* `--sdg-object-store-region`: The region of the object store. `SDG_OBJECT_STORE_REGION` environment variable can be used as well.

## Example End-To-End Workflow

### Generating and Uploading SDG Data

The following example demonstrates how to generate SDG data, package it as a tarball, and upload it
to an object store. This assumes that AWS CLI is installed and configured with the necessary
credentials.
In this scenario the name of the bucket is `sdg-data` and the tarball file is `sdg.tar.gz`.

```bash
ilab data generate
cd generated
tar -czvf sdg.tar.gz *
aws cp sdg.tar.gz s3://sdg-data/sdg.tar.gz
```

> [!CAUTION]
> Ensures SDG data is packaged as a tarball **without** top-level directories. So you must run `tar` inside the directory containing the SDG data.

### Creating the Kubernetes Secret

The simplest method to supply the script with the required information for retrieving SDG data is by
creating a Kubernetes secret. In the example below, we create a secret called `sdg-data` within the
`my-namespace` namespace, containing the necessary credentials. Ensure that you update the access
key and secret key as needed. The `data_key` field refers to the name of the tarball file in the
object store that holds the SDG data. In this case, it's named `sdg.tar.gz`, as we previously
uploaded the tarball to the object store using this name.

```bash
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Secret
metadata:
  name: sdg-data
  namespace: my-namespace
type: Opaque
stringData:
  bucket: sdg-data
  access_key: *****
  secret_key: *****
  data_key: sdg.tar.gz
EOF

./standalone run --namespace my-namespace --sdg-object-store-secret sdg-data
```

> [!WARNING]
> The secret must be part of the same namespace as the resources that the script interacts with.
> It's inherented from the `--namespace` option.

The list of all supported keys:

* `bucket`: The bucket name in the object store - **Required**
* `access_key`: The access key for the object store - **Required**
* `secret_key`: The secret key for the object store - **Required**
* `data_key`: The key for the SDG data in the object store - **Required**
* `verify_tls`: Whether to verify TLS for the object store endpoint (default: true) - **Optional**
* `endpoint`: The endpoint of the object store, e.g: https://s3.openshift-storage.svc:443 - **Optional**
* `region`: The region of the object store - **Optional**

#### Running the Script Without Kubernetes Secret

Alternatively, you can provide the necessary information directly via CLI options or environment,
the script will use the provided information to fetch the SDG data and create its own Kubernetes
Secret named `sdg-object-store-credentials` in the same namespace as the resources it interacts with (in this case, `my-namespace`).


```bash
./standalone run \
  --namespace my-namespace \
  --sdg-object-store-access-key key \
  --sdg-object-store-secret-key key \
  --sdg-object-store-bucket sdg-data \
  --sdg-object-store-data-key sdg.tar.gz
```

#### Advanced Configuration Using an S3-Compatible Object Store

If you don't use the official AWS S3 endpoint, you can provide additional information about the object store:

```bash
./standalone run \
  --namespace foo \
  --sdg-object-store-access-key key \
  --sdg-object-store-secret-key key \
  --sdg-object-store-bucket sdg-data \
  --sdg-object-store-data-key sdg.tar.gz \
  --sdg-object-store-verify-tls false \
  --sdg-object-store-endpoint https://s3.openshift-storage.svc:443
```

> [!IMPORTANT]
> The `--sdg-object-store-endpoint` option must be provided in the format
> `scheme://host:<port>`, the port can be omitted if it's the default port.

> [!TIP]
> If you don't want to run the entire workflow and only want to fetch the SDG data, you can use the `run sdg-data-fetch ` command
