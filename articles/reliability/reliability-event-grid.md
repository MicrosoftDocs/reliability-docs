---
title: Reliability in Azure Event Grid
description: Learn how to make Azure Event Grid resilient to various potential outages and problems, including transient faults, availability zone failures, and region-wide failures.
author: glynnniall
ms.author: pnp
ms.topic: reliability-article
ms.custom: subject-reliability
ms.service: azure-event-grid
ms.date: 02/25/2026
ai-usage: ai-assisted
---

# Reliability in Azure Event Grid

[Azure Event Grid](/azure/event-grid/overview) is a fully managed messaging service that enables event-based communication between services and applications. It's commonly used for building event-driven architectures and integrating Azure services with custom applications.

[!INCLUDE [Shared responsibility](includes/reliability-shared-responsibility-include.md)]

This article describes how to make Azure Event Grid resilient to various potential outages and problems, including transient faults, availability zone failures, and region-wide failures. It also highlights key information about the Azure Event Grid service-level agreement (SLA).

## Production deployment recommendations

The Azure Well-Architected Framework provides recommendations across reliability, security, cost, operations, and performance. To understand how these areas influence each other and contribute to a reliable Azure Event Grid solution, see [Azure Well-Architected Framework guidance for Azure Event Grid](/azure/well-architected/service-guides/azure-event-grid).

## Reliability architecture overview

[!INCLUDE [Introduction to reliability architecture overview section](includes/reliability-architecture-overview-introduction-include.md)]

### Logical architecture

Azure Event Grid routes events from event *publishers* to event *consumers*. It's used both by customer applications and by Azure services to emit and consume events, such as notifications when resources are created, updated, or deleted.

Event Grid supports multiple resource types and deployment models:

