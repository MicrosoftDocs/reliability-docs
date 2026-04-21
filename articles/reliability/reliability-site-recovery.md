---
title: Reliability in Azure Site Recovery
description: Learn how to make Azure Site Recovery resilient to a variety of potential outages and problems, including transient faults, availability zone outages, and region outages.
author: glynnniall
ms.author: pnp
ms.topic: reliability-article
ms.custom: subject-reliability, references_regions
ms.service: azure-site-recovery
ms.date: 03/03/2026
ai-usage: ai-assisted
---

# Reliability in Azure Site Recovery

[Azure Site Recovery](/azure/site-recovery/site-recovery-overview) is a managed replication and failover service for virtual machines (VMs) that keeps workloads available during outages. It continuously replicates workloads from primary sites to secondary locations and limits data loss and downtime. During planned maintenance or unexpected disruptions, it orchestrates failover and failback. This service supports disaster recovery (DR) for on-premises environments and Azure VMs, which helps organizations maintain business continuity.

[!INCLUDE [Shared responsibility](includes/reliability-shared-responsibility-include.md)]

This article describes how to make Site Recovery resilient to various potential outages and problems, including transient faults, availability zone outages, and region outages. It also highlights key information about the Site Recovery service-level agreement (SLA).

> [!NOTE]
> This article describes how the core Site Recovery service is resilient, or how you can make it resilient, to various problems. It doesn't explain how to use Site Recovery to protect your VMs or other assets. For more information, see [About Site Recovery](/azure/site-recovery/site-recovery-overview).

## Production deployment recommendations for reliability

When you use Site Recovery with production workloads, we recommend that you take these actions:

> [!div class="checklist"]
> - Deploy your Recovery Services vault in your target region for replication.
>
> - For Azure-to-Azure DR, use the Site Recovery [high churn](/azure/site-recovery/concepts-azure-to-azure-high-churn-support) feature for VMs that have a high rate of data change. High churn support improves your recovery point objective (RPO) and enables replication for many high-scale database workloads.
>
> - For Azure-to-Azure DR, configure the cache storage account to use zone-redundant storage (ZRS).
>
> - Perform test failovers on a regular basis as part of DR drills. Run DR drills every quarter or biannually to verify that your replication and failover processes are healthy.
>
> - Use [on-demand capacity reservations](/azure/virtual-machines/capacity-reservation-overview) to ensure that compute resources are available in your target region for failover.
>
> - Enable automatic updates for mobility agents.
>
> - Monitor the health of your replication, and configure alerts so that you're notified if a problem occurs.

## Reliability architecture overview

When you use Site Recovery, you define a *source* and *target*, which represent the replicated VMs:

- The *source* can be an Azure VM or a VM or server from another supported source, including on-premises physical servers, VMware VMs, and Hyper-V VMs.

- The *target* is always an Azure VM. For Azure-to-Azure VM replication, the target can be a different region or availability zone from the source VM.

You're responsible for deploying and configuring resources and related settings, including:

- *Recovery Services vault*, which Site Recovery uses to store your replication configuration settings. The vault doesn't store your replicated data. The redundancy configuration of the vault isn't important for Site Recovery, but it's important if you use the same vault for Azure Backup.

    A vault can include extra configuration, such as:

    - *A replication policy*, which configures the snapshot frequency and retention length.

    - *[A recovery plan](/azure/site-recovery/recovery-plan-overview)*, which coordinates the order in which machines fail over and can include scripts and manual actions. Recovery plans are especially useful for workloads that have multiple tiers, such as application and database tiers, that need to fail over in a specific order.

- For Azure-to-Azure replication, a *cache storage account* that stores a copy of the source data in its region before it's replicated to the target. The redundancy configuration of your cache storage account can affect your reliability during an availability zone outage.

