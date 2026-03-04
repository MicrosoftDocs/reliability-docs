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

[Azure Backup](/azure/backup/backup-overview) is a built-in Azure service that securely protects cloud and on-premises workloads. Azure Backup can seamlessly scale its protection across multiple workloads and provides native integration with Azure workloads, including virtual machines (VMs), SAP HANA in Azure VM, SQL in Azure VMs, Azure Files, Azure Blob, Azure Data Lake, Disks, Elastic SAN Volumes, AKS, and more. You don't need to manage automation or infrastructure. There’s no need to write scripts or provision storage.

[!INCLUDE [Shared responsibility](includes/reliability-shared-responsibility-include.md)]

This article describes how Azure Backup can be resilient to a variety of potential outages and problems, including transient faults, availability zone outages, and region outages. It also highlights some key information about the Azure Backup service level agreement (SLA).

> [!NOTE]
> This document describes how the Azure Backup service itself is, or can be made, resilient to various issues. It doesn't explain how to use Azure Backup to protect your VMs, data, or other assets. To learn about how to use Azure Backup, see [What is the Azure Backup service?](/azure/backup/backup-overview).

## Production deployment recommendations for reliability

For the backup of your production workloads, we recommend that you configure your vault as follows:

> [!div class="checklist"]
> - Use zone-redundant storage (ZRS) as the minimum redundancy tier for your backups. Zone-redundant storage replicates your backups across multiple availabilty zones, so that you can restore your backups during an availability zone outage.
> - If you use geo-redundant storage (GRS) to replicate your backups to a paired Azure region, you should also enable cross-region restore for supported datasources. Cross-region restore means that you can restore the backups into the paired region at any time.

The remaining sections of this article provides more detail on these configurations.

> [!NOTE]
> These storage redundancy recommendations apply to where backup copies are replicated, not to the Azure Backup service or the resources that you back up. Backup protection and storage redundancy are complementary - backups protect against data loss, while redundancy protects against infrastructure failures.

For a list of other recommendations for Azure Backup, including reliability-focused recommendations, see [Backup cloud and on-premises workloads to cloud](/azure/backup/guidance-best-practices).

## Reliability architecture overview

[!INCLUDE [Introduction to reliability architecture overview section](includes/reliability-architecture-overview-introduction-include.md)]

### Logical architecture

Azure Backup can back up and restore a variety of *datasources*. You configure backups differently depending on the datasource you're working with. Common datasources include:

- Azure virtual machines (VMs)
- Various databases
- Azure Blob accounts
- AKS clusters
- On-premises servers through the Microsoft Azure Recovery Services (MARS) agent

Azure Backup stores your backed-up data in *vaults*. A vault is an online-storage entity in Azure that's used to hold data, such as backup copies, recovery points, and backup policies. There are two types of vaults: [Recovery Services vaults](/azure/backup/backup-azure-recovery-services-vault-overview) and [Backup vaults](/azure/backup/backup-vault-overview). You might use one or both types of vault depending on what you need to protect. To see a list of the datasources that each vault type supports, see [What are the various vaults supported for backup and restore?](/azure/backup/backup-azure-backup-faq#what-are-the-various-vaults-supported-for-backup-and-restore-).

*Jobs* represent the activity of backing up or restoring your data. Backup jobs include scheduled or on-demand operations that copy your data from the source to the vault. Restore jobs include operations that recover your data from backup storage to a target location. Each job has a unique identifier and status tracking, enabling you to monitor the progress and troubleshoot any issues that arise during backup and restore operations. You also create *backup policies* that are associated with jobs. Policies specify configuration like the backup schedule and how long you want to retain data for.

Vaults store your backup policies and configuration, and they also store metadata about jobs, which enables tracking and troubleshooting.

### Physical architecture

The core Azure Backup service infrastructure is managed by Microsoft. This infrastructure is responsible for management and operation of the service, including triggering and monitoring jobs.

Azure Backup service stores backups in the vault. Vaults are built on top of Azure Storage. Vaults automatically replicate your backup data, and the backup durability and resilience depend on the vault's storage redundancy:

