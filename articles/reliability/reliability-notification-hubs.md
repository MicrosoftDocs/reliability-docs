---
title: Reliability in Azure Notification Hubs
description: Learn how to make Azure Notification Hubs resilient to transient faults, availability zone failures, region-wide failures, and service maintenance, and learn about backup and restore.
author: glynnniall
ms.author: pnp
ms.service: azure-notification-hubs
ms.topic: reliability-article
ms.custom: subject-reliability, references_regions
ms.date: 08/27/2026
---

# Reliability in Azure Notification Hubs

[Azure Notification Hubs](/azure/notification-hubs/notification-hubs-push-notification-overview) helps you manage push notifications across multiple platform notification systems (PNS), such as Apple Push Notification service (APNs), Firebase Cloud Messaging (FCM), and Windows Push Notification Service (WNS).

[!INCLUDE [Shared responsibility](includes/reliability-shared-responsibility-include.md)]

This article describes how to make Notification Hubs resilient to various potential outages and problems, including transient faults, availability zone failures, region-wide failures, and service maintenance. It also describes backup and restore options and key information about the Notification Hubs service-level agreement (SLA).

## Production deployment recommendations

For production workloads, follow these recommendations:

> [!div class="checklist"]
> - Use the Basic or Standard tier so that your namespace is eligible for the SLA.
>
> - When possible, use installations instead of registrations in device applications.
>
> - Use Microsoft-provided SDKs to interact with Notification Hubs.
>
> - Enable zone redundancy.
>
> - To prepare for region-wide outages, enable metadata disaster recovery to another Azure region. Plan how to back up and restore device registrations and installations.

## Reliability architecture overview

Azure Notification Hubs is organized around *namespaces* and *notification hubs*. A namespace is a management boundary that contains one or more hubs. Hubs represent endpoints for an application. Devices register with those endpoints by using either *registrations* or *installations*, which enable the service to send push notifications to the devices. For more information, see [Registration management](/azure/notification-hubs/notification-hubs-push-notification-registration-management).

Notification Hubs sends push notifications to platform notification systems (PNS), such as Apple Push Notification service (APNs) and Firebase Cloud Messaging (FCM). End-to-end notification delivery depends on the availability of Notification Hubs and the behavior of downstream PNS providers.

For reliability planning, it's important to distinguish between the following types of data that Notification Hubs manages:

- **Metadata:** Namespace and hub configuration, including connection information and disaster recovery configuration.
- **Registration data:** Device registrations and installations that map users and devices to tags and templates.

## Resilience to transient faults

[!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)]

Notification Hubs automatically handles transient faults that occur when connecting to a PNS. However, you're responsible for handling transient failures when your services or users' devices interact with Notification Hubs. Transient failures can occur during registration operations, notification send operations, and management operations. Follow this guidance:

- **Registrations and installations:** Your applications on devices should retry registration and installation operations that fail because of transient faults. Microsoft-provided SDKs handle retries automatically. If you can't use the provided SDKs, implement retry logic with exponential backoff and jitter, and make registration operations idempotent where possible.

  Creating or updating an installation is idempotent, so you can safely retry the operation. When possible, use installations instead of registrations.

- **Notification sends and management operations:** Use a Microsoft-provided SDK to send push notifications and perform management operations. These SDKs automatically retry when transient faults occur.

  If you can't use the provided SDKs, implement retry logic with exponential backoff and jitter, and make notification sending operations idempotent where possible.

## Resilience to availability zone failures

[!INCLUDE [Resilience to availability zone failures](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)]

In regions that support availability zones, Notification Hubs namespaces support a *zone-redundant* configuration. Notification Hubs automatically enables zone redundancy for all namespaces in some regions. When zone redundancy is enabled, Microsoft replicates both metadata and registration data across all availability zones in the region.

:::image type="complex" border="false" source="./media/reliability-notification-hubs/zone-redundant.svg" alt-text="Diagram that shows a zone-redundant Notification Hubs namespace that uses three availability zones in a region." lightbox="./media/reliability-notification-hubs/zone-redundant.svg":::
  The diagram shows availability zones 1, 2, and 3. A single Notification Hubs namespace spans all three zones.
:::image-end:::

### Requirements

