---
title: Configure Private Link for Anyscale on Azure
description: Connect an existing AKS cluster to the Anyscale control plane over Azure Private Link. Create a private endpoint and private DNS, then verify private connectivity.
author: kaysieyu
ms.author: kaysieyu
ms.reviewer: mbender
ms.date: 09/18/2026
ms.service: azure-kubernetes-service
ms.topic: how-to
ms.custom: references_regions
---

# Configure Private Link for Anyscale on Azure

[!INCLUDE [anyscale-public-preview](../../Includes/anyscale-public-preview.md)]

Azure Private Link connects components in your Anyscale data plane to Anyscale-hosted services through a private endpoint in your Azure virtual network. Supported outbound traffic then travels across the Microsoft backbone network instead of the public internet.

The *data plane* is the infrastructure in your Azure account where the Anyscale operator and Ray clusters run. It covers outbound connections from two components in your Azure Kubernetes Service (AKS) cluster:

- **The Anyscale operator**, the controller that registers your cluster and reconciles workloads.
- **Ray clusters**, the workload pods that reach the Anyscale control plane directly.

> [!IMPORTANT]
> Private Link for Anyscale on Azure is currently in preview. This preview is provided without a service level agreement, and some capabilities might be unsupported or constrained.

<!-- DJS 15 Sep 2026: diagram placeholder. Need an Azure-native Private Link data-plane diagram (control plane to AKS through a private endpoint). Source doc marked this [diagram] as TODO. Add as :::image type="complex"::: under media/private-link/ once the asset exists. -->

## Prerequisites

- A running AKS cluster with a subnet available for private endpoints.
- Sufficient permissions to provision private endpoints, configure private DNS zones, and establish virtual network links.

You don't need the Anyscale operator deployed before you start. You can install it later through the ARM template when you create the Anyscale cloud. If the operator is already running, that's also fine.

## Step-by-step onboarding

Complete the following steps to connect an existing AKS cluster to the Anyscale control plane over Private Link. Step 3 relies on values that Anyscale returns in the support response, so start with the support request.

## Step 1: Submit an Azure Help + Support request

To start the Private Link setup, submit a request through Azure Help + Support. Include the following information:

- Your Anyscale organization ID.
- Your Azure subscription ID and tenant ID.
- The region and resource group of the AKS cluster.
- The AKS cluster name and virtual network name.

## Step 2: Receive the Private Link service alias

Anyscale returns the following values. Keep all of them.

| Value | Format | Used in |
|---|---|---|
| Private Link service alias | `<prefix>.<guid>.<region>.azure.privatelinkservice` | Step 3 |
| Private DNS zone | `azure.anyscale-cloud.dev` | Step 3 |

Anyscale issues the alias, including its region, because Anyscale determines which region's Private Link service your private connection uses.

The alias region can differ from your cluster's region. That's expected, not an error. Anyscale currently offers the Private Link service in **East US 2** and **West US 2**. Cross-region private endpoints are supported, and Azure requires no extra steps for them. Provision your private endpoint in the same subscription and region as your virtual network.

## Step 3: Create the private endpoint and private DNS

Provision four resources in the following order:

1. A private endpoint that references the Private Link service alias from step 2.
1. A private DNS zone that matches the domain from step 2.
1. A virtual network link that connects the zone to your virtual network.
1. A wildcard A record that maps to the private endpoint's private IP.

Use any of the three methods in the following section. Each method produces the same configuration.

### Azure portal

1. Search for **Private Link**, go to **Private endpoints**, and select **Create**.
1. On the **Basics** tab, select your subscription and resource group. Enter an endpoint name, and set **Region** to match your virtual network's location. The endpoint must share its virtual network's region, which can differ from the alias region.
1. On the **Resource** tab, choose to connect by resource ID or alias. Enter the Private Link service alias from step 2 in the **Resource ID or alias** field, and add a request message that identifies your cluster. Leave the target sub-resource and resource type empty. Private Link services don't use them.
1. On the **Virtual Network** tab, choose your virtual network and its private endpoint subnet. A dynamic private IP address is sufficient.
1. On the **DNS** tab, set **Integrate with private DNS zone** to **No**.
1. Select **Review + create**.

