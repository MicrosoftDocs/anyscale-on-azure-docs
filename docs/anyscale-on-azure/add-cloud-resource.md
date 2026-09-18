---
title: Add a cloud resource with an ARM template
description: Add a Kubernetes cluster to an existing Anyscale cloud on Azure as a new cloud resource by deploying an Azure Resource Manager (ARM) template.
author: kaysieyu
ms.author: kaysieyu
ms.reviewer: mbender
ms.date: 09/18/2026
ms.service: azure-kubernetes-service
ms.topic: how-to
ms.custom: references_regions
---

# Add a cloud resource with an ARM template

[!INCLUDE [anyscale-public-preview](../../Includes/anyscale-public-preview.md)]

This article shows you how to add a Kubernetes cluster to an existing Anyscale cloud as a new cloud resource by deploying an Azure Resource Manager (ARM) template. For background on cloud resources, see [What is a cloud resource on Azure?](cloud-resources-overview.md)

> [!IMPORTANT]
> Only steps 1 through 4 use the Azure CLI. Steps 5 through 9 require kubectl, Helm, your cloud provider's CLI, and the Anyscale CLI. Each step names the tool it uses.

## Prerequisites

- An existing Anyscale cloud with its default cloud resource. The first cloud resource on a cloud must use provider `Azure`, so the cloud must already exist before you add a resource of any other provider.
- A running Kubernetes cluster that meets the [Anyscale on Kubernetes requirements](https://docs.anyscale.com/clouds/kubernetes#general-requirements-kubernetes): cluster version, egress, service account, GPU device plugin, and load balancer.
- An object storage bucket and an identity that can read and write it, per [Object storage and IAM](https://docs.anyscale.com/clouds/kubernetes#object-storage). Create the identity and bind it to the cluster's service account **before** you create the cloud resource. You record its value at creation and can't change it afterward.
- The **Anyscale Platform Contributor** role on the Anyscale cloud. Without it, the deployment fails with a 404 error, as though the cloud doesn't exist.
- The Anyscale cloud's ARM resource ID and its resource group. Take the ID from `anyscale cloud list -j` rather than reconstructing it. The `cloudName` parameter in the following section is its last segment.
- The following tools installed locally: [Azure CLI](/cli/azure/install-azure-cli), [kubectl](https://kubernetes.io/docs/tasks/tools/), [Helm](https://helm.sh/docs/intro/install/), and the [Anyscale CLI](https://docs.anyscale.com/reference/quickstart-cli).

## Cloud resource properties

This table describes the fields you set under the `properties` object in the ARM template. The `name` and `location` fields apply to every Azure resource, not only cloud resources, and are covered in [Resource name and location](#resource-name-and-location).

| Property | Description | Required | Notes |
|---|---|---|---|
| `provider` | Cloud provider hosting the Kubernetes cluster that backs this cloud resource. | Yes | Enum. Values are case-sensitive and must be spelled exactly `Azure`, `AWS`, `GCP`, or `Generic`. |
| `computeStack` | Compute stack for the cloud resource. | Yes | `K8S` is the only supported value. |
| `cloudStorageBucketName` | Fully qualified URI of the object storage bucket this cloud resource uses for logs and artifacts. | Yes | The scheme must match the provider: `abfss://` or `azure://` for `Azure`, `s3://` for `AWS`, `gs://` for `GCP`, and any fully qualified `<scheme>://<bucket>` for `Generic`. The URI must name the bucket only. Sub-paths such as `s3://my-bucket/prefix` are rejected. |
| `cloudStorageBucketEndpoint` | Endpoint the storage bucket is reached at, given as a bare `<scheme>://<host>` origin with no path, query, or credentials. | Yes for `Azure`. No for other providers. | A bare `https://<host>` origin for Azure. Empty for other providers, unless you're overriding the endpoint for S3-compatible storage. |
| `region` | Cloud provider region where compute for this cloud resource runs. | Yes | Must match `^[a-z0-9]+(-[a-z0-9]+)*$` and be 35 characters or fewer. |
| `cloudStorageBucketRegion` | Region the storage bucket lives in. | No | Defaults to `region`. Set it if the bucket is in a different region. |
| `anyscaleOperatorIamIdentity` | The identity the Anyscale operator presents when it registers this cloud resource. | Yes for `Azure`, `AWS`, and `GCP`. No for `Generic`. | For Azure, this is the managed identity's principal ID, while the operator is configured with the same identity's client ID. These are two different GUIDs. For AWS it's the IAM role ARN, and for Google Cloud the service account email, the same value in both places. Generic clusters have no cloud provider identity and authenticate with an Anyscale CLI token instead. |

All properties are set at creation and can't be changed afterward.

### Resource name and location

Set two values on the cloud resource itself rather than inside `properties`:

- `name`: The name of the cloud resource, supplied as the last segment of the resource path. It must match `^[a-zA-Z0-9_-]+$`, be 256 characters or fewer, and be unique within the cloud.
- `location`: The Azure region that Azure Resource Manager tracks the resource record in. Use the same Azure region as the parent Anyscale cloud. It doesn't affect where compute runs. `region` controls that.

## Add the cloud resource

Complete the following steps in order.

### Step 1: Confirm the API version

Using the Azure CLI, confirm the API version is available in your subscription:

```azurecli
az resource show \
  --ids "<cloud-arm-resource-id>/cloudResources/default" \
  --api-version 2026-09-01 -o json
```

This command returns the cloud's existing default resource and confirms the API version is available. An `InvalidApiVersionParameter` error means the API version isn't available yet.

### Step 2: Create the template file

In any text editor, save the following code as `cloudresource.json`. This template works for all four providers. Only the parameters change.

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "cloudName": {
      "type": "string"
    },
    "resourceName": {
      "type": "string"
    },
    "location": {
      "type": "string"
    },
    "provider": {
      "type": "string",
      "allowedValues": [ "Azure", "AWS", "GCP", "Generic" ]
    },
    "region": {
      "type": "string"
    },
    "cloudStorageBucketName": {
      "type": "string"
    },
    "cloudStorageBucketEndpoint": {
      "type": "string",
      "defaultValue": ""
    },
    "cloudStorageBucketRegion": {
      "type": "string"
    },
    "anyscaleOperatorIamIdentity": {
      "type": "string",
      "defaultValue": ""
    }
  },
  "resources": [
    {
      "type": "Anyscale.Platform/clouds/cloudResources",
      "apiVersion": "2026-09-01",
      "name": "[concat(parameters('cloudName'), '/', parameters('resourceName'))]",
      "location": "[parameters('location')]",
      "properties": {
        "provider": "[parameters('provider')]",
        "region": "[parameters('region')]",
        "computeStack": "K8S",
        "cloudStorageBucketName": "[parameters('cloudStorageBucketName')]",
        "cloudStorageBucketEndpoint": "[parameters('cloudStorageBucketEndpoint')]",
        "cloudStorageBucketRegion": "[parameters('cloudStorageBucketRegion')]",
        "anyscaleOperatorIamIdentity": "[parameters('anyscaleOperatorIamIdentity')]"
      }
    }
  ],
  "outputs": {
    "cloudResourceId": {
      "type": "string",
      "value": "[reference(resourceId('Anyscale.Platform/clouds/cloudResources', parameters('cloudName'), parameters('resourceName')), '2026-09-01').cloudResourceId]"
    }
  }
}
```

### Step 3: Create the parameters file

In any text editor, save one of the following files as `cloudresource.parameters.json` and edit the values for your environment.

#### Azure (AKS)

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "cloudName":                   { "value": "my-anyscale-cloud" },
    "resourceName":                { "value": "aksuksouth" },
    "location":                    { "value": "uksouth" },
    "provider":                    { "value": "Azure" },
    "region":                      { "value": "uksouth" },
    "cloudStorageBucketName":      { "value": "abfss://anyscale-data@mystorageacct.dfs.core.windows.net" },
    "cloudStorageBucketEndpoint":  { "value": "https://mystorageacct.blob.core.windows.net" },
    "cloudStorageBucketRegion":    { "value": "uksouth" },
    "anyscaleOperatorIamIdentity": { "value": "00000000-0000-0000-0000-000000000000" }
  }
}
```

For Azure, `anyscaleOperatorIamIdentity` is the managed identity's **principal ID**.

#### AWS (EKS)

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "cloudName":                   { "value": "my-anyscale-cloud" },
    "resourceName":                { "value": "eksuswest2" },
    "location":                    { "value": "westus2" },
    "provider":                    { "value": "AWS" },
    "region":                      { "value": "us-west-2" },
    "cloudStorageBucketName":      { "value": "s3://my-anyscale-eks-bucket" },
    "cloudStorageBucketEndpoint":  { "value": "" },
    "cloudStorageBucketRegion":    { "value": "us-west-2" },
    "anyscaleOperatorIamIdentity": { "value": "arn:aws:iam::123456789012:role/anyscale-operator-role" }
  }
}
```

#### Google Cloud (GKE)

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "cloudName":                   { "value": "my-anyscale-cloud" },
    "resourceName":                { "value": "gkeuscentral1" },
    "location":                    { "value": "westus2" },
    "provider":                    { "value": "GCP" },
    "region":                      { "value": "us-central1" },
    "cloudStorageBucketName":      { "value": "gs://my-anyscale-gke-bucket" },
    "cloudStorageBucketEndpoint":  { "value": "" },
    "cloudStorageBucketRegion":    { "value": "us-central1"  },
    "anyscaleOperatorIamIdentity": { "value": "anyscale-operator@my-project.iam.gserviceaccount.com" }
  }
}
```