- [Locally redundant storage (LRS)](/azure/storage/common/storage-redundancy?#locally-redundant-storage) replicates the data within your vault to one or more Azure availability zones located in the primary region of your choice. Although there's no option to choose your preferred availability zone, Azure may move or expand LRS accounts across zones to improve load balancing. There's no guarantee that your data will be spread across zones. For more information about availability zones, see [What are Availability Zones?](./availability-zones-overview.md).

- Zone-redundant storage (ZRS) and geo-redundant storage (GRS) provide extra protections. This article describes these options in detail.

> [!NOTE]
> Some datasources support *operational tier* backups, which store data in another location rather than in the vault. For example, [Azure Disk backup](/azure/backup/backup-managed-disks) and [AKS backups](/azure/backup/azure-kubernetes-service-cluster-backup) support operational tier backup, which are stored in disk snapshots. This article doesn't discuss operational tier backup storage, but the information about the Azure Backup service remains applicable.

## Resilience to transient faults

[!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)]

When you use Azure Backup, both backup and restore workflows are resilient to intermittent failures. The service automatically retries when it encounters transient network faults or temporary service interruptions. You don't configure any retry logic. If you experience repeated faults, consult [troubleshooting documentation](/azure/backup/backup-vault-troubleshoot).

## Resilience to availability zone failures

[!INCLUDE [Resilience to availability zone failures](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)]

Azure Backup separately manages the availability zone configuration of the service and for your data.

- **Service:** The Azure Backup service itself is automatically zone-resilient in supported regions, with no action required from you. However, this built-in zone resiliency doesn't apply to your backed-up data.

- **Backup storage redundancy:** Select the level of redundancy you want for your backup data by configuring your Recovery Services vault or Backup vault. If you select zone-redundant storage (ZRS), copies of your backup data are automatically stored across multiple availability zones in the Azure region you use.

    If you don't use ZRS, your backup data is considered to be *nonzonal* and might be stored in any zone. If any zone in the region has a problem, nonzonal backup data might be unavailable.

:::image type="content" source="./media/reliability-backup/zone-redundant.svg" alt-text="Diagram that shows Azure Backup core service, which is automatically zone-resilient, and zone-redundant backup storage." border="false":::

### Requirements

- **Region support:** The service is automatically zone-resilient in [all regions with availability zones](./regions-list.md). ZRS vaults are supported in the same regions.

- **New vaults only:** Configure ZRS on your vault before the first backup.

### Cost

When you enable zone-redundant storage (ZRS) for your backups, you're charged at a different rate than locally redundant storage (LRS) because of the extra replication and storage overhead. For more information, see [Azure Backup pricing](https://azure.microsoft.com/pricing/details/backup/).

### Configure availability zone support

- **Create a new vault that uses ZRS:** When you create a vault, you should also configure the storage redundancy. The steps you follow are different depending on the vault type. For more information, see:
    - [Create and delete Backup vaults](/azure/backup/create-manage-backup-vault)
    - [Create and configure a Recovery Services vault](/azure/backup/backup-create-recovery-services-vault)

