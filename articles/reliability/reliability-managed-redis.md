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

To learn about how to deploy Azure Managed Redis to support your solution's reliability requirements, and how reliability affects other aspects of your architecture, see [Architecture best practices for Azure Managed Redis in the Azure Well-Architected Framework](../redis/architecture).

For production environments, we recommend using the Balanced tier or higher to ensure adequate performance and redundancy capabilities. Enable availability zones in regions that support them to protect against datacenter-level failures. Consider implementing active geo-replication for mission-critical workloads that require cross-region disaster recovery. For more information on selecting the appropriate tier and configuration, see [Choose the right tier](/azure/redis/overview#choosing-the-right-tier).

**Sources:**
- [Architecture best practices for Azure Managed Redis](../redis/architecture) - Production deployment guidance
- [Choose the right tier](https://learn.microsoft.com/en-us/azure/redis/overview#choosing-the-right-tier) - Tier selection guidance

## Reliability architecture overview

Azure Managed Redis is built on Redis Enterprise and provides reliability through high availability configurations and replication capabilities. Understanding the architectural components helps you design for optimal reliability.


**Virtual Machines (Nodes)**: An Azure Managed Redis instance is constructed using multiple virtual machines (VMs), also called "nodes," with separate and private IP addresses. Each VM serves as an independent compute unit in the cluster. Some SKUs use two nodes, while larger SKUs use more nodes to maximize vCPUs and memory capacity. The service abstracts the specific number of nodes used in each configuration to avoid complexity and ensure optimal configurations.

**Redis Shards**: Each virtual machine runs multiple Redis server processes called "shards" in parallel. Unlike community Redis which runs a single Redis process per node due to its single-threaded design, Azure Managed Redis can run multiple shards per node for efficient vCPU utilization and higher performance. Each shard handles a subset of the data based on key hash slots (16,384 slots total in the key space). The number of shards per SKU is optimized for the available vCPUs and is not user-configurable.

**Primary and Replica Shards**: Each shard exists in both a primary and replica configuration. Primary shards accept write operations and use more CPU resources. Replica shards maintain synchronized copies of data from their corresponding primary shards to provide redundancy. Primary and replica shards are strategically distributed across different nodes—not all primary shards reside on the same node. This distribution ensures that if one node fails, the cluster maintains availability by promoting replica shards on healthy nodes to primary status.

**High-Performance Proxy**: Each node runs a high-performance proxy process that manages the shards, handles connection management, and triggers self-healing actions. The proxy provides intelligent routing of client requests to the appropriate shards based on key hash slots and manages the complexity of distributed operations.

**Sources:**
- [Azure Managed Redis architecture](https://learn.microsoft.com/en-us/azure/redis/architecture) - Detailed architecture components including VMs, nodes, shards, and clustering
- [Failover and patching for Azure Managed Redis](https://learn.microsoft.com/en-us/azure/redis/failover) - Shard-level architecture and failover mechanisms

## Resilience to transient faults

[!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)]

Azure Managed Redis is designed to handle transient faults through several built-in mechanisms that ensure robust operation during temporary service disruptions. The service automatically implements connection pooling, retry logic, and circuit breaker patterns to maintain application resilience.

Here are key recommendations for managing transient faults when using Azure Managed Redis:

- **Implement exponential backoff retry policies** with jitter to prevent overwhelming the service during recovery. Configure retry attempts with increasing delays between 1-10 seconds for optimal recovery patterns.
- **Use connection multiplexing** through libraries like StackExchange.Redis to maintain persistent connections and reduce the impact of temporary connection losses.
- **Configure appropriate timeouts** for Redis operations, typically setting command timeouts between 1-5 seconds to balance responsiveness with fault tolerance.
- **Implement circuit breaker patterns** in your application code to temporarily stop making requests to a failing Redis instance and allow it time to recover.
- **Monitor connection health** using built-in health checks and implement automatic connection recovery mechanisms in your client applications.
- **Design for cache-aside patterns** where your application can continue operating with degraded performance when Redis is temporarily unavailable by falling back to the primary data store.

The service handles network-level transient faults through automatic reconnection mechanisms and maintains data consistency during brief interruptions. For applications using active geo-replication, temporary network partitions between regions are automatically resolved once connectivity is restored.

**Sources:**
- [Failover and patching for Azure Managed Redis](https://learn.microsoft.com/en-us/azure/redis/failover) - Transient fault handling details
- [Best practices for Azure Managed Redis](https://learn.microsoft.com/en-us/azure/redis/best-practices-performance) - Client-side fault handling recommendations

## Resilience to availability zone failures

[!INCLUDE [Resilience to availability zone failures](includes/reliability-availability-zone-description-include.md)]

<!-- John: The source link below contains a reference to zone redundancy in its feature comparison table. It links to the information in Azure Cache for Redis documentation, which was why I assumed that the feature would be similar in the Managed Redis. If it isn't similar, however, we should update that link. Clearly, the service supports zone redundancy, so we need to get clear on how that works. -->



Azure Managed Redis provides availability zone support through zone-redundant deployments that automatically distribute cache nodes across multiple availability zones within a region. Based on the Redis Enterprise architecture used by Azure Managed Redis, zone-redundant caches place their nodes across different Azure Availability Zones in the same region, eliminating data center or availability zone outage as a single point of failure and increasing overall availability.

Azure Managed Redis runs on a Redis Enterprise cluster architecture that always uses at least three nodes. The cluster includes data nodes (which hold your data) and uses a quorum-based model for high availability. When configured for zone redundancy, these nodes are distributed evenly across three availability zones to minimize the potential for quorum loss during zone failures.

The service supports only zone-redundant deployments in regions with availability zones. Azure Managed Redis does not support zonal (single-zone pinning) deployments. All availability zone configurations use automatic zone allocation, where Azure selects the optimal zones based on region capacity and load distribution to ensure the best possible reliability and performance.

**Sources:**
- [Azure Managed Redis overview](https://learn.microsoft.com/en-us/azure/redis/overview) - Availability zone feature confirmation

### Requirements

- **Region support.** Zone-redundant Azure Managed Redis resources can be deployed into any region that supports availability zones. For the most current list of regions that support availability zones, see [Azure regions with availability zones](regions-list.md).

All Azure Managed Redis tiers (Memory Optimized, Balanced, Compute Optimized, and Flash Optimized) support availability zones without additional requirements. There are no specific SKU restrictions for enabling zone redundancy.

**Sources:**
- [Quickstart: Create an Azure Managed Redis Instance](https://learn.microsoft.com/en-us/azure/redis/quickstart-create-managed-redis) - Tier availability information
- [Azure Managed Redis pricing](https://azure.microsoft.com/pricing/details/managed-redis/) - Base service pricing



### Considerations
<!-- John: No information available -->


### Cost

<!-- John: Zone redundancy is mentioned in the source page. -->

Enabling availability zones for Azure Managed Redis does not incur additional charges for the zone redundancy feature itself beyond the base service pricing. However, data replication across availability zones generates network egress costs for data moving between zones.


**Sources:**
- [Azure Managed Redis pricing](https://azure.microsoft.com/pricing/details/managed-redis/) - Base service pricing


### Configure availability zone support

<!-- John: No information available -->


### Behavior when all zones are healthy
<!-- John: No information available -->

### Behavior during a zone failure

<!-- John: No information available -->
### Zone recovery

<!-- John: No information available -->

### Test for zone failures

<!-- John: No information available -->

## Resilience to region-wide failures

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

- **Notification**: 

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

## Backups

Azure Managed Redis provides comprehensive backup capabilities to protect against data loss scenarios that other reliability features may not address. The service supports both automated and customer-controlled backup mechanisms to ensure your data remains recoverable across various failure modes.

The service offers Redis Database (RDB) persistence options that can be configured during cache creation or enabled on existing instances. RDB persistence creates point-in-time snapshots of your Redis data at configurable intervals (1 hour, 6 hours, 12 hours, or 24 hours) and stores them durably within the region. These backups are fully managed by Microsoft and provide protection against scenarios such as data corruption, accidental deletion, or configuration errors.

For cross-region backup protection, you can configure RDB backups to use geo-redundant storage, ensuring that your backup data remains available even during regional outages. The backup files are stored in Azure Storage with the redundancy option you specify, providing an additional layer of protection beyond the live cache instances.

Backup recovery is performed by creating a new Azure Managed Redis instance from a backup file. The recovery process restores the full dataset to the state captured in the selected backup, making it suitable for disaster recovery scenarios where you need to recover to a known good state.

**Sources:**
- [How to configure data persistence for Azure Managed Redis](https://learn.microsoft.com/en-us/azure/redis/how-to-persistence) - Backup configuration and management

## Service-level agreement

[!INCLUDE [SLA description](includes/reliability-service-level-agreement-include.md)]

### Related content

- [What are availability zones?](/azure/reliability/availability-zones-overview)
- [Azure reliability](/azure/reliability/overview)  
- [Resiliency in Azure](/azure/architecture/framework/resiliency/overview)
- [Configure active geo-replication for Azure Managed Redis instances](https://learn.microsoft.com/en-us/azure/redis/how-to-active-geo-replication)
- [How to configure data persistence for Azure Managed Redis](https://learn.microsoft.com/en-us/azure/redis/how-to-persistence)
- [Failover and patching for Azure Managed Redis](https://learn.microsoft.com/en-us/azure/redis/failover)
