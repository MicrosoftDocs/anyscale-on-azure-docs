---
title: "Quickstart: Deploy Anyscale on Azure | Microsoft Learn"
description: Deploy your first Anyscale cloud on Azure Kubernetes Service — configure your subscription, provision infrastructure, register via the Azure portal, and install the ingress controller.
author: kaysieyu
ms.author: kaysieyu
ms.date: 04/03/2026
ms.topic: quickstart
---

# Quickstart: Deploy Anyscale on Azure

This quickstart walks you through deploying Anyscale on an existing Azure Kubernetes Service (AKS) cluster. By the end, you'll have a registered Anyscale cloud and be ready to run Ray workloads.

## Prerequisites

Before you begin, make sure you have:

- An Azure subscription with owner or administrator role
- Permission to create service principals from external Entra tenants
- An existing AKS cluster with [OIDC issuer enabled](/azure/aks/use-oidc-issuer)
- The following tools installed locally:
  - [Azure CLI](/cli/azure/install-azure-cli)
  - [kubectl](https://kubernetes.io/docs/tasks/tools/)
  - [Helm](https://helm.sh/docs/intro/install/)
  - [Terraform](https://developer.hashicorp.com/terraform/install)
  - [Git](https://git-scm.com/downloads)
  - [Anyscale CLI](https://docs.anyscale.com/reference/anyscale-cli): `pip install anyscale`

You must be enrolled in the Anyscale on Azure Public Preview. Contact [Anyscale support](https://docs.anyscale.com/support) to enroll and provide your Azure subscription ID and preferred deployment regions.

## Step 0: Configure your Azure subscription

### 0a: Create the Anyscale service principal

Run the following command to establish trust with Anyscale's control plane:

```azurecli
az ad sp create --id 086bc555-6989-4362-ba30-fded273e432b
```

### 0b: Register required resource providers

Check which providers are already registered:

```azurecli
az provider list --query "[?registrationState=='Registered']" --output table
```

Register any of the following that aren't listed:

```azurecli
for provider in Microsoft.Storage Microsoft.ManagedIdentity Microsoft.Authorization \
  Microsoft.Resources Microsoft.Network Microsoft.ContainerService; do
  az provider register --namespace "$provider"
done
```

## Step 1: Provision Azure resources

Use the Anyscale Terraform module in the [Azure Samples repository](https://github.com/Azure-Samples/anyscale-on-azure) to provision the AKS cluster and supporting Azure resources (storage account, managed identities, and role assignments).

The module README walks through authentication, variable configuration, and `terraform apply`. If you don't already have an AKS cluster with OIDC issuer enabled, see [Create an AKS cluster](/azure/aks/learn/quick-kubernetes-deploy-cli) and [Enable OIDC issuer](/azure/aks/use-oidc-issuer) in the Azure documentation.

After `terraform apply` completes, save the following values from the module output:

| Output | Used in |
|--------|---------|
| Operator IAM Principal ID | Step 3 |
| Operator IAM Client ID | Step 3 |
| Container name | Step 2 |
| Storage account name | Step 2 |
| Azure resource group name | Steps 2 and 4 |

## Step 2: Create an Anyscale cloud resource

### 2a: Navigate to the Public Preview portal

Go to `https://aka.ms/<anyscale-public-preview-alias>` in the Azure portal.

<!-- TODO: confirm the aka.ms short link for Public Preview with Elizabeth/Anyscale before publication -->

:::image type="content" source="media/quickstart/quickstart-clouds-landing.png" alt-text="Anyscale clouds page in the Azure portal showing a list of existing Anyscale cloud resources.":::

### 2b: Fill in the Basics tab

Select **Create** on the Anyscale clouds page.

Fill in the following fields:

1. Confirm the **Subscription** matches your Terraform deployment.
2. Select the **Resource group** from the Terraform output.
3. Enter a unique **Cloud name** (alphanumeric characters only).
4. Select the same **Region** you used in your Terraform configuration.

:::image type="content" source="media/quickstart/quickstart-create-basics-filled.png" alt-text="Basics tab with subscription, resource group, cloud name, and region filled in.":::

Select **Next**.

### 2c: Fill in the Cloud storage settings tab

:::image type="content" source="media/quickstart/quickstart-create-cloud-storage-settings.png" alt-text="Cloud storage settings tab with storage account name and AKS cluster fields filled in.":::

1. Enter the **Storage account name** from the Terraform output.
2. Select your **AKS cluster** from the dropdown.

Select **Next**.

### 2d: Add tags (optional)

:::image type="content" source="media/quickstart/quickstart-create-tags.png" alt-text="Tags tab showing name and value fields for adding resource tags for billing and cost management.":::

Add name/value tag pairs to categorize the created resources for billing and cost management. Tags are optional.

Select **Review + create**.

### 2e: Review and create

:::image type="content" source="media/quickstart/quickstart-review-create-summary.png" alt-text="Review + create tab showing a summary of the Basics and Cloud storage settings configuration.":::

Review your configuration, then select **Create**.

The portal creates the required storage, managed identity, and service account, and installs the Anyscale Kubernetes operator automatically.

## Step 3: Update your Anyscale cloud configuration

### 3a: Sign in to Anyscale

```bash
export ANYSCALE_HOST=https://console.azure.anyscale.com
anyscale login
```

Add `ANYSCALE_HOST` to your shell configuration (`.bashrc` or `.zshrc`) so it persists across sessions.

### 3b: Find your Anyscale cloud ID

```bash
anyscale cloud list
```

### 3c: Download and edit your cloud configuration

```bash
anyscale cloud get --id <your-cloud-id> > cloud-config.yaml
```

Open `cloud-config.yaml` and add the operator identity from your Terraform output:

```yaml
kubernetes_config:
  anyscale_operator_iam_identity: <anyscale-operator-principal-id>
```

Apply the update:

```bash
anyscale cloud update --id <your-cloud-id> -f cloud-config.yaml
```

## Step 4: Install the Nginx ingress controller

### 4a: Get AKS credentials

```bash
az aks get-credentials \
  --resource-group <azure-resource-group-name> \
  --name <your-aks-cluster-name> \
  --overwrite-existing
```

### 4b: Install nginx-ingress

```bash
helm repo add nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm upgrade ingress-nginx nginx/ingress-nginx \
  --version 4.12.1 \
  --namespace ingress-nginx \
  --values sample-values_nginx.yaml \
  --create-namespace \
  --install
```


## Verify the deployment

In the Azure portal, navigate to your Anyscale cloud resource and confirm the **Status** shows **Succeeded**.

You can also verify from the Anyscale CLI:

```bash
anyscale cloud verify --name <cloud-name>
```

A healthy cloud returns output similar to:

```plaintext
Cloud <cloud-name> is healthy.
```

<!-- TODO: confirm exact CLI command and expected output with Anyscale engineering before publication -->

## Run your first workload

1. Sign in to [console.azure.anyscale.com](https://console.azure.anyscale.com).

1. Select your cloud from the cloud picker.

1. Select **Workspaces** > **New workspace**.

1. When the workspace is running, open a terminal and run the following code to confirm Ray is connected to your cluster:

   ```python
   import ray
   ray.init()
   print(ray.cluster_resources())
   ```

   The output lists the CPU and memory resources available in your cluster, for example:

   ```plaintext
   {'CPU': 12.0, 'memory': 50000000000.0, 'node:__internal_head__': 1.0, ...}
   ```

<!-- TODO: confirm workspace launch steps and expected output with Anyscale engineering before publication -->

## Next steps

> [!div class="nextstepaction"]
> [Architecture overview](architecture.md)

- [Networking](networking.md)
- [Identity and access](identity-access.md)
- [Supported regions](supported-regions.md)
