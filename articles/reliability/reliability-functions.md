---
title: Reliability in Azure Functions
description: Learn how to make Azure Functions resilient to a variety of potential outages and problems, including transient faults, availability zone outages, region outages, and cross-region disaster recovery.
author: glynnniall
ms.author: glynnniall
ms.topic: reliability-article
ms.service: azure-functions
ms.custom: references_regions, subject-reliability
ms.date: 02/19/2026

#Customer intent: I want to understand reliability support in Azure Functions so that I can respond to and/or avoid failures in order to minimize downtime and data loss.
---

# Reliability in Azure Functions

[Azure Functions](/azure/azure-functions/functions-overview) is a serverless compute service that lets you run event-triggered code without having to explicitly provision or manage infrastructure. Functions provides a fully managed compute platform with high reliability, security, and zero administration for high-scale applications.

[!INCLUDE [Shared responsibility](includes/reliability-shared-responsibility-include.md)]

This article describes how to make Azure Functions resilient to a variety of potential outages and problems, including transient faults, availability zone failures, and region-wide failures. It also describes cross-region disaster recovery strategies and highlights key information about the Azure Functions service level agreement (SLA).

## Production deployment recommendations

For production workloads, we recommend that you:

> [!div class="checklist"]
>
> - Use zone-redundant hosting plans (Flex Consumption or Elastic Premium) in regions that support availability zones
> - Configure zone redundant storage (ZRS) for your function app's storage account
> - Deploy to multiple regions for critical workloads
> - Implement proper retry logic and error handling in your function code
> - Use Application Insights for monitoring and diagnostics
> - Follow the [function app configuration best practices](/azure/azure-functions/functions-best-practices#scalability-best-practices)

<!-- Additional guidance: Consider implementing circuit breaker patterns and proper timeout configurations for external dependencies -->

## Resilience to transient faults

[!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)]

**Microsoft responsibility:** The Azure Functions platform handles infrastructure-level transient faults automatically, including:

- Automatic instance replacement when hosts become unhealthy
- Built-in load balancing across healthy instances
- Platform-managed scaling based on demand

**Customer responsibility:** You are responsible for implementing transient fault handling in your function code, including:

- Implementing retry logic for external service calls
- Using timeout configurations appropriately
- Applying circuit breaker patterns for downstream dependencies
- Designing functions to be idempotent where possible

<!-- Key consideration: Functions have built-in retry policies for triggers, but you should implement additional retry logic for calls to external services within your function code -->

## Resilience to availability zone failures {#availability-zone-support}

[!INCLUDE [Resilience to availability zone failures](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)]

Azure Functions supports zone-redundant deployment, which means your function app instances are automatically spread across all availability zones in the selected region.

Azure Functions supports zone-redundant deployment, which means your function app instances are automatically spread across all availability zones in the selected region.

Availability zones support for Azure Functions depends on your [Functions hosting plan](/azure/azure-functions/functions-scale):

