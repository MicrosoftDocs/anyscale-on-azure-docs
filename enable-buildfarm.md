---
title: Enable container image build on an existing Anyscale cloud | Microsoft Learn
description: Configure Azure Container Registry and the required RBAC role assignments on an existing Anyscale cloud in Azure to enable container image builds.
author: kaysieyu
ms.author: kaysieyu
ms.date: 04/29/2026
ms.service: azure-kubernetes-service
ms.topic: how-to
---

# Enable container image build on an existing cloud

When you create an Anyscale cloud through the Azure portal, container image build support is configured by default using an Azure Container Registry (ACR). This configuration is optional and can be skipped at creation time. If your cloud was created without ACR configured, you can enable it manually using the steps in this article.

If you attempt an image build on a cloud without ACR configured, the operation fails with this error:

```
Error: Cloud does not have ACR configuration. Please configure ACR for the cloud before creating a cluster environment.
```

## Prerequisites

- An existing Anyscale cloud on Azure
- Azure CLI installed and authenticated (`az login`)
- Permissions to create role assignments on the ACR resource (Owner or User Access Administrator on the ACR scope)
- The following values for your cloud:
  - Subscription ID
  - Resource group name
  - AKS cluster name
  - Anyscale operator workload identity name (commonly `<cloudName>-anyscale-operator-identity`)

## What needs to be configured

Enabling container image build requires four things, each covered by a step in this article:

1. An ACR that the cloud can use—either an existing one or a newly created one
2. The Anyscale cloud resource updated with `properties.acrResourceId` pointing at that ACR
3. Three RBAC role assignments scoped to the ACR:
   - `AcrPull` for the **AKS kubelet identity** — so nodes can pull built images
   - `AcrPush` for the **Anyscale operator workload identity** — so the build manager can push images
   - `Container Registry Tasks Contributor` for the **Anyscale operator workload identity** — so the build manager can drive ACR Tasks
4. The Anyscale operator running version **≥ 1.5.1** and restarted after the cloud record is updated

## Step 1 - (Optional) Create an ACR

Skip this step if you already have an ACR you want to use.

> [!TIP]
> Provision the ACR in the **same subscription and resource group as your Anyscale cloud**. This mirrors what the portal does for new clouds, keeps role-assignment scopes within one subscription boundary, and makes lifecycle management consistent with the rest of the cloud's resources. Cross-subscription ACRs work but require role-assignment permission in the ACR's subscription.

```bash
az acr create \
  --subscription <SUBSCRIPTION_ID> \
  --resource-group <RESOURCE_GROUP> \
  --name <ACR_NAME> \
  --sku Standard \
  --location <LOCATION>
```

Capture the full resource ID for the following steps:

```bash
ACR_RESOURCE_ID=$(az acr show \
  --subscription <SUBSCRIPTION_ID> \
  --name <ACR_NAME> \
  --query id \
  --output tsv)
```

## Step 2 - Update the cloud with the ACR resource ID

Call the Anyscale RP API to set `properties.acrResourceId` on your cloud resource. Use API version `2026-02-01-preview` or newer.

```bash
az rest \
  --method PUT \
  --url "/subscriptions/<SUBSCRIPTION_ID>/resourceGroups/<RESOURCE_GROUP>/providers/Anyscale.Platform/clouds/<CLOUD_NAME>?api-version=2026-02-01-preview" \
  --body '{
    "location": "<LOCATION>",
    "properties": {
      "acrResourceId": "<ACR_RESOURCE_ID>"
    }
  }'
```

**Example:**

```bash
az rest \
  --method PUT \
  --url "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/my-cloud-rg/providers/Anyscale.Platform/clouds/mycloud?api-version=2026-02-01-preview" \
  --body '{"location": "westus2", "properties": {"acrResourceId": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/my-cloud-rg/providers/Microsoft.ContainerRegistry/registries/mycloudacr"}}'
```

## Step 3 - Grant RBAC on the ACR

These role assignments mirror exactly what the portal creates for new clouds. First retrieve the principal IDs of the two identities that need access.

### Find the AKS kubelet identity

The kubelet identity is a managed identity assigned to AKS agent nodes that allows them to pull images. For more information, see [Managed identities in AKS](https://learn.microsoft.com/en-us/azure/aks/managed-identity-overview).

```bash
KUBELET_PRINCIPAL_ID=$(az aks show \
  --subscription <SUBSCRIPTION_ID> \
  --resource-group <RESOURCE_GROUP> \
  --name <AKS_CLUSTER_NAME> \
  --query "identityProfile.kubeletidentity.objectId" \
  --output tsv)
```

### Find the Anyscale operator workload identity

The operator uses a user-assigned managed identity federated to the `anyscale-operator/anyscale-operator` service account. Its name is the value passed as `workloadIdentityName` during the original ARM deployment, commonly `<cloudName>-anyscale-operator-identity`.

```bash
OPERATOR_PRINCIPAL_ID=$(az identity show \
  --subscription <SUBSCRIPTION_ID> \
  --resource-group <RESOURCE_GROUP> \
  --name <OPERATOR_WORKLOAD_IDENTITY_NAME> \
  --query "principalId" \
  --output tsv)
```

### Assign the roles

**AcrPull for the AKS kubelet identity** — allows worker nodes to pull images that the build manager has pushed:

```bash
az role assignment create \
  --assignee $KUBELET_PRINCIPAL_ID \
  --role AcrPull \
  --scope $ACR_RESOURCE_ID
```

**AcrPush for the Anyscale operator workload identity** — allows the build manager to push built images into the registry:

```bash
az role assignment create \
  --assignee $OPERATOR_PRINCIPAL_ID \
  --role AcrPush \
  --scope $ACR_RESOURCE_ID
```

**Container Registry Tasks Contributor for the Anyscale operator workload identity** — allows the operator to create and run ACR Tasks, which is how image builds are executed:

```bash
az role assignment create \
  --assignee $OPERATOR_PRINCIPAL_ID \
  --role "Container Registry Tasks Contributor" \
  --scope $ACR_RESOURCE_ID
```

## Step 4 - Upgrade and restart the Anyscale operator

The operator registers Buildfarm support during startup. It must be on version **≥ 1.5.1** and must start *after* the cloud record has an `acrResourceId`.

**If the operator was installed via the AKS cluster extension**, upgrade it:

```bash
az k8s-extension update \
  --cluster-name <AKS_CLUSTER_NAME> \
  --subscription <SUBSCRIPTION_ID> \
  --resource-group <RESOURCE_GROUP> \
  --cluster-type managedClusters \
  --name <EXTENSION_NAME> \
  --version 1.5.1
```

**If the operator was installed via Helm**, upgrade to at least `1.5.1`:

```bash
helm upgrade anyscale-operator <chart> \
  --namespace anyscale-operator \
  --version 1.5.1 \
  -f <your-values.yaml>
```

**If the operator is already on ≥ 1.5.1**, a rollout restart is sufficient:

```bash
kubectl -n anyscale-operator rollout restart deployment/anyscale-operator
```

## Next steps

- [Identity and access](identity-access.md) — managed identities and role assignments in Anyscale on Azure
- [Architecture overview](architecture.md) — how the data plane components fit together
- [Quickstart](quickstart-azure-cli-gateway-envoy.md) — deploy your first Anyscale cloud on Azure
