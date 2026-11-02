---
title: Create workloads through Azure Resource Manager
description: Create Anyscale workspaces, jobs, and services on Azure through Azure Resource Manager, including how to reference compute configs and cloud resources by resource ID.
author: kaysieyu
ms.author: kaysieyu
ms.reviewer: mbender
ms.date: 10/05/2026
ms.service: azure-kubernetes-service
ms.topic: how-to
---

# Create workloads through Azure Resource Manager

<!-- DJS 18 Sep 2026: Draft from the GA spec review (DOC-1690). Grounded in the product repo GA spec at backend/azure/specifications/anyscale/resource-manager/Anyscale.Platform/stable/2026-09-01/openapi.json and its canonical example examples/Jobs_CreateOrUpdate.json. arm-id typing of computeConfigName/cloudResourceNames/acrResourceId and the x-ms-mutability write-only behavior of the five job fields are confirmed in that spec. SME to confirm before merge: (1) computeConfig with computeConfigName only (no inline spec) is a valid job body — the schema is ComputeConfigSpecOrReference, but the canonical example sends both; (2) rollout wording — the 2026-09-01 registration manifests are still feature-gated (Anyscale.Platform/DefaultFeature, readiness "privatepreview") in source, so this describes API behavior, not GA availability. -->

You can create Anyscale workspaces, jobs, and services on Azure through Azure Resource Manager (ARM), by using an ARM template, the Azure CLI, or any Azure SDK. This article describes the request shape for a job and the two behaviors that most often cause confusion: how you reference a compute config and cloud resources, and which job fields ARM accepts but never returns.

To create and run workloads interactively, use the Anyscale console, the Anyscale CLI, or the Anyscale SDK. See [Anyscale jobs](https://docs.anyscale.com/platform/jobs/) in the Anyscale documentation.

## Prerequisites

- An existing Anyscale cloud on Azure with at least one cloud resource. To create one, see the [Quickstart](quickstart-azure-cli.md).
- An Anyscale project to hold the workload. Jobs, services, and workspaces are children of a project.
- Azure CLI installed and authenticated (`az login`), or an equivalent ARM client.
- The **Anyscale Platform Contributor** role on the cloud resource. For role details, see [Identity and access](identity-access.md).

## Reference compute configs and cloud resources by resource ID

At API version `2026-09-01`, a workload references its compute config and cloud resources by full ARM resource ID, not by name. This change applies to three fields:

- `computeConfigName`, on a workspace, job, or service.
- `cloudResourceNames`, inside an inline compute config spec.
- `acrResourceId`, on a cloud. For that field, see [Configure container image builds](configure-container-image-builds.md).

Each of these fields keeps the word "Name," but its value is a resource ID. A bare name returns HTTP 400 with the message `Object didn't pass validation for format arm-id`. A compute config resource ID has this form, and it must point at the same cloud as the workload:

```output
/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Anyscale.Platform/clouds/<cloud-name>/computeConfigs/<compute-config-name>
```

> [!IMPORTANT]
> An ARM template or SDK call written against a preview API version that passes a bare compute config name fails when it moves to `2026-09-01`. Update these fields to full resource IDs before you move a template to the current API version.

## Create a job

Send a PUT request to the job resource path. The job's `config` is the only required property. The following example references a predefined compute config by resource ID:

```azurecli
az rest \
  --method PUT \
  --url "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Anyscale.Platform/clouds/<cloud-name>/projects/<project-name>/jobs/<job-name>?api-version=2026-09-01" \
  --body '{
    "properties": {
      "config": {
        "imageUri": "anyscale/ray:2.44.0-py312-cu124",
        "computeConfig": {
          "computeConfigName": "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Anyscale.Platform/clouds/<cloud-name>/computeConfigs/<compute-config-name>"
        },
        "entrypoint": "python train.py --epochs 10 --batch-size 32",
        "workingDir": "https://<account>.blob.core.windows.net/<container>/<path>",
        "maxRetries": 3,
        "timeout": 1380
      }
    }
  }'
```

To define compute inline instead of referencing a predefined config, replace `computeConfigName` with a `spec` object. Inside the spec, `cloudResourceNames` takes the resource IDs of the cloud resources the workload runs on.

## Job fields that ARM accepts but doesn't return

At API version `2026-09-01`, five job fields are write-only. ARM accepts these fields when you create or update a job, but it never returns them when you read a job:

- `entrypoint`
- `workingDir`
- `pyModules`
- `pipRequirements`
- `envVars`

When you read the job, either by using a GET request or in the create response, these five fields are absent. ARM returns no error and no warning. A read operation shows only `imageUri`, `computeConfig`, `maxRetries`, and `timeout`. To confirm the full configuration of a job, use the Anyscale console or CLI, which return all fields.

> [!NOTE]
> Treat any example that shows these five fields as a request body. No ARM read returns them.

## Create workspaces and services

Workspaces and services follow the same pattern. Send a PUT request to the resource path, and reference compute configs and cloud resources by resource ID:

- Workspace: `.../clouds/<cloud-name>/projects/<project-name>/workspaces/<workspace-name>`
- Service: `.../clouds/<cloud-name>/projects/<project-name>/services/<service-name>`

A service defines its rollout under `properties.deploymentConfig`, including its applications, image, compute config, and rollout strategy.

## Next steps

- [Identity and access](identity-access.md) for the roles and resource-provider operations that ARM requests require.
- [Configure container image builds](configure-container-image-builds.md) for setting `acrResourceId` on a cloud.
- [Architecture overview](architecture.md) for how the control plane and data plane interact.
</content>
</invoke>
