---
title: Reliability in Azure Disk Storage
description: Learn about reliability in Azure Disk Storage, including availability zones and multi-region deployments.
author: glynnniall
ms.author: glynnniall
ms.topic: reliability-article
ms.custom: subject-reliability
ms.service: azure-disk-storage
ms.date: 04/07/2026
ai-usage: ai-assisted
---

# Reliability in Azure Disk Storage

Azure Disk Storage provides managed disks for Azure virtual machines (VMs). Built for mission-critical workloads, it ensures enterprise-grade reliability and availability. Your data is automatically replicated to guard against hardware failures, with multiple redundancy options to meet your durability requirements.

[!INCLUDE [Shared responsibility](includes/reliability-shared-responsibility-include.md)]

This article describes how to make Azure Disk Storage resilient to a variety of potential outages and problems, including transient faults, availability zone failures, and region-wide failures. It also describes backup and recovery options, and highlights key information about the Azure Disk Storage service level agreement (SLA).

> [!IMPORTANT]
> When you consider the reliability of a disk, you also need to consider the reliability of your [VMs](./reliability-virtual-machines.md), network infrastructure, and applications that run on your VMs. Improving the resiliency of the disk alone might have limited impact if the other components aren't equally resilient. Depending on your resiliency requirements, you might need to make configuration changes across multiple areas.

## Production deployment recommendations

The Azure Well-Architected Framework provides recommendations across reliability, performance, security, cost, and operations. To understand how these areas influence each other and contribute to a reliable Azure Disk Storage solution, see [Architecture best practices for Azure Disk Storage](/azure/well-architected/service-guides/azure-disk-storage).

## Reliability architecture overview

Each virtual machine (VM) uses disks for different purposes:

- *OS disk*: A single OS disk runs the operating system. By default, it’s a managed disk that persists data. You can also use [ephemeral OS disks, which aren't managed](/azure/virtual-machines/ephemeral-os-disks). Avoid using the OS disk to store applications or data.
- *Data disks*: Zero or more managed disks for storing applications and data.
- *Temporary disk*: A non-persistent, unmanaged disk included with every VM.

This guide specifically focuses on managed disks, which reliably persist data. To learn more about the different disk roles, see [Disk roles](/azure/virtual-machines/managed-disks-overview#disk-roles).

Managed disks are designed for 99.999% VM availability and provide at least 99.999999999% (11 9’s) of durability. With managed disks, your data is replicated three times. If one of the three copies becomes unavailable, Azure automatically spawns a new copy of the data in the background. This ensures the persistence of your data and high fault tolerance.

By default, managed disks use [locally redundant storage (LRS)](/azure/storage/common/storage-redundancy#locally-redundant-storage). LRS keeps three copies of your disk data within a single datacenter, protecting against hardware failures such as drive or server rack issues.

Although LRS protects your disks against server rack and drive failures, it doesn't account for disasters such as fire or flooding within a datacenter. For higher levels of protection, use [zone-redundant storage (ZRS)](#resilience-to-availability-zone-failures), which replicates your disks across multiple availability zones.

For applications running on multiple VMs, multiple VMs have the highest availability SLA when distributed across multiple availability zones. For VMs and disks distributed across multiple availability zones, the disks and their parent VMs are respectively collocated in the same zone, which prevents multiple VMs from going down even if an entire zone experiences an outage.

When zones aren’t available or your workload is sensitive to inter-VM latency, deploy VMs and disks across multiple [fault domains](/azure/virtual-machines/availability-set-overview#fault-domains). Fault domains don’t provide zone redundancy, but they reduce the impact of hardware failures, network outages, or power interruptions. This prevents multiple VMs from failing if one storage fault domain goes down.

## Resilience to transient faults

[!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)]

Managed disks automatically recover from transient faults in the Azure infrastructure.

## Resilience to availability zone failures

[!INCLUDE [Resilience to availability zone failures](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)]

There are two ways to use availability zones with managed disks:
- You can deploy a [zone-redundant (ZRS) disk](#zone-redundant-disks), which is in three availability zones in a region. For the best reliability, we recommend using ZRS disks because ZRS disks provide automatic zone resiliency.
- You can deploy a [zonal LRS disk](#zonal-lrs-disks), which is only in a single zone. With zonal LRS disks, you're responsible for configuring your workload to be resilient to zone outages. You do this deploying multiple VMs and disks and spreading them across availability zones.

If you don't configure availability zone support, your disk is *nonzonal* or *regional* and might be placed in any availability zone in the region. These disks are considered LRS because they are replicated within the region.

### Zone-redundant disks

Zone-redundant storage (ZRS) synchronously replicates your data across three availability zones within a region. When you enable zone redundancy for a managed disk, Azure ensures that a failure in any single zone doesn’t affect data availability.

![Diagram of a zone-redundant disk, with its replicas spread across three availability zones in the region.](./media/reliability-storage-disk/zone-redundant.svg)

ZRS disks can be [shared between VMs](/azure/virtual-machines/disks-shared) to improve availability for clustered or distributed applications such as SQL FCI, SAP ASCS/SCS, or GFS2. You can attach a shared ZRS disk to primary and secondary VMs in different zones, taking advantage of both ZRS disks and VMs distributed across multiple availability zones. If the primary zone fails, you can quickly fail over to the secondary VM using [SCSI persistent reservation](/azure/virtual-machines/disks-shared-enable#supported-scsi-pr-commands).

If a ZRS disk is attached as a data disk to a single VM in a zone that goes down, you can [force detach](/rest/api/compute/virtual-machines/attach-detach-data-disks#diskdetachoptiontypes) the disk from the failed VM and attach it to another VM.

#### Requirements

- **Region support:** For a list of regions that support ZRS managed disks, see [Redundancy options for managed disks](/azure/virtual-machines/disks-redundancy).

- **Disk types:** Zone-redundant disks are supported with Premium SSD and Standard SSD managed disks. ZRS isn't supported with Premium SSD v2, Ultra Disks, or Standard HDD managed disks.

#### Cost

ZRS incurs a higher cost than LRS due to the additional replication overhead and infrastructure required to maintain data across multiple zones. The exact cost difference varies by region and disk type. For detailed pricing information, see [Azure managed disk pricing](https://azure.microsoft.com/pricing/details/managed-disks/).

#### Configure availability zone support

- **Create a new ZRS disk:** To create a new ZRS managed disk, see [Tutorial - Manage Azure disks with the Azure CLI](/azure/virtual-machines/linux/tutorial-manage-disks) for Linux VMs, or [Tutorial: Manage disks with Azure PowerShell](/azure/virtual-machines/windows/tutorial-manage-data-disk) for Windows VMs. Select a ZRS disk tier during disk creation.

    You're responsible for attaching your disk to VMs, including configuring [shared disks](/azure/virtual-machines/disks-shared) on multiple VMs in different zones if that's appropriate for your workload.

- **Change an existing disk to use ZRS:** You can convert an existing nonzonal (regional) disk to ZRS.
    
    While you can't convert a zonal LRS disk to ZRS, you can create a new ZRS disk from a snapshot. See [Convert a disk from LRS to ZRS](/azure/virtual-machines/disks-migrate-lrs-zrs) for step-by-step migration procedures and requirements.

- **Disable availability zone support:** It's not possible to change the availability zone configuration of an existing ZRS disk. Instead, you need to create a new disk with the new configuration using a snapshot from the previous disk, and delete the old one.

#### Behavior when all zones are healthy

This section describes what to expect when managed disks are configured to use ZRS, and all availability zones are operational.

- **Cross-zone operation:** Azure automatically manages traffic routing between availability zones when you use a VM with a zone-redundant disk. During normal operations, requests are distributed across zones transparently.

- **Cross-zone data replication:** ZRS disks replicate every write synchronously across multiple availability zones in the region. A write operation completes only after data is stored in clusters in multiple zones. This approach provides strong consistency and high availability, but it can introduce slightly higher write latency compared to LRS disks.

#### Behavior during a zone failure

This section describes what to expect when a managed disk is configured to use ZRS, and there's an availability zone outage.

 - **Detection and response:** Some zone outages might affect only disks, only VMs, or both. The behavior you'll observe depends on whether the zone outage affects the VM attached to the disk.

    If the VM remains healthy but the disk is affected by the outage, your VM continues to operate. Microsoft automatically redirects disk operations to work against the data in healthy availability zones, and there's nothing you need to do.

    If the VM is down, you need to switch your workload to another VM in a different availability zone.
    
    - *Shared disks:* If you've already created the secondary VM in a different zone and have configured [shared disks](/azure/virtual-machines/disks-shared), the disk is available for the secondary VM to use with no configuration changes required.

    - *Disks that aren't shared:* You can *force detach* the disk from the failed VM and then attach it a VM in a healthy zone. To perform a force detach:

        - Azure CLI: Use the [az vm disk detach command](/cli/azure/vm/disk#az-vm-disk-detach) with the `--force-detach` argument.
        - Azure PowerShell: Use the [Remove-AzVMDataDisk cmdlet](/powershell/module/az.compute/remove-azvmdatadisk) with the `-ForceDetach` argument.

[!INCLUDE [Availability zone down notification (Service Health and Resource Health)](./includes/reliability-availability-zone-down-notification-service-resource-include.md)]

- **Expected data loss:** No data loss occurs during zone failures.

- **Expected downtime:** When your disk is shared between multiple VMs, no downtime is expected.

- **Redistribution:** Azure automatically reroutes traffic to another copy of your disk in a healthy zone.

#### Zone recovery

Azure automatically detects when the previously failed zone is healthy and restores data synchronization to the recovered zone.

### Zonal LRS disks

Zonal LRS disks reside in a specific availability zone and attach only to VMs in that zone. All of the copies of the disk's data are in the same zone. A single zonal LRS disk and virtual machine don't provide zone resiliency. If the zone containing the disk experiences an outage, the disk may become unavailable.

![Diagram that shows a zonal LRS disk, with all of its replicas in a single availability zone.](./media/reliability-storage-disk/zonal.svg)

For multi-VM workloads, you can achieve zone resiliency by deploying multiple VMs and their zonal LRS disks across different availability zones. This approach is the most common way to achieve high availability for workloads like web servers, application tiers, and database clusters. If a zone fails, you can configure your workload to continue to operate byu using the VMs in healthy zones.

![Diagram that shows three VMs in different zones, each with their own zonal LRS disk.](./media/reliability-storage-disk/disks-availability-zones.svg)

This multi-zone distribution pattern works with all disk types, including Premium SSD v2 and Ultra Disks, which only support LRS. For more information on this approach, see [Distribute VMs and disks across availability zones](/azure/virtual-machines/disks-high-availability#distribute-vms-and-disks-across-availability-zones).

#### Requirements

- **Region support:** Zonal LRS managed disks are supported in [all regions with availability zones](./regions-list.md).

- **Disk types:** All managed disk types support zonal LRS deployments.

#### Cost

Zonal LRS disks are charged at the same rate as nonzonal disks. For detailed pricing information, see [Azure managed disk pricing](https://azure.microsoft.com/pricing/details/managed-disks/).

#### Configure availability zone support

- **Create a new disk with availability zone support:** To create a new managed disk with zonal LRS redundancy, see [Tutorial - Manage Azure disks with the Azure CLI](/azure/virtual-machines/linux/tutorial-manage-disks) for Linux VMs, or [Tutorial: Manage disks with Azure PowerShell](/azure/virtual-machines/windows/tutorial-manage-data-disk) for Windows VMs.

    Select the availability zone during disk creation.

    [!INCLUDE [Zonal resource description](includes/reliability-availability-zone-zonal-include.md)]

- **Change the availability zone configuration of an existing disk:** It's not possible to change the availability zone configuration of an existing zonal LRS disk. Instead, you need to create a new disk with the new configuration using a snapshot from the previous disk, and delete the old one.

#### Behavior when all zones are healthy

This section describes what to expect when a managed disk is configured to use zonal LRS, and all availability zones are operational.

- **Cross-zone operation:** Traffic between a zonal VM and a zonal LRS disk in the same zone remains within the availability zone.

    When you deploy multiple VMs across zones, you're responsible for distributing incoming requests across the VMs. Each VM reads and writes to its own zonal disk.

- **Cross-zone data replication:** All write operations to zonal LRS disks are replicated synchronously within the availability zone.

    When you deploy multiple VMs across zones, if your workload requires data consistency across VMs, you're responsible for synchronizing data, such as by using database replication or application-layer replication.

#### Behavior during a zone failure

This section describes what to expect when a managed disk is configured to use zonal LRS, and there's an availability zone outage.

 - **Detection and response:** If you have a single VM with a zonal LRS disk, you're responsible for detecting a zone outage and triggering a failover or another response.
 
    When you have VMs distributed across multiple zones, you're responsible for configuring your workload to detect the zone failure and continue to run on the VMs in healthy zones.

[!INCLUDE [Availability zone down notification (Service Health and Resource Health)](./includes/reliability-availability-zone-down-notification-service-resource-include.md)]

- **Expected data loss:** LRS replication provides at least 99.999999999% (11 9’s) of durability, which means that your disk retains its data and the data can be recovered after the zone recovers.

    When you have VMs distributed across zones, any data that was only on the disks in the failed zone is temporarily unavailable. If your application synchronizes data across VMs, the VMs in healthy zones continue to serve requests using their own data.

- **Expected downtime:** A single zonal LRS disk is unavailable until the availability zone recovers.

    When you have VMs and disks distributed across zones, your workload can continue to operate on the VMs in healthy zones.

- **Redistribution:** If you have a single VM with a zonal LRS disk, you're responsible for rerouting traffic to another VM if you have one available.

    When you have VMs distributed across zones, you can configure your workload to automatically redistribute traffic to VMs in healthy zones.

#### Zone recovery

When the failed availability zone recovers, managed disks recover automatically. If the VM attached to the disk has been affected by the outage, it restarts. You're responsible for resyncing application data to other VMs and disks in other availability zones, if you use them.

### Test for zone failures

You can't directly simulate zone failures at the disk level, but you can use Azure Chaos Studio's support for [simulating zone-down events in virtual machine scale sets](/azure/chaos-studio/chaos-studio-fault-library#virtual-machine-scale-set) and [simulating the loss of an individual virtual machine](/azure/chaos-studio/chaos-studio-fault-library#virtual-machines-service-direct).

You should test your application's resilience to zone failures and managed disk behavior during outages. Monitor disk performance during simulated zone outages, and validate that your applications handle increased latency appropriately. Implement automated testing scenarios that verify your applications can handle temporary I/O delays and force detach operations for shared disks.

## Resilience to region-wide failures

Azure Disk Storage is a single-region service that operates within the boundaries of a specific Azure region. The service doesn't provide native multi-region capabilities or automatic failover between regions. If a region becomes unavailable, managed disk resources in that region are also unavailable.

### Custom multi-region solutions for resiliency

You can build a multi-region solution by deploying virtual machines and disks in each region, replicating or backing up data across regions, and failing over or restoring from backups when needed. You’re responsible for managing resources in every region, coordinating and synchronizing data, and handling failover or restoration. Some common approaches include:

- [Azure Site Recovery](/azure/site-recovery/azure-to-azure-tutorial-enable-replication), which provides cross-region replication of your virtual machines and disks.
- [Azure Backup](/azure/backup/disk-backup-overview) provides managed backup services, including for disks. You can use [cross-region restore](/azure/backup/backup-azure-arm-restore-vms#cross-region-restore) to restore VMs in another region.
- You can build your own snapshot-based solution by [copying disk snapshots of your disks across regions](/azure/virtual-machines/disks-copy-incremental-snapshot-across-regions).
- Some databases and applications provide replication approaches that work across regions, by replicating changes and managing clusters. For example, [SQL Server Always On Availability Groups](/sql/database-engine/availability-groups/windows/overview-of-always-on-availability-groups-sql-server) provides application-aware cross-region data protection with customizable consistency and failover behavior.

## Backup and restore

Azure managed disks support multiple backup approaches to protect against data loss and corruption. [Azure Disk Backup](/azure/backup/disk-backup-overview) is a native, cloud-based solution that automates snapshot lifecycle management. It provides crash-consistent, incremental backups with configurable retention policies. This agentless approach supports multiple backups per day without impacting application performance and integrates with Azure Backup Center for centralized management. You can [use incremental snapshots](/azure/virtual-machines/disks-incremental-snapshots) to reduce storage costs and backup times.

For [VM-level protection, Azure Backup](/azure/backup/backup-azure-vms-introduction) offers application-consistent backups for the entire virtual machine, including all attached disks. This approach is ideal when you need coordinated backup of multiple disks or application-aware backups. For database workloads, consider application-specific backup solutions that provide transaction-consistent protection and faster recovery options.

For critical workloads, implement a layered backup strategy combining Azure Disk Backup, cross-region snapshot replication, and application-level backups for transaction consistency. Configure backup policies based on your recovery requirements, compliance needs, and cost considerations.

## Service-level agreement

[!INCLUDE [Service-level agreement](includes/reliability-service-level-agreement-include.md)]

Azure Disk Storage doesn't provide its own availability SLA, but instead is included in the SLA for VMs. The configuration of your disk can affect the availability SLA of your VM.

### Related content

- [Azure managed disk types](/azure/virtual-machines/disks-types)
- [Backup and disaster recovery for Azure managed disks](/azure/virtual-machines/backup-and-disaster-recovery-for-azure-iaas-disks)
- [Best practices for achieving high availability with Azure virtual machines and managed disks](/azure/virtual-machines/disks-high-availability)
- [Redundancy options for managed disks](/azure/virtual-machines/disks-redundancy)
- [Azure reliability](/azure/reliability/overview)
