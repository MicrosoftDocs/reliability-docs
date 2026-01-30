---
title: Reliability in Azure Application Configuration
description: Learn how to make Azure Application Configuration resilient to various potential outages and problems, including transient faults, availability zone outages, and region outages.
author: glynnniall
ms.author: pnp 
ms.topic: reliability-article 
ms.custom: subject-reliability
ai.usage: ai-assisted
ms.service: azure-app-configuration
ms.date: 1/29/2026
---

# Reliability in Azure Application Configuration

[Azure App Configuration](/azure/azure-app-configuration/overview) centrally stores and manages application configuration settings and feature flags, replacing configuration files embedded directly within applications. This approach enables dynamic updates, versioning of configuration values, and historical tracking of configuration changes over time. The availability and reliability of App Configuration are important considerations because application behavior can directly depend on access to configuration data at runtime.

[!INCLUDE [Shared responsibility](includes/reliability-shared-responsibility-include.md)]

This article describes the reliability architecture of Azure App Configuration and explains how the service is designed to remain available during transient faults, availability zone failures, and region outages.

### Production deployment recommendations for reliability

For a list of recommended practices and configuration for production workloads, see [Building applications with high resiliency](/azure/azure-app-configuration/howto-best-practices?tabs=dotnet#building-applications-with-high-resiliency).

## Reliability architecture overview

When you deploy App Configuration, you deploy a *store*.

Your store contains various types of settings that your application might use, including [keys and values](/azure/azure-app-configuration/concept-key-value), and [Feature flags](/azure/azure-app-configuration/concept-feature-management). The service provides other features for managing and organizing your settings. For more information, see [What is Azure App Configuration?](/azure/azure-app-configuration/overview)

App Configuration is a fully managed service. Microsoft is responsible for storing and managing your settings, as well as performing maintenance on the service.

When you build client applications, you can optionally use App Configuration with Azure Front Door to enable caching and global content delivery to clients. This configuration introduces other considerations for geo-replication, which are highlighted throughout this article where appropriate.

## Resilience to transient faults

[!INCLUDE [Transient fault description](includes/reliability-transient-fault-description-include.md)]

When you use Azure App Configuration, consider the following best practices to minimize the effect of transient faults on configuration access, especially within critical request paths.

- **Configuration providers:** Use the [Azure App Configuration provider libraries](/azure/azure-app-configuration/configuration-provider-overview), which have built-in retry and caching capabilities along with many other resiliency features.
- **SDKs:** Use App Configuration SDKs if your application needs to send write requests. Although the SDKs might not be as feature-rich as providers, they automatically retry on HTTP status code 429 responses and other transient errors.
- **Retry logic:** Include retry logic in custom clients if you can't use App Configuration Providers or SDKs. The `retry-after-ms` header in the response provides a suggested wait time in milliseconds before retrying the request.
- **Caching:** Cache settings in memory when possible to reduce direct requests to your store.

For other application configuration guidance, see [Azure App Configuration FAQ](/azure/azure-app-configuration/faq#my-application-receives-http-status-code-429-responses--why).


## Resilience to availability zone failures

[!INCLUDE [Resilience to availability zone failures](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)]

App Configuration automatically provides zone redundancy in [regions that support availability zones](./regions-list.md). This redundancy provides high availability within a region without requiring any specific configuration. 

When an availability zone becomes unavailable, App Configuration automatically redirects your requests to other healthy availability zones to ensure high availability.

### Requirements

- **Region support:** Stores deployed into the following regions are automatically zone-redundant:

    (Table from the legacy migration guide goes here)
- **SKU requirements:** Use the Standard tier or Premium tier to enable zone redundancy. No specific pricing tiers or SKUs are required.
<!--NOTE TO SELF:> Region support details are currently documented in legacy migration guidance. A long-term approach for managing this information in Learn content is under discussion. -->

### Considerations

The system automatically enables availability zone resiliency in supported regions. You can't opt out and don't need to take any action.

### Cost

There's no extra cost for zone redundancy for Azure App Configuration.

### Configure availability zone support

Microsoft automatically enables zone redundancy for a store when it's in [a region that supports availability zones](#requirements).

### Behavior when all zones are healthy

When a store is in a region that supports zone redundancy and all availability zones are operational, you can expect the following behavior:

- **Traffic routing between zones:** App Configuration automatically manages traffic routing between availability zones. During normal operations, it transparently distributes requests across zones.

- **Data replication between zones:** In regions that support zones, App Configuration synchronously replicates data across availability zones. This replication ensures that your settings remain consistent and available even if a zone becomes unavailable.

    > [!IMPORTANT]
    > **Note to PG:** Please confirm that cross-zone replication is synchronous.
### Behavior during a zone failure

This section describes what to expect when a store is in a region that supports zone redundancy and an availability zone is unavailable:

- **Detection and response:** The App Configuration service detects zone failures and automatically responds to them. You don't need to take any action during a zone failure.

[!INCLUDE [Availability zone down notification (Service Health only)](./includes/reliability-availability-zone-down-notification-service-include.md)]

- **Active requests:** During a zone failure, the affected zone might fail to handle in-flight requests, which requires client applications to retry them. Client applications should follow [transient fault handling practices](#resilience-to-transient-faults) to ensure that they can retry requests if a zone failure occurs.

- **Expected data loss:** No data loss is expected during a zone failure because of the synchronous replication between zones.

    > [!IMPORTANT]
    > **Note to PG:** Please confirm that cross-zone replication is synchronous.

- **Expected downtime:** A small amount of downtime, usually a few seconds, is expected while the service switches to use infrastructure in a healthy zone.

    > [!IMPORTANT]
    > **Note to PG:** Please confirm that this statement about downtime is accurate.

- **Traffic rerouting:** App Configuration automatically reroutes traffic away from the affected zone to healthy zones without requiring any customer intervention.

## Resilience to region-wide failures

Azure App Configuration provides native **geo-replication** capabilities to support resilience during regional outages. Geo-replication allows configuration data to be replicated across regions as a managed service feature.

### Geo-replication

Geo-replication is a product feature that enables a configuration store to be replicated across multiple Azure regions. Each store can have multiple replicas in different regions. The original store is also a replica. This capability helps protect applications from region-wide service disruptions.

#### Requirements

The configuration store must use a supported tier to enable geo-replication. For more information, see (/azure/azure-app-configuration/howto-geo-replication).


#### Considerations

When you enable geo-replication, consider the following factors:

- **Zone-redundant replicas:** Any replica you create in a region in which App Configuration supports availability zones is automatically zone-redundant.

- **Azure Front Door:** If you use Azure Front Door to access your store, your applications must connect through Azure Front Door, and Azure Front Door controls replica selection and failover. For more information, see:(/azure/azure-app-configuration/concept-hyperscale-client-configuration#failover-and-load-balancing).

#### Cost

Each geo-replicated region is billed separately according to the pricing for the respective tier and region.
    > [!IMPORTANT]
    > **Note to PG:** Is inter-region data egress charged? We say this in some other guides (for example, Azure Container Registry): "Egress charges also apply for data transfer between regions during initial replication and ongoing synchronization."

For pricing details, see [Azure App Configuration pricing](https://azure.microsoft.com/pricing/details/app-configuration/).

#### Configure multi-region support

To set up replication for a newly created configuration store, see [Enable geo-replication](/azure/azure-app-configuration/howto-geo-replication). 

#### Behavior when all regions are healthy

This section describes what to expect when an App Configuration store is configured for geo-replication, and the primary region is operational.

- **Traffic routing between regions:** 
    - Each replica is addressable individually and has its own DNS name (note to PG - confirm we're OK to say that)
    > [!IMPORTANT]
    > **Note to PG:** Is this correct?
    - When you use Microsoft's configuration providers, your application can optionally use automatic replica discovery, or specify a prioritized list of replicas. App Configuration selects the first healthy replica. This enables your application to control which replica it uses.
    - All replicas can accept write operations.

    > [!NOTE]
    > If you use Azure Front Door, traffic routing behavior is different. For more information, see [Failover and load balancing](/azure/azure-app-configuration/concept-hyperscale-client-configuration#failover-and-load-balancing).

- **Data replication between regions:** Data is replicated asynchronously and is eventually consistent. You can use the 'replication lag' metric in Azure Monitor to monitor the current replication lag between replicas.
    > [!IMPORTANT]
    > **Note to PG:** Can we give an approximate time here like 'typically within 15 minutes' or something?

#### Behavior during a region failure

This section describes what to expect when you configure app configuration for geo-replication and there's an outage in the primary or a secondary region.

- **Detection and response:** 
    - Microsoft is responsible for detecting region or replica failures and initiating recovery processes.
    - Your application must switch to another replica. 
<!--(Is this phrased correctly, John? Are there configuration steps needed or an article we should link to?)-->

- **Notification:** [!INCLUDE [Region down notification partial bullet (Azure Service Health only)](./includes/reliability-region-down-notification-service-partial-include.md)]

    Use that information and other metrics to decide when to promote a secondary region to a primary region.

- **Active requests:** Retry existing requests against a different replica. 

- **Expected data loss:** If a replica fails, recent changes on that replica might not yet exist on other replicas. Those changes can remain unavailable until the replica recovers. To estimate potential data loss, monitor the replication lag metric in Azure Monitor. 

- **Expected downtime:** When a replica becomes unavailable, it stays offline until its region recovers. Other replicas continue to handle requests. Applications might experience brief downtime while they detect the failure and switch to a healthy replica. The duration depends on how quickly each application performs this detection and failover.

- **Traffic rerouting:** Applications must route traffic to a healthy replica when a failure occurs.

- If you use Microsoft configuration provider libraries, the libraries automatically handle replica selection and failover.
- If you place Azure Front Door in front of your data store and configure the origin group for failover, applications continue to operate without manual intervention.

#### Region recovery

After the region recovers, App Configuration brings the replica back in sync with the other replicas. Applications using Microsoft configuration providers automatically start using it again.

#### Test for region failures

Applications control replica selection, so they can dynamically switch replicas to test failover behavior.

> [!IMPORTANT]
> **Note to PG:** Can we give an approximate time here like 'typically within 15 minutes' or something?

## Backup and recovery

Azure App Configuration enables you to [export configuration data](/azure/azure-app-configuration/concept-snapshots) from a store and use it as part of a broader backup strategy.

For most solutions, don't rely exclusively on backups. Instead, use the other capabilities described in this guide to support your resiliency requirements. However, backups protect against some risks that other approaches don't.

## Recovery features

App Configuration provides two key recovery features to prevent accidental or malicious deletion:

- **Soft delete:** When enabled, soft delete allows you to recover deleted stores and objects during a configurable retention period. Think of soft delete like a recycle bin for your App Configuration resources.

- **Purge protection:** When enabled, purge protection prevents permanent deletion of your store and its objects until the retention period elapses. This safeguard prevents malicious actors from permanently destroying your settings.

Use both features for production environments. For more information, see [Soft-delete and purge protection](/azure/azure-app-configuration/concept-soft-delete).

## Resilience to service maintenance


Microsoft regularly performs service updates and other maintenance. The service handles these activities automatically, ensuring that maintenance is seamless and transparent to customers. No downtime occurs during maintenance events. As a result, no customer actions are required to maintain reliability.

> [!IMPORTANT]
> **Note to PG:** Please verify that we're OK to say "No downtime is expected during maintenance events."

## Resilience to configuration problems

Incorrect or accidental configuration changes can cause application downtime. Use [configuration snapshots](/azure/azure-app-configuration/concept-snapshots) to safely roll out changes to configuration. Monitor the application instances and revert them to the last-known-good configuration snapshot if the changes introduce a problem.
## Service-level agreement

[!INCLUDE [Service-level agreement](includes/reliability-service-level-agreement-include.md)] 

## Related content

- [Reliability in Azure](./overview.md)