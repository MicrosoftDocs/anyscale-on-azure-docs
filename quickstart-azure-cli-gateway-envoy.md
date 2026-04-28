---
title: "Quickstart: Deploy Anyscale with Envoy Gateway | Microsoft Learn"
description: Deploy your first Anyscale cloud on Azure Kubernetes Service using the Azure CLI and the Envoy gateway controller. Configure your subscription, create an AKS cluster, and register via the Azure portal.
author: kaysieyu
ms.author: kaysieyu
ms.date: 04/16/2026
ms.service: azure-kubernetes-service
ms.topic: quickstart
---


# Quickstart: Deploy Anyscale with Envoy Gateway

> [!div class="op_multi_selector" title1="Quickstart" title2="Ingress"]
> - [Envoy Gateway](quickstart-azure-cli-gateway-envoy.md)
> - [Ingress-Nginx](quickstart-azure-cli-ingress-nginx.md)

This quickstart walks you through deploying Anyscale on an existing Azure Kubernetes Service (AKS) cluster using the Envoy gateway controller. By the end, you'll have a registered Anyscale cloud and be ready to run Ray workloads.

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

Azure validates your configuration before enabling the **Create** button. Once validation passes, select **Create**.

The portal creates the required storage, managed identity, and service account, and installs the Anyscale Kubernetes operator automatically.

## Step 3: install the Envoy gateway controller

The TLS certificate secrets (`anyscale-<cloud-resource-id>-certificate` and `anyscale-svc-<cloud-resource-id>-certificate`) are created automatically by the Anyscale operator after installation. Find the Cloud Resource ID in the Anyscale console under your cloud's settings.

| Identifier | Format | Where to find it | Used for |
|---|---|---|---|
| **Cloud ID** | `cld_*` | `anyscale cloud list` or the Anyscale console | `anyscale cloud verify --id` |
| **Cloud Resource ID** | `cldrsrc_*` | Anyscale console, cloud settings page | TLS cert secret names in `gateway.yaml` |

Use the Cloud Resource ID throughout Step 3, replacing underscores with hyphens: `cldrsrc-<id>`.

### 3a: Get AKS credentials

```bash
az aks get-credentials \
  --resource-group <azure-resource-group-name> \
  --name <your-aks-cluster-name> \
  --overwrite-existing
```

### 3b: Install Envoy Gateway

```bash
helm install eg oci://docker.io/envoyproxy/gateway-helm \
  --version v1.7.0 \
  --namespace envoy-gateway-system \
  --create-namespace

kubectl wait --for=condition=available deployment/envoy-gateway \
  -n envoy-gateway-system --timeout=120s
```

### 3c: Create and apply envoyproxy.yaml

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

### 3d: Create and apply gatewayclass.yaml

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

### 3e: Create and apply gateway.yaml

Replace `<cloud-resource-id>` with the value from Step 3 (the `global.cloudDeploymentId` value with underscores converted to hyphens, e.g. `cldrsrc-<id>`).

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
    hostname: '*.i.anyscaleuserdata.com'
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
    hostname: '*.s.anyscaleuserdata.com'
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

### 3f: Configure the Anyscale operator with gateway settings

Update the operator extension configuration, replacing `<cluster-name>`, `<resource-group>`, and `<gateway-lb-address>` with your values:

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

This updates only the gateway settings. All other operator configuration set during portal installation is preserved automatically.

## Verify the deployment

In the Azure portal, navigate to your Anyscale cloud resource and confirm the **Status** shows **Succeeded**.

You can also verify from the Anyscale CLI. First, set the Anyscale console URL and sign in:

```bash
export ANYSCALE_HOST=https://console.azure.anyscale.com
anyscale login
```

To avoid setting `ANYSCALE_HOST` each session, add the following to your shell configuration (`.bashrc` or `.zshrc`):

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

1. Submit the job, using the cloud ID from the verify step:

   ```bash
   anyscale job submit -f job.yaml --cloud <cloud-id>
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
