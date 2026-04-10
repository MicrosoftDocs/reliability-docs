---
title: Reliability in Azure App Configuration
description: Learn how to make Azure App Configuration resilient to a variety of potential outages and problems, including transient faults, availability zone outages, region outages, and service maintenance, and learn about backup and restore.
author: maud-lv
ms.author: malev
ms.topic: reliability-article
ms.custom: subject-reliability
ai.usage: ai-assisted
ms.service: azure-app-configuration
ms.date: 02/23/2026
---

# Reliability in Azure App Configuration

[Azure App Configuration](/azure/azure-app-configuration/overview) centrally stores and manages application configuration settings and feature flags, which replaces configuration files embedded directly within applications. With this approach, you can dynamically update configuration values, track version history, and maintain a record of configuration changes over time. The availability and reliability of App Configuration are important considerations because application behavior can directly depend on access to configuration data at runtime.

[!INCLUDE [Shared responsibility](includes/reliability-shared-responsibility-include.md)]

This article describes the reliability architecture of App Configuration and how the service is designed to remain available during transient faults, availability zone failures, and region outages.

## Production deployment recommendations

For most production deployments of App Configuration, consider the following recommendations:

> [!div class="checklist"]
>
> - **SKU:** Use the Standard or Premium SKU.
>
> - **Soft delete and purge protection:** Turn on soft delete and purge protection to help prevent data deletion.
>
> - **For mission-critical scenarios:** Use the Premium SKU and configure the included replica to replicate across multiple regions. This approach improves high availability and resilience to region outages.

