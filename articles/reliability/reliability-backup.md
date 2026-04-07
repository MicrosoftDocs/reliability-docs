---
title: Reliability in Azure Backup
description: Learn how to make Azure Backup resilient to a variety of potential outages and problems, including transient faults, availability zone outages, and region outages.
author: glynnniall
ms.author: pnp
ms.topic: reliability-article
ms.custom: subject-reliability
ms.service: azure-backup
ms.date: 02/23/2026
ai-usage: ai-assisted
---

# Reliability in Azure Backup

[Azure Backup](/azure/backup/backup-overview) is a built-in Azure service that securely protects cloud and on-premises workloads. Backup can scale its protection across multiple workloads and provides native integration with Azure workloads, including virtual machines (VMs), SAP HANA in Azure VMs, SQL in Azure VMs, Azure Files, Azure Blob Storage, Azure Data Lake Storage, Azure managed disks, Azure Elastic SAN volumes, and Azure Kubernetes Service (AKS). You don't need to manage automation or infrastructure, write scripts, or provision storage.

[!INCLUDE [Shared responsibility](includes/reliability-shared-responsibility-include.md)]

This article describes how Backup can be resilient to a variety of potential outages and problems, including transient faults, availability zone outages, and region outages. It also highlights some key information about the Backup service-level agreement (SLA).

> [!NOTE]
> This article describes how the Backup service itself is, or can be made, resilient to various problems. It doesn't explain how to use Backup to protect your VMs, data, or other assets. To learn about how to use Backup, see [Overview of Backup](/azure/backup/backup-overview).

## Production deployment recommendations

To back up your production workloads, we recommend that you set up your vault in the following ways:

> [!div class="checklist"]
> - Use zone-redundant storage (ZRS) as the minimum redundancy tier for your backups. ZRS replicates your backups across multiple availability zones so that you can restore your backups during an availability zone outage.
> - If you use geo-redundant storage (GRS) to replicate your backups to a paired Azure region, you should also turn on cross-region restore (CRR) for supported data sources. CRR lets you restore the backups into the paired region at any time.

The following sections of this article provide more detail about these configurations.

> [!NOTE]
> These storage redundancy recommendations apply to locations where backup copies are replicated, not to the Backup service or the resources that you back up. Backup protection and storage redundancy are complementary. Backups protect against data loss, and redundancy protects against infrastructure failures.

For a list of other recommendations for Backup, including reliability-focused recommendations, see [Backup cloud and on-premises workloads to cloud](/azure/backup/guidance-best-practices).

## Reliability architecture overview

[!INCLUDE [Introduction to reliability architecture overview section](includes/reliability-architecture-overview-introduction-include.md)]

### Logical architecture

Backup can back up and restore a variety of data sources. You set up backups differently depending on the data source that you work with. Common data sources include the following locations:

- Azure VMs
- Various databases
- Blob Storage accounts
- AKS clusters
- On-premises servers through the Microsoft Azure Recovery Services (MARS) agent

