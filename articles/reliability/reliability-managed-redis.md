---
title: Reliability in Azure Managed Redis
description: Learn how to make Azure Managed Redis resilient to a variety of potential outages and problems, including transient faults, availability zone outages, and region outages.
ms.author: glynnniall
author: glynnniall
ms.topic: reliability-article
ms.custom: subject-reliability
ms.service: azure-managed-redis
ms.date: 01/12/2026
ai-usage: ai-assisted

---

# Reliability in Azure Managed Redis

[Azure Managed Redis](/azure/redis/overview) is a fully managed Azure service based on Redis Enterprise. It provides high-performance, in-memory data storage for applications and is designed for enterprise workloads that require ultra-low latency, high throughput, and advanced data structures.

[!INCLUDE [Shared responsibility](includes/reliability-shared-responsibility-include.md)]

This article describes how to make Azure Managed Redis resilient to a variety of potential outages and problems, including transient faults, availability zone outages, and region outages.

## Production deployment recommendations

To ensure high reliability for your production Azure Managed Redis instances, we recommend that you:

> [!div class="checklist"]
> - **Enable high availability**, which deploys multiple nodes for your cache.
>
> - **Enable zone redundancy** by deploying a highly available cache into a region that has availability zones.
>
> - **Consider implementing active geo-replication** for mission-critical workloads that require cross-region failover.

## Reliability architecture overview

[!INCLUDE [Introduction to reliability architecture overview section](includes/reliability-architecture-overview-introduction-include.md)]

### Logical architecture

Azure Managed Redis is built on Redis Enterprise and provides reliability through high availability configurations and replication capabilities.

You deploy an *instance* of Azure Managed Redis, which is also known as a *cache instance* or a *cache*. Your client applications store and interact with data within the cache by using Redis APIs.

### Physical architecture

When you plan for resiliency in Azure Managed Redis, you must understand the key concepts of nodes and shards.

- **Nodes:** Each cache instance consists of *nodes*, which are virtual machines (VMs). Each VM serves as an independent compute unit in the cluster. You don't see or manage the VMs directly. The platform automatically creates instances, monitors instance health, and replaces any instances that become unhealthy. This set of VMs forms a *cluster*.

  You can set up your instance for high availability. When you enable high availability, Azure Managed Redis ensures that the instance has at least two nodes and automatically replicates data between them. In regions that have availability zones, the service places the nodes into different availability zones. For more information, see [Resilience to availability zone failures](#resilience-to-availability-zone-failures).

  The service abstracts the specific number of nodes in each configuration to avoid complexity and ensure optimal configurations.

- **Shards:** Each node runs multiple Redis server processes known as *shards*. Each shard manages a subset of your cache's data. When you set your cache for high availability, shards automatically distribute and replicate across nodes. You specify a *cluster policy*, which determines how shards distribute across nodes.

For more information, see [Azure Managed Redis architecture](/azure/redis/architecture) and [Failover and patching for Azure Managed Redis](/azure/redis/failover).

## Resilience to transient faults

[!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)]

Follow these recommendations for managing transient faults when you use Azure Managed Redis:

- **Use SDK configurations** that automatically retry when transient faults occur and apply appropriate backoff and timeout periods. Consider using the [Retry pattern](/azure/architecture/patterns/retry) and [Circuit Breaker pattern](/azure/architecture/patterns/circuit-breaker) in your applications.

- **Design for cache-aside patterns**, where your application can continue to operate with degraded performance by falling back to the primary data store when Redis temporarily becomes unavailable.

## Resilience to availability zone failures

[!INCLUDE [Resilience to availability zone failures](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)]

You can make Azure Managed Redis cache instances *zone redundant*, which automatically distributes the cache nodes across multiple availability zones within a region. Zone redundancy reduces the risk that a datacenter or availability-zone outage makes your cache unavailable.

To make a cache zone redundant, you must deploy it in a supported region and set it to use the high availability configuration. In regions without availability zones, the high availability configuration still creates at least two nodes, but they aren't placed in separate zones.

The following diagram shows a zone-redundant cache with two nodes, each in a separate zone.

:::image type="complex" border="false" source="./media/reliability-managed-redis/zone-redundant.svg" alt-text="Diagram that shows a cache with two nodes distributed across separate availability zones for zone redundancy." lightbox="./media/reliability-managed-redis/zone-redundant.svg":::
   Diagram that shows an Azure Managed Redis instance with high availability across two availability zones. An Azure Managed Redis instance section spans all three zones. Zones 1 and 2 each have one node, and zone 3 has no nodes.
:::image-end:::

### Requirements

