---
title: "Quickstart: Deploy Anyscale on Azure | Microsoft Learn"
description: Deploy your first Anyscale cloud on Azure Kubernetes Service — configure your subscription, provision infrastructure with the Azure CLI, register via the Azure portal, and install the ingress controller.
author: kaysieyu
ms.author: kaysieyu
ms.date: 04/15/2026
ms.topic: quickstart
---

# Quickstart: Deploy Anyscale on Azure

This quickstart walks you through deploying Anyscale on an existing Azure Kubernetes Service (AKS) cluster. By the end, you'll have a registered Anyscale cloud and be ready to run Ray workloads.

## Prerequisites

Before you begin, make sure you have:

- An Azure subscription with owner or administrator role
- Permission to create service principals from external Entra tenants
- The following tools installed locally:
  - [Azure CLI](/cli/azure/install-azure-cli)
  - [kubectl](https://kubernetes.io/docs/tasks/tools/)
  - [Helm](https://helm.sh/docs/intro/install/)
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

### 1a: Create or select a resource group

You can use an existing resource group or create a new one in one of the [supported regions](supported-regions.md):

```azurecli
az group create \
  --name <resource-group> \
  --location <location>
```

### 1b: Create the AKS cluster

Before creating the cluster, confirm you have sufficient quota for the VM SKU you plan to use in your chosen region. Ray workloads require at least 4 vCPUs per worker node — `Standard_D4s_v5` or equivalent is a good starting point. Check your current quota:

```azurecli
az vm list-usage --location <location> --query "[?contains(name.value, 'standardDSv5Family')]" -o table
```

If you need a quota increase, request one in the [Azure portal](https://portal.azure.com/#view/Microsoft_Azure_Capacity/QuotaMenuBlade).

Create the cluster with OIDC issuer and workload identity enabled:

```azurecli
az aks create \
  --resource-group <resource-group> \
  --name <cluster-name> \
  --location <location> \
  --node-count 3 \
  --node-vm-size Standard_D4s_v5 \
  --enable-oidc-issuer \
  --enable-workload-identity \
  --generate-ssh-keys
```


After the cluster is created, save the resource group name and cluster name — you'll need them in Steps 2 and 4.

## Step 2: Create an Anyscale cloud resource

### 2a: Navigate to the Public Preview portal

Go to `https://aka.ms/<anyscale-public-preview-alias>` in the Azure portal.


:::image type="content" source="media/quickstart/quickstart-clouds-landing.png" alt-text="Anyscale clouds page in the Azure portal showing a list of existing Anyscale cloud resources.":::

### 2b: Fill in the Basics tab

Select **Create** on the Anyscale clouds page.

Fill in the following fields:

1. Confirm the **Subscription** matches the subscription you used in Step 1.
2. Select the **Resource group** you created or used in Step 1.
3. Enter a unique **Cloud name** (alphanumeric characters only).
4. Select the same **Region** you used in Step 1.

:::image type="content" source="media/quickstart/quickstart-create-basics-filled.png" alt-text="Basics tab with subscription, resource group, cloud name, and region filled in.":::

Select **Next**.

### 2c: Fill in the Cloud storage settings tab

:::image type="content" source="media/quickstart/quickstart-create-cloud-storage-settings.png" alt-text="Cloud storage settings tab with storage account name and AKS cluster fields filled in.":::

1. Enter a unique **Storage account name** (3–24 lowercase alphanumeric characters).
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

Open `cloud-config.yaml` and add the operator identity from your Anyscale cloud resource:

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

Create a file named `sample-values_nginx.yaml`:

```yaml
controller:
  progressDeadlineSeconds: 600
  service:
    type: LoadBalancer
    annotations:
      service.beta.kubernetes.io/azure-load-balancer-health-probe-request-path: "/healthz"
  allowSnippetAnnotations: true
  config:
    enable-underscores-in-headers: true
    annotations-risk-level: "Critical"
  autoscaling:
    enabled: true
```

Then install the controller:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm upgrade ingress-nginx ingress-nginx/ingress-nginx \
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


## Run your first workload

1. Create a file named `main.py`:

   ```python
   import ray
   import time

   num_ray_tasks = 5

   @ray.remote
   def process(x):
       if x == (num_ray_tasks - 1):
           print("Hello from one of the Running Ray Tasks!")
           time.sleep(200)
       return x * 2

   result = ray.get([process.remote(x) for x in range(num_ray_tasks)])
   print("The job result is", result)
   ```

1. Create a file named `job.yaml` in the same directory:

   ```yaml
   name: my-first-job
   working_dir: .
   entrypoint: python main.py
   max_retries: 1
   ```

1. Submit the job:

   ```bash
   anyscale job submit -f job.yaml
   ```

   When the job completes, the output includes:

   ```plaintext
   Hello from one of the Running Ray Tasks!
   The job result is [0, 2, 4, 6, 8]
   ```

   You can also view job status and logs in the [Anyscale console](https://console.azure.anyscale.com).


## Next steps

> [!div class="nextstepaction"]
> [Architecture overview](architecture.md)

- [Networking](networking.md)
- [Identity and access](identity-access.md)
- [Supported regions](supported-regions.md)
