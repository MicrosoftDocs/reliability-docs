---
title: Reliability in Azure Site Recovery
description: Learn how to make Azure Site Recovery resilient to a variety of potential outages and problems, including transient faults, availability zone outages, and region outages.
author: glynnniall
ms.author: glynnniall
ms.topic: reliability-article
ms.custom: subject-reliability, references_regions
ms.service: azure-site-recovery
ms.date: 03/03/2026
ai-usage: ai-assisted
---

# Reliability in Azure Site Recovery

[Azure Site Recovery](/azure/site-recovery/site-recovery-overview) is a managed replication and failover service for virtual machines, designed to keep workloads available during outages. It continuously replicates workloads from primary sites to secondary locations, ensuring minimal data loss and downtime. In the event of planned maintenance or unexpected disruptions, it orchestrates failover and failback processes. This service supports disaster recovery for on-premises environments and Azure VMs, helping organizations maintain business continuity.

[!INCLUDE [Shared responsibility](includes/reliability-shared-responsibility-include.md)]

This article describes how to make Azure Site Recovery resilient to a variety of potential outages and problems, including transient faults, availability zone outages, and region outages. It also highlights some key information about the Azure Site Recovery service level agreement (SLA).

> [!NOTE]
> This document describes how the Azure Site Recovery service itself is, or can be made, resilient to various issues. It doesn't explain how to use Azure Site Recovery to protect your VMs or other assets. To learn about how to use Azure Site Recovery, see [About Site Recovery](/azure/site-recovery/site-recovery-overview).

## Production deployment recommendations for reliability

When using Site Recovery with production workloads, we recommend that you take these actions:

> [!div class="checklist"]
> - Deploy your Recovery Services vault in your target region for replication.
> - For Azure to Azure disaster recovery, use [High Churn](/azure/site-recovery/concepts-azure-to-azure-high-churn-support) for VMs that have a high rate of data change. High Churn support improves your recovery point objective (RPO) and enables replication for many high-scale database workloads.
> - For Azure to Azure disaster recovery, configure the cache storage account to use zone-redundant storage (ZRS).
> - Perform test failovers on a regular basis as part of disaster recovery (DR) drills. DR drills should be run every quarter or biannually to verify your replication and failover processes are healthy.
> - Use [on-demand capacity reservations](/azure/virtual-machines/capacity-reservation-overview) to ensure compute resources are available in your target region for failover.
> - Enable automatic updates for mobility agents.
> - Monitor the health of your replication, and configure alerts so you're notified if a problem happens.

## Reliability architecture overview

When you use Azure Site Recovery, you define a *source* and *target*, which represent the VMs that are replicated:

- The *source* can be an Azure VM, or a VM or server from another supported source, including on-premises physical servers, VMware VMs, and Hyper-V VMs.
- The *target* is always an Azure VM. For Azure-to-Azure VM replication, the target can be a different region or availability zone from the source VM.

You're responsible for deploying and configuring other resources, including the following:

- *Recovery Services vault*, which Site Recovery uses to store your replication configuration settings. The vault doesn't store your replicated data. The redundancy configuration of the vault isn't important for Site Recovery, but it's important if you use the same vault for Azure Backup.

    A vault can include additional configuration, such as:
    - *Replication policy*, which configures the snapshot frequency and retention length.
    - [*Recovery plan*](/azure/site-recovery/recovery-plan-overview), which coordinates the order in which machines fail over and can include scripts and manual actions. Recovery plans are particularly useful for workloads with multiple tiers, such as application and database tiers, that need to fail over in a coordinated fashion.

- For Azure-to-Azure replication, a *cache storage account* that stores a copy of the source data in its region before it's replicated to the target. The redundancy configuration of your cache storage account can affect your reliability during an availability zone outage.

> [!NOTE]
> This guide focuses on the reliability of the Azure-based components of Azure Site Recovery and the replication relationship. If you replicate data or VMs from an on-premises environment or another cloud provider, you should also consider the reliability of the components outside of Azure.

For more information about the components you deploy, see:

- [Azure to Azure disaster recovery architecture](/azure/site-recovery/azure-to-azure-architecture)
- [Hyper-V to Azure disaster recovery architecture](/azure/site-recovery/hyper-v-azure-architecture)
- [VMware to Azure disaster recovery architecture](/azure/site-recovery/vmware-azure-architecture-modernized)
- [Physical server to Azure disaster recovery architecture](/azure/site-recovery/physical-server-azure-architecture-modernized)

The core Site Recovery service runs on infrastructure that Microsoft manages. This document refers to these components collectively as the *core Site Recovery service*.

