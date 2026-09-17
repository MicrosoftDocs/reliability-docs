---
title: Reliability in Azure Blob Storage
description: Learn about resiliency in Azure Blob Storage, including resilience to transient faults, availability zone failures, and region failures.
ms.author: shaas
author: stevenmatthew
ms.topic: reliability-article
ms.custom: subject-reliability
ms.service: azure-blob-storage
ms.date: 08/15/2025
#Customer intent: As an engineer responsible for business continuity, I want to understand the details of how Azure Blob Storage works from a reliability perspective and plan disaster recovery strategies in alignment with the exact processes that Azure services follow during different kinds of situations.
---

# Reliability in Azure Blob Storage

[Azure Blob Storage](/azure/storage/blobs/storage-blobs-overview) is an object storage solution for the cloud from Microsoft. It's designed to store massive amounts of unstructured data such as text, binary data, documents, media files, and application backups. As a foundational Azure storage service, Blob Storage provides multiple reliability features to ensure that your data remains available and durable during both planned and unplanned events.

[!INCLUDE [Shared responsibility](includes/reliability-shared-responsibility-include.md)]

This article describes how to make Blob Storage resilient to a variety of potential outages and problems, including transient faults, availability zone outages, and region outages. It also describes how you can use backups to recover from other types of problems, and highlights some key information about the Blob Storage service level agreement (SLA).

> [!NOTE]
> Blob Storage is part of the Azure Storage platform. Some of the capabilities of Blob Storage are common across many Azure Storage services. In this article, we use *Azure Storage* to refer to these features.

## Production deployment recommendations

To learn about how to deploy Blob Storage to support your solution's reliability requirements, and how reliability affects other aspects of your architecture, see [Architecture best practices for Blob Storage](/azure/well-architected/service-guides/azure-blob-storage) in the Azure Well-Architected Framework.

## Reliability architecture overview

Azure Storage provides several redundancy options to help you protect your data against different types of failures. Each option provides a specific level of data redundancy, so you can choose the level that best matches your application's requirements.

[Locally redundant storage (LRS)](/azure/storage/common/storage-redundancy#locally-redundant-storage) replicates the data within your storage accounts to one or more Azure availability zones located in the primary region of your choice. Although there's no option to choose your preferred availability zone, Azure might move or expand LRS accounts across zones to improve load balancing. There's no guarantee that your data is spread across zones. For more information about availability zones, see [What are Availability Zones?](./availability-zones-overview.md)

:::image type="complex" source="./media/reliability-storage/locally-redundant-storage.png" alt-text="Diagram that shows how data is replicated in availability zones by using LRS." lightbox="./media/reliability-storage/locally-redundant-storage.png" border="false":::
  A blue box represents the primary region. It contains a gray box that represents the datacenter. A dark purple box inside the datacenter box represents LRS. It contains a light purple box that includes the storage account and three icons labeled copy 1, copy 2, and copy 3.
:::image-end:::

Zone-redundant storage (ZRS), geo-redundant storage (GRS), and geo-zone-redundant storage (GZRS) provide extra protections. This article describes these options in detail.

## Resilience to transient faults

[!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)]

To effectively manage transient faults when you use Blob Storage, implement the following recommendations:

- **Use the Azure Storage client libraries**, which include built-in retry policies with exponential backoff and jitter. The .NET, Java, Python, and JavaScript SDKs automatically handle retries for transient failures. For more information about retry configuration options, see [Implement a retry policy with .NET](/azure/storage/blobs/storage-retry-policy).

- **Configure appropriate timeout values** for your blob operations based on blob size and network conditions. Larger blobs require longer timeouts, but smaller operations can use shorter values to detect failures quickly.

## Resilience to availability zone failures

[!INCLUDE [Resilience to availability zone failures](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)]

Blob Storage provides robust availability zone support through ZRS configurations that automatically distribute your data across multiple availability zones within a region. Unlike locally redundant storage (LRS), ZRS guarantees that Azure synchronously replicates your blob data across multiple availability zones. ZRS ensures that your data remains accessible even if one zone experiences an outage.

