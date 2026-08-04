---
title: Anyscale on Azure support model
description: Learn how to get support for Anyscale on Azure. Azure Support handles support requests and routes them to the appropriate team.
author: kaysieyu
ms.author: kaysieyu
ms.reviewer: mbender
ms.date: 08/03/2026
ms.service: azure-kubernetes-service
ms.topic: concept-article
ms.custom: references_regions
---

# Anyscale on Azure support model

[!INCLUDE [anyscale-public-preview](../../Includes/anyscale-public-preview.md)]

Azure Support handles all Anyscale on Azure support requests. Open a support request through the Azure portal whether your issue concerns Azure infrastructure, AKS configuration, or the Anyscale platform itself. The support team routes your case appropriately.

> [!NOTE]
> Customers with direct support channels set up with Anyscale Support can contact Anyscale Support directly for Anyscale platform issues. At launch, Anyscale on Azure provides [Enterprise tier](https://www.anyscale.com/support) service level agreements (SLAs).

## How to get support

1. Sign in to the [Azure portal](https://portal.azure.com).
1. Select **Portal menu** in the upper-left corner, then select **Help + support** at the bottom of the navigation panel.
1. Select **Create a support request**.
1. For **Which service are you having an issue with?**, select **None of the above**.
1. In the **Select a service** dropdown, search for `Anyscale` and select **Anyscale on Azure** under **Compute**.

   :::image type="content" source="media/support/support-select-service.png" alt-text="Support + troubleshooting pane with None of the above selected and Anyscale on Azure listed under Compute in the Select a service dropdown.":::

1. Follow the remaining prompts to describe your issue and submit the request.

> [!IMPORTANT]
> Always select **Anyscale on Azure** in the **Select a service** dropdown when you open a support request in the Azure portal. Selecting any other service routes your case to the wrong team and delays resolution.

For guidance on opening Azure support requests, see [Create an Azure support request](/azure/azure-portal/supportability/how-to-create-azure-support-request).

## Support scope during Public Preview

Anyscale on Azure is in Public Preview. Anyscale provides support during Public Preview on a best-effort basis. For information on Azure support plans and their coverage, see [Azure Support Plans](https://azure.microsoft.com/support/plans/).

Before opening a support request, review the [Public Preview limitations](overview.md#public-preview-limitations) to confirm the behavior isn't a known constraint. Anyscale CLI commands that write to Azure cloud resources aren't supported during Public Preview, for example.

## Troubleshooting

For self-serve guidance on common issues, see the [Anyscale on Azure knowledge base](https://docs.anyscale.com/kb/azure).

## Next steps

- [Quickstart](quickstart-azure-cli.md) to deploy your first Anyscale cloud.
- [Architecture overview](architecture.md) to understand the deployment model.
- [Supported regions](supported-regions.md).
