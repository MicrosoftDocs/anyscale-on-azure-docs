---
title: Anyscale on Azure identity and access
description: Learn how Anyscale on Azure uses Microsoft Entra ID for single sign-on and Azure RBAC built-in roles to control access to cloud resources.
author: kaysieyu
ms.author: kaysieyu
ms.date: 05/06/2026
ms.service: azure-kubernetes-service
ms.topic: concept-article
---

# Anyscale on Azure identity and access

Anyscale on Azure uses Microsoft Entra ID for authentication and Azure role-based access control (RBAC) for authorization. Your team signs in with their existing Azure credentials. The Anyscale Kubernetes operator accesses Azure services through managed identities scoped to your resource group.

## Microsoft Entra ID single sign-on

Anyscale on Azure integrates with Entra ID to provide single sign-on (SSO) across your Azure tenant. Users sign in at [console.azure.anyscale.com](https://console.azure.anyscale.com) using their Azure credentials. You can also launch the console directly from the Azure portal: select **Launch Anyscale** on your Anyscale cloud resource, or select **Go to Anyscale** from the essentials dropdown.

You don't need a separate Anyscale identity.

Authentication uses the OAuth 2.0 authorization code flow with PKCE. At the end of the flow, the Anyscale backend verifies the JWT claims issued by Entra ID and resolves the caller to an Anyscale user account. Users must have at least read access to the Anyscale cloud resource through Azure RBAC before they can sign in.

### How does Entra ID sign-in work?

1. The user navigates to `console.azure.anyscale.com` and selects **Continue with Microsoft**.
1. Entra ID authenticates the user and issues an ID token containing the user's *tenant ID* (`tid`) and *object ID* (`oid`).
1. The Anyscale backend verifies the token and resolves the Entra identity to an Anyscale user account.
1. Anyscale sets a session token in the browser for the duration of the session.

## Managed identities for Azure resource access

When you create an Anyscale cloud resource through the Azure portal, the portal creates two managed identities in your resource group. These identities govern Azure resource access (storage, container registry) by the Anyscale operator and cluster workloads. They're distinct from the user authentication and RBAC authorization described above.

### Anyscale Operator managed identity

This identity governs all actions the operator takes in your Azure subscription, including provisioning nodes for Ray clusters. The Azure portal configures the identity automatically when you [create the cloud resource](quickstart-azure-cli-gateway-envoy.md#step-2-create-an-anyscale-cloud-resource).

### Cluster managed identity

This identity is the default that Anyscale uses when deploying clusters. By default, all workloads share it. You can create additional identities and map them to specific users, projects, or workload types for finer-grained access control.

By default, the cluster managed identity has the following Azure built-in roles within your resource group:

| Role | Scope | Purpose |
|------|-------|---------|
| Storage Blob Data Contributor | Storage account | Read and write artifacts and datasets. |
| AcrPull | Container registry | Pull container images. |

For environments where different teams or workload types need separate permissions, create additional user-assigned managed identities and map them to Anyscale workloads. The mapping uses AKS workload identity. Each Kubernetes service account has a managed identity client ID annotation, and a federated identity credential links the two.

Configure the mapping in the Anyscale console under **Cloud settings > IAM**. For step-by-step instructions, see [Managed identities for AKS](https://docs.anyscale.com/admin/azure/aks-iam) in the Anyscale documentation.

## Service principal for cloud registration

During setup, you create a service principal in your tenant from the Anyscale Entra application. The Anyscale control plane uses this service principal to authenticate against your Azure subscription during the cloud registration flow.

> [!IMPORTANT]
> Anyscale uses the service principal only during cloud creation and initial configuration. It doesn't have ongoing access to your AKS cluster nodes or data plane resources.

## Azure role requirements for setup

The person running the [Quickstart](quickstart-azure-cli-gateway-envoy.md) must have the following permissions on the target subscription:

| Permission | Required for |
|-----------|--------------|
| Subscription Owner, or Contributor plus User Access Administrator | AKS cluster creation and cloud resource setup. |
| Permission to create service principals from external tenants | Running `az ad sp create` in Step 1. |

After setup, day-to-day Anyscale operations such as launching workloads and managing jobs require only the Anyscale RBAC roles assigned in the Azure portal. They don't require elevated Azure permissions.

## Azure built-in roles for Anyscale

Anyscale on Azure provides three built-in Azure roles. Assign these roles to users or groups at the subscription or resource group scope in the Azure portal. Role assignments determine what users can do across the console, CLI, SDK, and API.

| Role | Description |
|------|-------------|
| *Anyscale Platform Administrator* | Full access to all Anyscale resources within the assigned scope, including infrastructure management and workload execution. Includes the `Anyscale.Platform/admin/action` data action for administrative operations such as managing resource quotas and usage budgets. |
| *Anyscale Platform Contributor* | Read and write access to Anyscale clouds, projects, workspaces, jobs, services, compute configs, and images. Doesn't include administrative data actions. |
| *Anyscale Platform Reader* | Read-only access to all Anyscale resources. Required for console sign-in. |

> [!IMPORTANT]
> These are Azure RBAC roles managed in the Azure portal or through the Azure CLI. They're separate from any roles you assign within the Anyscale console.

## Custom role permissions reference

You can create custom Azure RBAC roles to grant a subset of Anyscale permissions. The following actions are available on the `Anyscale.Platform` resource provider:

| Action | Resource |
|--------|----------|
| `Anyscale.Platform/clouds/read` | Clouds |
| `Anyscale.Platform/clouds/write` | Clouds |
| `Anyscale.Platform/clouds/delete` | Clouds |
| `Anyscale.Platform/clouds/cloudResources/read` | Cloud resources |
| `Anyscale.Platform/clouds/cloudResources/write` | Cloud resources |
| `Anyscale.Platform/clouds/cloudResources/delete` | Cloud resources |
| `Anyscale.Platform/clouds/computeConfigs/read` | Compute configs |
| `Anyscale.Platform/clouds/computeConfigs/write` | Compute configs |
| `Anyscale.Platform/clouds/computeConfigs/delete` | Compute configs |
| `Anyscale.Platform/clouds/images/read` | Images |
| `Anyscale.Platform/clouds/images/write` | Images |
| `Anyscale.Platform/clouds/images/delete` | Images |
| `Anyscale.Platform/clouds/projects/read` | Projects |
| `Anyscale.Platform/clouds/projects/write` | Projects |
| `Anyscale.Platform/clouds/projects/delete` | Projects |
| `Anyscale.Platform/clouds/projects/jobs/read` | Jobs |
| `Anyscale.Platform/clouds/projects/jobs/write` | Jobs |
| `Anyscale.Platform/clouds/projects/jobs/delete` | Jobs |
| `Anyscale.Platform/clouds/projects/services/read` | Services |
| `Anyscale.Platform/clouds/projects/services/write` | Services |
| `Anyscale.Platform/clouds/projects/services/delete` | Services |
| `Anyscale.Platform/clouds/projects/workspaces/read` | Workspaces |
| `Anyscale.Platform/clouds/projects/workspaces/write` | Workspaces |
| `Anyscale.Platform/clouds/projects/workspaces/delete` | Workspaces |
| `Anyscale.Platform/admin/action` (data action) | Admin operations |

For instructions on creating a custom role, see [Create or update Azure custom roles](https://learn.microsoft.com/azure/role-based-access-control/custom-roles) in the Azure documentation.

## Next steps

> [!div class="nextstepaction"]
> [Quickstart: Create an Anyscale cloud](quickstart-azure-cli-gateway-envoy.md)

- [Architecture overview](architecture.md) for how managed identities fit into the overall architecture.
- [Networking](networking.md) for egress domains and network security considerations.

### Azure service integration guides in the Anyscale documentation

- [Managed identities for AKS](https://docs.anyscale.com/admin/azure/aks-iam) for per-workload identity mapping.
- [Access blob storage and ADLS](https://docs.anyscale.com/admin/azure/storage) for granting storage roles to managed identities.
- [Configure shared storage with Azure Blob PVC](https://docs.anyscale.com/admin/azure/pvc) for setting up persistent volume claims for AKS.
- [Access Azure Container Registry](https://docs.anyscale.com/admin/azure/container-registry) for attaching ACR pull permissions to your cluster.
