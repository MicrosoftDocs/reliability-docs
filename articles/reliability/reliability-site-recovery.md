---
title: Reliability in Azure Site Recovery
description: Learn how to make Azure Site Recovery resilient to a variety of potential outages and problems, including transient faults, availability zone outages, and region outages.
author: anaharris-ms
ms.author: anaharris
ms.topic: reliability-article
ms.custom: subject-reliability
ms.service: azure-site-recovery
ms.date: 01/09/2026
ai-usage: ai-assisted

#Customer intent: As an engineer responsible for business continuity, I want to understand who need to understand the details of how Azure Site Recovery works from a reliability perspective and plan strategies in alignment with the exact processes that Azure services follow during different kinds of situations.
---

# Reliability in Azure Site Recovery

[Azure Site Recovery](../site-recovery/site-recovery-overview.md) is a managed replication and failover service for virtual machines and other infrastructure, designed to keep workloads available during outages. It continuously replicates workloads from primary sites to secondary locations, ensuring minimal data loss and downtime. In the event of planned maintenance or unexpected disruptions, it orchestrates failover and failback processes seamlessly. This service supports disaster recovery for on-premises environments and Azure VMs, helping organizations maintain business continuity.

[!INCLUDE [Shared responsibility](includes/reliability-shared-responsibility-include.md)]

This article describes how to make Azure Site Recovery resilient to a variety of potential outages and problems, including transient faults, availability zone outages, and region outages. It also highlights some key information about the Azure Site Recovery service level agreement (SLA).

> [!NOTE]
> This document describes how the Azure Site Recovery service itself is, or can be made, resilient to various issues. It doesn't explain how to use Azure Site Recovery to protect your VMs or other assets. To learn about how to use Azure Site Recovery, see [About Site Recovery](../site-recovery/site-recovery-overview.md).

## Production deployment recommendations for reliability

For production workloads, we recommend that you:

> [!div class="checklist"]
> - Use High Churn for VMs that have a high rate of data change.
> - Use ZRS on the cache storage account

<!-- TODO verify these -->

## Reliability architecture overview

[!INCLUDE [Introduction to reliability architecture overview section](includes/reliability-architecture-overview-introduction-include.md)]

### Logical

<!-- TODO -->
- To define:
    - source
    - target
    - vault
    - cache storage account (todo check if this is used for all types or just Azure-to-Azure)
- Optional [recovery plan](/azure/site-recovery/recovery-plan-overview)

- Configure replication from a source VM, or from another supported source - physical, VMware, Hyper-V
- Can go across zones or regions

> [!NOTE]
> This guide focuses on the reliability of the Azure-based components of Azure Site Recovery and the replication relationship. If you replicate data or VMs from an on-premises environment or another cloud provider, you should consider the reliability of the components outside of Azure, too.

### Physical

- Vault stores configuration. The replication config doesn't matter for ASR
- Replication uses a cache storage account in the source region, which stores logs before ASR writes the change to the destination. We recommend ZRS <!-- PG to confirm -->

## Resilience to transient faults

Site Recovery automatically handles transient faults that occur during replication by retrying. You don't need to configure transient fault handling for Azure Site Recovery.

## Resilience to availability zone failures

[!INCLUDE [Resilience to availability zone failures](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)]

<!-- TODO -->

- Site Recovery 

> [!NOTE]
> Azure Site Recovery can help you to fail over between VMs in different availability zones. For more information, see [Enable Azure VM disaster recovery between availability zones](../site-recovery/azure-to-azure-how-to-enable-zone-to-zone-disaster-recovery.md).

### Requirements

**Region support:** Azure Site Recovery is deploying support for availability zones in [all availability zone-enabled regions](./regions-list.md).

### Cost

Site Recovery is billed based on the number of VM instances protected, regardless of their availability zone configuration.

### Configure availability zone support

- No config on ASR itself
- Recovery vault replication setting doesn't affect ASR
- Cache storage account should be configured for ZRS - see storage guide

### Behavior when all zones are healthy

This section describes what to expect when Site Recovery is used in a region with availability zones, your cache storage account is configured to use ZRS, and all availability zones are operational.

