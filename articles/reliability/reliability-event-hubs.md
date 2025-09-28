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

<!-- TODO check this section -->
Azure Event Hubs operates on a distributed architecture designed for high availability and fault tolerance. The service consists of front-end gateways that handle client connections, a distributed messaging infrastructure that manages event streams, and a storage layer that persists events for the configured retention period.

Event Hubs namespaces contain one or more event hubs. Each event hub is partitioned to enable parallel processing and horizontal scaling. The service automatically manages load distribution across partitions and handles failures transparently. 

In regions that support availability zones, Event Hubs infrastructure components are automatically distributed across zones to provide resilience against zone-level failures without requiring any customer configuration.

For Premium and Dedicated tiers, Event Hubs provides dedicated compute resources (Event Hubs Clusters) that offer predictable performance, enhanced security through customer-managed keys, and network isolation capabilities through private endpoints and dedicated virtual network integration.

## Transient faults

[!INCLUDE [Transient fault description](includes/reliability-transient-fault-description-include.md)]

Azure Event Hubs handles transient faults through built-in retry mechanisms in the client SDKs and automatic request routing during temporary service disruptions. When working with Event Hubs:

- **Use the built-in retry policies in Event Hubs SDKs**, which implement exponential backoff by default. The SDKs automatically retry operations for retriable errors like network timeouts, throttling responses, or when the server is busy.
- **Configure appropriate timeout values** based on your application requirements. The default timeout is typically 60 seconds, but you can adjust this based on your scenario.
- **Implement checkpointing** in your event processor to track progress and enable recovery from the last processed position after transient failures.
- **Use batching for send operations** to improve throughput and reduce the impact of transient network issues on individual messages. <!-- PG: please verify this is accurate. -->

## Availability zone support

[!INCLUDE [Availability zone support description](includes/reliability-availability-zone-description-include.md)]

Azure Event Hubs supports zone-redundant deployments in regions with availability zones. When you create an Event Hubs namespace in a supported region, zone redundancy is automatically enabled at no additional cost. The service transparently replicates your configuration, metadata, and data across all three availability zones in the region, providing automatic failover capability without any customer intervention required.

Event Hubs implements zone-redundant storage for both the messaging infrastructure and the event data. All components including compute, networking, and storage are replicated across zones. This ensures that even if an entire availability zone becomes unavailable, Event Hubs continues to operate without data loss or interruption to your streaming applications. The zone-redundant deployment model applies to all Event Hubs features including Capture, Schema Registry, and Kafka protocol support.

### Region support

Zone-redundant Event Hubs namespaces can be deployed in [any Azure region that supports availability zones](./regions-list.md).

### Cost

There's no extra cost for zone redundancy with Event Hubs.

### Configure availability zone support

