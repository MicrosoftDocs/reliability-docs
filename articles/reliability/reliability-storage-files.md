---
title: Reliability in Azure Files
description: Learn about resiliency in Azure Files, including resilience to transient faults, availability zone failures, and region failures.
author: khdownie
ms.author: kendownie
ms.topic: reliability-article
ms.custom: subject-reliability
ms.service: azure-file-storage
ms.date: 01/05/2026
#Customer intent: As an engineer responsible for business continuity, I want to understand who needs to understand the details of how Azure Files works from a reliability perspective and plan disaster recovery strategies in alignment with the exact processes that Azure services follow during different kinds of situations. 
---

# Reliability in Azure Files

This article describes reliability support in [Azure Files](/azure/storage/files/storage-files-introduction). Azure Files provides fully managed file shares in the cloud that are accessible via industry-standard Server Message Block (SMB) and Network File System (NFS) protocols.

[!INCLUDE [Shared responsibility description](includes/reliability-shared-responsibility-include.md)]

This article describes how to make Azure Files resilient to a variety of potential outages and problems, including transient faults, availability zone outages, and region outages. It also describes how you can use backups to recover from other types of problems, and highlights some key information about the Azure Files service level agreement (SLA).

> [!NOTE]
> Azure Files is part of the Azure Storage platform. Some of the capabilities of Azure Files are common across many Azure Storage services. In this article, we use *Azure Storage* to refer to these common capabilities.

## Production deployment recommendations

To learn how to deploy Azure Files to support your solution's reliability requirements and how reliability affects other aspects of your architecture, see [Architecture best practices for Azure Files](/azure/well-architected/service-guides/azure-files) in the Azure Well-Architected Framework.

## Reliability architecture overview

