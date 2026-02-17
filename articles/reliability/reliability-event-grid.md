---
title: Reliability in Azure Event Grid
description: Learn how to make Azure Event Grid resilient to various potential outages and problems, including transient faults, availability zone outages, region outages, and service maintenance. Learn about backup and restore.
author: glynnniall
ms.author: pnp
ms.topic: reliability-article
ms.custom: subject-reliability
ms.service: azure-event-grid
ms.date: 02/04/2026
ai-usage: ai-assisted
---

<!-- For guidance on how to fill out this template, see https://learn.microsoft.com/help/patterns/level4/article-reliability -->

# Reliability in Azure Event Grid

Azure Event Grid is a fully managed messaging service that enables event-based communication between services and applications. This article describes how Azure Event Grid supports reliability and resilience, and what you should consider when you design reliable solutions that depend on Event Grid.

This article covers production deployment recommendations, how Event Grid behaves during faults and outages, and the options available to help you design for high availability and disaster recovery.

[!INCLUDE [Shared responsibility](includes/reliability-shared-responsibility-include.md)]

## Production deployment recommendations

Use the [Azure Well-Architected Framework guidance for Azure Event Grid](/azure/well-architected/service-guides/azure-event-grid) to inform your production deployments. The WAF guidance provides prescriptive recommendations for designing Event Grid solutions that align with Azure reliability, security, and operational best practices.

## Reliability architecture overview

Azure Event Grid routes events from event publishers to event subscribers. It's used both by customer applications and by Azure services to emit and consume events, such as notifications when resources are created, updated, or deleted.

Event Grid supports multiple resource types and deployment models:

- **Topics** are the primary entities that receive and store events, including:
  - **System topics**, which are created automatically by Azure services to emit events for specific Azure resource types.
  - **Custom topics**, which are created and managed by customers.
- **Domains** group multiple custom topics under a single endpoint.
- **Namespaces** are used with the standard tier and provide a container for multiple Event Grid resources.

Event Grid supports multiple tiers, including a long-standing basic tier and a newer standard tier. These tiers differ in how resources are deployed and managed.

<!-- TODO (PG confirmation): Expand this section with a clearer logical/physical architecture narrative and tier-specific nuances (basic vs standard) once verified. -->

## Resilience to transient faults

[!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)]

There are two primary transient fault scenarios to consider when you use Azure Event Grid:

- **Inbound connections (client to Event Grid):** When a client application publishes events to Event Grid, the client is responsible for handling transient failures. Applications should implement retry logic when publishing events.

- **Outbound connections (Event Grid to subscribers):** Event Grid delivers events to configured destinations. For these outbound connections, you configure retry policies on event subscriptions. These policies define how often and for how long Event Grid retries delivery when transient failures occur. For more information see [Message push delivery and retry with namespace topics](/azure/event-grid/namespace-delivery-retry)