:::image type="complex" border="false" source="media/reliability-site-recovery/recovery-vault-storage.svg" alt-text="Diagram that shows the relationship between the Recovery Services vault, cache storage account, source, and target in Site Recovery.":::
   The diagram shows three availability zones. Zone 1 includes a VM. The following sections span all three zones: Site Recovery core components, the Recovery Services vault, and the cache storage account for ZRS.
:::image-end:::

> [!NOTE]
> This guide focuses on the reliability of the Azure-based components of Site Recovery and the replication relationship. If you replicate data or VMs from an on-premises environment or another cloud provider, consider the reliability of the components outside of Azure.

For more information about the components that you deploy, see the following articles:

- [Azure-to-Azure DR architecture](/azure/site-recovery/azure-to-azure-architecture)
- [Hyper-V-to-Azure DR architecture](/azure/site-recovery/hyper-v-azure-architecture)
- [VMware-to-Azure DR architecture](/azure/site-recovery/vmware-azure-architecture-modernized)
- [Physical-server-to-Azure DR architecture](/azure/site-recovery/physical-server-azure-architecture-modernized)

The core Site Recovery service runs on infrastructure that Microsoft manages. This article refers to these components collectively as the *core Site Recovery service*.

## Resilience to transient faults

[!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)]

Site Recovery automatically handles transient faults that occur during the replication process by retrying its operations. You don't need to configure transient fault handling for Site Recovery.

## Resilience to availability zone failures

[!INCLUDE [Resilience to availability zone failures](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)]

To understand how Site Recovery replication behaves during availability zone failures, you need to consider the following service components:

- **Core Site Recovery service:** The core Site Recovery service is designed to be resilient to availability zone failures in supported regions. The internal components of the service support zone redundancy automatically with no customer configuration required.

- **Recovery Services vault:** The vault stores configuration data. In regions where Site Recovery supports zone resilience, configuration data in the vault is also zone resilient.

- **Cache storage account:** For Azure-to-Azure replication, you're responsible for making the cache storage account zone redundant by deploying it using the ZRS tier.

    If you use the locally redundant storage (LRS) Azure Storage replication tier for your cache storage account and a zone fails, Site Recovery might not replicate recently changed data to your target.

> [!NOTE]
> Site Recovery can help you fail over between VMs in different availability zones. For more information, see [Enable Azure VM DR between availability zones](/azure/site-recovery/azure-to-azure-how-to-enable-zone-to-zone-disaster-recovery).

### Requirements

**Region support:**

- **Core Site Recovery service and Recovery Services vaults:** Site Recovery is zone resilient in the following regions.

    | Americas       | Europe         | Middle East    | Asia Pacific      |
    |----------------|----------------|----------------|-------------------|
    | Chile Central  | Austria East   | Israel Central | Indonesia Central |
    | Mexico Central | Italy North    |                | Japan West        |
    | West US 3      | Poland Central |                | Malaysia West     |
    |                | Spain Central  |                | New Zealand North |

     Site Recovery deploys support for availability zones in [all availability zone-enabled regions](./regions-list.md). In regions that aren't zone resilient, zone failures might affect operations.

- **Cache storage account:** You can deploy a ZRS storage account in all availability zone-enabled regions.

### Cost

