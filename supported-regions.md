---
title: Anyscale on Azure supported regions | Microsoft Learn
description: View the Azure regions where Anyscale on Azure is available and find links to Microsoft's regional availability documentation for GPU and compute SKUs.
author: kaysieyu
ms.author: kaysieyu
ms.date: 04/03/2026
ms.topic: reference
---

# Supported regions

Anyscale on Azure is available in the following Azure regions during Public Preview. Contact [Anyscale support](https://docs.anyscale.com/support) to request enrollment and specify your preferred region.

## Available regions

| Region | Azure region name |
|--------|------------------|
| East US | `eastus` |
| East US 2 | `eastus2` |
| West US 2 | `westus2` |
| West US 3 | `westus3` |
| Central US | `centralus` |
| North Central US | `northcentralus` |
| South Central US | `southcentralus` |
| West Europe | `westeurope` |
| North Europe | `northeurope` |
| Southeast Asia | `southeastasia` |

> [!NOTE]
> This table is a placeholder. Replace with the confirmed list of supported regions before publication. Contact Anyscale to verify which regions are available at launch.

## GPU and compute availability

GPU and high-performance compute SKU availability varies by region. Anyscale does not maintain its own matrix of Azure VM SKU availability. For current regional availability of GPU instances (NC, ND, NV series), see the Microsoft documentation:

- [Azure GPU-optimized virtual machine sizes](/azure/virtual-machines/sizes-gpu)
- [Products available by region](https://azure.microsoft.com/explore/global-infrastructure/products-by-region/)
- [Azure Kubernetes Service supported VM sizes](/azure/aks/quotas-skus-regions)

## Regional behavior

All Anyscale clouds are region-specific. A cloud created in `eastus` can only run workloads on AKS node pools in `eastus`. Cross-region replication and multi-region clusters are not supported in Public Preview.

The region you select when creating the AKS cluster and Anyscale cloud resource sets the region for all resources in the cloud.

## Request a new region

If your required region is not listed, contact Anyscale to request support. Provide your Azure subscription ID and the target region.

## Next steps

- [Quickstart](quickstart.md) — deploy your first Anyscale cloud
- [Architecture overview](architecture.md) — understand the deployment model
