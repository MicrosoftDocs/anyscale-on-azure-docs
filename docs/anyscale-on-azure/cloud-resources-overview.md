---
title: What is a cloud resource on Azure?
description: Learn how an Anyscale cloud on Azure can hold multiple cloud resources, each a Kubernetes cluster that acts as an independent deployment target.
author: kaysieyu
ms.author: kaysieyu
ms.reviewer: mbender
ms.date: 09/18/2026
ms.service: azure-kubernetes-service
ms.topic: concept-article
ms.custom: references_regions
---

# What is a cloud resource on Anyscale on Azure?

[!INCLUDE [anyscale-public-preview](../../Includes/anyscale-public-preview.md)]

An Anyscale cloud on Azure can contain more than one *cloud resource*. A cloud resource is one Kubernetes cluster attached to the cloud. Each one is an independent deployment target with its own:

- Kubernetes cluster.
- Anyscale operator installation.
- Object storage bucket.
- Operator identity.
- Gateway or ingress controller.

On Azure, a cloud resource is an Azure resource. It's a child resource of the Anyscale cloud, at `Anyscale.Platform/clouds/{cloudName}/cloudResources/{cloudResourceName}`. Azure creates it in your resource group, and Azure Resource Manager (ARM) is the source of truth for it.

Anyscale clouds are created with a single cloud resource named `default`, which is the cloud's primary cloud resource. Add more cloud resources to:

- Isolate environments from each other. Each cloud resource can be in a different virtual network.
- Add capacity, or capacity in another region, without creating a second Anyscale cloud.
- Attach clusters from other Kubernetes offerings to the same Anyscale cloud.

To add a cloud resource, see [Add a cloud resource with an ARM template](add-cloud-resource.md).

## Azure-specific behavior and limitations

Anyscale on Azure has the following behavioral differences and limitations not documented in the [Anyscale documentation for cloud resources](https://docs.anyscale.com/clouds/multi-cloud).

### Create and manage cloud resources

- Create cloud resources in the Azure portal or with an ARM template. The Azure portal supports AKS-backed cloud resources. ARM supports all providers.
- The following Anyscale CLI commands aren't supported on Azure: `anyscale cloud setup`, `anyscale cloud register`, `anyscale cloud delete`, `anyscale cloud resource create`, `anyscale cloud resource setup`, and `anyscale cloud resource delete`.
- The Anyscale CLI and console are read-only for cloud resources on Azure. Use them to list resources and verify health.

### Providers and compute stack

- A cloud resource can use any of the following providers: `Azure`, `AWS`, `GCP`, or `Generic`. Use `Generic` to configure Kubernetes clusters from neoclouds and other providers.
- All cloud resources require Kubernetes as the compute substrate. Virtual machine compute stacks aren't supported.
- The first cloud resource on a cloud must use provider `Azure`. Resources you add afterward can use any provider. Creating a cloud whose first resource is non-Azure fails with `The first cloud resource on cloud <arm-id> must have provider AZURE`.
- Managed container image builds run on the cloud's primary cloud resource, using the Azure Container Registry configured for that cloud.

### Workload support

Support for cloud resources other than the primary cloud resource depends on the workload type.

- Jobs run on any cloud resource, and can fall back across several. See [Add a cloud resource with an ARM template](add-cloud-resource.md).
- Workspaces run on any single cloud resource. Fallback across resources isn't supported.
- Services run on the primary cloud resource only.

On Azure, the primary cloud resource is named `default` and must use provider `Azure`, so it's always AKS-backed. Services run on AKS only, whatever the providers of the other cloud resources on the cloud. Jobs and workspaces can run on any cloud resource, including non-AKS resources.

Anyscale stores a workspace snapshot in the object storage of the cloud resource where the workspace runs. Moving an existing workspace to a different cloud resource doesn't stop it from starting, but Anyscale doesn't restore it from the snapshot, and it loses the data stored in the object storage of the previous cloud resource.

### Naming

- Cloud resource names must match `^[a-zA-Z0-9_-]+$`, which allows letters, digits, hyphens, and underscores, and must be 256 characters or fewer.
- Names must be unique within a cloud. A duplicate name returns HTTP 409.
- There's no limit on the number of cloud resources per cloud.

### Regions

- `location` and `region` are different values:
  - `location` is the Azure region that Azure Resource Manager tracks the resource record in.
  - `region` is the cloud provider region where compute runs, such as `us-west-2`.
- Every provider requires a `region`.
- `region` must match `^[a-z0-9]+(-[a-z0-9]+)*$` and be at most 35 characters.
- For Azure, AWS, and Google Cloud, Anyscale validates `region` against that provider's known regions. Anyscale doesn't convert the casing you provide. For Generic, `region` is a free-form routing label and isn't checked against a region list.

### Object storage

- Each cloud resource has its own bucket. The URI scheme must match the provider:
  - Azure: `abfss://<container>@<account>.dfs.core.windows.net` or `azure://<container>`.
  - AWS: `s3://<bucket>`.
  - Google Cloud: `gs://<bucket>`.
  - Generic: any fully qualified `<scheme>://<bucket>`.
- Bucket URIs can't contain a sub-path. Anyscale rejects `s3://my-bucket/prefix`.
- You must provide `cloudStorageBucketEndpoint` for Azure. It must be a bare `https://<host>` origin with no path. For other providers, leave it empty unless you're pointing at S3-compatible storage at a custom endpoint.
- `cloudStorageBucketRegion` defaults to `region`. Set it explicitly if the bucket is in a different region from the resource's compute.
- The Anyscale console fetches log contents and files in your browser directly from the bucket, so each bucket needs a cross-origin resource sharing (CORS) rule that allows `https://console.azure.anyscale.com`. Anyscale configures this rule for you when you add an AKS-backed resource. For AWS, Google Cloud, and Generic resources, you configure it yourself.

### Identity

- The identity the Anyscale operator presents is recorded when you create the cloud resource, and you can't change it afterward. The value you configure on the operator must match it.
- For Azure, the cloud resource records the managed identity's **principal ID**, while the operator is configured with the same identity's **client ID**. These are different GUIDs.
- For AWS it's the IAM role ARN, and for Google Cloud the service account email. This value is the same in both places.
- Generic clusters have no cloud provider identity, and the operator authenticates with an Anyscale CLI token instead.

### Networking

- Each cloud resource needs its own gateway or ingress controller before you can run workloads on it. Anyscale recommends Envoy Gateway.
- On Azure, the gateway's HTTPS listeners must use `*.i.azure.anyscaleuserdata.com` and `*.s.azure.anyscaleuserdata.com`. These hostnames follow the control plane, not the cluster, so they're the same for every cloud resource on an Azure cloud regardless of provider.
- The operator creates the TLS certificate secrets, named after the cloud resource ID with underscores replaced by hyphens: `anyscale-<cloud-resource-id>-certificate` and `anyscale-svc-<cloud-resource-id>-certificate`.

### Change and remove a cloud resource

- You can't change cloud resource properties after creation. Redeploying with a modified property fails with `Cannot update fields in the existing cloud resource <name>`. Redeploying with unchanged values succeeds and makes no change. To correct a value, delete the resource and create it again.
- Deletion is per resource and by name. It doesn't affect other cloud resources on the same cloud.
- Deleting a cloud resource is blocked while that resource has running clusters, and returns HTTP 409. Terminate its workloads first.

### Other limitations

- Create cloud resources one at a time. If a single ARM template defines more than one, chain them with `dependsOn` so you create them in sequence.
