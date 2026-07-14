---
title: Reliability in Azure Storage Mover
description: Learn how Azure Storage Mover responds to a variety of potential outages and problems, including transient faults, availability zone failures, and region outages. Also describes backup options and service maintenance.
ms.author: shaas
author: stevenmatthew
ms.topic: reliability-article
ms.custom: subject-reliability
ms.service: azure-storage-mover
ms.date: 07/13/2026
---

# Reliability in Azure Storage Mover

[Azure Storage Mover](/azure/storage-mover/service-overview) is a fully managed service that migrates files and folders to Azure Storage, and keeps files synchronized across storage accounts. Use Storage Mover when you're moving data into Azure, or when you need to keep data in sync among different locations within Azure.

[!INCLUDE [Shared responsibility](includes/reliability-shared-responsibility-include.md)]

This article describes how Azure Storage Mover responds to a variety of potential outages and problems, including transient faults, availability zone failures, and region-wide failures. It also describes how to protect your Storage Mover configuration.

> [!IMPORTANT]
> This article covers the reliability of the Azure Storage Mover service and its resources only. The reliability of an end-to-end migration depends on all components: the Storage Mover service, any Storage Mover agents you deploy, the source environment and network connectivity, and the target storage account. You're responsible for the reliability of agents, source systems, and target storage. For more information about Azure Storage reliability, see [Reliability in Azure Blob Storage](reliability-storage-blob.md) and [Reliability in Azure Files](reliability-storage-files.md).

## Reliability architecture overview

[!INCLUDE [Introduction to reliability architecture overview section](includes/reliability-architecture-overview-introduction-include.md)]

### Logical architecture

Azure Storage Mover is designed to migrate and synchronize data between storage locations, not to serve requests in the runtime path of a production workload. It has a [resource hierarchy](/azure/storage-mover/resource-hierarchy) that defines the components you deploy and manage. The top-level resource is called a *storage mover*. Within a storage mover, you define *projects* that contain *job definitions* describing what to migrate and where. *Endpoints* define the source and target locations for a migration or synchronization job.

For some scenarios, such as migrations from on-premises environments, you also deploy one or more *Storage Mover agents*. An agent is software that you run on a machine you control, such as a virtual machine or physical machine. Some scenarios don't require an agent.

The service stores configuration metadata, including projects, endpoints, agent registrations, job definitions, and job run history. This metadata doesn't include the data you migrate.

### Physical architecture

The Azure Storage Mover service runs on Microsoft-managed infrastructure. Agents run on hardware you manage. You're responsible for the reliability of agents, which is out of scope for this article.

## Resilience to transient faults

[!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)]

If a transient fault affects communication between an agent and the Storage Mover service, or when connecting to a source or target, the agent retries automatically. For Azure-to-Azure jobs, the service is also resilient to many transient faults. When connectivity restores, in-progress migration jobs resume.

In some cases, transient faults appear as errors in job run history. For a description of error codes, including transient errors, see [Azure Storage Mover status codes and error types](/azure/storage-mover/status-code). For guidance on resolving persistent network connectivity issues, see [Troubleshoot Azure Storage Mover network connectivity](/azure/storage-mover/network-troubleshooting).

## Resilience to availability zone failures

[!INCLUDE [Resilience to availability zone failures](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)]

