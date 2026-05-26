---
title: Configure container image builds for an existing cloud
description: Configure Azure Container Registry and the required Role-Based Access Control (RBAC) role assignments on an existing Anyscale cloud in Azure to enable container image builds.
author: kaysieyu
ms.author: kaysieyu
ms.date: 05/21/2026
ms.service: azure-kubernetes-service
ms.topic: how-to
---

# Configure container image builds for an existing cloud

By default, the Azure portal configures container image build support using an Azure Container Registry (ACR) when you create an Anyscale cloud. For setup instructions, see the [Quickstart](quickstart-azure-cli-gateway-envoy.md). This configuration is optional. You can skip it at creation time. If you created your cloud without ACR, you can enable it manually. Manual enablement requires the following:
- an ACR
- a cloud record updated with the ACR resource ID
- three Role-Based Access Control (RBAC) role assignments on the ACR
- the Anyscale operator on version 1.5.1 or greater. 

The following steps walk through each requirement.

If you attempt an image build on a cloud without ACR configured, the operation fails with this error:

```plaintext
Error: Cloud does not have ACR configuration. Please configure ACR for the cloud before creating a cluster environment.
```

## Prerequisites

- An existing Anyscale cloud on Azure
- Azure CLI installed and authenticated (`az login`)
- Permissions to create role assignments on the ACR resource (Owner or User Access Administrator on the ACR scope)
- The following values for your cloud:
  - Subscription ID
  - Resource group name
  - Azure Kubernetes Service (AKS) cluster name
  - Anyscale operator workload identity name (commonly `<cloud-name>-anyscale-operator-identity`)

## (Optional) Create an ACR

Skip this step if you already have an ACR you want to use.

Anyscale recommends provisioning the ACR in the same subscription, resource group, and region as your Anyscale cloud. This mirrors what the portal does for new clouds, keeps role-assignment scopes within one subscription boundary, and avoids cross-region data transfer costs. Cross-subscription ACRs work but require role-assignment permission in the ACR subscription.

```azurecli
az acr create \
  --subscription <subscription-id> \
  --resource-group <resource-group> \
  --name <acr-name> \
  --sku Standard \
  --location <location>
```

Capture the full resource ID for the following steps:

```azurecli
ACR_RESOURCE_ID=$(az acr show \
  --subscription <subscription-id> \
  --name <acr-name> \
  --query id \
  --output tsv)
```

## Update the cloud with the ACR resource ID

Call the Anyscale RP API to set `properties.acrResourceId` on your cloud resource. Use API version `2026-02-01-preview` or newer.

```azurecli
az rest \
  --method PATCH \
  --url "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Anyscale.Platform/clouds/<cloud-name>?api-version=2026-02-01-preview" \
  --body '{
    "location": "<location>",
    "properties": {
      "acrResourceId": "<acr-resource-id>"
    }
  }'
```

For example:

```azurecli
az rest \
  --method PATCH \
  --url "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/my-cloud-rg/providers/Anyscale.Platform/clouds/mycloud?api-version=2026-02-01-preview" \
  --body '{
    "location": "westus2",
    "properties": {
      "acrResourceId": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/my-cloud-rg/providers/Microsoft.ContainerRegistry/registries/mycloudacr"
    }
  }'
```

## Grant RBAC on the ACR

These role assignments mirror exactly what the portal creates for new clouds. First retrieve the principal IDs of the two identities that need access.

### Find the AKS kubelet identity

The kubelet identity is a managed identity assigned to AKS agent nodes for pulling images. For more information, see [Managed identities in AKS](/azure/aks/managed-identity-overview).

```azurecli
KUBELET_PRINCIPAL_ID=$(az aks show \
  --subscription <subscription-id> \
  --resource-group <resource-group> \
  --name <aks-cluster-name> \
  --query "identityProfile.kubeletidentity.objectId" \
  --output tsv)
```

### Find the Anyscale operator workload identity

The operator uses a user-assigned managed identity federated to the `anyscale-operator/anyscale-operator` service account. Its name is the value passed as `workloadIdentityName` during the original Azure Resource Manager deployment, commonly `<cloud-name>-anyscale-operator-identity`.

```azurecli
OPERATOR_PRINCIPAL_ID=$(az identity show \
  --subscription <subscription-id> \
  --resource-group <resource-group> \
  --name <operator-workload-identity-name> \
  --query "principalId" \
  --output tsv)
```

### Assign the roles

Assign `AcrPull` to the AKS kubelet identity so worker nodes can pull images that the build manager pushed.

```azurecli
az role assignment create \
  --assignee $KUBELET_PRINCIPAL_ID \
  --role AcrPull \
  --scope $ACR_RESOURCE_ID
```

Assign `AcrPush` to the Anyscale operator workload identity so the build manager can push built images into the registry.

```azurecli
az role assignment create \
  --assignee $OPERATOR_PRINCIPAL_ID \
  --role AcrPush \
  --scope $ACR_RESOURCE_ID
```

Assign `Container Registry Tasks Contributor` to the Anyscale operator workload identity so the operator can create and run ACR Tasks to execute image builds.

```azurecli
az role assignment create \
  --assignee $OPERATOR_PRINCIPAL_ID \
  --role "Container Registry Tasks Contributor" \
  --scope $ACR_RESOURCE_ID
```

## Upgrade and restart the Anyscale operator

The operator registers container image build support during startup. It must be on version **≥ 1.5.1** and must start after the cloud record has an `acrResourceId`.

If the operator is already on ≥ 1.5.1, a rollout restart is sufficient:

```bash
kubectl -n anyscale-operator rollout restart deployment/anyscale-operator
```

Otherwise, upgrade it using the Azure CLI:

```azurecli
az k8s-extension update \
  --cluster-name <aks-cluster-name> \
  --subscription <subscription-id> \
  --resource-group <resource-group> \
  --cluster-type managedClusters \
  --name <extension-name> \
  --version 1.5.1
```

> [!NOTE]
> If you onboarded previously to Anyscale on Azure and installed the operator through Helm rather than the AKS cluster extension, upgrade using `helm upgrade` instead:
>
> ```bash
> helm upgrade anyscale-operator <chart> \
>   --namespace anyscale-operator \
>   --version 1.5.1 \
>   -f <values-file>
> ```

## Next steps

- [Identity and access](identity-access.md) for managed identities and role assignments in Anyscale on Azure.
- [Architecture overview](architecture.md) for how the data plane components fit together.
- [Quickstart](quickstart-azure-cli-gateway-envoy.md) to deploy your first Anyscale cloud on Azure.
