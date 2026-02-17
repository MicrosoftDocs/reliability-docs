---
title: Reliability in Azure Event Grid
description: Learn how to make Azure Event Grid resilient to various potential outages and problems, including transient faults, availability zone outages, region outages, and service maintenance. Learn about backup and restore.
author: glynnniall
ms.author: glynnniall
ms.topic: reliability-article
ms.custom: subject-reliability
ms.service: event-grid
ms.date: 02/04/2026
---

<!-- For guidance on how to fill out this template, see https://learn.microsoft.com/help/patterns/level4/article-reliability -->

# Reliability in Azure Event Grid

Azure Event Grid is a fully managed messaging service that enables event-based communication between services and applications. This article describes how Azure Event Grid supports reliability and resilience, and what you should consider when you design reliable solutions that depend on Event Grid.

This article covers production deployment recommendations, how Event Grid behaves during faults and outages, and the options available to help you design for high availability and disaster recovery.

[!INCLUDE [Shared responsibility](../includes/reliability-shared-responsibility-include.md)]

## Production deployment recommendations

Use the Azure Well-Architected Framework (WAF) service guide for Azure Event Grid to inform your production deployments. The WAF guidance provides prescriptive recommendations for designing Event Grid solutions that align with Azure reliability, security, and operational best practices.

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

[!INCLUDE [Resilience to transient faults](../includes/reliability-transient-fault-description-include.md)]

There are two primary transient fault scenarios to consider when you use Azure Event Grid:

- **Inbound connections (client to Event Grid):** When a client application publishes events to Event Grid, the client is responsible for handling transient failures. Applications should implement retry logic when publishing events.

- **Outbound connections (Event Grid to subscribers):** Event Grid delivers events to configured destinations. For these outbound connections, you configure retry policies on event subscriptions. These policies define how often and for how long Event Grid retries delivery when transient failures occur.

Event Grid also supports features such as dead-lettering for undeliverable events. Configure retry and dead-letter policies according to your application’s reliability requirements, and refer to the Event Grid documentation for detailed configuration guidance.

## Resilience to availability zone failures

[!INCLUDE [Resilience to availability zone failures](../includes/reliability-availability-zone-description-include.md)]

Azure Event Grid automatically supports availability zone resiliency in regions that support availability zones. You don't need to enable or configure this capability explicitly.

### Requirements

- **Region support:** Availability zone resiliency is available in all Azure regions that support availability zones.

### Cost

There's no additional cost for availability zone resiliency. You can't enable or disable this feature; it's included by default in supported regions.

### Configure availability zone support

No configuration is required. All Event Grid resources in supported regions are automatically zone-resilient.

### Behavior when all zones are healthy

When availability zones are healthy:

- **Traffic routing between zones:** Event Grid operates in an active-active model across availability zones.
- **Data replication between zones:** Event Grid replicates metadata and event data across availability zones to maintain resiliency.

<!-- TODO (PG confirmation): Confirm whether zone replication is synchronous and whether “active-active” is the preferred wording for Event Grid. -->

### Behavior during a zone failure

When an availability zone experiences an outage:

- **Detection and response:** Azure Event Grid detects the zone failure and routes traffic away from the affected zone.
- **Notification:** Use Azure Service Health to understand the overall health of the service, including any zone failures, and set up Service Health alerts to notify you of problems.
- **Active requests:** In-flight requests can interrupted, we recommend you retry your requests.
- **Expected data loss:** No data loss is expected due to replication across zones.
- **Expected downtime:** You might experience a brief interruption.
- **Traffic rerouting:** Traffic is automatically rerouted to healthy zones.

<!-- TODO (PG confirmation): Confirm whether the guidance can quantify downtime (for example “a few seconds”) and whether SDK connection management wording applies to Event Grid. -->

### Zone recovery

When the affected zone recovers, Event Grid automatically reintegrates it into the service without requiring customer action.

### Test for zone failures

Microsoft manages availability zone failure testing. Customers can't directly simulate zone failures for Event Grid.

## Resilience to region-wide failures

Azure Event Grid provides limited built-in support for region-wide outages and also supports customer-managed multi-region designs.

### Geo-disaster recovery

Geo-disaster recovery is a service-managed feature that replicates **metadata only** for supported Event Grid resources to a paired Azure region. Event data isn't replicated.

This feature is designed as a best-effort, Microsoft-managed fallback for severe regional outages and isn't intended to provide rapid or predictable recovery times.

> [!IMPORTANT]
> Microsoft triggers Microsoft-managed failover at its discretion and on a best-effort basis. The failover time for Event Grid resources might differ from the failover time for other Azure services. If you need to be resilient to region outages, consider using the custom multi-region solutions described later in this article.

#### Requirements

- **Region support:** Geo-disaster recovery is available only in Azure regions that have a paired region.
- **Resource types:** Custom topics and domains support geo-disaster recovery. System topics are enabled automatically. Other resource types, such as namespaces and partner namespaces, aren't supported.

#### Cost

There's no additional cost for geo-disaster recovery.

#### Configure multi-region support

No customer configuration is required. Geo-disaster recovery is either enabled or disabled at the service level.

#### Behavior when all regions are healthy

- **Traffic routing between regions:** All traffic is routed to the primary region.
- **Data replication between regions:** Metadata is replicated to the paired region. Event data isn't replicated.

#### Behavior during a region failure

- **Detection and response:** Microsoft detects regional failures and determines whether and when to initiate failover.
- **Notification:** Azure Service Health provides outage notifications.
- **Active requests:** Active requests to the primary region are terminated and must be retried after failover.
- **Expected data loss:** Metadata loss isn't expected. Event data in the primary region is unavailable and might be lost if the region is unrecoverable.
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

[!INCLUDE [Service-level agreement](../includes/reliability-service-level-agreement-include.md)]

## Related content

- Azure Well-Architected Framework guidance for Azure Event Grid
- Azure Event Grid documentation