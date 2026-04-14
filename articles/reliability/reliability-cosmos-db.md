---
title: Reliability in Azure Cosmos DB
description: Learn how to make Azure Cosmos DB resilient to a variety of potential outages and problems, including transient faults, availability zone outages, region outages, and service maintenance, and learn about backup and restore.
author: seesharprun
ms.author: sidandrews
ms.topic: reliability-article
ms.custom: subject-reliability
ms.service: azure-cosmos-db
ms.date: 04/09/2026
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

You can configure throughput, which represents the resources available. You can [manually provision throughput](/azure/cosmos-db/set-throughput), [use autoscale](/azure/cosmos-db/provision-throughput-autoscale) to match your workload's requirements, or use the [serverless account type](/azure/cosmos-db/serverless) to be charged for your actual usage.

A single account can [span multiple Azure regions](/azure/cosmos-db/distribute-data-globally), which increases your resiliency to region outages. You can configure multiple regions for reading, and if you use the [Business Critical tier](/azure/cosmos-db/multi-region-writes), you can use multiple regions for writing. Azure Cosmos DB automatically geo-replicates your data. Geo-replication behavior is affected by the configuration you use, such as the [consistency level](/azure/cosmos-db/consistency-levels), which indicates how you wish to make tradeoffs between data consistency, availability, latency, and throughput. Different consistency levels optimize for different concerns, support different guarantees, and provide different types of cross-region replication.

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

