---
title: Reliability in Azure Event Grid
description: Learn how to make Azure Event Grid resilient to various potential outages and problems, including transient faults, availability zone failures, and region-wide failures.
author: glynnniall
ms.author: pnp
ms.topic: reliability-article
ms.custom: subject-reliability
ms.service: azure-event-grid
ms.date: 02/04/2026
ai-usage: ai-assisted
---

# Reliability in Azure Event Grid

[Azure Event Grid](/azure/event-grid/overview) is a fully managed messaging service that enables event-based communication between services and applications. It's commonly used for building event-driven architectures and integrating Azure services with custom applications.

[!INCLUDE [Shared responsibility](includes/reliability-shared-responsibility-include.md)]

This article describes how to make Azure Event Grid resilient to various potential outages and problems, including transient faults, availability zone failures, and region-wide failures. It also highlights key information about the Azure Event Grid service-level agreement (SLA).

## Production deployment recommendations

The Azure Well-Architected Framework provides recommendations across reliability, security, cost, operations, and performance. To understand how these areas influence each other and contribute to a reliable Azure Event Grid solution, see [Azure Well-Architected Framework guidance for Azure Event Grid](/azure/well-architected/service-guides/azure-event-grid).

## Reliability architecture overview

Azure Event Grid routes events from event publishers to event subscribers. It's used both by customer applications and by Azure services to emit and consume events, such as notifications when resources are created, updated, or deleted.

Event Grid supports multiple resource types and deployment models:

- **Topics** are the primary entities that receive and store events, including:
  - **System topics**, which are created automatically by Azure services to emit events for specific Azure resource types.
  - **Custom topics**, which are created and managed by you.
- **Domains** group multiple custom topics under a single endpoint.
- **Namespaces** are used with the standard tier and provide a container for multiple Event Grid resources.

Event Grid supports multiple tiers, including the basic tier and the standard tier. These tiers differ in how resources are deployed and managed.

Azure Event Grid is a fully managed service. Microsoft manages the underlying infrastructure, including compute and storage resources. Event Grid automatically distributes resources across availability zones to provide built-in redundancy in supported regions.

<!-- TODO (PG confirmation): Expand this section with a clearer logical/physical architecture narrative and tier-specific nuances (basic vs standard) once verified. -->

## Resilience to transient faults

[!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)]

There are two primary transient fault scenarios to consider when you use Azure Event Grid:

- **Publishing events:** When a client application publishes events to Event Grid, the client is responsible for handling transient failures. Applications should implement retry logic when publishing events.

- **Receiving events:** Event Grid delivers events to configured destinations. For these outbound connections, you configure retry policies on event subscriptions. These policies define how often and for how long Event Grid retries delivery when transient failures occur. For more information see [Message push delivery and retry with namespace topics](/azure/event-grid/namespace-delivery-retry)

Event Grid also supports features such as dead-lettering for undeliverable events. Configure retry and dead-letter policies according to your application's reliability requirements. For more information, see [Event Grid documentation](/azure/event-grid/troubleshoot-network-connectivity#troubleshoot-transient-connectivity-issues).

## Resilience to availability zone failures

[!INCLUDE [Resilience to availability zone failures](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)]

Azure Event Grid automatically supports zone redundancy in regions that support availability zones. You don't need to enable or configure this capability explicitly.

<!-- TODO explain what it does -->

### Requirements

- **Region support:** Zone redundancy is available in [all Azure regions that support availability zones](./regions-list.md).

### Cost

There's no additional cost for zone redundancy. You can't enable or disable this feature; it's included by default in supported regions.

### Configure availability zone support

No configuration is required. All Event Grid resources in supported regions are automatically zone-resilient.

### Behavior when all zones are healthy

This section describes what to expect when an Event Grid resource is zone-redundant, and all zones are operational.

- **Cross-zone operation:** Event Grid operates in an active-active model across availability zones. Client connections are automatically load-balanced across zones, and the service routes operations to available messaging infrastructure regardless of zone.

- **Cross-zone data replication:** Event Grid replicates metadata and event data across availability zones to maintain resiliency. Multiple copies of the messaging store must acknowledge write operations before the service considers them complete, helping to ensure data consistency across zones during normal operations.

<!-- TODO (PG confirmation): Confirm whether zone replication is synchronous and whether “active-active” is the preferred wording for Event Grid. -->

### Behavior during a zone failure

This section describes what to expect when an Event Grid resource is zone-redundant, and there's an outage in one of the zones.

