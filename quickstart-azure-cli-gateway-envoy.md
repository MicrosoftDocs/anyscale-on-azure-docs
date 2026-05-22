---
title: "Quickstart: Deploy Anyscale on Azure with Envoy Gateway"
description: Deploy your first Anyscale cloud on Azure Kubernetes Service using the Azure CLI and the Envoy Gateway controller. Configure your subscription, create an AKS cluster, and register through the Azure portal.
author: kaysieyu
ms.author: kaysieyu
ms.date: 05/21/2026
ms.service: azure-kubernetes-service
ms.topic: quickstart
---

# Quickstart: Deploy Anyscale on Azure with Envoy Gateway

[!INCLUDE [anyscale-public-preview](Includes/anyscale-public-preview.md)]

> [!div class="op_multi_selector" title1="Quickstart" title2="Ingress"]
> - [Envoy Gateway](quickstart-azure-cli-gateway-envoy.md)
> - [Ingress-Nginx](quickstart-azure-cli-ingress-nginx.md)

This quickstart walks you through deploying Anyscale on an existing Azure Kubernetes Service (AKS) cluster using the Envoy Gateway controller. By the end, you have a registered Anyscale cloud and are ready to run Ray workloads.

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

Enroll in the Anyscale on Azure Public Preview before you start. Contact [Anyscale support](https://www.anyscale.com/support) to enroll, and provide your Azure subscription ID and preferred deployment regions.

## Configure your Azure subscription

Step 0a requires permission to create service principals from external Microsoft Entra tenants. Review the prerequisite above before you proceed.

### Create the Anyscale service principal

To establish trust with the Anyscale control plane, run the following command:

```azurecli
az ad sp create --id aaaaaaaa-bbbb-cccc-1111-222222222222
```

### Register required resource providers

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

## Provision Azure resources

### Create or select a resource group

You can use an existing resource group or create a new one in one of the [supported regions](supported-regions.md):

```azurecli
az group create \
  --name <resource-group> \
  --location <location>
```

### Create the AKS cluster

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

### Node pools and workload placement

By default, Ray head nodes and worker nodes share the default node pool. For production workloads, create dedicated AKS node pools for Ray workloads and use Kubernetes taints and tolerations to steer Ray pods to those nodes.

Apply a `NoSchedule` taint to the dedicated node pool to prevent non-Ray workloads from scheduling on it. Then configure matching tolerations in your Anyscale cluster configuration so Ray pods are admitted to the tainted pool. This setup keeps Ray workers isolated from operator and system pods on the default node pool.

For production deployments, pair dedicated node pools with [declarative compute configs](https://docs.anyscale.com/configuration/compute) to define instance types, resource requirements, and workload placement in code. This is the preferred approach for Anyscale on Azure.

For GPU workloads, you need a node pool backed by a GPU-capable VM SKU. If your subscription doesn't have sufficient GPU quota, [request a quota increase in the Azure portal](/azure/quotas/quickstart-increase-quota-portal) before creating the node pool.

For supported VM types and Ray sizing recommendations, see [Supported instance types](https://docs.anyscale.com/configuration/compute#supported-types) in the Anyscale documentation. To map model size to GPU memory for batch inference workloads, see [GPU costs and selection](https://docs.anyscale.com/llm/batch-inference/resource-allocation/cost-performance#gpu-costs).

For full details on creating and configuring AKS node pools, see [Manage node pools in AKS](/azure/aks/manage-node-pools).

## Create an Anyscale cloud resource

> [!NOTE]
> The Anyscale Operator is also available through the Azure Marketplace, but Anyscale doesn't recommend that route. Use the Anyscale Clouds Resource Provider in the Azure portal instead.

In the [Azure portal](https://portal.azure.com), search for **Anyscale clouds** in the global search bar and select **Anyscale clouds** under **Services** from the results.

:::image type="content" source="media/quickstart/quickstart-anyscale-clouds-landing.png" alt-text="Anyscale clouds page in the Azure portal showing a list of existing Anyscale cloud resources.":::

Select **Create** and complete the wizard:

1. On the **Basics** tab, confirm your **Subscription**, select the **Resource group** you created, enter a unique **Cloud name**, select the same **Region**, and select your **Cluster**. Select **Next**.

   :::image type="content" source="media/quickstart/quickstart-create-basics-filled.png" alt-text="Basics tab with subscription, resource group, cloud name, region, and AKS cluster filled in.":::

1. On the **Infrastructure settings** tab, the portal pre-populates a **Storage account name** and **Anyscale operator identity name**. Accept the defaults or enter custom names. Select **Next**.

1. On the **Container registry** tab, select an **ACR mode**:

   - **Create new ACR** (default): The portal pre-populates a name. Accept the default or enter a custom name. Role assignments are configured automatically.
   - **Use Existing ACR**: Select an existing ACR from the dropdown. Role assignments are configured automatically.
   - **No ACR**: Skip ACR configuration. You can [configure container image builds](configure-container-image-builds.md) later.

   > [!NOTE]
   > This ACR is used exclusively for Anyscale container image builds. To configure your cluster to pull Ray images from a different registry, see [Configure a custom container image registry](https://docs.anyscale.com/container-image/image-registry) in the Anyscale documentation.

   Select **Next**.

1. On the **Support plan** tab, review the support tier for your Anyscale cloud. This value is fixed and can't be changed. For details, see [Support model](support-model.md). Select **Next**.

1. On the **Tags** tab, optionally add name/value pairs to categorize resources for billing and cost management. Select **Next**.

1. On the **Review + submit** tab, review the Marketplace terms of use. It may take a moment for validation to complete. After validation passes, select **Create**.

The portal creates the required storage, managed identity, container registry, and service account, and installs the Anyscale Kubernetes operator. The deployment takes about 5–8 minutes. Wait for it to finish before you proceed.

### Assign access to your team

After cloud creation completes, you and any teammates who need to create workspaces, jobs, or services must hold the **Anyscale Platform Contributor** role on the cloud resource. Subscription Owner or Contributor permissions on the underlying Azure resources don't carry over to the Anyscale resource provider. Workload operations require an explicit Anyscale platform role.

To assign the role, navigate to your Anyscale cloud resource in the Azure portal, select **Access control (IAM)** > **Add** > **Add role assignment**, and assign **Anyscale Platform Contributor** to the appropriate users, groups, or service principals.

For the full list of Anyscale platform roles and the resource-provider actions they control, see [Identity and access](identity-access.md#azure-built-in-roles-for-anyscale).

> [!NOTE]
> Skipping this step can cause workspace, job, or service creation to fail with a `404` error. Azure Resource Manager returns 404 instead of 403 when the caller doesn't have read permission on the parent Anyscale cloud resource.

## Install the Envoy Gateway controller

After installation, the Anyscale operator creates the TLS certificate secrets (`anyscale-<cloud-resource-id>-certificate` and `anyscale-svc-<cloud-resource-id>-certificate`) automatically. Find the Cloud Resource ID in the Anyscale console under your cloud's settings.

| Identifier | Format | Where to find it | Used for |
|---|---|---|---|
| **Cloud ID** | `cld_*` | `anyscale cloud list` or the Anyscale console | `anyscale cloud verify --id` |
| **Cloud Resource ID** | `cldrsrc_*` | Anyscale console, cloud settings page | TLS cert secret names in `gateway.yaml` |

Throughout Step 3, replace underscores in the Cloud Resource ID with hyphens: `cldrsrc-<id>`.

### Get AKS credentials



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

### Install Envoy Gateway

```bash
helm install eg oci://docker.io/envoyproxy/gateway-helm \
  --version v1.7.0 \
  --namespace envoy-gateway-system \
  --create-namespace

kubectl wait --for=condition=available deployment/envoy-gateway \
  -n envoy-gateway-system --timeout=120s
```

### Create and apply envoyproxy.yaml

```yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: EnvoyProxy
metadata:
  name: envoy-proxy
  namespace: envoy-gateway-system
spec:
  provider:
    type: Kubernetes
    kubernetes:
      envoyService:
        type: LoadBalancer
        annotations:
          service.beta.kubernetes.io/azure-load-balancer-internal: "false"
```

```bash
kubectl apply -f envoyproxy.yaml
```

### Create and apply gatewayclass.yaml

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: eg
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
  parametersRef:
    group: gateway.envoyproxy.io
    kind: EnvoyProxy
    name: envoy-proxy
    namespace: envoy-gateway-system
```

```bash
kubectl apply -f gatewayclass.yaml
```

### Create and apply gateway.yaml

Replace `<cloud-resource-id>` with the value from the table at the start of Step 3. Take the `global.cloudDeploymentId` value and convert underscores to hyphens, for example `cldrsrc-<id>`.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: gateway
  namespace: anyscale-operator
spec:
  gatewayClassName: eg
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: Same
  - name: https
    port: 443
    protocol: HTTPS
    hostname: '*.i.azure.anyscaleuserdata.com'
    tls:
      mode: Terminate
      certificateRefs:
      - kind: Secret
        name: anyscale-<cloud-resource-id>-certificate
    allowedRoutes:
      namespaces:
        from: Same
  - name: https-session
    port: 443
    protocol: HTTPS
    hostname: '*.s.azure.anyscaleuserdata.com'
    tls:
      mode: Terminate
      certificateRefs:
      - kind: Secret
        name: anyscale-svc-<cloud-resource-id>-certificate
    allowedRoutes:
      namespaces:
        from: Same
```

```bash
kubectl apply -f gateway.yaml
```

After applying, retrieve the load balancer address:

```bash
kubectl get gateway gateway -n anyscale-operator -o jsonpath='{.status.addresses[0].value}'
```

### Configure the Anyscale operator with gateway settings

Update the operator extension configuration. Replace `<cluster-name>`, `<resource-group>`, and `<gateway-lb-address>` with your values:

```azurecli
az k8s-extension update \
  --cluster-name <cluster-name> \
  --resource-group <resource-group> \
  --cluster-type managedClusters \
  --name anyscaleoperator \
  --yes \
  --configuration-settings \
    networking.gateway.enabled=true \
    networking.gateway.name=gateway \
    networking.gateway.className=eg \
    networking.gateway.namespace=anyscale-operator \
    "networking.gateway.apiVersion=gateway.networking.k8s.io/v1" \
    networking.gateway.hostname=<gateway-lb-address>
```

This command updates only the gateway settings. The update preserves all other operator configuration set during portal installation.

Anyscale on Azure installs the operator as an AKS extension, not a standalone Helm release. Use `az k8s-extension update --configuration-settings` to pass Helm values to the operator. Don't use the Helm CLI directly to configure the operator.

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

1. Navigate to the Anyscale clouds page (the same page as [Create an Anyscale cloud resource](#create-an-anyscale-cloud-resource)) and select your cloud from the list.
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
