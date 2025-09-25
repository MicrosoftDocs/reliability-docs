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

To learn about how to deploy Event Hubs to support your solution's reliability requirements, and how reliability affects other aspects of your architecture, see [Architecture best practices for Azure Event Hubs in the Azure Well-Architected Framework](/azure/well-architected/service-guides/eventhubs).

## Reliability architecture overview

Azure Event Hubs operates on a distributed architecture designed for high availability and fault tolerance. The service consists of front-end gateways that handle client connections, a distributed messaging infrastructure that manages event streams, and a storage layer that persists events for the configured retention period.

Event Hubs namespaces contain one or more event hubs. Each event hub is partitioned to enable parallel processing and horizontal scaling. The service automatically manages load distribution across partitions and handles failures transparently. 

In regions that support availability zones, Event Hubs infrastructure components are automatically distributed across zones to provide resilience against zone-level failures without requiring any customer configuration.

For Premium and Dedicated tiers, Event Hubs provides dedicated compute resources (Event Hubs Clusters) that offer predictable performance, enhanced security through customer-managed keys, and network isolation capabilities through private endpoints and dedicated virtual network integration.

**Sources:**
- [Event Hubs architecture](../event-hubs/event-hubs-about.md#key-architecture-components)
- [Event Hubs scalability](../event-hubs/event-hubs-scalability.md)
- [Event Hubs Premium and Dedicated tiers](../event-hubs/compare-tiers.md)

## Transient faults

[!INCLUDE [Transient fault description](includes/reliability-transient-fault-description-include.md)]

Azure Event Hubs handles transient faults through built-in retry mechanisms in the client SDKs and automatic request routing during temporary service disruptions. When working with Event Hubs:

- Use the built-in retry policies in Event Hubs SDKs, which implement exponential backoff by default. The SDKs automatically retry operations for retriable errors like network timeouts, throttling responses, or when the server is busy.
- Configure appropriate timeout values based on your application requirements. The default timeout is typically 60 seconds, but you can adjust this based on your scenario.
- Implement checkpointing in your event processor to track progress and enable recovery from the last processed position after transient failures.
- For send operations, use batching to improve throughput and reduce the impact of transient network issues on individual messages. <!-- PG: please verify this is accurate. -->

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

- **Traffic routing between zones**. Event Hubs operates in an active/active model where all zones simultaneously process incoming events. Client connections are distributed across zones through the front-end gateways, and events are written to all zones synchronously. Partition ownership is distributed across zones to ensure balanced processing. <!-- TODO confirm -->

- **Data replication between zones**. Event Hubs uses synchronous replication across availability zones. When an event is sent to Event Hubs, it's written to multiple zones before the send operation is acknowledged to the client. This ensures zero data loss even if an entire zone becomes unavailable. The synchronous replication approach provides strong consistency guarantees while maintaining low latency through optimized replication protocols.

### Zone-down experience

This section describes what to expect when Event Hubs namespaces are configured for zone redundancy and there's an availability zone outage.

- **Detection and response**: The Event Hubs service is responsible for detecting a failure in an availability zone. You don't need to do anything to initiate a zone failover.

+ **Notification**: Event Hubs doesn't notify you when a zone is down. However, you can use [Azure Service Health](/azure/service-health/overview) to understand the overall health of the Event Hubs service, including any zone failures.

  Set up alerts on these services to receive notifications of zone-level problems. For more information, see [Create Service Health alerts in the Azure portal](/azure/service-health/alerts-activity-log-service-notifications-portal).

- **Active requests**: During a zone failure, active requests might be dropped. Your clients should have sufficient [retry logic](../iot/concepts-manage-device-reconnections.md#retry-patterns) implemented to handle transient faults.

- **Expected data loss**: No data loss occurs with zone-redundant Event Hubs as events are synchronously replicated across zones before acknowledgment. <!-- TODO confirm -->

- **Expected downtime**: A zone failure shouldn't cause downtime to your namespace.

- **Traffic rerouting**: Event Hubs detects the loss of the zone. Then, any new requests are automatically redirected to a new primary instance of the service in one of the healthy availability zones. <!-- TODO confirm -->
    
    Event Hubs client SDKs and frontend gateways handle connection management and retry logic transparently.

### Zone recovery

When an availability zone recovers, Event Hubs automatically reintegrates the zone into the active service topology. The recovered zone begins accepting new connections and processing events alongside the other zones. Data that was replicated to surviving zones during the outage remains intact, and normal synchronous replication resumes across all zones. No customer action is required for zone recovery and reintegration.

### Testing for zone failures

Because Event Hubs fully manages traffic routing, failover, and failback for zone failures, you don't need to validate availability zone failure processes or provide any further input.

## Multi-region support

Azure Event Hubs provides two multi-region capabilities:

- *Geo-disaster recovery (Standard tier and above)*.  Geo-Disaster recovery ensures that the entire configuration of a namespace (Event Hubs, Consumer Groups, and settings) is continuously replicated from a primary namespace to a secondary namespace when paired. With Geo-Disaster recovery, you can initiate a once-only failover move from the primary to the secondary at any time. The failover move points the chosen alias name for the namespace to the secondary namespace. After the move, the pairing is then removed. The failover is nearly instantaneous once initiated.

 - *Geo-replication (Premium tier)* provides replication of both metadata (entities, configuration and properties) and data (event payloads) for the namespace. This feature ensures that metadata and data of a namespace is continuously replicated from a primary region to the secondary region. The namespace can be thought of as being extended to more than one region, with one region being the primary and the other being the secondary.
    
    At any time, the secondary region can be promoted to become a primary region. Promoting a secondary repoints the namespace FQDN (fully qualified domain name) to the selected secondary region, and the previous primary region is demoted to a secondary region.

    For geo-replication scenarios, see [Event Hubs geo-replication scenarios](../event-hubs/geo-replication.md#scenarios)


**Sources:**
- [Event Hubs geo-disaster recovery](../event-hubs/event-hubs-geo-dr.md)
- [Event Hubs geo-replication](../event-hubs/geo-replication.md)
- [Multi-region federation](../event-hubs/event-hubs-federation-overview.md)

### Region support

Both geo-disaster recovery and geo-replication support pairing between any two Azure regions where Event Hubs is available. Because you're not restricted to Azure paired regions, you have the flexibility to choose secondary regions based on your requirements for latency, compliance, or data residency.

**Sources:**
- [Event Hubs geo-disaster recovery regions](../event-hubs/event-hubs-geo-dr.md#availability-zones)
- [Azure Event Hubs availability](../event-hubs/event-hubs-availability-and-consistency.md)

### Requirements

- *Geo-disaster recovery (Standard tier and above):*
    - The following combinations of primary and secondary namespaces are supported:  

        | Primary namespace tier | Allowed secondary namespace tier |
        | ----------------- | -------------------- |
        | Standard | Standard, Dedicated | 
        | Premium | Premium | 
        | Dedicated | Dedicated | 

        > [!IMPORTANT]
        > You can't pair namespaces that are in the same dedicated cluster. You can pair namespaces that are in separate clusters. 

    - The secondary namespace must not contain any event hubs before pairing.

- *Geo-replication (Premium tier only):*
    - Both namespaces must be Premium tier with the same processing unit configuration.
    - Namespaces must be in different regions.

**Sources:**
- [Geo-disaster recovery setup requirements](../event-hubs/event-hubs-geo-dr.md#setup)
- [Geo-replication requirements](../event-hubs/event-hubs-geo-replication.md#prerequisites)

### Considerations

- *Geo-disaster recovery (Standard tier and above)* enables instantaneous continuity of operations with the same configuration, but **does not replicate the event data**. Unless the disaster caused the loss of all zones, the event data that is preserved in the primary Event Hub after failover will be recoverable and the historic events can be obtained from there once access is restored. For replicating event data and operating corresponding namespaces in active/active configurations to cope with outages and disasters, don't lean on this Geo-disaster recovery feature set, but follow the [replication guidance](event-hubs-federation-overview.md).  

     Microsoft Entra role-based access control (RBAC) assignments to entities in the primary namespace aren't replicated to the secondary namespace. Create role assignments manually in the secondary namespace to secure access to them. 


**Sources:**
- [Geo-disaster recovery concepts](../event-hubs/event-hubs-geo-dr.md#basic-concepts)
- [Geo-replication considerations](../event-hubs/event-hubs-geo-replication.md#considerations)

### Cost

- *Geo-disaster recovery (Standard tier and above).* You're charged for both the primary and secondary namespaces even though only the primary actively processes events. The secondary namespace must match the primary's capacity configuration.

- *Geo-replication (Premium tier only).* Because geo-replication maintains active replicas in multiple regions, it incurs higher operational costs. You pay for processing units in each region plus inter-region data transfer charges for replication traffic.

**Sources:**
- [Event Hubs pricing](https://azure.microsoft.com/pricing/details/event-hubs/)
- [Bandwidth pricing](https://azure.microsoft.com/pricing/details/bandwidth/)

### Configure multi-region support


- **Create geo-disaster recovery pairing.** To configure disaster recovery between primary and secondary namespaces see [Implement geo-disaster recovery](../event-hubs/event-hubs-geo-dr.md#setup).

- **Enable geo-replication (Premium only).** To set up active-active replication between Premium namespaces, see [Configure geo-replication](../event-hubs/event-hubs-geo-replication.md#set-up-geo-replication).

- **Disable geo-disaster recovery or geo-replication.** To break the pairing between namespaces, see [Disable disaster recovery](../event-hubs/event-hubs-geo-dr.md#setup).

- **Migrate to multi-region configuration.**  To create a new namespace in the secondary region and establish pairing for existing single-region deployments, see  [Relocate Azure Event Hubs to another region](/azure-resource-manager/management/relocation/relocation-event-hub?tabs=azure-portal).

**Sources:**
- [Event Hubs geo-disaster recovery setup](../event-hubs/event-hubs-geo-dr.md#setup)
- [Event Hubs geo-replication configuration](../event-hubs/event-hubs-geo-replication.md)
- [PowerShell commands for geo-DR](../event-hubs/event-hubs-geo-dr.md#powershell)

### Capacity planning and management

When planning for multi-region deployments, ensure that both regions have sufficient capacity to handle the full load if one region fails. For geo-disaster recovery, the secondary namespace must match the primary's throughput unit or processing unit configuration. Although the secondary remains passive during normal operations, it must be ready to handle all traffic immediately after failover.

For geo-replication, both regions actively process events, so size each region to handle the expected load plus overhead for replication traffic. Monitor metrics like incoming messages, throughput unit utilization, and replication lag to ensure adequate capacity.

**Sources:**
- [Event Hubs auto-inflate](../event-hubs/event-hubs-auto-inflate.md)
- [Monitor Event Hubs metrics](../event-hubs/monitor-event-hubs.md)

### Normal operations

**Traffic routing between regions**. For geo-disaster recovery, only the primary region actively processes events during normal operations. The secondary remains passive in standby mode. For geo-replication, both regions can actively process events with bi-directional replication maintaining consistency.

**Data replication between regions**. With geo-disaster recovery, only configuration metadata replicates continuously and synchronously between regions. Event data remains in the primary region only. With geo-replication, both metadata and event data replicate asynchronously between regions with configurable replication policies to balance consistency and performance.

**Sources:**
- [Geo-disaster recovery normal operations](../event-hubs/event-hubs-geo-dr.md#basic-concepts)
- [Geo-replication data flow](../event-hubs/event-hubs-geo-replication.md#data-replication)

### Region-down experience

**Customer-managed failover**: For both geo-disaster recovery and geo-replication, you must monitor region health and initiate failover manually. 

The geo-disaster recovery feature doesn't support an automatic failover. Failing over activates the secondary namespace and removes the primary namespace from the Geo-Disaster Recovery pairing. Create another namespace to have a new geo-disaster recovery pair.

For geo-replication (Premium tier only):

- **Detection and response**: You must detect the region failure through monitoring and initiate failover through Azure portal, PowerShell, or CLI. To learn how to initiate failover, see [Initiate failover](../event-hubs/event-hubs-geo-dr.md#failover-flow).
- **Notification**: Configure Azure Monitor alerts for Event Hubs metrics and Service Health notifications to detect region issues.
- **Active requests**: All in-flight requests to the failed region are lost. Clients must reconnect to the new primary endpoint.
- **Expected data loss**: All events not consumed from the primary region before failure are lost (geo-disaster recovery). With geo-replication, only events not yet replicated are lost.
- **Expected downtime**: Typically 15-30 minutes for failover completion plus time for client reconnection.
- **Traffic rerouting**: After failover, the secondary namespace becomes primary with a new connection endpoint. Update client applications to use the new endpoint or use the alias if configured.

**Sources:**
- [Initiate failover](../event-hubs/event-hubs-geo-dr.md#fail-over)
- [Event Hubs monitoring](../event-hubs/monitor-event-hubs.md)
- [Geo-replication failover](../event-hubs/event-hubs-geo-replication.md#failover)

### Region recovery

**Customer-managed failover recovery**: After the original primary region recovers, you must manually re-establish the pairing and optionally fail back. For geo-disaster recovery, create a new pairing with the recovered region as secondary, then perform another failover if you want to return to the original region. This process involves potential data loss of events sent to the temporary primary.

For geo-replication, re-establish replication to the recovered region and promote it back to primary when ready. The bi-directional replication minimizes data loss during failback operations.

**Sources:**
- [Geo-disaster recovery failback considerations](../event-hubs/event-hubs-geo-dr.md#replication)
- [Geo-replication recovery procedures](../event-hubs/event-hubs-geo-replication.md#failback)

### Testing for region failures

To test your disaster recovery configuration, you can perform a planned failover during a maintenance window. Initiate failover from primary to secondary namespace and verify that your applications can connect and process events from the new primary. Monitor the failover duration and validate that your runbooks and automation work correctly. After testing, you can fail back to the original configuration.

For geo-replication testing, temporarily promote the secondary region to primary and validate that applications can switch between regions with minimal disruption.

**Sources:**
- [Test failover procedures](../event-hubs/event-hubs-geo-dr.md#fail-over)
- [Disaster recovery testing best practices](../event-hubs/event-hubs-geo-dr.md#management)

### Alternative multi-region approaches

If you need multi-region capabilities beyond what geo-disaster recovery or geo-replication provide, consider implementing event replication using Azure Functions or custom applications. This approach gives you full control over replication logic, filtering, and transformation.

You can also implement active-active patterns using Event Hubs federation with custom replication functions to synchronize events across multiple independent namespaces. This provides flexibility for complex routing scenarios and selective replication.

For architectural patterns and examples, see:
- [Event Hubs federation patterns](../event-hubs/event-hubs-federation-patterns.md)
- [Multi-region federation with Azure Functions](../event-hubs/event-hubs-federation-replicator-functions.md)
- [Cross-region replication with Event Hubs](https://learn.microsoft.com/azure/architecture/reference-architectures/event-hubs/cross-region-replication)

**Sources:**
- [Event Hubs federation overview](../event-hubs/event-hubs-federation-overview.md)
- [Federation replicator functions](../event-hubs/event-hubs-federation-replicator-functions.md)
- [Event Hubs federation patterns](../event-hubs/event-hubs-federation-patterns.md)

## Service-level agreement

[!INCLUDE [SLA description](includes/reliability-service-level-agreement-include.md)]


### Related content

- [Azure reliability overview](/azure/reliability/overview)
- [Availability zones overview](/azure/reliability/availability-zones-overview)
- [Azure regions and availability zones](/azure/reliability/availability-zones-service-support)
- [Event Hubs documentation](../event-hubs/event-hubs-about.md)
