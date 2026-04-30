---
title: Anyscale on Azure architecture overview
description: Learn how Anyscale on Azure is structured, including the control plane, data plane, and Kubernetes operator model, and how these components interact within your Azure tenant.
author: kaysieyu
ms.author: kaysieyu
ms.date: 04/29/2026
ms.service: azure-kubernetes-service
ms.topic: concept-article
---

# Architecture overview

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
1. Manages the ingress or gateway controller for head node access.

This polling model means all network connections originate from your cluster outbound to the Anyscale control plane. You don't need inbound firewall rules. For details on required egress domains and ports, see [Networking](networking.md).

## Managed identities and permissions

The operator authenticates to Azure services using a managed identity. The portal provisions this identity during cloud creation and scopes it to the resources in your resource group.

You can configure permissions at different levels of granularity:

- **Shared identity**: One managed identity covers all workloads in the cloud.
- **Granular identity mapping**: Map separate identities to individual users, projects, or workload types through cloud IAM configuration.

For information on Microsoft Entra ID integration and Azure role assignments, see [Identity and access](identity-access.md).

## Architecture diagram overview

:::image type="complex" source="media/architecture/anyscale-on-azure-architecture.png" alt-text="Anyscale on Azure architecture with two planes: Anyscale Control Plane on the left and Customer Data Plane on the right, connected by arrows.":::
   The diagram shows two bordered boxes side by side. The left box (Anyscale Control Plane, in the Anyscale Azure tenant) contains three components stacked vertically: Scheduling and Job Management, Anyscale Console, and REST API / SDK. The right box (Customer Data Plane, in your Azure subscription and AKS cluster) contains the Anyscale Kubernetes Operator in the center, with two Ray Cluster boxes to its right. The top Ray Cluster shows a head node and N worker nodes. The bottom Ray Cluster shows a head node and N replicas. Two arrows run between the planes: one from the control plane to the operator labeled "deploys clusters, runs jobs and services", and one back labeled "logs, metrics". From the operator, two arrows point right labeled "deploys and manages", one to each Ray Cluster. Below both boxes, a Developer, ML Engineer, or Data Scientist figure connects to the control plane with an arrow labeled "deploy, configure, monitor" and to the Customer Data Plane with an arrow labeled "interact with Ray clusters".
:::image-end:::

## Component summary table

| Component | Location | Owner |
|-----------|----------|-------|
| Anyscale console | Anyscale-hosted Azure tenant | Anyscale |
| Scheduling and management APIs | Anyscale-hosted Azure tenant | Anyscale |
| AKS cluster | Your Azure subscription | You |
| Anyscale Kubernetes operator | Your AKS cluster | Anyscale, installed by the Azure portal |
| Ray clusters | Your AKS cluster | You |
| Azure Blob Storage and Azure Data Lake Storage (ADLS) | Your Azure subscription | You |
| Azure Load Balancer | Your Azure subscription | You |

## Next steps

- [Networking](networking.md) for required egress domains and traffic flow details.
- [Identity and access](identity-access.md) for Microsoft Entra ID SSO and Azure role assignments.
- [Quickstart](quickstart-azure-cli-gateway-envoy.md) to deploy your first Anyscale cloud on Azure.