Zone redundancy is enabled at the storage account level and applies to all blob containers within that account. You can't set different redundancy levels for individual containers. The redundancy configuration is applied to the entire storage account. When an availability zone experiences an outage, Azure Storage automatically routes requests to healthy zones without requiring intervention from you or your application.

:::image type="complex" source="./media/reliability-storage/zone-redundant-storage.png" alt-text="Diagram that shows how data is replicated in the primary region with zone-redundant storage (ZRS)." lightbox="./media/reliability-storage/zone-redundant-storage.png" border="false":::
  A blue box represents the primary region. It contains a dark purple box that represents ZRS. This box contains three white boxes that represent availability zone 1, availability zone 2, and availability zone 3. Each availability zone box contains a gray box that represents a datacenter. Each datacenter box contains a light purple box that includes the storage account and an icon labeled copy 1, copy 2, and copy 3.
:::image-end:::

### Requirements

- **Region support:** You can deploy zone-redundant Azure Storage accounts [in any region that supports availability zones](./regions-list.md).

- **Storage account types:** Zone redundancy is available for both Standard general-purpose v2 and Premium Block Blob storage account types. Block blobs, append blobs, and page blobs all support zone-redundant configurations, but the type of storage account that you use determines which capabilities are available. For more information, see [Supported storage account types](/azure/storage/common/storage-redundancy#supported-storage-account-types).

### Cost

When you enable zone-redundant storage (ZRS), you pay at a different rate than locally redundant storage (LRS) because of the extra replication and storage overhead.

For more information, see [Blob Storage pricing](https://azure.microsoft.com/pricing/details/storage/blobs/).

### Configure availability zone support

- **Create a blob storage account with zone redundancy.** To create a new storage account with ZRS, see [Create a storage account](/azure/storage/common/storage-account-create) and select **ZRS**, **geo-zone-redundant storage (GZRS)**, or **read-access geo-redundant storage (RA-GZRS)** as the redundancy option during account creation.

- **Change replication type.** To learn how to change an existing storage account to zone-redundant storage (ZRS) and about configuration options and requirements, see [Change how a storage account is replicated](/azure/storage/common/redundancy-migration).

- **Disable zone redundancy.** Convert ZRS accounts back to a nonzonal configuration, such as locally redundant storage (LRS), by using the same redundancy configuration change process.

### Behavior when all zones are healthy

This section describes what to expect when a blob storage account is configured for zone redundancy and all availability zones are operational.

- **Traffic routing between zones:** Azure Storage with zone-redundant storage (ZRS) automatically distributes requests across storage clusters in multiple availability zones. Traffic distribution is transparent to applications and requires no client-side configuration.

- **Data replication between zones:** All write operations to ZRS are replicated synchronously across all availability zones within the region. When you upload or modify data, the operation isn't considered complete until the data is successfully replicated across all of the availability zones. This synchronous replication ensures strong consistency and zero data loss during zone failures.

### Behavior during a zone failure

This section describes what to expect when a blob storage account is configured for ZRS and there's an availability zone outage.

- **Detection and response:** Microsoft automatically detects zone failures and initiates recovery processes. No customer action is required for zone-redundant storage (ZRS) accounts. If a zone becomes unavailable, Azure undertakes networking updates such as Domain Name System (DNS) repointing.

[!INCLUDE [Resilience to availability zone failures (Service Health and Resource Health)](./includes/reliability-availability-zone-down-notification-service-resource-include.md)]

- **Active requests:** In-flight requests might be dropped during the recovery process and should be retried. Applications should [implement retry logic](#resilience-to-transient-faults) to handle these temporary interruptions.

- **Expected data loss:** No data loss occurs during zone failures because data is synchronously replicated across multiple zones before write operations complete.

- **Expected downtime:** A small amount of downtime, typically, a few seconds, might occur during automatic recovery as traffic is redirected to healthy zones. When you design applications for ZRS, follow practices for [transient fault handling](#resilience-to-transient-faults), including implementing retry policies with exponential back-off.

- **Traffic rerouting:** If an availability zone goes offline, Azure initiates networking changes like Domain Name System (DNS) repointing. These updates ensure that traffic is rerouted to the remaining healthy availability zones. The service maintains full functionality by using the surviving zones and doesn't require customer intervention.

### Zone recovery

When the failed availability zone recovers, Azure Storage automatically restores normal operations across all of the availability zones. The service automatically ensures data consistency by synchronizing any operations that occurred during the outage period.

### Test for zone failures

When you use zone-redundant storage (ZRS), Azure Storage manages replication, traffic routing, and zone-down responses automatically. Because this feature is fully managed, you don't need to initiate or validate availability zone failure processes.

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

- [Read-access geo-redundant storage (RA-GRS) and read-access geo-zone-redundant storage (RA-GZRS)](/azure/storage/common/storage-redundancy#read-access-to-data-in-the-secondary-region) extends geo-redundant storage (GRS) and geo-zone-redundant storage (GZRS), with the added benefit of read access to the secondary endpoint. These options are ideal for applications designed for high availability business-critical applications. In the unlikely event that the primary endpoint experiences an outage, applications configured for read access to the secondary region can continue to operate.

#### Failover types

Azure Storage supports three types of failover for different scenarios.

- **Customer-managed unplanned failover:** You're responsible for initiating recovery if there's a region-wide storage failure in your primary region.

- **Customer-managed planned failover:** You're responsible for initiating recovery if another part of your solution has a failure in your primary region, and you need to switch your whole solution over to a secondary region. Use a planned failover when storage remains operational in the primary region, but you need to fail over your whole solution to a secondary region, such as for disaster recovery drills designed to ensure compliance and audit requirements.

- **Microsoft-managed failover:** In exceptional circumstances, Microsoft might initiate failover for all geo-redundant storage (GRS) accounts in a region. However, Microsoft-managed failover is a last resort and is expected to only be performed after an extended period of outage. You shouldn't rely on Microsoft-managed failover.

GRS accounts can use any of these failover types. You don't need to preconfigure a storage account to use any of the failover types ahead of time.

#### Requirements

- **Region support:** Azure Storage geo-redundant configurations use [Azure paired regions](./regions-paired.md) for secondary region replication. The secondary region is automatically determined based on your primary region selection and can't be customized. For a complete list of Azure paired regions, see [Azure regions list](./regions-list.md).

  If your storage account's region isn't paired, consider using the [custom multiregion solutions for resiliency](#custom-multiregion-solutions-for-resiliency).

- **Storage account types:** Geo-redundant storage (GRS) and customer-initiated failover and failback are available in all [Azure paired regions](./regions-paired.md) that support general-purpose v2 Azure Storage accounts.

#### Considerations

When you implement multiregion Blob Storage, consider the following key factors:

- **Asynchronous replication latency:** Data replication to the secondary region is asynchronous, which means that there's a lag between when data is written to the primary region and when it becomes available in the secondary region. This lag can result in potential data loss if a primary region failure occurs before recent data is replicated. The data loss is measured by the recovery point objective (RPO). You can expect the replication lag to be less than 15 minutes, but this time is an estimate and not guaranteed.

   You can check the [Last Sync Time property](/azure/storage/common/last-sync-time-get) to understand how much data might be lost if your storage account has an unplanned failover.

- **Secondary region access:** With geo-redundant storage (GRS) and geo-zone-redundant storage (GZRS) configurations, the secondary region isn't accessible for reads until a failover occurs.

  read-access geo-redundant storage (RA-GRS) and read-access geo-zone-redundant storage (RA-GZRS) configurations provide read access to the secondary region during normal operations, but because of the asynchronous replication latency, they might return slightly outdated data.

- **Feature limitations:** Some Azure Storage features aren't supported or have limitations when you use geo-redundant storage (GRS) or customer-managed failover. Review [feature compatibility](/azure/storage/common/storage-disaster-recovery-guidance#unsupported-features-and-services) before you implement geo-redundancy.

#### Cost

Multi-region Azure Storage account configurations incur extra costs for cross-region replication and storage in the secondary region. Data transfer between Azure regions is charged based on standard inter-region bandwidth rates.

For more information, see [Blob Storage pricing](https://azure.microsoft.com/pricing/details/storage/blobs/).

#### Configure multiregion support

- **Create a new geo-redundant storage (GRS) account.** To create a GRS account, see [Create a storage account](/azure/storage/common/storage-account-create) and select GRS, read-access geo-redundant storage (RA-GRS), geo-zone-redundant storage (GZRS), or read-access geo-zone-redundant storage (RA-GZRS) during account creation.

- **Enable geo-redundancy on an existing storage account.** To convert an existing storage account to geo-redundant storage (GRS), see [Change how a storage account is replicated](/azure/storage/common/redundancy-migration).

  > [!WARNING]
  > After your account is reconfigured for geo-redundancy, it might take a significant amount of time before existing data in the new primary region is fully copied to the new secondary region.
  >
  > **To avoid a major data loss**, check the value of the [Last Sync Time property](/azure/storage/common/last-sync-time-get) before you initiate an unplanned failover. To evaluate potential data loss, compare the last sync time to the last time that data was written to the new primary region.

- **Disable geo-redundancy.** Convert GRS accounts back to single-region configurations like locally redundant storage (LRS) or zone-redundant storage (ZRS) by using the same redundancy configuration change process.

#### Behavior when all regions are healthy

This section describes what to expect when a storage account is configured for geo-redundancy and all regions are operational.

- **Traffic routing between regions:** Azure Storage uses an active-passive approach where all write operations and most read operations are directed to the primary region.

  For read-access geo-redundant storage (RA-GRS) and read-access geo-zone-redundant storage (RA-GZRS) configurations, applications can optionally read from the secondary region by accessing the secondary endpoint. This approach requires explicit application configuration and isn't automatic. Also, because of the asynchronous replication lag, data in the secondary region might be slightly outdated.

- **Data replication between regions:** Write operations are first committed to the primary region by using the following configured redundancy types:

   - Locally redundant storage (LRS) for geo-redundant storage (GRS) and RA-GRS
   - Zone-redundant storage (ZRS) for geo-zone-redundant storage (GZRS) and RA-GZRS

   After successful completion in the primary region, data is asynchronously replicated to the secondary region where it's stored by using LRS.

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

  - **Notification:** [!INCLUDE [Region down notification partial (Service Health and Resource Health)](./includes/reliability-region-down-notification-service-resource-partial-include.md)]

  > [!IMPORTANT]
  > Use customer-managed failover options to develop, test, and implement your DR plans. **Don't rely on Microsoft-managed failover**, which might only be used in extreme circumstances. A Microsoft-managed failover is likely initiated for an entire region. It can't be initiated for individual storage accounts, subscriptions, or customers. Failover might occur at different times for different Azure services. We recommend that you use customer-managed failover.

#### Region recovery

The failback process differs significantly between Microsoft-managed and customer-managed failover scenarios.

- **Customer-managed failover (unplanned):** After an unplanned failover, the storage account is configured with locally redundant storage (LRS). To fail back, you need to re-establish the geo-redundant storage (GRS) relationship and wait for the data to be replicated.

- **Customer-managed failover (planned):** After a planned failover, the storage account remains geo-replicated. You can initiate another customer-managed failover to fail back to the original primary region. [The same failover considerations apply](#behavior-during-a-region-failure).

- **Microsoft-managed failover:** If Microsoft initiates a failover, it's likely that a significant disaster occurred in the primary region, and the primary region might not be recoverable. Any timelines or recovery plans depend on the extent of the regional disaster and recovery efforts. You should monitor Azure Service Health communications for details.

#### Test for region failures

You can simulate regional failures to test your disaster recovery procedures.

- **Planned failover testing:** For geo-redundant storage (GRS) accounts, you can perform planned failover operations during maintenance windows to test the complete failover and failback process. Planned failover doesn't require data loss, but it does involve downtime during both failover and failback.

- **Secondary endpoint testing:** For read-access geo-redundant storage (RA-GRS) and read-access geo-zone-redundant storage (RA-GZRS) configurations, regularly test read operations against the secondary endpoint to ensure that your application can successfully read data from the secondary region.

### Custom multiregion solutions for resiliency

The cross-region failover capabilities of Azure Storage might be unsuitable because of the following reasons:

- Your storage account is in a nonpaired region.

- Your business uptime goals aren't satisfied by the recovery time or data loss that the built-in failover options provide.

- You need to fail over to a region that isn't your primary region's pair.

- You need an active/active configuration across regions.

This section provides a high-level overview of some approaches to consider. A comprehensive overview of multiregion deployment topologies for Azure Storage is outside the scope of this article.

You can deploy Azure Storage across multiple regions by using separate storage accounts in each region. This approach provides flexibility in region selection, the ability to use nonpaired regions, and more granular control over replication timing and data consistency. When you implement multiple storage accounts across regions, you need to configure cross-region data replication, implement load balancing and failover policies, and ensure data consistency across regions.

**Object replication** provides an extra option for cross-region data replication that provides asynchronous copying of block blobs between storage accounts. Unlike the built-in geo-redundant storage options that use fixed paired regions, object replication allows you to replicate data between storage accounts in any Azure region, including nonpaired regions. This approach gives you full control over source and destination regions, replication policies, and the specific containers and blob prefixes to replicate.

You can configure object replication to replicate all blobs within a container or specific subsets based on blob prefixes and tags. The replication is asynchronous and occurs in the background. You can configure multiple replication policies and even chain replication across multiple storage accounts to create sophisticated multiregion topologies.

Object replication isn't compatible with all storage accounts. For example, it doesn't work with storage accounts that use hierarchical namespaces (also known as *Azure Data Lake Storage Gen2 accounts*).

For more information, see [Object replication for block blobs](/azure/storage/blobs/object-replication-overview) and [Configure object replication](/azure/storage/blobs/object-replication-configure).

## Backup and recovery

Blob Storage provides multiple data protection mechanisms that complement redundancy for comprehensive backup strategies. The service's built-in redundancy protects against infrastructure failures, and extra backup capabilities protect against accidental deletion, corruption, and malicious activities.

**Point-in-time restore (PITR)** allows you to restore block blob data to a previous state within a configured retention period of up to 365 days. Microsoft fully manages this feature. It also provides granular recovery capabilities at the container or blob level. PITR data is stored in the same region as the source account and is accessible even during regional outages if you use geo-redundant configurations.

**Blob versioning** automatically maintains previous versions of blobs when they're modified or deleted. Each version is stored as a separate object and can be accessed independently. Versions are stored in the same region as the current blob and follow the same redundancy configuration as the storage account.

**Soft delete** provides a safety net for accidentally deleted blobs and containers by retaining deleted data for a configurable period. Soft-deleted data remains in the same storage account and region, which makes it immediately available for recovery. For geo-redundant accounts, soft-deleted data is also replicated to the secondary region.

**Blob snapshots** create read-only, point-in-time copies of blobs that you can use for backup and recovery scenarios. Snapshots are stored in the same storage account and follow the same redundancy and geo-replication settings as the base blob.

For cross-region backup requirements, consider using **Azure Backup for blobs**, which provides centralized backup management and can store backup data in regions different from the source data. This service provides operational and vaulted backup options that have configurable retention policies and restore capabilities. For more information, see [Backup for blobs overview](/azure/backup/blob-backup-overview).

[!INCLUDE [Backups include ](includes/reliability-backups-include.md)]

## Service-level agreement

The service-level agreement (SLA) for Azure Storage describes the expected availability of the service and the conditions that must be met to achieve that availability expectation. The availability SLA you're eligible for depends on the storage tier and the replication type that you use. For more information, see [SLAs for Online Services](https://aka.ms/csla).

## Related content

- [Blob Storage overview](/azure/storage/blobs/storage-blobs-overview)
- [Azure Storage redundancy](/azure/storage/common/storage-redundancy)
- [Disaster recovery and storage account failover](/azure/storage/common/storage-disaster-recovery-guidance)
- [Use geo-redundancy to design highly available applications](/azure/storage/common/geo-redundant-design)
- [Change how a storage account is replicated](/azure/storage/common/redundancy-migration)

