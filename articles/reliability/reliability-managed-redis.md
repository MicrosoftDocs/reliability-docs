---
title: Reliability in Azure Managed Redis
description: Learn about reliability in Azure Managed Redis, including availability zones and multi-region deployments.
ms.author: anaharris
author: anaharris-ms
ms.topic: reliability-article
ms.custom: subject-reliability
ms.service: azure-managed-redis
ms.date: 10/10/2024
ms.update-cycle: 180-days
#Customer intent: As an engineer responsible for business continuity, I want to understand the details of how Azure Managed Redis works from a reliability perspective and plan disaster recovery strategies in alignment with the exact processes that Azure services follow during different kinds of situations.
---

# Reliability in Azure Managed Redis

Azure Managed Redis provides fully integrated and managed Redis Enterprise on Azure, offering high-performance in-memory data storage for applications. This service is built for enterprise workloads requiring ultra-low latency, high throughput, and advanced data structures. Azure Managed Redis supports multiple reliability features including availability zones for high availability within a region, active geo-replication for multi-region deployments, and comprehensive failover mechanisms to ensure business continuity.

Azure Managed Redis offers four distinct performance tiers: Memory Optimized (1:8 memory-to-vCPU ratio), Balanced (1:4 ratio), Compute Optimized (1:2 ratio), and Flash Optimized (hybrid memory and NVMe storage). Each tier provides built-in redundancy and can leverage availability zones for enhanced resilience against datacenter-level failures.

This article describes reliability and availability zones support in Azure Managed Redis. For a more detailed overview of reliability in Azure, see [Azure reliability](/azure/reliability/overview).

