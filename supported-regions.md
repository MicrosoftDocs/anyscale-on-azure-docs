---
title: Anyscale on Azure Supported Regions | Microsoft Learn
description: View the Azure regions where Anyscale on Azure is available and find links to Microsoft's regional availability documentation for GPU and compute SKUs.
author: kaysieyu
ms.author: kaysieyu
ms.date: 04/22/2026
ms.topic: reference
---

# Supported regions

Anyscale on Azure is available in the following Azure regions during Public Preview. Contact [Anyscale support](https://www.anyscale.com/support) to request enrollment and specify your preferred region.

## Available regions and region names

| Region | Azure region name |
|--------|------------------|
| West Central US | `westcentralus` |
| East US | `eastus` |
| East US 2 | `eastus2` |
| West US 2 | `westus2` |
| West US 3 | `westus3` |

## GPU and compute availability

GPU and high-performance compute SKU availability varies by region. Anyscale does not maintain its own matrix of Azure VM SKU availability. For current regional availability of GPU instances (NC, ND, NV series), see the Microsoft documentation:

- [Azure GPU-optimized virtual machine sizes](/azure/virtual-machines/sizes-gpu)
- [Products available by region](https://azure.microsoft.com/explore/global-infrastructure/products-by-region/)
- [Azure Kubernetes Service supported VM sizes](/azure/aks/quotas-skus-regions)

## Regional behavior and constraints

All Anyscale clouds are region-specific. A cloud created in `eastus` can only run workloads on AKS node pools in `eastus`. Cross-region replication and multi-region clusters are not supported in Public Preview.

The region you select when creating the AKS cluster and Anyscale cloud resource sets the region for all resources in the cloud.

## Next steps

- [Quickstart](quickstart-azure-cli-gateway-envoy.md)—deploy your first Anyscale cloud
- [Architecture overview](architecture.md)—understand the deployment model
