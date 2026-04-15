---
title: Identity and access in Anyscale on Azure | Microsoft Learn
description: Learn how Anyscale on Azure integrates with Microsoft Entra ID for single sign-on and uses Azure managed identities and built-in roles to control access to resources.
author: kaysieyu
ms.author: kaysieyu
ms.date: 04/03/2026
ms.topic: conceptual
---

# Identity and access

Anyscale on Azure uses Microsoft Entra ID for authentication and Azure managed identities for resource access. Your team signs in with their existing Azure credentials, and the Anyscale Kubernetes operator accesses Azure services through managed identities scoped to your resource group.

## Entra ID single sign-on

Anyscale on Azure integrates with Entra ID to provide single sign-on (SSO) across your Azure tenant. Users sign in at [console.azure.anyscale.com](https://console.azure.anyscale.com) using their Azure credentials — no separate Anyscale identity is required.

**How it works:**

1. The user navigates to `console.azure.anyscale.com` and selects **Sign in with Microsoft**.
2. Entra ID authenticates the user and issues a token.
3. Anyscale validates the token and maps the user to their Anyscale organization.

SSO is configured automatically when you create your Anyscale cloud through the Azure portal. No additional Entra ID application registration is required from your side.

## Managed identities

When you create an Anyscale cloud resource through the Azure portal, the portal creates Azure managed identities in your resource group. There are two distinct identities:

**Anyscale Operator managed identity** — Governs all actions the operator takes in your Azure subscription, including provisioning nodes for Ray clusters. This identity is configured during setup and referenced in your cloud configuration YAML.

**Cluster managed identity** — The default identity used by Anyscale when deploying clusters. By default, all workloads share this identity. You can create additional identities and map them to specific users, projects, or workload types for finer-grained access control.

### Default configuration

By default, a single cluster managed identity is shared across all workloads in the cloud. This identity is granted the following Azure built-in roles within your resource group:

| Role | Scope | Purpose |
|------|-------|---------|
| Storage Blob Data Contributor | Storage account | Read and write artifacts and datasets |
| AcrPull | Container registry | Pull container images |
| Key Vault Secrets User | Key Vault | Read secrets at runtime |

### Granular identity mapping

For environments where different teams or workload types need separate permissions, create additional user-assigned managed identities and map them to Anyscale workloads. The mapping uses AKS workload identity: each Kubernetes service account is annotated with a managed identity client ID, and a federated identity credential links the two.

This is configured in the Anyscale console under **Cloud settings > IAM**. For step-by-step instructions, see [Managed identities for AKS](https://docs.anyscale.com/admin/azure/aks-iam) in the Anyscale documentation.

## Service principal

During setup, you create a service principal in your tenant from Anyscale's Entra application (app ID `086bc555-6989-4362-ba30-fded273e432b`). This service principal allows Anyscale's control plane to authenticate against your Azure subscription during the cloud registration flow.

> [!IMPORTANT]
> The service principal is used only during cloud creation and initial configuration. It does not have ongoing access to your AKS cluster nodes or data plane resources.

## Azure role requirements for setup

The person running the [Quickstart](quickstart.md) must have the following permissions on the target subscription:

| Permission | Required for |
|-----------|--------------|
| Subscription Owner or Contributor + User Access Administrator | AKS cluster creation and cloud resource setup |
| Permission to create service principals from external tenants | Running `az ad sp create` in Step 1 |

After setup is complete, day-to-day Anyscale operations (launching workloads, managing jobs) require only the Anyscale-level roles assigned through the Anyscale console, not elevated Azure permissions.

## Anyscale platform roles

Within the Anyscale platform, access is controlled by Anyscale roles. These are separate from Azure RBAC and are managed in the Anyscale console.

For a full reference of Anyscale platform roles and permissions, see [Anyscale role-based access control](https://docs.anyscale.com/organization/rbac).

## Next steps

- [Architecture overview](architecture.md) — how managed identities fit into the overall architecture
- [Networking](networking.md) — egress domains and network security considerations
- [Quickstart](quickstart.md) — step-by-step setup including service principal creation

### Azure service integration guides (Anyscale docs)

- [Managed identities for AKS](https://docs.anyscale.com/admin/azure/aks-iam) — configure per-workload identity mapping
- [Access blob storage and ADLS](https://docs.anyscale.com/admin/azure/storage) — grant storage roles to managed identities
- [Configure shared storage with Azure Blob PVC](https://docs.anyscale.com/admin/azure/pvc) — set up persistent volume claims for AKS
- [Access Azure Container Registry](https://docs.anyscale.com/admin/azure/container-registry) — attach ACR pull permissions to your cluster
- [Use Key Vault secrets](https://docs.anyscale.com/admin/azure/key-vault) — grant Key Vault Secrets User role to managed identities