- **Region support:** You can deploy zone-redundant Azure Managed Redis caches into any region that supports availability zones and where the service is available. For the list of regions that support availability zones, see [Azure regions with availability zones](regions-list.md). For the list of regions that support Azure Managed Redis, see [Product availability by region](https://azure.microsoft.com/explore/global-infrastructure/products-by-region/table).

- **High availability configuration:** You must use high availability configuration on your cache for it to be zone redundant.

- **Tiers:** All Azure Managed Redis tiers support availability zones.

### Cost

Zone redundancy requires you to set up your cache for high availability, which deploys a minimum of two nodes for your cache. High availability configuration costs more than a non-high availability configuration. For more information, see [Azure Managed Redis pricing](https://azure.microsoft.com/pricing/details/managed-redis/).

### Configure availability zone support

- **Create a new zone-redundant instance.** When you create a new Azure Managed Redis instance, use high availability configuration and deploy it into a region that supports availability zones. The instance includes zone redundancy by default, with no extra configuration required.

  For more information, see [Create an Azure Managed Redis instance](/azure/redis/quickstart-create-managed-redis).

- **Make an existing instance zone redundant.** To make an existing Azure Managed Redis instance zone redundant, deploy it in a region that supports availability zones and use high availability on the cache.

- **Turn off zone redundancy.** You can't turn off zone redundancy on existing instances because you can't reverse high availability after you apply it to a cache.

### Capacity planning and management

During a zone-down event, your instance might have fewer resources available to serve your workload. If your instance often experiences resource pressure and you need to prepare for availability zone failure, consider one of the following approaches:

- **Overprovision your instance.** Select a higher performance tier than you require so that your instance can tolerate some capacity loss and continue to function without degraded performance. For more information, see [Manage capacity by overprovisioning](/azure/reliability/concept-redundancy-replication-backup#manage-capacity-with-over-provisioning) and [Scale an Azure Managed Redis instance](/azure/redis/how-to-scale).

- **Use active geo-replication.** You can deploy multiple instances in different regions and set up [active geo-replication](#active-geo-replication) to spread your load across those separate instances.

### Behavior when all zones are healthy

This section describes what to expect when a managed Redis cache is zone redundant and all availability zones operate normally:

- **Traffic routing between zones:** Azure Managed Redis distributes shards across nodes based on your cluster policy. Your cluster policy also determines how traffic routes to each node. Zone redundancy doesn't change how the service routes traffic.

- **Data replication between zones:** Azure Managed Redis automatically replicates shards across nodes by using asynchronous replication. You typically measure the replication lag between shards in seconds, but the exact duration depends on your cache's workload. Write-heavy and network-heavy scenarios typically experience higher replication lag.

### Behavior during a zone failure

This section describes what to expect when a managed Redis cache is zone redundant and one or more availability zones become unavailable:

- **Detection and response:** Azure Managed Redis is responsible for detecting a failure in an availability zone. You don't need to do anything to initiate a zone failover.

[!INCLUDE [Availability zone down notification (Service Health)](./includes/reliability-availability-zone-down-notification-service-include.md)]

- **Active requests:** The service might drop in‑flight requests, and applications should retry them. Applications should [implement retry logic](#resilience-to-transient-faults) to handle these temporary interruptions.

- **Expected data loss:** Any data that hasn't been replicated to shards in another zone might be lost during a zone failure. You typically measure the amount of data loss in seconds, but it depends on the replication lag.

- **Expected downtime:** A small amount of downtime, typically 10 to 15 seconds, might occur while shards fail over to nodes in healthy zones. For more information about the unplanned failover process, see [Explanation of a failover](/azure/redis/failover#explanation-of-a-failover). When you design applications, follow practices for [transient fault handling](#resilience-to-transient-faults).

- **Traffic rerouting:** Azure Managed Redis automatically redirects traffic to nodes in healthy zones.

### Zone recovery

When the affected availability zone recovers, Azure Managed Redis automatically restores operations to that zone. The Azure platform fully manages this process and doesn't require any customer intervention.

### Test for zone failures

Azure Managed Redis fully manages traffic routing, failover, and failback for zone failures, so you don't need to validate availability zone failure processes or provide any further input.

## Resilience to region-wide failures

Azure Managed Redis provides native multi-region support through *active geo-replication*, which lets you link multiple Azure Managed Redis instances across different Azure regions into a single replication group. You can then set up your own failover approach between the instances.

### Active geo-replication

When you use [active geo-replication](/azure/redis/how-to-active-geo-replication), applications can read from and write to any cache instance in the group, with changes automatically synced across all regions. The service supports active-active replication patterns where each region can handle both read and write operations simultaneously. When conflicts occur because of concurrent writes in different regions, the service automatically resolves them by using predetermined conflict resolution algorithms without requiring manual intervention. This approach provides resiliency to region failures while maintaining low-latency access for globally distributed applications.

The following diagram shows two cache instances in different regions within the same active geo-replication group, along with client applications that connect to each cache instance.

:::image type="complex" border="false" source="./media/reliability-managed-redis/active-geo-replication.svg" alt-text="Diagram that shows two caches in different regions, within the same active geo-replication group. Applications connect to each instance." lightbox="./media/reliability-managed-redis/active-geo-replication.svg":::
   Diagram that shows two Azure Managed Redis instances in different regions within the same active geo-replication group. Both regions contain an Azure Managed Redis instance and connect to an application section via an arrow. Asynchronous replication connects the two cache instances bidirectionally.
:::image-end:::

You're responsible for setting up your client applications to redirect requests to a healthy instance if any regional instance fails. The following diagram shows how an application can redirect their requests to a healthy cache instance when the instance that they typically use fails.

:::image type="complex" border="false" source="./media/reliability-managed-redis/active-geo-replication-failover.svg" alt-text="Diagram that shows two caches in different regions. One cache is failing, and applications connect to the healthy instance." lightbox="./media/reliability-managed-redis/active-geo-replication-failover.svg":::
   Diagram that shows two Azure Managed Redis instances in different regions within the same active geo-replication group. A red X in region A indicates a failure. The application under region A and under region B both connect to the Azure Managed Redis instance in Region B.
:::image-end:::

#### Requirements

- **Region support:** You can set up Azure Managed Redis active geo-replication between any Azure regions where the service is available.

- **Instance configuration:** Active geo-replication requires Azure Managed Redis instances that use the same tier and size in every region. All cache instances in a replication group must use identical settings, including persistence options, modules, and clustering policies.

- **Other requirements:** Your cache instances must meet other requirements, including the modules that you use. These requirements affect how you can scale your cache instances. For more information, see [Active geo-replication prerequisites](/azure/redis/how-to-active-geo-replication#active-geo-replication-prerequisites).

#### Considerations

- **Failover responsibility:** When you use active geo-replication, **you're responsible for failover between cache instances**. Prepare your application to handle failover. Failover involves preparation and might require you to do multiple steps. For more information, see [Force-unlink during a region outage](/azure/redis/how-to-active-geo-replication#force-unlink-if-theres-a-region-outage).

- **Eventual consistency:** Design applications to handle eventual consistency scenarios because changes can take time to propagate across all regions, depending on network conditions and geographic distance. During region outages, you might experience more data inconsistencies until connectivity is restored and synchronization finishes.

#### Cost

When you enable active geo-replication, you pay for each Azure Managed Redis instance in every region within the replication group. You might also incur data transfer charges for cross-region replication traffic between regions. For more information, see [Azure Managed Redis pricing](https://azure.microsoft.com/pricing/details/managed-redis/) and [Bandwidth pricing details](https://azure.microsoft.com/pricing/details/bandwidth/).

#### Configure multi-region support

- **Create a new geo-replicated cache instance.** Set up active geo-replication when you provision the cache by specifying a replication group and linking multiple instances. For more information, see [Create or join an active geo-replication group](/azure/redis/how-to-active-geo-replication#create-or-join-an-active-geo-replication-group).

- **Enable an existing cache instance for geo-replication.** You can add an existing cache instance to an active geo-replication group.

  When you add an existing instance to an active geo-replication group, the platform needs to clear the data in the instance, and you experience a small amount of downtime. If possible, plan to enable active geo-replication when you create cache instances.

  For more information, see [Add an existing instance to an active geo-replication group](/azure/redis/how-to-active-geo-replication#add-an-existing-instance-to-an-active-geo-replication-group).

- **Turn off geo-replication on a cache instance.** Remove an instance from a geo-replication group by deleting the cache instance. The remaining instances automatically reconfigure themselves.

#### Capacity planning and management

During a region-down event, the other instances might experience higher load. If an instance often runs near its resource limits and you must prepare for the increased capacity requirements during a region failure, consider [overprovisioning the instance](/azure/reliability/concept-redundancy-replication-backup#manage-capacity-with-over-provisioning). For more information, see [Scale an Azure Managed Redis instance](/azure/redis/how-to-scale).

#### Behavior when all regions are healthy

This section describes what to expect when you set up instances to use active geo-replication and all regions operate normally.

- **Traffic routing between regions:** You're responsible for setting up your applications to connect to a specific cache instance. Applications can connect to any cache instance in the replication group and do both read and write operations. The application handles traffic routing, which lets you direct clients to the nearest region for minimal latency. Azure Managed Redis doesn't automatically route traffic between regions.

- **Data replication between regions:** The service uses asynchronous replication between regions to maintain eventual consistency. It immediately commits write operations in the local region and then propagates them to other regions in the background. Conflict-free replicated data types (CRDTs) ensure that the service automatically merges concurrent writes in different regions.

#### Behavior during a region failure

This section describes what to expect when you set up instances to use active geo-replication and an outage occurs in one region.

- **Detection and response:** You're responsible for detecting the failure of a cache instance and deciding when to fail over. You can monitor the health of a geo-replicated cluster, which can help you decide when to begin failover. For more information, see [Geo-replication metric](/azure/redis/how-to-active-geo-replication#geo-replication-metric).

  Failover requires you to do multiple steps. For more information, see [Force-unlink during a region outage](/azure/redis/how-to-active-geo-replication#force-unlink-if-theres-a-region-outage).

- **Notification:** [!INCLUDE [Region down notification partial bullet (Service Health only)](./includes/reliability-region-down-notification-service-partial-include.md)]

  You can also monitor the health of each instance.

  To monitor the health of the geo-replication relationship, use the *geo-replication healthy* metric. The metric always has a value of `1` (healthy) or `0` (unhealthy). Set up Azure Monitor alerts on this metric to identify when the instances might be out of sync.

- **Active requests:** The service terminates requests to the failed region and your application's failover logic must handle them. Applications should implement retry policies that can redirect traffic to healthy caches.

- **Expected data loss:** Asynchronous replication between regions can result in the loss of recent writes to the failed region if those writes haven't replicated to other regions. The amount of potential data loss depends on the replication lag at the time of failure. For more information, see [Active-active geo-distributed Redis](https://redis.io/docs/latest/operate/rs/databases/active-active/#strong-eventual-consistency) and [Considerations about consistency and data loss in a conflict‑free replicated database (CRDB) regional failure](https://redis.io/faq/doc/21rbquorvb/considerations-about-consistency-and-data-loss-in-a-crdb-regional-failure).

- **Expected downtime:** Applications experience downtime only for the duration needed to detect the failure and redirect traffic to healthy regions. This downtime typically ranges from seconds to a few minutes and depends on how you set up your application's health check and failover configuration.

- **Traffic rerouting:** You're responsible for implementing logic in your applications to detect region failures and route traffic to healthy regions. You can implement this logic through health checks, circuit breaker patterns, or external load balancing solutions.

#### Region recovery

When a failed region recovers, Azure Managed Redis automatically reintegrates instances in that region into the active geo-replication group without your intervention. The service automatically syncs data from healthy instances. During this process, the recovered instances gradually sync the changes that occurred during the outage. After synchronization finishes, the recovered instances become fully active and can handle both read and write operations.

You're responsible for reconfiguring your application to route traffic back to the recovered region instance.

#### Test for region failures

Regularly test your application's failover procedures. Your application must be able to fail over between instances and meet your business requirements for downtime during the transition. Also test your overall response processes, including any reconfiguration of firewalls and other infrastructure, and your recovery process.

## Resilience to service maintenance

Azure Managed Redis handles regular service upgrades and other maintenance tasks.

During maintenance, Azure Managed Redis automatically creates new nodes and fails over. Client applications might experience connection interruptions and other transient faults. Applications should [implement retry logic](#resilience-to-transient-faults) to handle these temporary interruptions.

For more information about Azure Managed Redis maintenance processes, see [Failover and patching for Azure Managed Redis](/azure/redis/failover).

## Backup and restore

Azure Managed Redis provides both data persistence and backup capabilities to protect against data loss scenarios that other reliability features might not address. Backups protect against scenarios like data corruption, accidental deletion, or configuration errors.

- **Data persistence:** By default, Azure Managed Redis stores all cache data in memory. The service can optionally write snapshots of your data to disk by using [data persistence](/azure/redis/how-to-persistence). If a hardware failure affects the node, Azure Managed Redis automatically restores the data. You can choose from different types of data persistence, which provide different trade-offs between snapshot frequency and the performance effects on your cache.

  You can't restore data files to another instance, and you can't access the files. Data persistence doesn't protect you from data corruption or accidental deletion.

- **Import and export:** Azure Managed Redis supports backup of your data when you use the [import and export functionality](/azure/redis/how-to-import-export-data), which saves backup files to Azure Blob Storage. You can set up geo-redundant storage (GRS) on your Azure Storage account, or you can copy or move the backup blobs to other locations for further protection.

  You can restore exported files to the same cache instance or a different cache instance.

  The service doesn't include a built-in import or export scheduler, but you can develop your own automation processes that use the Azure CLI or Azure PowerShell to start export operations.

## Service-level agreement

[!INCLUDE [SLA description](includes/reliability-service-level-agreement-include.md)]

The SLA for Azure Managed Redis covers connectivity to the cache endpoints. It doesn't cover protection from data loss.

To be eligible for availability SLAs for Azure Managed Redis, you must meet the following requirements:

- You must enable high availability configuration.

- You must not initiate any product features or management actions that are documented to produce temporary unavailability.

Higher availability SLAs apply when your instance is zone redundant. In some tiers, you can become eligible for a higher availability SLA when you deploy zone-redundant instances into at least three regions by using active geo-replication.

## Related content

- [Reliability in Azure](/azure/reliability)