Event Hubs namespaces automatically support zone redundancy when deployed in [supported regions](#region-support). No further configuration is required.

### Normal operations

This section describes what to expect when Event Hubs namespaces are configured for zone redundancy and all availability zones are operational.

- **Traffic routing between zones**. Event Hubs operates in an active/active model where infrastructure in all of the region's zones simultaneously process incoming events. <!-- PG: Please verify, and advise whether South Central US going to 4 zones changes anything. -->

    Partition ownership is distributed across zones to ensure balanced processing. <!-- TODO confirm -->

- **Data replication between zones**. Event Hubs uses synchronous replication across availability zones. When an event is sent to Event Hubs, it's written to multiple zones before the send operation is acknowledged to the client. This ensures zero data loss even if an entire zone becomes unavailable. The synchronous replication approach provides strong consistency guarantees while maintaining low latency through optimized replication protocols. <!-- PG: Please verify replication is synchronous. -->

### Zone-down experience

This section describes what to expect when Event Hubs namespaces are configured for zone redundancy and there's an availability zone outage.

- **Detection and response**: The Event Hubs service is responsible for detecting a failure in an availability zone. You don't need to do anything to initiate a zone failover.

+ **Notification**: Event Hubs doesn't notify you when a zone is down. However, you can use [Azure Service Health](/azure/service-health/overview) to understand the overall health of the Event Hubs service, including any zone failures.

  Set up alerts on these services to receive notifications of zone-level problems. For more information, see [Create Service Health alerts in the Azure portal](/azure/service-health/alerts-activity-log-service-notifications-portal).

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

Azure Event Hubs provides two multi-region capabilities:

- *Geo-disaster recovery (Standard tier and above):* Geo-disaster recovery, as its name implies, is a solution to assist in disaster recovery scenarios, including the catastrophic loss of a region. Geo-disaster recovery only replicates the configuration and metadata of your namespace, but it doesn't replicate event data. It assists disaster recovery by ensuring a namespace in another region is preconfigured and ready to immediately accept events from clients. Geo-disaster recovery is intended as a one-way recovery solution, and doesn't support failback to the prior primary region.

- *Geo-replication (Premium and Dedicated tiers):* Geo-replication replicates your namespace's configuration and the event data using a replication approach that you configure. Gei-replication ensures that your events are available in another region, and you can switch to use the secondary region when you need to.

Typically you would only use one of these two approaches. <!-- PG: Do we have documented guidance about when to use one over the other? -->

### Geo-disaster recovery

Geo-disaster recovery, available in the Standard tier and above, ensures that the entire configuration of a namespace (Event Hubs, Consumer Groups, and settings) is continuously replicated from a primary namespace to a secondary namespace.

> [!IMPORTANT]
> Geo-disaster recovery enables instantaneous continuity of operations with the same configuration, but **doesn't replicate the event data**. If you need to replicate event data, or if you need to operate multiple regional namespaces in active/active configurations for resiliency purposes, consider using [geo-replication](#geo-replication) or an [alternative multi-region approach](#alternative-multi-region-approaches).

When you use geo-disaster recovery, you can initiate a single-time, one-way failover move from the primary to the secondary at any time. The failover move points the chosen alias name for the namespace to the secondary namespace. After the move, the pairing is then removed. The failover is nearly instantaneous once initiated.

This section provides a summary of some of the important aspects of geo-disaster recovery, but it's important you review the full documentation to understand exactly how it works. For more information, see [Azure Event Hubs - Geo-disaster recovery](../event-hubs/event-hubs-geo-dr.md).

#### Region support

You can select any Azure region where Event Hubs is available for your primary or secondary namespace. Because you're not restricted to Azure paired regions, you have the flexibility to choose secondary regions based on your requirements for latency, compliance, or data residency.

#### Requirements

- **Tiers:** Geo-disaster recovery supports specific combinations of tiers for the primary and secondary namespaces. For detailed information, see TODO.

- **Empty namespaces:** The secondary namespace must not contain any event hubs before pairing. <!-- TODO confirm -->

#### Considerations

- Microsoft Entra role-based access control (RBAC) assignments to entities in the primary namespace aren't replicated to the secondary namespace. Create role assignments manually in the secondary namespace to secure access to them.

- **Application design.** Geo-disaster recovery requires specific considerations when you design your client applications. For more information, see [Considerations](../event-hubs/event-hubs-geo-dr.md#considerations).

#### Cost

You're charged for both the primary and secondary namespaces even though only the primary actively processes events. The secondary namespace must match the primary's capacity configuration.

#### Configure multi-region support

- **Create geo-disaster recovery pairing.** To configure disaster recovery between primary and secondary namespaces see [Implement geo-disaster recovery](../event-hubs/event-hubs-geo-dr.md#setup).

- **Disable geo-disaster recovery.** To break the pairing between namespaces, see [Disable disaster recovery](../event-hubs/event-hubs-geo-dr.md#setup).

#### Capacity planning and management

When planning for multi-region deployments, ensure that both regions have sufficient capacity to handle the full load if one region fails. For geo-disaster recovery, the secondary namespace must match the primary's throughput unit or processing unit configuration. Although the secondary remains passive during normal operations, it must be ready to handle all traffic immediately after failover.

#### Normal operations

This section describes what to expect when an IoT hub is configured for geo-disaster recovery, and the primary region is operational.

- **Traffic routing between regions**. TODO alias. Only the primary region actively processes events during normal operations. The secondary remains passive in standby mode.

- **Data replication between regions**. Only configuration metadata is replicated between regions. Replication of configuration occurs continuously and is performed asynchronously. Event data remains in the primary region only.

#### Region-down experience

This section describes what to expect when an Event Hubs namespace is configured for geo-disaster recovery and there's an outage in the primary region.

- **Detection and response:** You must monitor region health and initiate failover manually. Microsoft does not perform failover automatically.

    To initiate failover, TODO.

    > [!IMPORTANT]
    > Failing over activates the secondary namespace and removes the primary namespace from the geo-disaster recovery pairing. You must create a new namespace to have a new geo-disaster recovery pair.

- **Notification**: TODO ARH?

- **Active requests:** TODO

- **Expected data loss:** TODO config might not be updated if recent and complex. Data will not be replicated at all.

- **Expected downtime:** TODO failover time? "near instantaneous" - what are the TTLs on the DNS?

- **Traffic rerouting:** TODO the function of the alias is to redirect

#### Region recovery

After the original primary region recovers, you must manually re-establish the pairing and optionally fail back. For geo-disaster recovery, create a new pairing with the recovered region as secondary, then perform another failover if you want to return to the original region. This process involves potential data loss of events sent to the temporary primary.

<!-- TODO integrate this -->
Unless the disaster caused the loss of all zones, the event data that is preserved in the primary Event Hub after failover will be recoverable and the historic events can be obtained from there once access is restored.

#### Testing for region failures

To test your disaster recovery configuration, you can perform a planned failover during a maintenance window. Initiate failover from primary to secondary namespace and verify that your applications can connect and process events from the new primary. Monitor the failover duration and validate that your runbooks and automation work correctly. After testing, you can fail back to the original configuration.

### Geo-replication

Geo-replication, which is available in the Premium and Dedicated tiers, provides replication of both metadata (entities, configuration and properties) and data (event payloads) for the namespace. This feature ensures that metadata and data of a namespace is continuously replicated from a primary region to the secondary region. The namespace can be thought of as being extended to more than one region, with one region being the primary and the other being the secondary.
    
At any time, the secondary region can be promoted to become a primary region. Promoting a secondary repoints the namespace FQDN (fully qualified domain name) to the selected secondary region, and the previous primary region is demoted to a secondary region.

This section provides a summary of some of the important aspects of geo-replication, but it's important you review the full documentation to understand exactly how it works. For more information, see [Azure Event Hubs Geo-replication](../event-hubs/geo-replication.md).

#### Region support

Both geo-disaster recovery and geo-replication support pairing between any two Azure regions where Event Hubs is available. Because you're not restricted to Azure paired regions, you have the flexibility to choose secondary regions based on your requirements for latency, compliance, or data residency.

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

Event Hubs isn't designed as a long-term storage location for your data. Typically, data is stored in an event hub for a short period of time, and if required it can be then persisted into another data storage system.

However, if you need to retain a copy of your events, consider using [Event Hubs Capture](../event-hubs/event-hubs-capture-overview.md), which saves copies of events to an Azure Blob Storage account.

## Service-level agreement

[!INCLUDE [SLA description](includes/reliability-service-level-agreement-include.md)]

Your availability SLA is higher when you use the Premium or Dedicated tiers.

### Related content

- [Event Hubs documentation](../event-hubs/event-hubs-about.md)
- [Azure reliability overview](/azure/reliability/overview)
