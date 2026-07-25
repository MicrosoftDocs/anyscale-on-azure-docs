---
title: Anyscale on Azure supported regions
description: View the Azure regions where Anyscale on Azure is available and find links to Microsoft regional availability documentation for GPU and compute SKUs.
author: kaysieyu
ms.author: kaysieyu
ms.reviewer: mbender
ms.date: 07/23/2026
ms.service: azure-kubernetes-service
ms.topic: reference
ms.custom: references_regions
---

# Anyscale on Azure supported regions

[!INCLUDE [anyscale-public-preview](../../Includes/anyscale-public-preview.md)]

Anyscale on Azure is available in the following Azure regions during Public Preview. If you need a region that isn't listed, [contact support](support-model.md) to request it.

## Available regions and region names

| Region | Azure region name |
|--------|------------------|
| West Central US | `westcentralus` |
| East US | `eastus` |
| East US 2 | `eastus2` |
| West US 2 | `westus2` |
| West US 3 | `westus3` |
| South Central US | `southcentralus` |
| West Europe | `westeurope` |
| North Europe | `northeurope` |
| Sweden Central | `swedencentral` |
| UK South | `uksouth` |
| Australia East | `australiaeast` |
| Southeast Asia | `southeastasia` |

## GPU and compute availability

GPU and high-performance compute SKU availability varies by region. Anyscale doesn't maintain its own matrix of Azure VM SKU availability. For current regional availability of GPU instances, such as the NC, ND, and NV series, see the Microsoft documentation:

- [Azure GPU-optimized virtual machine sizes](/azure/virtual-machines/sizes-gpu)
- [Products available by region](https://azure.microsoft.com/explore/global-infrastructure/products-by-region/)
- [Azure Kubernetes Service supported VM sizes](/azure/aks/quotas-skus-regions)

## Quota and SKU availability

Azure enforces vCPU quota by VM family and by region. GPU SKUs typically require quota approval before you can deploy them in a given region. To check current quota and request increases:

- [Increase VM-family vCPU quotas](/azure/quotas/per-vm-quota-requests) for GPU and other VM-family quotas.
- [Increase regional vCPU quotas](/azure/quotas/regional-quota-requests) for region-wide capacity.
- [Quickstart: Request a quota increase](/azure/quotas/quickstart-increase-quota-portal) for the general portal flow.

For AKS-specific cluster limits, see [Quotas, virtual machine size restrictions, and region availability in AKS](/azure/aks/quotas-skus-regions).

## Regional behavior and constraints

All Anyscale clouds are region-specific. A cloud created in `eastus` can only run workloads on AKS node pools in `eastus`. Public Preview doesn't support cross-region replication or multi-region clusters.

The region you select when you create the AKS cluster and Anyscale cloud resource sets the region for all resources in the cloud.

## Data residency and the control plane

Each Anyscale cloud runs in the Azure region you select, within your Azure subscription. Your application data and datasets stay in your data plane, in that region.

The Anyscale control plane runs separately in the United States, not in the region you select. Operational metadata flows from your data plane to that control plane, regardless of the region you select. This metadata includes system logs, observability metrics, and cluster state. When log ingestion is enabled, it also includes structured application logs. For the full breakdown of what Anyscale stores in the control plane versus what stays in your data plane, see [Data classification](https://docs.anyscale.com/administration/security-and-compliance/data-classification).

> [!IMPORTANT]
> If your deployment is subject to data-residency or regulatory requirements, review what operational metadata leaves your data plane before you select a region. For how Anyscale handles your data, see the [Anyscale Privacy Policy](https://www.anyscale.com/privacy-policy) and [Terms and privacy](../../legal.md).

## Next steps

- [Quickstart](quickstart-azure-cli.md) to deploy your first Anyscale cloud.
- [Architecture overview](architecture.md) to understand the deployment model.
