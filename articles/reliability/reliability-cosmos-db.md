---
title: Reliability in Azure Cosmos DB
description: Learn how to make Azure Cosmos DB resilient to a variety of potential outages and problems, including transient faults, availability zone outages, region outages, and service maintenance, and learn about backup and restore.
author: seesharprun
ms.author: sidandrews
ms.topic: reliability-article
ms.custom: subject-reliability
ms.service: azure-cosmos-db
ms.date: 03/13/2026
---

# Reliability in Azure Cosmos DB

Azure Cosmos DB for NoSQL is a globally distributed, multi-model database service that supports document data models with flexible schemas. Azure Cosmos DB offers comprehensive reliability features including multiple consistency levels that let you balance durability and availability, zone-redundant deployments that protect against availability zone failures, multi-region replication with service-managed or customer-managed failover, and continuous and periodic backup options for data protection.

[!INCLUDE [Shared responsibility](includes/reliability-shared-responsibility-include.md)]

This article describes how to make Azure Cosmos DB resilient to various potential outages and problems, including transient faults, availability zone outages, region outages, and service maintenance. It also describes how to use backups to recover from other types of problems and highlights key information about the Azure Cosmos DB service-level agreement (SLA).

## Production deployment recommendations

The Azure Well-Architected Framework provides recommendations across reliability, security, cost, operations, and performance. To understand how these areas influence each other and contribute to a reliable Azure Cosmos DB solution, see [Architecture best practices for Azure Cosmos DB](/azure/well-architected/service-guides/cosmos-db).

## Reliability architecture overview

[!INCLUDE [Introduction to reliability architecture overview section](includes/reliability-architecture-overview-introduction-include.md)]

### Logical architecture

The primary resource you deploy is an Azure Cosmos DB *account*. Each account can have multiple *databases*, and databases can have multiple *containers*. Containers serve as the logical units of distribution and scalability. You create collections, tables, and graphs, depending on the API you use to interact with Azure Cosmos DB. These entities are internally represented as containers. For more information about the resource model, see [Databases, containers, and items in Azure Cosmos DB](/azure/cosmos-db/resource-model). Each container uses [partitioning](/azure/cosmos-db/partitioning), which supports high scale and high performance.

A single account can [span multiple Azure regions](/azure/cosmos-db/distribute-data-globally), which increases your resiliency to region outages. You can configure multiple regions for reading or [for writing](/azure/cosmos-db/multi-region-writes). Azure Cosmos DB automatically geo-replicates your data. Geo-replication behavior is affected by the configuration you use, such as the [consistency level](/azure/cosmos-db/consistency-levels), which indicates how you wish to make tradeoffs between data consistency, availability, latency, and throughput. Different consistency levels optimize for different concerns, support different guarantees, and provide different types of cross-region replication.

<!-- TODO
- RUs, and throughput types - provisioned, autoscale, serverless
- Tiers (Business Critical etc)
-->

### Physical architecture

Azure Cosmos DB stores multiple *replicas* of your data for redundancy. The service automatically mitigates replica outages by guaranteeing at least three replicas of your data in each Azure region for your account within a four-replica quorum. This guarantee results in zero downtime and zero data loss during individual node outages, without requiring application changes or configurations.

Internally, Azure Cosmos DB manages your data through various constructs including *physical partitions*, *partition sets*, and *replica sets*. For more detailed information on how Azure Cosmos DB works, see [Global data distribution with Azure Cosmos DB - under the hood](/azure/cosmos-db/global-distribution).

## Resilience to transient faults

[!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)]

The Azure Cosmos DB SDKs automatically implement support for a range of resiliency considerations, including transient fault handling through automatic retries, and honoring rate limit responses sent by the service. For more information, see [Design resilient applications with Azure Cosmos DB SDKs](/azure/cosmos-db/conceptual-resilient-sdk-applications).

<a name="availability-zone-support"></a>

## Resilience to availability zone failures

[!INCLUDE [Resilience to availability zone failures](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)]