> [!NOTE]
> Set **Integrate with private DNS zone** to **No**. Automatic integration manages one record for the endpoint's own name, but Anyscale needs a wildcard. The control plane and the workload image registry are different hostnames under the same domain. You create the zone and the wildcard record yourself in the next three steps.

To configure the DNS zone manually:

1. Go to **Private DNS zones** and select **Create**. Set **Name** to the domain from step 2. Region settings don't affect lookups, because private DNS zones are global.
1. Open your new zone, select **Virtual network links**, and select **Add**. Point to your virtual network, and leave **Enable auto registration** unchecked.
1. In the zone menu, select **Record sets**, then **Add**. Set **Name** to `*`, **Type** to **A**, and **TTL** to 60 seconds. Enter the private endpoint IP address, which appears on the endpoint's **Overview** tab or in its network interface settings.

### Azure CLI

Replace each `<placeholder>` with your value, including the angle brackets.

```azurecli
# 1. Provision the private endpoint by alias. Anyscale approves the request manually.
az network private-endpoint create \
  --name "<prefix>-anyscale-pe" \
  --resource-group "<resource-group>" \
  --location "<region>" \
  --vnet-name "<vnet>" --subnet "<subnet>" \
  --private-connection-resource-id "<alias-from-step-2>" \
  --connection-name "<prefix>-anyscale" \
  --manual-request true \
  --request-message "Anyscale cloud for AKS cluster <cluster>"

# 2. Create the DNS zone and link it to the virtual network.
az network private-dns zone create --resource-group "<resource-group>" --name "<zone>"

az network private-dns link vnet create \
  --resource-group "<resource-group>" --zone-name "<zone>" \
  --name "<prefix>-anyscale" \
  --virtual-network "<vnet>" --registration-enabled false

# 3. Read the private IP from the network interface and map the wildcard A record.
#    Query networkInterfaces directly, because customDnsConfigs stays empty
#    for Private Link service resources.
NIC_ID=$(az network private-endpoint show \
  --name "<prefix>-anyscale-pe" --resource-group "<resource-group>" \
  --query 'networkInterfaces[0].id' -o tsv)

PE_IP=$(az network nic show --ids "$NIC_ID" \
  --query 'ipConfigurations[0].privateIPAddress' -o tsv)

az network private-dns record-set a add-record \
  --resource-group "<resource-group>" --zone-name "<zone>" \
  --record-set-name "*" --ipv4-address "$PE_IP"
```

### Terraform

This configuration uses the `azurerm` provider. Supply the five variables to integrate it into existing infrastructure without assuming predefined virtual network resources.

