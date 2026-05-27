---
title: "Quickstart: Deploy Anyscale on Azure with Ingress-Nginx"
description: Deploy your first Anyscale cloud on Azure Kubernetes Service using the Azure CLI and the Ingress-Nginx controller. Configure your subscription, create an AKS cluster, and register through the Azure portal.
author: kaysieyu
ms.author: kaysieyu
ms.date: 05/21/2026
ms.service: azure-kubernetes-service
ms.topic: quickstart
---

# Quickstart: Deploy Anyscale on Azure with Ingress-Nginx

[!INCLUDE [anyscale-public-preview](Includes/anyscale-public-preview.md)]

> [!div class="op_multi_selector" title1="Quickstart" title2="Ingress"]
> - [Envoy Gateway](quickstart-azure-cli-gateway-envoy.md)
> - [Ingress-Nginx](quickstart-azure-cli-ingress-nginx.md)

This quickstart walks you through deploying Anyscale on an existing Azure Kubernetes Service (AKS) cluster using the Ingress-Nginx ingress controller. By the end, you have a registered Anyscale cloud and are ready to run Ray workloads.

> [!CAUTION]
> Ingress-Nginx reached end of life in March 2026 and is no longer actively maintained. For new deployments that don't require an Ingress controller, use the [Envoy Gateway quickstart](quickstart-azure-cli-gateway-envoy.md) instead.

## Prerequisites and required tools

Before you begin, make sure you have:

- An Azure subscription with the Owner or Administrator role.
- Enroll in the Anyscale on Azure Public Preview before you start. Contact [Anyscale support](https://www.anyscale.com/support) to enroll, and provide your Azure subscription ID and preferred deployment regions.
- Permission to create service principals from external Microsoft Entra tenants.
- Install the following tools locally. Use the latest version of each.
  - [Azure CLI](/cli/azure/install-azure-cli)
  - [kubectl](https://kubernetes.io/docs/tasks/tools/). Your version must be within one minor version of your AKS cluster. See the [Kubernetes version skew policy](https://kubernetes.io/releases/version-skew-policy/#kubectl).
  - [Helm](https://helm.sh/docs/intro/install/)
  - [Anyscale CLI](https://docs.anyscale.com/reference/quickstart-cli): `pip install anyscale`

## Configure your Azure subscription

Creating the service principal requires permission to create service principals from external Microsoft Entra tenants. Review the prerequisite above before you proceed.

### Create the Anyscale service principal

To establish trust with the Anyscale control plane, run the following command:

```azurecli
az ad sp create --id 086bc555-6989-4362-ba30-fded273e432b
```

### Register required resource providers

Check which providers are already registered:

```azurecli
# List registered providers
az provider list --query "[?registrationState=='Registered']" --output table
```

Register any of the following that aren't listed:

```azurecli
# Register required providers
for provider in Microsoft.Storage Microsoft.ManagedIdentity Microsoft.Authorization \
  Microsoft.Resources Microsoft.Network Microsoft.ContainerService; do
  az provider register --namespace "$provider"
done
```

## Create Azure resources

You can also use existing resources if you have them. For example, if you already have an AKS cluster with OIDC issuer and workload identity enabled, you can skip to [Create an Anyscale cloud resource](#create-an-anyscale-cloud-resource).

### Create or select a resource group

You can use an existing resource group or create a new one in one of the [supported regions](supported-regions.md):

```azurecli
# Create a resource group
az group create \
  --name <resource-group> \
  --location <location>
```

### Create the AKS cluster

Before you create the cluster, confirm you have sufficient quota for the VM SKU you plan to use in your chosen region. Ray workloads require at least 4 vCPUs per worker node. `Standard_D4s_v5` or equivalent is a good starting point. Check your current quota:

```azurecli
# Check vCPU quota for the desired region and VM family
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

After Azure creates the cluster, save the resource group name and cluster name. You'll need them later.

### Node pools and workload placement

By default, Ray head nodes and worker nodes share the default node pool. For production workloads, create dedicated AKS node pools for Ray workloads and use Kubernetes taints and tolerations to steer Ray pods to those nodes.

Apply a `NoSchedule` taint to the dedicated node pool to prevent non-Ray workloads from scheduling on it. Then configure matching tolerations in your Anyscale cluster configuration so Ray pods are admitted to the tainted pool. This setup keeps Ray workers isolated from operator and system pods on the default node pool.

For production deployments, pair dedicated node pools with [declarative compute configs](https://docs.anyscale.com/configuration/compute) to define instance types, resource requirements, and workload placement in code. This is the preferred approach for Anyscale on Azure.

For GPU workloads, you need a node pool backed by a GPU-capable VM SKU. If your subscription doesn't have sufficient GPU quota, [request a quota increase in the Azure portal](/azure/quotas/quickstart-increase-quota-portal) before creating the node pool.

For supported VM types and Ray sizing recommendations, see [Supported instance types](https://docs.anyscale.com/configuration/compute#supported-types) in the Anyscale documentation. To map model size to GPU memory for batch inference workloads, see [GPU costs and selection](https://docs.anyscale.com/llm/batch-inference/resource-allocation/cost-performance#gpu-costs).

For full details on creating and configuring AKS node pools, see [Manage node pools in AKS](/azure/aks/manage-node-pools).

## Create an Anyscale cloud resource

In this section, you create an Anyscale cloud resource in the Azure portal and link it to your AKS cluster. The Anyscale cloud resource represents the cluster in the Anyscale control plane and allows you to run Ray workloads on it.

> [!NOTE]
> The Anyscale Operator is also available through Azure Marketplace, but Anyscale doesn't recommend that route. Use the Anyscale Clouds Resource Provider in the Azure portal instead.

In the [Azure portal](https://portal.azure.com), search for **Anyscale clouds** in the global search bar and select **Anyscale clouds** under **Services** from the results.

:::image type="content" source="media/quickstart/quickstart-anyscale-clouds-landing.png" alt-text="Anyscale clouds page in the Azure portal showing a list of existing Anyscale cloud resources.":::

Select **Create**. In the **Create Anyscale Operator** pane, complete the following steps:

1. On the **Basics** tab, enter or select the following information:

   - **Subscription**: Select the Azure subscription where you created your AKS cluster.
   - **Resource group**: Select the resource group where you created your AKS cluster.
   - **Cloud name**: Enter a unique name for your Anyscale cloud. This is the name that appears in the Anyscale console and CLI.
   - **Region**: Select the same region where you created your AKS cluster.
   - **Cluster**: Select the AKS cluster you created.

   :::image type="content" source="media/quickstart/quickstart-create-basics-filled.png" alt-text="Basics tab with subscription, resource group, cloud name, region, and AKS cluster filled in.":::

   Select **Next**.

1. On the **Infrastructure settings** tab, the portal prepopulates a **Storage account name** and **Anyscale operator identity name**. Accept the defaults or enter custom names. Select **Next**.

1. On the **Container registry** tab, select an **ACR mode**:

   - **Create new ACR** (default): The portal prepopulates a name. Accept the default or enter a custom name. Role assignments are configured automatically.
   - **Use Existing ACR**: Select an existing ACR from the dropdown. Role assignments are configured automatically.
   - **No ACR**: Skip ACR configuration. You can [configure container image builds](configure-container-image-builds.md) later.

   > [!NOTE]
   > This ACR is used exclusively for Anyscale container image builds. To configure your cluster to pull Ray images from a different registry, see [Configure a custom container image registry](https://docs.anyscale.com/container-image/image-registry) in the Anyscale documentation.

   Select **Next**.

1. On the **Support plan** tab, review the support tier for your Anyscale cloud. This value is fixed and can't be changed. For details, see [Support model](support-model.md). Select **Next**.

1. On the **Tags** tab, optionally add name/value pairs to categorize resources for billing and cost management. Select **Next**.

1. On the **Review + submit** tab, review the Marketplace terms of use. It can take a moment for validation to complete. After validation passes, select **Create**.

The portal creates the required storage, managed identity, container registry, and service account, and installs the Anyscale Kubernetes operator. The deployment takes about 5–8 minutes. Wait for it to finish before you proceed.

## Assign access to your team

After cloud creation completes, you and any teammates who need to create workspaces, jobs, or services must hold the **Anyscale Platform Contributor** role on the cloud resource. Subscription Owner or Contributor permissions on the underlying Azure resources don't carry over to the Anyscale resource provider. Workload operations require an explicit Anyscale platform role.

To assign the role, navigate to your Anyscale cloud resource in the Azure portal, select **Access control (IAM)** > **Add** > **Add role assignment**, and assign **Anyscale Platform Contributor** to the appropriate users, groups, or service principals.

For the full list of Anyscale platform roles and the resource-provider actions they control, see [Identity and access](identity-access.md#azure-built-in-roles-for-anyscale).

> [!NOTE]
> Skipping this step can cause workspace, job, or service creation to fail with a `404` error. Azure Resource Manager returns 404 instead of 403 when the caller doesn't have `read` permission on the parent Anyscale cloud resource.

## Install the Ingress-Nginx controller

Before you can run Ray workloads on your cluster, you need to install an Ingress controller. The controller manages external access to the Ray head node and services running on the cluster. This quickstart uses the Ingress-Nginx controller, but you can use any Kubernetes-native Ingress controller that supports annotations.

### Get AKS credentials

Run the following command to configure your local `kubectl` to connect to the AKS cluster. Replace the placeholders with your resource group and cluster name:

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

### Install Ingress-Nginx

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

After the controller is up and running, verify that your Anyscale cloud is healthy and can communicate with the operator.

1. Set the Anyscale console URL and sign in:

   ```bash
   export ANYSCALE_HOST=https://console.azure.anyscale.com
   anyscale login
   ```

   To avoid setting `ANYSCALE_HOST` each session, add the export to your shell configuration file (`.bashrc` or `.zshrc`) and start a new shell.

1. Set your `kubectl` context to the correct cluster:

   ```bash
   kubectl config use-context <cluster-name>
   ```

1. Find your cloud ID from the Anyscale console or by running:

   ```bash
   anyscale cloud list
   ```

   The cloud ID has the format `cld_*`.

1. Verify the cloud:

   ```bash
   anyscale cloud verify --id <cloud-id>
   ```

   The CLI prompts you to select your `kubectl` context and confirm the operator namespace. After you confirm, a healthy cloud returns output similar to:

   ```plaintext
   Overall Result: ALL 1 cloud resources verified successfully
   ```

> [!NOTE]
> During Public Preview, the Anyscale CLI supports only read operations against Azure cloud resources. Manage clouds and cloud resources through the Anyscale Clouds Resource Provider in the Azure portal. For details, see [Public Preview limitations](overview.md#public-preview-limitations).

## Run your first workload

Now that your cloud is set up and verified, you can run a Ray job on it. Create a simple Ray program and submit it as a job through the Anyscale CLI.

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
1. In the Azure portal, navigate to **Anyscale clouds**, select the cloud resources to delete, and select **Delete**. If you follow this guide, there should only be one cloud resource.
1. In the Azure portal, navigate to **Anyscale clouds**, select the cloud to delete, and select **Delete**.
1. In the Azure portal, navigate to your AKS cluster and select **Delete**.
1. If you created a resource group specifically for this quickstart, navigate to it in the Azure portal and select **Delete resource group** to remove any remaining resources.

During Public Preview, if you're unable to delete a resource through the portal, contact [Anyscale support](support-model.md) for assistance.

## Add a second cloud resource

An Anyscale cloud can include multiple AKS clusters through cloud resources. Each cloud resource represents one AKS cluster, so you can run Ray workloads across different configurations within the same cloud.

To add another cloud resource to an existing Anyscale cloud, use the Azure portal:

1. Navigate to the **Anyscale clouds** page and select your cloud from the list.
1. Select **Resources** to expand the menu, then select **Cloud Resources**.
1. Select **Create** and follow the setup wizard.

## Next steps

> [!div class="nextstepaction"]
> [Architecture overview](architecture.md)

- [Networking](networking.md)
- [Identity and access](identity-access.md)
- [Configure container image builds for an existing cloud](configure-container-image-builds.md)
- [Supported regions](supported-regions.md)
- [Configure head node fault tolerance](https://docs.anyscale.com/administration/resource-management/head-node-fault-tolerance) before running production Anyscale Services