#### Generic Kubernetes

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "cloudName":                   { "value": "my-anyscale-cloud" },
    "resourceName":                { "value": "onpremdc1" },
    "location":                    { "value": "westus2" },
    "provider":                    { "value": "Generic" },
    "region":                      { "value": "dc1" },
    "cloudStorageBucketName":      { "value": "s3://my-anyscale-onprem-bucket" },
    "cloudStorageBucketEndpoint":  { "value": "" },
    "cloudStorageBucketRegion":    { "value": "dc1" }
  }
}
```

For Generic Kubernetes, `region` is a free-form routing label and isn't validated against a region list.

### Step 4: Preview and deploy

Preview the deployment first with the Azure CLI. The what-if output shows one `Create`. Other resources in the group listed as `Ignore` are incremental mode listing them, not proposed changes.

```azurecli
az deployment group what-if \
  --resource-group <resource-group> \
  --template-file cloudresource.json \
  --parameters cloudresource.parameters.json

az deployment group create \
  --name add-cloudresource-<resource-name> \
  --resource-group <resource-group> \
  --template-file cloudresource.json \
  --parameters cloudresource.parameters.json \
  --query properties.outputs
```

Record the `cloudResourceId` from the output, the generated `cldrsrc_*` ID. The remaining steps need it.

### Step 5: Install the Anyscale operator

Install the operator with the Azure CLI or Helm. One operator serves one cloud resource. Adding a second cluster always means a second cloud resource and a second operator installation.

The identity you configure here must match the one recorded on the cloud resource:

| Provider | Cloud resource `anyscaleOperatorIamIdentity` | Operator `global.auth.iamIdentity` |
|---|---|---|
| `Azure` | Managed identity **principal ID** | The same identity's **client ID**, a different GUID |
| `AWS` | IAM role ARN | The same ARN |
| `GCP` | Service account email | The same email |
| `Generic` | Not required | Not used. Set `global.auth.anyscaleCliToken`. |

#### AKS cluster

Install the operator as a cluster extension:

```azurecli
az k8s-extension create \
  --cluster-name <aks-cluster-name> \
  --resource-group <resource-group> \
  --cluster-type managedClusters \
  --name anyscaleoperator \
  --extension-type Anyscale.AKS.Operator \
  --release-train stable \
  --plan-name anyscale-operator \
  --plan-publisher anyscale1750870039553 \
  --plan-product anyscale-operator-aks \
  --no-wait \
  --configuration-settings \
    global.cloudDeploymentId=<cloud-resource-id> \
    global.controlPlaneURL=https://console.azure.anyscale.com \
    global.auth.iamIdentity=<managed-identity-client-id> \
    global.auth.audience=<control-plane-audience> \
    workloads.serviceAccount.name=anyscale-operator
