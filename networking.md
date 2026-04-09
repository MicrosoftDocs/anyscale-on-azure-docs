---
title: Anyscale on Azure networking | Microsoft Learn
description: Understand the network traffic flows, required egress domains, and Kubernetes ingress configuration for Anyscale on Azure deployments.
author: tdwyer
ms.author: tdwyer
ms.date: 04/03/2026
ms.topic: conceptual
---

# Networking

Anyscale on Azure uses an egress-only network model. All connections originate from the Anyscale Kubernetes operator and Ray clusters in your AKS cluster outbound to Anyscale's control plane and external services. No inbound firewall rules are required to reach your cluster from Anyscale.

## Network traffic flows

Four primary traffic paths connect Anyscale's components:

:::image type="content" source="media/networking/azure-network-flows.png" alt-text="Four network flows between client, Anyscale control plane, AKS cluster with operator and Ray, and Azure storage and registry resources.":::

| # | Flow | Description |
|---|------|-------------|
| 1 | Client → control plane | Console access, CLI commands, and SDK calls via `console.azure.anyscale.com` |
| 2 | Client → AKS cluster | Dashboard access, job submission, and service requests via Azure Load Balancer |
| 3 | AKS cluster → control plane | Operator polling for scheduling instructions; telemetry and health reporting |
| 4 | AKS cluster → Azure resources | Application data access (Azure Blob Storage, Container Registry, Key Vault) |

## Operator and Ray cluster communication

The Anyscale Kubernetes operator polls Anyscale's control plane for pending operations and reports Ray cluster health back. This polling pattern means all connections from your cluster are outbound — no inbound rules are needed on your AKS nodes.

:::image type="content" source="media/networking/azure-aks-to-control-plane.png" alt-text="Operator Pod in customer AKS cluster with two outbound connections to Anyscale control plane: operator polling (1) and Ray cluster health reporting (2).":::

## Client access to Ray clusters

Clients reach Ray cluster head nodes and Anyscale Services through an Azure Load Balancer fronting the Nginx ingress controller in your AKS cluster.

:::image type="content" source="media/networking/azure-client-flows.png" alt-text="Authenticated user with two flows: to Anyscale control plane (1) and to Ray cluster head and worker nodes via Azure Load Balancer in customer AKS (2).":::

The ingress controller terminates TLS and forwards traffic to port 80 of the head pod. DNS for `*.i.anyscaleuserdata.com` (head node access) and `*.s.anyscaleuserdata.com` (service requests) resolves to the load balancer address.

> [!IMPORTANT]
> The Nginx ingress controller requires a Layer 4 (TCP) load balancer. Azure Load Balancer (standard SKU) satisfies this requirement. Application Gateway is not supported as the primary ingress load balancer.

## Container image flow

The Anyscale operator pulls base images from Anyscale's container registry. Workloads can also pull from your Azure Container Registry (ACR).

:::image type="content" source="media/networking/azure-container-images.png" alt-text="Anyscale Operator pulling images from Anyscale Container Registry (1), with Ray cluster nodes in AKS (2) and optional Azure Container Registry pull (3).":::

## Required egress domains

Configure your AKS cluster's egress rules to allow outbound HTTPS (port 443) traffic to the following domains.

### Anyscale control plane

| Domain | Purpose |
|--------|---------|
| `console.azure.anyscale.com` | Anyscale console and API endpoint |
| `*.azure.anyscale-cloud.dev` | Operator polling, health checks, and operational reporting |
| `grafana-*.azure.anyscale-cloud.dev` | Metrics collection and visualization |
| `registry-*.azure.anyscale-cloud.dev` | Anyscale container image distribution |

### Ray cluster routing

| Domain | Purpose |
|--------|---------|
| `*.i.anyscaleuserdata.com` | Ray dashboard, terminal, and Jupyter access |
| `vscode-*.i.anyscaleuserdata.com` | VS Code access |
| `*.s.anyscaleuserdata.com` | Anyscale Services request routing |

### Strict egress filtering

Organizations that require per-cloud egress filtering can replace the `*.azure.anyscale-cloud.dev` wildcard with cloud-ID-specific entries. This requires maintaining a separate entry for each Anyscale cloud in your environment.

## TLS and certificate management

Anyscale manages TLS certificates automatically and rotates them at minimum every three months. The ingress controller must be able to read the certificate secret, which may reside in a different namespace from the ingress controller itself.

## Private networking considerations

For clusters without public internet access, route all egress traffic through an Azure NAT Gateway or equivalent. Ensure your network security group (NSG) rules and any Azure Firewall policies allow outbound traffic to all domains listed in [Required egress domains](#required-egress-domains).

Private clusters (no public node IPs) are supported. Configure the ingress controller's load balancer as internal and use a private DNS zone with VPN or ExpressRoute for client access.

## Next steps

- [Architecture overview](architecture.md) — how the control plane and data plane interact
- [Identity and access](identity-access.md) — managed identity and Entra ID configuration
- [Quickstart](quickstart.md) — deploy your first Anyscale cloud on Azure
