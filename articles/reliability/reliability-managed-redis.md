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

**Sources:**
- [Azure Managed Redis architecture](https://learn.microsoft.com/en-us/azure/redis/architecture) - Detailed architecture components including VMs, nodes, shards, and clustering
- [Failover and patching for Azure Managed Redis](https://learn.microsoft.com/en-us/azure/redis/failover) - Shard-level architecture and failover mechanisms

## Resilience to transient faults
<!-- TODO -->

[!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)]

Azure Managed Redis is designed to handle transient faults through several built-in mechanisms that ensure robust operation during temporary service disruptions. The service automatically implements connection pooling, retry logic, and circuit breaker patterns to maintain application resilience.

Here are key recommendations for managing transient faults when using Azure Managed Redis:

- **Implement exponential backoff retry policies** with jitter to prevent overwhelming the service during recovery. Configure retry attempts with increasing delays between 1-10 seconds for optimal recovery patterns.
- **Use connection multiplexing** through libraries like StackExchange.Redis to maintain persistent connections and reduce the impact of temporary connection losses.
- **Configure appropriate timeouts** for Redis operations, typically setting command timeouts between 1-5 seconds to balance responsiveness with fault tolerance.
- **Implement circuit breaker patterns** in your application code to temporarily stop making requests to a failing Redis instance and allow it time to recover.
- **Monitor connection health** using built-in health checks and implement automatic connection recovery mechanisms in your client applications.
- **Design for cache-aside patterns** where your application can continue operating with degraded performance when Redis is temporarily unavailable by falling back to the primary data store.

<!-- mentin cluster policies -->

The service handles network-level transient faults through automatic reconnection mechanisms and maintains data consistency during brief interruptions. For applications using active geo-replication, temporary network partitions between regions are automatically resolved once connectivity is restored.

**Sources:**
- [Failover and patching for Azure Managed Redis](https://learn.microsoft.com/en-us/azure/redis/failover) - Transient fault handling details
- [Best practices for Azure Managed Redis](https://learn.microsoft.com/en-us/azure/redis/best-practices-performance) - Client-side fault handling recommendations

## Resilience to availability zone failures

[!INCLUDE [Resilience to availability zone failures](includes/reliability-availability-zone-description-include.md)]

Azure Managed Redis caches can be made *zone-redundant*, which automatically distributes the cache nodes across multiple availability zones within a region. Zone redundancy reduces the risk of data center or availability zone outages causing your cache to be unavailable.

<!-- TODO diagram -->

To make a cache zone-redundant, you must deploy it into a supported region and enable high availability configuration. In regions without availability zones, the high availability configuration still creates at least two nodes but they aren't in separate zones.

### Requirements

- **Region support.** Zone-redundant Azure Managed Redis caches can be deployed into any region that supports availability zones. For the most current list of regions that support availability zones, see [Azure regions with availability zones](regions-list.md).

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
<!-- TODO -->

Azure Managed Redis provides native multi-region support through active geo-replication, enabling you to create globally distributed Redis deployments with automatic conflict resolution and eventual consistency across regions.

Active geo-replication allows you to link multiple Azure Managed Redis instances across different Azure regions into a single replication group. This creates a multi-master configuration where applications can read from and write to any cache instance in the group, with changes automatically synchronized across all regions using conflict-free replicated data types (CRDTs).

The service supports active-active replication patterns where each region can handle both read and write operations simultaneously. When conflicts occur due to concurrent writes in different regions, the service automatically resolves them using predetermined conflict resolution algorithms without requiring manual intervention. This approach provides both high availability and disaster recovery capabilities while maintaining low-latency access for globally distributed applications.

Unlike passive geo-replication solutions, Azure Managed Redis active geo-replication does not require manual failover procedures. Applications can seamlessly switch between regions during outages, and the service maintains data consistency across all healthy regions in the replication group.

**Sources:**
- [Configure active geo-replication for Azure Managed Redis instances](https://learn.microsoft.com/en-us/azure/redis/how-to-active-geo-replication) - Multi-region capabilities overview

### Requirements

- **Region support** Azure Managed Redis active geo-replication can be configured between any Azure regions where the service is available. You can create replication groups that span multiple continents, though performance is optimized when regions are geographically closer due to network latency considerations. For optimal performance, select regions with low inter-region latency. The service supports replication across any combination of supported regions, allowing for flexible disaster recovery and global distribution strategies. For regional availability, see [Products available by region](https://azure.microsoft.com/global-infrastructure/services/?products=redis-cache&regions=all).

 - **Tier:** Active geo-replication requires Azure Managed Redis instances of the same tier and size across all participating regions. All cache instances in a replication group must be configured with identical settings including persistence options, modules, and clustering policies.

- **Clustering policy:** The clustering policy cannot be changed after creating a cache instance, so all caches in a geo-replication group must be created with compatible clustering configurations from the beginning.

**Sources:**
- [Configure active geo-replication for Azure Managed Redis instances](https://learn.microsoft.com/en-us/azure/redis/how-to-active-geo-replication) - Configuration requirements

### Considerations

When using active geo-replication, network latency between regions affects replication performance and potential conflict resolution timing. Write-heavy workloads may experience higher resource utilization due to cross-region synchronization overhead.

Applications should be designed to handle eventual consistency scenarios, as changes may take time to propagate across all regions depending on network conditions and geographic distance. During regional outages, you may experience temporary data inconsistencies until connectivity is restored and synchronization completes.

**Sources:**
- [Configure active geo-replication for Azure Managed Redis instances](https://learn.microsoft.com/en-us/azure/redis/how-to-active-geo-replication) - Design considerations

### Cost

When you enable active geo-replication, you are billed for each Azure Managed Redis instance in every region within the replication group. Additionally, you incur data transfer charges for cross-region replication traffic between regions.

For more information about pricing, see [Azure Managed Redis pricing](https://azure.microsoft.com/pricing/details/managed-redis/) and [Bandwidth pricing details](https://azure.microsoft.com/pricing/details/bandwidth/).

**Sources:**
- [Azure Managed Redis pricing](https://azure.microsoft.com/pricing/details/managed-redis/) - Multi-region pricing details

### Configure multi-region support

Active geo-replication must be configured during the creation of your Azure Managed Redis instances. You cannot add existing cache instances to a geo-replication group after they have been created.

- **Create**: Configure active geo-replication during cache provisioning by specifying a replication group and linking multiple instances. See [Configure active geo-replication for Azure Managed Redis instances](https://learn.microsoft.com/en-us/azure/redis/how-to-active-geo-replication).
- **Disable**: Remove an instance from a geo-replication group by deleting the cache instance. The remaining instances automatically reconfigure themselves.
- **Migrate**: Existing single-region caches cannot be migrated to use geo-replication. You must create new geo-replicated caches and migrate your data.

**Sources:**
- [Configure active geo-replication for Azure Managed Redis instances](https://learn.microsoft.com/en-us/azure/redis/how-to-active-geo-replication) - Configuration procedures

### Capacity planning and management

Azure Managed Redis with active geo-replication automatically manages capacity across regions without requiring customer intervention. The service handles load balancing and resource allocation across all instances in the replication group.

Consider the cumulative resource requirements across all regions when planning capacity, as each region maintains a full copy of your data. Monitor resource utilization across all regions to ensure optimal performance during normal operations and regional failover scenarios.

**Sources:**
- [Configure active geo-replication for Azure Managed Redis instances](https://learn.microsoft.com/en-us/azure/redis/how-to-active-geo-replication) - Capacity planning guidance

### Behavior when all regions are healthy

During normal operations when all regions in the geo-replication group are healthy, Azure Managed Redis maintains active-active replication across all instances.

**Traffic routing between regions**: Applications can connect to any cache instance in the replication group and perform both read and write operations. Traffic routing is typically handled at the application level, allowing you to direct clients to the nearest region for optimal latency. The service does not provide automatic traffic routing between regions.

**Data replication between regions**: The service uses asynchronous replication between regions to maintain eventual consistency. Write operations are immediately committed in the local region and then propagated to other regions in the background. Conflict-free replicated data types (CRDTs) ensure that concurrent writes in different regions are automatically merged without data loss.

**Sources:**
- [Configure active geo-replication for Azure Managed Redis instances](https://learn.microsoft.com/en-us/azure/redis/how-to-active-geo-replication) - Normal operations details

### Behavior during a region failure

When a region becomes unavailable, Azure Managed Redis continues operating in the remaining healthy regions with automatic conflict resolution and data synchronization resuming once connectivity is restored.

- **Detection and response**: Regional failures are automatically detected through Azure's monitoring infrastructure. The service continues operating in healthy regions without requiring customer intervention. Applications should be configured to route traffic away from the failed region to available regions.

[!INCLUDE [Region down notification (Service Health and Resource Health)](./includes/reliability-region-down-notification-service-resource-include.md)]

- **Active requests**: Requests to the failed region are terminated and must be handled by your application's failover logic. Applications should implement retry policies that can redirect traffic to healthy regions in the replication group.

- **Expected data loss**: Due to asynchronous replication between regions, some recent writes to the failed region may be lost if they had not yet been replicated to other regions. The amount of potential data loss depends on replication lag at the time of failure.

- **Expected downtime**: Applications experience downtime only for the duration needed to detect the failure and redirect traffic to healthy regions. This typically ranges from seconds to a few minutes depending on your application's health check and failover configuration.

- **Traffic rerouting**: Applications must implement their own logic to detect regional failures and route traffic to healthy regions. This can be accomplished through health checks, circuit breaker patterns, or external load balancing solutions.

**Sources:**
- [Configure active geo-replication for Azure Managed Redis instances](https://learn.microsoft.com/en-us/azure/redis/how-to-active-geo-replication) - Regional failure handling

### Region recovery

When a failed region recovers, Azure Managed Redis automatically reintegrates it into the active geo-replication group without requiring customer intervention.

The service automatically detects when the previously failed region becomes available and begins synchronizing data from healthy regions. During this process, the recovered region gradually catches up with changes that occurred during the outage. Once synchronization is complete, the region becomes fully active and can handle both read and write operations.

Applications can begin routing traffic back to the recovered region once they detect its availability through health checks. The failback process is designed to be seamless and does not disrupt operations in other regions.

**Sources:**
- [Configure active geo-replication for Azure Managed Redis instances](https://learn.microsoft.com/en-us/azure/redis/how-to-active-geo-replication) - Failback procedures

### Test for region failures

<!-- After punishing AI for lying, it printed this. May want to inquire further. -->
The official Azure Managed Redis documentation does not provide specific guidance on how to test regional failure scenarios. Documentation is not available for customer-initiated testing procedures or Microsoft-managed testing capabilities for regional failover scenarios.

For application-level preparation guidance, see the region-down experience section above which covers operational preparation strategies for regional outages.

**Sources:**
- No official Microsoft documentation found for Azure Managed Redis regional failure testing procedures

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

### Related content

- [What are availability zones?](/azure/reliability/availability-zones-overview)
- [Azure reliability](/azure/reliability/overview)  
- [Resiliency in Azure](/azure/architecture/framework/resiliency/overview)
- [Failover and patching for Azure Managed Redis](../redis/failover.md)
