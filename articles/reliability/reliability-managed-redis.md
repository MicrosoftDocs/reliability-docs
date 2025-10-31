---
title: Reliability in Azure Managed Redis
description: Learn about resiliency in Azure Managed Redis, including availability zones and multi-region deployments.
ms.author: anaharris
author: anaharris-ms
ms.topic: reliability-article
ms.custom: subject-reliability
ms.service: azure-managed-redis
ms.date: 10/30/2025
ai-usage: ai-assisted

#Customer intent: As an engineer responsible for business continuity, I want to understand the details of how Azure Managed Redis works from a reliability perspective and plan both resiliency and recovery strategies in alignment with the exact processes that Azure services follow during different kinds of situations.
---

# Reliability in Azure Managed Redis

Azure Managed Redis provides fully integrated and managed Redis Enterprise on Azure, offering high-performance in-memory data storage for applications. This service is built for enterprise workloads requiring ultra-low latency, high throughput, and advanced data structures. 

[!INCLUDE [Shared responsibility](includes/reliability-shared-responsibility-include.md)]

This article describes reliability in [Azure Managed Redis](/azure/redis/overview), including resilience to transient faults, availability zone failures, and region-wide failures. The article also describes backup strategies and the service-level agreement (SLA).

## Production deployment recommendations

To ensure high reliability for your production managed Redis caches, we recommend that you:

> [!div class="checklist"]
> - **Enable high availability**, which deploys multiple nodes for your cache.
> - **Enable zone redundancy** by deploying a highly available cache into a region with availability zones.
> - **Consider implementing active geo-replication** for mission-critical workloads that require cross-region failover.

## Reliability architecture overview
<!-- TODO -->

Azure Managed Redis is built on Redis Enterprise and provides reliability through high availability configurations and replication capabilities. Understanding the architectural components helps you design for optimal reliability.

**Virtual Machines (Nodes)**: An Azure Managed Redis instance is constructed using multiple virtual machines (VMs), also called "nodes," with separate and private IP addresses. Each VM serves as an independent compute unit in the cluster. Some SKUs use two nodes, while larger SKUs use more nodes to maximize vCPUs and memory capacity. The service abstracts the specific number of nodes used in each configuration to avoid complexity and ensure optimal configurations.

**Redis Shards**: Each virtual machine runs multiple Redis server processes called "shards" in parallel. Unlike community Redis which runs a single Redis process per node due to its single-threaded design, Azure Managed Redis can run multiple shards per node for efficient vCPU utilization and higher performance. Each shard handles a subset of the data based on key hash slots (16,384 slots total in the key space). The number of shards per SKU is optimized for the available vCPUs and is not user-configurable.

**Primary and Replica Shards**: Each shard exists in both a primary and replica configuration. Primary shards accept write operations and use more CPU resources. Replica shards maintain synchronized copies of data from their corresponding primary shards to provide redundancy. Primary and replica shards are strategically distributed across different nodes—not all primary shards reside on the same node. This distribution ensures that if one node fails, the cluster maintains availability by promoting replica shards on healthy nodes to primary status.

**High-Performance Proxy**: Each node runs a high-performance proxy process that manages the shards, handles connection management, and triggers self-healing actions. The proxy provides intelligent routing of client requests to the appropriate shards based on key hash slots and manages the complexity of distributed operations.

<!-- mention cluster policies -->

