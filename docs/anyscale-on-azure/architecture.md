---
title: Anyscale on Azure architecture overview
description: Learn how Anyscale on Azure is structured, including the control plane, data plane, and Kubernetes operator model, and how these components interact within your Azure tenant.
author: kaysieyu
ms.author: kaysieyu
ms.reviewer: mbender
ms.date: 09/23/2026
ms.service: azure-kubernetes-service
ms.topic: concept-article
ms.custom: references_regions
---

# Anyscale on Azure architecture overview

[!INCLUDE [anyscale-public-preview](../../Includes/anyscale-public-preview.md)]

Anyscale on Azure separates the platform into two distinct planes. The **control plane** is managed by Anyscale and hosted in Azure. The **data plane** runs entirely within your Azure subscription. This separation keeps your workloads, data, and container images inside your tenant while Anyscale handles orchestration and management.

## Control plane components

Anyscale hosts and operates the control plane. It provides:

- The Anyscale console at [console.azure.anyscale.com](https://console.azure.anyscale.com).
- Scheduling and job management APIs.
- Monitoring, logging aggregation, and the metrics dashboard.
- Cloud and cluster lifecycle management.

You interact with the control plane through the Azure portal, which handles cloud creation and deletion, and through the Anyscale console, the Anyscale CLI, or the Anyscale SDK.

The control plane is isolated from your data plane and never directly accesses your AKS cluster. Instead, a Kubernetes operator running in your cluster polls the control plane for instructions and acts on them locally.

## Data plane components

The data plane runs inside your Azure subscription and consists of:

- Your **AKS cluster**, which hosts all Ray workloads.
- The **Anyscale Kubernetes operator**, which manages Ray cluster lifecycles.
- **Azure storage**, in either Azure Blob Storage or Azure Data Lake Storage (ADLS), for artifacts and datasets.
- **Azure Load Balancer** for client access to Ray clusters.

Your subscription owns all compute, data, and networking resources in the data plane. Anyscale has no direct access to your cluster nodes or your data.

## Kubernetes operator model

The Anyscale operator is a Kubernetes controller. The Azure portal installs it into your AKS cluster automatically during cloud creation. The operator:

1. Polls the control plane endpoint (`<cloud-id>.anyscale-cloud.dev`) for pending operations.
1. Creates and manages Kubernetes resources, such as pods, services, and ingress rules, for Ray clusters.
1. Reports cluster health and telemetry to the control plane.
1. Creates the ingress or gateway routing resources that connect clients to the Ray head node.

This polling model means all network connections originate from your cluster outbound to the Anyscale control plane. You don't need inbound firewall rules. For details on required egress domains and ports, see [Networking](networking.md).

The operator creates the ingress or gateway routing resources, but you provide the ingress or gateway controller that serves them. Without a controller, client traffic can't reach the head node, and workspace creation fails. For the controller requirement, see [Ingress or gateway controller requirement](networking.md#ingress-or-gateway-controller-requirement).

## Managed identities and permissions

The operator authenticates to Azure services using a managed identity. The portal deploys this identity during cloud creation and scopes it to the resources in your resource group.

You can configure permissions at different levels of granularity:

- **Shared identity**: One managed identity covers all workloads in the cloud.
- **Granular identity mapping**: Map separate identities to individual users, projects, or workload types through cloud IAM configuration.

For information on Microsoft Entra ID integration and Azure role assignments, see [Identity and access](identity-access.md).

## Architecture diagram overview

:::image type="complex" source="media/architecture/anyscale-on-azure-architecture.png" alt-text="Anyscale on Azure architecture with two planes: Anyscale Control Plane on the left and Customer Data Plane on the right, connected by arrows.":::
   The diagram shows two bordered boxes side by side. The left box is the Anyscale Control Plane in the Anyscale Azure tenant. It contains three stacked components: Scheduling and Job Management, Anyscale Console, and REST API / SDK. The right box is the Customer Data Plane in your Azure subscription and AKS cluster. It contains the Anyscale Kubernetes Operator in the center and two Ray Clusters to its right. Both Ray Clusters show a head and N workers. An arrow from the control plane to the operator is labeled deploys clusters, runs jobs and services. An arrow back is labeled logs, metrics. Two arrows from the operator to the Ray Clusters are labeled deploys and manages. A user figure connects to the control plane with deploy, configure, monitor and to the Customer Data Plane with interact with Ray clusters.
:::image-end:::

## Responsibility matrix

Owning a component is distinct from operating it and from supporting it. Anyscale on Azure uses a shared-responsibility model similar to the model Azure Kubernetes Service (AKS) defines. For each component, the following matrix separates three responsibilities:

- **Owns**: the party whose Azure subscription or tenant holds the resource.
- **Operates and maintains**: the party that runs, patches, and upgrades the component.
- **Support responsibility**: the team that owns resolving issues with the component. You start every support request through the standard Azure support process. Microsoft triages the request and routes it to Anyscale when the issue needs Anyscale product expertise. For the full flow, see [Support model](support-model.md).

Several components are shared. For a shared component, Anyscale provides and manages the software while you provision, configure, or run it inside your subscription. For the AKS shared-responsibility model that this matrix builds on, see [AKS support policies](/azure/aks/support-policies).

| Component | Location | Owns | Operates and maintains | Support responsibility |
|-----------|----------|------|------------------------|------------------------|
| Anyscale console | Anyscale-hosted Azure tenant | Anyscale | Anyscale | Anyscale |
| Scheduling and management APIs | Anyscale-hosted Azure tenant | Anyscale | Anyscale | Anyscale |
| AKS cluster | Your Azure subscription | You | You and Microsoft | Microsoft |
| Anyscale Kubernetes operator | Your AKS cluster | Anyscale | Anyscale | Anyscale |
| Ray clusters | Your AKS cluster | You | You and Anyscale | Anyscale |
| Azure Blob Storage and Azure Data Lake Storage (ADLS) | Your Azure subscription | You | You | Microsoft |
| Azure Load Balancer | Your Azure subscription | You | You and Anyscale | Microsoft |
| Managed identity and role assignments | Your Azure subscription | You | You and Microsoft | Microsoft |

The Anyscale Kubernetes operator shows the ownership distinction most clearly. It runs inside your AKS cluster, but Anyscale develops the software, and the Azure portal installs it during cloud creation. You own the cluster it runs in, and Anyscale owns and maintains the operator itself.

## Next steps

- [Networking](networking.md) for required egress domains and traffic flow details.
- [Identity and access](identity-access.md) for Microsoft Entra ID SSO and Azure role assignments.
- [Quickstart](quickstart-azure-cli.md) to deploy your first Anyscale cloud on Azure.