Azure Cosmos DB supports *zone redundancy*. When you enable zone redundancy, Azure distributes the replicas of your data across multiple availability zones, providing resiliency to datacenter problems and outages. Microsoft selects the availability zones to use.

![Diagram showing an Azure Cosmos DB account with a replica set that contains four replicas, which are distributed across the zones.](./media/reliability-cosmos-db/zone-redundant.png)

An Azure Cosmos DB account might use multiple regions (locations). You can configure zone redundancy separately for each region in your account.

Using zone redundancy in Azure Cosmos DB has no discernible impact on performance or latency. It doesn't require any adjustments to the selected consistency mode, and also doesn't require any modification to application code.

We recommend using zone redundancy in regions where it's supported, especially for single-region accounts. Because availability zones are physically separate and provide distinct power source, network, and cooling, the availability SLAs for Azure Cosmos DB are higher for zone-redundant accounts than accounts that don't use availability zones.

> [!TIP]
> Enabling zone redundancy is a great way to increase the resilience of your Azure Cosmos DB database without introducing additional application complexities, affecting performance, or even incurring additional costs (if autoscale is also used).

If you don't enable zone redundancy, the account is *nonzonal* in that region. That means that all of the replicas could be located in a single availability zone, leading to potential downtime if that specific zone experiences an issue.

### Requirements

- **Region support:** You can enable zone redundancy in Azure regions that supports availability zones. To see if your region supports availability zones, see [the list of supported regions](./regions-list.md).

    Enabling zone redundancy isn't an account-wide choice. A single Azure Cosmos DB account can span an arbitrary number of Azure regions, each of which can independently be configured to use zone redundancy. Some regions don't yet support availability zones, but adding them to an Azure Cosmos DB account won't prevent enabling zone redundancy in other regions configured for that account.

- **Serverless:** Serverless accounts can use zone redundancy, but this choice is only available during account creation. Existing serverless accounts without availability zones can;t be converted to an availability zone configuration. For mission critical workloads, we recommend you use provisioned throughput.

### Considerations

- **Multiple simultaneous zone outages:** A single-region account with zone redundancy can maintain read-write availability when an outage affects only one availability zone. However, if multiple availability zones or the entire region is impacted, single-region accounts lose read and write access until service is restored.

<!-- TODO interactions between multi-region and AZ support -->

### Cost

