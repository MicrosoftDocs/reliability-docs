---
title: Reliability in Azure Container Apps Sandboxes
description: Learn how Azure Container Apps Sandboxes responds to transient faults, availability zone failures, region-wide failures, and service maintenance, with backup and restore guidance.
author: glynnniall
ms.author: glynnniall
ms.topic: reliability-article
ms.custom: subject-reliability
ms.service: azure-container-apps
ms.date: 09/15/2026
---

# Reliability in Azure Container Apps Sandboxes (preview)

[Azure Container Apps Sandboxes](/azure/container-apps/sandboxes-overview) provide isolated environments for running code. Each sandbox runs in a lightweight virtual machine (microVM) that starts in less than a second and can preserve its in-memory state when suspended. The service supports the reliability of sandbox workloads through capabilities you configure and capabilities the platform manages on your behalf.

[!INCLUDE [Shared responsibility](includes/reliability-shared-responsibility-include.md)]

This article describes how to make Container Apps Sandboxes resilient to transient faults, availability zone failures, region-wide failures, and service maintenance. It also describes backup and restore options and key information about the service-level agreement (SLA).

> [!IMPORTANT]
> Container Apps Sandboxes is currently in PREVIEW.
> See the [Supplemental Terms of Use for Microsoft Azure Previews](https://azure.microsoft.com/support/legal/preview-supplemental-terms/) for legal terms that apply to Azure features that are in beta, preview, or otherwise not yet released into general availability.

## Production deployment recommendations for reliability

For production workloads, we recommend that you:

> [!div class="checklist"]
>
> - **Store durable data outside sandbox memory and choose a storage redundancy option that matches your recovery goals.** Use sandbox volumes for data that must persist when a sandbox stops. For resilience to a region-wide failure, use an external data store that replicates data to another region.
>
>   When you use Azure Blob Storage, geo-redundant storage (GRS) replicates data to a paired region. For nonpaired regions, deploy separate storage accounts and configure a supported replication method, such as object replication for block blobs. For more information, see [Custom multiregion solutions for Azure Blob Storage](reliability-storage-blob.md#custom-multiregion-solutions-for-resiliency).
>
> - **Deploy separate sandbox groups across multiple regions** if your uptime target can't be met by a single-region deployment. For more information, see [Resilience to region-wide failures](#resilience-to-region-wide-failures).

## Reliability architecture overview

[!INCLUDE [Introduction to reliability architecture overview section](includes/reliability-architecture-overview-introduction-include.md)]

### Logical architecture

Azure Container Apps provides distinct compute options for apps, jobs, dynamic sessions, and sandboxes. Sandbox groups don't require a Container Apps environment. For details on the reliability of other Container Apps components, see [Reliability in Azure Container Apps](reliability-container-apps.md).

The main resources in Container Apps Sandboxes are:

- **Sandbox group:** A *sandbox group* is the top-level regional management boundary for sandboxes and uses the `Microsoft.App/sandboxGroups` resource type. All sandboxes, disk images, snapshots, volumes, and sensitive configuration values (*secrets*) are scoped to a sandbox group.

- **Sandbox:** Each *sandbox* is a lightweight, isolated microVM that runs from a disk image or snapshot and has its own CPU, memory, local disk, and network boundary.

  A *disk image* is an Open Container Initiative (OCI) container image that's converted for use as a sandbox root filesystem.

  A *snapshot* is a point-in-time capture of a sandbox's complete state that persists independently of the source sandbox.

  A sandbox's state can either be *running* or *stopped*. When a sandbox stops, either automatically or on request, it releases its compute resources. *Memory mode* preserves the sandbox's full memory image and local disk. *Disk mode* preserves only the local disk, so the microVM and its processes restart when you resume the sandbox.

- **Volumes:** The local disk belongs to an individual sandbox. A sandbox *volume* provides persistent storage that exists independently of an individual sandbox. You can mount Azure Blob Storage volumes into multiple sandboxes at the same time, while data disk volumes backed by Azure Disk Storage can mount into only one sandbox at a time. The backing storage service determines the durability and recovery options for volume data.

For more information about sandbox architecture and resources, see [Azure Container Apps Sandboxes overview](/azure/container-apps/sandboxes-overview).

### Physical architecture

Sandboxes run on multiple independent compute *clusters* that Microsoft operates. You're responsible for configuring the sandbox groups, sandboxes, and other resources that you deploy. Microsoft is responsible for cluster deployment, configuration, capacity management, health monitoring, and maintenance. You don't select, deploy, configure, or manage the clusters. The service schedules new sandboxes, and restarts stopped sandboxes, on healthy clusters and routes placements around unhealthy clusters.

Microsoft maintains redundant state stores for service configuration, sandbox metadata and artifacts such as disk images and snapshots.

## Resilience to transient faults

[!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)]

When you use Container Apps Sandboxes, consider transient faults in the following parts of your solution:

- **Sandbox management operations:** When your automation manages sandbox groups, sandboxes, or related resources, retry requests that fail because of transient faults, and use exponential backoff. Limit the number of retry attempts, and retry only operations that are safe to repeat.

- **Code running in a sandbox:** Implement transient fault handling for calls to external APIs, databases, and other services. Follow the retry guidance for each dependency because retry behavior and operations that are safe to repeat vary by service.

## Resilience to availability zone failures

Container Apps Sandboxes don't support deployment to a specific availability zone or zone redundancy for a sandbox group. To make your workload resilient to availability zone failures, deploy separate sandbox groups in multiple regions. For more information, see [Resilience to region-wide failures](#resilience-to-region-wide-failures).

## Resilience to region-wide failures

Container Apps Sandboxes is a single-region service. If the region becomes unavailable, your sandbox groups and the sandboxes they contain are also unavailable. The service doesn't replicate sandbox groups or sandboxes across regions, and it doesn't automatically fail over to another region. However, you can deploy separate sandbox groups in multiple regions. You're responsible for making dependencies available in each region and managing workload distribution and failover. For more information, see [Custom multiregion solutions for resiliency](#custom-multiregion-solutions-for-resiliency).

During a region-wide failure, you might lose any state held only in the memory of a running sandbox. Sandbox groups, sandboxes, and service-managed artifacts in the affected region remain unavailable until the region recovers.

Sandbox volumes provide storage that persists beyond the lifecycle of an individual sandbox. During a region-wide failure, volume availability and recovery depend on the backing storage service and its configuration. Container Apps Sandboxes don't provide cross-region replication or failover for volume data. Instead, the backing storage service provides those capabilities when configured. For example, for information about Azure Blob Storage volumes, see [Reliability in Azure Blob Storage](reliability-storage-blob.md).

### Custom multiregion solutions for resiliency

Azure Container Apps Sandboxes doesn't coordinate multiregion deployments or replicate sandbox groups, sandboxes, or their related resources between regions. To create a custom multiregion solution, you have the following responsibilities:

- **Regional deployments and dependencies:** Deploy a separate sandbox group in each region that you plan to use. Keep configuration, disk images, secrets, and other dependencies available in each region.

- **Failure detection and workload recovery:** Configure your application or orchestration layer to detect when a region is unavailable, direct new sandbox creation and workload processing to a healthy region, and determine how to restart interrupted work.

- **Traffic routing:** If clients connect through region-specific endpoints that your application exposes, use a global load-balancing service, such as Azure Front Door or Azure Traffic Manager, to route traffic to a healthy endpoint.

- **Data replication and recovery:** Store any state required after failover in an external data store that supports cross-region replication and recovery. If a backing storage service provides cross-region replication for volume data, that service determines the replication and failover behavior. Azure Container Apps Sandboxes doesn't replicate or fail over volume data between regions.

## Backup and restore

Don't use sandbox memory or local disk as your only durable data store. Suspending a sandbox preserves its local disk and, in memory mode, its memory state. You can also create snapshots that persist independently of the source sandbox. Suspended state and snapshots remain scoped to the regional sandbox group and aren't cross-region backups.

Use a sandbox volume for data that must persist beyond the lifecycle of an individual sandbox. The backing storage service and its configuration determine the backup and restore capabilities for volume data. For external data stores that you manage, you're responsible for configuring backup and cross-region recovery to meet your durability and recovery objectives.

To recreate your sandbox deployment after accidental deletion or a region-wide failure, store your sandbox group configuration in version-controlled infrastructure-as-code templates, such as Bicep or Terraform. Keep your source disk images in a registry that meets your recovery requirements.

[!INCLUDE [Backups description](includes/reliability-backups-include.md)]

## Resilience to service maintenance

[!INCLUDE [Service maintenance (transient fault handling)](includes/reliability-maintenance-transient-fault-include.md)]

When maintenance affects a running sandbox, the platform preserves its state, moves it to healthy compute capacity, and resumes it automatically. For sandboxes that use *memory mode*, the platform preserves memory and local disk state. For sandboxes that use *disk mode*, the platform preserves local disk state only.

## Service-level agreement

Azure Container Apps Sandboxes doesn't offer an availability service-level agreement (SLA). Storage services that back your sandbox volumes and external data stores used by your solution might have separate SLAs. For more information, see [Service Level Agreements for Online Services](https://aka.ms/csla).

## Related content

- [Reliability in Azure](overview.md)
- [Azure Container Apps Sandboxes overview](/azure/container-apps/sandboxes-overview)
- [Reliability in Azure Container Apps](reliability-container-apps.md)
- [Recommendations for handling transient faults](/azure/well-architected/reliability/handle-transient-faults)
- [Well-Architected Framework reliability guidance](/azure/well-architected/reliability/)
