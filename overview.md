---
title: What is Anyscale on Azure?
description: Anyscale on Azure is a managed Ray platform that runs on your Azure Kubernetes Service cluster, with Microsoft Entra ID SSO and built-in Azure service integrations.
author: kaysieyu
ms.author: kaysieyu
ms.date: 05/26/2026
ms.service: azure-kubernetes-service
ms.topic: overview
---

# What is Anyscale on Azure?

[!INCLUDE [anyscale-public-preview](Includes/anyscale-public-preview.md)]

Anyscale on Azure is a managed platform for running distributed Python workloads on [Ray](https://docs.ray.io). It deploys directly onto your [Azure Kubernetes Service (AKS)](/azure/aks/) cluster and integrates with the Azure services your team already uses.

Anyscale and Microsoft jointly develop the platform as an Azure Native Integration. You access it through the Azure portal and the Anyscale console at [console.azure.anyscale.com](https://console.azure.anyscale.com). To sign in to the Anyscale console, your Azure tenant and your user account need the correct permissions. The [deployment quickstart](quickstart-azure-cli-gateway-envoy.md) walks you through the required configuration.

## How it works

Anyscale on Azure separates responsibilities into two planes:

- **Control plane**: Anyscale hosts this plane in Azure. It handles scheduling, monitoring, job management, and the Anyscale console. You interact with it through the Azure portal, the Anyscale CLI, or the Anyscale SDK.
- **Data plane**: Runs inside your Azure subscription, on your AKS cluster. Your Ray workloads, container images, and data stay within your own tenant.

For a detailed breakdown of these components, see [Architecture overview](architecture.md).

## Key platform capabilities

### Kubernetes-native deployment

All Anyscale cloud resources on Azure use Kubernetes. Anyscale deploys an operator into your AKS cluster that manages Ray cluster lifecycles on your behalf.

### Microsoft Entra ID single sign-on

Your team signs in to Anyscale using their existing Azure credentials. You don't need to set up a separate identity provider. For details, see [Identity and access](identity-access.md).

### Azure service integrations

Anyscale on Azure works with the Azure services you use:

| Service | Use |
|---------|-----|
| Azure Kubernetes Service (AKS) | Compute platform for Ray workloads |
| Azure Blob Storage and Azure Data Lake Storage (ADLS) | Artifact storage and dataset access |
| Azure Container Registry (ACR) | Custom container image distribution |
| Azure Load Balancer | Client access to Ray clusters and services |

### Managed permissions

Azure managed identities govern access to cloud resources. You can use a single shared identity or map permissions granularly to users, projects, or workload types.

## Public Preview limitations

Anyscale on Azure is in Public Preview. The following limitations apply:

- Anyscale on Azure supports only AKS-based deployment. VM stack features and Anyscale-hosted clouds aren't available.
- Cloud creation and deletion require the Azure portal. The following CLI commands aren't supported: `anyscale cloud setup`, `anyscale cloud register`, `anyscale cloud delete`, `anyscale cloud resource create`, and `anyscale cloud resource delete`.
- The following workload CLI commands aren't supported: `anyscale workspace_v2 ssh`, `anyscale workspace_v2 pull`, and `anyscale image archive`.
- Anyscale on Azure is available in a limited set of Azure regions. See [Supported regions](supported-regions.md).

Anyscale on Azure doesn't support the following features documented in the [Anyscale documentation](https://docs.anyscale.com):

- Machine pools and the Global Resource Scheduler (GRS)
- Services
- Lineage tracking
- Job queues
- The following Anyscale console organization settings:
   - Billing
   - Budgets
   - Resource notifications
   - Cost analysis

### Multi-resource cloud support for Anyscale on Azure

Because of limited support for multi-resource cloud features, Anyscale recommends only using the default cloud resource for each Anyscale cloud during Public Preview.

During Public Preview, Anyscale on Azure has the following support for multi-resource clouds:

- You can use the Azure portal to add new Anyscale cloud resources to an existing Anyscale cloud.
  - Anyscale clouds deploy with a single cloud resource by default.
  - Each cloud resource correspond to an Anyscale operator installed on an AKS cluster.
  - Cloud resources serve as an isolation boundary for a dedicated environment. Each cloud resource can be in a different VPC.
- Anyscale on Azure only supports cloud resources backed by AKS clusters. You can't add cloud resources from other Kubernetes offerings or backed by virtual machines.
- Anyscale on Azure only supports defining compute configs for a single cloud resource.
  - By default, compute configs always use the default cloud resource for an Anyscale cloud.
  - If you have multiple cloud resources defined for a cloud, you can use the `cloud_resource` field in your compute config to specify a cloud resource while configuring a job or workspace.
  - Anyscale on Azure doesn't support autoscaling or scheduling across multiple cloud resources.

## Get started with Anyscale on Azure

To deploy your first Anyscale cloud on Azure, see the [Quickstart](quickstart-azure-cli-gateway-envoy.md).

## Learn more about Anyscale on Azure

- [Architecture overview](architecture.md)
- [Networking](networking.md)
- [Identity and access](identity-access.md)
- [Support model](support-model.md)
- [Supported regions](supported-regions.md)
- [Configure head node fault tolerance](https://docs.anyscale.com/administration/resource-management/head-node-fault-tolerance) for production Anyscale Services
- [Anyscale documentation](https://docs.anyscale.com) for full platform reference

## Resources and policies

- [Anyscale on Azure terms and conditions](https://www.anyscale.com/anyscale-on-azure-terms)-review before deploying production workloads.
- [Anyscale Privacy Policy](https://www.anyscale.com/privacy-policy)-how Anyscale handles your data.
- [Anyscale on Azure knowledge base](https://docs.anyscale.com/kb/azure)-troubleshooting and operational guidance beyond the MS Learn content.