- **Configure ZRS on existing vaults:** For Backup vaults, configure storage redundancy when you create the vault. Once a Backup vault is created, the setting is locked and can't be changed.

    For Recovery Services vaults, storage redundancy must be configured *before* protecting any workloads. Once a workload is protected, the setting is locked and can't be changed.
    
    You can create a new vault configured to use ZRS and reassign your workloads to the new vault. However, this approach requires downtime, and you're responsible for manually deleting manually deleting any existing recovery points and other data because the old vault's retention policies no longer apply. For more information about deleting a vault, see [Delete a Backup vault](/azure/backup/create-manage-backup-vault#delete-a-backup-vault) or [Delete a Recovery Services vault](/azure/backup/backup-azure-delete-vault#delete-protected-items-in-the-cloud).

### Behavior when all zones are healthy

When vaults use zone-redundant storage and all availability zones are operational, expect the following:

- **Cross-zone operation:** Backup jobs run on infrastructure that is replicated across zones. Azure manages jobs from infrastructure in any zone.

- **Cross-zone data replication:** ZRS replicates backed up data across zones. Replication occurs synchronously, which means that multiple zones acknowledge each write operation before it's completed.

### Behavior during a zone failure

When vaults are configured to use zone-redundant storage and an availability zone outage occurs, expect the following:

- **Detection and response:** For the Azure Backup service itself, Microsoft is responsible for detecting a failure in an availability zone and responding. You don't need to do anything to initiate a zone failover.

    > [!IMPORTANT]
    > For any data or resources that are unavailable due to the zone outage, you're responsible for detecting the outage and taking any recovery actions, including restoring backups to a healthy zone.

[!INCLUDE [Availability zone down notification (Service Health and Resource Health)](./includes/reliability-availability-zone-down-notification-service-resource-include.md)]

- **Active requests:** The behavior of any active jobs depend on which zone has failed.

    - For any datasources in the failed availability zone, the datasources are likely to be unavailable due to the zone failure. Any active jobs might pause or fail.
    - For any datasources in healthy availability zones that are running active jobs, a small amount of downtime, typically a few seconds, might occur while the platform switches to using healthy availability zones for the Azure Backup service.

- **Expected data loss:** The amount of data loss you can expect is also referred to as the recovery point objective (RPO). The RPO for your backup data depends on multiple factors, including your backup schedule. In general, for a zone outage no loss of backed-up data is expected because all data is replicated synchronously across zones.

- **Expected downtime:** The amount of downtime you can expect is also referred to as the recovery time objective (RTO). The RTO is different for each of the following scenarios:

    - For any datasources in the failed availability zone, the datasources might not be available until the zone recovers. Backup jobs might fail to run until the datasource is available again. The RTO is undefined.
    - For any datasources in healthy availability zones, a small amount of downtime, typically a few seconds, might occur while the platform switches to using healthy availability zones for the Azure Backup service.

- **Redistribution:** Any subsequent job executions automatically use infrastructure in healthy zones, as long as the datasources are available.

    You're responsible for restoring your backup to infrastructure in a healhty zone, and for reconfiguring any load balancers, clients, and other systems to redirect traffic to healthy infrastructure in the new zone.

### Zone recovery

When the availability zone recovers, Azure Backup automatically restores operations in the availability zone and reroutes traffic between the zones as normal. Jobs continue to run and data remains available.

### Test for zone failures

The Azure Backup platform manages traffic routing, data replication, failover, and failback. This feature is fully managed, so you don't need to initiate or validate availability zone failure processes.

## Resilience to region-wide failures

Azure Backup supports geo-redundancy and failover through geo-redundant storage (GRS) and cross-region restore.

> [!IMPORTANT]
> GRS for Azure Backup only works within [paired Azure regions](/azure/reliability/regions-paired).

### Geo-redundant storage and cross-region restore

To achieve regional redundancy for your backup data, Azure Backup allows you to replicate your backups to an [Azure paired region](./regions-paired.md) by using [geo-redundant storage (GRS)](/azure/storage/common/storage-redundancy#geo-redundant-storage). GRS protects your backups from regional outages.

The region you deploy your vault to is called the *primary region*. Your datasources must be located in the primary region. You can't configre backups to a vault in another region.

The paired region is also referred to as the *secondary region*.

:::image type="complex" source="./media/reliability-storage/geo-redundant-storage.png" alt-text="Diagram that shows how data is replicated by using GRS." lightbox="./media/reliability-storage/geo-redundant-storage.png" border="false":::
    Two blue boxes represent the primary region and the secondary region. They each contain a gray box that represents the datacenter. A dark purple box inside the datacenter box represents LRS. It contains a light purple box that includes the storage account and three icons labeled copy 1, copy 2, and copy 3. A dotted line that represents GRS encompasses the LRS boxes in both regions. An arrow labeled geo-replication points from the storage account in the primary region to the storage account in the secondary region.
:::image-end:::

If you don’t configure GRS, an outage in the vault’s region might still allow you to access the vault and view backup items. However, without regional redundancy, the underlying backup data remains unavailable for restore operations.

#### Cross-region restore

When you configure GRS on a vault, Microsoft makes backups in the paired region available after declaring an outage in the primary region. If your datasource supports enabling [cross-region restore](/azure/backup/backup-create-recovery-services-vault#set-cross-region-restore), you can restore from secondary region recovery points even when no outage occurs in the primary region. Cross-region restore also lets you run drills to assess your resiliency against regional outages. Enabling cross-region restore upgrades your backup storage from GRS to read-access geo-redundant storage (RA-GRS).

#### Requirements

- **Region support:** GRS for Azure Backup only works within [paired Azure regions](/azure/reliability/regions-paired).

- **New vaults only:** GRS must be configured on your vault before the first backup takes place.

#### Considerations

- **Cross-region restore:** After you turn on cross-regon restore, it can take up to 48 hours for the backup items to be available in the secondary region.

#### Cost

GRS vaults incur extra costs for cross-region replication and storage in the secondary region. Data transfer between Azure regions is charged based on standard inter-region bandwidth rates. Cross-region restore is charged at a different rate because Microsoft upgrades your vault storage from GRS to RA-GRS. For more information, see [Azure Backup pricing](https://azure.microsoft.com/pricing/details/backup/).

#### Configure multi-region support

- **Create a new vault that uses GRS and CRR:** When you create a vault, you should also configure the storage redundancy. After selecting GRS, you can optionally enable CRR on the vault. The steps you follow are different depending on the vault type. For more information, see:
    - [Create and delete Backup vaults](/azure/backup/create-manage-backup-vault)
    - [Create and configure a Recovery Services vault](/azure/backup/backup-create-recovery-services-vault)

- **Configure GRS and CRR on existing vaults:** For Backup vaults, storage redundancy must be configured when you create the vault.

    For Recovery Services vaults, storage redundancy must be configured *before* protecting any workloads. Once a workload is protected, the setting is locked and can't be changed.

    You can enable CRR on existing GRS vaults. You can't disable CRR once it's enabled.

#### Behavior when all regions are healthy

This section describes what to expect when vaults are configured to use geo-redundant storage and all regions are operational.

- **Cross-region operation:** Backups are always completed in the primary region, which is the region where the vault and datasource are deployed.

- **Cross-region data replication:** When the vault is configured to use GRS, backups are first committed to the primary region by using locally redundant storage (LRS). After successful completion in the primary region, data is asynchronously replicated to the secondary region where it's stored by using LRS. It can take up to 12 hours to replicate the backup data from the primary region to the secondary region.

#### Behavior during a region failure

This section describes what to expect when vaults are configured to use geo-redundant storage and an outage occurs in the primary region.

- **Detection and response:** For datasources that support [cross-region restore](/azure/backup/backup-support-matrix#cross-region-restore) and where cross-region restore is enabled on the vault, you can initiate your own cross-region restore to the paired region at any time, including during a region outage or disaster. You're responsible for detecting the outage and taking any recovery actions, including restoring backups to a healthy region.

    For all other scenarios, the data that is replicated to the secondary region is available to restore in the secondary region only if Azure declares a disaster in the primary region. Microsoft is responsible for declaring such a disaster. The amount of time it takes to declare a disaster depends on the severity of the incident and the time required to assess the situation. It would likely be performed only after an extended period of time.

[!INCLUDE [Region down notification (Service Health and Resource Health)](./includes/reliability-region-down-notification-service-resource-include.md)]

- **Expected data loss:** The RPO for your backup data depends on multiple factors, including your backup schedule. In general, for a region outage you should expect up to 36 hours of data loss, because the RPO in the primary region is 24 hours, and it can take up to 12 hours to replicate the backup data from the primary to secondary region.

- **Expected downtime:** The RTO is different for each of the following scenarios:

    - Datasources and other resources in the failed region might not be available until the region recovers, so the RTO is undefined.
    - Azure Backup might not be able to perform backup or restore operations in the failed region until the region recovers, so the RTO is undefined.
    - If you use cross-region restore, the RTO for initiating the restoration of backups that are already replicated to the paired region is zero. If you don't use cross-region restore, the RTO depends on how long it takes for Microsoft to declare a disaster in the failed region.

    No backup jobs can run while the primary region is offline. You can restore data in the vault but not add new data.

- **Redistribution:** No backup jobs can run while the primary region is offline. You can restore data in the vault but not add new data.

    You're responsible for restoring your backup to infrastructure in the paired region, and for reconfiguring any load balancers, clients, and other systems to redirect traffic to healthy infrastructure in the paired region.

#### Region recovery

When the primary region recovers, Azure Backup automatically restores operations in the region. Jobs resume and data remains available.

#### Testing for region failures

You can use [cross-region restore](/azure/backup/backup-create-recovery-services-vault#set-cross-region-restore) to perform a restore operation to the paired region. You can use this approach to verify your restore and other recovery processes.

## Resilience to loss of backup data

Azure Backup provides two key recovery features to prevent accidental or malicious deletion of your backup data:

- **Soft delete** allows you to recover deleted objects and vaults during a configurable retention period. By default, this period is 14 days, but you can configure it. Think of soft delete like a recycle bin for your backups and vaults. For more information, see [Secure by default with soft delete for Azure Backup](/azure/backup/secure-by-default).

- **Immutable vaults** can help you protect your backup data by blocking any operations that could lead to loss of recovery points. You can lock the immutable vault setting to make it irreversible. You can also use WORM (write once, read many) storage for backups to prevent any malicious actors from disabling immutability and deleting backups. For more information, see [Immutable vault for Azure Backup](/azure/backup/backup-azure-immutable-vault-concept).

## Service-level agreement

[!INCLUDE [Service-level agreement](includes/reliability-service-level-agreement-include.md)]

The Azure Backup SLA covers the availability of the service for both backup and restore operations. In order to be covered by the SLA, you need to retry failed backup or restore jobs no less frequently than once every thirty minutes.