## Resilience to transient faults

Transient faults are short, intermittent failures in components. They occur frequently in a distributed environment like the cloud, and they're a normal part of operations. Transient faults correct themselves after a short period of time.

Site Recovery automatically handles transient faults that occur during the replication process by retrying its operations. You don't need to configure transient fault handling for Azure Site Recovery.

## Resilience to availability zone failures

[!INCLUDE [Resilience to availability zone failures](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)]

To understand how Azure Site Recovery replication behaves during availability zone failures, you need to consider the following components of the service:

- **Core Site Recovery service:** The Site Recovery service is designed to be resilient to availability zone failures in supported regions. The internal components of the service support zone redundancy automatically with no customer configuration required.

- **Recovery Services vault:** The vault stores configuration data. In regions where Site Recovery supports zone resilience, configuration data in the vault is also zone-resilient.

- **Cache storage account:** For Azure-to-Azure replication, you're responsible for ensuring that the cache storage account is zone-redundant by deploying it using the ZRS tier.

    If you use the locally redundant storage (LRS) Azure Storage replication tier for your cache storage account, then if a zone fails, Site Recovery might not be able to replicate recently changed data to your target.

> [!NOTE]
> Azure Site Recovery can help you to fail over between VMs in different availability zones. For more information, see [Enable Azure VM disaster recovery between availability zones](/azure/site-recovery/azure-to-azure-how-to-enable-zone-to-zone-disaster-recovery).

### Requirements

**Region support:**

- **Core Site Recovery service and Recovery Services vaults:** Azure Site Recovery is zone-resilient in the following regions:

    | Americas       | Europe         | Middle East    | Asia Pacific      |
    |----------------|----------------|----------------|-------------------|
    | Chile Central  | Austria East   | Israel Central | Indonesia Central |
    | Mexico Central | Italy North    |                | Japan West        |
    | West US 3      | Poland Central |                | Malaysia West     |
    |                | Spain Central  |                | New Zealand North |

     Azure Site Recovery is currently deploying support for availability zones in [all availability zone-enabled regions](./regions-list.md). In regions that aren't yet zone-resilient, zone failures might affect operations.

- **Cache storage account:** You can deploy a ZRS storage account in all availability zone-enabled regions.

### Cost