Event Grid also supports features such as dead-lettering for undeliverable events. Configure retry and dead-letter policies according to your application’s reliability requirements, and refer to the [Event Grid documentation](/azure/event-grid/troubleshoot-network-connectivity#troubleshoot-transient-connectivity-issues)  for more information.

## Resilience to availability zone failures {#availability-zone-support}

[!INCLUDE [Resilience to availability zone failures](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)]

Azure Event Grid automatically supports availability zone resiliency in regions that support availability zones. You don't need to enable or configure this capability explicitly.

### Requirements

- **Region support:** Availability zone resiliency is available in all Azure regions that support availability zones.

### Cost

There's no additional cost for availability zone resiliency. You can't enable or disable this feature; it's included by default in supported regions.

### Configure availability zone support

No configuration is required. All Event Grid resources in supported regions are automatically zone-resilient.

### Behavior when all zones are healthy

When availability zones are healthy:

- **Traffic routing between zones:** Event Grid operates in an active-active model across availability zones. Client connections are automatically load-balanced across zones, and the service routes operations to available messaging infrastructure regardless of zone. 
- **Data replication between zones:** Event Grid replicates metadata and event data across availability zones to maintain resiliency. Multiple copies of the messaging store must acknowledge write operations before the service considers them complete, helping to ensure data consistency across zones during normal operations. 

<!-- TODO (PG confirmation): Confirm whether zone replication is synchronous and whether “active-active” is the preferred wording for Event Grid. -->

### Behavior during a zone failure

When an availability zone experiences an outage:

- **Detection and response:** Microsoft automatically detects zone failures and initiates failover to healthy zones. No customer action is required for zone-level failover.
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

Azure Event Grid provides limited built-in support for region-wide outages and also supports customer-managed multi-region designs.

### Geo-disaster recovery

Geo-disaster recovery is a service-managed feature that replicates **metadata only** for supported Event Grid resources to a paired Azure region. Event data isn't replicated.

This feature is designed as a best-effort, Microsoft-managed fallback for severe regional outages and isn't intended to provide rapid or predictable recovery times.

The following table shows the disaster recovery support for different Event Grid resource types:

| Event Grid resource | Client-side failover support | Geo disaster recovery (GeoDR) support |
| --- | --- | --- |
| Custom Topics | Supported | Cross-Geo / Regional |
| System Topics | Not supported | Enabled automatically |
| Domains | Supported | Cross-Geo / Regional |
| Partner Namespaces | Supported | Not supported |
| Namespaces | Supported | Not supported |

> [!IMPORTANT]
> Microsoft triggers Microsoft-managed failover at its discretion and on a best-effort basis. The failover time for Event Grid resources might differ from the failover time for other Azure services. If you need to be resilient to region outages, consider using the custom multi-region solutions described later in this article.

#### Failover types

You can choose between two failover options when configuring custom topics and domains:

**Microsoft-initiated failover**  
Microsoft-initiated failover is exercised by Microsoft in rare situations to fail over Event Grid resources from an affected region to the corresponding geo-paired region. Microsoft reserves the right to determine when this option will be exercised. This mechanism doesn't involve user consent before traffic is failed over.

> [!IMPORTANT]
> Microsoft triggers Microsoft-managed failover. It's likely to occur after a significant delay and is done on a best-effort basis. The failover of Event Grid resources might occur at a time that's different from the failover time of other Azure services.
> 
> If you need to be resilient to region outages, consider using one of the custom multi-region solutions for resiliency.

**Customer-initiated failover**  
Customer-initiated failover is defined by your custom disaster recovery plan for Azure Event Grid topics and domains. No event data is replicated to another region by Microsoft. While this failover option requires more effort, it enables faster failover and you control the choice of secondary regions.

You may want to disable Microsoft-initiated failover because:
- Microsoft-initiated failover is done on a best-effort basis
- Some geo pairs don't meet your organization's data residency requirements

#### Requirements

- **Region support:** Geo-disaster recovery is available only in Azure regions that have a paired region.
- **Resource types:** Custom topics and domains support geo-disaster recovery. System topics are enabled automatically. Other resource types, such as namespaces and partner namespaces, aren't supported.

#### Cost

There's no additional cost for geo-disaster recovery.

#### Configure multi-region support

To configure your failover preference:

- For **Microsoft-initiated failover**: Update the configuration for your topic or domain and select **Cross-Geo** (default)
- For **Customer-initiated failover**: Update the configuration for your topic or domain and select **Regional**

> [!NOTE]
> If you use a region without a geo pair, your metadata will only be replicated within the region regardless of configuration.

#### Behavior when all regions are healthy

- **Traffic routing between regions:** All traffic is routed to the primary region.
- **Data replication between regions:** Metadata is replicated to the paired region. Event data isn't replicated.

#### Behavior during a region failure

- **Detection and response:** Microsoft detects regional failures and determines whether and when to initiate failover.
- **Notification:** Azure Service Health provides outage notifications.
- **Active requests:** Active requests to the primary region will terminate and must be retried after failover. 
- **Expected data loss:** Event Grid preserves metadata during failover. Event data in the primary region is unavailable and might be lost if the region is unrecoverable.
- **Expected downtime:** Downtime depends on the severity of the outage and the time required for Microsoft to assess and initiate failover.
- **Traffic rerouting:** After failover, traffic is automatically routed to the secondary region.

#### Region recovery

Microsoft manages region recovery behavior depends on the specific outage scenario. Failover is typically treated as a one-way operation.

<!-- TODO (PG confirmation): Confirm region recovery/failback behavior and whether the service treats failover as one-way in all scenarios. -->

#### Test for region failures

Customers can't test geo-disaster recovery. Microsoft manages failover testing.

### Custom multi-region solutions for resiliency

For higher levels of control and predictability, you can implement custom multi-region architectures. This approach involves deploying separate Event Grid resources in multiple regions and managing failover at the application level.

With this model:

- You're responsible for deploying, configuring, and keeping resources in sync across regions.
- Your client applications must detect regional failures and route events to the appropriate region.

This approach provides greater flexibility but also places significant operational responsibility on you. Refer to the Event Grid documentation for detailed guidance and examples.

## Backup and restore

Azure Event Grid doesn't provide a built-in backup and restore mechanism for Event Grid resources or event data.

<!-- TODO (PG confirmation): Verify whether any backup/export options exist for Event Grid resources beyond standard ARM/IaC redeploy patterns. -->

## Resilience to service maintenance

Azure manages all service maintenance for Azure Event Grid. Maintenance activities are designed to avoid customer impact, and no downtime is expected.

## Service-level agreement

[!INCLUDE [Service-level agreement](includes/reliability-service-level-agreement-include.md)]

## Related content

- [Azure Well-Architected Framework guidance for Azure Event Grid](/azure/well-architected/service-guides/azure-event-grid)
- [Azure Event Grid documentation](/azure/event-grid/overview) 