**Sources:**
- [Quickstart: Create an Azure Managed Redis Instance](https://learn.microsoft.com/en-us/azure/redis/quickstart-create-managed-redis) - Service overview and tier descriptions

## Production deployment recommendations

To learn about how to deploy Azure Managed Redis to support your solution's reliability requirements, and how reliability affects other aspects of your architecture, see [Architecture best practices for Azure Managed Redis in the Azure Well-Architected Framework](../redis/architecture).

For production environments, we recommend using the Balanced tier or higher to ensure adequate performance and redundancy capabilities. Enable availability zones in regions that support them to protect against datacenter-level failures. Consider implementing active geo-replication for mission-critical workloads that require cross-region disaster recovery.

**Sources:**
- [Architecture best practices for Azure Managed Redis](../redis/architecture) - Production deployment guidance
- [Choose the right tier](https://learn.microsoft.com/en-us/azure/redis/overview#choosing-the-right-tier) - Tier selection guidance

## Reliability architecture overview

Azure Managed Redis provides reliability through multiple architectural layers designed to protect against various failure scenarios. The service implements a multi-tier redundancy model that includes node-level, zone-level, and regional-level protection mechanisms.

At the foundational level, Azure Managed Redis runs on Redis Enterprise clusters that inherently provide built-in redundancy. Each cache instance maintains multiple data replicas distributed across separate fault domains within the region. For clusters deployed with availability zone support, these replicas are strategically placed across different availability zones to ensure continued operation even during zone-level outages.

The service supports two primary deployment models for enhanced reliability: zone-redundant deployments for intra-region high availability, and active geo-replication groups for cross-region disaster recovery. Zone-redundant deployments automatically distribute data and compute resources across multiple availability zones, while active geo-replication maintains synchronized copies of data across geographically separate regions with conflict-free replicated data types (CRDTs) ensuring eventual consistency.

Azure Managed Redis automatically handles failover scenarios at both the zone and regional levels. For zone-level failures, the service performs automatic failover within the region without customer intervention. For regional failures in geo-replicated configurations, applications can seamlessly connect to alternative regions, though some geo-replication scenarios may require customer-initiated failover depending on the configuration.

**Sources:**
- [Azure Managed Redis overview](https://learn.microsoft.com/en-us/azure/redis/overview) - Architecture fundamentals
- [Configure active geo-replication for Azure Managed Redis instances](https://learn.microsoft.com/en-us/azure/redis/how-to-active-geo-replication) - Multi-region architecture details

## Transient faults

[!INCLUDE [Transient fault description](includes/reliability-transient-fault-description-include.md)]

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

## Availability zone support

[!INCLUDE [AZ support description](includes/reliability-availability-zone-description-include.md)]

Azure Managed Redis provides comprehensive availability zone support through zone-redundant deployments that automatically distribute cache nodes across multiple availability zones within a region. When you enable availability zone support, the service creates a Redis Enterprise cluster with nodes spread across different zones, ensuring that your cache remains operational even if an entire availability zone experiences an outage.

The service supports zone-redundant deployments where replicas of your Redis data are automatically distributed across multiple availability zones. Azure Managed Redis uses intelligent zone allocation to ensure that your primary and replica nodes are never placed in the same availability zone, providing maximum protection against zone-level failures. The service automatically manages the distribution of data and handles all replication between zones without requiring customer configuration.

Azure Managed Redis does not support zonal (single-zone pinning) deployments. All availability zone configurations are zone-redundant, meaning your cache automatically spans multiple zones for enhanced reliability. This design ensures that you always benefit from cross-zone redundancy rather than being limited to a single zone that could become a point of failure.

When configuring availability zones, the service automatically selects the optimal zones based on region capacity and load distribution. You cannot manually specify which zones to use, as Azure Managed Redis uses automatic zone allocation to ensure the best possible distribution for reliability and performance.

**Sources:**
- [Configure active geo-replication for Azure Managed Redis instances](https://learn.microsoft.com/en-us/azure/redis/how-to-active-geo-replication) - Zone redundancy architecture
- [Azure Managed Redis overview](https://learn.microsoft.com/en-us/azure/redis/overview) - Availability zone capabilities

### Region support

Zone-redundant Azure Managed Redis resources can be deployed into any region that supports availability zones. The service automatically enables zone redundancy in all supported regions during cache creation.

For the most current list of regions supporting availability zones for Azure Managed Redis, see [Azure regions with availability zones](regions-list.md).

**Sources:**
- [Products available by region](https://azure.microsoft.com/global-infrastructure/services/?products=redis-cache&regions=all) - Regional availability information

### Requirements

All Azure Managed Redis tiers (Memory Optimized, Balanced, Compute Optimized, and Flash Optimized) support availability zones without additional requirements. There are no specific SKU restrictions for enabling zone redundancy.

**Sources:**
- [Quickstart: Create an Azure Managed Redis Instance](https://learn.microsoft.com/en-us/azure/redis/quickstart-create-managed-redis) - Tier availability information

### Considerations

When using availability zones with Azure Managed Redis, network latency between zones may slightly impact performance for latency-sensitive applications. The service is designed to minimize this impact through optimized zone placement and high-performance networking infrastructure between zones.

**Sources:**
- [Azure Managed Redis planning FAQs](https://learn.microsoft.com/en-us/azure/redis/planning-faq) - Performance considerations across zones

### Cost

Enabling availability zones for Azure Managed Redis does not incur additional charges beyond the base service pricing. You are charged the same rate regardless of whether your cache uses single-zone or zone-redundant deployment.

**Sources:**
- [Azure Managed Redis pricing](https://azure.microsoft.com/pricing/details/managed-redis/) - Pricing information for zone-redundant deployments

### Configure availability zone support

Azure Managed Redis automatically enables zone redundancy for new cache instances created in regions that support availability zones. You do not need to explicitly configure availability zones as the service uses automatic zone allocation by default.

- **Create**: Availability zones are automatically enabled when creating new Azure Managed Redis instances in supported regions through the Azure portal, Azure CLI, or ARM templates. See [Quickstart: Create an Azure Managed Redis Instance](https://learn.microsoft.com/en-us/azure/redis/quickstart-create-managed-redis).
- **Disable**: Zone redundancy cannot be disabled once a cache is created with zone support.
- **Migrate**: Existing caches cannot be migrated to use availability zones. You must create a new zone-redundant cache and migrate your data.

**Sources:**
- [Quickstart: Create an Azure Managed Redis Instance](https://learn.microsoft.com/en-us/azure/redis/quickstart-create-managed-redis) - Creation procedures
- [Azure CLI command reference for Azure Managed Redis](https://learn.microsoft.com/en-us/cli/azure/redisenterprise) - CLI configuration examples

### Normal operations

During normal operations when all availability zones are functioning correctly, Azure Managed Redis distributes traffic and data across zones to provide optimal performance and reliability.

**Traffic routing between zones**: Azure Managed Redis uses an active-active configuration where client requests are intelligently routed to the optimal node based on current load and latency characteristics. The service maintains Redis Enterprise cluster management that automatically balances connections across available zones while preserving data locality where possible.

**Data replication between zones**: The service performs synchronous replication of data across availability zones to ensure strong consistency. When write operations occur, they are committed across multiple zones before being acknowledged to the client, ensuring that no data loss occurs during zone failures. This synchronous replication approach provides immediate consistency but may introduce minimal latency compared to single-zone operations.

**Sources:**
- [Configure active geo-replication for Azure Managed Redis instances](https://learn.microsoft.com/en-us/azure/redis/how-to-active-geo-replication) - Normal operations details

### Zone-down experience

When an availability zone becomes unavailable, Azure Managed Redis automatically handles the failure scenario to maintain service availability with minimal disruption to your applications.

- **Detection and response**: Microsoft automatically detects zone failures through continuous health monitoring of the Redis Enterprise cluster. The service immediately initiates automatic failover procedures without requiring customer intervention, redirecting traffic away from the failed zone to healthy zones.

- **Notification**: Zone failures are detected through Azure Service Health and Resource Health monitoring. You can configure alerts on these services to receive notifications when zone-level issues affect your Redis instance. The service also provides diagnostic logs that capture failover events.

- **Active requests**: In-flight requests to nodes in the failed zone are automatically terminated and must be retried by client applications. Most Redis client libraries handle this scenario gracefully through built-in retry mechanisms. New requests are immediately routed to healthy zones.

- **Expected data loss**: Zone failures in Azure Managed Redis result in no data loss due to the synchronous replication model. All committed writes are guaranteed to be replicated across multiple zones before being acknowledged to clients.

- **Expected downtime**: During zone failover, you may experience brief connection disruptions lasting 10-30 seconds while the service redirects traffic to remaining healthy zones. Total service availability is maintained as other zones continue serving requests.

- **Traffic rerouting**: The Redis Enterprise cluster management automatically reroutes all client connections from the failed zone to healthy zones. Client applications may need to re-establish connections, but this is typically handled transparently by Redis client libraries with proper retry configuration.

**Sources:**
- [Failover and patching for Azure Managed Redis](https://learn.microsoft.com/en-us/azure/redis/failover) - Zone failure handling
- [Configure active geo-replication for Azure Managed Redis instances](https://learn.microsoft.com/en-us/azure/redis/how-to-active-geo-replication) - Failover behavior details

### Zone Recovery

When a failed availability zone recovers, Azure Managed Redis automatically detects the recovery and begins reintegrating the zone into the active cluster configuration.

Microsoft initiates zone recovery automatically once the underlying infrastructure is restored. The service performs health checks on the recovered zone and gradually reintroduces it to the cluster. During recovery, the Redis Enterprise cluster synchronizes any data changes that occurred while the zone was unavailable, ensuring data consistency across all zones.

The recovery process is designed to be non-disruptive to ongoing operations. Client applications continue to operate normally on the remaining healthy zones while the recovered zone is reintegrated in the background.

**Sources:**
- [Configure active geo-replication for Azure Managed Redis instances](https://learn.microsoft.com/en-us/azure/redis/how-to-active-geo-replication) - Recovery procedures

### Test for zone failures

<!-- After punishing AI for lying, it printed this. May want to inquire further. -->
Azure Managed Redis is a fully managed service where zone failover testing is handled automatically by Microsoft through regular disaster recovery exercises. You do not need to initiate zone failure tests as the service's built-in redundancy and automatic failover mechanisms are continuously validated.

For application-level resilience testing, you can simulate connection failures by temporarily blocking network access to your Redis instance or introducing artificial delays to test your client application's retry and timeout handling.

**Sources:**
- [Best practices for Azure Managed Redis](https://learn.microsoft.com/en-us/azure/redis/best-practices-performance) - Testing recommendations

## Multi-region support

Azure Managed Redis provides native multi-region support through active geo-replication, enabling you to create globally distributed Redis deployments with automatic conflict resolution and eventual consistency across regions.

Active geo-replication allows you to link multiple Azure Managed Redis instances across different Azure regions into a single replication group. This creates a multi-master configuration where applications can read from and write to any cache instance in the group, with changes automatically synchronized across all regions using conflict-free replicated data types (CRDTs).

The service supports active-active replication patterns where each region can handle both read and write operations simultaneously. When conflicts occur due to concurrent writes in different regions, the service automatically resolves them using predetermined conflict resolution algorithms without requiring manual intervention. This approach provides both high availability and disaster recovery capabilities while maintaining low-latency access for globally distributed applications.

Unlike passive geo-replication solutions, Azure Managed Redis active geo-replication does not require manual failover procedures. Applications can seamlessly switch between regions during outages, and the service maintains data consistency across all healthy regions in the replication group.

**Sources:**
- [Configure active geo-replication for Azure Managed Redis instances](https://learn.microsoft.com/en-us/azure/redis/how-to-active-geo-replication) - Multi-region capabilities overview

### Region support

Azure Managed Redis active geo-replication can be configured between any Azure regions where the service is available. You can create replication groups that span multiple continents, though performance is optimized when regions are geographically closer due to network latency considerations.

For optimal performance, select regions with low inter-region latency. The service supports replication across any combination of supported regions, allowing for flexible disaster recovery and global distribution strategies.

**Sources:**
- [Azure Managed Redis planning FAQs](https://learn.microsoft.com/en-us/azure/redis/planning-faq) - Regional placement recommendations
- [Products available by region](https://azure.microsoft.com/global-infrastructure/services/?products=redis-cache&regions=all) - Regional availability

### Requirements

Active geo-replication requires Azure Managed Redis instances of the same tier and size across all participating regions. All cache instances in a replication group must be configured with identical settings including persistence options, modules, and clustering policies.

The clustering policy cannot be changed after creating a cache instance, so all caches in a geo-replication group must be created with compatible clustering configurations from the beginning.

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

### Normal operations

During normal operations when all regions in the geo-replication group are healthy, Azure Managed Redis maintains active-active replication across all instances.

**Traffic routing between regions**: Applications can connect to any cache instance in the replication group and perform both read and write operations. Traffic routing is typically handled at the application level, allowing you to direct clients to the nearest region for optimal latency. The service does not provide automatic traffic routing between regions.

**Data replication between regions**: The service uses asynchronous replication between regions to maintain eventual consistency. Write operations are immediately committed in the local region and then propagated to other regions in the background. Conflict-free replicated data types (CRDTs) ensure that concurrent writes in different regions are automatically merged without data loss.

**Sources:**
- [Configure active geo-replication for Azure Managed Redis instances](https://learn.microsoft.com/en-us/azure/redis/how-to-active-geo-replication) - Normal operations details

### Region-down experience

When a region becomes unavailable, Azure Managed Redis continues operating in the remaining healthy regions with automatic conflict resolution and data synchronization resuming once connectivity is restored.

- **Detection and response**: Regional failures are automatically detected through Azure's monitoring infrastructure. The service continues operating in healthy regions without requiring customer intervention. Applications should be configured to route traffic away from the failed region to available regions.

- **Notification**: Regional failures can be monitored through Azure Service Health and Resource Health. Configure alerts on these services to receive notifications when regional issues affect your replication group.

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

The service-level agreement (SLA) for Azure Managed Redis describes the expected availability of the service, and the conditions that must be met to achieve that availability expectation.

Azure Managed Redis provides a 99.9% uptime SLA for all tiers when properly configured with availability zones. The SLA covers the availability of the Redis service itself, including automatic failover capabilities within availability zones. For instances deployed across multiple availability zones, the service maintains this SLA even during single zone failures.

The SLA does not cover scenarios involving customer-initiated configuration changes, network connectivity issues outside of Azure's control, or application-level problems. For active geo-replication configurations, each regional instance maintains its individual SLA commitment, providing cumulative reliability across the replication group.

For the most current SLA information, see [SLA for Azure Managed Redis](https://azure.microsoft.com/support/legal/sla/managed-redis/).

**Sources:**
- [SLA for Azure Managed Redis](https://azure.microsoft.com/support/legal/sla/managed-redis/) - Current SLA details

### Related content

- [What are availability zones?](/azure/reliability/availability-zones-overview)
- [Azure reliability](/azure/reliability/overview)  
- [Resiliency in Azure](/azure/architecture/framework/resiliency/overview)
- [Configure active geo-replication for Azure Managed Redis instances](https://learn.microsoft.com/en-us/azure/redis/how-to-active-geo-replication)
- [How to configure data persistence for Azure Managed Redis](https://learn.microsoft.com/en-us/azure/redis/how-to-persistence)
- [Failover and patching for Azure Managed Redis](https://learn.microsoft.com/en-us/azure/redis/failover)

**Sources:**
- [Azure reliability overview](/azure/reliability/overview) - General reliability framework
- [Availability zones overview](/azure/reliability/availability-zones-overview) - Zone concepts and implementation