```

> [!IMPORTANT]
> On AKS, configure the operator only with `az k8s-extension update --configuration-settings`. Changes applied with the Helm CLI are reverted when the extension agent next reconciles.

#### EKS, GKE, or generic cluster

Install the operator with Helm. For the full list of values, see [Configure the Helm chart](https://docs.anyscale.com/clouds/kubernetes/helm-ref).

Create `values.yaml`. This example is for EKS:

```yaml
global:
  cloudDeploymentId: <cloud-resource-id>
  cloudProvider: aws
  controlPlaneURL: https://console.azure.anyscale.com
  aws:
    region: us-west-2
  auth:
    iamIdentity: arn:aws:iam::123456789012:role/anyscale-operator-role
workloads:
  serviceAccount:
    name: anyscale-operator
```

For GKE, set `cloudProvider: gcp` and `auth.iamIdentity` to the service account email, and omit the `aws` block. For a generic cluster, set `cloudProvider: generic` and replace `auth.iamIdentity` with `auth.anyscaleCliToken`, generated in the Anyscale console under **User settings** > **API keys**.

```bash
helm repo add anyscale https://anyscale.github.io/helm-charts
helm repo update anyscale

helm upgrade anyscale-operator anyscale/anyscale-operator \
  --version <chart-version> \
  --namespace anyscale-operator --create-namespace \
  -f values.yaml --wait -i