- **Topics** are the primary entities that receive and store events.

  **System topics** are created automatically by Azure services to emit events for specific Azure resource types. **Custom topics** are created and managed by you.

  Topics can support both [push and pull delivery](/azure/event-grid/pull-delivery-overview#push-and-pull-delivery).

- **Event domains** group multiple custom topics under a single endpoint, simplifying publishing of events. For more information, see [Understand event domains for managing Event Grid topics](/azure/event-grid/event-domains).

- **Namespaces** are used with the standard tier, and provide a container for multiple Event Grid resources. For more information, see [Azure Event Grid namespace concepts](/azure/event-grid/concepts-event-grid-namespaces).

Event Grid supports multiple tiers, including the basic tier and the standard tier. These tiers differ in how resources are deployed and managed.

### Physical architecture

Azure Event Grid is a fully managed service. Microsoft manages the underlying infrastructure, including compute and storage resources. In supported regions, Event Grid [automatically distributes resources across availability zones](#resilience-to-availability-zone-failures) to provide built-in zone redundancy.

## Resilience to transient faults

[!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)]

When you use Event Grid, consider the following practices to ensure your solution is resilient to transient faults:

- **Event publishers:** When a client application publishes events to Event Grid, it's responsible for handling transient failures. Applications should implement retry logic when publishing events. For more information, see [Troubleshoot transient conection issues](/azure/event-grid/troubleshoot-network-connectivity#troubleshoot-transient-connectivity-issues).

  We recommend you use the [Event Grid data plane SDKs](/azure/event-grid/sdk-overview#data-plane-sdks), which automatically provide transient fault handling.

- **Event consumers:** Event Grid delivers events to configured destinations. For these outbound connections, you configure retry policies on event subscriptions. These policies define how often and for how long Event Grid retries delivery when failures occur, including transient faults. For more information see [Message push delivery and retry with namespace topics](/azure/event-grid/namespace-delivery-retry)

- **Idempotency:** It's a good practice to design your eventing architecture for *idempotency*, which means that events can safely be received and processed multiple times. For example, if a transient fault or another problem happens when your application is processing an event, then an idempotent approach means your application can reprocess the message and recover.

  You're responsible for designing your eventing architecture and application to support idempotency. For general information, see [Idempotency](/azure/architecture/serverless/event-hubs-functions/resilient-design#idempotency).

- **Dead-lettering:** Event Grid supports *dead-lettering* for undeliverable events, which helps to persist data during longer-lasting faults in event consumers. For more information, see [Dead lettering for event subscriptions to namespaces topics in Azure Event Grid](/azure/event-grid/dead-letter-event-subscriptions-namespace-topics).

## Resilience to availability zone failures

[!INCLUDE [Resilience to availability zone failures](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)]

Azure Event Grid resources are zone-redundant in regions that support availability zones. Zone redundancy means that even when an availability zone has a problem, your Event Grid resources continue to work by using infrastructure in other zones. Event data is automatically replicated across three availability zones for intra-region resiliency, and Event Grid self-heals during a zone-wide outage. You don't need to enable or configure this capability.

:::image type="complex" source="./media/reliability-event-grid/zone-redundant.png" alt-text="Diagram that shows zone-redundant Event Grid resources in a region with three availability zones." border="false":::
The diagram shows a various Event Grid resources, each distributed across three availability zones.
:::image-end:::

### Requirements

**Region support:** Zone redundancy is available in [all Azure regions that support availability zones](./regions-list.md).

### Cost

There's no additional cost for zone redundancy. You can't enable or disable this feature; it's included by default in supported regions.

### Configure availability zone support

No configuration is required. All Event Grid resources in supported regions are automatically zone-redundant.

### Behavior when all zones are healthy

This section describes what to expect when an Event Grid resource is zone-redundant, and all zones are operational.

- **Cross-zone operation:** Event Grid operates in an active-active model across availability zones. Client connections are automatically load-balanced across zones, and the service routes operations to available messaging infrastructure regardless of zone.

- **Cross-zone data replication:** Event Grid automatically replicates metadata and event data across availability zones to maintain resiliency.

### Behavior during a zone failure

This section describes what to expect when an Event Grid resource is zone-redundant, and there's an outage in one of the zones.

- **Detection and response:** Event Grid automatically detects zone failures and initiates failover to healthy zones. You don't need to do anything to initiate a zone failover.

[!INCLUDE [Availability zone down notification (Service Health only)](./includes/reliability-availability-zone-down-notification-service-include.md)]

- **Active requests:** During a zone failure, Event Grid might drop active requests. If your clients handle [transient faults](#resilience-to-transient-faults) appropriately, such as by retrying after a short period of time, they typically avoid significant impact.

- **Expected data loss:** The Event Grid zone redundancy model is designed to enable resilience to zone failures with minimal impact. However, during a zone failure, there's some possibility of data loss.

  If you need to ensure your application doesn't lose data even during a zone failure, you should:
  - Design your event producers and consumers to follow [transient fault handling recommendations](#resilience-to-transient-faults), including retries and idempotency.
  - Plan for event durability at the source, or in a durable event store.

- **Expected downtime:** A zone failure might cause a few seconds of downtime. If your clients handle [transient faults](#resilience-to-transient-faults) appropriately, such as by retrying after a short period of time, they typically avoid significant impact.

- **Traffic rerouting:** Event Grid detects the loss of the zone and automatically redirects new requests to infrastructure in one of the healthy availability zones.

### Zone recovery

When the affected zone recovers, Event Grid automatically reintegrates it into the service without requiring customer action. The recovered zone then accepts new connections and processes messages alongside the other zones. Data that replicated to surviving zones during the outage remains intact, and normal replication resumes across all zones. You don't need to take action for zone recovery or reintegration.

### Test for zone failures

Event Grid manages traffic routing, failover, and zone recovery for zone failures, so you don't need to validate availability zone failure processes or provide further input.

## Resilience to region-wide failures

Event Grid resources are deployed into a single region. If there's a region-wide failure, your Event Grid resources are unavailable.

In [paired Azure regions](./regions-paired.md), Event Grid provides limited geo-disaster recovery for your Event Grid resources' metadata. You can also design and build your own multi-region solution, which can support your disaster recovery planning. The following table shows how different Event Grid resource types support each model:

| Event Grid resource | Supports geo-disaster recovery | Supports custom solution |
| --- | --- | --- |
| Custom topics | Supported | Supported |
| System topics | Enabled automatically | Not supported |
| Domains | Supported | Supported |
| Namespaces | Not supported | Supported |
| Partner namespaces | Not supported | Supported |

### Geo-disaster recovery

Geo-disaster recovery replicates Event Grid metadata to your primary region's paired region for supported resources. Event data isn't replicated.

Geo-disaster recovery is designed as a best-effort, Microsoft-managed fallback for severe regional outages and isn't intended to provide rapid or predictable recovery times. Microsoft-initiated failover is exercised by Microsoft in rare situations to fail over Event Grid resources from an affected region to the corresponding geo-paired region. Microsoft reserves the right to determine when this option will be exercised. This mechanism doesn't involve customer consent before traffic is failed over.

> [!IMPORTANT]
> Microsoft triggers Microsoft-managed failover. It's likely to occur after a significant delay and is done on a best-effort basis. The failover of Event Grid resources might occur at a time that's different from the failover time of other Azure services.
>
> If you need to be resilient to region outages, consider using one of the custom multi-region solutions for resiliency.

You can optionally disable geo-disaster recovery and use your own [custom multi-region solution](#custom-multi-region-solutions-for-resiliency) that meets your requirements for region selection, failover time, and more. When you disable geo-disaster recovery, no event data is replicated to another region by Microsoft.

This feature isn't available in regions without a paired region.

#### Requirements

- **Region support:** Geo-disaster recovery is available only in Azure regions that have a paired region.

- **Resource types:** Custom topics and domains support geo-disaster recovery. System topics are enabled for geo-disasaster recovery automatically. Other resource types, such as namespaces and partner namespaces, aren't supported.

#### Cost

There's no additional cost for geo-disaster recovery.

#### Configure multi-region support

In supported regions, system topics are automatically configured for geo-disaster recovery. For other Event Grid resource types:

- **To enable geo-disaster recovery:** Update the configuration for your topic or domain and select **Cross-Geo** (default).

- **To disable geo-diaster recovery:** Update the configuration for your topic or domain and select **Regional**.

#### Behavior when all regions are healthy

This section describes what to expect when an Event Grid resource is configured for geo-disaster recovery, and all regions are operational.

- **Cross-region operation:** All traffic is routed to the primary region.

- **Cross-region data replication:** Metadata is synchronously replicated to the paired region. Event data isn't replicated.

#### Behavior during a region failure

This section describes what to expect when an Event Grid resource is configured for geo-disaster recovery, and there's an outage in the primary region.

- **Detection and response:** Microsoft detects region failures and determines whether and when to initiate failover.

[!INCLUDE [Region down notification (Service Health only)](./includes/reliability-region-down-notification-service-include.md)]

- **Active requests:** Active requests to the primary region are terminated. Client applications must retry these requests after failover completes.

- **Expected data loss:**

  - *Metadata:* Event Grid preserves metadata during failover. Because all metadata changes are synchronously replicated, no loss of metadata is expected.

  - *Event data:* Event data in the primary region is unavailable and might be lost if the region is unrecoverable.

    After a failover occurs, new data is processed from the paired region. The unprocessed events are dispatched from the primary region as soon as the outage is mitigated. If the primary region's recovery requires a longer time than the [time-to-live value set on events](/azure/event-grid/delivery-and-retry#dead-letter-events), the data in the primary region might be dropped. To mitigate this data loss, we recommend that you [configure a dead-letter destination for an event subscription](/azure/event-grid/manage-event-delivery).

    If the affected region is lost and nonrecoverable, there will be some data loss. In the best-case scenario, the consumer is keeping up with the publishing rate and only a few seconds of data is lost. The worst-case scenario would be when the consumer isn't actively processing events and with a maximum time to live of 24 hours, the data loss can be up to 24 hours.

    > [!NOTE]
    > Event Grid can't guarantee data retention during a region outage. If you need guaranteed retention, you need to design your application to durably store events in another data store.

- **Expected downtime:** The amount of downtime depends on the severity of the outage and the time required for Microsoft to assess and initiate failover. You should expect downtime to be at least one hour, and maybe longer.

  After failover is initiated, within five minutes, Event Grid begins to accept traffic for topics and subscriptions, including for creation, update, and deletion operations.

- **Redistribution:** After failover completes, traffic is automatically routed to the secondary region.

#### Region recovery

Microsoft manages region recovery and the recovery process depends on the specific outage scenario. In general, failover is typically treated as a one-way operation.

#### Test for region failures

Azure Event Grid manages traffic routing, failover, and recovery for geo-disaster recovery. You don't need to initiate anything. Because this feature is fully managed, you don't need to validate region failure processes.

### Custom multi-region solutions for resiliency

You may want to disable, or not rely on, Microsoft-initiated failover for any of these reasons:

- You require event data to be replicated across regions, and not just metadata.
- You need to guarantee a specific failover time or approach. Microsoft-initiated failover is done on a best-effort basis.
- Your region isn't paired with another Azure region.
- Your region's pair doesn't meet your organization's data residency requirements.

For higher levels of control and predictability, you can implement custom multi-region architectures. This approach involves deploying separate Event Grid resources in multiple regions and managing failover at the application level. When you use this model, you're responsible for deploying, configuring, and keeping resources in sync across regions.

Consider the following factors when designing a multi-region solution:

- **Replication:** You should implement a custom process to replicate your Event Grid resources and their configuration between primary and secondary regions. Remember to replicate client identities, CA certificates, client groups, topic spaces, and permission bindings, where applicable. You can decide whether to implement manual or automated replication.

- **Failover approaches:** You can choose whether to create an active-active or active-passive solution:

  - *Active-active solutions* can be achieved by replicating the metadata and balancing load across the namespaces.
  - *Active-passive solutions* can be achieved by replicating the metadata to keep the secondary namespace ready so that when the primary namespace is unavailable, the traffic can be directed to secondary namespace.

- **Health monitoring:** You can use built-in health APIs provided by Event Grid to monitor the health of topics.

  Your client applications must detect failures of a region and route events to another appropriate region.

  Alternatively, you can implement a *concierge service* that directs clients to the primary or secondary endpoints for their topics or namespaces by performing health checks on those endpoints. The concierge service can be a web application that is geo-replicated and kept reachable using DNS-redirection techniques or services like Azure Traffic Manager.

For more information about one approach, including example code, see [Client-side failover implementation in Azure Event Grid](/azure/event-grid/custom-disaster-recovery-client-side).

## Backup and restore

Event Grid is primarily an event routing service and doesn't have native backup or restore features.

If you need to implement backup capabilities, or if you have long-term retention needs, we recommend you perform archiving in your application. To do so, you should build logic to route or copy your events to a durable store, such as Azure Blob Storage, in parallel with the primary delivery path. If downstream systems are unavailable, your application can use the archive to replay the events.

## Resilience to service maintenance

[!INCLUDE [Service maintenance (no special callouts)](includes/reliability-maintenance-include.md)]

## Service-level agreement

[!INCLUDE [Service-level agreement](includes/reliability-service-level-agreement-include.md)]

The Event Grid availability SLA covers publishing of events.

## Related content

- [Reliability in Azure](/azure/reliability/overview)
- [Azure Well-Architected Framework guidance for Azure Event Grid](/azure/well-architected/service-guides/azure-event-grid)
- [Azure Event Grid documentation](/azure/event-grid/overview)