Site Recovery is billed based on the number of VM instances protected, regardless of their availability zone configuration. For more information, see [Site Recovery pricing](https://azure.microsoft.com/pricing/details/site-recovery/).

### Configure availability zone support

- **Core Site Recovery service:** You don't configure zone resiliency on the core Site Recovery service. Microsoft provides zone resiliency in supported regions.

    If Microsoft enables zone resiliency in a region at a later time, your Site Recovery resources automatically benefit from zone resilience. You don't need to take any action.

- **Recovery Services vault:** Recovery Services vaults let you configure a level of redundancy, but Site Recovery doesn't use this configuration setting. You don't need to configure your vault for zone redundancy when you use Site Recovery.

- **Cache storage account:** When you use Azure-to-Azure replication, you're responsible for creating the cache storage account and for configuring it with the appropriate level of redundancy. To make it zone redundant, configure it for the ZRS replication type. For more information, see [Reliability in Azure Blob Storage](./reliability-storage-blob.md).

### Behavior when all zones are healthy

This section describes what to expect when you use Site Recovery in a region with availability zone support for the core service, your cache storage account is configured to use ZRS, and all availability zones are operational.

- **Cross-zone operation:** The replication process can use infrastructure in multiple availability zones to trigger and run replication jobs. The service manages this infrastructure transparently to you.

- **Cross-zone data replication:** Site Recovery and Storage handle zone data replication:

    - *Site Recovery configuration:* Site Recovery replicates your configuration data across zones even if you configure your vault to use LRS.

    - *Cache storage account:* If you configure your cache storage account to use ZRS, Storage synchronously replicates the cached data between zones.

### Behavior during a zone failure

This section describes what to expect when you use Site Recovery in a region with availability zone support for the core service, your cache storage account is configured to use ZRS, and an availability zone outage occurs.

> [!NOTE]
> If the failed zone contains the source VM, you're responsible for triggering failover to the target. For more information, see the following articles:
> - [Fail over Azure VMs to a secondary region](/azure/site-recovery/azure-to-azure-tutorial-failover-failback)
> - [Fail over VMware VMs](/azure/site-recovery/vmware-azure-tutorial-failover-failback-modernized)
> - [Fail over Hyper-V VMs to Azure](/azure/site-recovery/hyper-v-azure-failover-failback-tutorial)

- **Detection and response:** The Site Recovery platform automatically detects failures in an availability zone and initiates a response. You don't need to manually initiate a zone failover for the core Site Recovery service. However, if the zone outage affects your source VM, you might need to [initiate failover of your VM](/azure/site-recovery/azure-to-azure-tutorial-failover-failback).

[!INCLUDE [Availability zone down notification (Service Health only)](./includes/reliability-availability-zone-down-notification-service-include.md)]

- **Active requests:** The effect on active replication jobs depends on the replication type:

    - *Zone-to-zone and region-to-region replication of Azure VMs:* If either the source or target instance is in the failed zone, replication stops until both instances are available again.

        If the failed zone doesn't contain the source or target VM, and you configure the cache storage account to use ZRS, replication continues to run.

    - *On-premises to Azure:* If the target instance is in the failed zone, replication stops until the instance is available again.

        If the failed zone doesn't contain the target VM, replication continues to run.

- **Expected data loss:** No data loss is expected during a zone failure.

- **Expected downtime:** If the failed zone contains either the source or target VM, replication stops until both instances are available again.

- **Redistribution:** Site Recovery and Storage automatically adapt to zone failures:

    - *Core Site Recovery service:* The core Site Recovery service automatically uses infrastructure in healthy availability zones to perform replication. You don't need to take any action.

    - *Cache storage account:* Storage automatically routes requests for cache data to healthy zones.

### Zone recovery

When the affected availability zone recovers, Site Recovery automatically resumes replication jobs that stopped during the zone outage.

You're responsible for initiating failback for servers or VMs that you failed over during the zone outage. For more information, see the following articles:

- *Zone-to-zone and region-to-region replication of Azure VMs:* [Fail back an Azure VM to the primary region](/azure/site-recovery/azure-to-azure-tutorial-failback)

- *On-premises-to-Azure replication:*

    - *Physical-to-Azure replication:* [Physical-server-to-Azure DR architecture](/azure/site-recovery/physical-server-azure-architecture-modernized)

    - *Hyper-V-to-Azure replication:* [Hyper-V-to-Azure DR architecture](/azure/site-recovery/hyper-v-azure-architecture#failover-and-failback-process)

    - *VMware-to-Azure replication:* [About on-premises DR failover and failback](/azure/site-recovery/failover-failback-overview-modernized)

### Test for zone failures

The Site Recovery platform manages zone resiliency for its internal components. This feature is fully managed, so you don't need to initiate or validate availability zone failure processes.

It's important to perform regular DR drills, which should test your VM failover and your overall response procedures. Design your DR drills to avoid impact to your production environment. For more information, see the following articles:

- *Zone-to-zone and region-to-region replication of Azure VMs:* [Run a DR drill for Azure VMs](/azure/site-recovery/azure-to-azure-tutorial-dr-drill)

- *On-premises-to-Azure replication:*

    - *Physical-to-Azure replication:* [Run a DR drill to Azure](/azure/site-recovery/site-recovery-test-failover-to-azure)

    - *Hyper-V-to-Azure replication:* [Run a DR drill to Azure](/azure/site-recovery/tutorial-dr-drill-azure)

    - *VMware-to-Azure replication:* [Run a DR drill to Azure](/azure/site-recovery/tutorial-dr-drill-azure)

## Resilience to region-wide failures

For Azure-to-Azure replication, Site Recovery provides resilience to region failures by enabling failover of VMs to a healthy target region. For more information, see [Replicate Azure VMs to another Azure region](/azure/site-recovery/azure-to-azure-how-to-enable-replication).

### Considerations

- **Vault region:** You deploy a Recovery Services vault into a specific Azure region that you select. The vault's region is important. Replication continues during an outage in the vault's region. However, you can't perform Site Recovery management operations, including failover and failback, until the region recovers.

    Deploying the vault in the target region helps ensure that failover and recovery operations remain available during a source-region outage. It also prevents an outage in a third region from affecting failover and recovery operations.

    > [!NOTE]
    > If your vault is in the region that you typically use as your target region, then after you fail over and reestablish replication, that region becomes your new source region. If that region subsequently experiences a problem, you might not be able to perform failback until both regions are healthy.

- **Capacity reservations:** You're responsible for verifying that your target region supports the VM types that you need and that it has available capacity for your workload. We recommend that you use [on-demand capacity reservations](/azure/virtual-machines/capacity-reservation-overview) to ensure that compute resources are available for your workload if a failover occurs.

### Configure multi-region support

- **Recovery Services vault:** You need to select the vault's region. For more information, see [Considerations](#considerations).

    Recovery Services vaults let you configure a level of redundancy, but Site Recovery doesn't use this configuration setting. You don't need to configure your vault for geo-redundancy when you use Site Recovery.

- **Cache storage account:** The cache storage account is only used as a temporary location for data before it's replicated, so you shouldn't configure it to use geo-redundant storage (GRS).

### Behavior during a region failure

The specific behavior of the Site Recovery core service during a region failure depends on which region experiences the failure:

- **Failure in the source region:** For Azure-to-Azure replication, you can trigger a failover when the source region is unavailable.

    Because the source region is unavailable, replication stops until the VM in the source region is healthy.

    :::image type="complex" border="false" source="media/reliability-site-recovery/site-recovery-denied.svg" alt-text="Diagram that shows the behavior when the source region fails. The source is unavailable, and replication stops until the source region recovers.":::
      The diagram shows the source region and the target region. Two failures are shown in the source VM. An arrow labeled Site Recovery replication points to the target region. The target region includes the target VM and the Recovery Services vault.
    :::image-end:::

- **Failure in the target region:** Because the target region is unavailable, replication stops, and you can't fail over to the target until the region is healthy.

    :::image type="complex" border="false" source="media/reliability-site-recovery/source-available-site-recovery-denied.svg" alt-text="Diagram that shows the behavior when the target region fails. Replication stops, and failover is unavailable until the target region recovers.":::
      The diagram shows the source region and the target region. The source region contains the source VM. An arrow labeled Site Recovery replication points to the target region. An X indicates a replication failure. The target region includes the target VM and the Recovery Services vault. Failures are indicated in the target VM and Recovery Services vault.
    :::image-end:::

- **Failure in the region that contains the vault:** If you deploy the vault into a third region (not the source or target region) and that region experiences a failure, Site Recovery continues to replicate your data. However, you can't initiate any operations, including failover or failback, until the vault is healthy.

    :::image type="complex" border="false" source="media/reliability-site-recovery/replication-available-failover-denied.svg" alt-text="Diagram that shows the behavior when the vault region fails. Replication continues, but failover and failback operations are unavailable until the vault region recovers.":::
      The diagram shows the source region, target region, and the vault region. An arrow labeled Site Recovery replication to points from the source VM to the VM in the target region. A failure is indicated in the Recovery Services vault. An arrow labeled failover, failback, and other operations blocked but replication continues points from the Services Recovery vault to Site Recovery replication.
    :::image-end:::

### Region recovery

You're responsible for initiating failback for servers or VMs that you failed over during the region outage. For more information, see the following articles:

- *Zone-to-zone and region-to-region replication of Azure VMs:* [Fail back Azure VM to the primary region](/azure/site-recovery/azure-to-azure-tutorial-failback)

- *On-premises-to-Azure replication:*

    - *Physical-to-Azure replication:* [Physical-server-to-Azure DR architecture](/azure/site-recovery/physical-server-azure-architecture-modernized)

    - *Hyper-V-to-Azure replication:* [Hyper-V-to-Azure DR architecture](/azure/site-recovery/hyper-v-azure-architecture#failover-and-failback-process)

    - *VMware-to-Azure replication:* [On-premises DR failover and failback](/azure/site-recovery/failover-failback-overview-modernized)

### Test for region failures

It's important to perform regular DR drills that test your VM failover and overall response procedures. Design your DR drills to prevent impact to your production environment. For more information, see the following articles:

- *Zone-to-zone and region-to-region replication of Azure VMs:* [Run a DR drill for Azure VMs](/azure/site-recovery/azure-to-azure-tutorial-dr-drill)

- *On-premises-to-Azure replication:*

    - *Physical-to-Azure replication:* [Run a DR drill to Azure](/azure/site-recovery/site-recovery-test-failover-to-azure)

    - *Hyper-V-to-Azure replication:* [Run a DR drill to Azure](/azure/site-recovery/tutorial-dr-drill-azure)

    - *VMware-to-Azure replication:* [Run a DR drill to Azure](/azure/site-recovery/tutorial-dr-drill-azure)

## Resilience to configuration and replication problems

A DR solution is only reliable when you know that it works before a disaster occurs. Monitor Site Recovery to detect problems such as configuration errors or VM replication health problems. For more information, see [Monitor Site Recovery](/azure/site-recovery/monitor-site-recovery).

We recommend that you configure Azure Monitor alerts so that you're informed about problems with replication health. For more information, see [Built-in Azure Monitor alerts for Site Recovery](/azure/site-recovery/site-recovery-monitor-and-troubleshoot#built-in-azure-monitor-alerts-for-azure-site-recovery).

## Resilience to service maintenance

Azure automatically manages updates and maintenance for the core Site Recovery service. Maintenance operations don't require downtime and don't interrupt replication of your VMs and servers.

However, you're responsible for applying updates to Site Recovery components on your VMs and servers, including the mobility agent when required.

> [!IMPORTANT]
> We strongly recommend that you enable automatic updates for agents. If the agent version falls more than four versions behind, replication is turned off and your workload's recoverability is compromised.

For more information, see [Service updates in Site Recovery](/azure/site-recovery/service-updates-how-to).

## Service-level agreement

[!INCLUDE [Service-level agreement](includes/reliability-service-level-agreement-include.md)]

For Site Recovery, separate SLAs cover:

- **Service availability**, which means that Site Recovery is available to fail over protected instances. A protected instance is a VM or physical server that replicates to a secondary location. To be eligible for this SLA, you must retry failed failover attempts at least every 30 minutes.

- **Recovery time objective (RTO)**, which is the time from when you trigger a failover (or your scripts trigger it) to when the target VM runs. This time excludes manual actions or script execution.

The SLA provides service credits only when the secondary region has sufficient compute capacity.

## Related content

- [About Site Recovery](/azure/site-recovery/site-recovery-overview)
- [Reliability in Azure](overview.md)