Backup stores your backed-up data in *vaults*. A vault is an online-storage entity in Azure that holds data, such as backup copies, recovery points, and backup policies. [Recovery Services vaults](/azure/backup/backup-azure-recovery-services-vault-overview) and [Backup vaults](/azure/backup/backup-vault-overview) are two types of vaults. You might use one or both types depending on what you need to protect. For a list of the data sources that each vault type supports, see [FAQ for Recovery Services vault](/azure/backup/backup-azure-backup-faq#what-are-the-various-vaults-supported-for-backup-and-restore-).

*Jobs* represent the activity of backing up or restoring your data. Backup jobs include scheduled or on-demand operations that copy your data from the source to the vault. Restore jobs include operations that recover your data from backup storage to a target location. Each job has a unique identifier and status tracking so that you can monitor the progress and troubleshoot problems that arise during backup and restore operations. You also create *backup policies* associated with jobs. Policies specify configuration like the backup schedule and how long you want to retain data.

Vaults store your backup policies and configuration along with metadata about jobs, which lets you track jobs and troubleshoot.

### Physical architecture

Microsoft manages the core Backup service infrastructure. This infrastructure is responsible for the management and operation of the service, including triggering and monitoring jobs.

Backup stores backups in the vault. Vaults are built on top of Azure Storage. Vaults automatically replicate your backup data, and the backup durability and resilience depend on the vault's storage redundancy.

- [Locally redundant storage (LRS)](/azure/storage/common/storage-redundancy?#locally-redundant-storage) replicates the data within your vault to one or more Azure availability zones located in the primary region of your choice. You can't choose your preferred availability zone, but Azure might move or expand LRS accounts across zones to improve load balancing. Your data isn't guaranteed to be spread across zones. For more information, see [Overview of availability zones](./availability-zones-overview.md).

- ZRS and GRS provide extra protections. This article describes these options in detail.

> [!NOTE]
> Some data sources support *operational tier* backups, which store data in another location rather than in the vault. For example, [Azure managed disks backup](/azure/backup/backup-managed-disks) and [AKS backups](/azure/backup/azure-kubernetes-service-cluster-backup) support operational tier backups, which are stored in disk snapshots. This article doesn't discuss operational tier backup storage, but you can apply the information about Backup.

## Resilience to transient faults

[!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)]

When you use Backup, both backup and restore workflows are resilient to intermittent failures. The service automatically retries when it encounters transient network faults or temporary service interruptions. You don't set up any retry logic. If you experience repeated faults, see [Troubleshoot Backup vault management operations](/azure/backup/backup-vault-troubleshoot).

## Resilience to availability zone failures

[!INCLUDE [Resilience to availability zone failures](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)]

Backup separately manages the availability zone configuration of the service and for your data.

- **Service:** Backup is automatically zone resilient in supported regions. However, this built-in zone resiliency doesn't apply to your backed-up data.

- **Backup storage redundancy:** Select the level of redundancy that you want for your backup data by configuring your Recovery Services vault or Backup vault. If you select ZRS, copies of your backup data are automatically stored across multiple availability zones in the Azure region that you use.

    If you don't use ZRS, your backup data is considered *nonzonal* and might be stored in any zone. If any zone in the region has a problem, nonzonal backup data might be unavailable.

:::image type="complex" source="./media/reliability-backup/zone-redundant.svg" alt-text="Diagram that shows the Backup core service, which is automatically zone-resilient, and zone-redundant backup storage." border="false" lightbox="./media/reliability-backup/zone-redundant.svg":::
   The diagram shows the zone-resilient architecture of Backup across three availability zones. Three columns represent availability zone 1, availability zone 2, and availability zone 3. A box labeled Backup core service spans all three zones. Below this box, the diagram shows a single row labeled ZRS that also spans all three availability zones. Below the ZRS row, another box spans all three availability zones. This box contains two cloud icons that represent a Backup vault and a Recovery Services vault.
:::image-end:::

### Requirements

- **Region support:** The service is automatically zone-resilient in [all regions that have availability zones](./regions-list.md). ZRS vaults are supported in the same regions.

- **New vaults only:** Set up ZRS on your vault before the first backup.

### Cost

When you turn on ZRS for your backups, you're charged at a different rate than LRS because of the extra replication and storage overhead. For more information, see [Backup pricing](https://azure.microsoft.com/pricing/details/backup/).

### Configure availability zone support

- **Create a new vault that uses ZRS:** Set up storage redundancy when you create a vault. You follow different steps depending on the vault type. For more information, see the following articles:

    - [Create and delete Backup vaults](/azure/backup/create-manage-backup-vault)
    - [Create and set up a Recovery Services vault](/azure/backup/backup-create-recovery-services-vault)

- **Set up ZRS on existing vaults:** For Backup vaults, set up storage redundancy when you create the vault. After you create a Backup vault, the setting is locked and you can't change it.

    For Recovery Services vaults, you must set up storage redundancy before you protect any workloads. After you protect a workload, the setting is locked and you can't change it.
    
    You can create a new vault configured to use ZRS and reassign your workloads to the new vault. However, this approach requires downtime. For more information, see [Modify default settings](/azure/backup/backup-create-recovery-services-vault#modify-default-settings). You're also responsible for manually deleting existing recovery points and other data because the old vault's retention policies no longer apply. For more information, see [Delete a Backup vault](/azure/backup/create-manage-backup-vault#delete-a-backup-vault) or [Delete a Recovery Services vault](/azure/backup/backup-azure-delete-vault#delete-protected-items-in-the-cloud).

### Behavior when all zones are healthy

This section describes what to expect when you configure vaults for ZRS, and all zones are operational.

- **Cross-zone operation:** Backup jobs run on infrastructure replicated across zones. Azure manages jobs from infrastructure in any zone.

- **Cross-zone data replication:** ZRS replicates backed-up data across zones. Replication occurs synchronously, which means that multiple zones acknowledge each write operation before it completes.

### Behavior during a zone failure

This section describes what to expect when you configure vaults for ZRS, and there's an outage in one of the zones.

- **Detection and response:** For the Backup service itself, Microsoft is responsible for detecting a failure in an availability zone and responding. You don't need to do anything to initiate a zone failover.

    > [!IMPORTANT]
    > For any data or resources that are unavailable because of the zone outage, you're responsible for detecting the outage and taking recovery actions, including restoring backups to a healthy zone.

[!INCLUDE [Availability zone down notification (Service Health and Resource Health)](./includes/reliability-availability-zone-down-notification-service-resource-include.md)]

- **Active requests:** The behavior of active jobs depends on which zone fails.

    - For any data sources in the failed availability zone, the zone failure makes the data sources unavailable. Active jobs might pause or fail.
    
    - For any data sources in healthy availability zones that run active jobs, a small amount of downtime, typically a few seconds, might occur while the platform switches to healthy availability zones for the Backup service.

- **Expected data loss:** The expected amount of data loss is also called the *recovery point objective* (RPO). The RPO for your backup data depends on multiple factors, including your backup schedule. In general, for a zone outage, no loss of backed-up data is expected because all data is replicated synchronously across zones.

- **Expected downtime:** The expected amount of downtime is also referred to as the *recovery time objective* (RTO). The RTO is different for each of the following scenarios:

    - For any data sources in the failed availability zone, the data sources might not be available until the zone recovers. Backup jobs might fail to run until the data source is available again. The RTO is undefined.

    - For any data sources in healthy availability zones, a small amount of downtime, typically a few seconds, might occur while the platform switches to healthy availability zones for the Backup service.

- **Redistribution:** Subsequent job runs automatically use infrastructure in healthy zones, as long as the data sources are available.

    You're responsible for restoring your backup to infrastructure in a healthy zone and for reconfiguring load balancers, clients, and other systems to redirect traffic to healthy infrastructure in the new zone.

### Zone recovery

When the availability zone recovers, Backup automatically restores operations in the availability zone and reroutes traffic between the zones as normal. Jobs continue to run and data remains available.

### Test for zone failures

The Backup platform manages traffic routing, data replication, failover, and failback. This feature is fully managed, so you don't need to initiate or validate availability zone failure processes.

## Resilience to region-wide failures

Backup supports geo-redundancy and failover through GRS and CRR.

> [!IMPORTANT]
> GRS for Backup only works within [paired Azure regions](/azure/reliability/regions-paired).

### Geo-redundant storage and CRR

To achieve regional redundancy for your backup data, use Backup to replicate your backups to an [Azure paired region](./regions-paired.md) by using [GRS](/azure/storage/common/storage-redundancy#geo-redundant-storage). GRS protects your backups from regional outages.

The region that you deploy your vault to is called the *primary region*. Your data sources must be located in the primary region. You can't configure backups to a vault in another region.

The paired region is also known as the *secondary region*.

:::image type="complex" source="./media/reliability-storage/geo-redundant-storage.png" alt-text="Diagram that shows how data is replicated by using GRS." lightbox="./media/reliability-storage/geo-redundant-storage.png" border="false":::
    Two boxes represent the primary region and the secondary region. They each contain a smaller box that represents the datacenter. A box inside the datacenter box represents LRS. It includes the storage account and three icons labeled copy 1, copy 2, and copy 3. A dotted line that represents GRS encompasses the LRS boxes in both regions. An arrow labeled geo-replication points from the storage account in the primary region to the storage account in the secondary region.
:::image-end:::

If you don't configure GRS and an outage occurs in the vault's region, you might be able access the vault and view backup items. However, without regional redundancy, the underlying backup data remains unavailable for restore operations.

#### CRR

When you configure GRS on a vault, Microsoft makes backups in the paired region available after an outage in the primary region occurs. If your data source supports [CRR](/azure/backup/backup-create-recovery-services-vault#set-cross-region-restore), you can restore from secondary region recovery points even when no outage occurs in the primary region. CRR also lets you run drills to assess resiliency against regional outages. When you turn on CRR, Microsoft upgrades your backup storage from GRS to read-access geo-redundant storage (RA-GRS).

#### Requirements

- **Region support:** GRS for Backup only works within [paired Azure regions](/azure/reliability/regions-paired).

- **New vaults only:** You must set up GRS on your vault before the first backup takes place.

#### Considerations

- **CRR:** After you turn on CRR, backup items can take up to 48 hours to be available in the secondary region.

#### Cost

GRS vaults incur extra costs for cross-region replication and storage in the secondary region. Data transfer between Azure regions is charged based on standard inter-region bandwidth rates. CRR is charged at a different rate because Microsoft upgrades your vault storage from GRS to RA-GRS. For more information, see [Backup pricing](https://azure.microsoft.com/pricing/details/backup/).

#### Configure multi-region support

- **Create a new vault that uses GRS and CRR:** When you create a vault, you should also set up storage redundancy. After you select GRS, you can optionally turn on CRR on the vault. The steps that you follow depend on the vault type. For more information, see the following articles:

    - [Create and delete Backup vaults](/azure/backup/create-manage-backup-vault)
    - [Create and set up a Recovery Services vault](/azure/backup/backup-create-recovery-services-vault)

- **Set up GRS and CRR on existing vaults:** For Backup vaults, you must set up storage redundancy when you create the vault.

    For Recovery Services vaults, you must set up storage redundancy before you protect any workloads. After a workload is protected, the setting is locked and you can't change it.

    You can turn on CRR on existing GRS vaults. After you turn on CRR, you can't turn it off.

#### Behavior when all regions are healthy

This section describes what to expect when you configure vaults to use geo-redundant storage and all regions are operational.

- **Cross-region operation:** Backups are always completed in the primary region, which is the region where the vault and data source are deployed.

- **Cross-region data replication:** When you set up the vault to use GRS, backups are first committed to the primary region by using LRS. After successful completion in the primary region, data is asynchronously replicated to the secondary region. The secondary region uses LRS to store data. The backup data can take up to 12 hours to replicate from the primary region to the secondary region.

#### Behavior during a region failure

This section describes what to expect when you configure vaults to use geo-redundant storage and an outage occurs in the primary region.

- **Detection and response:** For data sources that support [CRR](/azure/backup/backup-support-matrix#cross-region-restore) and where CRR is set up on the vault, you can initiate your own CRR to the paired region at any time, including during a region outage or disaster. You're responsible for detecting the outage and taking recovery actions, including restoring backups to a healthy region.

    For all other scenarios, the data replicated to the secondary region is available to restore in the secondary region only if Azure declares a disaster in the primary region. Microsoft is responsible for declaring a disaster. The amount of time it takes to declare a disaster depends on the severity of the incident and the time required to assess the situation. Microsoft typically declares a disaster only after an extended period of time.

[!INCLUDE [Region down notification (Service Health and Resource Health)](./includes/reliability-region-down-notification-service-resource-include.md)]

- **Expected data loss:** The RPO for your backup data depends on multiple factors, including your backup schedule. In general, for a region outage, expect up to 36 hours of data loss because the RPO in the primary region is 24 hours, and it can take up to 12 hours to replicate the backup data from the primary to secondary region.

- **Expected downtime:** The RTO is different for each of the following scenarios:

    - Data sources and other resources in the failed region might not be available until the region recovers, so the RTO is undefined.

    - Backup might not be able to perform backup or restore operations in the failed region until the region recovers, so the RTO is undefined.

    - If you use CRR, the RTO for initiating the restoration of backups already replicated to the paired region is zero. If you don't use CRR, the RTO depends on how long it takes for Microsoft to declare a disaster in the failed region.

- **Redistribution:** No backup jobs can run while the primary region is offline. You can restore data in the vault, but you can't add new data.

    You're responsible for restoring your backup to infrastructure in the paired region and for reconfiguring load balancers, clients, and other systems to redirect traffic to healthy infrastructure in the paired region.

#### Region recovery

When the primary region recovers, Backup automatically restores operations in the region. Jobs resume and data remains available.

#### Test for region failures

You can use [CRR](/azure/backup/backup-create-recovery-services-vault#set-cross-region-restore) to perform a restore operation to the paired region. Use this approach to verify restore and other recovery processes.

## Resilience to service maintenance

Backup provides two key recovery features to prevent accidental or malicious deletion of your backup data:

- **Soft delete** allows you to recover deleted objects and vaults during a configurable retention period. By default, this period is 14 days, but you can edit it. Think of soft delete as a recycle bin for your backups and vaults. For more information, see [Secure by default with soft delete for Backup](/azure/backup/secure-by-default).

- **Immutable vaults** can help you protect your backup data by blocking operations that might lead to loss of recovery points. You can lock the immutable vault setting to make it irreversible. You can also use write once, read many (WORM) storage for backups to prevent malicious actors from disabling immutability and deleting backups. For more information, see [Immutable vault for Backup](/azure/backup/backup-azure-immutable-vault-concept).

## Service-level agreement

[!INCLUDE [Service-level agreement](includes/reliability-service-level-agreement-include.md)]

The Backup SLA covers the availability of the service for both backup and restore operations. To be covered by the SLA, you need to retry failed backup or restore jobs at least once every thirty minutes.
