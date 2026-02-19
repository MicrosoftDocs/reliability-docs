---
title: Reliability in Azure Functions
description: Learn about reliability support in Azure Functions including availability zones and cross-region disaster recovery strategies.
author: glynnniall
ms.author: glynnniall
ms.topic: reliability-article
ms.service: azure-functions
ms.custom: references_regions, subject-reliability
ms.date: 02/19/2026

#Customer intent: I want to understand reliability support in Azure Functions so that I can respond to and/or avoid failures in order to minimize downtime and data loss.
---

# Reliability in Azure Functions

This article describes reliability support in [Azure Functions](/azure/azure-functions/functions-overview), covering both intra-regional resiliency with [availability zones](#availability-zone-support) and [cross-region recovery and business continuity](#cross-region-disaster-recovery-and-business-continuity). For a more detailed overview of reliability principles in Azure, see [Azure reliability](/azure/architecture/framework/resiliency/overview).

For step-by-step configuration guidance, see [Configure availability zones for Azure Functions](/azure/azure-functions/functions-availability-zones-configuration).

Availability zones support for Azure Functions depends on your [Functions hosting plan](/azure/azure-functions/functions-scale):

| Hosting plan | Support level | Configuration guide |
| ----- | ----- | ----- |
| [Flex Consumption plan](/azure/azure-functions/flex-consumption-plan) | GA | [Configure Flex Consumption availability zones](/azure/azure-functions/functions-availability-zones-configuration?pivots=flex-consumption-plan) |
| [Elastic Premium plan](/azure/azure-functions/functions-premium-plan) | GA | [Configure Premium availability zones](/azure/azure-functions/functions-availability-zones-configuration?pivots=premium-plan) |
| [Dedicated (App Service) plan](/azure/azure-functions/dedicated-plan) | GA | See [Reliability in Azure App Service](reliability-app-service.md). |
| [Consumption plan](/azure/azure-functions/consumption-plan) | n/a | Not supported by the Consumption plan. |

[!INCLUDE [Availability zone description](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)]

Azure Functions supports a [zone-redundant deployment](availability-zones-service-support.md).  

## <a name="availability-zone-support"></a>Availability zones support

Azure Functions provides availability zone support through different mechanisms depending on your hosting plan:

### How availability zones work with Flex Consumption plans

When you configure Flex Consumption plan apps as zone redundant, the platform automatically spreads instances of your function app across the zones in the selected region, with different rules for always-ready versus on-demand instances.

When zone redundancy is enabled in a Flex Consumption plan, instance spreading follows these rules:

- **Always-ready instances** are distributed across at least two zones in a round-robin fashion.
- **On-demand instances**, which are created as a result of event source volumes as the app scales beyond always-ready, are distributed across availability zones on a _best effort_ basis. This means that for on-demand instances, faster scale-out is given preference over even distribution across availability zones. The platform attempts to even-out distribution over time.
- To ensure zone resiliency with availability zones, the platform automatically maintains at least two always-ready instances for each [per-function scaling function or group](/azure/azure-functions/flex-consumption-plan#per-function-scaling), regardless of the always-ready configuration for the app. Any instances created by the platform are platform-managed, billed as always-ready instances, and don't change the always-ready configuration settings.

### How availability zones work with Premium plans  

When you configure Elastic Premium function app plans as zone redundant, the platform automatically spreads the function app instances across the zones in the selected region.

Instance spreading with a zone-redundant deployment follows these rules, even as the app scales in and out:

- The minimum function app instance count is two. 
- When you specify a capacity larger than the number of zones, the instances are spread evenly only when the capacity is a multiple of the number of zones. 
- For a capacity value more than Number of Zones * Number of instances, extra instances are spread across the remaining zones.

>[!IMPORTANT]
>Azure Functions can run on the Azure App Service platform. In the App Service platform, plans that host Premium plan function apps are referred to as Elastic Premium plans, with SKU names like `EP1`. If you choose to run your function app on a Premium plan, make sure to create a plan with an SKU name that starts with `E`, such as `EP1`. App Service plan SKU names that start with `P`, such as `P1V2` (Premium V2 Small plan), are [Dedicated hosting plans](/azure/azure-functions/dedicated-plan). Because they're Dedicated and not Elastic Premium, plans with SKU names starting with `P` don't scale dynamically and can increase your costs.  
### Regional availability

Zone redundancy is not available in all regions. For current regional availability information and configuration steps, see [Configure availability zones for Azure Functions](/azure/azure-functions/functions-availability-zones-configuration#regional-availability).

### Prerequisites and considerations

Before enabling zone redundancy for Azure Functions, consider the following requirements:

**For Flex Consumption plans:**
- You must use a [zone redundant storage account (ZRS)](/azure/storage/common/storage-redundancy#zone-redundant-storage) for your function app's [default host storage account](/azure/azure-functions/storage-considerations#storage-account-requirements).
- You can enable availability zones during app creation or update an existing plan.
- Both creation and updates are supported.

**For Elastic Premium plans:**
- You must use a [zone redundant storage account (ZRS)](/azure/storage/common/storage-redundancy#zone-redundant-storage) for your function app's [default host storage account](/azure/azure-functions/storage-considerations#storage-account-requirements).
- You can only enable availability zones during app creation. You can't convert an existing Premium plan to use availability zones.
- Function apps must have a minimum of two [always ready instances](/azure/azure-functions/functions-premium-plan#always-ready-instances).
- For existing Premium plans without zone redundancy, migration to zone-redundant plans requires following the [migration guidance](migrate-functions.md).

**For both hosting plans:**
- If you use a different type of storage account than ZRS, your app might behave unexpectedly during a zonal outage.
- Both Windows and Linux are supported.

### Pricing considerations

**Flex Consumption plans:**
- There's no separate meter associated with enabling availability zones. Pricing for instances used for a zone-redundant Flex Consumption app is the same as a single zone Flex Consumption app.
- When you enable availability zones in an app with always-ready instance configuration of fewer than two instances for each [per-function scaling function or group](/azure/azure-functions/flex-consumption-plan#per-function-scaling), the platform automatically creates two instances of the [always-ready](/azure/azure-functions/flex-consumption-plan#always-ready-instances) type for each per-function scaling function or group. These new instances are also billed as always-ready instances.

**Premium plans:**
- There's no extra cost associated with enabling availability zones. Pricing for a zone-redundant Premium App Service plan is the same as a single zone Premium plan.
- If you enable availability zones on a plan with fewer than two instances, the platform enforces a minimum instance count of two for that App Service plan, and you're charged for both instances.

For step-by-step configuration instructions, see [Configure availability zones for Azure Functions](/azure/azure-functions/functions-availability-zones-configuration).
### Zone down experience

**Flex Consumption plans:**
All available function app instances of zone-redundant Flex Consumption plan apps are enabled and processing events. Flex Consumption apps continue to run even when other zones in the same region suffer an outage. However, it's possible that nonruntime behaviors might be impacted as a result of an outage in other availability zones. Standard function app behaviors that can affect availability include:

- Scaling
- App creation
- Configuration changes
- Deployments

Zone redundancy for Flex Consumption plans only guarantees continued uptime for deployed applications that are running.

When a zone goes down, Functions detects lost instances and automatically attempts to locate or create replacement instances, as needed, in the available zones. During zonal outage, the platform tries to restore balance on the available zones remaining.

**Premium plans:**
All available function app instances of zone-redundant function apps are enabled and processing events. When a zone goes down, Functions detect lost instances and automatically attempts to find new replacement instances if needed. Elastic scale behavior still applies. However, in a zone-down scenario there's no guarantee that requests for more instances can succeed, since back-filling lost instances occurs on a best-effort basis.

Applications that are deployed in an availability zone enabled Premium plan continue to run even when other zones in the same region suffer an outage. However, it's possible that nonruntime behaviors could still be impacted from an outage in other availability zones. These impacted behaviors can include Premium plan scaling, application creation, application configuration, and application publishing. Zone redundancy for Premium plans only guarantees continued uptime for deployed applications.

When Functions allocates instances to a zone redundant Premium plan, it uses best effort zone balancing offered by the underlying Azure Virtual Machine Scale Sets. A Premium plan is considered balanced when each zone has either the same number of virtual machines in all of the other zones used by the Premium plan, plus-or-minus one virtual machine.

## Cross-region disaster recovery and business continuity

[!INCLUDE [introduction to disaster recovery](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-disaster-recovery-description-include.md)]

This section explains some of the strategies that you can use to deploy a function app to allow for disaster recovery.

For disaster recovery for Durable Functions, see [Disaster recovery and geo-distribution in Azure Durable Functions](/azure/azure-functions/durable/durable-functions-disaster-recovery-geo-distribution).

### Multi-region disaster recovery

Because there's no built-in redundancy available, functions run in a function app in a specific Azure region. To avoid loss of execution during outages, you can redundantly deploy the same functions to function apps in multiple regions. To learn more about multi-region deployments, see the guidance in [Highly available multi-region web application](/azure/architecture/reference-architectures/app-service-web-app/multi-region).
 
When you run the same function code in multiple regions, there are two patterns to consider, [active-active](#active-active-pattern-for-http-trigger-functions) and [active-passive](#active-passive-pattern-for-non-https-trigger-functions).

#### Active-active pattern for HTTP trigger functions

With an active-active pattern, functions in both regions are actively running and processing events, either in a duplicate manner or in rotation. You should use an active-active pattern in combination with [Azure Front Door](/azure/frontdoor/front-door-overview) for your critical HTTP triggered functions, which can route and round-robin HTTP requests between functions running in multiple regions. Front door can also periodically check the health of each endpoint. When a function in one region stops responding to health checks, Azure Front Door takes it out of rotation, and only forwards traffic to the remaining healthy functions.  

![Architecture for Azure Front Door and Functions.](/azure/azure-functions/media/functions-geo-dr/front-door.png)  

### Active-passive pattern for non-HTTPS trigger functions

It's recommended that you use active-passive pattern for your event-driven, non-HTTP triggered functions, such as Service Bus and Event Hubs triggered functions.

To create redundancy for non-HTTP trigger functions, use an active-passive pattern. With an active-passive pattern, functions run actively in the region that's receiving events; while the same functions in a second region remain idle. The active-passive pattern provides a way for only a single function to process each message while providing a mechanism to fail over to the secondary region in a disaster. Function apps work with the failover behaviors of the partner services, such as [Azure Service Bus geo-recovery](/azure/service-bus-messaging/service-bus-geo-dr) and [Azure Event Hubs geo-recovery](/azure/event-hubs/event-hubs-geo-dr). 

Consider an example topology using an Azure Event Hubs trigger. In this case, the active/passive pattern requires involve the following components:

* Azure Event Hubs deployed to both a primary and secondary region.
* [Geo-disaster enabled](/azure/service-bus-messaging/service-bus-geo-dr) to pair the primary and secondary event hubs. This also creates an _alias_ you can use to connect to event hubs and switch from primary to secondary without changing the connection info.
* Function apps are deployed to both the primary and secondary (failover) region, with the app in the secondary region essentially being idle because messages aren't being sent there.
* Function app triggers on the *direct* (nonalias) connection string for its respective event hub. 
* Publishers to the event hub should publish to the alias connection string. 

![Active-passive example architecture.](/azure/azure-functions/media/functions-geo-dr/active-passive.png)

Before failover, publishers sending to the shared alias route to the primary event hub. The primary function app is listening exclusively to the primary event hub. The secondary function app is passive and idle. As soon as failover is initiated, publishers sending to the shared alias are routed to the secondary event hub. The secondary function app now becomes active and starts triggering automatically. Effective failover to a secondary region can be driven entirely from the event hub, with the functions becoming active only when the respective event hub is active.

Read more on information and considerations for failover with [Service Bus](/azure/service-bus-messaging/service-bus-geo-dr) and [Event Hubs](/azure/event-hubs/event-hubs-geo-dr).

### Active-active pattern for non-HTTPS trigger functions

While you're encouraged to use the [active-passive pattern](#active-passive-pattern-for-non-https-trigger-functions) for non-HTTPS trigger functions, you can still create active-active deployments for non-HTTP triggered functions. Before you implement this pattern, you must consider how the two active regions interact or coordinate with one another and the trigger source. 

For example, consider having the same Service Bus triggered function code deployed to two regions but triggering on the same Service Bus queue. In this case, both functions act as competing consumers on dequeueing the single queue. While each message can only be processed by one of the two app instances, it also means there's still a single point of failure, which is the single Service Bus instance. Consider enabling the [geo-disaster recovery](/azure/service-bus-messaging/service-bus-geo-dr) and [geo-replication](/azure/service-bus-messaging/service-bus-geo-replication) features of Service Bus to ensure it is also resilient.

## Next steps

- [Configure availability zones for Azure Functions](/azure/azure-functions/functions-availability-zones-configuration)
- [Disaster recovery and geo-distribution in Azure Durable Functions](/azure/azure-functions/durable/durable-functions-disaster-recovery-geo-distribution)
- [Create Azure Front Door](/azure/frontdoor/quickstart-create-front-door)
- [Event Hubs failover considerations](/azure/event-hubs/event-hubs-geo-dr#considerations)
- [Azure Architecture Center's guide on availability zones](/azure/architecture/high-availability/building-solutions-for-high-availability)
- [Reliability in Azure](./overview.md)