In regions that support availability zones, the platform distributes a storage mover's configuration metadata across zones on a best-effort basis, but this behavior isn't guaranteed. If your storage migrations need to withstand the loss of a zone, design your migration process to tolerate the loss of a storage mover, and review [Resilience to region-wide failures](#resilience-to-region-wide-failures).

Consider the effect of a zone failure in the context of how you use Storage Mover. The service orchestrates data migration and synchronization, and it typically isn't in the runtime path of your production workload. If a storage mover is unavailable during a zone failure, a migration or synchronization job is usually delayed rather than causing a production outage, and you can resume or retry the job after the service recovers. Storage Mover also doesn't offer an availability [service-level agreement (SLA)](#service-level-agreement), so your design shouldn't assume that the service is continuously available. If your workload depends on ongoing synchronization, evaluate whether this kind of delay is acceptable for your scenario.

The following diagram shows a storage mover with infrastructure and configuration metadata spread across three zones:

:::image type="content" source="media/reliability-storage-mover/zone-redundant.svg" alt-text="Diagram of a zone-redundant storage mover that's spread across three availability zones." border="false":::

> [!NOTE]
> The reliability of any data migration also depends on the storage accounts and agents you use. For example, if your target storage account uses locally redundant storage (LRS), it isn't resilient to a zone failure. To make a migration resilient to a zone failure, use a zone-redundant target storage account.

### Requirements

**Region support:** Best-effort distribution of configuration metadata across zones can occur only in a region that supports both Storage Mover and availability zones. Check [Storage Mover region availability](https://azure.microsoft.com/explore/global-infrastructure/products-by-region/table) and compare it with the [list of regions that support availability zones](./regions-list.md). Even in these regions, zone resilience isn't guaranteed.

### Cost

Storage Mover doesn't offer configurable availability zone support, so you don't incur any extra costs related to availability zones. For more information about how Storage Mover is billed, see [Understanding Azure Storage Mover billing](/azure/storage-mover/billing).  

### Configure availability zone support

Storage Mover doesn't offer configurable availability zone support, so there's nothing for you to enable or opt into. For more information about creating a Storage Mover resource, see [Deployment planning for Azure Storage Mover](/azure/storage-mover/deployment-planning).  

### Behavior when all zones are healthy

This section describes what to expect when a storage mover is in a region that supports availability zones and all zones are operational.

- **Cross-zone operation:** Infrastructure in any of the availability zones in the region might serve management operations and metadata access. Agent connections can reach the service through any zone.

- **Cross-zone data distribution:** The service aims to synchronously replicate configuration metadata across availability zones in the region.

### Behavior during a zone failure

This section describes what to expect when a storage mover is in a region that supports availability zones and there's an outage in one of the zones.

- **Detection and response:** The platform is designed to detect the loss of an availability zone and reroute traffic to healthy zones, but this response is best-effort and isn't guaranteed.

[!INCLUDE [Availability zone down notification (Service Health and Resource Health)](./includes/reliability-availability-zone-down-notification-service-resource-include.md)]

- **Active requests:** Management operations in progress that depend on infrastructure in the affected zone might fail, and you need to retry them. Active data migration jobs that run on agents might continue to run, but management operations that depend on the metadata service might be unavailable.

- **Expected data loss:** The data that a storage mover is migrating isn't lost during a zone failure.

    Because Storage Mover doesn't guarantee that configuration metadata is distributed across zones, a zone failure might make some of the storage mover's configuration metadata temporarily unavailable until the zone recovers.

- **Expected downtime:** The platform attempts to restore operations by using another zone, but in some situations operations might be unavailable until the affected zone recovers. Prepare your workload by following [transient fault handling guidance](#resilience-to-transient-faults).

- **Redistribution:** If the platform reroutes traffic to healthy availability zones, it does so on a best-effort basis.

### Zone recovery

When an availability zone recovers, the platform aims to restore capacity in the recovered zone and rebalance traffic between zones. This behavior is best-effort and isn't guaranteed. You don't need to take any action to initiate the zone recovery.

### Test for zone failures

You can't initiate or test an availability zone failure for a storage mover. Because Storage Mover doesn't guarantee zone resilience, don't assume that a storage mover survives a zone failure. If your migration process needs to withstand the loss of a zone, validate the end-to-end resilience of that process yourself, and review [Resilience to region-wide failures](#resilience-to-region-wide-failures) for approaches that let you control failover.

## Resilience to region-wide failures

Azure Storage Mover is a single-region service. When you deploy an Azure Storage Mover resource, you [select a region](/azure/storage-mover/deployment-planning#select-an-azure-region-for-your-deployment) to store the resource's configuration metadata. If the storage mover's region experiences an outage, management operations that the agent performs and that rely on Azure might not complete. In addition, any active data migrations to storage accounts located within the affected region might fail.

If your storage mover is in an Azure region with a pair, its configuration metadata is [replicated to the paired Azure region](#microsoft-managed-failover-to-a-paired-region) for disaster recovery purposes, and Microsoft can trigger failover to the paired region during a disaster affecting your primary region.

If your storage mover is in a *nonpaired region*, Microsoft doesn't replicate configuration metadata, and there's no built-in failover to another region. However, you can deploy separate resources into multiple regions. In this scenario, it's your responsibility to manage replication, traffic distribution, and failover. If you use a nonpaired region, or if the built-in metadata replication doesn't meet your needs, you can create a [custom multi-region failover strategy](#custom-multi-region-solutions-for-resiliency).

> [!NOTE]
> You're responsible for disaster recovery for your data sources (including Azure and on-premises data sources), targets, and agents.

### Microsoft-managed failover to a paired region

If your Storage Mover resource is in a [region that pairs with another region](./regions-paired.md), Microsoft replicates your storage mover's configuration metadata to the paired region.

:::image type="content" source="media/reliability-storage-mover/multi-region-replication.svg" alt-text="Diagram of a storage mover replicating its metadata to a paired region." border="false":::

In the event of a region outage, Microsoft might perform a failover to the paired region by using the replicated configuration metadata. This process is a default option and requires no intervention from you.

:::image type="content" source="media/reliability-storage-mover/multi-region-failover.svg" alt-text="Diagram of a storage mover failing over to a paired region." border="false":::

Failover of Storage Mover resources might happen at a different time than any failover of other Azure services.

> [!IMPORTANT]
> Microsoft is unlikely to initiate failover except after a significant delay and is done on a best-effort basis. If you need to meet specific timeframes for Storage Mover's recovery, or if the default replication and failover behavior doesn't meet your needs, use [custom multi-region solutions for resiliency](#custom-multi-region-solutions-for-resiliency) to plan for and initiate your own failover.

Cross-region replication applies to configuration metadata only. It doesn't apply to the source data or to the target storage account, which has its own reliability and replication options. For more information, see [Reliability in Azure Blob Storage](reliability-storage-blob.md) and [Reliability in Azure Files](reliability-storage-files.md).

#### Requirements

**Region support:** Microsoft-managed cross-region replication is available only for Storage Mover resources that you deploy into a region that has a [paired region](regions-paired.md). For resources in nonpaired regions, no cross-region replication or failover is provided. To achieve cross-region resilience in nonpaired regions, use a [custom multi-region solution](#custom-multi-region-solutions-for-resiliency).

#### Cost

Storage Mover doesn't charge for Microsoft-managed cross-region replication of your storage mover's configuration. However, there might be a small charge for cross-region replication. For more information, see [Bandwidth pricing](https://azure.microsoft.com/pricing/details/bandwidth/).

#### Configure multi-region support

Microsoft-managed cross-region replication is automatically enabled for Storage Mover resources in paired regions. You don't configure or opt into this behavior.

#### Behavior when all regions are healthy

This section describes what to expect when a storage mover is configured for cross-region replication and failover, and the primary region is operational.

- **Cross-region operation:** Your Storage Mover resource in the primary region serves all requests. The paired region is used only in the event of a Microsoft-initiated failover.

- **Cross-region data replication:** The primary region replicates configuration asynchronously to the paired region. Because replication is asynchronous, recent changes to configuration might not be reflected in the paired region at the time of a failure.

#### Behavior during a region failure

This section describes what to expect when a storage mover is configured for cross-region replication and failover, and there's an outage in the primary region.

- **Detection and response:** Microsoft detects region failures and decides whether to initiate a failover. Microsoft is unlikely to initiate failover except after a significant delay, and failover is done on a best-effort basis.

[!INCLUDE [Region down notification (Service Health and Resource Health)](./includes/reliability-region-down-notification-service-resource-include.md)]

- **Active requests:** Active management requests are dropped and need to be retried after failover completes. Active data migration jobs running on agents might fail if they depend on the region that's experiencing the outage.

- **Expected data loss:** Because cross-region replication is asynchronous, any configuration metadata changes that aren't replicated to the paired region at the time of the outage might be lost.

- **Expected downtime:** A region failover might take up to 24 hours to complete. During this time, your storage mover is unavailable.

- **Redistribution:** After failover completes, Storage Mover begins to run jobs from the paired region.

    However, you need to re-register agents against the storage mover in the paired region.

#### Region recovery

When the original primary region recovers, Microsoft coordinates failback. You need to re-register agents against the storage mover in the primary region.

#### Test for region failures

The Azure Storage Mover platform manages cross-region replication, failover, and region recovery. Because Microsoft fully manages this feature, you can't initiate or test a region failover.

### Custom multi-region solutions for resiliency

If you need to control when failover occurs, or if you're in a nonpaired region but still need your storage mover to be resilient to region outages, deploy independent Storage Mover resources in multiple Azure regions. You're responsible for all aspects of this approach, including:

- Creating and maintaining equivalent projects, endpoints, agents, and job definitions in each region.
- Detecting region failures and deciding when to fail over.
- Redirecting agents and migration jobs to the secondary region.
- Reconciling job state and run history between regions.

A custom multiregion solution works for both paired and nonpaired regions, and gives you full control over your failover process.

For more information, see [Customer-initiated disaster recovery for Azure Storage Mover](/azure/storage-mover/customer-initiated-disaster-recovery).

## Backup and restore

Azure Storage Mover is a migration and data movement orchestration service. It doesn't store the data you migrate. The service stores only configuration metadata, such as projects, endpoints, job definitions, and job run history. There's no migration data to back up.

To protect your Storage Mover configuration, define your resources using infrastructure as code, such as Bicep files, and store those definitions in source control. If you need to recreate a resource, you can redeploy it from your stored configuration.

[!INCLUDE [Backups include](includes/reliability-backups-include.md)]

## Resilience to service maintenance

[!INCLUDE [Service maintenance (no special callouts)](includes/reliability-maintenance-include.md)]

Storage Mover agents are automatically upgraded.

## Service-level agreement

Storage Mover is a migration service and doesn't offer an availability service-level agreement (SLA). However, the [Storage Mover documentation describes expected scale and performance targets](/azure/storage-mover/performance-targets). These targets are based on simulated migrations, and aren't a guarantee or commitment.

## Related content

- [Reliability in Azure](./overview.md)
- [Azure Storage Mover deployment planning](/azure/storage-mover/deployment-planning)
- [Azure Storage redundancy](/azure/storage/common/storage-redundancy)