> [!WARNING]
> **Note to PG:** In the legacy document, [we have a table](https://learn.microsoft.com/azure/reliability/reliability-cosmos-db-nosql#zone-redundancy-and-multi-region-accounts) that gives an opinionated benefit of using zone redundancy depending on the account's configuration. As I understand it, this was written to try to dissuade unnecessary use of zone redundancy because capacity was at a premium. Is this still necessary and valuable or should we omit it?

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

When an Azure Cosmos DB account is deployed in a single region and the region has an outage that affects all of its Azure Cosmos DB nodes, generally no data loss occurs, but your application can't access your data. Data access is restored after Azure Cosmos DB services recover in the affected region. Data loss might occur only with an unrecoverable disaster in the Azure Cosmos DB region.

To prepare for the rare cases of region outages, you can configure Azure Cosmos DB to support various levels of durability and availability by using one of these approaches:
- Multiple read regions, optionally with service-managed failover or per-partition automatic failover (PPAF) enabled
- Multiple write regions

This section focuses on the reliability aspects of these capabilities, but there are other benefits to multiple read and write regions, such as higher performance and scale for globally distributed applications. You should evaluate your whole solution architecture and consider all of the benefits of using these capabilities.

| Configuration | Outage type | Availability impact | Durability impact| What to do |
| -- | -- | -- | -- | -- |
| Multiple read regions, single write region  | Read region outage | **For most account configurations:** Client SDK automatically redirects reads to other regions. There's no read or write availability loss. <br /><br /> **If the account has exactly two regions and uses strong consistency:** *If you enable service-managed failover*, the service marks the region as failed and a failover occurs. You can also perform a manual failover to another region used by your account. If you don't, the account loses write availability until restoration of the service. | No data loss. | During the outage, ensure that there are enough provisioned Request Units (RUs) in the remaining regions to support read traffic. <br/><br/> When the outage is over, readjust provisioned RUs as appropriate. |
| Multiple read regions, single write region | Write region outage | **For all account configurations:** Client SDK automatically redirect reads to other healthy regions. <br /><br /> **If PPAF is enabled:** Azure Cosmos DB automatically fails over unhealthy partitions to a healthy region. <br/><br/> **If service-managed failover is enabled:** clients experience write availability loss until the service manages a failover to a new write region that you previously selected. <br/><br/> **Without PPAF or service-managed failover:** Clients experience write availability loss. You can perform a manual failover to another region used by your account. If you don't, the account loses write availability until restoration of the service. | If you don't select the strong consistency level, the service might not replicate some data to the remaining active regions. This replication depends on the [consistency level](/azure/cosmos-db/consistency-levels#rto) that you select. If the affected region suffers permanent data loss, you could lose unreplicated data. | During the outage, ensure that there are enough provisioned RUs in the remaining regions to support read traffic. <br/><br/> *Don't* trigger a manual failover during the outage, because it can't succeed. <br/><br/> When the outage is over, readjust provisioned RUs as appropriate. Accounts that use the API for NoSQL might also recover the unreplicated data in the failed region from your [conflict feed](/azure/cosmos-db/how-to-manage-conflicts#read-from-conflict-feed). |
| Multiple write regions | Outage of any region used by the account | <!-- TODO square this info with the DR doc --> There's a possibility of temporary loss of write availability, which is analogous to a single write region with service-managed failover. The failover of the [conflict-resolution region](#conflict-resolution-region) might also cause a loss of write availability if a high number of conflicting writes happen at the time of the outage. | Recently updated data in the failed region might be unavailable in the remaining active regions, depending on the selected [consistency level](/azure/cosmos-db/consistency-levels). If the affected region suffers permanent data loss, you might lose unreplicated data. | During the outage, ensure that there are enough provisioned RUs in the remaining regions to support more traffic. <br/><br/> When the outage is over, you can readjust provisioned RUs as appropriate. If possible, Azure Cosmos DB automatically recovers unreplicated data in the failed region. This automatic recovery uses the conflict resolution method that you configure for accounts that use the API for NoSQL. For accounts that use other APIs, this automatic recovery uses *last write wins*. |

#### Potential data loss during region outages

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

### Multiple read regions with a single write region
 
If your solution requires continuous uptime during region outages, you can configure Azure Cosmos DB to replicate your data across multiple regions and to transparently fail over to operating regions when required. You can optionally configure your applications to connect to specific read regions, which can help to improve their performance.

![Diagram showing an Azure Cosmos DB account. Region A is a write and read region, and region B is a read region. An application in region A performs reads and writes against the Azure Cosmos DB account in region A. An application in region B performs reads against the account in region B, but writes against region A.](./media/reliability-cosmos-db/multiple-read-regions.png)

#### Failover types

TODO SDK

Failover is the process of making one of your account's regions unavailable, either completely or in part. The effect of a failover depends on whether the region is a write region or a read region:

- If a write region becomes unavailable, another region becomes the write region.
- If a read region becomes unavailable, that region can't serve read requests and other regions are used for read operations instead.

Azure Cosmos DB provides multiple types of failover:

- **Service-managed failover:** When your account uses service-managed failover, Microsoft is responsible for deciding when to fail over between regions. To enable service-managed failover, you specify priorities for each region. In ordinary operations, traffic goes to the highest priority region.

    If Microsoft detects a failure and declares an outage or disruption, it can trigger service-managed failover to use the next highest priority region. The process of declaring an outage and triggering service-managed failover can take some time.

    If Microsoft triggers service-managed failover for the account's write region, any unreplicated writes could be lost.

    After a service-managed failover, Microsoft must bring a region back online. Microsoft automatically brings the region online but this process can take several days.

- **Per-partition automatic failover (PPAF):** Internally, Azure Cosmos DB spreads your data across multiple physical partitions. If a problem occurs with the infrastructure supporting a partition, other partitions might not be affected. PPAF enables single-write region accounts to automatically fail over individual partitions to a secondary region while keeping healthy partitions in the primary region. PPAF can help to minimize downtime and enable faster recovery during a partial region failure. For more information, see [How to onboard and adopt Per-Partition Automatic Failover (PPAF) for Azure Cosmos DB](/azure/cosmos-db/how-to-configure-per-partition-automatic-failover).

    > [!NOTE]
    > Per Partition Automatic Failover is in public preview. This feature is provided without a service level agreement. For more information, see [Supplemental Terms of Use for Microsoft Azure Previews](https://azure.microsoft.com/support/legal/preview-supplemental-terms/).

- **Forced (manual) failover:** You can manually take one of your account's region offline. This is referred to as a *forced failover* or a *manual failover*. If you need to rapidly respond during an outage, before Microsoft declares an outage or disruption, you can trigger a forced failover. You're responsible for detecting the outage and triggering the failover. You can also use forced failovers to simulate region-down scenarios for testing, like during a diaster recovery drill.

    If you take the write region offline, the read region with the next highest priority becomes the new write region. If you take a read region offline, your applications can connect to any other read region in the account.

    A forced failover of your write region carries the possibility of data loss for any unreplicated writes.

    After a forced failover, Microsoft must bring a region back online. During a real region outage, Microsoft automatically brings the region online but this process can take several days. If you use this approach for a failover drill, open a support case to ask Microsoft to bring the region online.

- **Switch write region:** When the regions are healthy, you can switch your account's write region. This change is effectively a planned failover of the write region for your account.

    Switching the write region results in no data loss, because the data replication catches up before the new write region is promoted. There might be a small amount of downtime, but clients that use retry logic and other transient fault handling don't typically experience significant impact.

    This operation requires the regions to be healthy, so it can't be used during a region outage.

#### Requirements

**Region support:** You can configure any Azure region as a read region for your Azure Cosmos DB account.

#### Cost

Adding an additional read region to an Azure Cosmos DB account increases your existing costs for each region. For more information, see [Azure Cosmos DB pricing](https://azure.microsoft.com/pricing/details/cosmos-db/).

#### Configure multiple read regions

- **Add read regions to an account:** You can configure multiple regions on your account when you create the account or at any time after the account is created. For more information, see [Add/remove regions from your database account](/azure/cosmos-db/how-to-manage-database-account#add-remove-regions-from-your-database-account).

- **Enable failover:** Some types of failover must be configured in advance:

    - *Service-managed failover:* First, [enable service-managed failover](/azure/cosmos-db/how-to-manage-database-account#enable-service-managed-failover-for-your-azure-cosmos-db-account) and then [set failover priorities for each region in your account](/azure/cosmos-db/how-to-manage-database-account#set-failover-priorities-for-your-azure-cosmos-db-account).

    - *Per-partition automatic failover*: For more information, see [How to onboard and adopt Per-Partition Automatic Failover (PPAF) for Azure Cosmos DB](/azure/cosmos-db/how-to-configure-per-partition-automatic-failover).

#### Capacity planning and management

If your application spreads requests across regions and one region goes offline, the remaining regions experience higher request volume. Use autoscale throughput to dynamically adjust capacity based on demand. If you use provisioned throughput, plan for adequate capacity to handle the loss of a region without service degradation, and consider over-provisioning. For more information, see [Manage capacity with over-provisioning](./concept-redundancy-replication-backup.md#manage-capacity-with-over-provisioning).

#### Behavior when all regions are healthy

This section describes what to expect when you configure an Azure Cosmos DB account with multiple read regions, and all regions are operational.

- **Cross-region operation:** Your application configures the region that should receive read operations. For more information about how region selection works, see [Diagnose and troubleshoot the availability of Azure Cosmos DB SDKs in multiregional environments](/azure/cosmos-db/troubleshoot-sdk-availability). To learn how to configure your application, see [Configure multi-region writes in applications that use Azure Cosmos DB](/azure/cosmos-db/how-to-multi-master).

    All write operations are directed to your account's write region.

- **Cross-region data replication:** All write operations occur in your account's primary region. Writes are replicated asynchronously to the other read regions. For information about the maximum replication lag, see [Potential data loss during region outages](#potential-data-loss-during-region-outages).

#### Behavior during a region failure

This section describes what to expect when you configure an Azure Cosmos DB account with multiple read regions, and there's an outage in one of the account's regions.

- **Detection and response:** Responsibility for detecting the outage and responding depends on the type of failover your account uses.

    - *Service-managed failover:* Microsoft automatically detects the outage and initiates a failover of your account. Your application doesn't need to take any action.

    - *PPAF:* Microsoft automatically detects the outage and initiates a failover of some partitions, if appropriate. Your application doesn't need to take any action.

    - *Manual failover:* Your application detects the loss of the region. You can perform a manual (forced) failover. For detailed steps, see [Perform forced failover for your Azure Cosmos DB Account](/azure/cosmos-db/how-to-manage-database-account#perform-forced-failover-for-your-azure-cosmos-db-account).

        If there's an outage of your account's write region, avoid performing a *switch write region* operation. Write region switches don't succeed if there's an outage of the source or destination region. The reason is that the switch procedure includes a consistency check that requires connectivity between the regions.

[!INCLUDE [Region down notification (Service Health and Resource Health)](./includes/reliability-region-down-notification-service-resource-include.md)]

- **Active requests:** Any active requests might be terminated and need to be retried by the client after failover completes. If your clients handle [transient faults](#resilience-to-transient-faults) appropriately by retrying after a short period of time, they typically avoid significant impact.

- **Expected data loss:** If the account's write region experiences an outage, any unreplicated writes might be lost after the failover completes. For information about the maximum data loss expected during a region outage, see [Potential data loss during region outages](#potential-data-loss-during-region-outages).

    An outage in a read region doesn't cause data loss.

- **Expected downtime:** The amount of downtime your account experiences depends on the type of failover your account uses.

    - *Service-managed failover:* Microsoft is responsible for initiating service-managed failover, and the downtime your account experiences is based on the time it takes Microsoft to declare the outage and initiate failover.

        > [!WARNING]
        > **Note to PG:** Can we give an approximate indication of how long this is expected to take (e.g. "a few minutes")?

    - *PPAF:*

        > [!WARNING]
        > **Note to PG:** Is there anything we can say here about how long downtime might be when PPAF is enabled?

    - *Forced failover:* Downtime depends on:
        - How long it takes you to discover the outage and initiate a failover
        - How long the failover takes, which is usually a few seconds.

            > [!WARNING]
            > **Note to PG:** Please verify that a forced failover is likely to take a few seconds.

- **Redistribution:** <!-- TODO -->

    Read region outages: The affected region is disconnected and marked offline. No changes are required in your application code to handle read region outages. The [Azure Cosmos DB SDKs](/azure/cosmos-db/nosql/sdk-dotnet-v3) redirect read operations to the next available region in the preferred region list. If none of the regions in the preferred region list are available, read operations automatically fall back to the current write region.

    Write region outages: Azure Cosmos DB promotes one of the account's secondary regions to be the new primary write region when *service-managed failover* is configured on the Azure Cosmos DB account. The failover occurs to another region in the order of region priority that you specify.

#### Region recovery

Microsoft must bring a region back online. The region recovery processes are different depending on the type of failover:

- **After a service-managed failover:** Microsoft automatically brings the region online but this process can take several days.

- **After a forced failover:** During a real region outage, Microsoft automatically brings the region online but this process can take several days.

    If you use this approach for a failover drill, open a support case to ask Microsoft to bring the region online.

After the region is online, the actions you take are different depending on whether the outage was in a read region or a write region.

- **For read region outages:** When the affected region is back online, it syncs with the current write region and is available again to serve read requests after it has fully caught up. Subsequent reads are redirected to the recovered region without requiring any changes to your application code. During both failover and rejoining of a previously failed region, Azure Cosmos DB continues to honor read consistency guarantees.

- **For write region outages:** When the affected region is back online, the region shows as "online" in the Azure portal, and becomes available as a read region. At this point, it is safe to switch back to the recovered region as the write region by using [PowerShell, the Azure CLI, or the Azure portal](/azure/cosmos-db/how-to-manage-database-account#perform-manual-failover-on-an-azure-cosmos-db-account). There is *no data or availability loss* before, while, or after you switch the write region. Your application continues to be highly available.

    > [!WARNING]
    > In the event of a write region outage, where the Azure Cosmos DB account promotes a secondary region to be the new primary write region via *service-managed failover*, the original write region will **not be be promoted back as the write region automatically** once it is recovered. It's your responsibility to switch back to the recovered region as the write region, once it's safe to do so (as described above).

#### Test for region failures

Even if your Azure Cosmos DB account is highly available, your application might not be correctly designed to remain highly available when a region failover occurs. To test the end-to-end high availability of your application as a part of your application testing or disaster recovery (DR) drills, temporarily disable service-managed failover for the account. Invoke [manual failover by using PowerShell, the Azure CLI, or the Azure portal](/azure/cosmos-db/how-to-manage-database-account#perform-forced-failover-for-your-azure-cosmos-db-account), and then monitor your application. After you complete the test, you can fail back over to the primary region and restore service-managed failover for the account.

### Multiple write regions

TODO

#### Requirements

**Region support:** You can configure any Azure region as a read or write region for your Azure Cosmos DB account.

#### Cost

Adding an additional write region to an Azure Cosmos DB account increases your existing costs for each region. For more information, see [Azure Cosmos DB pricing](https://azure.microsoft.com/pricing/details/cosmos-db/).

#### Configure multiple write regions

<!-- TODO -->

TODO [Configure multiple write regions](/azure/cosmos-db/how-to-manage-database-account#configure-multiple-write-regions).

For multiple write regions, your app needs to be configured appropriately. See [Configure multi-region writes in applications that use Azure Cosmos DB](/azure/cosmos-db/how-to-multi-master).

#### Capacity planning and management

If your application spreads requests across regions and one region goes offline, the remaining regions experience higher request volume. Use autoscale throughput to dynamically adjust capacity based on demand. If you use provisioned throughput, plan for adequate capacity to handle the loss of a region without service degradation, and consider over-provisioning. For more information, see [Manage capacity with over-provisioning](./concept-redundancy-replication-backup.md#manage-capacity-with-over-provisioning).

#### Behavior when all regions are healthy

This section describes what to expect when you configure an Azure Cosmos DB account with multiple write regions, and all regions are operational.

- **Cross-region operation:** When an account is configured with multiple write regions, your application configures the region that should be used for read and write operations. For more information about how region selection works, see [Diagnose and troubleshoot the availability of Azure Cosmos DB SDKs in multiregional environments](/azure/cosmos-db/troubleshoot-sdk-availability). To learn how to configure your application, see [Configure multi-region writes in applications that use Azure Cosmos DB](/azure/cosmos-db/how-to-multi-master).

- **Cross-region data replication:** <!-- TODO async -->

    When an account is configured for multiple write regions, applications in different regions might make changes that conflict. Cosmos DB provides conflict resolution capabilities. For more information, see [Conflict types and resolution policies when using multiple write regions](/azure/cosmos-db/conflict-resolution-policies). To learn about how to configure your own conflict resolution policy, see [Manage conflict resolution policies in Azure Cosmos DB](/azure/cosmos-db/how-to-manage-conflicts).

#### Behavior during a region failure

This section describes what to expect when you configure an Azure Cosmos DB account with multiple write regions, and there's an outage in one of the account's read or write regions.

- **Detection and response:** Your application detects the loss of the region. Azure Cosmos DB SDKs provide automatic region selection capabilities that route read and write operations to healthy regions.

[!INCLUDE [Region down notification (Service Health and Resource Health)](./includes/reliability-region-down-notification-service-resource-include.md)]

- **Active requests:** <!-- TODO -->

- **Expected data loss:** TODO

    <!-- TODO repeated -->
    [Potential data loss during region outages](#potential-data-loss-during-region-outages)

- **Expected downtime:** <!-- TODO -->

- **Redistribution:** <!-- TODO -->

    The affected region is disconnected and marked offline. No changes are required in your application code to read region outages. The [Azure Cosmos DB SDKs](/azure/cosmos-db/nosql/sdk-dotnet-v3) redirect read and write operations to the next available region in the preferred region list.

#### Region recovery

The region recovery processes are different depending on whether the outage was in a read region or a write region.

- **For read region outages:** When the affected region is back online, it syncs with the current write region and is available again to serve read requests after it has fully caught up. Subsequent reads are redirected to the recovered region without requiring any changes to your application code. During both failover and rejoining of a previously failed region, Azure Cosmos DB continues to honor read consistency guarantees.

<!-- TODO verify the below -->
- **For write region outages:** When the affected region is back online, the region shows as "online" in the Azure portal, and becomes available as a read region. At this point, it is safe to switch back to the recovered region as the write region by using [PowerShell, the Azure CLI, or the Azure portal](/azure/cosmos-db/how-to-manage-database-account#perform-manual-failover-on-an-azure-cosmos-db-account). There is *no data or availability loss* before, while, or after you switch the write region. Your application continues to be highly available.

    Any write data that wasn't replicated when the region failed is made available through the [conflict feed](/azure/cosmos-db/how-to-manage-conflicts#read-from-conflict-feed). Applications can read the conflict feed, resolve the conflicts based on the application-specific logic, and write the updated data back to the Azure Cosmos DB container as appropriate.

    > [!WARNING]
    > In the event of a write region outage, where the Azure Cosmos DB account promotes a secondary region to be the new primary write region via *service-managed failover*, the original write region will **not be be promoted back as the write region automatically** once it is recovered. It is your responsibility to switch back to the recovered region as the write region using [PowerShell, the Azure CLI, or the Azure portal](/azure/cosmos-db/how-to-manage-database-account#perform-manual-failover-on-an-azure-cosmos-db-account), once it's safe to do so (as described above).

#### Test for region failures

<!-- TODO -->

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
- Accounts that use multiple write regions (Business Critical tier)

## Related content

- [Azure reliability](/azure/reliability/overview)
- [Azure Cosmos DB overview](/azure/cosmos-db/overview)
- [Consistency levels in Azure Cosmos DB](/azure/cosmos-db/consistency-levels)
- [Global data distribution with Azure Cosmos DB](/azure/cosmos-db/distribute-data-globally)
- [Diagnose and troubleshoot the availability of Azure Cosmos DB SDKs in multiregional environments](/azure/cosmos-db/troubleshoot-sdk-availability)
