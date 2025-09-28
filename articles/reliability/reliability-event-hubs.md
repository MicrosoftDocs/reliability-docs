---
title: Reliability in Azure Event Hubs
description: Learn how to improve reliability in Azure Event Hubs by using availability zones, geo-disaster recovery, and geo-replication for mission-critical streaming applications.
author: anaharris-ms 
ms.author: anaharris
ms.topic: reliability-article
ms.custom: subject-reliability
ms.service: azure-event-hubs
ms.date: 09/26/2025

#Customer intent: As an engineer responsible for business continuity, I want to understand the details of how Azure Event Hubs works from a reliability perspective and plan disaster recovery strategies in alignment with the exact processes that Azure services follow during different kinds of situations.
---

# Reliability in Azure Event Hubs

Azure Event Hubs is a native data-streaming service in the cloud that can stream millions of events per second, with low latency, from any source to any destination. Businesses can use Event Hubs to ingest and store streaming data.

This article describes reliability support in [Azure Event Hubs](../event-hubs/event-hubs-about.md). It covers intra-regional resiliency via [availability zones](#availability-zone-support) and [multi-region support options](#multi-region-support).

[!INCLUDE [Shared responsibility description](includes/reliability-shared-responsibility-include.md)]

## Production deployment recommendations

To learn about how to deploy Event Hubs to support your solution's reliability requirements, and how reliability affects other aspects of your architecture, see [Architecture best practices for Azure Event Hubs in the Azure Well-Architected Framework](/azure/well-architected/service-guides/event-hubs).

## Reliability architecture overview

The container for your Event Hubs resources is a *namespace*. A namespace contains one or more *event hubs*, which store your *events*. The sequencing of events is managed by the service based on timestamps, and consumers that read from the event hub can maintain a *checkpoint* to read their events sequentially.

Each event hub contains one or more *partitions*, which are sequences of sequential events. Partitions enable your event hub to perform parallel processing and allow for horizontal scaling. You should select a number of partitions that meets your requirements for throughput and consistency.

When you use Event Hubs, you should make an explicit decision about whether you maximize availability or the consistency of your sequence of events. Your decision affects how you design your client applications to write to and read from partitions. If you need to maximize availability, you should avoid addressing partitions directly from your client applications. For more information, see [Availability and consistency in Event Hubs](../event-hubs/event-hubs-availability-and-consistency.md).

Event Hubs also provides a *schema registry* to maintain a repository of event schemas that you can use throughout your organization. For more information, see [Schema Registry in Azure Event Hubs](../event-hubs/schema-registry-concepts.md).

### Clusters

A namespace is associated with a *cluster*, which provides the underlying compute and storage resources for the service. Most namespaces run on clusters that are shared with other Azure customers. When your namespace uses the Premium tier, the namespace is allocated dedicated resources within a shared cluster. When you use the Dedicated tier, you have a cluster dedicated to your namespaces. To learn more about dedicated clusters, see [Azure Event Hubs Dedicated tier overview](../event-hubs/event-hubs-dedicated-overview.md). Whichever tier and cluster type you use, Microsoft manages the clusters and their underlying virtual machines and storage.

Clusters provide the physical resources that its namespaces use. Each cluster has multiple storage replicas for high availability and performance optimization. You scale your namespace's compute resources by deploying throughput units, processing units, or capacity units, depending on the tier. For more information, see [Scaling with Event Hubs](../event-hubs/event-hubs-scalability.md).

Clusters span multiple physical machines and racks, which reduces the risk of catastrophic failures affecting your namespace. In regions with availability zones, clusters are spread across separate physical datacenters. For more information, see [Availability zone support](#availability-zone-support).

The service implements transparent failure detection and failover mechanisms such that the service continues to operate within the assured service-levels and typically without noticeable interruptions when such failures occur.

## Transient faults

[!INCLUDE [Transient fault description](includes/reliability-transient-fault-description-include.md)]

Azure Event Hubs handles transient faults through built-in retry mechanisms in the client SDKs and automatic request routing during temporary service disruptions. When working with Event Hubs:

- **Use the built-in retry policies in Event Hubs SDKs**, which implement exponential backoff by default. The SDKs automatically retry operations for retriable errors like network timeouts, throttling responses, or when the server is busy.
- **Configure appropriate timeout values** based on your application requirements. The default timeout is typically 60 seconds, but you can adjust this based on your scenario.
- **Implement checkpointing** in your event processor to track progress and enable recovery from the last processed position after transient failures.
- **Use batching for send operations** to improve throughput and reduce the impact of transient network issues on individual messages. <!-- PG: please verify this is accurate. -->

## Availability zone support

[!INCLUDE [Availability zone support description](includes/reliability-availability-zone-description-include.md)]

Azure Event Hubs supports zone-redundant deployments in all service tiers. When you create an Event Hubs namespace in a supported region, zone redundancy is automatically enabled at no additional cost. The service transparently replicates your configuration, metadata, and data across all three availability zones in the region, providing automatic failover capability without any customer intervention required. 

Event Hubs implements zone-redundant storage for both the messaging infrastructure and the event data. All Event Hubs components including compute, networking, and storage are replicated across zones. The service has enough capacity reserves to instantly cope with the complete, catastrophic loss of a zone. This ensures that even if an entire availability zone becomes unavailable, Event Hubs continues to operate without data loss or interruption to your streaming applications. The zone-redundant deployment model applies to all Event Hubs features including Capture, Schema Registry, and Kafka protocol support.

### Region support

Zone-redundant Event Hubs namespaces can be deployed in [any Azure region that supports availability zones](./regions-list.md).

### Cost

There's no extra cost for zone redundancy with Event Hubs.

### Configure availability zone support

Event Hubs namespaces automatically support zone redundancy when deployed in [supported regions](#region-support). No further configuration is required.

### Normal operations

This section describes what to expect when Event Hubs namespaces are configured for zone redundancy and all availability zones are operational.

- **Traffic routing between zones**. Event Hubs operates in an active/active model where infrastructure in all of the region's zones simultaneously process incoming events. <!-- PG: Please verify, and advise whether South Central US going to 4 zones changes anything. -->

- **Data replication between zones**. Event Hubs uses synchronous replication across availability zones. When an event is sent to Event Hubs, it's written to multiple zones before the send operation is acknowledged to the client. This ensures zero data loss even if an entire zone becomes unavailable. The synchronous replication approach provides strong consistency guarantees while maintaining low latency through optimized replication protocols. <!-- PG: Please verify replication is synchronous. -->

### Zone-down experience

This section describes what to expect when Event Hubs namespaces are configured for zone redundancy and there's an availability zone outage.

- **Detection and response**: The Event Hubs service is responsible for detecting a failure in an availability zone. You don't need to do anything to initiate a zone failover.

- **Notification**: Event Hubs doesn't notify you when a zone is down. However, you can use [Azure Service Health](/azure/service-health/overview) to understand the overall health of the Event Hubs service, including any zone failures.

  Set up alerts to receive notifications of zone-level problems. For more information, see [Create Service Health alerts in the Azure portal](/azure/service-health/alerts-activity-log-service-notifications-portal).

- **Active requests**: During a zone failure, active requests might be dropped. If your clients should handle [transient faults](#transient-faults) appropriately by retrying after a short period of time, it's unlikely they'll be substantially affected. <!-- PG: Please verify this statement. -->

- **Expected data loss**: No data loss occurs with zone-redundant Event Hubs because events are synchronously replicated across zones before acknowledgment. <!-- PG: Please verify this statement. -->

- **Expected downtime**: A zone failure shouldn't cause downtime to your namespace. <!-- PG: Please verify this statement. -->

- **Traffic rerouting**: Event Hubs detects the loss of the zone. Then, any new requests are automatically redirected to another instance of the service in one of the healthy availability zones.
    
    Event Hubs client SDKs typically handle connection management and retry logic transparently.

### Zone recovery

When an availability zone recovers, Event Hubs automatically reintegrates the zone into the active service topology. The recovered zone begins accepting new connections and processing events alongside the other zones. Data that was replicated to surviving zones during the outage remains intact, and normal synchronous replication resumes across all zones. No customer action is required for zone recovery and reintegration.

### Testing for zone failures

Because Event Hubs fully manages traffic routing, failover, and failback for zone failures, you don't need to validate availability zone failure processes or provide any further input.

## Multi-region support

Azure Event Hubs provides two types of multi-region support, which each provide different types of resiliency to region outages. If you require your namespace to be resilient to region issues, you can select one of these approaches to use.

- *Metadata geo-disaster recovery (Standard tier and above):* Metadata geo-disaster recovery is a solution to assist in your recovery from disaster scenarios, including the catastrophic loss of a region. Geo-disaster recovery only replicates the configuration and metadata of your namespace, but it doesn't replicate event data. It assists disaster recovery by ensuring a namespace in another region is preconfigured and ready to immediately accept events from clients. Geo-disaster recovery is intended as a one-way recovery solution, and doesn't support failback to the prior primary region.

    Metadata geo-disaster recovery is most useful for applications that don't have a strict need to maintain every event and that can tolerate some loss of data during a disaster scenario. For example, if your events represent sensor readings that you later aggregate, it might be acceptable to lose some events from a lost region as long as you can quickly start to process new events in another region. <!-- PG: Please verify this positioning is accurate. -->

- *Geo-replication (Premium and Dedicated tiers):* Geo-replication replicates your namespace's configuration and the event data by using a replication approach that you configure. Geo-replication ensures that your events are available in another region, and you can switch to use the secondary region when you need to.

    Geo-replication is appropriate when you have a low tolerance for the loss of event data. <!-- PG: Please verify this positioning is accurate. -->

### Metadata geo-disaster recovery

Metadata geo-disaster recovery, available in the Standard tier and above, ensures that the configuration of a namespace is continuously replicated from a primary namespace to a secondary namespace in a different Azure region.

> [!IMPORTANT]
> Geo-disaster recovery enables instantaneous continuity of operations with the same configuration, but **doesn't replicate the event data**. If you need to replicate event data, or if you need to operate multiple regional namespaces in active/active configurations for resiliency purposes, consider using [geo-replication](#geo-replication) or an [alternative multi-region approach](#alternative-multi-region-approaches).

When you configure metadata geo-disaster recovery, you create an *alias* that client applications connect to. The alias is a FQDN (fully qualified domain name). By default, the alias directs all traffic to the primary namespace.

If the primary region fails or there's another type of disaster, you can manually initiate a single-time, one-way failover move from the primary to the secondary at any time. The failover is nearly instantaneous once initiated. During the failover process, the geo-disaster recovery alias is repointed to the secondary namespace. After the move, the pairing is then removed.

This section provides a summary of some of the important aspects of geo-disaster recovery, but it's important you review the full documentation to understand exactly how it works. For more information, see [Azure Event Hubs - Geo-disaster recovery](../event-hubs/event-hubs-geo-dr.md).

#### Region support

You can select any Azure region where Event Hubs is available for your primary or secondary namespace. Because you're not restricted to Azure paired regions, you have the flexibility to choose secondary regions based on your requirements for latency, compliance, or data residency.

#### Requirements

- **Primary namespace tier:** Your primary namespace must be in the Standard tier or above to use metadata geo-disaster recovery.

- **Secondary namespace tier:** Metadata geo-disaster recovery supports specific combinations of tiers for the primary and secondary namespaces. For detailed information, see [Supported namespace pairs](../event-hubs/event-hubs-geo-dr.md#supported-namespace-pairs).

#### Considerations

- **Role assignments:** Microsoft Entra role-based access control (RBAC) assignments to entities in the primary namespace aren't replicated to the secondary namespace. Create role assignments manually in the secondary namespace to secure access to them.

- **Application design:** Geo-disaster recovery requires specific considerations when you design your client applications. For more information, see [Considerations](../event-hubs/event-hubs-geo-dr.md#considerations).

#### Cost

You're charged for both the primary and secondary namespaces even though only the primary actively processes events.

#### Configure multi-region support

- **Create metadata geo-disaster recovery pairing.** To configure disaster recovery between primary and secondary namespaces see [Setup and failover flow](../event-hubs/event-hubs-geo-dr.md#setup).

- **Disable metadata geo-disaster recovery.** To break the pairing between namespaces, see [Setup and failover flow](../event-hubs/event-hubs-geo-dr.md#setup).

#### Capacity planning and management

When planning for multi-region deployments, ensure that both regions have sufficient capacity to handle the full load if one region fails. Although the secondary remains passive during normal operations, it must be ready to handle your traffic immediately after failover. Consider how you can scale secondary namespace capacity so it's ready to receive production traffic. If you can tolerate additional downtime during the failover process, you can scale the secondary namespace capacity during or after failover. If you need to reduce downtime, you can provision capacity in the secondary namespace ahead of time so it's always ready to receive production load.

#### Normal operations

This section describes what to expect when an IoT hub is configured for geo-disaster recovery, and the primary region is operational.

- **Traffic routing between regions**. Client applications connect through the geo-disaster recovery alias for your namespace, and their traffic is routed to the primary namespace in the primary region.

    Only the primary namespace actively processes events during normal operations. The secondary remains passive in standby mode, and any requests to access data in the secondary namespace fail.

- **Data replication between regions**. Only configuration metadata is replicated between the namespaces. Replication of configuration occurs continuously and is performed asynchronously.

    All event data remains in the primary region only.

#### Region-down experience

This section describes what to expect when an Event Hubs namespace is configured for geo-disaster recovery and there's an outage in the primary region.

- **Detection and response:** You must monitor region health and initiate failover manually. Microsoft does not perform failover automatically.

    To initiate failover, see [Manual failover](../event-hubs/event-hubs-geo-dr.md#manual-failover).

    > [!IMPORTANT]
    > Failing is a one-way operation. It activates the secondary namespace and removes the primary namespace from the geo-disaster recovery pairing. You must create a new secondary namespace before you create a new geo-disaster recovery pair.

- **Notification**: Event Hubs doesn't notify you when a region is down. However, you can use [Azure Service Health](/azure/service-health/overview) to understand the overall health of the Event Hubs service, including any region failures.

    Set up alerts to receive notifications of region-level problems. For more information, see [Create Service Health alerts in the Azure portal](/azure/service-health/alerts-activity-log-service-notifications-portal).

- **Active requests:** Any active requests in progress are terminated when the failover starts. Client applications should retry operations once the failover completes. If your clients should handle [transient faults](#transient-faults) appropriately by retrying after a short period of time, it's unlikely they'll be substantially affected. <!-- PG: Please confirm. --> 

- **Expected data loss:** 

    - *Metadata:* Generally all configuration and metadata is expected to be replicated to the secondary namespace. However, because metadata replication occurs asynchronously, any recent changes might not have been replicated, especially complex changes. Verify the configuration of your secondary namespace before clients start to access it.

    - *Event data:* Event data isn't replicated between regions, so all of the events in your primary namespace are unavailable if its region is down. However, the events aren't permanently lost except in a catastrophic disaster that causes the total loss of the primary region. Assuming the region recovers, you can retrieve events from the primary namespace later.

- **Expected downtime:** Failover occurs nearly instantly. In most failover scenarios, the alias is updated within 60 seconds. <!-- PG: Please verify -->

- **Traffic rerouting:** Clients that use the geo-disaster recovery alias to connect to the namespace are automatically redirected to the secondary namespace after failover. However, this redirection depends on client applications updating their DNS records and honoring the time-to-live (TTL) of the namespace DNS records.

#### Region recovery

After the original primary region recovers, you must manually re-establish the pairing and optionally fail back. For geo-disaster recovery, create a new pairing with the recovered region as secondary, then perform another failover if you want to return to the original region. This process involves potential data loss of events sent to the temporary primary.

Unless the disaster caused the loss of all zones, the event data that is preserved in the primary namespace from before the failover is recoverable, and the historic events can be obtained from the old primary namespace once access is restored.

#### Testing for region failures

To test your response and disaster recovery processes, you can perform a planned failover during a maintenance window. Initiate failover from primary to secondary namespace and verify that your applications can connect and process events from the new primary. Monitor the failover duration and validate that your runbooks and automation work correctly. After testing, you can fail back to the original configuration.

### Geo-replication

Geo-replication, which is available in the Premium and Dedicated tiers, provides replication of both metadata (entities, configuration and properties) and data (event payloads) for the namespace.

Geo-replication ensures that metadata and data of a namespace is continuously replicated from a primary region to the secondary region, and you can customize how the replication occurs. The namespace can be thought of as being extended to more than one region, with one region being the primary and the other being the secondary.
    
At any time, the secondary region can be promoted to become a primary region. Promoting a secondary repoints the namespace FQDN (fully qualified domain name) to the selected secondary region, and the previous primary region is demoted to a secondary region.

This section provides a summary of some of the important aspects of geo-replication, but it's important you review the full documentation to understand exactly how it works. For more information, see [Azure Event Hubs Geo-replication](../event-hubs/geo-replication.md).

#### Region support

You can select any Azure region where Event Hubs is available for your primary or secondary namespace. Because you're not restricted to Azure paired regions, you have the flexibility to choose secondary regions based on your requirements for latency, compliance, or data residency.

#### Requirements

<!-- TODO does Dedicated support it? -->

- Both namespaces must be Premium tier with the same processing unit configuration.
- Namespaces must be in different regions.

#### Cost

When you enable geo-replication, you pay for processing units in each region plus inter-region data transfer charges for replication traffic.

#### Configure multi-region support

- **Enable geo-replication on a new namespace.** To set up active-active replication between a newly created pair of namespaces, see [Enable Geo-replication on a new namespace](../event-hubs/use-geo-replication.md#enable-geo-replication-on-a-new-namespace).

- **Enable geo-replication on a new namespace.** To set up active-active replication between existing namespaces, see [Enable Geo-replication on a new namespace](../event-hubs/use-geo-replication.md#enable-geo-replication-on-an-existing-namespace).

- **Change replication approach.** TODO see [Switch replication mode](../event-hubs/use-geo-replication.md#switch-replication-mode).

- **Disable geo-replication.** To disable geo-replication to a secondary namespace, see [Remove a secondary](../event-hubs/use-geo-replication.md#remove-a-secondary).

#### Capacity planning and management

When planning for multi-region deployments, ensure that both regions have sufficient capacity to handle the full load if one region fails.

For geo-replication, both regions actively process events, so size each region to handle the expected load plus overhead for replication traffic. Monitor metrics like incoming messages, throughput unit utilization, and replication lag to ensure adequate capacity.

#### Normal operations

This section describes what to expect when an IoT hub is configured for geo-replication, and the primary region is operational. <!-- TODO is there a concept of a primary region? -->

- **Traffic routing between regions**. Both regions can actively process events with bi-directional replication maintaining consistency. <!-- TODO confirm -->

- **Data replication between regions**. Both metadata and event data replicate asynchronously between regions with configurable replication policies to balance consistency and performance.

#### Region-down experience

This section describes what to expect when an Event Hubs namespace is configured for geo-replication and there's an outage in the primary region.

- **Detection and response**: You're responsible for deciding when to promote your secondary namespace to primary. Microsoft doesn't make this decision or initiate the process for you.

    To learn how to initiate failover, see [Promote secondary](../event-hubs/use-geo-replication.md#promote-secondary).

- **Notification**: TODO

- **Active requests**: All in-flight requests to the failed region are lost. After the promotion or failover completes, clients must reconnect to the namespace's endpoint.

- **Expected data loss**: TODO sync vs async

- **Expected downtime**: TODO Typically 15-30 minutes for failover completion plus time for client reconnection. <!-- PG: Please confirm -->

- **Traffic rerouting**: TODO

#### Region recovery

TODO

#### Testing for region failures

For geo-replication testing, temporarily promote the secondary region to primary and validate that applications can switch between regions with minimal disruption.

### Alternative multi-region approaches

<!-- TODO check this section -->
If you need multi-region capabilities beyond what geo-disaster recovery or geo-replication provide, consider implementing event replication using Azure Functions or custom applications. This approach gives you full control over replication logic, filtering, and transformation.

You can also implement active-active patterns using Event Hubs federation with custom replication functions to synchronize events across multiple independent namespaces. This provides flexibility for complex routing scenarios and selective replication.

## Backups

Event Hubs isn't designed as a long-term storage location for your data. Typically, data is stored in an event hub for a short period of time, and is then processed or persisted into another data storage system.

However, if you need to retain a copy of your events, consider using [Event Hubs Capture](../event-hubs/event-hubs-capture-overview.md), which saves copies of events to an Azure Blob Storage account.

## Service-level agreement

[!INCLUDE [SLA description](includes/reliability-service-level-agreement-include.md)]

Your availability SLA is higher when you use the Premium or Dedicated tiers.

### Related content

- [Event Hubs documentation](../event-hubs/event-hubs-about.md)
- [Azure reliability overview](/azure/reliability/overview)
