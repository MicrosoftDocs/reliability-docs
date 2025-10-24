---
title: Reliability in Azure Event Hubs
description: Learn how to improve reliability in Azure Event Hubs by using availability zones, geo-disaster recovery, and geo-replication for mission-critical streaming applications.
author: anaharris-ms 
ms.author: anaharris
ms.topic: reliability-article
ms.custom: subject-reliability
ms.service: azure-event-hubs
ms.date: 10/17/2025
#Customer intent: As an engineer responsible for business continuity, I want to understand who needs to understand the details of how Azure Event Hubs works from a reliability perspective and plan disaster recovery strategies in alignment with the exact processes that Azure services follow during different kinds of situations.

---

# Reliability in Azure Event Hubs

This article describes reliability support in [Azure Event Hubs](../event-hubs/event-hubs-about.md), covering intra-regional resiliency via [availability zones](#availability-zone-support) and [multi-region deployments](#multi-region-support).

[!INCLUDE [Shared responsibility description](includes/reliability-shared-responsibility-include.md)]

Event Hubs is a native data-streaming service in the cloud that can stream millions of events per second with low latency, from any source to any destination. You can use Event Hubs to ingest and store streaming data, and you can integrate with client applications built for Apache Kafka or applications that use the Event Hubs client SDKs.

## Production deployment recommendations

To learn how to deploy Event Hubs to support your solution's reliability requirements and understand how reliability affects other aspects of your architecture, see [Architecture best practices for Event Hubs in the Azure Well-Architected Framework](/azure/well-architected/service-guides/event-hubs).

## Reliability architecture overview

This section describes important aspects of how Event Hubs works from a reliability perspective. The section introduces the logical architecture, which includes the resources and features that you deploy and use. It also explains the physical architecture, which provides details about how the service manages operations internally.

### Logical architecture

An Event Hubs [**namespace**](../event-hubs/event-hubs-features.md#namespace) serves as the management container for one or more event hubs. Management tasks such as allocating streaming capacity, configuring network security, and enabling geo-resilency and geo-disaster recovery are handled at the namespace level.

Within a namespace, you can organize events into an **event hub**. The Apache® Kafka ecosystem refers to this type of grouping as a **topic**. The event hub or topic is an append-only distributed log of your events.

Each event hub contains one or more [**partitions**](/azure/event-hubs/event-hubs-scalability#partitions), which are logs of sequential events. An event hub can use multiple partitions to perform parallel processing and horizontal scaling. Event Hubs only guarantees ordering within a single partition. Partitioning is an important aspect of your application's reliability design. When you design your application, you need to make a trade-off between maximizing availability and consistency. To maximize uptime for most applications, avoid addressing partitions directly from your client applications. For more information, see [Availability and consistency in Event Hubs](../event-hubs/event-hubs-availability-and-consistency.md).

Consumers that read from the event hub can read their events sequentially by maintaining their own **checkpoint**, which is the identifier of the last event that they receive.

For more information about partitions and other fundamental concepts in Event Hubs, see [Features and terminology in Event Hubs](../event-hubs/event-hubs-features.md).

### Physical architecture

In the physical architecture, an Event Hubs namespace runs within a **cluster**. A cluster provides the underlying compute and storage resources. Most namespaces run on clusters that are shared with other Azure customers. When you use the Premium tier, the namespace is allocated dedicated resources within a shared cluster. When you use the Dedicated tier, a cluster is dedicated to your namespaces. For more information about dedicated clusters, see [Event Hubs Dedicated tier overview](../event-hubs/event-hubs-dedicated-overview.md). Regardless of tier and cluster type, Microsoft manages the clusters and their underlying virtual machines and storage.

To achieve redundancy, each cluster has multiple replicas that process read and write requests. For high availability and performance optimization, all data is stored on three storage replicas. You can scale your namespace's compute resources by deploying throughput units, processing units, or capacity units, depending on the tier. For more information, see [Scaling with Event Hubs](../event-hubs/event-hubs-scalability.md).

Clusters span multiple physical machines and racks, which reduces the risk of catastrophic failures affecting your namespace. In regions that have availability zones, clusters are spread across separate physical datacenters. For more information, see [Availability zone support](#availability-zone-support).

## Transient faults

[!INCLUDE [Transient fault description](includes/reliability-transient-fault-description-include.md)]

Event Hubs implements transparent failure detection and failover mechanisms so that the service continues to operate within the assured service levels, typically without noticeable interruptions when failures occur.

When you design client applications to work with Event Hubs, follow this guidance:

- **Use the built-in retry policies.** The Event Hubs and Apache Kafka SDKs automatically retry operations for retryable errors like network timeouts, throttling responses, or when the server is busy. To avoid unnecessarily overloading the service, they implement exponential backoff by default.

- **Configure appropriate timeout values** based on your application requirements. The default timeout is typically 60 seconds, but you can adjust it based on your scenario.
- **Implement checkpointing** in your event processor to track progress and enable recovery from the last processed position after transient failures.
- **Use batching for send operations** to improve throughput and reduce the impact of transient network problems on individual messages.
- **Use Apache Kafka SDKs** if you work with the Kafka protocol. The Kafka SDKs also implement retry policies and other best practices that help handle transient faults.

## Availability zone support

[!INCLUDE [Availability zone support description](includes/reliability-availability-zone-description-include.md)]

Event Hubs supports zone-redundant deployments in all service tiers. In the Dedicated tier, availability zones require a minimum of three capacity units (CUs). When you create an Event Hubs namespace in a supported region, zone redundancy is automatically enabled at no extra cost. The zone-redundant deployment model applies to all Event Hubs features, including Capture, Schema Registry, and Kafka protocol support.

Event Hubs transparently replicates your configuration, metadata, and event data across three availability zones in the region. This replication provides automatic failover without any customer intervention required. All Event Hubs components including compute, networking, and storage are replicated across zones. Event Hubs has enough capacity reserves to instantly handle the complete loss of a zone. Even if an entire availability zone becomes unavailable, Event Hubs continues to operate without data loss or interruption to streaming applications.

:::image type="content" source="./media/reliability-event-hubs/availability-zones.svg" alt-text="Diagram that shows a zone-redundant Event Hubs namespace." lightbox="./media/reliability-event-hubs/availability-zones.svg" border="false":::

### Region support

Zone-redundant Event Hubs namespaces can be deployed into any [Azure region that supports availability zones](./regions-list.md).

### Requirements

- Standard and Premium tiers support availability zones with no extra configuration required.

- For the Dedicated tier, availability zones require a minimum of three CUs.

### Cost

Zone redundancy in Event Hubs doesn't add extra cost.

### Configure availability zone support

Event Hubs namespaces automatically support zone redundancy when deployed in [supported regions](#region-support). No further configuration is required.

### Normal operations

When Event Hubs namespaces use zone redundancy and all availability zones operate normally, expect the following behavior:

- **Traffic routing between zones:** Event Hubs operates in an active-active model where infrastructure in three availability zones simultaneously processes incoming events.

- **Data replication between zones:** Event Hubs uses synchronous replication across availability zones. When an event is sent to Event Hubs, it's written to replicas in multiple zones before the write operation is acknowledged to the client. This approach ensures zero data loss even if an entire zone becomes unavailable. The synchronous replication approach provides strong consistency guarantees while maintaining low latency through optimized replication protocols.

### Zone-down experience

When Event Hubs namespaces use zone redundancy and an availability zone outage occurs, expect the following behavior:

- **Detection and response:** Event Hubs automatically detects failures in an availability zone. You don't need to do anything to initiate a zone failover.

- **Notification:** Event Hubs doesn't notify you when a zone is down. But you can use [Azure Service Health](/azure/service-health/overview) to understand the overall health of the Event Hubs service, including any zone failures.

  Set up alerts to receive notifications of zone-level problems. For more information, see [Create Service Health alerts in the Azure portal](/azure/service-health/alerts-activity-log-service-notifications-portal).

- **Active requests:** During a zone failure, active requests might be dropped. If your clients handle [transient faults](#transient-faults) appropriately by retrying after a short period of time, they're unlikely to experience significant impact.

- **Expected data loss:** No data loss occurs during a zone failure because events are synchronously replicated across zones before acknowledgment.

- **Expected downtime:** A zone failure might cause a few seconds of downtime. If your clients handle [transient faults](#transient-faults) appropriately by retrying after a short period of time, they're unlikely to experience significant impact.

- **Traffic rerouting:** Event Hubs detects the loss of the zone and automatically redirects new requests to another replica in one of the healthy availability zones.
    
    Event Hubs client SDKs typically handle connection management and retry logic transparently.

### Zone recovery

When an availability zone recovers, Event Hubs automatically reintegrates the zone into the active service topology. The recovered zone begins accepting new connections and processing events alongside the other zones. Data that's replicated to surviving zones during the outage remains intact, and normal synchronous replication resumes across all zones. You don't need to take any action for zone recovery and reintegration.

### Testing for zone failures

Event Hubs manages traffic routing, failover, and zone recovery for zone failures, so you don't need to validate availability zone failure processes or provide any further input.

## Multi-region support

Event Hubs provides two types of multi-region support:

- **[Geo-replication (Premium and Dedicated tiers)](#geo-replication)** provides active-active replication of both metadata and event data between a primary and one or more secondary regions. Use geo-replication for most applications that need to remain resilient to region outages and can't tolerate event data loss.

- **[Metadata geo-disaster recovery (Standard tier and higher)](#metadata-geo-disaster-recovery)** provides active-passive replication of configuration and metadata between a primary and a secondary region, but it doesn't replicate event data. Use geo-disaster recovery for applications that can tolerate some loss of event data during a disaster scenario, and that need to quickly resume operations in another region.

Both geo-replication and metadata geo-disaster recovery require you to manually initiate failover or promotion of a secondary region to become the new primary region. Microsoft doesn't automatically perform failover or promotion for you, even if your primary region is down.

### Geo-replication

The Premium and Dedicated tiers include geo-replication. This feature replicates both metadata—such as entities, configuration, and properties—and data, such as event payloads, for the namespace. Use geo-replication to configure the replication approach for your namespace's configuration and event data. This feature ensures that your events remain available in another region and allows you to switch to the secondary region when needed. It also replicates schema registry metadata and data.

Use geo-replication for scenarios that require resiliency to region outages and have a low tolerance for event data loss.

The namespace essentially extends across regions. One region serves the primary, and the other regions serve as secondaries. Your Azure subscription shows a single namespace, no matter how many secondary regions you configure for geo-replication.

:::image type="content" source="./media/reliability-event-hubs/geo-replication.svg" alt-text="Diagram that shows an Event Hubs namespace configured for geo-replication." lightbox="./media/reliability-event-hubs/geo-replication.svg" border="false":::
    
At any time, you can *promote* the secondary region to a primary region. When you promote a secondary region, Event Hubs repoints the namespace's fully qualified domain name (FQDN) to the selected secondary region, and demotes the previous primary region to a secondary region. You decide whether to perform a *planned promotion*, which means that you wait for data replication to complete, or to perform a *forced promotion*, which might result in data loss.

> [!NOTE]
> Event Hubs geo-replication uses the term *promotion* because it best represents the process of promoting a secondary region to a primary region (and later demoting a secondary region to a primary region). You might also see the term *failover* used to describe the general process.

This section summarizes important aspects of geo-replication. Review the full documentation to understand exactly how it works. For more information, see [Event Hubs geo-replication](../event-hubs/geo-replication.md).

#### Region support

You can choose any Azure region that supports Event Hubs as your primary or secondary regions. You don't need to use Azure paired regions, so you can choose secondary regions based on your latency, compliance, or data residency requirements.

#### Requirements

To enable geo-replication, your namespace must use the Premium or Dedicated tier.

#### Considerations

When you enable geo-replication, consider the following factors:

- **Checkpoint format:**  The format of checkpoints changes. For more information, see [Consume data](../event-hubs/geo-replication.md#consuming-data).

- **Private endpoints:** If you use private endpoints to connect to your namespace, you also need to configure networking in your primary and secondary regions. For more information, see [Private endpoints](../event-hubs/geo-replication.md#private-endpoints).

#### Cost

To understand how pricing works for geo-replication, see [Pricing](../event-hubs/geo-replication.md#pricing).

#### Configure multi-region support

- **Enable geo-replication on a new or existing namespace.** To set up active-active replication for a newly created namespace, see [Enable geo-replication on a new namespace](../event-hubs/use-geo-replication.md#enable-geo-replication-on-a-new-namespace). To set up active-active replication on an existing namespace, see [Enable geo-replication on an existing namespace](../event-hubs/use-geo-replication.md#enable-geo-replication-on-an-existing-namespace).

- **Change the replication approach.** To change between synchronous and asynchronous replication modes, see [Switch replication mode](../event-hubs/use-geo-replication.md#switch-replication-mode).

- **Disable geo-replication.** To disable geo-replication to a secondary region, see [Remove a secondary region](../event-hubs/use-geo-replication.md#remove-a-secondary).

#### Normal operations

When an Event Hubs namespace uses geo-replication and the primary region operates normally, expect the following behavior:

- **Traffic routing between regions:** Client applications connect through the FQDN for your namespace, and their traffic routes to the primary region.

    Only the primary region actively processes events from clients during normal operations. The secondary region receives replicated events but otherwise remains passive in standby mode.

- **Data replication between regions:** Data replication behavior between the primary and secondary regions depends on whether you configure your replication pairing to use synchronous or asynchronous replication.

    - *Synchronous:* Event Hubs replicates events to the secondary region and then completes the write operation.
    
        This mode provides the highest data safety because both regions must commit the event. But synchronous replication substantially increases the write latency for incoming events. It also requires that the secondary region is available to accept the write operation, so an outage in any secondary region causes the write operation to fail.

    - *Asynchronous:* Event Hubs writes events to the primary region, completes the write operation, and then replicates the events to the secondary region.
    
        This mode provides a higher write throughput than synchronous replication because no inter-region replication latency occurs during write operations. The asynchronous replication mode also tolerates the loss of a secondary region while allowing write operations in the primary region. But if an outage occurs in the primary region, unreplicated data might be unavailable or lost.

        When you use asynchronous replication, you configure the maximum acceptable lag time for replication. You can verify the current replication lag by using [Azure Monitor metrics](../event-hubs/geo-replication.md#monitoring-data-replication).
        
        If the asynchronous replication lag increases beyond the maximum that you specify, the primary region throttles incoming requests so that the replication can catch up. To avoid this situation, select secondary regions that aren't too geographically distant, and ensure that your capacity supports the throughput.

    For more information, see [Replication modes](../event-hubs/geo-replication.md#replication-modes).

#### Region-down experience

When an Event Hubs namespace uses geo-replication and an outage occurs in the primary or secondary region, expect the following behavior:

- **Detection and response:** You decide when to promote your namespace's secondary region to the new primary region. Microsoft doesn't make this decision or initiate the process for you, even if a region outage occurs.

    For more information about how to promote a secondary region to the new primary, see [Promote secondary](../event-hubs/use-geo-replication.md#promote-secondary).
    
    When you promote a secondary region, choose whether to perform a *planned promotion* or a *forced promotion*. A planned promotion waits for the secondary region to catch up before accepting new traffic. This approach eliminates data loss but introduces downtime.
    
    During an outage in the primary region, you typically need to perform a forced promotion. If the primary region is available and you trigger a promotion for another reason, you might choose a planned promotion.

- **Notification:** Event Hubs doesn't notify you when a region is down. But you can use [Azure Service Health](/azure/service-health/overview) to understand the overall health of the Event Hubs service, including region failures. Use that information and other metrics to decide when to promote a secondary region to primary.

    Set up alerts to receive notifications about region-level problems. For more information, see [Create Service Health alerts in the Azure portal](/azure/service-health/alerts-activity-log-service-notifications-portal).

- **Active requests:** This behavior depends on whether the region outage occurs in the primary region or a secondary region:

    - *Primary region outage:* If the primary region is unavailable, all active requests terminate. Client applications should retry operations after the promotion completes.

    - *Secondary region outage:* An outage in the secondary region might cause problems for active requests in the following situations:

        - If you use the synchronous replication mode, the primary region can't complete write operations if any secondary region is unavailable.

        - If you use the asynchronous replication mode, your namespace throttles and doesn't accept new events after the replication lag reaches the maximum replication lag that you configure.

        To continue using the namespace in the primary region, remove the secondary namespace from your geo-replication configuration.

- **Expected data loss:** The amount of data loss depends on the type of promotion that you perform (planned or forced) and the replication mode (synchronous or asynchronous):

    - *Planned promotion:* No data loss is expected.
    
        This option requires all primary and secondary regions to be available, so it might not be possible during a region outage.

    - *Forced promotion, synchronous replication:* No data loss is expected.

    - *Forced promotion, asynchronous replication:* You might experience some data loss for recent events that aren't replicated to the secondary region. The amount depends on the replication lag. You can verify the current replication lag by using [Azure Monitor metrics](../event-hubs/geo-replication.md#monitoring-data-replication).

- **Expected downtime:** The amount of expected downtime depends on the type of promotion that you perform (planned or forced):

    - *Planned promotion:* The first step in a planned promotion replicates data to the secondary region. Usually that completes quickly, but in some situations it might take up to the length of the replication lag. After the replication completes, the promotion process typically takes about 5 to 10 minutes. It can sometimes take longer for clients to fully replicate and update Domain Name System (DNS) entries.
    
        The primary region doesn't accept write operations during the entire promotion process.

        This option might not be possible during a region outage because it requires all primary and secondary regions to be available.

    - *Forced promotion:* During a forced promotion, Event Hubs doesn't wait for data replication to complete, and it initiates the promotion immediately. The promotion process typically takes about 5 to 10 minutes. It can sometimes take longer for clients to fully replicate and update DNS entries.

        The primary region doesn't accept write operations during the entire promotion process.

- **Traffic rerouting:** After the promotion completes, the namespace's FQDN points to the new primary region. But this redirection depends on how quickly clients' DNS records are updated, including for their DNS servers to honor the time-to-live (TTL) of the namespace DNS records.

    In some situations, you must configure consumer applications to behave consistently after region promotion occurs. For more information, see [Consume data](../event-hubs/geo-replication.md#consuming-data).

#### Region recovery

After the original primary region recovers, you might promote the namespace's primary region again. To return the namespace to its original primary region, follow the same region promotion process.

If you perform a forced promotion, any lost data is unrecoverable, even after the primary region becomes available.

#### Testing for region failures

To test geo-replication, temporarily promote the secondary region to primary and validate that your client applications can switch between regions with minimal disruption.

Monitor the promotion duration and validate that your runbooks and automation work correctly. After testing, you can fail back to the original configuration.

But understand the potential downtime and data loss that you might experience during and after the promotion process. Test geo-replication in a nonproduction environment that mirrors the configuration of your production namespace.

### Metadata geo-disaster recovery

The Standard tier and higher include metadata geo-disaster recovery. This feature improves recovery from disaster scenarios, including the catastrophic loss of a region. Geo-disaster recovery only replicates the configuration and metadata of your namespace, but it doesn't replicate event data. To support disaster recovery, it ensures that a namespace in another region is preconfigured and ready to immediately accept events from clients. Geo-disaster recovery serves as a one-way recovery solution and doesn't support failback to the prior primary region.

Metadata geo-disaster recovery works best for applications that don't strictly need to maintain every event and can tolerate some loss of data during a disaster scenario. For example, if your events represent sensor readings that you later aggregate, you can likely lose some events from a lost region if you can quickly resume processing new events in another region.

> [!IMPORTANT]
> Geo-disaster recovery enables continuity of operations that have the same configuration but **doesn't replicate event data**. If you need to replicate event data, consider using [geo-replication](#geo-replication).

When you configure metadata geo-disaster recovery, you create an *alias* that client applications connect to. The alias is an FQDN that directs all traffic to the primary namespace by default.

:::image type="content" source="./media/reliability-event-hubs/geo-disaster-recovery.svg" alt-text="Diagram that shows two Event Hubs namespaces that are configured for metadata geo-disaster recovery." lightbox="./media/reliability-event-hubs/geo-disaster-recovery.svg" border="false":::

If the primary region fails or another type of disaster occurs, you can manually initiate a single-time, one-way failover move from the primary to the secondary region at any time. The failover completes almost instantly. During the failover process, the geo-disaster recovery alias repoints to the secondary namespace and the pairing is removed.

This section summarizes important aspects of geo-disaster recovery. Review the full documentation to understand exactly how it works. For more information, see [Event Hubs geo-disaster recovery](../event-hubs/event-hubs-geo-dr.md).

#### Region support

You can select any Azure region that supports Event Hubs as your primary or secondary namespace. You don't need to use Azure paired regions, so you can choose secondary regions based on your latency, compliance, or data residency requirements.

#### Requirements

- **Primary namespace tier:** Your primary namespace must be in the Standard tier or higher to use metadata geo-disaster recovery.

- **Secondary namespace tier:** Metadata geo-disaster recovery supports specific combinations of tiers for the primary and secondary namespaces. For more information, see [Supported namespace pairs](../event-hubs/event-hubs-geo-dr.md#supported-namespace-pairs).

#### Considerations

- **Role assignments:** Microsoft Entra role-based access control (RBAC) assignments to entities in the primary namespace aren't replicated to the secondary namespace. Create role assignments manually in the secondary namespace to secure access to them.

- **Schema registry:** Schema registry metadata replicates when you use metadata geo-disaster recovery, but schemas registered with the schema registry don't replicate.

- **Application design:** Geo-disaster recovery requires specific considerations when you design your client applications. For more information, see [Considerations](../event-hubs/event-hubs-geo-dr.md#considerations).

- **Private endpoints:** If you use private endpoints to connect to your namespace, configure networking in both your primary and secondary region. For more information, see [Private endpoints](../event-hubs/event-hubs-geo-dr.md#private-endpoints).

#### Cost

When you enable metadata geo-disaster recovery, you pay for both the primary and secondary namespaces.

#### Configure multi-region support

- **Create metadata geo-disaster recovery pairing.** To configure disaster recovery between primary and secondary namespaces, see [Setup and failover flow](../event-hubs/event-hubs-geo-dr.md#setup).

- **Disable metadata geo-disaster recovery.** To break the pairing between namespaces, see [Setup and failover flow](../event-hubs/event-hubs-geo-dr.md#setup).

#### Capacity planning and management

When you plan for multi-region deployments, ensure that both regions have sufficient capacity to handle the full load if one region fails. The secondary region remains passive during normal operations, but it must immediately handle traffic after failover. Plan how you to scale the secondary namespace capacity so that it can receive production traffic without delay. If you can tolerate extra downtime during the failover process, scale the secondary namespace capacity during or after failover. To reduce downtime, provision capacity in the secondary namespace in advance so that it remains ready to receive production load.

#### Normal operations

When an Event Hubs namespace uses geo-disaster recovery and the primary region operates normally, expect the following behavior:

- **Traffic routing between regions:** Client applications connect through the geo-disaster recovery alias for your namespace, and their traffic routes to the primary namespace in the primary region.

    Only the primary namespace actively processes events from clients during normal operations. The secondary remains passive in standby mode, and any requests to access data in the secondary namespace fail.

- **Data replication between regions:** Only configuration metadata replicates between the namespaces. Replication of configuration occurs continuously and asynchronously.

    All event data remains in the primary namespace only and doesn't replicate to the secondary namespace.

#### Region-down experience

When an Event Hubs namespace uses geo-disaster recovery and an outage occurs in the primary region, expect the following behavior:

- **Detection and response:** Monitor region health and initiate failover manually. Microsoft doesn't perform failover or promote a secondary region automatically, even if your primary region is down.

    For more information about how to initiate failover, see [Manual failover](../event-hubs/event-hubs-geo-dr.md#manual-failover).

    Failover is a one-way operation, so you need to reestablish the geo-disaster recovery pairing later. For more information, see [Region recovery](#region-recovery-1).

- **Notification:** Event Hubs doesn't notify you when a region is down. But you can use [Service Health](/azure/service-health/overview) to understand the overall health of the Event Hubs service, including region failures. You can use that information and other metrics to decide when to initiate a failover.

    Set up alerts to receive notifications about region-level problems. For more information, see [Create Service Health alerts in the Azure portal](/azure/service-health/alerts-activity-log-service-notifications-portal).

- **Active requests:** Active requests in progress terminate when the failover starts. Client applications should retry operations after the failover completes.

- **Expected data loss:** 

    - *Metadata:* Configuration and metadata typically replicate to the secondary namespace. But metadata replication occurs asynchronously, so recent changes might not replicate, especially complex changes. Verify the configuration of your secondary namespace before clients access it.

    - *Event data:* Event data doesn't replicate between regions. If the primary region goes down, events in your primary namespace become unavailable.
    
        The events aren't permanently lost unless a catastrophic disaster causes total loss of the primary region. If the region recovers, you can retrieve events from the primary namespace later.

- **Expected downtime:** Failover typically occurs within 5 to 10 minutes. It can take longer for clients to fully replicate and update DNS entries.

- **Traffic rerouting:** Clients that use the geo-disaster recovery alias to connect to the namespace automatically redirect to the secondary namespace after failover. But this redirection depends on client applications updating their DNS records and honoring the TTL of the namespace DNS records.

#### Region recovery

After the original primary region recovers, you must manually re-establish the pairing and optionally fail back. Create a new geo-disaster recovery pairing with the recovered region as secondary, then perform another failover if you want to return to the original region. This process involves potential data loss of events sent to the temporary primary.

If the disaster causes the loss of all zones in the primary region, your data might be unrecoverable. In other scenarios, your event data remains in the primary namespace from before the failover is recoverable. You can obtain historic events from the old primary namespace after access is restored. You configure your applications to receive and process those events. Microsoft doesn't automatically restore them into your secondary region.

#### Testing for region failures

To test your response and disaster recovery processes, perform a planned failover during a maintenance window. Initiate failover from your primary namespace to your secondary namespace and verify that your applications can connect and process events from the new primary.

Monitor the failover duration and validate that your runbooks and automation work correctly. After testing, you can fail back to the original configuration.

Understand the potential downtime and data loss that you might experience during and after the failover process. Test geo-replication in a nonproduction environment that mirrors the configuration of your production namespace.

### Alternative multi-region approaches

Geo-replication and metadata geo-disaster recovery provide resiliency to region outages and other problems, and they support most workloads. But some Event Hubs tiers don't support these features, or you might require custom replication or need to maintain multiple active regions simultaneously.

Various design patterns can achieve different types of multi-region support in Event Hubs. Many of the patterns require deploying multiple namespaces and using services like Azure Functions to replicate events between them. For more information, see [Multi-site and multi-region federation](../event-hubs/event-hubs-federation-overview.md).

## Backups

Event Hubs doesn't serve as a long-term storage location for data. Typically, you store data in an event hub for a short period of time and then process or persist it in another data storage system. You can configure the data retention period for your event hub based on your requirements and the tier that your namespace uses. For more information, see [Event retention](../event-hubs/event-hubs-features.md#event-retention).

If you need to retain a copy of your events, consider using [Event Hubs Capture](../event-hubs/event-hubs-capture-overview.md), which saves copies of events to an Azure Blob Storage account.

## Service-level agreement

[!INCLUDE [SLA description](includes/reliability-service-level-agreement-include.md)]

Your namespace's availability service-level agreement (SLA) is higher when it uses the Premium or Dedicated tiers.

### Related content

- [Event Hubs documentation](../event-hubs/event-hubs-about.md)
- [Azure reliability overview](/azure/reliability/overview)
