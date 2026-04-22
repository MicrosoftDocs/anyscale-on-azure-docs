---
title: Anyscale on Azure architecture overview | Microsoft Learn
description: Learn how Anyscale on Azure is structured, including the control plane and data plane components, the Kubernetes operator model, and how they interact within your Azure tenant.
author: kaysieyu
ms.author: kaysieyu
ms.date: 04/03/2026
ms.topic: conceptual
---

# Architecture overview

Anyscale on Azure separates the platform into two distinct planes: a **control plane** managed by Anyscale and hosted in Azure, and a **data plane** that runs entirely within your Azure subscription. This separation ensures that your workloads, data, and container images never leave your tenant while Anyscale handles orchestration and management.

## Control plane

The control plane is hosted and operated by Anyscale. It provides:

- The Anyscale console at [console.azure.anyscale.com](https://console.azure.anyscale.com)
- Scheduling and job management APIs
- Monitoring, logging aggregation, and the metrics dashboard
- Cloud and cluster lifecycle management

You interact with the control plane through the Azure portal (for cloud creation and deletion), the Anyscale console, the Anyscale CLI, or the Anyscale SDK.

The control plane is isolated from your data plane. It never directly accesses your AKS cluster. Instead, a Kubernetes operator running in your cluster polls the control plane for instructions and acts on them locally.

## Data plane

The data plane runs inside your Azure subscription and consists of:

- Your **AKS cluster**, which hosts all Ray workloads
- The **Anyscale Kubernetes operator**, which manages Ray cluster lifecycles
- **Azure storage** (Blob Storage or ADLS) for artifacts and datasets
- **Azure Load Balancer** for client access to Ray clusters

All compute, data, and networking resources in the data plane are owned by your subscription. Anyscale has no direct access to your cluster nodes or your data.

## Kubernetes operator model

The Anyscale operator is a Kubernetes controller deployed into your AKS cluster via Helm. It:

1. Polls the control plane endpoint (`<cloud-id>.anyscale-cloud.dev`) for pending operations
2. Creates and manages Kubernetes resources (pods, services, ingress rules) for Ray clusters
3. Reports cluster health and telemetry back to the control plane
4. Manages the Nginx ingress controller for head node access

This polling model means all network connections originate from your cluster outbound to Anyscale's control plane. No inbound firewall rules are required. For details on required egress domains and ports, see [Networking](networking.md).

## Managed identities and permissions

The operator authenticates to Azure services using a managed identity provisioned by the portal during cloud creation. This identity is scoped to the resources in your resource group.

You can configure permissions at different levels of granularity:

- **Shared identity** — One managed identity covers all workloads in the cloud
- **Granular identity mapping** — Map separate identities to individual users, projects, or workload types via cloud IAM configuration

For information on Entra ID integration and Azure role assignments, see [Identity and access](identity-access.md).

## Architecture diagram

:::image type="complex" source="media/architecture/anyscale-on-azure-architecture.png" alt-text="Anyscale on Azure architecture with two planes: Anyscale Control Plane on the left and Customer Data Plane on the right, connected by arrows.":::
   The diagram shows two bordered boxes side by side. The left box (Anyscale Control Plane, in Anyscale's Azure tenant) contains three components stacked vertically: Scheduling and Job Management, Anyscale Console, and REST API / SDK. The right box (Customer Data Plane, in your Azure subscription / AKS cluster) contains the Anyscale Kubernetes Operator in the center, with two Ray Cluster boxes to its right. The top Ray Cluster shows a head node and N worker nodes; the bottom Ray Cluster shows a head node and N replicas. Two arrows run between the planes: one from the control plane to the operator labeled "deploys clusters, runs jobs / services", and one back labeled "logs, metrics". From the operator, two arrows point right labeled "deploys and manages", one to each Ray Cluster. Below both boxes, a Developer / ML Engineer / Data Scientist figure connects to the control plane with an arrow labeled "deploy, configure, monitor" and to the Customer Data Plane with an arrow labeled "interact with Ray clusters".
:::image-end:::

## Component summary

| Component | Location | Owner |
|-----------|----------|-------|
| Anyscale console | Anyscale-hosted Azure | Anyscale |
| Scheduling and management APIs | Anyscale-hosted Azure | Anyscale |
| AKS cluster | Your Azure subscription | You |
| Anyscale Kubernetes operator | Your AKS cluster | Anyscale (deployed via Helm) |
| Ray clusters | Your AKS cluster | You |
| Azure Blob Storage / ADLS | Your Azure subscription | You |
| Azure Load Balancer | Your Azure subscription | You |

## Next steps

- [Networking](networking.md) — required egress domains and traffic flow details
- [Identity and access](identity-access.md) — Entra ID SSO and Azure role assignments
- [Quickstart](quickstart-azcli-gateway-envoy.md) — deploy your first Anyscale cloud on Azure
