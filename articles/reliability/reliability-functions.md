---
title: Reliability in Azure Functions
description: Learn how to ensure serverless reliability with Azure Functions by using Azure availability zones, SKUs, and cross-region disaster recovery strategies.
author: ggailey777
ms.author: glenga
ms.topic: reliability-article
ms.service: azure-functions
ms.custom: references_regions, subject-reliability
ai.usage: ai-assisted
ms.date: 03/12/2026
zone_pivot_groups: azure-functions-hosting-plans
---

# Reliability in Azure Functions

[Azure Functions](/azure/azure-functions/functions-overview) is an event-driven compute service that runs small blocks of code, or *functions*, without explicit infrastructure provisioning or management. Functions can respond to events such as HTTP requests, timers, queue messages, and changes in other Azure services. This capability makes Functions well-suited for data processing, system integration, and background tasks.

[!INCLUDE [Shared responsibility](includes/reliability-shared-responsibility-include.md)]

This article describes how to make Functions resilient to various potential outages and problems, including transient faults, availability zone failures, and region-wide failures. It also highlights key information about the Functions service-level agreement (SLA).

## Production deployment recommendations

The Azure Well-Architected Framework provides recommendations across reliability, security, cost, operations, and performance. To learn how these areas influence each other and contribute to a reliable Functions solution, see [Architecture best practices for Functions](/azure/well-architected/service-guides/azure-functions).

## Reliability architecture overview

When you deploy Functions, it's important to familiarize yourself with these concepts:

- **[Hosting plans](/azure/azure-functions/functions-scale):** Plans represent the hosting environment for your function apps. The plan determines the available compute resources, pricing model, and scaling behavior.

- **[Storage accounts](/azure/azure-functions/storage-considerations):** When you create a function app, you must specify a host storage account. The storage account manages aspects of the function app's internal operations, including function code storage, logging, and concurrency management (such as blob leases for specific trigger types).

    You can also use a storage account for deployment. This storage account might be the same as your host storage account or a different storage account.

    > [!IMPORTANT]
    > Storage accounts are a critical part of your Functions reliability architecture. Configure them to meet your function app's resiliency requirements.

- **[Triggers and bindings](/azure/azure-functions/functions-triggers-bindings):** Triggers and bindings let your function respond to events, receive data from other services, and write data to other services.

- **[Durable functions](/azure/azure-functions/durable/durable-functions-overview):** Durable functions is a feature of Functions. It provides stateful functions such as long-running orchestrations and stateful entities.

    When you use durable functions, you configure a [storage provider](/azure/azure-functions/durable/durable-functions-storage-providers) that stores the state. Evaluate the reliability characteristics of the state store that you choose and configure it to meet your resiliency requirements.

## Resilience to transient faults

[!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)]

Consider the following recommendations for handling transient faults in your function apps:

- **Triggers and bindings:** The Functions platform includes built-in transient fault handling for triggers and bindings. When a transient fault occurs while a supported trigger fires or a supported binding reads or writes data, the platform can automatically retry the operation. This built-in retry behavior ensures that temporary connectivity problems or service blips don't prevent your function from running. For more information, see [Functions error handling and retries](/azure/azure-functions/functions-bindings-error-pages#retries).

    This protection only covers transient faults. The platform doesn't retry persistent failures, such as a misconfigured connection string or a deleted resource.

    The platform treats persistent failures and repeated transient failures as errors. Configure logging to capture information about function execution errors. For more information, see [Configure monitoring for Functions](/azure/azure-functions/configure-monitoring).

- **Your function code:** In your function code, you're responsible for handling transient faults when your function calls external services. Implement retry logic, timeouts, and circuit breaker patterns as appropriate for external service calls made in your function code. Design your functions to be idempotent where possible so that retries don't create duplicate side effects.

- **Clients:** Client applications that connect to functions synchronously, such as through an HTTP connection, should be resilient to transient faults.

## Resilience to availability zone failures

[!INCLUDE [Resilience to availability zone failures](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)]

::: zone pivot="consumption"

Consumption plans don't support availability zones. If zone redundancy is a requirement for your workload, consider using the Flex Consumption, Premium, or Dedicated (App Service) plans instead.

::: zone-end

::: zone pivot="flex-consumption"

Flex Consumption plans support zone-redundant deployments.

::: zone-end

::: zone pivot="premium"

Premium plans support zone-redundant deployments.

::: zone-end

::: zone pivot="flex-consumption,premium"

When zone redundancy is enabled, the platform automatically spreads your plan instances across all availability zones in the selected region. If any availability zone in the region has a problem, your functions continue to run by using instances in healthy zones.

