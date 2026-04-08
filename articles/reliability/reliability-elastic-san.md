---
title: Reliability in Azure Elastic SAN
description: Learn how to make Azure Elastic SAN resilient to a variety of potential outages and problems, including transient faults, availability zone outages, and region outages, and learn about backup and restore.
author: roygara
ms.author: rogarana
ms.topic: reliability-article
ms.custom: subject-reliability
ms.service: azure-elastic-san-storage
ms.date: 04/09/2026
---

# Reliability in Azure Elastic SAN

[Azure Elastic SAN](/azure/storage/elastic-san/elastic-san-introduction) is a cloud-native storage area network (SAN) service that provides a scalable, cost-effective, high-performance, and comprehensive storage solution for a range of compute options. Elastic SAN enables you to create and manage volumes, which are virtual disks that you can connect to your virtual machines, containers, or other Azure services via the iSCSI protocol.

[!INCLUDE [Shared responsibility](includes/reliability-shared-responsibility-include.md)]

This article describes how to make Azure Elastic SAN resilient to a variety of potential outages and problems, including transient faults, availability zone failures, and region-wide failures. It also describes backup and recovery options, and highlights key information about the Azure Elastic SAN service-level agreement (SLA).

## Production deployment recommendations for reliability

For production workloads, we recommend that you:

> [!div class="checklist"]
> - Use zone-redundant storage (ZRS) to spread your data across three availability zones.
> - Use private endpoints for network access to enable automatic zone failover without manual intervention.
> - Create snapshots of your volumes regularly and export them to managed disk snapshots for data protection.
> - For workloads that require disaster recovery, copy snapshots to a secondary region that is geographically distant from your primary region.

## Reliability architecture overview

[!INCLUDE [Introduction to reliability architecture overview section](includes/reliability-architecture-overview-introduction-include.md)]

### Logical architecture

Elastic SAN has a three-level resource hierarchy:

- **Elastic SAN**: The top-level resource where you configure redundancy (LRS or ZRS), allocate storage capacity, and set performance limits. The storage capacity you allocate determines the total IOPS and throughput available across the entire SAN.
- **Volume groups**: Management constructs used to manage volumes at scale. Network access settings (such as private endpoints or service endpoints) are configured at the volume group level and inherited by all volumes in the group.
- **Volumes**: Individual storage volumes partitioned from the SAN's total capacity. Volumes are connected to compute resources via the iSCSI protocol.

### Physical architecture

<!-- TODO redo -->
Depending on the redundancy option you select when you create the Elastic SAN, the platform manages the underlying storage infrastructure:

- **LRS (locally redundant storage)**: The SAN is replicated three times within a single storage cluster in one datacenter.
- **ZRS (zone-redundant storage)**: Three copies of the SAN are stored synchronously across three distinct availability zones, each in a physically isolated datacenter.

You don't manage or see the underlying storage clusters or availability zone placement directly. Microsoft handles replication and zone distribution automatically based on your redundancy selection.

## Resilience to transient faults

<!-- TODO check if the iSCSI protocol/clients require anything to handle TFs effectively -->

[!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)]

If your iSCSI connection to an Elastic SAN volume is interrupted, the iSCSI initiator on the client automatically attempts to reconnect. You might experience a brief pause in I/O operations during the reconnection. Configure your iSCSI initiator with appropriate retry and timeout settings to handle transient interruptions.

## Resilience to availability zone failures

[!INCLUDE [Resilience to availability zone failures](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)]

Azure Elastic SAN can be configured to use zone-redundant storage (ZRS), which means your data is replicated synchronously across three availability zones in the region. Zone redundancy helps you achieve resiliency and reliability for your production workloads.

<!-- TODO diagram -->

If you configure your Elastic SAN with locally redundant storage (LRS) instead of ZRS, your Elastic SAN is *nonzonal*, and data is stored in a single availability zone. LRS Elastic SANs aren't protected against availability zone failures.

### Requirements

**Region support:** Zone-redundant Elastic SAN resources can be deployed into a subset of regions. For a list of regions, see [Scale targets for Elastic SAN](/azure/storage/elastic-san/elastic-san-scale-targets).

### Considerations

<!-- TODO move -->
- **Connectivity for automatic failover:** To ensure automatic zone failover without manual intervention, use private endpoints to connect to your Elastic SAN. When you use service endpoints instead of private endpoints, you might need to restart the iSCSI initiator manually to initiate a failover to a healthy zone.

### Cost