- **Traffic routing between zones:** Site Recovery uses infrastructure in multiple availability zones to trigger and run replication jobs. The service manages this infrastucture transparently to you.

- **Data replication between zones:**

    - ASR replicates config data even if your vault is LRS <!-- PG to verify -->
    - Cache storage account uses Azure Storage replication betwen zones

### Behavior during a zone failure

This section describes what to expect when Site Recovery is used in a region with availability zones, your cache storage account is configured to use ZRS, and an availability zone outage occurs.

- **Detection and response:** The Site Recovery platform automatically detects failures in an availability zone and initiates a response. No manual intervention is required to initiate a zone failover for the Site Recovery service itself. However, if the zone outage affects your source VM, you might need to [initiate failover of your VM](/azure/site-recovery/azure-to-azure-tutorial-failover-failback).

[!INCLUDE [Availability zone down notification (Service Health only)](./includes/reliability-availability-zone-down-notification-service-include.md)]

- **Active requests:** The effect on active replication jobs depends on the type of replication:

    - *Zone-to-zone and region-to-region replication of Azure VMs:* If either the source or target instance is in the failed zone, replication pauses until both instances are available again.

        If the failed zone doesn't contain the source or target VM, replication continues to run. <!-- TODO PG confirm timeline in which this will be true -->

    - *On-premises to Azure:* If the target instance is in the failed zone, replication pauses until the instance is available again.

        If the failed zone doesn't contain the target VM, replication continues to run. <!-- TODO PG confirm timeline in which this will be true -->
    
- **Expected data loss:** No data loss is expected during a zone failure.

- **Expected downtime:** If the failed zone contains either the source or target instance, replication pauses until both instances are available agian.

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

However, you can use [disaster recovery drills](/azure/site-recovery/azure-to-azure-tutorial-dr-drill) to test your VM failover.

## Resilience to region-wide failures

> [!NOTE]
> Azure Site Recovery can help you to fail over between VMs in different regions. For more information, see [Replicate Azure VMs to another Azure region](../site-recovery/azure-to-azure-how-to-enable-replication.md).

### Multi-region support type 1

#### Requirements

- **Region support:** <!-- Complete this line. -->

#### Considerations

#### Cost

#### Configure multi-region support

#### Capacity planning and management

#### Behavior when all regions are healthy

This section describes what to expect when \[service-name\] resources are configured with multi-region suypport type 1 support and all regions are operational.

- **Traffic routing between regions:** <!-- Complete this line. -->

- **Data replication between regions:** <!-- Complete this line. -->

#### Behavior during a region failure

This section describes what to expect when \[service-name\] resources are configured with multi-region suypport type 1 support and a region outage occurs.

- **Detection and response:** <!-- Complete this line. -->

- **Notification:** <!-- Complete this line. -->

- **Active requests:** <!-- Complete this line. -->

- **Expected data loss:** <!-- Complete this line. -->

- **Expected downtime:** <!-- Complete this line. -->

- **Traffic rerouting:** <!-- Complete this line. -->

#### Region recovery

#### Testing for region failures

## Resilience to service maintenance

Azure automatically manages updates and maintenance for the core Site Recovery service. Maintenance operations don't require downtime and don't interrupt replication of your VMs and servers.

However, you're responsible for applying updates to Site Recovery components on your VMs and servers. For more information, see [Service updates in Site Recovery](/azure/site-recovery/service-updates-how-to).

## Service-level agreement

[!INCLUDE [Service-level agreement](includes/reliability-service-level-agreement-include.md)]

For Azure Site Recovery, there are separate SLAs that cover the following:
- The availability of protected VMs or physical machines during an attempted failover.
- The recovery time objective (RTO), which is the amount of downtime <!-- TODO -->
SLA covers the availability of the protected VM or physical machine during a failover attempt, and the RTO

Downtime - Failover of a Protected Instance is unsuccessful due to unavailability of the Azure Site Recovery Service, provided that retries are continually attempted no less frequently than once every thirty minutes.

Only applies when compute capacity is available in the secondary region.