Regions where zone redundancy is enabled are charged at a premium. However, the premium pricing for availability zones is waived for accounts configured with multi-region writes, and for collections configured to use autoscale throughput mode. For more information, see [Azure Cosmos DB pricing](https://azure.microsoft.com/pricing/details/cosmos-db/).

### Configure availability zone support

You can configure availability zones only when you add a new region to an Azure Cosmos DB account.

> [!NOTE]
> If you receive an error during deployment indicating the region is constrained and you can't enable zone redundancy, [open a support request](/azure/azure-portal/supportability/how-to-create-azure-support-request) to request capacity in the region's zones.

- **Create a new Azure Cosmos DB account with zone redundancy:** When you create a new Azure Cosmos DB account, you can configure zone redundancy on one or more regions by using these instructions:

    - [Azure portal](/azure/cosmos-db/quickstart-portal). When deploying, set the *Availability Zones* setting to *Enabled*.
    - [Azure CLI](/azure/cosmos-db/how-to-create-account?tabs=azure-cli). When setting the `--locations` argument, set `isZoneRedundant=True` for the regions you want to make zone-redundant.
    - [Bicep](/azure/cosmos-db/quickstart-template-bicep). Update the `isZoneRedundant` property to `true` for the regions you want to make zone-redundant.
    - [Azure Resource Manager templates](/azure/cosmos-db/quickstart-template-json). Update the `isZoneRedundant` property to `true` for the regions you want to make zone-redundant.

- **Add a zone-redundant region to an existing Azure Cosmos DB account:** To enable availability zone support on an existing account, you need to add the region and enable zone redundancy on itr. Follow these instructions to add a region:

    - [Azure portal](/azure/cosmos-db/how-to-manage-database-account#add-remove-regions-from-your-database-account)
    - [Azure CLI](/azure/cosmos-db/manage-with-cli#add-or-remove-regions)
    - [Bicep](/azure/cosmos-db/manage-with-bicep)
    - [Azure Resource Manager templates](/azure/cosmos-db/manage-with-templates)

- **Enable zone redundancy on an existing region in your account:** Because you can't enable zone redundancy for a region that has already been added to your account, you need to remove that region and add it again with availability zones enabled. To avoid any service disruption, add a temporary region and fail over to it until the availability zone configuration is complete. Follow the steps below to enable availability zones for your account in select regions.

    #### [Azure portal](#tab/portal)

    1. Add a temporary region to your database account by following steps in [Add region to your database account](/azure/cosmos-db/how-to-manage-database-account#add-remove-regions-from-your-database-account).

    1. If your Azure Cosmos DB account is configured with multi-region writes, skip to the next step. Otherwise, perform manual failover to the temporary region by following the steps in [Perform manual failover on an Azure Cosmos DB account](/azure/cosmos-db/how-to-manage-database-account?source=recommendations#manual-failover).

    1. Remove the region for which you would like to enable availability zones by following steps in [Remove region to your database account](/azure/cosmos-db/how-to-manage-database-account#add-remove-regions-from-your-database-account).

    1. Add back the region to be enabled with availability zones:
        1. [Add region to your database account](/azure/cosmos-db/how-to-manage-database-account#add-remove-regions-from-your-database-account).
        1. Find the newly added region in the **Write region** column, and enable **Availability Zone** for that region. 
        1. Select **Save**.

    1. Perform failback to the availability zone-enabled region by following the steps in [Perform manual failover on an Azure Cosmos DB account](/azure/cosmos-db/how-to-manage-database-account?source=recommendations#manual-failover).

    1. Remove the temporary region by following steps in [Remove region to your database account](/azure/cosmos-db/how-to-manage-database-account#add-remove-regions-from-your-database-account).

    #### [Azure CLI](#tab/cli)

    1. Add a temporary region to your database account. The following example shows how to add West US as a secondary region to an account configured with East US region only. You must include all existing regions and any new ones in the command.

        ```azurecli        
        az cosmosdb update \
            --name MyCosmosDBDatabaseAccount \
            --resource-group MyResourceGroup \
            --locations regionName=eastus failoverPriority=0 isZoneRedundant=False \
            --locations regionName=westus failoverPriority=1 isZoneRedundant=False        
        ```

    1. If your Azure Cosmos DB account is configured with multi-region writes, skip to the next step.
    
        Otherwise, perform manual failover to the newly added temporary region. The following example shows how to perform a failover from East US region (current write region) to West US region (current read-only region). You must include both regions in the command. 

        ```azurecli
        az cosmosdb failover-priority-change \
            --name MyCosmosDBDatabaseAccount \
            --resource-group MyResourceGroup \
            --failover-policies westus=0 eastus=1
        ```

    1. Remove the region for which you would like to enable availability zones. The following example shows how to remove East US region from an account configured with West US (write region) and East US (read-only) regions. You must include all regions that shouldn't be removed in the command. 

        ```azurecli        
        az cosmosdb update \
            --name MyCosmosDBDatabaseAccount \
            --resource-group MyResourceGroup \
            --locations regionName=westus failoverPriority=0 isZoneRedundant=False        
        ```
    
    1. Add back the region to be enabled with availability zones. The following example shows how to add East US as an AZ-enabled secondary region to an account configured with West US region only. You must include any existing regions and all new ones in the command. 
        
        ```azurecli
        az cosmosdb update \
            --name MyCosmosDBDatabaseAccount \
            --resource-group MyResourceGroup \
            --locations regionName=westus failoverPriority=0 isZoneRedundant=False \
            --locations regionName=eastus failoverPriority=1 isZoneRedundant=True
        ```

    1. Perform failback to the availability zone-enabled region. The following example shows how to perform a failover from West US region (current write region) to East US region (current read-only region). You must include both regions in the command. 
    
        ```azurecli
        az cosmosdb failover-priority-change \
            --name MyCosmosDBDatabaseAccount \
            --resource-group MyResourceGroup \
            --failover-policies eastus=0 westus=1
        ```

    1. Remove the temporary region. The following example shows how to remove West US region from an account configured with East US (write region) and West US (read-only) regions. You must include all accounts that shouldn't be removed in the command. 
    
        ```azurecli        
        az cosmosdb update \
            --name MyCosmosDBDatabaseAccount \
            --resource-group MyResourceGroup \
            --locations regionName=eastus failoverPriority=0 isZoneRedundant=True        
        ```
    
    ---

    > [!WARNING]
    > When you enable zone redundancy for a region, a few seconds of of write unavailability occurs when adding and removing the secondary region, as the system deliberately stops writes in order to check consistency between regions.

### Behavior when all zones are healthy

This section describes what to expect when you configure an Azure Cosmos DB account for zone redundancy, and all zones are operational.

- **Cross-zone operation:** Requests are automatically spread across the replicas in each availability zone. A request might go to a replica in any availability zone.

- **Cross-zone data replication:** When a client makes a change to any data, that change is applied to multiple replicas in different zones to achieve quorum. This approach is referred to as *synchronous replication*. Synchronous replication ensures a high level of data consistency, which reduces the likelihood of data loss during a zone failure. Availability zones are located relatively close together, which means there's minimal effect on latency or throughput.

### Behavior during a zone failure

This section describes what to expect when you configure an Azure Cosmos DB account for zone redundancy, and there's an outage in one of the zones.

- **Detection and response:** The Azure Cosmos DB platform is responsible for detecting a failure in an availability zone. You don't need to do anything to initiate a zone failover.

[!INCLUDE [Availability zone down notification (Service Health and Resource Health)](./includes/reliability-availability-zone-down-notification-service-resource-include.md)]

- **Active requests:** When an availability zone is unavailable, any requests in progress that are connected to a replica in the faulty availability zone are terminated and need to be retried. Ensure that your applications are prepared by following [transient fault handling guidance](#resilience-to-transient-faults).

- **Expected data loss:** A zone failure isn't expected to cause any data loss.

- **Expected downtime:** During zone outages, connections might experience brief interruptions that typically last a few seconds as traffic is redistributed. Ensure that your applications are prepared by following [transient fault handling guidance](#resilience-to-transient-faults).

- **Redistribution:** Azure Cosmos DB automatically redirects incoming requests to healthy replicas in other availability zones. When an availability zone has an outage, the platform automatically reallocates provisioned throughput to other replicas.

### Zone recovery

When the availability zone recovers, Azure Cosmos DB automatically restores replicas in the availability zone, and reroutes traffic between replicas as normal.

### Test for zone failures

Your applications can partially simulate the zone outage behavior by using the Azure Cosmos DB Fault Injection library for Java. The [Server Return Gone](/java/api/overview/azure/cosmos-test-readme#server-return-gone-scenario) scenario lets you inject `GONE` errors into specific replicas. This approach helps you to exercise the same SDK retry logic, and re-routes to use the code paths that activate during a real zone outage.

## Resilience to region-wide failures

Region outages are outages that affect all Azure Cosmos DB nodes in an Azure region.

When an Azure Cosmos DB account is deployed in a single region and the region has an outage, generally no data loss occurs, but your application can't access your data. Data access is restored after Azure Cosmos DB services recover in the affected region. Data loss might occur only with an unrecoverable disaster in the Azure Cosmos DB region.

For the rare cases of region outages, you can configure Azure Cosmos DB to support various outcomes of durability and availability by using multiple read or write regions.

### Multiple read or write regions

If your solution requires continuous uptime during region outages, you can configure Azure Cosmos DB to replicate your data across multiple regions and to transparently fail over to operating regions when required.

<!-- TODO single write region and multiple read regions; or multiple write regions -->

> [!NOTE]
> Single-region accounts might lose accessibility after a regional outage. To ensure business continuity at all times, we recommend that you set up your Azure Cosmos DB account with *a single write region and at least a second (read) region* and enable *service-managed failover*.

#### Failover types

<!-- TODO manual and service-managed failover -->

#### Requirements

**Region support:** You can configure any Azure region as a read or write region for your Azure Cosmos DB account.

#### Cost

Adding an additional region to an Azure Cosmos DB account increases your existing costs for each region. For more information, see [Azure Cosmos DB pricing](https://azure.microsoft.com/pricing/details/cosmos-db/).

#### Configure multi-region support

<!-- TODO -->

#### Capacity planning and management

If your application spreads requests across regions and one region goes offline, the remaining regions experience higher request volume. Use autoscale throughput to dynamically adjust capacity based on demand. If you use provisioned throughput, plan for adequate capacity to handle the loss of a region without service degradation, and consider over-provisioning. For more information, see [Manage capacity with over-provisioning](./concept-redundancy-replication-backup.md#manage-capacity-with-over-provisioning).

#### Behavior when all regions are healthy

This section describes what to expect when you configure an Azure Cosmos DB account for multi-region support, and all regions are operational.

- **Cross-region operation:** <!-- TODO -->

- **Cross-region data replication:** <!-- TODO -->

#### Behavior during a region failure

This section describes what to expect when you configure an Azure Cosmos DB account for multi-region support, and there's an outage in one of the replica regions.

- **Detection and response:** <!-- TODO -->

    Write region outage: Manual failover shouldn't be triggered and won't succeed if there's an outage of the source or destination region. The reason is that the failover procedure includes a consistency check that requires connectivity between the regions. <!-- TODO check this is accurate - the other docs talk about forced failovers -->

[!INCLUDE [Region down notification (Service Health and Resource Health)](./includes/reliability-region-down-notification-service-resource-include.md)]

- **Active requests:** <!-- TODO -->

- **Expected data loss:** TODO

    When an Azure Cosmos DB account is deployed in multiple regions, data durability depends on the consistency level that you configure on the account. The following table details, for all consistency levels, the RPO of an Azure Cosmos DB account that's deployed in at least two regions.

    |**Consistency level**|**RPO for region outage**|
    |---------|---------|
    |Session, consistent prefix, eventual|Less than 15 minutes|
    |Bounded staleness|*K* and *T*|
    |Strong|0|

    *K* = The number of versions (that is, updates) of an item.

    *T* = The time interval since the last update.

    For multiple-region accounts, the minimum value of *K* and *T* is 100,000 write operations or 300 seconds. This value defines the minimum RPO for data when you're using bounded staleness.

    For more information on the differences between consistency levels, see [Consistency levels in Azure Cosmos DB](/azure/cosmos-db/consistency-levels).

- **Expected downtime:** <!-- TODO -->

- **Redistribution:** <!-- TODO -->

    Read region outages: The affected region is disconnected and marked offline. No changes are required in your application code to handle read region outages. The [Azure Cosmos DB SDKs](/azure/cosmos-db/nosql/sdk-dotnet-v3) redirect read calls to the next available region in the preferred region list. If none of the regions in the preferred region list are available, calls automatically fall back to the current write region.

    Write region outages: Azure Cosmos DB promotes one of the account's secondary regions to be the new primary write region when *service-managed failover* is configured on the Azure Cosmos DB account. The failover occurs to another region in the order of region priority that you specify.

#### Region recovery

The region recovery processes are different depending on whether the outage was in a read region or a write region.

- **For read region outages:** When the affected region is back online, it syncs with the current write region and is available again to serve read requests after it has fully caught up. Subsequent reads are redirected to the recovered region without requiring any changes to your application code. During both failover and rejoining of a previously failed region, Azure Cosmos DB continues to honor read consistency guarantees.

- **For write region outages:** When the affected region is back online, the region shows as "online" in the Azure portal, and becomes available as a read region. At this point, it is safe to switch back to the recovered region as the write region by using [PowerShell, the Azure CLI, or the Azure portal](/azure/cosmos-db/how-to-manage-database-account#perform-manual-failover-on-an-azure-cosmos-db-account). There is *no data or availability loss* before, while, or after you switch the write region. Your application continues to be highly available.

    Any write data that wasn't replicated when the region failed is made available through the [conflict feed](/azure/cosmos-db/how-to-manage-conflicts#read-from-conflict-feed). Applications can read the conflict feed, resolve the conflicts based on the application-specific logic, and write the updated data back to the Azure Cosmos DB container as appropriate.

    > [!WARNING]
    > In the event of a write region outage, where the Azure Cosmos DB account promotes a secondary region to be the new primary write region via *service-managed failover*, the original write region will **not be be promoted back as the write region automatically** once it is recovered. It is your responsibility to switch back to the recovered region as the write region using [PowerShell, the Azure CLI, or the Azure portal](/azure/cosmos-db/how-to-manage-database-account#perform-manual-failover-on-an-azure-cosmos-db-account), once it's safe to do so (as described above).

#### Test for region failures

Even if your Azure Cosmos DB account is highly available, your application might not be correctly designed to remain highly available when a region failover occurs. To test the end-to-end high availability of your application as a part of your application testing or disaster recovery (DR) drills, temporarily disable service-managed failover for the account. Invoke [manual failover by using PowerShell, the Azure CLI, or the Azure portal](/azure/cosmos-db/how-to-manage-database-account#perform-forced-failover-for-your-azure-cosmos-db-account), and then monitor your application. After you complete the test, you can fail back over to the primary region and restore service-managed failover for the account.

## Backup and restore

[!INCLUDE [Backups include](includes/reliability-backups-include.md)]

Data loss can occur because of accidental deletions or other problems in your application that cause data corruption. When you use a single-region account, data loss might also occur because of an unrecoverable disaster in the Azure Cosmos DB region. To help you protect against data loss, Azure Cosmos DB provides two backup modes:

- [Continuous backups](/azure/cosmos-db/continuous-backup-restore-introduction) back up the data in each region every 100 seconds. They enable you to restore your data to any point in time with 1-second granularity. In each region, the backup is dependent on the data committed in that region. If the region has zone redundancy enabled, then the backup is stored in zone-redundant storage.
- [Periodic backups](/azure/cosmos-db/periodic-backup-restore-introduction) fully back up all partitions from all containers under your account, with no synchronization across partitions. The minimum backup interval is 1 hour.

## Resilience to service maintenance

Azure Cosmos DB transparently manages all details of individual compute nodes, and automatically performs patching and other types of planned maintenance. The Azure Cosmos DB SLAs for availability and P99 latency apply through all automatic maintenance operations that the system performs.

## Service-level agreement

[!INCLUDE [Service-level agreement](includes/reliability-service-level-agreement-include.md)]

Azure Cosmos DB provides SLAs for a range of configurations and service characteristics, including availability, latency, throughput, and consistency.

The availability SLAs are different depending on whether you use any of the following product capabilities:

- Provisioned throughput
- Single-region account with availability zone support (zone redundancy)
- Accounts that use multiple read regions
- Accounts that use multiple write regions

## Related content

- [Azure reliability](/azure/reliability/overview)
- [Azure Cosmos DB overview](/azure/cosmos-db/overview)
- [Consistency levels in Azure Cosmos DB](/azure/cosmos-db/consistency-levels)
- [Global data distribution with Azure Cosmos DB](/azure/cosmos-db/distribute-data-globally)
- [Diagnose and troubleshoot the availability of Azure Cosmos DB SDKs in multiregional environments](/azure/cosmos-db/troubleshoot-sdk-availability) <!-- TODO read -->