```terraform
###############################################################################
# Anyscale control plane over Azure Private Link — consumer (customer) side
#
# Provider compatibility
# ----------------------
# Pinned to azurerm v4. On azurerm >= 5.0 two resources below changed schema:
#   azurerm_private_dns_a_record:
#     v4: zone_name = <name>             + resource_group_name = <rg>
#     v5: private_dns_zone_id = azurerm_private_dns_zone.<name>.id
#   azurerm_private_dns_zone_virtual_network_link:
#     v4: private_dns_zone_name = <name> + resource_group_name = <rg>
#     v5: private_dns_zone_id = azurerm_private_dns_zone.<name>.id
# Everything else here is identical across v4 and v5.
###############################################################################

terraform {
  required_version = ">= 1.5"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.0" # see "Provider compatibility" above before moving to v5
    }
  }
}

provider "azurerm" {
  features {}
}

variable "anyscale_privatelink_service_alias" {
  description = "Private Link service alias from step 2."
  type        = string
}

variable "anyscale_private_dns_zone_name" {
  description = "Private DNS zone from step 2, for example azure.anyscale-cloud.dev"
  type        = string
}

variable "name_prefix"                { type = string }
variable "resource_group_name"        { type = string }
variable "location"                   { type = string }
variable "vnet_id"                    { type = string }
variable "private_endpoint_subnet_id" { type = string }

# is_manual_connection must be true. Connecting by alias across tenants is a
# request that Anyscale approves, not an automatic attachment. request_message
# is optional when it's true, and Azure caps it at 140 characters.
resource "azurerm_private_endpoint" "anyscale" {
  name                = "${var.name_prefix}-anyscale-pe"
  resource_group_name = var.resource_group_name
  location            = var.location
  subnet_id           = var.private_endpoint_subnet_id

  private_service_connection {
    name                              = "${var.name_prefix}-anyscale"
    is_manual_connection              = true
    private_connection_resource_alias = var.anyscale_privatelink_service_alias
    request_message                   = "Anyscale cloud for ${var.name_prefix}"
  }
}

# Private DNS zones are global. They take no location argument.
resource "azurerm_private_dns_zone" "anyscale" {
  name                = var.anyscale_private_dns_zone_name
  resource_group_name = var.resource_group_name
}

# Without this link the zone exists but resolves for nothing, and nothing errors.
resource "azurerm_private_dns_zone_virtual_network_link" "anyscale" {
  name                  = "${var.name_prefix}-anyscale"
  resource_group_name   = var.resource_group_name
  private_dns_zone_name = azurerm_private_dns_zone.anyscale.name
  virtual_network_id    = var.vnet_id
  registration_enabled  = false
}

# Wildcard: the zone is authoritative for the whole domain inside the virtual
# network, so any name without a record fails to resolve rather than falling
# back to public DNS. A short TTL keeps a rebuilt endpoint from being cached.
resource "azurerm_private_dns_a_record" "anyscale" {
  name                = "*"
  zone_name           = azurerm_private_dns_zone.anyscale.name
  resource_group_name = var.resource_group_name
  ttl                 = 60
  records             = [azurerm_private_endpoint.anyscale.private_service_connection[0].private_ip_address]
}

output "anyscale_private_endpoint_id" {
  value = azurerm_private_endpoint.anyscale.id
}

output "anyscale_private_endpoint_ip" {
  value = azurerm_private_endpoint.anyscale.private_service_connection[0].private_ip_address
}
```

Keep these points in mind across all three methods:

- Creating a private endpoint opens a connection request rather than an immediate attachment. Azure reports a successful deployment while the connection status stays **Pending** until you complete step 4.
- Azure allocates the private IP when it creates the endpoint, before approval, so you can map the DNS record right away.
- Manual DNS management requires updating the static A record if you recreate the endpoint. Terraform updates the record mapping on the next apply. Portal and CLI deployments require an explicit update.

## Step 4: Wait for connection approval

Update the open support ticket with your private endpoint's resource ID and its allocated private IP.

To find the resource ID, list the private endpoints in your resource group or show a specific endpoint:

```azurecli
# List private endpoints to find the resource ID.
az network private-endpoint list --resource-group <resource-group> --output table

# Or show one endpoint's resource ID.
az network private-endpoint show \
  --name <private-endpoint-name> --resource-group <resource-group> \
  --query id --output tsv
```

Check the connection status:

```azurecli
az network private-endpoint show \
  --name <private-endpoint-name> \
  --resource-group <resource-group> \
  --query 'manualPrivateLinkServiceConnections[0].privateLinkServiceConnectionState'
```

- **Pending**: authorization is still pending.
- **Approved**: you can proceed.
- **Rejected** or **Disconnected**: resubmit the request in your support ticket.

> [!IMPORTANT]
> A green deployment status doesn't confirm connectivity. Azure provisions the resource in a Pending state even when Terraform completes successfully. Traffic can't pass until Anyscale grants authorization.

## Step 5: Verify the hostname resolves privately

