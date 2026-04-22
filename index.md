---
title: What is Anyscale on Azure? | Microsoft Learn
description: Anyscale on Azure is a managed Ray platform that runs on your Azure Kubernetes Service cluster, with Entra ID SSO and native Azure service integrations.
author: kaysieyu
ms.author: kaysieyu
ms.date: 04/03/2026
ms.topic: overview
---

# What is Anyscale on Azure?

Anyscale on Azure is a managed platform for running distributed Python workloads on [Ray](https://docs.ray.io). It deploys directly onto your Azure Kubernetes Service (AKS) cluster and integrates with the Azure services your team already uses.

The platform is developed jointly by Anyscale and Microsoft as an Azure Native Integration. You access it through the Azure portal and the Anyscale console at [console.azure.anyscale.com](https://console.azure.anyscale.com).

## How it works

Anyscale on Azure separates responsibilities into two planes:

- **Control plane**: Hosted by Anyscale in Azure. Handles scheduling, monitoring, job management, and the Anyscale console. You interact with it through the Azure portal, the Anyscale CLI, or the Anyscale SDK.
- **Data plane**: Runs inside your Azure subscription, on your AKS cluster. Your Ray workloads, container images, and data stay within your own tenant.

For a detailed breakdown of these components, see [Architecture overview](architecture.md).

## Key capabilities

**Kubernetes-native deployment**
All Anyscale cloud resources on Azure use Kubernetes. Anyscale deploys an operator into your AKS cluster that manages Ray cluster lifecycles on your behalf.

**Entra ID single sign-on**
Your team signs in to Anyscale using their existing Azure credentials. No separate identity provider setup is required. For details, see [Identity and access](identity-access.md).

**Azure service integrations**
Anyscale on Azure works with the Azure services you use:

| Service | Use |
|---------|-----|
| Azure Kubernetes Service (AKS) | Compute platform for Ray workloads |
| Azure Blob Storage / ADLS | Artifact storage and dataset access |
| Azure Container Registry (ACR) | Custom container image distribution |
| Azure Load Balancer | Client access to Ray clusters and services |

**Managed permissions**
Azure managed identities govern access to cloud resources. You can use a single shared identity or map permissions granularly to users, projects, or workload types.

## Limitations

Anyscale on Azure is currently in Public Preview. The following limitations apply:

- Cloud creation and deletion require the Azure portal. The following CLI commands aren't supported: `anyscale cloud setup`, `anyscale cloud register`, `anyscale cloud delete`, `anyscale cloud resource create`, and `anyscale cloud resource delete`.
- Only AKS-based deployment is supported. VM stack features and Anyscale-hosted clouds aren't available.
- The following CLI commands aren't supported: `anyscale workspace_v2 ssh`, `anyscale workspace_v2 pull`, and all `anyscale image` commands.
- The Global Resource Scheduler (GRS) isn't supported.

## Get started

To deploy your first Anyscale cloud on Azure, see the [Quickstart](quickstart-azcli-gateway-envoy.md).

## Learn more

- [Architecture overview](architecture.md)
- [Networking](networking.md)
- [Identity and access](identity-access.md)
- [Support model](support-model.md)
- [Supported regions](supported-regions.md)
- [Anyscale documentation](https://docs.anyscale.com)—full platform reference