You must enable zone-redundant storage (ZRS) on the host storage account, which ensures that it's also resilient to zone outages.

:::image type="complex" border="false" source="./media/reliability-functions/zone-redundant.svg" alt-text="Diagram that shows a zone-redundant Functions plan that has three instances spread across three zones and a ZRS account.":::
   The diagram shows three availability zones. Each zone contains a Functions instance. A ZRS account spans all three availability zones.
:::image-end:::

::: zone-end

::: zone pivot="dedicated"

The Dedicated (App Service) plan supports zone-redundant deployments. When zone redundancy is enabled, the platform automatically spreads your instances across all availability zones in the selected region. You configure zone redundancy on the plan. For more information about how Azure App Service handles zone redundancy, see [Reliability in App Service](reliability-app-service.md).

::: zone-end

::: zone pivot="flex-consumption,premium"

Your plan is *nonzonal* or *regional* when you don't enable zone redundancy. The platform can place plan instances in any availability zone in the region or in the same zone. Plan instances aren't resilient to availability zone failures. Your plan might experience downtime during an outage in any zone in the region.

### Requirements

::: zone-end

::: zone pivot="flex-consumption"

- **Region support:** You can deploy zone-redundant Flex Consumption plans into a specific set of regions. You can retrieve the current list of supported regions by using the Azure CLI. For more information, see [View regions that support availability zones](/azure/azure-functions/functions-zone-redundancy?pivots=flex-consumption-plan##view-regions-that-support-availability-zones).

::: zone-end

::: zone pivot="premium"

- **Region support:** You can deploy zone-redundant Premium plans into the following regions.

    | Americas         | Europe               | Middle East    | Africa             | Asia Pacific   |
    |------------------|----------------------|----------------|--------------------|----------------|
    | Brazil South     | France Central       | Israel Central | South Africa North | Australia East |
    | Canada Central   | Germany West Central | Qatar Central  |                    | Central India  |
    | Central US       | Italy North          | UAE North      |                    | China North 3  |
    | East US          | North Europe         |                |                    | East Asia      |
    | East US 2        | Norway East          |                |                    | Japan East     |
    | South Central US | Sweden Central       |                |                    | Southeast Asia |
    | West US 2        | Switzerland North    |                |                    |                |
    | West US 3        | UK South             |                |                    |                |
    |                  | West Europe          |                |                    |                |

- **Operating systems:** The platform supports both Windows and Linux plans.

- **Minimum instance count:** Zone redundancy for Premium plans requires a minimum of two always-ready instances.

::: zone-end

::: zone pivot="flex-consumption,premium"

- **Host storage account:** You must configure your function app's default host storage account to use [ZRS](/azure/storage/common/storage-redundancy#zone-redundant-storage). If you use a host storage account that's not configured for ZRS, your app might behave unexpectedly during a zone outage.

::: zone-end

::: zone pivot="flex-consumption"

- **Deployment container storage account:** If you use a separate storage account for the app's deployment container, also update it to be zone redundant.

::: zone-end

::: zone pivot="flex-consumption,premium"

### Considerations

Zone redundancy only guarantees continued uptime for deployed applications. An availability zone outage might affect aspects of Functions, even though the application continues to serve traffic. These behaviors include plan scaling, application creation, application configuration, and application publishing.

### Instance distribution across zones

::: zone-end

::: zone pivot="flex-consumption"

When you configure Flex Consumption plan apps as zone redundant, the platform automatically spreads plan instances across multiple zones in the selected region, with different rules for always-ready versus on-demand instances:

- **Always-ready instances** are distributed across at least two zones by using round-robin distribution.

    To ensure zone resiliency, the platform automatically maintains at least two always-ready instances for each [per-function scaling function or group](/azure/azure-functions/flex-consumption-plan#per-function-scaling), regardless of the always-ready configuration for the app. Instances that the platform creates are platform-managed, billed as always-ready instances, and don't change the always-ready configuration settings.

- **On-demand instances** result from event source volumes as the app scales beyond the always-ready instance count. On-demand instances are distributed across availability zones on a best-effort basis. The platform prioritizes faster scale-out over even distribution across zones. The platform attempts to even out the distribution over time.

::: zone-end

::: zone pivot="premium"

When you configure Elastic Premium function app plans as zone redundant, the platform automatically spreads plan instances across multiple zones in the selected region. Instance spreading follows these rules, even as the app scales in and out:

- The minimum function app instance count is two.

- When you specify a capacity larger than the number of zones, instances are spread evenly only when the capacity is a multiple of the number of zones.

- For a capacity value more than Number of Zones *Number of instances, extra instances are spread across the remaining zones.

When Functions allocates instances to a zone-redundant Premium plan, it uses [best-effort zone balancing](/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-zone-balancing), which the underlying Azure Virtual Machine Scale Sets provides. Azure considers a Premium plan *balanced* when each zone has the same number of virtual machines (VMs) as the other zones in the plan, plus or minus one VM.

::: zone-end

::: zone pivot="flex-consumption,premium"

### Cost

::: zone-end

::: zone pivot="flex-consumption,premium"

You incur no extra cost when you enable zone redundancy. Pricing for a zone-redundant plan is the same as a single-zone plan.

::: zone-end

::: zone pivot="flex-consumption"

When you enable availability zones in an app with an always-ready instance configuration of fewer than two instances for each [per-function scaling function or group](/azure/azure-functions/flex-consumption-plan#per-function-scaling), the platform automatically creates two instances of the [always-ready type](/azure/azure-functions/flex-consumption-plan#always-ready-instances) for each per-function scaling function or group. These new instances are also billed as always-ready instances.

::: zone-end

::: zone pivot="premium"

If you enable availability zones on a plan that have fewer than two instances, the platform enforces a minimum instance count of two for that plan, and you're charged for both instances.

::: zone-end

::: zone pivot="flex-consumption,premium"

For full pricing details, see [Functions pricing](https://azure.microsoft.com/pricing/details/functions/).

### Configure availability zone support

::: zone-end

::: zone pivot="flex-consumption"

- **Create a new zone-redundant Functions plan.** You can enable zone redundancy when you create a new plan. For more information, see [Create a zone-redundant Function App](/azure/azure-functions/functions-zone-redundancy?pivots=flex-consumption-plan#create-a-zone-redundant-function-app).

- **Enable zone redundancy on an existing plan:** You can update an existing Flex Consumption plan to enable zone redundancy. For more information, see [Enable zone redundancy on an existing plan](/azure/azure-functions/functions-zone-redundancy?pivots=flex-consumption-plan#enable-zone-redundancy-on-an-existing-plan).

::: zone-end

::: zone pivot="premium"

- **Create a new zone-redundant Functions plan.** You can enable zone redundancy when you create a new plan. For more information, see [Create a zone-redundant Function App](/azure/azure-functions/functions-zone-redundancy?pivots=premium-plan#create-a-zone-redundant-function-app).

- **Enable zone redundancy on an existing plan:** For Premium plans, you can only enable zone redundancy during plan creation. You can't convert an existing Premium plan to be zone redundant. Instead, migrate your app by creating a side-by-side deployment on a new Premium plan app. For more information, see [Enable zone redundancy on an existing plan](/azure/azure-functions/functions-zone-redundancy?pivots=premium-plan#enable-zone-redundancy-on-an-existing-plan).

::: zone-end

::: zone pivot="flex-consumption,premium"

### Capacity planning and management

Zone-redundant function apps continue to run even when zones in the region experience an outage.

During a zone outage, Functions detects lost instances and automatically tries to locate or create replacement instances in the healthy zones. The platform performs this process on a best-effort basis and doesn't guarantee success. If your workload requires a specific number of instances to maintain your expected service level, consider *overprovisioning* the number of always-ready instances. Overprovisioning lets the solution tolerate capacity loss and continue to function without reduced performance. For more information, see [Manage capacity by overprovisioning](/azure/reliability/concept-redundancy-replication-backup#manage-capacity-with-over-provisioning).

### Behavior when all zones are healthy

This section describes what to expect when a plan is zone redundant, the host storage account uses ZRS, and all availability zones are operational.

- **Cross-zone operation:** When you configure zone redundancy on Functions, requests are automatically spread across the instances in each availability zone. A request might go to any instance in any availability zone.

- **Cross-zone data replication:** Functions is a stateless compute service, so there's no data to replicate between zones. The platform replicates configuration across zones automatically.

    If your host storage account uses ZRS, Azure Storage synchronously replicates its data across multiple availability zones.

    For durable functions, review your storage provider to learn how it replicates data across zones.

### Behavior during a zone failure

This section describes what to expect when a plan is zone redundant, the host storage account uses ZRS, and there's an availability zone outage.

- **Detection and response:** The Functions platform is responsible for detecting a failure in an availability zone. You don't need to initiate a zone failover.

[!INCLUDE [Availability zone down notification (Service Health and Resource Health)](includes/reliability-availability-zone-down-notification-service-resource-include.md)]

- **Active requests:** When an availability zone is unavailable, in-progress requests that connect to an instance in the faulty availability zone terminate and must be retried. Ensure that your applications are prepared by following the [transient fault handling guidance](#resilience-to-transient-faults).

- **Expected data loss:** Zone failures aren't expected to cause data loss because Functions is a stateless service.

    If your host storage account uses ZRS, Storage ensures no data loss from a zone failure.

    For durable functions, review your storage provider to learn whether data loss is possible during a zone failure.

- **Expected downtime:** During zone outages, connections might experience brief interruptions that typically last a few seconds as traffic is redistributed. Ensure that your applications are prepared by following the [transient fault handling guidance](#resilience-to-transient-faults).

- **Traffic rerouting:** Functions detects the lost instances from that zone and attempts to find new replacement instances. After Functions finds replacements, it distributes traffic across the new instances as needed.

    > [!IMPORTANT]
    > Azure doesn't guarantee that requests for more instances succeed in a zone-down scenario. The platform attempts to backfill lost instances on a best-effort basis. If you need guaranteed capacity during an availability zone failure, create and configure your plans to account for zone loss by overprovisioning the capacity.

- **Nonruntime behaviors:** Applications in a zone-redundant function app plan continue to run and serve traffic even if an availability zone experiences an outage. However, nonruntime behaviors might be affected during an availability zone outage. These behaviors include function app scaling, application creation, application configuration, and application publishing.

### Zone recovery

When the availability zone recovers, Functions automatically restores instances in the availability zone, removes temporary instances created in the other availability zones, and reroutes traffic between your instances as normal.

### Test for zone failures

The Functions platform manages traffic routing, failover, and zone recovery for zone-redundant resources. You don't need to initiate anything. Azure fully manages this feature, so you don't need to validate availability zone failure processes.

::: zone-end

## Resilience to region-wide failures

Functions is a single-region service. If the region becomes unavailable, your Functions resource is also unavailable.

### Custom multi-region solutions for resiliency

To avoid interrupted executions during outages, you can redundantly deploy the same functions to function apps in multiple regions.

You're responsible for:

- Deploying function apps to multiple regions.

- Managing traffic distribution between regions.

- Implementing failover mechanisms.

- Ensuring data consistency across regions (if applicable).

- Monitoring and managing cross-region deployments.

When you run the same function code in multiple regions, consider the active-active and active-passive patterns. The following sections introduce these patterns but don't provide detailed guidance or configuration steps.

#### Active-active pattern for HTTP trigger functions

In an active-active pattern, functions in both regions actively run and process events, either in a duplicate manner or in rotation. Use an active-active pattern in combination with [Azure Front Door](/azure/frontdoor/front-door-overview) for your critical HTTP-triggered functions, which can route and round-robin HTTP requests between functions that run in multiple regions. Azure Front Door can also periodically check the health of each endpoint. If a function in one region stops responding to health checks, Azure Front Door removes it from rotation and only forwards traffic to the remaining healthy functions.

:::image type="complex" border="false" source="./media/reliability-functions/active-active.svg" alt-text="Diagram that shows an example active-active architecture. Azure Front Door routes between Functions apps in different regions that each have their own database.":::
   The diagram shows Azure Front Door at the top. Two regions appear below: the primary region on the left and the secondary region on the right. Each region contains a Functions app and a database. Arrows point from Azure Front Door to both function apps. An arrow points from each function app to its respective database.
:::image-end:::

#### Active-passive pattern for non-HTTP trigger functions

For event-driven, non-HTTP-triggered functions (such as Service Bus and Event Hubs triggers), use an active-passive pattern. In an active-passive pattern, function instances run in the region that receives events, while the instances in the secondary region remain idle. This pattern ensures that only one function processes each message, which helps maintain data consistency. It also provides a way to fail over to the secondary region during a disaster such as a region outage.

Consider Function app failover with the failover behaviors of other services such as:

- [Azure Service Bus geo-replication and geo-disaster recovery](./reliability-service-bus.md#resilience-to-region-wide-failures)
- [Azure Event Hubs geo-replication and geo-disaster recovery](./reliability-event-hubs.md#resilience-to-region-wide-failures)

Consider an example topology that uses an Event Hubs trigger, where your Event Hubs namespace is configured for geo-disaster recovery. In this case, the active-passive pattern requires the following components:

- Event Hubs deployed to both a primary and secondary region.

- [Geo-disaster recovery enabled](/azure/service-bus-messaging/service-bus-geo-dr) to pair the primary and secondary event hubs. This configuration also creates an *alias* that you can use to connect to the Event Hubs namespace, and switch the alias from the primary to the secondary without changes to the connection info.

- Function apps deployed to both the primary and secondary region. The app in the secondary region remains idle because it doesn't receive messages.

- Function app triggers on the *direct* (nonalias) connection string for its respective Event Hubs namespace.

- Publishers to the Event Hubs namespace publish to the alias connection string.

:::image type="complex" border="false" source="./media/reliability-functions/active-passive.svg" alt-text="Diagram that shows an example active-passive architecture. Event Hubs geo-disaster recovery spans multiple regions and separate function apps and databases in each region.":::
   The diagram shows a primary region on the left and a secondary region on the right. The primary region contains an active Event Hubs namespace, a Functions app, and a database. The secondary region contains a passive Event Hubs namespace, a Functions app, and a database. An arrow points from the alias to the Events Hubs geo-disaster recovery. That arrow connects the primary and secondary Event Hubs namespaces. Arrows point from each event hub to its respective function app. An arrow points from each function app to its respective database.
:::image-end:::

Before failover, publishers that send events to the shared alias route traffic to the primary event hub. The primary function app listens exclusively to the primary event hub. The secondary function app remains passive and idle.

When failover starts, publishers that send events to the shared alias route traffic to the secondary event hub. The secondary function app becomes active and starts to trigger automatically. The event hub drives the entire failover process, and each function app runs only when its corresponding event hub is active.

#### Durable functions

For multi-region disaster recovery for durable functions, see [Disaster recovery and geo-distribution in Azure durable functions](/azure/azure-functions/durable/durable-functions-disaster-recovery-geo-distribution).

## Resilience to service maintenance

Functions performs regular service upgrades and other maintenance tasks.

- **Transient fault resilience:** During service maintenance, the instances that run your function app might restart or experience temporary interruptions. Ensure that client applications that interact with your function app are [resilient to transient faults](#resilience-to-transient-faults).

::: zone pivot="flex-consumption"

- **Enable zone redundancy:** When you enable zone redundancy on your plan, you also improve resiliency during platform updates. Deploying multiple instances in your plan and enabling zone redundancy for your plan adds an extra layer of resiliency if an instance or a zone becomes unhealthy during an upgrade.

::: zone-end

::: zone pivot="premium,dedicated"

To maintain your expected capacity during an upgrade, the platform automatically adds extra instances of the plan during the upgrade process.

- **Enable zone redundancy:** When you enable zone redundancy on your plan, you also improve resiliency during platform updates. *Update domains* consist of collections of VMs that go offline during an update, and they map to availability zones. Deploying multiple instances in your plan and enabling zone redundancy for your plan adds an extra layer of resiliency if an instance or a zone becomes unhealthy during an upgrade.

::: zone-end

::: zone pivot="dedicated"

- **App Service Environment:** If you host your function app on an App Service Environment, you can customize the upgrade cycle. If you must validate the effect of upgrades on your workload, enable manual upgrades. This approach lets you validate and test on a nonproduction instance before you apply the upgrades to your production instance.

    For more information about maintenance preferences, see [Upgrade preferences for App Service Environment planned maintenance](/azure/app-service/environment/how-to-upgrade-preference).

::: zone-end

## Resilience to application deployments

Application deployments introduce the risk of problems in a production environment. Be ready to roll back an update if it causes problems. Control how updates roll out to reduce disruption from application restarts.

::: zone pivot="flex-consumption"

Flex Consumption plans support [site update strategies](/azure/azure-functions/flex-consumption-site-updates), which provide multiple ways to deploy your app updates, including rolling updates for zero-downtime deployments.

::: zone-end

::: zone pivot="consumption,premium,dedicated"

Functions [deployment slots](/azure/azure-functions/functions-deployment-slots) enable zero-downtime deployments of your function apps. Use deployment slots to minimize the effect of deployments and configuration changes for your users. Deployment slots also reduce the likelihood that your application restarts. Application restarts cause transient faults.

::: zone-end

## Service-level agreement

[!INCLUDE [Service-level agreement](includes/reliability-service-level-agreement-include.md)]

Functions provides distinct availability SLAs for the Consumption plan and for other plan types.

## Related content

- [Configure availability zones for Functions](/azure/azure-functions/functions-premium-plan#availability-zones)
- [Disaster recovery and geo-distribution in Azure durable functions](/azure/azure-functions/durable/durable-functions-disaster-recovery-geo-distribution)
- [Create Azure Front Door](/azure/frontdoor/quickstart-create-front-door)
- [Event Hubs failover considerations](/azure/event-hubs/event-hubs-geo-dr#considerations)
- [Architecture strategies for how to use availability zones and regions](/azure/architecture/high-availability/building-solutions-for-high-availability)
- [Reliability in Azure](./overview.md)
