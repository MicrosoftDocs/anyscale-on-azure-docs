---
title: Identity and access in Anyscale on Azure
description: Learn how Anyscale on Azure integrates with Microsoft Entra ID for single sign-on and uses Azure managed identities and roles to control access to cloud resources.
author: kaysieyu
ms.author: kaysieyu
ms.date: 04/29/2026
ms.service: azure-kubernetes-service
ms.topic: concept-article
---

# Identity and access

Anyscale on Azure uses Microsoft Entra ID for authentication and Azure managed identities for resource access. Your team signs in with their existing Azure credentials. The Anyscale Kubernetes operator accesses Azure services through managed identities scoped to your resource group.

## Microsoft Entra ID single sign-on

Anyscale on Azure integrates with Entra ID to provide single sign-on (SSO) across your Azure tenant. Users sign in at [console.azure.anyscale.com](https://console.azure.anyscale.com) using their Azure credentials. You don't need a separate Anyscale identity.

### How it works

1. The user navigates to `console.azure.anyscale.com` and selects **Continue with Microsoft**.
1. Entra ID authenticates the user and issues a token.
1. Anyscale validates the token and maps the user to their Anyscale organization.

The Azure portal configures SSO automatically when you create your Anyscale cloud. You don't need to register an additional Entra ID application.

## Managed identities and their roles

When you create an Anyscale cloud resource through the Azure portal, the portal creates two distinct managed identities in your resource group:

### Anyscale Operator managed identity

This identity governs all actions the operator takes in your Azure subscription, including provisioning nodes for Ray clusters. The Azure portal configures the identity automatically when you create the cloud resource.

### Cluster managed identity

This identity is the default that Anyscale uses when deploying clusters. By default, all workloads share it. You can create additional identities and map them to specific users, projects, or workload types for finer-grained access control.

### Default configuration

By default, all workloads in the cloud share a single cluster managed identity. This identity has the following Azure built-in roles within your resource group:

| Role | Scope | Purpose |
|------|-------|---------|
| Storage Blob Data Contributor | Storage account | Read and write artifacts and datasets. |
| AcrPull | Container registry | Pull container images. |

### Granular identity mapping

For environments where different teams or workload types need separate permissions, create additional user-assigned managed identities and map them to Anyscale workloads. The mapping uses AKS workload identity. Each Kubernetes service account has a managed identity client ID annotation, and a federated identity credential links the two.

Configure the mapping in the Anyscale console under **Cloud settings > IAM**. For step-by-step instructions, see [Managed identities for AKS](https://docs.anyscale.com/admin/azure/aks-iam) in the Anyscale documentation.

## Service principal for cloud registration

During setup, you create a service principal in your tenant from the Anyscale Entra application (app ID `086bc555-6989-4362-ba30-fded273e432b`). The Anyscale control plane uses this service principal to authenticate against your Azure subscription during the cloud registration flow.

> [!IMPORTANT]
> Anyscale uses the service principal only during cloud creation and initial configuration. It doesn't have ongoing access to your AKS cluster nodes or data plane resources.

## Azure role requirements for setup

The person running the [Quickstart](quickstart-azure-cli-gateway-envoy.md) must have the following permissions on the target subscription:

| Permission | Required for |
|-----------|--------------|
| Subscription Owner, or Contributor plus User Access Administrator | AKS cluster creation and cloud resource setup. |
| Permission to create service principals from external tenants | Running `az ad sp create` in Step 1. |

After setup, day-to-day Anyscale operations such as launching workloads and managing jobs require only the Anyscale-level roles assigned through the Anyscale console. They don't require elevated Azure permissions.

## Anyscale platform roles

Anyscale roles control access within the Anyscale platform. These roles are separate from Azure role-based access control (RBAC) and are managed in the Anyscale console.

For a full reference of Anyscale platform roles and permissions, see the [Anyscale documentation](https://docs.anyscale.com).

## Next steps

- [Architecture overview](architecture.md) for how managed identities fit into the overall architecture.
- [Networking](networking.md) for egress domains and network security considerations.
- [Quickstart](quickstart-azure-cli-gateway-envoy.md) for step-by-step setup, including service principal creation.

### Azure service integration guides in the Anyscale documentation

- [Managed identities for AKS](https://docs.anyscale.com/admin/azure/aks-iam) for per-workload identity mapping.
- [Access blob storage and ADLS](https://docs.anyscale.com/admin/azure/storage) for granting storage roles to managed identities.
- [Configure shared storage with Azure Blob PVC](https://docs.anyscale.com/admin/azure/pvc) for setting up persistent volume claims for AKS.
- [Access Azure Container Registry](https://docs.anyscale.com/admin/azure/container-registry) for attaching ACR pull permissions to your cluster.

