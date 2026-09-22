---
title: What is Anyscale on Azure?
description: Anyscale on Azure is a managed Ray platform that runs on your Azure Kubernetes Service cluster, with Microsoft Entra ID SSO and built-in Azure service integrations.
author: kaysieyu
ms.author: kaysieyu
ms.reviewer: mbender
reviewer: mbender-ms
ms.date: 09/21/2026
ms.service: azure-kubernetes-service
ms.topic: overview
ms.custom: references_regions
---

# What is Anyscale on Azure?

[!INCLUDE [anyscale-public-preview](../../Includes/anyscale-public-preview.md)]

Anyscale on Azure is a managed platform for running distributed Python workloads on [Ray](https://docs.ray.io). It deploys directly onto your [Azure Kubernetes Service (AKS)](/azure/aks/) cluster and integrates with the Azure services your team already uses.

Anyscale on Azure is an Azure Native Integration. You access it through the Azure portal and the Anyscale console at [console.azure.anyscale.com](https://console.azure.anyscale.com). To sign in to the Anyscale console, your Azure tenant and your user account need the correct permissions. The [deployment quickstart](quickstart-azure-cli.md) walks you through the required configuration.

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
- The Anyscale scheduler applies workload priority to jobs and workspaces, not to services.

Anyscale on Azure doesn't support the following features documented in the [Anyscale documentation](https://docs.anyscale.com):

- Machine pools and the Global Resource Scheduler (GRS)
- Lineage tracking
- Job queues
- The following Anyscale console organization settings:
   - Billing
   - Budgets
   - Resource notifications
   - Cost analysis

### Multi-resource cloud support

An Anyscale cloud on Azure can hold more than one *cloud resource*, where each cloud resource is a Kubernetes cluster attached to the cloud. Use additional cloud resources to isolate environments, add capacity in another region, or attach clusters from other Kubernetes offerings. Add them in the Azure portal or by using an Azure Resource Manager (ARM) template.

Support for cloud resources other than the primary cloud resource depends on the workload type:

- Jobs run on any cloud resource and can fall back across several when one can't start a cluster in time.
- Workspaces run on any single cloud resource. Anyscale doesn't support fallback across resources.
- Services run on the primary cloud resource only.

The following constraints apply:

- Anyscale creates each cloud with one primary cloud resource named `default`. It's AKS-backed and must use the `Azure` provider.
- Each cluster runs entirely within one cloud resource. Anyscale doesn't autoscale or schedule a single workload across cloud resources.
- The Anyscale CLI and console are read-only for cloud resources. Create and delete cloud resources through Azure.

To learn how cloud resources work on Azure and how to add one, see [What is a cloud resource on Azure?](cloud-resources-overview.md) and [Add a cloud resource](add-cloud-resource.md).

## Get started with Anyscale on Azure

To deploy your first Anyscale cloud on Azure, see the [Quickstart](quickstart-azure-cli.md).

## Learn more about Anyscale on Azure

- [Architecture overview](architecture.md)
- [Networking](networking.md)
- [Identity and access](identity-access.md)
- [Support model](support-model.md)
- [Supported regions](supported-regions.md)
- [Configure head node fault tolerance](https://docs.anyscale.com/administration/resource-management/head-node-fault-tolerance) for production Anyscale Services
- [Anyscale documentation](https://docs.anyscale.com) for full platform reference

## Resources and policies

- [Anyscale on Azure terms and conditions](https://www.anyscale.com/anyscale-on-azure-terms): review before deploying production workloads.
- [Anyscale on Azure pricing](https://azure.microsoft.com/pricing/details/anyscale-on-azure/): review before planning a deployment.
- [Anyscale Privacy Policy](https://www.anyscale.com/privacy-policy): how Anyscale handles your data.
- [Anyscale on Azure knowledge base](https://docs.anyscale.com/kb/azure): troubleshooting and operational guidance beyond the MS Learn content.
