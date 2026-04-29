---
title: "Quickstart: Deploy Anyscale with Ingress-Nginx | Microsoft Learn"
description: Deploy your first Anyscale cloud on Azure Kubernetes Service using the Azure CLI and the Nginx ingress controller. Configure your subscription, create an AKS cluster, and register via the Azure portal.
author: kaysieyu
ms.author: kaysieyu
ms.date: 04/15/2026
ms.service: azure-kubernetes-service
ms.topic: quickstart
---

# Quickstart: Deploy Anyscale with Ingress-Nginx

> [!div class="op_multi_selector" title1="Quickstart" title2="Ingress"]
> - [Envoy Gateway](quickstart-azure-cli-gateway-envoy.md)
> - [Ingress-Nginx](quickstart-azure-cli-ingress-nginx.md)

This quickstart walks you through deploying Anyscale on an existing Azure Kubernetes Service (AKS) cluster. By the end, you'll have a registered Anyscale cloud and be ready to run Ray workloads.

## Prerequisites and required tools

Before you begin, make sure you have:

- An Azure subscription with owner or administrator role
- Permission to create service principals from external Entra tenants
- The following tools installed locally:
  - [Azure CLI](/cli/azure/install-azure-cli)
  - [kubectl](https://kubernetes.io/docs/tasks/tools/)
  - [Helm](https://helm.sh/docs/intro/install/)
  - [Anyscale CLI](https://docs.anyscale.com/reference/quickstart-cli): `pip install anyscale`

You must be enrolled in the Anyscale on Azure Public Preview. Contact [Anyscale support](https://www.anyscale.com/support) to enroll and provide your Azure subscription ID and preferred deployment regions.

## Step 0: configure your Azure subscription

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

## Step 1: provision Azure resources

### 1a: Create or select a resource group

You can use an existing resource group or create a new one in one of the [supported regions](supported-regions.md):

```azurecli
az group create \
  --name <resource-group> \
  --location <location>
```

### 1b: Create the AKS cluster

Before creating the cluster, confirm you have sufficient quota for the VM SKU you plan to use in your chosen region. Ray workloads require at least 4 vCPUs per worker node—`Standard_D4s_v5` or equivalent is a good starting point. Check your current quota:

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


After the cluster is created, save the resource group name and cluster name—you'll need them in Steps 2 and 3.

### Node pools and workload placement (optional)

By default, Ray head nodes and worker nodes share the default node pool. For production workloads, you can create dedicated AKS node pools for Ray workloads and use Kubernetes taints and tolerations to steer Ray pods to those nodes. Apply a `NoSchedule` taint to the dedicated node pool to prevent non-Ray workloads from scheduling on it, then configure matching tolerations in your Anyscale cluster configuration so Ray pods are admitted to the tainted pool. This keeps Ray workers isolated from operator and system pods on the default node pool.

## Step 2: create an Anyscale cloud resource

### 2a: Navigate to the Anyscale clouds page

In the [Azure portal](https://portal.azure.com), search for **Anyscale** in the global search bar and select **Anyscale** from the results.

:::image type="content" source="media/quickstart/quickstart-anyscale-clouds-landing.png" alt-text="Anyscale clouds page in the Azure portal showing a list of existing Anyscale cloud resources.":::

### 2b: Fill in the Basics tab

Select **Create** on the Anyscale clouds page.

Fill in the following fields:

1. Confirm the **Subscription** matches the subscription you used in Step 1.
2. Select the **Resource group** you created or used in Step 1.
3. Enter a unique **Cloud name**.
4. Select the same **Region** you used in Step 1.
5. Select your **Cluster** from the dropdown.

Leave **Support tier** at the default value.

:::image type="content" source="media/quickstart/quickstart-create-basics-filled.png" alt-text="Basics tab with subscription, resource group, cloud name, region, and AKS cluster filled in.":::

Select **Next**.

### 2c: Configure Infrastructure settings

The portal pre-populates a storage account name and operator identity name. Accept the defaults or enter custom names:

- **Storage account name**: 3–24 lowercase alphanumeric characters.
- **Operator identity name**: the managed identity used by the Anyscale operator.

:::image type="content" source="media/quickstart/quickstart-create-infrastructure-settings.png" alt-text="Infrastructure settings tab showing storage account and managed identity configuration with default values pre-populated.":::

Select **Next**.

### 2d: Configure Container registry settings

The portal pre-populates an Azure Container Registry (ACR) name. ACR is used for container image builds. Accept the default or enter a custom name.

:::image type="content" source="media/quickstart/quickstart-create-container-registry-settings.png" alt-text="Container registry settings tab with ACR mode set to Create new ACR and an auto-generated ACR name.":::

Select **Next**.

### 2e: Review Operator settings

The Operator settings tab shows pre-populated values for the Anyscale operator. Leave these at their defaults.

:::image type="content" source="media/quickstart/quickstart-create-operator-settings.png" alt-text="Operator settings tab showing pre-populated extension resource name, control plane URL, and authentication audience.":::

Select **Next**.

### 2f: Add tags (optional)

Add name/value tag pairs to categorize the created resources for billing and cost management. Tags are optional.

:::image type="content" source="media/quickstart/quickstart-create-tags.png" alt-text="Tags tab showing name and value fields for adding resource tags for billing and cost management.":::

Select **Next**.

### 2g: Review and submit

:::image type="content" source="media/quickstart/quickstart-review-submit-summary.png" alt-text="Review + submit tab showing a summary of the cloud configuration across all tabs.":::

Azure validates your configuration before enabling the **Create** button. Once validation passes, select **Create**.

The portal creates the required storage, managed identity, container registry, and service account, and installs the Anyscale Kubernetes operator automatically. Wait for the deployment to complete before proceeding to Step 3.

## Step 3: install the Nginx ingress controller

### 3a: Get AKS credentials

```bash
az aks get-credentials \
  --resource-group <azure-resource-group-name> \
  --name <your-aks-cluster-name> \
  --overwrite-existing
```

Confirm the Anyscale operator is running:

```bash
kubectl get pods -n anyscale-operator
```

The operator pod should show a status of `Running`.

### 3b: Install nginx-ingress

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

Verify the deployment from the Anyscale CLI. First, set the Anyscale console URL and sign in:

```bash
export ANYSCALE_HOST=https://console.azure.anyscale.com
anyscale login
```

To avoid setting `ANYSCALE_HOST` each session, add the following to your shell configuration file (for example, `.bashrc` or `.zshrc`) and start a new shell:

```bash
export ANYSCALE_HOST=https://console.azure.anyscale.com
```


Make sure your `kubectl` context is set to the correct cluster:

```bash
kubectl config use-context <cluster-name>
```

Find your cloud ID from the Anyscale console or by running:

```bash
anyscale cloud list
```

The cloud ID has the format `cld_*`. Then run:

```bash
anyscale cloud verify --id <cloud-id>
```

The CLI prompts you to select your `kubectl` context and confirm the operator namespace. After confirming, a healthy cloud returns output similar to:

```plaintext
Overall Result: ALL 1 cloud resources verified successfully
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

1. Submit the job, using the cloud name as it appears in the Anyscale console or `anyscale cloud list` (it starts with `/subscriptions/`):

   ```bash
   anyscale job submit -f job.yaml --cloud <cloud-name>
   ```

   The command returns a URL to track job status and view output in the Anyscale console.


## Next steps

> [!div class="nextstepaction"]
> [Architecture overview](architecture.md)

- [Networking](networking.md)
- [Identity and access](identity-access.md)
- [Supported regions](supported-regions.md)