Verify resolution before you create the Anyscale cloud. This check confirms that your private endpoint, DNS zone, and virtual network link work together.

Run the check from inside the virtual network. A private DNS zone applies only to networks it's linked to. From a workstation elsewhere, you get Anyscale's public addresses, which is correct and proves nothing.

From a pod in the cluster, run:

```bash
kubectl run dnstest --rm -it --image=busybox:1.36 --restart=Never -- \
  nslookup cld-<id>.azure.anyscale-cloud.dev
```

Expect a single address that matches the private endpoint IP from step 3 and falls inside your private endpoint subnet.

To narrow a failed lookup:

```azurecli
# 1. Is the zone linked to this virtual network?
az network private-dns link vnet list --resource-group <resource-group> --zone-name <zone> -o table

# 2. Does the record exist and point at the endpoint IP?
az network private-dns record-set a list --resource-group <resource-group> --zone-name <zone> -o table
```

```bash
# 3. Bypass CoreDNS and ask the Azure resolver directly, from a node.
dig @168.63.129.16 +short cld-<id>.azure.anyscale-cloud.dev
```

## Step 6: Create the Anyscale cloud resource

Follow the [quickstart to create an Anyscale cloud resource](quickstart-azure-cli.md#create-an-anyscale-cloud-resource). After the cloud is created, you receive a cloud ID in the format `cld_xxx`.

## Step 7: Share your identifiers in the support ticket

Provide the following identifiers in the existing support ticket:

- Your cloud ID, in the format `cld_...`.
- Your cloud resource ID, in the format `cldrsrc_...`.
- Your organization ID, in the format `org_...`.

Anyscale uses these identifiers to complete the connection on the control plane side.

## Troubleshooting

Work through these checks if traffic doesn't route privately after approval.

### Connection stays in the Pending state

The private endpoint can't pass traffic until Anyscale approves the connection request. Confirm the status with the command in [step 4](#step-4-wait-for-connection-approval). If the status is **Rejected** or **Disconnected**, resubmit the request through your support ticket. A **Disconnected** endpoint can't be restored by re-approval. Recreate the private endpoint instead.

### The hostname resolves to a public address

If the lookup in step 5 returns a public address, the virtual network link or a CoreDNS rule is the likely cause. Confirm that the private DNS zone is linked to the virtual network, and run the check from inside the virtual network. A lookup from elsewhere returns public addresses by design.

### The lookup returns NXDOMAIN or the wrong IP

An `NXDOMAIN` result means the zone is active but has no matching record. Confirm that the wildcard A record targets the private endpoint IP. An incorrect IP usually means the record is stale after the endpoint was recreated. Update the A record to the current endpoint IP.

### The operator still uses the public control plane

If the operator routes over the public internet after approval, `global.controlPlaneURL` is likely unset or wrong. Set it to the full control plane URL, in the form `https://cld-<id>.azure.anyscale-cloud.dev`, then apply the change with a Helm upgrade rather than an infrastructure change. This misconfiguration often comes from an older ARM template that deployed the operator without the value.

### Authentication fails instead of the network

The operator authenticates to Microsoft Entra ID over public egress. Private Link and private endpoints don't cover token retrieval from Entra ID, so that traffic never routes through your private connection. If the operator logs show authentication failures rather than network errors and you restrict outbound traffic, allow egress to Entra ID. Choose one path: allow the `AzureActiveDirectory` service tag through a network security group, or allow `login.microsoftonline.com` through an Azure Firewall rule.

### Workload pods still egress publicly

If the operator routes internally but workload pods still egress over the public internet, the organization-level Private Link configuration needs a change on the Anyscale side. Confirm that you completed step 7, then contact Anyscale support with your cloud ID, cloud resource ID, and organization ID.

## Related content

- [Anyscale on Azure networking](networking.md)
- [Anyscale on Azure architecture overview](architecture.md)
- [Quickstart: Deploy Anyscale on Azure](quickstart-azure-cli.md)