**Sources:**
- [Azure Managed Redis architecture](https://learn.microsoft.com/en-us/azure/redis/architecture) - Detailed architecture components including VMs, nodes, shards, and clustering
- [Failover and patching for Azure Managed Redis](https://learn.microsoft.com/en-us/azure/redis/failover) - Shard-level architecture and failover mechanisms

## Resilience to transient faults

[!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)]

Follow these recommendations for managing transient faults when using Azure Managed Redis:

- **Use SDK configurations** that automatically retry when transient faults occur, and that use appropriate backoff and timeout periods. Consider using the [Retry pattern](/azure/architecture/patterns/retry) and [Circuit Breaker pattern](/azure/architecture/patterns/circuit-breaker) in your applications.
- **Use connection multiplexing** to maintain persistent connections and reduce the impact of temporary connection losses.
- **Design for cache-aside patterns** where your application can continue operating with degraded performance when Redis is temporarily unavailable by falling back to the primary data store.

## Resilience to availability zone failures

[!INCLUDE [Resilience to availability zone failures](includes/reliability-availability-zone-description-include.md)]

Azure Managed Redis caches can be made *zone-redundant*, which automatically distributes the cache nodes across multiple availability zones within a region. Zone redundancy reduces the risk of data center or availability zone outages causing your cache to be unavailable.

<!-- TODO diagram -->

To make a cache zone-redundant, you must deploy it into a supported region and enable high availability configuration. In regions without availability zones, the high availability configuration still creates at least two nodes but they aren't in separate zones.

### Requirements

- **Region support.** Zone-redundant Azure Managed Redis caches can be deployed into any region that supports availability zones and where the service is available. For the most current list of regions that support availability zones, see [Azure regions with availability zones](regions-list.md). For the list of regions that support Azure Managed Redis, see [Product availability by region](https://azure.microsoft.com/explore/global-infrastructure/products-by-region/table).

- **High availability configuration.** You must enable high availability configuration on your cache for it to be zone-redundant.

- **Tiers.** All Azure Managed Redis tiers support availability zones.

### Cost

Zone redundancy requires that your cache is configured for high availability, which provides two nodes for your cache. You're billed for both nodes. For more information, see [Azure Managed Redis pricing](https://azure.microsoft.com/pricing/details/managed-redis/)

### Configure availability zone support

- **New instances:** When you create a new Azure Managed Redis instance, enable high availability configuration and deploy it into a region with availability zones. Then, it automatically includes zone redundancy by default. There's no need for you to perform any more configuration.

  For detailed steps, see [Quickstart: Create an Azure Managed Redis Instance](../redis/quickstart-create-managed-redis.md).

- **Existing instances:** To configure an existing Azure Managed Redis instance to be zone-redundant, ensure it's deployed in a region that supports availability zones, and enable high availability on the cache.

- **Disable:** Zone redundancy can't be disabled on existing instances.

### Behavior when all zones are healthy

This section describes what to expect when a managed Redis cache is zone-redundant and all availability zones are operational:

- **Traffic routing between zones:** Shards are distributed across nodes based on your cluster policy. Your cluster policy also determines how traffic is routed to each node. Zone redundancy doesn't change how traffic is routed.

- **Data replication between zones:** Shards are replicated across nodes automatically, and use asynchronous replication. <!-- PG: Can we give an idea of the replication frequency/lag? -->

### Behavior during a zone failure

This section describes what to expect when a managed Redis cache is zone-redundant and one or more availability zones are unavailable:

- **Detection and response:** Azure Managed Redis is responsible for detecting a failure in an availability zone. You don't need to do anything to initiate a zone failover.

[!INCLUDE [Availability zone down notification (Service Health)](./includes/reliability-availability-zone-down-notification-service-include.md)]

- **Active requests:** In-flight requests might be dropped and should be retried. Applications should [implement retry logic](#transient-faults) to handle these temporary interruptions.

- **Expected data loss:** <!-- PG: Please advise -->

- **Expected downtime:** A small amount of downtime, typically, a few seconds, might occur while shards fail over to nodes in healthy zones. When you design applications, follow practices for [transient fault handling](#transient-faults).

- **Traffic rerouting:** Azure Managed Redis automatically redirects traffic to nodes in healthy zones.

### Zone recovery

When the affected availability zone recovers, Azure Managed Redis automatically restores operations to that zone. The Azure platform fully manages this process and doesn't require any customer intervention.

### Test for zone failures

Because Azure Managed Redis fully manages traffic routing, failover, and failback for zone failures, you don't need to validate availability zone failure processes or provide any further input.

## Resilience to region-wide failures

Azure Managed Redis provides native multi-region support through *active geo-replication*, which enables you to link multiple Azure Managed Redis instances across different Azure regions into a single replication group. You can then configure your own failover approach between the instances.

### Active geo-replication

When you use [active geo-replication](../redis/how-to-active-geo-replication.md), applications can read from and write to any cache instance in the group, with changes automatically synchronized across all regions. The service supports active-active replication patterns where each region can handle both read and write operations simultaneously. When conflicts occur due to concurrent writes in different regions, the service automatically resolves them using predetermined conflict resolution algorithms without requiring manual intervention. This approach provides resiliency to region failures while maintaining low-latency access for globally distributed applications.

You're responsible for configuring your client applications so that, if any regional instance fails, they can redirect their requests to a healthy instance. 

#### Requirements

- **Region support** Azure Managed Redis active geo-replication can be configured between any Azure regions where the service is available.

- **Instance configuration:** Active geo-replication requires Azure Managed Redis instances of the same tier and size across all participating regions. All cache instances in a replication group must be configured with identical settings including persistence options, modules, and clustering policies.

- **Other requirements:** Your cache instances must meet other requirements. For more information, see [Active geo-replication prerequisites](../redis/how-to-active-geo-replication.md#active-geo-replication-prerequisites).

#### Considerations

- **Failover responsibility:** When you use active geo-replication, **you're responsible for failover between cache instances**. You should prepare and configuring your application to handle failover. Failover involves preparation and might require you complete multiple steps. For more information, see [Force-unlink if there's a region outage](../redis/how-to-active-geo-replication.md#force-unlink-if-theres-a-region-outage).

- **Eventual consistency:** Applications should be designed to handle eventual consistency scenarios, because changes can take time to propagate across all regions depending on network conditions and geographic distance. During region outages, you may experience more data inconsistencies until connectivity is restored and synchronization completes.

#### Cost

When you enable active geo-replication, you are billed for each Azure Managed Redis instance in every region within the replication group. Additionally, you might incur data transfer charges for cross-region replication traffic between regions. For more information about pricing, see [Azure Managed Redis pricing](https://azure.microsoft.com/pricing/details/managed-redis/) and [Bandwidth pricing details](https://azure.microsoft.com/pricing/details/bandwidth/).

#### Configure multi-region support

- **Create a new geo-replicated cache instance**: Configure active geo-replication during cache provisioning by specifying a replication group and linking multiple instances. For more information, see [Create or join an active geo-replication group](../redis/how-to-active-geo-replication.md#create-or-join-an-active-geo-replication-group).

- **Enable an existing cache instance for geo-replication**: You can add an existing cache instance to an active geo-replication group. For more information, see [Add an existing instance to an active geo-replication group](../redis/how-to-active-geo-replication.md#add-an-existing-instance-to-an-active-geo-replication-group)

- **Disable geo-replication on a cache instance**: Remove an instance from a geo-replication group by deleting the cache instance. The remaining instances automatically reconfigure themselves.

#### Behavior when all regions are healthy

This section describes what to expect when instances are configured to use active geo-replication and all regions are operational.

- **Traffic routing between regions**: Applications can connect to any cache instance in the replication group and perform both read and write operations. Traffic routing is handled by the application, allowing you to direct clients to the nearest region for optimal latency. Azure Managed Redis doesn't provide automatic traffic routing between regions.

- **Data replication between regions**: The service uses asynchronous replication between regions to maintain eventual consistency. Write operations are immediately committed in the local region and then propagated to other regions in the background. Conflict-free replicated data types (CRDTs) ensure that concurrent writes in different regions are automatically merged.

#### Behavior during a region failure

This section describes what to expect when instances are configured to use active geo-replication and there's an outage in one region:

- **Detection and response**: You're responsible for detecting the failure of a cache instance, and deciding when to fail over. You can monitor the health of a geo-replicated cluster, which can help you to decide when to begin failover. For more information, see [Geo-replication metric](../redis/how-to-active-geo-replication.md#geo-replication-metric)

  Failover requires that you perform multiple steps. For more detail, see [Force-unlink if there's a region outage](../redis/how-to-active-geo-replication.md#force-unlink-if-theres-a-region-outage).

[!INCLUDE [Region down notification (Service Health)](./includes/reliability-region-down-notification-service-include.md)]

- **Active requests**: Requests to the failed region are terminated and must be handled by your application's failover logic. Applications should implement retry policies that can redirect traffic to healthy caches.

- **Expected data loss**: Due to asynchronous replication between regions, some recent writes to the failed region may be lost if they had not yet been replicated to other regions. The amount of potential data loss depends on replication lag at the time of failure. <!-- PG: Can we give a rough idea? -->

- **Expected downtime**: Applications experience downtime only for the duration needed to detect the failure and redirect traffic to healthy regions. This typically ranges from seconds to a few minutes depending on your application's health check and failover configuration.

- **Traffic rerouting**: You're responsible for implementiing logic in your applications to detect region failures and route traffic to healthy regions. This can be accomplished through health checks, circuit breaker patterns, or external load balancing solutions.

#### Region recovery

When a failed region recovers, Azure Managed Redis automatically reintegrates instances in that region into the active geo-replication group without requiring your intervention. The service automatically begins synchronizing data from healthy instances. During this process, the recovered instance gradually catches up with changes that occurred during the outage. Once synchronization is complete, the recovered instances becomes fully active and can handle both read and write operations.

You're responsible for reconfiguring your application to route traffic back to the recovered region instance.

#### Test for region failures

You should regularly test your appliation's failover procedures. It's important that your application can fail over between instances, and that it stays within your business requirements for downtime while doing so. It's also important that you test your overall response processes, including any reconfiguration of firewalls and other infrastructure, and your recovery process.


## Backup and recovery

Azure Managed Redis provides backup capabilities to protect against data loss scenarios that other reliability features may not address. Backups provide protection against scenarios such as data corruption, accidental deletion, or configuration errors.

Azure Managed Redis supports backup of your data by using the [import and export functionality](../redis/how-to-import-export-data.md), which saves backup files to Azure Blob Storage. You can configure geo-redundant storage on your Azure Storage account, or you can copy or move the backup blobs to other locations for further protection.

There's no built-in backup scheduler, but you can develop your own automation processes that use the Azure CLI or Azure PowerShell to initiate export operations.

Backup recovery is performed by importing a backup file to an Azure Managed Redis instance.

## Service-level agreement

[!INCLUDE [SLA description](includes/reliability-service-level-agreement-include.md)]

To be eligible for availability SLAs for Azure Managed Redis:
- You must enable high availability configuration.
- You must not initiate any product features or management actions that are documented to produce temporary unavailability.

Higher availability SLAs apply when your instance is zone-redundant. In some tiers, you can be eligible for a higher availability SLA when you have deployed zone-redundant instances into at least three regions using active geo-replication.

## Related content

- [What are availability zones?](/azure/reliability/availability-zones-overview)
- [Azure reliability](/azure/reliability/overview)  
- [Resiliency in Azure](/azure/architecture/framework/resiliency/overview)
- [Failover and patching for Azure Managed Redis](../redis/failover.md)