When you create an Elastic SAN with ZRS, the cost is higher than LRS. For more information about pricing, see [Azure Elastic SAN pricing](https://azure.microsoft.com/pricing/details/elastic-san/).

### Configure availability zone support

- **Create a new Elastic SAN with ZRS:** When you create an Elastic SAN and select ZRS as the redundancy option, your Elastic SAN is automatically zone-redundant. You can't change the redundancy option after the Elastic SAN is created. For more information about creating a new Elastic SAN resource, see [Deploy an Elastic SAN](/azure/storage/elastic-san/elastic-san-create).

- **Enable zone redundancy on an existing LRS Elastic SAN:** You can't convert an LRS Elastic SAN to ZRS in place. To migrate, snapshot your Elastic SAN volumes, export them to managed disk snapshots, deploy a new Elastic SAN on ZRS, and then create volumes on the new Elastic SAN using those disk snapshots. For more information, see [Snapshot Azure Elastic SAN volumes](/azure/storage/elastic-san/elastic-san-snapshots).

### Behavior when all zones are healthy

This section describes what to expect when you configure an Elastic SAN for zone redundancy, and all zones are operational.

- **Cross-zone operation:** When you connect to an Elastic SAN volume, your iSCSI connection is routed to a cluster in one of the availability zones. The platform automatically routes traffic between zones.

- **Cross-zone data replication:** When a client writes data to an Elastic SAN volume, that data is written synchronously to clusters within three availability zones before the write operation is acknowledged. Synchronous replication ensures a high level of data consistency and ensures there's no data loss during a zone failure.

### Behavior during a zone failure

This section describes what to expect when you configure an Elastic SAN for zone redundancy, and there's an outage in one of the zones.

- **Detection and response:** The Elastic SAN platform is responsible for detecting a failure in an availability zone. You don't need to do anything to initiate a zone failover for ZRS Elastic SANs.

- **Notification:** TODO

- **Active requests:** When an availability zone is unavailable, any in-progress I/O operations connected to a replica in the faulty availability zone might be terminated and need to be retried. If you use private endpoints, the failover happens automatically. If you use service endpoints, you might need to restart the iSCSI initiator to fail over to a healthy zone.

- **Expected data loss:** A zone failure isn't expected to cause any data loss because data is synchronously replicated across three availability zones.

- **Expected downtime:** When you use private endpoints, zone failover happens automatically. You might experience availability and performance degradation for a few minutes after a failover while the SAN rebalances itself. When you use service endpoints, your volumes might be unavailable until you restart the iSCSI initiator.

- **Traffic rerouting:** When a zone is unavailable, the Elastic SAN platform detects the loss of the zone and routes traffic to the remaining healthy zones. This process is automatic when you use private endpoints.

### Zone recovery

When the availability zone recovers, the Elastic SAN platform automatically restores normal operations and resumes replication across three zones. You don't need to take any action.

### Test for zone failures

The Azure Elastic SAN platform manages traffic routing, failover, and zone recovery for zone-redundant resources. Because this feature is fully managed, you don't need to validate availability zone failure processes.

## Resilience to region-wide failures

Azure Elastic SAN is a single-region service. If the region becomes unavailable, your Elastic SAN resource is also unavailable. There's no built-in cross-region replication or failover for Elastic SAN. You're responsible for architecting your own multi-region disaster recovery solution if your workload requires region-level resiliency.

### Custom multi-region solutions for resiliency

You're responsible for implementing multi-region disaster recovery for your Elastic SAN data. The recommended approach is to use volume snapshots:

1. **Create snapshots regularly.** Use [volume snapshots](/azure/storage/elastic-san/elastic-san-snapshots) to capture point-in-time copies of your Elastic SAN volumes.

    Your recovery point objective (RPO) depends on how frequently you create and copy snapshots to the secondary region. The more frequently you create snapshots and copy them, the lower your potential data loss during a disaster.

1. **Export snapshots to managed disk snapshots.** [Export your volume snapshots](/azure/storage/elastic-san/elastic-san-snapshots#export-volume-snapshot) to managed disk snapshots, which can be copied to other regions.

1. **Copy snapshots to a secondary region.** [Copy the incremental snapshot to a new region](/azure/virtual-machines/disks-copy-incremental-snapshot-across-regions) that is geographically distant from your primary region. This reduces the risk of multiple regions being affected by a single disaster.

1. **Restore from snapshots.** In a disaster recovery scenario, create new volumes on the secondary Elastic SAN from the copied managed disk snapshots.

Your recovery time objective (RTO) depends on the size of your data, the time it takes to copy snapshots across regions, and the time needed to deploy and configure a new Elastic SAN in the secondary region. To reduce recovery time, consider deploying a secondary Elastic SAN in your recovery region before a disaster occurs. This also helps avoid capacity constraints during an outage.

## Backup and restore

[!INCLUDE [Backups description](includes/reliability-backups-include.md)]

Microsoft doesn't provide automatic backups for Elastic SAN. You're responsible for creating and managing snapshots based on your data protection requirements.

Azure Elastic SAN supports volume snapshots for data protection. Snapshots are incremental, point-in-time copies of your volumes that consume space from the total capacity of your Elastic SAN. To protect your data, create snapshots regularly. The frequency depends on how much data you can afford to lose (your RPO). You can create snapshots manually or build your own automation to create them on a schedule.

Snapshots are stored within the same Elastic SAN as your volumes and use the same redundancy setting. To protect against region-wide failures, export your snapshots to managed disk snapshots and copy them to a different region. For more information, see [Export volume snapshot](/azure/storage/elastic-san/elastic-san-snapshots#export-volume-snapshot) and [Copy an incremental snapshot to a new region](/azure/virtual-machines/disks-copy-incremental-snapshot-across-regions).

You can create a new Elastic SAN volume from a snapshot or from a managed disk snapshot. For more information, see [Create a volume from a snapshot](/azure/storage/elastic-san/elastic-san-snapshots#create-a-volume-from-a-snapshot).

## Resilience to service maintenance

[!INCLUDE [Service maintenance (no special callouts)](includes/reliability-maintenance-include.md)]

> [!WARNING]
> **Note to PG:** Please verify whether there are any other recommendations for customers to follow to be resilient to service maintenance.

## Service-level agreement

[!INCLUDE [Service-level agreement](includes/reliability-service-level-agreement-include.md)]

## Related content

- [What is Azure Elastic SAN?](/azure/storage/elastic-san/elastic-san-introduction)
- [Plan for deploying an Elastic SAN](/azure/storage/elastic-san/elastic-san-planning)
- [Deploy an Elastic SAN](/azure/storage/elastic-san/elastic-san-create)
- [Snapshot Azure Elastic SAN volumes](/azure/storage/elastic-san/elastic-san-snapshots)
- [Reliability in Azure](/azure/reliability/overview)
