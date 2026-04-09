---
title: Anyscale on Azure architecture overview | Microsoft Learn
description: Learn how Anyscale on Azure is structured, including the control plane and data plane components, the Kubernetes operator model, and how they interact within your Azure tenant.
author: tdwyer
ms.author: tdwyer
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
- **Azure Container Registry (ACR)** for custom container images
- **Azure Key Vault** for secrets

All compute, data, and networking resources in the data plane are owned by your subscription. Anyscale has no direct access to your cluster nodes or your data.

## Kubernetes operator model

The Anyscale operator is a Kubernetes controller deployed into your AKS cluster via Helm. It:

1. Polls the control plane endpoint (`<cloud-id>.anyscale-cloud.dev`) for pending operations
2. Creates and manages Kubernetes resources (pods, services, ingress rules) for Ray clusters
3. Reports cluster health and telemetry back to the control plane
4. Manages the Nginx ingress controller for head node access

This polling model means all network connections originate from your cluster outbound to Anyscale's control plane. No inbound firewall rules are required. For details on required egress domains and ports, see [Networking](networking.md).

## Managed identities and permissions

The operator authenticates to Azure services using a managed identity provisioned by the Terraform module. This identity is scoped to the resources in your resource group.

You can configure permissions at different levels of granularity:

- **Shared identity** — One managed identity covers all workloads in the cloud
- **Granular identity mapping** — Map separate identities to individual users, projects, or workload types via cloud IAM configuration

For information on Entra ID integration and Azure role assignments, see [Identity and access](identity-access.md).

<!-- ## Architecture diagram

TODO: insert architecture diagram before publication -->

## Component summary

| Component | Location | Owner |
|-----------|----------|-------|
| Anyscale console | Anyscale-hosted Azure | Anyscale |
| Scheduling and management APIs | Anyscale-hosted Azure | Anyscale |
| AKS cluster | Your Azure subscription | You |
| Anyscale Kubernetes operator | Your AKS cluster | Anyscale (deployed via Helm) |
| Ray clusters | Your AKS cluster | You |
| Azure Blob Storage / ADLS | Your Azure subscription | You |
| Azure Container Registry | Your Azure subscription | You |
| Azure Key Vault | Your Azure subscription | You |

## Next steps

- [Networking](networking.md) — required egress domains and traffic flow details
- [Identity and access](identity-access.md) — Entra ID SSO and Azure role assignments
- [Quickstart](quickstart.md) — deploy your first Anyscale cloud on Azure