[Locally redundant storage (LRS)](/azure/storage/common/storage-redundancy#locally-redundant-storage) replicates the data within your storage accounts to one or more Azure availability zones located in the primary region of your choice. Although there's no option to choose your preferred availability zone, Azure might move or expand LRS accounts across zones to improve load balancing. There's no guarantee that your data is spread across zones. For more information about availability zones, see [What are Availability Zones?](./availability-zones-overview.md)

:::image type="complex" source="./media/reliability-storage/locally-redundant-storage.png" alt-text="Diagram that shows how data is replicated in availability zones by using LRS." lightbox="./media/reliability-storage/locally-redundant-storage.png" border="false":::
  A blue box represents the primary region. It contains a gray box that represents the datacenter. A dark purple box inside the datacenter box represents LRS. It contains a light purple box that includes the storage account and three icons labeled copy 1, copy 2, and copy 3.
:::image-end:::

Zone-redundant storage (ZRS), geo-redundant storage (GRS), and geo-zone-redundant storage (GZRS) provide extra protections. This article describes these options in detail.

Azure Files is available in two media tiers: 

- The **Premium tier** uses solid-state drives (SSD) for high performance. This tier is recommended for workloads that require low latency.

- The **Standard tier** supports hard disk drives (HDD). HDD file shares provide a cost-effective storage option for general purpose file shares.

For more information, see [Plan to deploy Azure Files - Storage tiers](/azure/storage/files/storage-files-planning#storage-tiers).

Azure Files implements redundancy at the storage account level, and file shares inherit that redundancy configuration automatically. The service supports multiple redundancy models that differ in their approach to data protection.

## Resilience to transient faults

[!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)]

To effectively manage transient faults when you use Azure Files, configure appropriate timeout values for your file operations based on file size and network conditions. Larger files require longer timeouts, while smaller operations can use shorter values to detect failures quickly.

To ensure that only secure connections are established to your NFS share, we recommend that you configure a private endpoint for your storage account. A private endpoint uses Azure Private Link to assign a static IP address to your storage account from within your virtual network's private address space. A private endpoint helps to prevent connectivity interruptions from dynamic IP address changes. For more information about security for your NFS shares, see [NFS file shares - Security and networking](/azure/storage/files/files-nfs-protocol#security-and-networking).

## Resilience to availability zone failures

[!INCLUDE [Resilience to availability zone failures](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)]

Azure Files provides two types of availability zone support:

- **Zone redundant storage (ZRS):** ZRS configurations automatically distribute your data across multiple availability zones within a region. Unlike LRS, ZRS guarantees that Azure synchronously replicates your file data across multiple availability zones. ZRS ensures that your data remains accessible even if one zone experiences an outage.

    :::image type="complex" source="./media/reliability-storage/zone-redundant-storage.png" alt-text="Diagram that shows how data is replicated in the primary region with zone-redundant storage (ZRS)." lightbox="./media/reliability-storage/zone-redundant-storage.png" border="false":::
        A blue box represents the primary region. It contains a dark purple box that represents ZRS. This box contains three white boxes that represent availability zone 1, availability zone 2, and availability zone 3. Each availability zone box contains a gray box that represents a datacenter. Each datacenter box contains a light purple box that includes the storage account and an icon labeled copy 1, copy 2, and copy 3.
    :::image-end:::

- **Zonal placement with LRS:** For premium storage accounts (SSD media tier), you can use zonal placement to select the specific availability zone in which your Azure Files storage account resides. You can use zonal placement if you need to place virtual machines (VMs) in the same zone to reduce latency between compute and storage.

  [!INCLUDE [Zonal resource description](includes/reliability-availability-zone-zonal-include.md)]

### Requirements

- **Region support:**

    - *ZRS:* ZRS is supported in:

      - *HDD (standard) file shares* in [all regions with availability zones](./regions-list.md).

      - *SSD (premium) file shares* through the `FileStorage` storage account kind. For a list of regions that support ZRS for SSD file share accounts, see [ZRS support for SSD file shares](/azure/storage/files/redundancy-premium-file-shares#zrs-support-for-ssd-azure-file-shares).

    - *LRS with zonal placement:* LRS with zonal placement is supported for SSD (premium) file shares in [supported regions](/azure/storage/files/zonal-placement#region-support).

- **File share types:**

    - *ZRS:* ZRS is supported by all file share types.

    - *LRS with zonal placement:* LRS with zonal placement is available for storage accounts that meet the following requirements:
      - Must use the premium storage tier (SSD media tier).
      - Classic Azure file shares only (use the Microsoft.Storage resource provider). You can't use zonal placement for file shares created with the Microsoft.FileShares resource provider.

### Cost

The cost impact is different depending on the type of availability zone support you use:

- *ZRS:* When you enable zone-redundant storage (ZRS), you pay at a different rate than locally redundant storage (LRS) because of the extra replication and storage overhead.

- *LRS with zonal placement:* LRS with zonal placement is charged at the same rate as LRS.

For detailed pricing information, see [Azure Files pricing](https://azure.microsoft.com/pricing/details/storage/files/).

### Configure availability zone support

- **Create a file share with availability zone support:**

  - *ZRS:* To create a new file share with ZRS, see [Create an Azure file share](/azure/storage/files/create-classic-file-share) and select **ZRS** or **GZRS** as the redundancy option during account creation.

  - *LRS with zonal placement:* To create a new file storage account with zonal placement, see [Create a new zonal storage account](/azure/storage/files/zonal-placement#create-a-new-zonal-storage-account).

- **Change replication type:**

  - *ZRS:* To convert an existing storage account to ZRS and learn about migration options and requirements, see [Change redundancy configuration for Azure Files](/azure/storage/files/files-change-redundancy-configuration?tabs=portal).

  - *LRS with zonal placement:* To pin an existing storage account to an Azure-selected zone, see [Pin an existing storage account to an Azure-selected zone](/azure/storage/files/zonal-placement#pin-an-existing-storage-account-to-an-azure-selected-zone).

- **Disable availability zone support:**

  - *ZRS:* Convert ZRS accounts back to a nonzonal configuration, such as LRS, through the same redundancy configuration change process.

  - *LRS with zonal placement:* To unpin a storage account from a zone and then convert the zonal storage account to a regional storage account, see [Unpin a storage account from a zone](/azure/storage/files/zonal-placement#unpin-a-storage-account-from-a-zone).

### Behavior when all zones are healthy

This section describes what to expect when a file storage account is configured for availability zone support and all availability zones are operational.

- **Traffic routing between zones:**

  - *ZRS:* Azure Storage with zone-redundant storage (ZRS) automatically distributes requests across storage clusters in multiple availability zones. Traffic distribution is transparent to applications and requires no client-side configuration.

  - *LRS with zonal placement:* Azure Storage with locally redundant storage (LRS) automatically distributes requests across storage clusters in the availability zone you selected. Traffic distribution is transparent to applications and requires no client-side configuration.

- **Data replication between zones:**
  
  - *ZRS:* All write operations to ZRS are replicated synchronously across all availability zones within the region. When you upload or modify data, the operation isn't considered complete until the data is successfully replicated across all of the availability zones. This synchronous replication ensures strong consistency and zero data loss during zone failures.

  - *LRS with zonal placement:* All write operations to LRS are replicated synchronously across multiple storage replicas within the zone. When you upload or modify data, the operation isn't considered complete until the data has been successfully replicated across all of the replicas.

### Behavior during a zone failure

This section describes what to expect when a file storage account is configured for availability zone support and there's an availability zone outage.

- **Detection and response:**

  - *ZRS:* Microsoft automatically detects zone failures and initiates recovery processes. No customer action is required for zone-redundant storage (ZRS) accounts. If a zone becomes unavailable, Azure undertakes networking updates such as Domain Name System (DNS) repointing.

  - *LRS with zonal placement:* You need to detect the loss of an availability zone. If necessary, you can initiate a failover to a secondary file share that you precreated in another availability zone.

[!INCLUDE [Resilience to availability zone failures (Service Health and Resource Health)](./includes/reliability-availability-zone-down-notification-service-resource-include.md)]

- **Active requests:**

  - *ZRS:* In-flight requests might be dropped during the recovery process and should be retried. Applications should [implement retry logic](#resilience-to-transient-faults) to handle these temporary interruptions.

  - *LRS with zonal placement:* In-flight requests are dropped and should be retried when the zone recovers.

- **Expected data loss:**

  - *ZRS*: No data loss occurs during zone failures because data is synchronously replicated across multiple zones before write operations complete.

  - *LRS with zonal placement:* Data on file shares in the affected zone is unavailable until the zone recovers.

- **Expected downtime:**

  - *ZRS:* A small amount of downtime, typically, a few seconds, might occur during automatic recovery as traffic is redirected to healthy zones. When you design applications for ZRS, follow practices for [transient fault handling](#resilience-to-transient-faults), including implementing retry policies with exponential back-off.

  - *LRS with zonal placement:* File shares in the affected zone remain down until the availability zone recovers.

- **Traffic rerouting:**

  - *ZRS:* Azure automatically reroutes traffic to the remaining healthy availability zones. The service maintains full functionality by using the surviving zones with no customer intervention required. No remounting of Azure file shares from the connected clients is required.

  - *LRS with zonal placement:* You're responsible for switching to other file storage accounts in healthy zones, if required.

### Zone recovery

Zone recovery behavior depends on the type of replication the file storage account uses:

- *ZRS:* When the failed availability zone recovers, Azure Storage automatically restores normal operations across all of the availability zones. The service automatically ensures data consistency by synchronizing any operations that occurred during the outage period.

- *LRS with zonal placement:* After the zone is healthy, file shares in the zone are available again. You're responsible for any zone recovery procedures and data synchronization that your workloads require.

### Test for zone failures

Zone-down testing options depend on the type of replication the file storage account uses:

- *ZRS:* When you use zone-redundant storage (ZRS), Azure Storage manages replication, traffic routing, and zone-down responses automatically. Because this feature is fully managed, you don't need to initiate or validate availability zone failure processes.

- *LRS with zonal placement:* There's no way to simulate an outage of the availability zone that contains your file storage account. However, you can manually configure upstream applications, firewalls, gateways or load balancers to redirect traffic to a different file storage account in a different availability zone.

## Resilience to region-wide failures

Azure Storage, including Azure Blob Storage, Azure Files, Azure Table Storage, and Azure Queue Storage, provides a range of geo-redundancy and failover capabilities to suit different requirements.

> [!IMPORTANT]
> Geo-redundant storage (GRS) only works within [Azure paired regions](/azure/reliability/regions-paired). If your storage account's region isn't paired, consider using the [custom multiregion solutions for resiliency](#custom-multiregion-solutions-for-resiliency).

### Geo-redundant storage for paired regions

Azure Storage provides several types of GRS in paired regions. Whichever type of GRS you use, the service always replicates data in the secondary region by using locally redundant storage (LRS). This approach provides protection against hardware failures within the secondary region.

- [GRS](/azure/storage/common/storage-redundancy#geo-redundant-storage) provides support for planned and unplanned failovers to the Azure paired region when there's an outage in the primary region. GRS asynchronously replicates data from the primary region to the paired region.

    :::image type="complex" source="./media/reliability-storage/geo-redundant-storage.png" alt-text="Diagram that shows how data is replicated by using GRS." lightbox="./media/reliability-storage/geo-redundant-storage.png" border="false":::
        Two blue boxes represent the primary region and the secondary region. They each contain a gray box that represents the datacenter. A dark purple box inside the datacenter box represents LRS. It contains a light purple box that includes the storage account and three icons labeled copy 1, copy 2, and copy 3. A dotted line that represents GRS encompasses the LRS boxes in both regions. An arrow labeled geo-replication points from the storage account in the primary region to the storage account in the secondary region.
    :::image-end:::

- [Geo-zone-redundant storage (GZRS)](/azure/storage/common/storage-redundancy#geo-zone-redundant-storage) replicates data in multiple availability zones in the primary region and into the paired region.

    :::image type="complex" source="./media/reliability-storage/geo-zone-redundant-storage.png" alt-text="Diagram that shows how data is replicated by using GZRS." lightbox="./media/reliability-storage/geo-zone-redundant-storage.png" border="false":::
        A blue box represents the primary region. It contains a dark purple box that represents ZRS. This box contains three white boxes that represent availability zone 1, availability zone 2, and availability zone 3. Each availability zone box contains a gray box that represents a datacenter. Each datacenter box contains a light purple box that includes the storage account and an icon labeled copy 1, copy 2, and copy 3. Another blue box represents the secondary region. That box contains a gray box that represents the datacenter. A dark purple box inside the datacenter box represents LRS. It contains a light purple box that includes the storage account and three icons labeled copy 1, copy 2, and copy 3. A dotted line that represents GZRS encompasses the ZRS and LRS boxes in both regions. An arrow labeled geo-replication points from ZRS in the primary region to the storage account in the secondary region.
    :::image-end:::

> [!IMPORTANT]
> Azure Files only supports geo-redundancy (GRS or GZRS) for standard (HDD) file shares. 
>
> Azure Files doesn't support read-access geo-redundant storage (RA-GRS) or read-access geo-zone-redundant storage (RA-GZRS). If a storage account is configured to use RA-GRS or RA-GZRS, the standard (HDD) file shares are configured and billed as GRS or GZRS.

#### Failover types

Azure Storage supports three types of failover for different scenarios.

- **Customer-managed unplanned failover:** You're responsible for initiating recovery if there's a region-wide storage failure in your primary region.

- **Customer-managed planned failover:** You're responsible for initiating recovery if another part of your solution has a failure in your primary region, and you need to switch your whole solution over to a secondary region. Use a planned failover when storage remains operational in the primary region, but you need to fail over your whole solution to a secondary region, such as for disaster recovery drills designed to ensure compliance and audit requirements.

- **Microsoft-managed failover:** In exceptional circumstances, Microsoft might initiate failover for all geo-redundant storage (GRS) accounts in a region. However, Microsoft-managed failover is a last resort and is expected to only be performed after an extended period of outage. You shouldn't rely on Microsoft-managed failover.

GRS accounts can use any of these failover types. You don't need to preconfigure a storage account to use any of the failover types ahead of time.

#### Requirements

- **Region support:** Azure Storage geo-redundant configurations use [Azure paired regions](./regions-paired.md) for secondary region replication. The secondary region is automatically determined based on your primary region selection and can't be customized. For a complete list of Azure paired regions, see [Azure regions list](./regions-list.md).

  If your storage account's region isn't paired, consider using the [custom multiregion solutions for resiliency](#custom-multiregion-solutions-for-resiliency).

- **Standard file shares only:** Azure Files only supports geo-redundancy (GRS or GZRS) for standard (HDD) file shares. Premium (SSD) file shares must use LRS or ZRS. If you have premium file shares and you want to replicate the data across regions for higher resiliency, see [Custom multiregion solutions for resiliency](#custom-multiregion-solutions-for-resiliency).

- **GRS and GZRS only:** Azure Files doesn't support read-access geo-redundant storage (RA-GRS) or read-access geo-zone-redundant storage (RA-GZRS). If a storage account is configured to use RA-GRS or RA-GZRS, the standard (HDD) file shares are configured and billed as GRS or GZRS.

#### Considerations

When you implement multiregion Azure Files, consider the following important factors:

- **Asynchronous replication latency:** Data replication to the secondary region is asynchronous, which means that there's a lag between when data is written to the primary region and when it becomes available in the secondary region. This lag can result in potential data loss if a primary region failure occurs before recent data is replicated. The data loss is measured by the recovery point objective (RPO). You can expect the replication lag to be less than 15 minutes, but this time is an estimate and not guaranteed.

  You can check the [Last Sync Time property](/azure/storage/common/last-sync-time-get) to understand how much data might be lost if your storage account has an unplanned failover.

- **Last Sync Time:** For Azure Files, the Last Sync Time is based on the latest system snapshot in the secondary region.

    The Last Sync Time calculation can time out if there are more than 100 file shares in a storage account. We recommend that you deploy 100 or fewer file shares for each storage account to avoid timeouts.

- **Secondary region access:** The secondary region isn't accessible for reads until a failover occurs.

- **Feature limitations:** Some Azure Files features aren't supported or have limitations when you use GRS or customer-managed failover. These limitations include specific file share types, access tiers, and management tools and operations. Review [feature compatibility documentation](/azure/storage/common/storage-disaster-recovery-guidance#unsupported-features-and-services) before you implement geo-redundancy.

#### Cost

Multi-region Azure Storage account configurations incur extra costs for cross-region replication and storage in the secondary region. Data transfer between Azure regions is charged based on standard inter-region bandwidth rates.

For detailed pricing information, see [Azure Files pricing](https://azure.microsoft.com/pricing/details/storage/files/).

#### Configure multiregion support

- **Create a new geo-redundant storage (GRS) account.** To create a GRS account, see [Create a storage account](/azure/storage/common/storage-account-create) and select GRS or geo-zone-redundant storage (GZRS) during account creation.

- **Enable geo-redundancy on an existing file storage account.** To convert an existing file storage account to GRS, see [Change redundancy configuration for Azure Files](/azure/storage/files/files-change-redundancy-configuration?tabs=portal).

  > [!WARNING]
  > After your account is reconfigured for geo-redundancy, it might take a significant amount of time before existing data in the new primary region is fully copied to the new secondary region.
  >
  > **To avoid a major data loss**, check the value of the [Last Sync Time property](/azure/storage/common/last-sync-time-get) before you initiate an unplanned failover. To evaluate potential data loss, compare the last sync time to the last time at which data was written to the new primary region.

- **Disable geo-redundancy.** Convert GRS accounts back to single-region configurations (LRS or ZRS) through the same redundancy configuration change process.

#### Behavior when all regions are healthy

This section describes what to expect when a storage account is configured for geo-redundancy and all regions are operational.

- **Traffic routing between regions:** Azure Files uses an active-passive approach where all read and write operations are directed to the primary region.

- **Data replication between regions:** Write operations are first committed to the primary region by using the configured redundancy type (LRS for GRS, or ZRS for GZRS). After successful completion in the primary region, data is asynchronously replicated to the secondary region, where it's stored by using LRS.

  The asynchronous nature of cross-region replication means that there's typically a lag time between when data is written to the primary region and when it's available in the secondary region. You can monitor the replication time by using the [Last Sync Time property](/azure/storage/common/last-sync-time-get).

#### Behavior during a region failure

This section describes what to expect when a storage account is configured for geo-redundancy and there's an outage in the primary region.

- **Customer-managed failover (unplanned):** Use an unplanned failover when storage in the primary region is unavailable.

  - **Detection and response:** In the unlikely event that your storage account is unavailable in your primary region, you can consider initiating a customer-managed unplanned failover. To make this decision, consider the following factors:

    - Whether [Azure Resource Health](/azure/service-health/resource-health-overview) shows problems accessing the storage account in your primary region

    - Whether Microsoft advises you to perform failover to another region

    > [!WARNING]
    > An unplanned failover can [result in data loss](/azure/storage/common/storage-disaster-recovery-guidance#anticipate-data-loss-and-inconsistencies). Before you initiate a customer-managed failover, decide whether the restoration of service justifies the risk of data loss.

  - **Notification:** [!INCLUDE [Region down notification partial (Service Health and Resource Health)](./includes/reliability-region-down-notification-service-resource-partial-include.md)]

  - **Active requests:** During the failover process, both the primary and secondary storage account endpoints become temporarily unavailable for both reads and writes. Any active requests might be dropped, and client applications need to retry after the failover completes.

  - **Expected data loss:** Data loss is common during an unplanned failover because of the asynchronous replication lag, which means that recent writes might not be replicated. You can check the [Last Sync Time property](/azure/storage/common/last-sync-time-get) to understand how much data might be lost during an unplanned failover. Expected data loss is often referred to as the recovery point objective (RPO). You can typically expect the RPO to be less than 15 minutes, but that time isn't guaranteed.

  - **Expected downtime:** The amount of expected downtime is often referred to as the recovery time objective (RTO). Customer-managed failover typically completes within 60 minutes, depending on the account size and complexity.

  - **Traffic rerouting:** As the failover completes, Azure automatically updates the storage account endpoints so that applications don't need to be reconfigured. If your application keeps Domain Name System (DNS) entries cached, it might be necessary to clear the cache to ensure that the application sends traffic to the new primary region.

  - **Post-failover configuration:** After an unplanned failover completes, your storage account in the destination region uses the locally redundant storage (LRS) tier. If you need to geo-replicate it again, you need to re-enable geo-redundant storage (GRS) and wait for the data to be replicated to the new secondary region.

  For more information about how to initiate customer-managed failover, see [How customer-managed (unplanned) failover works](/azure/storage/common/storage-failover-customer-managed-unplanned) and [Initiate a storage account failover](/azure/storage/common/storage-initiate-account-failover).

- **Customer-managed failover (planned):** Use a planned failover when storage remains operational in the primary region, but you need to fail over your whole solution to a secondary region for another reason. For example, another Azure service might be experiencing a problem and you need to switch to using a secondary region for your whole solution. Or you might use a planned failover to conduct a disaster recovery (DR) drill for compliance and audit purposes.

  - **Detection and response:** You're responsible for deciding to fail over. You typically make this decision if you need to fail over between regions, even though your storage account is healthy. For example, you might trigger a failover when there's a major outage of another application component that you can't recover from in the primary region.

  - **Notification:** [!INCLUDE [Region down notification partial (Service Health and Resource Health)](./includes/reliability-region-down-notification-service-resource-partial-include.md)]

  - **Active requests:** During the failover process, both the primary and secondary storage account endpoints become temporarily unavailable for both reads and writes. Any active requests might be dropped, and client applications need to retry after the failover completes.

  - **Expected data loss:** No data loss is expected because the failover process completes only after all data is synchronized, which results in an RPO of zero.

  - **Expected downtime:** Failover typically completes within 60 minutes, which means that the expected RTO is 60 minutes, depending on account size and complexity. During the failover process, both the primary and secondary storage account endpoints become temporarily unavailable for both reads and writes.

  - **Traffic rerouting:** As the failover completes, Azure automatically updates the storage account endpoints so that applications don't need to be reconfigured. If your application keeps DNS entries cached, it might be necessary to clear the cache to ensure that the application sends traffic to the new primary region.

  - **Post-failover configuration:** After a planned failover completes, your storage account in the destination region continues to be geo-replicated and remains on the GRS tier.

  For more information about how to initiate customer-managed failover, see [How customer-managed (planned) failover works](/azure/storage/common/storage-failover-customer-managed-planned) and [Initiate a storage account failover](/azure/storage/common/storage-initiate-account-failover).

- **Microsoft-managed failover:** In the rare event of a major disaster where Microsoft determines that the primary region is permanently unrecoverable, an automatic failover to the secondary region might be initiated. Microsoft handles the entire process and no customer action is required. The amount of time that elapses before failover occurs depends on the severity of the disaster and the time required to assess the situation.

  [!INCLUDE [Region down notification (Service Health and Resource Health)](includes/reliability-region-down-notification-service-resource-include.md)]

  > [!IMPORTANT]
  > Use customer-managed failover options to develop, test, and implement your DR plans. **Don't rely on Microsoft-managed failover**, which might only be used in extreme circumstances. A Microsoft-managed failover is likely initiated for an entire region. It can't be initiated for individual storage accounts, subscriptions, or customers. Failover might occur at different times for different Azure services. We recommend that you use customer-managed failover.

#### Region recovery

The failback process differs significantly between Microsoft-managed and customer-managed failover scenarios.

- **Customer-managed failover (unplanned):** After an unplanned failover, the storage account is configured with locally redundant storage (LRS). To fail back, you need to re-establish the geo-redundant storage (GRS) relationship and wait for the data to be replicated.

- **Customer-managed failover (planned):** After a planned failover, the storage account remains geo-replicated. You can initiate another customer-managed failover to fail back to the original primary region. [The same failover considerations apply](#behavior-during-a-region-failure).

- **Microsoft-managed failover:** If Microsoft initiates a failover, it's likely that a significant disaster occurred in the primary region, and the primary region might not be recoverable. Any timelines or recovery plans depend on the extent of the regional disaster and recovery efforts. You should monitor Azure Service Health communications for details.

#### Test for region failures

For GRS accounts, you can perform planned failover operations during maintenance windows to test the complete failover and failback process. Planned failover doesn't require data loss, but it does require downtime during both failover and failback.

### Custom multiregion solutions for resiliency

The cross-region failover capabilities of Azure Storage might be unsuitable because of the following reasons:

- Your storage account is in a nonpaired region.

- Your business uptime goals aren't satisfied by the recovery time or data loss that the built-in failover options provide.

- You need to fail over to a region that isn't your primary region's pair.

- You need an active/active configuration across regions.

- You use file share types that don't support geo-redundancy.

This section provides a high-level overview of some approaches to consider. A comprehensive overview of multiregion deployment topologies for Azure Storage is outside the scope of this article.

Consider the following common high-level approaches:

- **Multiple storage accounts:** Azure Files can be deployed across multiple regions by using separate storage accounts in each region. This approach provides flexibility in region selection, the ability to use nonpaired regions, and more granular control over replication timing and data consistency. When you implement multiple storage accounts across regions, you need to configure cross-region data replication, implement load balancing and failover policies, and ensure data consistency across regions.

- **Application-level replication:** Implement custom replication logic by using [Azure Data Factory](/azure/data-factory/introduction) or [AzCopy](/azure/storage/common/storage-use-azcopy-v10) to synchronize data between file shares in different regions. This approach requires custom development and conflict resolution mechanisms.

- **Use Azure File Sync to replicate files to a file share in another Azure region.** You can use [Azure File Sync](/azure/storage/file-sync/file-sync-introduction) to sync between an SMB Azure file share (*cloud endpoint*), an on-premises Windows file server, and a mounted file share that runs on a virtual machine (VM) in another Azure region (a *DR server endpoint*).

  This approach requires you to deploy multiple file shares and a VM to coordinate the synchronization process.

  If you use this approach for multiregion file replication:

  - Disable cloud tiering to ensure that all data is present locally on the file server.

  - Provision enough storage on the Azure VM to hold the entire dataset.

  - Access and modify files on the server endpoint, and not in Azure, to ensure that changes replicate quickly to the secondary region.

## Backup and restore

[Azure Files backup](/azure/backup/azure-file-share-backup-overview) is a native integration between Azure Files and Azure Backup that's designed to safeguard data against accidental deletion, corruption, and ransomware attacks.

Azure Files backup creates share-level snapshots stored within the same storage account. This capability enables the rapid recovery of both individual files and entire file shares. You can also use *backup policies* to provide long retention periods with customizable backup frequency.

You can create your snapshots and store them in two different ways:

- **Share-level storage:** For operational and short-term recovery scenarios, you can create share-level snapshots and store them within the same storage account. Share-level snapshots enable rapid recovery of individual files or entire file shares to either the original or an alternate location.

- **Vaulted backup storage:** By using vaulted backup, you can copy your daily snapshots to an Azure Recovery Services vault. To enhance security, this vault is isolated and air-gapped from the primary storage account.
  
  When you use a paired Azure region and configure the vault to use GRS, the vault replicates data to the paired region. This replication supports cross-region recovery and DR workflows.

## Service-level agreement

The service-level agreement (SLA) for Azure Storage describes the expected availability of the service and the conditions that must be met to achieve that availability expectation. The availability SLA you're eligible for depends on the storage tier and the replication type that you use. For more information, see [SLAs for Online Services](https://aka.ms/csla).

## Related content

- [Azure Files documentation](/azure/storage/files/storage-files-introduction)
- [Azure Files redundancy](/azure/storage/files/files-redundancy)
- [Plan for an Azure Files deployment](/azure/storage/files/storage-files-planning)
- [Azure reliability](overview.md)