| Hosting plan | Support level | Configuration guide |
| ----- | ----- | ----- |
| [Flex Consumption plan](/azure/azure-functions/flex-consumption-plan) | GA | [Configure Flex Consumption availability zones](/azure/azure-functions/functions-premium-plan#availability-zones) |
| [Elastic Premium plan](/azure/azure-functions/functions-premium-plan) | GA | [Configure Premium availability zones](/azure/azure-functions/functions-premium-plan#availability-zones) |
| [Dedicated (App Service) plan](/azure/azure-functions/dedicated-plan) | GA | See [Reliability in Azure App Service](reliability-app-service.md). |
| [Consumption plan](/azure/azure-functions/consumption-plan) | n/a | Not supported by the Consumption plan. |

### Requirements

- **Region support:** Zone-redundant Azure Functions resources can be deployed into [any region that supports availability zones](/azure/reliability/availability-zones-service-support).
- **Hosting plan requirements:** You must use either Flex Consumption plan, Elastic Premium plan, or App Service plan (Dedicated). The Consumption plan does not support availability zones.
- **Storage account requirements:** You must use a [zone redundant storage account (ZRS)](/azure/storage/common/storage-redundancy#zone-redundant-storage) for your function app's default host storage account.

### Considerations

- **Flex Consumption plans:** You can enable availability zones during app creation or update an existing plan. Both creation and updates are supported.
- **Elastic Premium plans:** You can only enable availability zones during app creation. You cannot convert an existing Premium plan to use availability zones. For existing Premium plans without zone redundancy, migration to zone-redundant plans requires following the [migration guidance](migrate-functions.md).
- **Function apps must have a minimum of two always ready instances** for Premium plans when zone redundancy is enabled.
- **Storage account compatibility:** If you use a different type of storage account than ZRS, your app might behave unexpectedly during a zone outage.
- **Platform support:** Both Windows and Linux are supported.

### Instance distribution across zones

**Flex Consumption plans:**

When you configure Flex Consumption plan apps as zone-redundant, the platform automatically spreads instances of your function app across the zones in the selected region, with different rules for always-ready versus on-demand instances.

When zone redundancy is enabled in a Flex Consumption plan, instance spreading follows these rules:

- **Always-ready instances** are distributed across at least two zones in a round-robin fashion.
- **On-demand instances**, which are created as a result of event source volumes as the app scales beyond always-ready, are distributed across availability zones on a _best effort_ basis. This means that for on-demand instances, faster scale-out is given preference over even distribution across availability zones. The platform attempts to even-out distribution over time.
- To ensure zone resiliency with availability zones, the platform automatically maintains at least two always-ready instances for each [per-function scaling function or group](/azure/azure-functions/flex-consumption-plan#per-function-scaling), regardless of the always-ready configuration for the app. Any instances created by the platform are platform-managed, billed as always-ready instances, and don't change the always-ready configuration settings.

**Elastic Premium plans:**

When you configure Elastic Premium function app plans as zone-redundant, the platform automatically spreads the function app instances across the zones in the selected region.

Instance spreading with a zone-redundant deployment follows these rules, even as the app scales in and out:

- The minimum function app instance count is two.
- When you specify a capacity larger than the number of zones, the instances are spread evenly only when the capacity is a multiple of the number of zones.
- For a capacity value more than Number of Zones * Number of instances, extra instances are spread across the remaining zones.

>[!IMPORTANT]
>Azure Functions can run on the Azure App Service platform. In the App Service platform, plans that host Premium plan function apps are referred to as Elastic Premium plans, with SKU names like `EP1`. If you choose to run your function app on a Premium plan, make sure to create a plan with an SKU name that starts with `E`, such as `EP1`. App Service plan SKU names that start with `P`, such as `P1V2` (Premium V2 Small plan), are [Dedicated hosting plans](/azure/azure-functions/dedicated-plan). Because they're Dedicated and not Elastic Premium, plans with SKU names starting with `P` don't scale dynamically and can increase your costs.

### Cost

**Flex Consumption plans:**

- There's no separate meter associated with enabling availability zones. Pricing for instances used for a zone-redundant Flex Consumption app is the same as a single zone Flex Consumption app.
- When you enable availability zones in an app with always-ready instance configuration of fewer than two instances for each [per-function scaling function or group](/azure/azure-functions/flex-consumption-plan#per-function-scaling), the platform automatically creates two instances of the [always-ready](/azure/azure-functions/flex-consumption-plan#always-ready-instances) type for each per-function scaling function or group. These new instances are also billed as always-ready instances.

**Premium plans:**

- There's no extra cost associated with enabling availability zones. Pricing for a zone-redundant Premium App Service plan is the same as a single zone Premium plan.
- If you enable availability zones on a plan with fewer than two instances, the platform enforces a minimum instance count of two for that App Service plan, and you're charged for both instances.

For pricing details, see [Azure Functions pricing](https://azure.microsoft.com/pricing/details/functions/).

### Configure availability zone support {#create-a-function-app-in-a-zone-redundant-plan}

- **Create a new zone-redundant Azure Functions resource.** For step-by-step configuration instructions, see [Configure availability zones for Azure Functions](/azure/azure-functions/functions-premium-plan#availability-zones).
- **Flex Consumption plans:** You can enable availability zones during app creation or update an existing plan. Both creation and updates are supported.
- **Elastic Premium plans:** You can only enable availability zones during app creation. You cannot convert an existing Premium plan to use availability zones. For existing Premium plans without zone redundancy, migration to zone-redundant plans requires following the [migration guidance](migrate-functions.md).

### Behavior when all zones are healthy

This section describes what to expect when you configure Azure Functions for zone-redundant deployment, and all zones are operational.

- **Cross-zone operation:** When you configure zone redundancy on Azure Functions, requests are automatically spread across the instances in each availability zone. A request might go to any instance in any availability zone.
- **Cross-zone data replication:** Because Azure Functions is a stateless compute service, there's no customer data to replicate between zones. The platform handles configuration and metadata replication automatically.

### Behavior during a zone failure

This section describes what to expect when you configure Azure Functions for zone-redundant deployment, and there's an outage in one of the zones.

- **Detection and response:** The Azure Functions platform is responsible for detecting a failure in an availability zone. You don't need to do anything to initiate a zone failover.

- **Notification:**

[!INCLUDE [Availability zone down notification (Service Health and Resource Health)](includes/reliability-availability-zone-down-notification-service-resource-include.md)]

- **Active requests:** When an availability zone is unavailable, any requests in progress that are connected to an instance in the faulty availability zone are terminated and need to be retried.

- **Expected data loss:** Zone failures aren't expected to cause data loss because Azure Functions is a stateless service.

- **Expected downtime:** During zone outages, connections might experience brief interruptions that typically last a few seconds as traffic is redistributed. Ensure that your applications are prepared by following [transient fault handling guidance](#resilience-to-transient-faults).

- **Traffic rerouting:** When a zone is unavailable, Azure Functions detects the loss of the zone and creates new instances in another availability zone. Then, any new requests are automatically spread across all active instances.

### Zone recovery

When the availability zone recovers, Azure Functions automatically restores instances in the availability zone, removes any temporary instances created in the other availability zones, and reroutes traffic between your instances as normal.

### Test for zone failures

The Azure Functions platform manages traffic routing, failover, and zone recovery for zone-redundant resources. You don't need to initiate anything. Because this feature is fully managed, you don't need to validate availability zone failure processes.

## Migration to availability zone support {#availability-zone-migration}

To migrate existing Azure Functions to use availability zones:

**Microsoft responsibility:** Microsoft provides the platform capabilities to support availability zones and ensures the underlying infrastructure is available.

**Customer responsibility:** You are responsible for:

- Creating new zone-redundant function apps in supported hosting plans
- Migrating your function code and configuration to the new zone-redundant resources
- Testing the migrated functions to ensure proper operation
- Updating any dependent systems to use the new function app endpoints

**Migration process:**

1. **Premium plans:** Create a new Premium plan with zone redundancy enabled, then redeploy your functions
2. **Flex Consumption plans:** You can enable zone redundancy on existing plans or create new zone-redundant plans
3. **Migration guidance:** For detailed migration steps, see [Migrate Azure Functions to availability zones](migrate-functions.md)

**Cost considerations:** Zone redundancy requires minimum instance counts for Premium plans, which may increase costs.

## Resilience to region-wide failures

Azure Functions is a single-region service. If the region becomes unavailable, your Azure Functions resource is also unavailable. However, you can deploy separate resources into multiple regions and it is your responsibility to manage replication, traffic distribution, failover, etc. For more information on how to deploy across multiple regions, see [Custom multi-region solutions for resiliency](#custom-multi-region-solutions-for-resiliency).

### Custom multi-region solutions for resiliency

Because there's no built-in redundancy available, functions run in a function app in a specific Azure region. To avoid loss of execution during outages, you can redundantly deploy the same functions to function apps in multiple regions. To learn more about multi-region deployments, see the guidance in [Highly available multi-region web application](/azure/architecture/reference-architectures/app-service-web-app/multi-region).

**Microsoft responsibility:** Microsoft ensures that the Azure Functions platform is available in each region where you deploy your function apps.

**Customer responsibility:** You are responsible for:

- Deploying function apps to multiple regions
- Managing traffic distribution between regions
- Implementing failover mechanisms
- Ensuring data consistency across regions (if applicable)
- Monitoring and managing cross-region deployments

When you run the same function code in multiple regions, there are two patterns to consider: active-active and active-passive.

#### Active-active pattern for HTTP trigger functions

With an active-active pattern, functions in both regions are actively running and processing events, either in a duplicate manner or in rotation. You should use an active-active pattern in combination with [Azure Front Door](/azure/frontdoor/front-door-overview) for your critical HTTP triggered functions, which can route and round-robin HTTP requests between functions running in multiple regions. Front Door can also periodically check the health of each endpoint. When a function in one region stops responding to health checks, Azure Front Door takes it out of rotation, and only forwards traffic to the remaining healthy functions.

![Architecture for Azure Front Door and Functions.](/azure/azure-functions/media/functions-geo-dr/front-door.png)

#### Active-passive pattern for non-HTTP trigger functions

It's recommended that you use active-passive pattern for your event-driven, non-HTTP triggered functions, such as Service Bus and Event Hubs triggered functions.

To create redundancy for non-HTTP trigger functions, use an active-passive pattern. With an active-passive pattern, functions run actively in the region that's receiving events; while the same functions in a second region remain idle. The active-passive pattern provides a way for only a single function to process each message while providing a mechanism to fail over to the secondary region in a disaster. Function apps work with the failover behaviors of the partner services, such as [Azure Service Bus geo-recovery](/azure/service-bus-messaging/service-bus-geo-dr) and [Azure Event Hubs geo-recovery](/azure/event-hubs/event-hubs-geo-dr).

Consider an example topology using an Azure Event Hubs trigger. In this case, the active/passive pattern requires the following components:

- Azure Event Hubs deployed to both a primary and secondary region.
- [Geo-disaster enabled](/azure/service-bus-messaging/service-bus-geo-dr) to pair the primary and secondary event hubs. This also creates an _alias_ you can use to connect to event hubs and switch from primary to secondary without changing the connection info.
- Function apps are deployed to both the primary and secondary (failover) region, with the app in the secondary region essentially being idle because messages aren't being sent there.
- Function app triggers on the _direct_ (nonalias) connection string for its respective event hub.
- Publishers to the event hub should publish to the alias connection string.

![Active-passive example architecture.](/azure/azure-functions/media/functions-geo-dr/active-passive.png)

Before failover, publishers sending to the shared alias route to the primary event hub. The primary function app is listening exclusively to the primary event hub. The secondary function app is passive and idle. As soon as failover is initiated, publishers sending to the shared alias are routed to the secondary event hub. The secondary function app now becomes active and starts triggering automatically. Effective failover to a secondary region can be driven entirely from the event hub, with the functions becoming active only when the respective event hub is active.

Read more on information and considerations for failover with [Service Bus](/azure/service-bus-messaging/service-bus-geo-dr) and [Event Hubs](/azure/event-hubs/event-hubs-geo-dr).

For disaster recovery for Durable Functions, see [Disaster recovery and geo-distribution in Azure Durable Functions](/azure/azure-functions/durable/durable-functions-disaster-recovery-geo-distribution).

<!-- Note: Active-active pattern for non-HTTP trigger functions is also possible but requires careful consideration of message deduplication and coordination between regions -->

## Service-level agreement

[!INCLUDE [Service-level agreement](includes/reliability-service-level-agreement-include.md)]

For Azure Functions, the SLA varies based on the hosting plan:

- **Flex Consumption plan:** 99.95% availability SLA when zone redundancy is enabled
- **Premium plan:** 99.95% availability SLA when zone redundancy is enabled
- **Consumption plan:** No SLA provided
- **Dedicated (App Service) plan:** Inherits the SLA from the underlying App Service plan

For the most current SLA information, see [SLA for Azure Functions](https://azure.microsoft.com/support/legal/sla/functions/).

## Related content

- [Configure availability zones for Azure Functions](/azure/azure-functions/functions-premium-plan#availability-zones)
- [Disaster recovery and geo-distribution in Azure Durable Functions](/azure/azure-functions/durable/durable-functions-disaster-recovery-geo-distribution)
- [Create Azure Front Door](/azure/frontdoor/quickstart-create-front-door)
- [Event Hubs failover considerations](/azure/event-hubs/event-hubs-geo-dr#considerations)
- [Azure Architecture Center's guide on availability zones](/azure/architecture/high-availability/building-solutions-for-high-availability)
- [Reliability in Azure](./overview.md)