- **Detection and response:** Microsoft automatically detects zone failures and initiates failover to healthy zones. You don't need to do anything to initiate a zone failover.

[!INCLUDE [Availability zone down notification (Service Health only)](./includes/reliability-availability-zone-down-notification-service-include.md)]

- **Active requests:** During a zone failure, Event Grid might drop active requests. If your clients handle [transient faults](#resilience-to-transient-faults) appropriately by retrying after a short period of time, they typically avoid significant impact.

- **Expected data loss:** No data loss occurs during a zone failure because Event Grid synchronously replicates messages across zones before acknowledgment.

- **Expected downtime:** A zone failure might cause a few seconds of downtime. If your clients handle [transient faults](#resilience-to-transient-faults) appropriately by retrying after a short period of time, they typically avoid significant impact.

- **Traffic rerouting:** Event Grid detects the loss of the zone and automatically redirects new requests to another replica in one of the healthy availability zones.

<!-- TODO (PG confirmation): Confirm whether the guidance can quantify downtime (for example “a few seconds”) and whether SDK connection management wording applies to Event Grid. -->

### Zone recovery

When the affected zone recovers, Event Grid automatically reintegrates it into the service without requiring customer action. The recovered zone then accepts new connections and processes messages alongside the other zones. Data that replicated to surviving zones during the outage remains intact, and normal synchronous replication resumes across all zones. You don't need to take action for zone recovery or reintegration.

### Test for zone failures

Event Grid manages traffic routing, failover, and zone recovery for zone failures, so you don't need to validate availability zone failure processes or provide further input.

## Resilience to region-wide failures

Event Grid resources are deployed into a single region. If there's a region-wide failure, your Event Grid resources are unavailable.

In paired regions, Event Grid provides limited geo-disaster recovery for metadata. You can also design and build your own multi-region solution, which can support your disaster recovery planning. The following table shows how different Event Grid resource types support each model:

| Event Grid resource | Supports geo-disaster recovery | Supports custom solution |
| --- | --- | --- |
| Custom topics | Supported | Supported |
| System topics | Enabled automatically | Not supported |
| Domains | Supported | Supported |
| Namespaces | Not supported | Supported |
| Partner namespaces | Not supported | Supported |

### Geo-disaster recovery

Geo-disaster recovery replicates Event Grid metadata to your primary region's paired region for supported resources. Event data isn't replicated. This feature isn't available in regions without a paired region.

This feature is designed as a best-effort, Microsoft-managed fallback for severe regional outages and isn't intended to provide rapid or predictable recovery times. Microsoft-initiated failover is exercised by Microsoft in rare situations to fail over Event Grid resources from an affected region to the corresponding geo-paired region. Microsoft reserves the right to determine when this option will be exercised. This mechanism doesn't involve user consent before traffic is failed over.

> [!IMPORTANT]
> Microsoft triggers Microsoft-managed failover. It's likely to occur after a significant delay and is done on a best-effort basis. The failover of Event Grid resources might occur at a time that's different from the failover time of other Azure services.
>
> If you need to be resilient to region outages, consider using one of the custom multi-region solutions for resiliency.

You can optionally disable geo-disaster recovery and use your own [custom multi-region solution](#custom-multi-region-solutions-for-resiliency) that meets your requirements for region selection, failover time, and more. When you disable geo-disaster recovery, no event data is replicated to another region by Microsoft. 

#### Requirements

- **Region support:** Geo-disaster recovery is available only in Azure regions that have a paired region.

- **Resource types:** Custom topics and domains support geo-disaster recovery. System topics are enabled for geo-disasaster recovery automatically. Other resource types, such as namespaces and partner namespaces, aren't supported.

#### Cost

There's no additional cost for geo-disaster recovery.

#### Configure multi-region support

- **To enable geo-disaster recovery:** Update the configuration for your topic or domain and select **Cross-Geo** (default)

- **To disable geo-diaster recovery:** Update the configuration for your topic or domain and select **Regional**

#### Behavior when all regions are healthy

This section describes what to expect when an Event Grid resource is configured for geo-disaster recovery, and all regions are operational.

- **Cross-region operation:** All traffic is routed to the primary region.

- **Cross-region data replication:** Metadata is synchronously replicated to the paired region. Event data isn't replicated.

#### Behavior during a region failure

This section describes what to expect when an Event Grid resource is configured for geo-disaster recovery, and there's an outage in the primary region.

- **Detection and response:** Microsoft detects region failures and determines whether and when to initiate failover.

[!INCLUDE [Region down notification (Service Health only)](./includes/reliability-region-down-notification-service-include.md)]

- **Active requests:** Active requests to the primary region are terminated and must be retried after failover completes.

- **Expected data loss:**

  - *Metadata:* Event Grid preserves metadata during failover. Because all metadata changes are synchronously replicated, no loss of metadata is expected.

  - *Event data:* Event data in the primary region is unavailable and might be lost if the region is unrecoverable.

    After a failover occurs, new data is processed from the paired region. The unprocessed events are dispatched from the primary region as soon as the outage is mitigated. If the region recovery required longer time than the [time-to-live value set on events](/azure/event-grid/delivery-and-retry#dead-letter-events), the data could get dropped.
    
    To mitigate this data loss, we recommend that you [configure a dead-letter destination for an event subscription](/azure/event-grid/manage-event-delivery). If the affected region is lost and nonrecoverable, there will be some data loss. In the best-case scenario, the subscriber is keeping up with the publishing rate and only a few seconds of data is lost. The worst-case scenario would be when the subscriber isn't actively processing events and with a maximum time to live of 24 hours, the data loss can be up to 24 hours.

- **Expected downtime:** The amount of downtime depends on the severity of the outage and the time required for Microsoft to assess and initiate failover. You should expect downtime to be at least one hour, and maybe longer.

  After failover is initiated, within five minutes, Event Grid begins to accept traffic for topics and subscriptions, including for creation, update, and deletion operations.

- **Redistribution:** After failover, traffic is automatically routed to the secondary region.

#### Region recovery

Microsoft manages region recovery and the recovery process depends on the specific outage scenario. In general, failover is typically treated as a one-way operation.

#### Test for region failures

Azure Event Grid manages traffic routing, failover, and recovery for geo-disaster recovery. You don't need to initiate anything. Because this feature is fully managed, you don't need to validate region failure processes.

### Custom multi-region solutions for resiliency

You may want to disable, or not rely on, Microsoft-initiated failover for any of these reasons:

- You require event data to be replicated as well as metadata.
- You need to guarantee a specific failover time and approach. Microsoft-initiated failover is done on a best-effort basis.
- Your region isn't paired with another Azure region.
- Your region's pair doesn't meet your organization's data residency requirements.

For higher levels of control and predictability, you can implement custom multi-region architectures. This approach involves deploying separate Event Grid resources in multiple regions and managing failover at the application level. When you use this model, you're responsible for deploying, configuring, and keeping resources in sync across regions. The following list provides some suggestions for how you might design a solution that fits your needs:

- **Replication:** You should implement a custom process to replicate namespaces and their configuration between primary and secondary regions. Remember to replicate client identities, CA certificates, client groups, topic spaces, and permission bindings. You can decide whether to implement manual or automated replication.

- **Failover approaches:** You can choose whether to create an active-active or active-passive solution:

  - *Active-active solution* can be achieved by replicating the metadata and balancing load across the namespaces.
  - (Active-passive solutions* can be achieved by replicating the metadata to keep the secondary namespace ready so that when the primary namespace is unavailable, the traffic can be directed to secondary namespace.

- **Health monitoring:** Your client applications must detect regional failures and route events to the appropriate region. You can use built-in health APIs provided by Event Grid to monitor the topic.

  Alternatively, you can implement a *concierge service* that directs clients to the primary or secondary endpoints for their topics or namespaces by performing health checks on those endpoints. The concierge service can be a web application that is geo-replicated and kept reachable using DNS-redirection techniques or services like Azure Traffic Manager.

For more information about one approach, including example code, see [Client-side failover implementation in Azure Event Grid](/azure/event-grid/custom-disaster-recovery-client-side).

## Backup and restore

Azure Event Grid doesn't provide a built-in backup and restore mechanism for Event Grid resources or event data.

<!-- TODO (PG confirmation): Verify whether any backup/export options exist for Event Grid resources beyond standard ARM/IaC redeploy patterns. -->

## Resilience to service maintenance

[!INCLUDE [Service maintenance (no special callouts)](includes/reliability-maintenance-include.md)]

## Service-level agreement

[!INCLUDE [Service-level agreement](includes/reliability-service-level-agreement-include.md)]

The Event Grid availability SLA covers publishing events.

## Related content

- [Reliability in Azure](/azure/reliability/overview)
- [Azure Well-Architected Framework guidance for Azure Event Grid](/azure/well-architected/service-guides/azure-event-grid)
- [Azure Event Grid documentation](/azure/event-grid/overview)
