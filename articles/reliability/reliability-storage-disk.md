---
title: Reliability in Azure Disk Storage
description: Learn about reliability in Azure Disk Storage, including availability zones and multi-region deployments.
author: glynnniall
ms.author: glynnniall
ms.topic: reliability-article
ms.custom: subject-reliability
ms.service: azure-disk-storage
ms.date: 01/20/2026
ai-usage: ai-assisted
#Customer intent: As an engineer responsible for business continuity, I want to understand the details of how Azure Disk Storage works from a reliability perspective and plan disaster recovery strategies in alignment with the exact processes that Azure services follow during different kinds of situations.
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

Managed disks are designed for 99.999% availability and provide at least 99.999999999% (11 9’s) of durability. With managed disks, your data is replicated three times. If one of the three copies becomes unavailable, Azure automatically spawns a new copy of the data in the background. This ensures the persistence of your data and high fault tolerance.

This guide specifically focuses on managed disks, which reliably persist data. To learn more about the different disk roles, see [Disk roles](/azure/virtual-machines/managed-disks-overview#disk-roles).

By default, managed disks use [locally redundant storage (LRS)](/azure/storage/common/storage-redundancy#locally-redundant-storage). LRS keeps three copies of your disk data within a single datacenter, protecting against hardware failures such as drive or server rack issues.

Although LRS protects your disks against server rack and drive failures, it doesn't account for disasters such as fire or flooding within a datacenter. In the face of such disasters, all replicas of a disk configured to use LRS might be lost or unrecoverable. For higher levels of protection, use [zone-redundant storage (ZRS)](#resilience-to-availability-zone-failures), which replicates your disks across multiple availability zones.

For applications running on multiple VMs, multiple VMs have the highest availability SLA when distributed across three availability zones. For VMs and disks distributed across multiple availability zones, the disks and their parent VMs are respectively collocated in the same zone, which prevents multiple VMs from going down even if an entire zone experiences an outage.

When zones aren’t available or your workload is sensitive to inter-VM latency, deploy VMs and disks across multiple [fault domains](/azure/virtual-machines/availability-set-overview#fault-domains). Fault domains don’t provide zone redundancy, but they reduce the impact of hardware failures, network outages, or power interruptions. This prevents multiple VMs from failing if one storage fault domain goes down.

## Resilience to transient faults

[!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)]

Managed disks automatically recover from transient faults in the Azure infrastructure.

## Resilience to availability zone failures

[!INCLUDE [Resilience to availability zone failures](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)]

Managed disks support availability zones in both zone-redundant and zonal configurations:

- *Zone-redundant storage (ZRS)* synchronously replicates your data across multiple availability zones within a region. When you enable zone redundancy for a managed disk, Azure ensures that a failure in any single zone doesn’t affect data availability. If a zone goes down and your VM remains unaffected, your workloads continue running. However, if your VM is impacted and you need to recover before the outage is resolved, you can [force detach](/rest/api/compute/virtual-machines/attach-detach-data-disks?tabs=HTTP#diskdetachoptiontypes) your ZRS disks from the failed VM and attach them to another VM.

    ZRS disks can be shared between VMs to improve availability for clustered or distributed applications such as SQL FCI, SAP ASCS/SCS, or GFS2. You can attach a shared ZRS disk to primary and secondary VMs in different zones, taking advantage of both ZRS disks and VMs distributed across multiple [availability zones](/azure/reliability/availability-zones-overview). If the primary zone fails, you can quickly fail over to the secondary VM using [SCSI persistent reservation](/azure/virtual-machines/disks-shared-enable#supported-scsi-pr-commands).

- *Zonal LRS disks* reside in a specific availability zone and attach only to VMs in that zone. They replicate within the zone for high durability.

    Zonal LRS disks offer lower write latency than ZRS because replication happens inside one zone. For workloads that need very low disk latency, they’re a strong choice.

    A single zonal LRS disk and virtual machine don't provide zone resiliency. However, when you have a set of VMs that act together, such as a cluster or set of web servers, you can distribute an individual VM and its disks across multiple zones to achieve zone resiliency.

    [!INCLUDE [Zonal resource description](includes/reliability-availability-zone-zonal-include.md)]

If you don't configure availability zone support, your disk is *nonzonal* or *regional* and might be placed in any availability zone in the region. These disks are still considered LRS because they replicate within region. However, you should avoid configuring disks this way in production environments because nonzonal disks don't provide protection against availability zone outages.

### Requirements

- **Region support:**
    - *Zone-redundant:* For a list of regions that support ZRS managed disks, see [Redundancy options for managed disks](/azure/virtual-machines/disks-redundancy).
    - *Zonal:* Zonal LRS managed disks are supported in [all regions with availability zones](./regions-list.md).

- **Disk types:**
    - *Zone-redundant:* Zone-redundant disks are supported with Premium SSD and Standard SSD managed disks. ZRS isn't supported with Premium SSD v2, Ultra Disks, or Standard HDD managed disks.
    - *Zonal:* All managed disk types support zonal deployments.

### Considerations

When using ZRS for Azure managed disks, it's important to consider the performance implications. ZRS disks have slightly higher write latency compared to LRS disks due to synchronous replication across zones. This latency difference is typically minimal but may be noticeable for latency-sensitive workloads requiring sub-millisecond response times.

### Cost

ZRS incurs a higher cost than LRS due to the additional replication overhead and infrastructure required to maintain data across multiple zones. The exact cost difference varies by region and disk type. For detailed pricing information, see [Azure managed disk pricing](https://azure.microsoft.com/pricing/details/managed-disks/).

### Configure availability zone support

- **Create a new disk with availability zone support:**

    <!-- TODO Stefan: We need to include the steps for creating a new disk with AZ support. Can you add to the tutorial? -->
    To create a new managed disk with zonal LRS or ZRS redundancy, see [Tutorial - Manage Azure disks with the Azure CLI](/azure/virtual-machines/linux/tutorial-manage-disks) for Linux VMs, or [Tutorial: Manage disks with Azure PowerShell](/azure/virtual-machines/windows/tutorial-manage-data-disk) for Windows VMs.

    - *Zone-redundant:* Select a ZRS disk tier during disk creation.

    - *Zonal:* Select the availability zone during disk creation.

- **Change the availability zone configuration of an existing disk:**

    - *Zone-redundant:* You can convert an existing nonzonal (regional) disk to ZRS.
    
       While you can't convert a zonal LRS disk to ZRS, you can create a new ZRS disk from a snapshot. For step-by-step instructions, see [Convert a disk from LRS to ZRS](/azure/virtual-machines/disks-migrate-lrs-zrs) for step-by-step migration procedures and requirements.

    - *Zonal:* It's not possible to change the availability zone configuration of an existing zonal or zone-redundant disk. Instead, you need to create a new disk with the new configuration using a snapshot from the previous disk, and delete the old one.

- **Disable availability zone support:** It's not possible to change the availability zone configuration of an existing zonal or zone-redundant disk. Instead, you need to create a new disk with the new configuration using a snapshot from the previous disk, and delete the old one.

### Behavior when all zones are healthy

This section describes what to expect when managed disks are configured with availability zone support and all availability zones are operational.

- **Cross-zone operation:**

    - *Zone-redundant:* Azure automatically manages traffic routing between availability zones when you use a VM with a zone-redundant disk. During normal operations, requests are distributed across zones transparently.

    - *Zonal:* Traffic between a zonal VM and a zonal LRS disk in the same zone remains within the availability zone.

- **Cross-zone data replication:**

    - *Zone-redundant:* ZRS disks replicate every write synchronously across all availability zones in the region. A write operation completes only after data is stored in clusters in every zone. This approach provides strong consistency and immediate availability from any zone, but it can introduce slightly higher write latency compared to LRS disks.

    - *Zonal:* All write operations to LRS disks are replicated synchronously within the availability zone.

    To learn more about data replication for LRS and ZRS storage, see [Azure Storage redundancy](/azure/storage/common/storage-redundancy).

### Behavior during a zone failure

This section describes what to expect when a managed disk is configured for availability zone support and there's an availability zone outage.

 - **Detection and response:**

    - *Zone-redundant:* Some zone outages might affect only disks, only VMs, or both. The behavior you'll observe depends on whether the zone outage affects the VM attached to the disk.

        If the VM remains healthy but the disk is affected by the outage, your VM continues to operate. Microsoft automatically redirects disk operations to work against a replica in a healthy availability zone, and there's nothing you need to do.

        If the VM is down, you need to switch your workload to another VM in a different availability zone. If you've already created the secondary VM, you may have mounted the disk there. If the disk isn't attached to the active VM, you can *force detach* the disk from the failed VM and then attach it to the active VM. To perform a force detach:
            - Azure CLI: Use the [az vm disk detach command](/cli/azure/vm/disk?view=azure-cli-latest#az-vm-disk-detach) with the `--force-detach` argument.
            - Azure PowerShell: Use the [Remove-AzVMDataRisk cmdlet](/powershell/module/az.compute/remove-azvmdatadisk) with the `-ForceDetach` argument.

    - *Zonal:* You're responsible for detecting a zone outage, and for triggering a failover or another response. Typically, your response will involve switching your application traffic to a different VM in a different availability zone with its own zonal disk.

[!INCLUDE [Availability zone down notification (Service Health and Resource Health)](./includes/reliability-availability-zone-down-notification-service-resource-include.md)]

- **Expected data loss:**

    - *Zone-redundant:* No data loss occurs during zone failures because data is synchronously replicated across multiple zones before write operations complete.

    - *Zonal:* A zonal disk is unavailable until the availability zone recovers. In most scenarios, LRS replication means that your disk retains its data and the data can be recovered after the zone recovers. However, if a zone is permanently lost, any zonal disks in that zone are lost too.

- **Expected downtime:** Minimal downtime (typically a few seconds) may occur during automatic failover as I/O operations are redirected to healthy zones.

    > [!WARNING]
    > **Note to PG:** Please verify, and if it's accurate, how this would manifest in the VM (e.g. a freeze or a crash).

- **Redistribution:**

    - *Zone-redundant:* Azure automatically reroutes traffic to another copy of your disk in a healthy zone.

    - *Zonal:* A zonal disk is unavailable until the availability zone recovers.

### Zone recovery

When the failed availability zone recovers, managed disks recover automatically. The response process depends on the type of redundancy the disk uses:

- *Zone-redundant:* Azure automatically detects when the previously failed zone is healthy and begins redistributing I/O operations across all available zones. The service restores data synchronization to the recovered zone and resumes normal load balancing. No customer action is required, and the process is transparent to applications.

    If the VM attached to the disk has been affected by the outage, you might need to restart the VM to recover.

    During failback, you may experience slightly elevated latency for a brief period as the service rebalances I/O operations across all zones and ensures data consistency. The failback process typically completes within minutes, and disk performance returns to normal baseline levels.
    
    > [!WARNING]
    > **Note to PG:** Please verify the statement above.

- *Zonal:* If the VM attached to the disk has been affected by the outage, you might need to restart the VM to recover.

    You're responsible for resyncing data between other VMs and disks in other availability zones, if you use them.

### Test for zone failures

You can't directly simulate zone failures at the disk level, but you can use Azure Chaos Studio's support for [simulating zone-down events in virtual machine scale sets](/azure/chaos-studio/chaos-studio-fault-library#virtual-machine-scale-set) to simulate the loss of a virtual machine.

You should test your application's resilience to zone failures and managed disk behavior during outages. Test application resilience by monitoring disk performance during simulated zone outages. Validate that your applications handle increased latency appropriately. Implement automated testing scenarios that verify your applications can handle temporary I/O delays and force detach operations for shared disks.

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
