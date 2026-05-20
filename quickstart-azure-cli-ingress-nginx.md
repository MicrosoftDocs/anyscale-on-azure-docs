---
title: "Quickstart: Deploy Anyscale on Azure with Ingress-Nginx"
description: Deploy your first Anyscale cloud on Azure Kubernetes Service using the Azure CLI and the Ingress-Nginx controller. Configure your subscription, create an AKS cluster, and register through the Azure portal.
author: kaysieyu
ms.author: kaysieyu
ms.date: 05/19/2026
ms.service: azure-kubernetes-service
ms.topic: quickstart
---

# Quickstart: Deploy Anyscale on Azure with Ingress-Nginx

> [!div class="op_multi_selector" title1="Quickstart" title2="Ingress"]
> - [Envoy Gateway](quickstart-azure-cli-gateway-envoy.md)
> - [Ingress-Nginx](quickstart-azure-cli-ingress-nginx.md)

This quickstart walks you through deploying Anyscale on an existing Azure Kubernetes Service (AKS) cluster. By the end, you have a registered Anyscale cloud and are ready to run Ray workloads.

> [!CAUTION]
> Ingress-Nginx reached end of life in March 2026 and is no longer actively maintained. For new deployments that don't require an Ingress controller, use the [Envoy Gateway quickstart](quickstart-azure-cli-gateway-envoy.md) instead.

## Prerequisites and required tools

Before you begin, make sure you have:

- An Azure subscription with the Owner or Administrator role.
- Permission to create service principals from external Microsoft Entra tenants.
- Install the following tools locally. Use the latest version of each.
  - [Azure CLI](/cli/azure/install-azure-cli)
  - [kubectl](https://kubernetes.io/docs/tasks/tools/). Your version must be within one minor version of your AKS cluster. See the [Kubernetes version skew policy](https://kubernetes.io/releases/version-skew-policy/#kubectl).
  - [Helm](https://helm.sh/docs/intro/install/)
  - [Anyscale CLI](https://docs.anyscale.com/reference/quickstart-cli): `pip install anyscale`

Enroll in the Anyscale on Azure Public Preview before you start. Contact [Anyscale support](https://www.anyscale.com/support) to enroll, and provide your Azure subscription ID and preferred deployment regions.

## Step 0: configure your Azure subscription

Step 0a requires permission to create service principals from external Microsoft Entra tenants. Review the prerequisite above before you proceed.

### 0a: create the Anyscale service principal

To establish trust with the Anyscale control plane, run the following command:

```azurecli
az ad sp create --id 086bc555-6989-4362-ba30-fded273e432b
```

### 0b: register required resource providers

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

### 1a: create or select a resource group

You can use an existing resource group or create a new one in one of the [supported regions](supported-regions.md):

```azurecli
az group create \
  --name <resource-group> \
  --location <location>
```

### 1b: create the AKS cluster

Before you create the cluster, confirm you have sufficient quota for the VM SKU you plan to use in your chosen region. Ray workloads require at least 4 vCPUs per worker node. `Standard_D4s_v5` or equivalent is a good starting point. Check your current quota:

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

After Azure creates the cluster, save the resource group name and cluster name. You need them in Steps 2 and 3.

### Node pools and workload placement (optional)

By default, Ray head nodes and worker nodes share the default node pool. For production workloads, create dedicated AKS node pools for Ray workloads and use Kubernetes taints and tolerations to steer Ray pods to those nodes.

Apply a `NoSchedule` taint to the dedicated node pool to prevent non-Ray workloads from scheduling on it. Then configure matching tolerations in your Anyscale cluster configuration so Ray pods are admitted to the tainted pool. This setup keeps Ray workers isolated from operator and system pods on the default node pool.

For production deployments, pair dedicated node pools with [declarative compute configs](https://docs.anyscale.com/configuration/compute/) to define instance types, resource requirements, and workload placement in code. This is the preferred approach for Anyscale on Azure.

## Step 2: create an Anyscale cloud resource

> [!NOTE]
> The Anyscale Operator is also available through the Azure Marketplace, but Anyscale doesn't recommend that route. Use the Anyscale Clouds Resource Provider in the Azure portal instead.

### 2a: navigate to the Anyscale clouds page

In the [Azure portal](https://portal.azure.com), search for **Anyscale clouds** in the global search bar and select **Anyscale clouds** under **Services** from the results.

:::image type="content" source="media/quickstart/quickstart-anyscale-clouds-landing.png" alt-text="Anyscale clouds page in the Azure portal showing a list of existing Anyscale cloud resources.":::

### 2b: fill in the Basics tab

Select **Create** on the Anyscale clouds page.

Fill in the following fields:

1. Confirm the **Subscription** matches the subscription you used in Step 1.
1. Select the **Resource group** you created or used in Step 1.
1. Enter a unique **Cloud name**.
1. Select the same **Region** you used in Step 1.
1. Select your **Cluster** from the dropdown.

:::image type="content" source="media/quickstart/quickstart-create-basics-filled.png" alt-text="Basics tab with subscription, resource group, cloud name, region, and AKS cluster filled in.":::

Select **Next**.

### 2c: configure infrastructure settings

The portal pre-populates a storage account name and Anyscale operator identity name. Accept the defaults or enter custom names:

- **Storage account name**: 3 to 24 lowercase alphanumeric characters.
- **Anyscale operator identity name**: the user-assigned managed identity that the Anyscale operator uses.

:::image type="content" source="media/quickstart/quickstart-create-infrastructure-settings.png" alt-text="Infrastructure settings tab showing storage account and user-assigned managed identity configuration with default values pre-populated.":::

Select **Next**.

### 2d: configure container registry settings

The portal pre-populates an Azure Container Registry (ACR) name. Anyscale uses ACR for container image builds. Accept the default or enter a custom name.

:::image type="content" source="media/quickstart/quickstart-create-container-registry-settings.png" alt-text="Container registry settings tab with ACR mode set to Create new ACR and an auto-generated ACR name.":::

> [!NOTE]
> This ACR is used exclusively for Anyscale container image builds. To configure your cluster to pull Ray images from a different container registry, see [Configure a custom container image registry](https://docs.anyscale.com/container-image/image-registry) in the Anyscale documentation.

Select **Next**.

### 2e: review the support plan

The Support plan tab shows the support tier for your Anyscale cloud. This value is fixed and can't be changed. For details, see [Support model](support-model.md).

:::image type="content" source="media/quickstart/quickstart-create-support-plan.png" alt-text="Support plan tab showing the support tier set to Enterprise Tier (15% Consumption).":::

Select **Next**.

### 2f: add tags (optional)

Add name/value tag pairs to categorize the created resources for billing and cost management. Tags are optional.

:::image type="content" source="media/quickstart/quickstart-create-tags.png" alt-text="Tags tab showing name and value fields for adding resource tags for billing and cost management.":::

Select **Next**.

### 2g: review the terms and conditions, then create

:::image type="content" source="media/quickstart/quickstart-review-submit-summary.png" alt-text="Review + submit tab showing a summary of the cloud configuration, Marketplace terms, and contact fields.":::

The **Review + submit** tab includes a **Terms** section with the Marketplace terms of use. It may take a moment for validation to complete and the terms to load. Select **Terms of use** or **Privacy policy** in the **Price** section to read the full documents.

After validation passes, select **Create**.

The portal creates the required storage, managed identity, container registry, and service account. It also installs the Anyscale Kubernetes operator. Wait for the deployment to finish before you proceed to Step 3.

## Step 3: install the Ingress-Nginx controller

### 3a: get AKS credentials

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

### 3b: install Ingress-Nginx

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

To avoid setting `ANYSCALE_HOST` each session, add the following to your shell configuration file, such as `.bashrc` or `.zshrc`, and start a new shell:

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

The CLI prompts you to select your `kubectl` context and confirm the operator namespace. After you confirm, a healthy cloud returns output similar to:

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


## Clean up resources

Complete the following steps to remove the resources you created in this quickstart:

1. In the [Anyscale console](https://console.azure.anyscale.com), stop any running jobs, workspaces, and services associated with the cloud.
1. In the Azure portal, navigate to **Anyscale clouds**, select the cloud resources you wish to delete and select **Delete**. If you follow this guide, there should only be one cloud resource.
1. In the Azure portal, navigate to **Anyscale clouds**, select the cloud you wish to delete and select **Delete**.
1. In the Azure portal, navigate to your AKS cluster and select **Delete**.
1. If you created a resource group specifically for this quickstart, navigate to it in the Azure portal and select **Delete resource group** to remove any remaining resources.

During Public Preview, if you're unable to delete a resource through the portal, contact [Anyscale support](support-model.md) for assistance.

## Next steps

> [!div class="nextstepaction"]
> [Architecture overview](architecture.md)

- [Networking](networking.md)
- [Identity and access](identity-access.md)
- [Supported regions](supported-regions.md)