Site Recovery is billed based on the number of VM instances protected, regardless of their availability zone configuration. For more information, see [Azure Site Recovery pricing](https://azure.microsoft.com/pricing/details/site-recovery/).

### Configure availability zone support

- **Core Site Recovery service:** You don't configure zone resiliency on the core Site Recovery service. Microsoft provides zone resiliency in supported regions.

    If Microsoft enables zone resiliency in a region at a later time, your Site Recovery resources automatically benefit from the zone resilience. You don't need to take any action.

- **Recovery Services vault:** Although Recovery Services vaults enable you to configure a level of redundancy, this configuration setting isn't used for Site Recovery. You don't need to configure your vault for zone redundancy when you use Site Recovery.

- **Cache storage account:** When you use Azure-to-Azure replication, you're responsible for creating the cache storage account and for configuring it with the appropriate level of redundancy. To make it zone-redundant, configure it for the ZRS replication type. For more information, see [Reliability in Azure Blob Storage](./reliability-storage-blob.md).

### Behavior when all zones are healthy

This section describes what to expect when Site Recovery is used in a region with availability zone support for the core service, your cache storage account is configured to use ZRS, and all availability zones are operational.

- **Cross-zone operation:** The replication process can use infrastructure in multiple availability zones to trigger and run replication jobs. The service manages this infrastructure transparently to you.

- **Cross-zone data replication:** Site Recovery and Azure Storage handle zone data replication as follows:

    - *Site Recovery configuration:* Site Recovery replicates your configuration data across zones even if your vault is configured to use LRS.

    - *Cache storage account:* If your cache storage account is configured to use ZRS, Azure Storage synchronously replicates the cached data between zones.

### Behavior during a zone failure

This section describes what to expect when Site Recovery is used in a region with availability zone support for the core service, your cache storage account is configured to use ZRS, and an availability zone outage occurs.

> [!NOTE]
> If the failed zone contains the source VM, you're responsible for triggering failover to the target. For more information, see:
> - [Fail over Azure VMs to a secondary region](/azure/site-recovery/azure-to-azure-tutorial-failover-failback)
> - [Fail over VMware VMs](/azure/site-recovery/vmware-azure-tutorial-failover-failback-modernized)
> - [Fail over Hyper-V VMs to Azure](/azure/site-recovery/hyper-v-azure-failover-failback-tutorial)

- **Detection and response:** The Site Recovery platform automatically detects failures in an availability zone and initiates a response. No manual intervention is required to initiate a zone failover for the Site Recovery service itself. However, if the zone outage affects your source VM, you might need to [initiate failover of your VM](/azure/site-recovery/azure-to-azure-tutorial-failover-failback).

[!INCLUDE [Availability zone down notification (Service Health only)](./includes/reliability-availability-zone-down-notification-service-include.md)]

- **Active requests:** The effect on active replication jobs depends on the type of replication:

    - *Zone-to-zone and region-to-region replication of Azure VMs:* If either the source or target instance is in the failed zone, replication pauses until both instances are available again.

        If the failed zone doesn't contain the source or target VM, and the cache storage account is configured to use ZRS, then replication continues to run.

    - *On-premises to Azure:* If the target instance is in the failed zone, replication pauses until the instance is available again.

        If the failed zone doesn't contain the target VM, replication continues to run.

- **Expected data loss:** No data loss is expected during a zone failure.

- **Expected downtime:** If the failed zone contains either the source or target VM, replication pauses until both instances are available again.

- **Redistribution:** Site Recovery and Azure Storage automatically adapt to zone failures:

    - *Site Recovery core service:* The Site Recovery service automatically uses infrastructure in healthy availability zones to perform replication. You don't need to take any action.

    - *Cache storage account:* Azure Storage automatically routes requests for cache data to healthy zones.

### Zone recovery

When the affected availability zone recovers, Site Recovery automatically resumes any replication jobs that might have paused during the zone outage.

You're responsible for initiating failback for any servers or VMs that you failed over during the zone outage. For more information, see:

- *Zone-to-zone and region-to-region replication of Azure VMs:* [Fail back Azure VM to the primary region](/azure/site-recovery/azure-to-azure-tutorial-failback)

- *On-premises to Azure replication:*
    - *Physical to Azure replication:* [Physical server to Azure disaster recovery architecture](/azure/site-recovery/physical-server-azure-architecture-modernized)
    - *Hyper-V to Azure replication:* [Hyper-V to Azure disaster recovery architecture](/azure/site-recovery/hyper-v-azure-architecture#failover-and-failback-process)
    - *VMware to Azure replication:* [About on-premises disaster recovery failover/failback](/azure/site-recovery/failover-failback-overview-modernized)

### Test for zone failures

The Site Recovery platform manages zone resiliency for its internal components. Because this feature is fully managed, you don't need to initiate or validate availability zone failure processes.

It's important to perform regular disaster recovery drills, which should test your VM failover as well as your overall response procedures. Design your DR drills to avoid impact to your production environment. For more information, see:

- *Zone-to-zone and region-to-region replication of Azure VMs:* [Run a disaster recovery drill for Azure VMs](/azure/site-recovery/azure-to-azure-tutorial-dr-drill)

- *On-premises to Azure replication:*
    - *Physical to Azure replication:* [Run a test failover (disaster recovery drill) to Azure](/azure/site-recovery/site-recovery-test-failover-to-azure)
    - *Hyper-V to Azure replication:* [Run a disaster recovery drill to Azure](/azure/site-recovery/tutorial-dr-drill-azure)
    - *VMware to Azure replication:* [Run a disaster recovery drill to Azure](/azure/site-recovery/tutorial-dr-drill-azure)

## Resilience to region-wide failures

For Azure-to-Azure replication, Site Recovery is designed to provide resilience to region failures by enabling failover of VMs to a healthy target region. For more information, see [Replicate Azure VMs to another Azure region](/azure/site-recovery/azure-to-azure-how-to-enable-replication).

### Considerations

- **Vault region:** A Recovery Services vault is deployed into a specific Azure region, which you select. The vault's region is an important decision. Replication can continue during an outage in the vault's region. However, Site Recovery management operations, including failover and failback, aren’t available until the region recovers.

    Deploying the vault in the target region helps ensure that failover and recovery operations remain accessible during a source-region outage, and prevents an outage in a third region from affecting failover and recovery operations.

    > [!NOTE]
    > If your vault is in the region that you ordinarily use as your target region, then after you fail over and re-establish replication, the vault is now in your new source region. If that region subsequently experiences a problem, you might not be able to perform failback until both regions are healthy.

- **Capacity reservations:** You're responsible for verifying that your target region supports the VM types that you need, and that it has available capacity for your workload. We recommend using [on-demand capacity reservations](/azure/virtual-machines/capacity-reservation-overview), which ensure that compute resources are available for your workload if a failover occurs.

### Configure multi-region support

- **Recovery Services vault:** You need to select the vault's region. For more information, see the preceding [considerations](#considerations) section.

    Although Recovery Services vaults enable you to configure a level of redundancy, this configuration setting isn't used for Site Recovery. You don't need to configure your vault for geo-redundancy when you use Site Recovery.

- **Cache storage account:** Because the cache storage account is only used as a temporary location for data before it's replicated, you shouldn't configure it to use GRS.

### Behavior during a region failure

The specific behavior of the Site Recovery core service during a region failure depends on which region experiences the failure:

- **Failure in source region:** For Azure-to-Azure replication, you can trigger a failover when the source region is unavailable.

    Because the source region is unavailable, replication stops until the VM in the source region is healthy.

- **Failure in target region:** Because the target region is unavailable, replication stops, and you can't fail over to the target until the region is healthy.

- **Failure in the region that contains the vault:** If the vault is deployed into a third region (not the source or target region) and that region experiences a failure, Site Recovery continues to replicate your data. However, you can't initiate any operations, including failover or failback, until the vault is healthy.

### Region recovery

You're responsible for initiating failback for any servers or VMs that you failed over during the region outage. For more information, see:

- *Zone-to-zone and region-to-region replication of Azure VMs:* [Fail back Azure VM to the primary region](/azure/site-recovery/azure-to-azure-tutorial-failback)

- *On-premises to Azure replication:*
    - *Physical to Azure replication:* [Physical server to Azure disaster recovery architecture](/azure/site-recovery/physical-server-azure-architecture-modernized)
    - *Hyper-V to Azure replication:* [Hyper-V to Azure disaster recovery architecture](/azure/site-recovery/hyper-v-azure-architecture#failover-and-failback-process)
    - *VMware to Azure replication:* [About on-premises disaster recovery failover/failback](/azure/site-recovery/failover-failback-overview-modernized)

### Test for region failures

It's important to perform regular disaster recovery drills, which should test your VM failover as well as your overall response procedures. Design your DR drills to avoid impact to your production environment. For more information, see:

- *Zone-to-zone and region-to-region replication of Azure VMs:* [Run a disaster recovery drill for Azure VMs](/azure/site-recovery/azure-to-azure-tutorial-dr-drill)

- *On-premises to Azure replication:*
    - *Physical to Azure replication:* [Run a test failover (disaster recovery drill) to Azure](/azure/site-recovery/site-recovery-test-failover-to-azure)
    - *Hyper-V to Azure replication:* [Run a disaster recovery drill to Azure](/azure/site-recovery/tutorial-dr-drill-azure)
    - *VMware to Azure replication:* [Run a disaster recovery drill to Azure](/azure/site-recovery/tutorial-dr-drill-azure)

## Resilience to configuration and replication problems

A disaster recovery solution is only reliable if you know it's working before a disaster strikes. This means it's important to monitor Azure Site Recovery in case any problems arise, such as configuration problems, or problems with the health of your VM replication. For more information, see [Monitor Azure Site Recovery](/azure/site-recovery/monitor-site-recovery).

We recommend that you configure Azure Monitor alerts so that you're informed of problems with replication health. For more information, see [Built-in Azure Monitor alerts for Azure Site Recovery](/azure/site-recovery/site-recovery-monitor-and-troubleshoot#built-in-azure-monitor-alerts-for-azure-site-recovery).

## Resilience to service maintenance

Azure automatically manages updates and maintenance for the core Site Recovery service. Maintenance operations don't require downtime and don't interrupt replication of your VMs and servers.

However, you're responsible for applying updates to Site Recovery components on your VMs and servers, including the mobility agent where required.

> [!IMPORTANT]
> We strongly recommend you enable automatic updates for agents. If the agent version falls more than four versions behind, replication is disabled and your workload's recoverability is compromised.

For more information, see [Service updates in Site Recovery](/azure/site-recovery/service-updates-how-to).

## Service-level agreement

[!INCLUDE [Service-level agreement](includes/reliability-service-level-agreement-include.md)]

For Azure Site Recovery, there are separate SLAs that cover the following:

- **Service availability**, which means the Site Recovery service is available to fail over protected instances. A protected instance is a VM or physical server that's replicated to a secondary location. To be eligible for this SLA, you must retry failed failover attempts at least every 30 minutes.
- **Recovery time objective (RTO)**, which is the length of time it takes from when you (or scripts you write) trigger a failover to when the target VM is running. This time excludes any manual actions or script execution.

The SLA only provides for service credits when there's sufficient capacity available in the secondary region.

## Related content

- [About Azure Site Recovery](/azure/site-recovery/site-recovery-overview)
- [Reliability in Azure](/azure/reliability/overview)
