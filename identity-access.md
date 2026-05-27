---
title: Anyscale on Azure identity and access
description: Learn how Anyscale on Azure uses Microsoft Entra ID for single sign-on and Azure RBAC built-in roles to control access to cloud resources.
author: kaysieyu
ms.author: kaysieyu
ms.date: 05/21/2026
ms.service: azure-kubernetes-service
ms.topic: concept-article
---

# Anyscale on Azure identity and access

[!INCLUDE [anyscale-public-preview](Includes/anyscale-public-preview.md)]

Anyscale on Azure uses Microsoft Entra ID for authentication and Azure role-based access control (RBAC) for authorization. Your team signs in with their existing Azure credentials. The Anyscale Kubernetes operator accesses Azure services through managed identities scoped to your resource group.

## Microsoft Entra ID single sign-on

Anyscale on Azure integrates with Microsoft Entra ID to provide single sign-on (SSO) across your Azure tenant. Users sign in at [console.azure.anyscale.com](https://console.azure.anyscale.com) using their Azure credentials. You can also launch the console directly from the Azure portal: select **Launch Anyscale** on your Anyscale cloud resource, or select **Go to Anyscale** from the essentials dropdown.

You don't need a separate Anyscale identity.

Authentication uses the OAuth 2.0 authorization code flow with PKCE. At the end of the flow, the Anyscale backend verifies the JWT claims issued by Microsoft Entra ID and resolves the caller to an Anyscale user account. Users must have at least read access to the Anyscale cloud resource through Azure RBAC before they can sign in.

### How does Microsoft Entra ID sign-in work?

1. The user navigates to `console.azure.anyscale.com` and selects **Continue with Microsoft**.
1. Microsoft Entra ID authenticates the user and issues an ID token containing the user's *tenant ID* (`tid`) and *object ID* (`oid`).
1. The Anyscale backend verifies the token and resolves the Microsoft Entra identity to an Anyscale user account.
1. Anyscale sets a session token in the browser for the duration of the session.

> [!IMPORTANT]
> To sign in to the Anyscale console, a user must hold at least the **Anyscale Platform Reader** role on the Anyscale cloud resource through Azure RBAC. Without this assignment, the sign-in flow completes against Microsoft Entra ID but the user can't access any Anyscale resources. See [Azure built-in roles for Anyscale](#azure-built-in-roles-for-anyscale) for the full role list and how to assign one.

## Managed identities for Azure resource access

When you create an Anyscale cloud resource through the Azure portal, the portal creates two managed identities in your resource group. These identities govern how the Anyscale operator and cluster workloads access Azure resources, including storage and container registry. They're separate from user authentication and role assignment.

### Anyscale operator managed identity

This identity governs all actions the operator takes in your Azure subscription, including provisioning nodes for Ray clusters. The Azure portal configures the identity automatically when you [create the cloud resource](quickstart-azure-cli-gateway-envoy.md#create-an-anyscale-cloud-resource).

### Cluster managed identity

This identity is the default that Anyscale uses when deploying clusters. By default, all workloads share it. You can create other identities and map them to specific users, projects, or workload types for finer-grained access control.

For environments where different teams or workload types need separate permissions, create other user-assigned managed identities and map them to Anyscale workloads.

Configure the mapping in the Anyscale console under **Cloud settings > IAM**. For AKS-specific configuration, see [Managed identities for AKS](https://docs.anyscale.com/clouds/azure/aks-iam) in the Anyscale documentation. For mapping rule syntax, see [Cloud IAM mapping](https://docs.anyscale.com/iam/cloud-iam-mapping).

## Service principal for cloud registration

You create a service principal in your tenant from the Anyscale Entra application (app ID `086bc555-6989-4362-ba30-fded273e432b`). Anyscale uses this service principal to sign Microsoft Entra ID tokens that validate the connection between your AKS deployment and the Anyscale control plane. You only need to do this once per tenant.

## Azure role requirements for setup

The person running the [Quickstart](quickstart-azure-cli-gateway-envoy.md) must have the following permissions on the target subscription:

| Permission | Required for |
|-----------|--------------|
| Subscription Owner, or Contributor plus User Access Administrator | AKS cluster creation and cloud resource setup. |
| Permission to create service principals from external tenants | Creating the Anyscale service principal. Required once per tenant. |

After setup, day-to-day Anyscale operations such as launching workloads and managing jobs require only the Anyscale RBAC roles assigned in the Azure portal. They don't require elevated Azure permissions.

## Azure built-in roles for Anyscale

Anyscale on Azure provides three built-in Azure roles managed in the Azure portal or through the Azure CLI. Role assignments determine what users, groups, and service principals can do across the console, CLI, SDK, and API. You can assign a role at any level of the Azure scope hierarchy, including subscription, resource group, cloud, project, or individual resource. All resources below that scope inherit the assignment.

| Role | Description |
|------|-------------|
| *Anyscale Platform Administrator* | Full access to all Anyscale resources within the assigned scope, including infrastructure management and workload execution. Includes the `Anyscale.Platform/admin/action` data action for administrative operations such as managing resource quotas and usage budgets. Currently only effective at subscription scope. |
| *Anyscale Platform Contributor* | Read and write access to Anyscale clouds, projects, workspaces, jobs, services, compute configs, and images. Doesn't include administrative data actions. |
| *Anyscale Platform Reader* | Read-only access to all Anyscale resources. Required for console sign-in. |

To assign a role, navigate to the Anyscale cloud resource in the Azure portal, select **Access control (IAM)** from the left menu, and select **Add** > **Add role assignment**. For detailed steps, see [Assign Azure roles using the Azure portal](/azure/role-based-access-control/role-assignments-portal).

## Custom role permissions reference

You can create custom Azure RBAC roles to grant a subset of Anyscale permissions. The following actions are available on the `Anyscale.Platform` resource provider:

| Resource type | Operations | Resource |
|---------------|------------|----------|
| `Anyscale.Platform/clouds` | read, write, delete | Clouds |
| `Anyscale.Platform/clouds/cloudResources` | read, write, delete | Cloud resources |
| `Anyscale.Platform/clouds/computeConfigs` | read, write, delete | Compute configs |
| `Anyscale.Platform/clouds/images` | read, write, delete | Images |
| `Anyscale.Platform/clouds/projects` | read, write, delete | Projects |
| `Anyscale.Platform/clouds/projects/jobs` | read, write, delete | Jobs |
| `Anyscale.Platform/clouds/projects/services` | read, write, delete | Services |
| `Anyscale.Platform/clouds/projects/workspaces` | read, write, delete | Workspaces |
| `Anyscale.Platform/admin` | action | Admin operations |

To construct a full action string, append the operation to the resource type with a slash, for example, `Anyscale.Platform/clouds/read`. For instructions on creating a custom role, see [Create or update Azure custom roles](/azure/role-based-access-control/custom-roles) in the Azure documentation.

## Next steps

> [!div class="nextstepaction"]
> [Quickstart: Create an Anyscale cloud](quickstart-azure-cli-gateway-envoy.md)

## Related content

- [Architecture overview](architecture.md) for how managed identities fit into the overall architecture.
- [Networking](networking.md) for egress domains and network security considerations.
- [Managed identities for AKS](https://docs.anyscale.com/clouds/azure/aks-iam) for per-workload identity mapping.
- [Cloud IAM mapping](https://docs.anyscale.com/iam/cloud-iam-mapping) for rule syntax and supported parameters.
- [Access blob storage and ADLS](https://docs.anyscale.com/clouds/azure/storage) for granting storage roles to managed identities.