For a list of recommended practices and configuration for production workloads, see [Build applications that have high resiliency](/azure/azure-app-configuration/howto-best-practices#building-applications-with-high-resiliency).

## Reliability architecture overview

When you deploy App Configuration, you deploy a *store*. Your store contains various types of settings that your application might use, including [keys and values](/azure/azure-app-configuration/concept-key-value), and [feature flags](/azure/azure-app-configuration/concept-feature-management). The service also includes built-in capabilities for organizing, securing, versioning, and safely rolling out configuration changes across environments. For more information, see [What is App Configuration?](/azure/azure-app-configuration/overview)

App Configuration is a fully managed service. Microsoft is responsible for performing maintenance on the service and storing and managing your settings.

When you build client applications that connect to App Configuration, you can optionally [use App Configuration with Azure Front Door (preview)](/azure/azure-app-configuration/concept-hyperscale-client-configuration) to enable caching and global traffic acceleration. This configuration introduces other considerations for geo-replication, which are highlighted throughout this article where appropriate.

## Resilience to transient faults

[!INCLUDE [Transient fault description](includes/reliability-transient-fault-description-include.md)]

When you use App Configuration, consider the following best practices to minimize the effect of transient faults on configuration access, especially within critical code paths.

- **Configuration providers:** Use [App Configuration providers](/azure/azure-app-configuration/configuration-provider-overview), which have built-in retry and caching capabilities and other resiliency features.

- **Azure SDKs:** Use App Configuration SDKs if your application needs to send write requests. SDKs automatically retry on HTTP status code 429 responses and other transient errors.

- **Retry logic:** Include retry logic in custom clients if you can't use App Configuration providers or SDKs. The `retry-after-ms` header in the response provides a suggested wait time in milliseconds before the client retries the request.

- **Caching:** Cache settings in memory when possible to reduce direct requests to your store.

For other application configuration guidance, see [App Configuration FAQ](/azure/azure-app-configuration/faq#my-application-receives-http-status-code-429-responses--why).

## Resilience to availability zone failures

[!INCLUDE [Resilience to availability zone failures](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)]

App Configuration automatically provides zone redundancy in [regions that support availability zones](./regions-list.md). This redundancy provides high availability within a region without requiring any specific configuration.

:::image type="complex" border="false" source="media/reliability-app-configuration/zone-redundant.svg" alt-text="Diagram that shows a zone-redundant App Configuration store that spans three zones in the region.":::
   The diagram shows availability zones 1, 2, and 3. The App Configuration store spans all three zones in the region.
:::image-end:::

When an availability zone becomes unavailable, App Configuration automatically redirects your requests to other healthy availability zones to ensure high availability.

### Requirements

**Region support:** Stores deployed into the following regions are automatically zone redundant.

[!INCLUDE [Azure App Configuration availability zones table](~/reusable-content/ce-skilling/azure/includes/azure-app-configuration-availability-zones.md)]

### Cost

There's no extra cost for zone redundancy for App Configuration.

### Configure availability zone support

Microsoft automatically configures zone redundancy for a store when it's in [a region that supports availability zones](#requirements).

If App Configuration adds availability zone support to an existing region, you don't need to take any action to benefit from the availability zone support. Your store benefits from the availability zone support that becomes available for App Configuration stores in the region.

### Behavior when all zones are healthy

This section describes what to expect when you configure an App Configuration store for zone redundancy, and all zones are operational.

- **Cross-zone operation:** App Configuration automatically manages traffic routing between availability zones. During normal operations, it transparently distributes requests across zones.

- **Cross-zone data replication:** In regions that support zones, App Configuration synchronously replicates data across availability zones. This replication ensures that your settings remain consistent and available even if a zone becomes unavailable.

### Behavior during a zone failure

This section describes what to expect when you configure an App Configuration store for zone redundancy, and there's an outage in one of the zones.

- **Detection and response:** The App Configuration service detects zone failures and automatically responds to them. You don't need to take any action during a zone failure.

[!INCLUDE [Availability zone down notification (Service Health only)](./includes/reliability-availability-zone-down-notification-service-include.md)]

- **Active requests:** During a zone failure, the affected zone might fail to handle in-flight requests, which requires client applications to retry them. Client applications should follow [transient fault handling practices](#resilience-to-transient-faults) to ensure that they can retry requests if a zone failure occurs.

- **Expected data loss:** No data loss is expected during a zone failure because of the synchronous replication between zones.

- **Expected downtime:** No downtime is expected.

- **Redistribution:** App Configuration automatically reroutes traffic away from the affected zone to healthy zones without requiring any customer intervention.

### Zone recovery

When a previously unavailable zone recovers, App Configuration automatically restores normal operations across all availability zones. You don't need to take any action to recover from a zone failure.

### Test for zone failures

The App Configuration platform manages traffic routing, failover, and zone recovery for zone-redundant stores. Microsoft fully manages this process, so you don't need to validate availability zone failure processes.

## Resilience to region-wide failures

App Configuration provides native geo-replication capabilities to support resilience during region outages. Geo-replication allows configuration data to replicate across regions as a managed service feature.

### Geo-replication

With geo-replication, you can replicate a store across multiple Azure regions. Each store can have multiple *replicas* in different regions. The original store is also a replica. This capability helps protect applications from region-wide disruptions.

#### Requirements

- **Region support:** You can create replicas in any Azure region that App Configuration supports, even if the regions aren't Azure paired regions.

- **Tier:** The configuration store must use a supported tier to enable geo-replication. For more information, see [Enable geo-replication](/azure/azure-app-configuration/howto-geo-replication).

#### Considerations

When you enable geo-replication, consider the following factors:

- **Zone-redundant replicas:** Any replica that you create in a region in which App Configuration supports availability zones is automatically zone redundant.

- **Azure Front Door:** To enable geo-redundant configuration delivery with Azure Front Door, configure App Configuration replicas as origins within an origin group. Correctly configured origins are required for Azure Front Door to perform health-based routing, load balancing, and automatic failover across regions. For more information, see [Traffic routing methods to origin](/azure/frontdoor/routing-methods).

#### Cost

Each geo-replicated region is billed separately according to the pricing for the respective tier and region. No data egress charges apply for cross-region replication. For pricing details, see [App Configuration pricing](https://azure.microsoft.com/pricing/details/app-configuration/).

#### Configure multi-region support

To set up replication for a newly created configuration store, see [Enable geo-replication](/azure/azure-app-configuration/howto-geo-replication).

#### Behavior when all regions are healthy

This section describes what to expect when you configure an App Configuration store for geo-replication, and all regions are operational.

- **Cross-zone operation:** Each replica is individually addressable and has its own Domain Name System (DNS) name. All replicas can accept both read and write operations.
    
    App Configuration doesn't automatically route traffic between regions. When you use [App Configuration configuration providers](/azure/azure-app-configuration/configuration-provider-overview), your application can optionally use automatic replica discovery. Alternatively, you can specify a prioritized list of replicas, and App Configuration selects the first healthy replica. This approach enables your application to control which replica it uses.

    > [!NOTE]
    > If you use Azure Front Door, traffic routing behavior is different. For more information, see [Failover and load balancing](/azure/azure-app-configuration/concept-hyperscale-client-configuration#failover-and-load-balancing).

- **Cross-zone data replication:** Data replicates asynchronously and is eventually consistent. You can use the [replication latency metric in Azure Monitor](/azure/azure-app-configuration/concept-geo-replication#monitoring) to monitor the current replication latency between replicas.

#### Behavior during a region failure

This section describes what to expect when you configure an App Configuration store for geo-replication, and there's an outage in one of the replica regions.

- **Detection and response:** Microsoft is responsible for detecting region or replica failures and initiating recovery processes.

    When you use [App Configuration configuration providers](/azure/azure-app-configuration/configuration-provider-overview) and perform [automatic replica discovery](/azure/azure-app-configuration/howto-geo-replication#automatic-replica-discovery) or with a list of multiple replicas, your application automatically fails over to another healthy replica.

    If you don't use App Configuration providers, you're responsible for switching your application to a healthy replica.

- **Notification:** [!INCLUDE [Region down notification partial bullet (Azure Service Health only)](./includes/reliability-region-down-notification-service-partial-include.md)]

- **Active requests:** Active requests against a replica in the region might fail. Client applications should retry the requests against a different replica.

- **Expected data loss:** If a replica fails, recent changes made on that replica might not yet be replicated to other replicas. Those changes can remain unavailable until the replica recovers. To estimate potential data loss, monitor the [replication latency metric in Azure Monitor](/azure/azure-app-configuration/concept-geo-replication#monitoring).

- **Expected downtime:** When a replica becomes unavailable, it remains offline until its region recovers. Other replicas continue to handle requests. Applications might experience brief downtime while they detect the failure and switch to a healthy replica. The duration depends on how quickly each application performs this detection and failover.

- **Redistribution:** Applications must route traffic to a healthy replica when a failure occurs.

    If you use App Configuration configuration providers, the providers automatically handle replica selection and failover.
    
    If you place Azure Front Door in front of your data store and [configure the origin group with multiple replicas as origins for failover](/azure/frontdoor/routing-methods), Azure Front Door automatically reroutes requests to a healthy replica.

#### Region recovery

After the region recovers, App Configuration syncs the replica with the other replicas without your intervention.

You're responsible for reconfiguring your application to route traffic back to the recovered region instance. Applications that use App Configuration providers automatically start to use the replica again.

#### Test for region failures

You can't directly simulate a replica failover in App Configuration. However, because applications control replica selection, you can test failover behavior by forcing the application into a state where it must switch replicas.

To validate your application's replica failover behavior, you can introduce a controlled connectivity failure in a nonproduction environment and observe how the application responds.

One approach is to use your local machine or another environment where you have administrative access. Follow these steps:

1. Turn on verbose logging for the Azure SDK. In .NET, use the `AzureEventSourceListener` class to configure a logger. For more information, see [Logging and monitoring](/azure/azure-app-configuration/enable-dynamic-configuration-dotnet-core#logging-and-monitoring).

1. Manually configure your `hosts` file so that requests to your App Configuration store are routed to an IP address that can't receive them, like `127.0.0.1` (localhost).

    > [!WARNING]
    > This step effectively blocks access from your computer to your App Configuration store. Only follow these steps in a nonproduction environment.

1. Monitor the logs for a message similar to the following example:

    ```console
    [Warning] Microsoft-Extensions-Configuration-AzureAppConfiguration-Refresh:
    Failed to get configuration settings from endpoint 'https://myappconfigstore.azconfig.io'. Failing over to endpoint https://myappconfigstore-eus.azconfig.io'.
    ```

    This message indicates that the application successfully failed over to use another replica of your store.

1. After you complete the test, undo the changes to your `hosts` file.

## Backup and restore

You can use App Configuration to [export configuration data](/azure/azure-app-configuration/howto-import-export-data) from a store and use it as part of a broader backup strategy.

[!INCLUDE [Backups description](includes/reliability-backups-include.md)]

## Resilience to accidental deletion

App Configuration provides two key recovery features to prevent accidental or malicious deletion:

- **Soft delete:** When you turn on soft delete, you can recover deleted stores and settings during a configurable retention period. Soft delete functions like a recycle bin for your App Configuration resources.

- **Purge protection:** When you turn on purge protection, the service prevents permanent deletion of your store and its settings until the retention period elapses. This safeguard prevents malicious actors from permanently destroying your settings.

Use both features for production environments. For more information, see [Soft-delete and purge protection](/azure/azure-app-configuration/concept-soft-delete).

## Resilience to service maintenance

Microsoft regularly performs service updates and other maintenance. The service handles these activities automatically, which makes maintenance simple and transparent to you. No downtime is expected during maintenance events unless [Azure Service Health](/azure/service-health/service-health-planned-maintenance) provides a planned maintenance notice.

## Resilience to configuration problems

Incorrect or accidental configuration changes can cause application downtime. Use [configuration snapshots](/azure/azure-app-configuration/concept-snapshots) to safely roll out changes to configuration. Monitor your application health after any configuration changes, and revert to the last-known-good configuration snapshot if the changes introduce a problem.

## Service-level agreement

[!INCLUDE [Service-level agreement](includes/reliability-service-level-agreement-include.md)]

## Related content

- [Reliability in Azure](./overview.md)
- [App Configuration documentation](/azure/azure-app-configuration/)