```

> [!IMPORTANT]
> Set `global.controlPlaneURL` to `https://console.azure.anyscale.com`. It isn't included in the example values in the Anyscale documentation and defaults to a different control plane, so an installation that omits it contacts the wrong one.

### Step 6: Install the gateway controller

Every cloud resource needs its own gateway or ingress controller before workloads can run on it. Install the gateway controller by using Helm and kubectl. Follow the steps in [Install Envoy Gateway](https://docs.anyscale.com/clouds/kubernetes/gateway-envoy#install-envoy-gateway), with one change: on an Anyscale cloud on Azure, the gateway's HTTPS listeners use the Azure control plane's hostnames.

Use these listeners in place of the two HTTPS listeners in that procedure. The `http` listener on port 80 stays the same. Replace `<cloud-resource-id>` with the ID from step 4, converting underscores to hyphens. For example, `cldrsrc_abc123` becomes `cldrsrc-abc123`:

```yaml
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
        from: All
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
        from: All
```

On an AKS cluster, apply that procedure's final operator configuration step by using `az k8s-extension update --configuration-settings` instead of `helm upgrade`:

```azurecli
az k8s-extension update \
  --cluster-name <aks-cluster-name> \
  --resource-group <resource-group> \
  --cluster-type managedClusters \
  --name anyscaleoperator --yes \
  --configuration-settings \
    networking.gateway.enabled=true \
    networking.gateway.name=gateway \
    networking.gateway.className=eg \
    networking.gateway.namespace=anyscale-operator \
    "networking.gateway.apiVersion=gateway.networking.k8s.io/v1" \
    networking.gateway.hostname=<gateway-lb-address>
```

### Step 7: Configure bucket CORS

Skip this step for Azure cloud resources. Anyscale configures their storage account during installation.

The Anyscale console fetches log contents and files in your browser, directly from the bucket, so the bucket must allow the console's origin. Without it, the **Logs** tab is empty for workloads on this cloud resource.

These commands replace the bucket's entire CORS configuration. Read the existing configuration first if you have rules to preserve.

#### AWS (S3)

```bash
aws s3api put-bucket-cors --bucket <bucket> --region <bucket-region> \
  --cors-configuration '{"CORSRules":[{
    "AllowedOrigins":["https://console.azure.anyscale.com"],
    "AllowedMethods":["GET","HEAD","PUT","POST","DELETE"],
    "AllowedHeaders":["*"],
    "ExposeHeaders":["Accept-Ranges","Content-Range","Content-Length"],
    "MaxAgeSeconds":3600}]}'
```

Use the cloud resource's own region, not the Anyscale cloud's ARM location. The three exposed headers are required by the file viewer. Log output renders without them.

#### Google Cloud (Cloud Storage)

Save this as `cors.json`:

```json
[
  {
    "origin": ["https://console.azure.anyscale.com"],
    "method": ["GET", "HEAD", "PUT", "POST", "DELETE"],
    "responseHeader": ["*"],
    "maxAgeSeconds": 3600
  }
]
```

```bash
gcloud storage buckets update gs://<bucket> --cors-file=cors.json
```

After the rule is in place, reload the Anyscale console with a hard refresh. A soft refresh serves the cached failure.

### Step 8: Verify the cloud resource

Using kubectl, check the operator pods:

```bash
kubectl get pods -n anyscale-operator
```

The operator pods report `Running`. A pod becomes `Ready` only after every registration check passes, so readiness is the signal that registration succeeded.

Then verify with the Anyscale CLI:

```bash
export ANYSCALE_HOST=https://console.azure.anyscale.com
anyscale login

anyscale cloud verify -n <cloud-name> --cloud-resource-name <resource-name>
```

> [!NOTE]
> On Azure, a cloud's name is its full ARM resource ID in lowercase, in the form `/subscriptions/<subscription-id>/resourcegroups/<resource-group>/providers/anyscale.platform/clouds/<cloud>`. Pass this full value wherever a command or compute config names the cloud, such as the `-n` flag here, `--cloud` in Step 9, and the `cloud` field in a compute config. A short name doesn't resolve. Copy the exact value from `anyscale cloud list -j` rather than typing it by hand.

The command verifies the named cloud resource and reports it as `PASSED` or `FAILED`.

Static verification confirms only that a gateway of the configured name exists. It never inspects listener hostnames. Confirm a cluster launches end to end:

```bash
anyscale cloud verify -n <cloud-name> --functional-verify workspace
```

### Step 9: Run a workload on the new cloud resource

Workloads that don't name a resource run on the cloud's primary cloud resource, so pin this one explicitly with the Anyscale CLI. `cloud_resource` is a compute config field, not a job field.

Create `job.yaml`:

```yaml
name: my-first-job
entrypoint: python main.py
working_dir: .
compute_config:
  cloud_resource: <resource-name>
  head_node:
    instance_type: <instance-type>
  worker_nodes:
  - instance_type: <instance-type>
    min_nodes: 0
    max_nodes: 4
```

```bash
anyscale job submit -f job.yaml --cloud <cloud-name>
```

#### Fall back across cloud resources

A job can list several candidate cloud resources. With `cloud_resource_strategy: input_order`, Anyscale tries them in the order given, moving to the next if the current one can't provide capacity within `cloud_resource_starting_timeout`. Each entry must name a different cloud resource.

```yaml
name: fallback-example
entrypoint: python main.py
working_dir: .
compute_config:
  cloud: <cloud-name>
  flags:
    cloud_resource_strategy: input_order
    cloud_resource_starting_timeout: 15m
  configs:
  - cloud_resource: aksuksouth
    head_node:
      instance_type: <instance-type>
  - cloud_resource: eksuswest2
    head_node:
      instance_type: <instance-type>
```

Fallback applies to jobs only. Workspaces run on a single cloud resource with no fallback, and services run on the primary cloud resource only. For which workloads run on non-primary resources, see [What is a cloud resource on Azure?](cloud-resources-overview.md)

## Change or remove a cloud resource

You can't change cloud resource properties after creation. Redeploying with a modified property fails with `Cannot update fields in the existing cloud resource <name>`. Redeploying with unchanged values succeeds and makes no change. To correct a value, delete the cloud resource and create it again.

To delete it, terminate the workloads running on that cloud resource, then delete the ARM child resource by name:

```azurecli
az resource delete \
  --ids "<cloud-arm-resource-id>/cloudResources/<resource-name>" \
  --api-version 2026-09-01
```

Other cloud resources on the same cloud aren't affected, and repeating the delete returns success. To remove the operator from the cluster afterward, see [Uninstall the Anyscale operator](https://docs.anyscale.com/clouds/kubernetes#uninstall).