- **Region support:**

  Notification Hubs automatically enables zone redundancy for all namespaces in the following regions. You can't disable zone redundancy in these regions:

  | Europe            | Middle East   | Africa             | Asia Pacific  |
  |-------------------|---------------|--------------------|---------------|
  | France Central    | Qatar Central | South Africa North | China North 3 |
  | Italy North       |               |                    | Korea Central |
  | Norway East       |               |                    |               |
  | Poland Central    |               |                    |               |
  | Sweden Central    |               |                    |               |
  | Switzerland North |               |                    |               |

  In other regions that [support Notification Hubs](https://azure.microsoft.com/explore/global-infrastructure/products-by-region/table) and have [availability zones](./regions-list.md), zone redundancy is optional. You can enable it only when you create a namespace.

- **Tier support:** You can use availability zones with all tiers of Notification Hubs.

### Cost

Zone redundancy incurs an additional charge beyond the tier price. For more information, see [Notification Hubs pricing](https://azure.microsoft.com/pricing/details/notification-hubs).

### Configure availability zone support

- **Create a new zone-redundant namespace:** The process to create a new zone-redundant namespace depends on the region you use:

  - In regions where Notification Hubs automatically enables zone redundancy, you don't need to configure it.

    > [!IMPORTANT]
    > In these regions, Notification Hubs always creates namespaces with zone redundancy enabled, even if a code-based deployment, such as a Bicep file or Azure Resource Manager template, specifies that zone redundancy is disabled.
    >
    > If you don't want a zone-redundant namespace, create it in a region that supports optional zone redundancy.

  - In regions where zone redundancy is optional, you can enable it only when you create a namespace. To learn how to set up a new namespace with zone redundancy, see [Create an Azure notification hub in the Azure portal](/azure/notification-hubs/create-notification-hub-portal).

- **Make an existing namespace zone redundant:** Notification Hubs doesn't support in-place migration of an existing namespace to availability zone support. You need to deploy a new namespace and move your registrations to that namespace. Follow the guidance in [Move resources between Azure regions](/azure/notification-hubs/move-registrations), which also applies if you deploy the new namespace in the same region.

### Behavior when all zones are healthy

This section describes what to expect when you configure a Notification Hubs namespace for zone redundancy, and all zones are operational.

- **Cross-zone operation:** Notification Hubs automatically distributes and serves requests by using infrastructure in any zone in the region.

- **Cross-zone data replication:** Both registration data and metadata are synchronously replicated across all zones in the specified region.

### Behavior during a zone failure

This section describes what to expect when you configure a Notification Hubs namespace for zone redundancy, and there's an outage in one of the zones.

- **Detection and response:** Microsoft detects zone failures and manages failover within the region. You don't need to initiate failover.

[!INCLUDE [Availability zone down notification (Service Health only)](./includes/reliability-availability-zone-down-notification-service-include.md)]

- **Active requests:** In-flight management operations, device registrations, and new requests to send notifications might fail during failover. Your applications should retry failed operations by following [transient fault handling guidance](#resilience-to-transient-faults).

- **Expected data loss:** Data loss isn't expected during a single-zone outage because Notification Hubs synchronously replicates namespace and hub configuration and registration data across availability zones.

  This replication isn't a backup. Under the shared responsibility model, you're responsible for backing up registration and installation data. For more information, see [Backup and restore](#backup-and-restore).

- **Expected downtime:** A brief service interruption is possible while Microsoft reroutes traffic. Follow the [transient fault handling guidance](#resilience-to-transient-faults) to prepare your applications for these interruptions.

- **Redistribution:** The service automatically redirects requests to healthy zones.

### Zone recovery

When the affected zone recovers, you don't need to take any action. Microsoft restores and rebalances Notification Hubs infrastructure to use the recovered zone.

### Test for zone failures

You can't trigger a Notification Hubs zone failover directly. To test your workload behavior, run resilience tests for retries, idempotency, and dependency failures in nonproduction environments. You can also use Azure Chaos Studio to test surrounding application components.

## Resilience to region-wide failures

Notification Hubs provides metadata disaster recovery by [replicating namespace metadata across regions](#microsoft-managed-metadata-geo-disaster-recovery), but it doesn't replicate device registration data. This capability requires manual intervention during a region outage and involves some downtime for your notification hub.

If you need to reduce downtime and manual intervention during failover, consider using a [custom multiregion solution](#custom-multiregion-solutions-for-resiliency).

### Microsoft-managed metadata geo-disaster recovery

Notification Hubs supports Microsoft-managed metadata disaster recovery to a secondary Azure region. If your primary region has a paired region, you can select that paired region. Regardless of the pairing status of your primary region, you can also choose a secondary region from a list of *flexible recovery regions*. Notification Hubs then replicates namespace metadata, such as the namespace name, connection strings, and other critical information.

:::image type="complex" border="false" source="./media/reliability-notification-hubs/multiregion.svg" alt-text="Diagram that shows Notification Hubs metadata disaster recovery from a primary region to a secondary region." lightbox="./media/reliability-notification-hubs/multiregion.svg":::
  A client application connects through the namespace endpoint to the primary namespace in the primary region. The primary namespace contains metadata and registrations. Notification Hubs replicates only metadata to the secondary namespace in the secondary region; registration data isn't replicated.
:::image-end:::

> [!IMPORTANT]
> Metadata geo-disaster recovery doesn't replicate registration data. If a disaster recovery scenario is triggered, registration and installation data can be lost. You're responsible for implementing a solution to repopulate registration data in your hub after recovery.

Microsoft is responsible for declaring a disaster and initiating failover. When that happens, Microsoft creates a new namespace in the secondary region. Because it uses the metadata from the primary region, applications can connect to that namespace by using the existing namespace name, connection string, and hub names.

:::image type="complex" border="false" source="./media/reliability-notification-hubs/multiregion-failover.svg" alt-text="Diagram that shows failover from a primary Notification Hubs region to a secondary region." lightbox="./media/reliability-notification-hubs/multiregion-failover.svg":::
  The primary region and its namespace are unavailable, and metadata replication from the primary region has stopped. The client application continues to use the namespace endpoint, which connects to the secondary namespace in the secondary region. The secondary namespace contains the replicated metadata but not the registrations from the primary namespace.
:::image-end:::

#### Requirements

- **Region support:** In paired Azure regions, your namespace can use the [Azure paired region](./regions-paired.md#azure-region-pairs-list) as the secondary region.

  If your namespace is in a nonpaired region, or if you want to replicate data to a different region, you can select one of the following flexible recovery regions as the secondary region:

  | Americas     | Europe       | Africa             | Asia Pacific   |
  |--------------|--------------|--------------------|----------------|
  | Brazil South | North Europe | South Africa North | Australia East |
  | West US 2    |              |                    | Southeast Asia |

- **Tier support:** Metadata disaster recovery options are available across all Notification Hubs tiers.

#### Cost

Notification Hubs doesn't charge extra to configure or use metadata geo-disaster recovery. However, you pay for the cross-region bandwidth used to replicate metadata. For pricing details, see [Bandwidth pricing](https://azure.microsoft.com/pricing/details/bandwidth/) and [Notification Hubs pricing](https://azure.microsoft.com/pricing/details/notification-hubs/).

#### Configure multiregion support

- **Enable metadata geo-disaster recovery for a new namespace:** Follow the procedure in [Create an Azure notification hub in the Azure portal](/azure/notification-hubs/create-notification-hub-portal). Select the disaster recovery configuration when you create the namespace.

- **Enable or disable metadata geo-disaster recovery for an existing namespace:** Follow the procedure in [Enable disaster recovery for an existing Azure Notification Hubs namespace](/azure/notification-hubs/enable-disaster-recovery-existing-namespace).

- **Back up your device registration data:** See [Export and import Azure Notification Hubs registrations in bulk](/azure/notification-hubs/export-modify-registrations-bulk).

#### Behavior when all regions are healthy

This section describes what to expect when you configure a Notification Hubs namespace for metadata geo-disaster recovery, and both your primary and secondary regions are operational.

- **Cross-region operation:** The primary region serves all requests. The secondary region doesn't serve requests unless failover occurs.

- **Cross-region data replication:** Metadata, such as the namespace name, hub configuration, connection strings, and other critical information, is replicated asynchronously across regions. Registration data isn't replicated. You're responsible for exporting it regularly to maintain a backup.

#### Behavior during a region failure

This section describes what to expect when you configure a Notification Hubs namespace for metadata geo-disaster recovery, and there's an outage in the primary region.

- **Detection and response:** Microsoft is responsible for detecting the region failure and deciding whether to trigger failover to the configured secondary region.

[!INCLUDE [Region down notification (Service Health only)](./includes/reliability-region-down-notification-service-include.md)]

- **Active requests:** In-flight requests to the namespace in the primary region might fail when the region goes offline. Clients should retry operations after failover completes.

- **Expected data loss:** Metadata is preserved. Registration data isn't automatically backed up, but you can back it up yourself. For more information, see [Export and import Azure Notification Hubs registrations in bulk](/azure/notification-hubs/export-modify-registrations-bulk). If you don't, registration data is unavailable until the primary region recovers.

- **Expected downtime:** It takes some time for Microsoft to trigger failover of metadata and then for failover to complete. While the time can vary, it generally takes several hours.

  After failover completes, you're responsible for restoring any backups of registration data.

- **Redistribution:** After failover, requests are routed to a namespace in the secondary region that uses the replicated data from the primary region. After failover completes, clients connect to the namespace in the secondary region automatically.

#### Region recovery

If the primary region recovers, it might be possible to fail back to the primary namespace in the primary region. The primary namespace would retain the registration data from before the outage. This would be a manual process and Microsoft would communicate with you to explain how this works.

After the primary region recovers, you need to:

- Validate the state of your namespace and its data.
- Determine whether to synchronize recent registration data changes from the secondary region back to the primary region.

#### Test for region failures

You can't initiate a geo-failover. However, you should test your own disaster recovery procedures. Verify that registrations are backed up and that you can restore them to a new namespace.

### Custom multiregion solutions for resiliency

Microsoft-managed metadata geo-disaster recovery only replicates metadata. The feature can recover that metadata to a secondary namespace, but you're responsible for importing device registrations into that namespace so your application can continue to operate. This approach requires manual intervention during a disaster and involves downtime.

If your recovery objectives require less downtime or manual intervention, you can implement a custom active-active multiregion solution. Deploy a second Notification Hubs namespace to another Azure region ahead of time.

> [!NOTE]
> This section provides basic guidance for designing this type of solution. You're responsible for designing, implementing, testing, deploying, failing over, and managing the solution.

- **Failover:** Because the second namespace is a working resource, you can implement logic to detect a region failure and switch to that namespace.

- **Synchronization:** To keep a second notification hub in sync with the primary notification hub, use one of the following options:

  - **For installations:** Use an app backend that simultaneously creates and updates installations in both notification hubs. Installations enable you to specify your own unique device identifier, which supports this replication scenario. For more information, see the [RedundantHub sample](https://github.com/Azure/azure-notificationhubs-dotnet/tree/main/Samples/RedundantHubSample).

  - **For registrations:** Use an app backend that regularly exports registrations from the primary notification hub as a backup and bulk imports them into the secondary notification hub. For more information, see [Export and import Azure Notification Hubs registrations in bulk](/azure/notification-hubs/export-modify-registrations-bulk).

  Alternatively, if you don't have a backend, configure your app to create installations in both hubs when the app starts on target devices. The devices create new registrations in both notification hubs. Eventually the secondary notification hub has all the active devices registered.

- **Expired registrations and installations:** The secondary notification hub might have expired registrations and installations. When a push is made to an expired handle, Notification Hubs automatically cleans the associated registration or installation record on the notification hub, based on the response received from the PNS server. You can clean expired records from the backup solution of your choice by adding custom logic that processes feedback from each send and removes expired registrations and installations.

- **Unopened apps:** There's a period of time during which devices with unopened apps don't receive notifications.

- **Cost:** If you use your own secondary hub to protect registration data, that hub incurs normal service charges. Similarly, if you deploy any other Azure resources into your secondary region to support your recovery, you pay for those at normal service rates.

## Backup and restore

Notification Hubs doesn't provide a single built-in backup and restore feature for all of the data stored in your namespace. You're responsible for combining the following approaches:

- Use infrastructure as code (IaC), such as Bicep, to define your namespace, hub, and policy configuration. Store those definitions in source control so that you can redeploy the resources when necessary.
- Back up your device registration data by [exporting Azure Notification Hubs registrations in bulk](/azure/notification-hubs/export-modify-registrations-bulk).

## Resilience to service maintenance

[!INCLUDE [Service maintenance (no special callouts)](includes/reliability-maintenance-include.md)]

## Service-level agreement

[!INCLUDE [Service-level agreement](includes/reliability-service-level-agreement-include.md)]

For Notification Hubs, the availability SLA applies to namespaces that use the Basic and Standard tiers.

## Related content

- [Reliability in Azure](./overview.md)
