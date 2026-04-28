---
title: Reliability in Azure SignalR Service
description: Learn how to make Azure SignalR Service resilient to a variety of potential outages and problems, including transient faults, availability zone outages, region outages, and service maintenance.
author: glynnniall
ms.author: pnp
ms.topic: reliability-article
ms.custom: subject-reliability, references_regions
ms.service: azure-signalr-service
ms.date: 04/24/2026
ai-usage: ai-assisted
---

# Reliability in Azure SignalR Service

[Azure SignalR Service](/azure/azure-signalr/signalr-overview) is a fully managed service that enables real-time bidirectional communication in web and mobile applications. It abstracts the underlying transport mechanism. When WebSockets are available, the service uses them. When they aren't, it falls back to Server-Sent Events or long polling. This abstraction means your client and server code can communicate in real time without being tied to a specific transport protocol.

[!INCLUDE [Shared responsibility](includes/reliability-shared-responsibility-include.md)]

This article describes how to make Azure SignalR Service resilient to a variety of potential outages and problems, including transient faults, availability zone outages, region outages, and service maintenance. It also highlights key information about the Azure SignalR Service service-level agreement (SLA).

## Production deployment recommendations for reliability

For production workloads, we recommend that you:

> [!div class="checklist"]
>
> - Use the Premium tier. The premium tier is resilient to availability zone failures in supported regions, and enables you to configure geo-replication.
> - Design client applications and app servers to safely reconnect automatically when connections drop. Zone failovers, region failovers, and transient faults all drop active connections.
> - Enable geo-replication to protect against region-wide failures. Size each replica with enough units to handle your full expected traffic load during a failover event.

## Reliability architecture overview

[!INCLUDE [Introduction to reliability architecture overview section](includes/reliability-architecture-overview-introduction-include.md)]

### Logical architecture

The resource you create is a *SignalR Service resource*. You configure a resource with a number of *units*, which represents the capacity of the resource, including the maximum number of concurrent connections. For more information, see [Performance guide for Azure SignalR Service](/azure/azure-signalr/signalr-concept-performance).

Azure SignalR Service supports two *service modes*. In default mode, your app servers connect to the Azure SignalR Service resource and contain your hub logic. In serverless mode, the service integrates with Azure Functions, and Functions act as event-driven message handlers so you don't manage app servers directly. For more information, see [Service mode in Azure SignalR Service](/azure/azure-signalr/concept-service-mode).

A SignalR Service resource has a globally unique endpoint similar to `contoso.service.signalr.net`. Clients establish connections to this endpoint. Application servers connect to the same endpoint to send messages and receive events from clients.

### Physical architecture

Azure SignalR Service manages connection state and message routing across a set of compute resources. Microsoft manages the underlying infrastructure. You don't directly see or interact with individual VMs that the service uses, or other infrastructure components.

## Resilience to transient faults

[!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)]

Azure SignalR Service uses long-lived connections between clients, app servers, and the service. These connections can be dropped due to transient faults such as network instability, load balancer reconfigurations, or browser tab suspensions. Design your client applications and app servers to handle connection drops and reconnect automatically.

The Azure SignalR Service SDKs include built-in reconnection handling for server connections to the service. Client-side reconnection depends on the client library you use. ASP.NET Core SignalR clients support stateful reconnect, which allows a client to resume its previous connection without losing state if it reconnects quickly using the same connection ID. If the reconnection results in a new connection ID, for example, after a longer outage, the service treats the client as a new connection. In this case, the client needs to rejoin any groups it was previously in and restore any session state.

For detailed guidance on designing your application to handle client disconnections and reconnections, see [Understanding client disconnections and reconnection in Azure SignalR Service](/azure/azure-signalr/signalr-concept-client-disconnections).

## Resilience to availability zone failures

[!INCLUDE [Resilience to availability zone failures](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)]

Azure SignalR Service supports zone-redundant deployments in the Premium tier. When you create or upgrade to a Premium tier resource in a region that supports availability zones, Azure SignalR Service automatically distributes its compute capacity across all availability zones in the region. If an availability zone fails, Azure SignalR Service routes new connections to instances in the remaining healthy zones.

:::image type="content" source="./media/reliability-signalr/zone-redundant.svg" alt-text="Diagram that shows a zone-redundant Azure SignalR service, spread across multiple availability zones." border="false":::

### Requirements

- **Region support:** Zone redundancy is supported in most regions where both of these conditions apply:
  - Azure SignalR Service is available. For a list of regions where the service is available, see [Product Availability by Region](https://azure.microsoft.com/explore/global-infrastructure/products-by-region/table).
  - The region supports availability zones. For a list of regions with availability zones, see [Azure regions list](./regions-list.md).

  However, Japan West doesn't currently support zone redundancy for Azure SignalR Service.

- **Tier:** Zone redundancy is available on the Premium tier.

### Cost

Zone redundancy doesn't add cost, and you pay the standard Premium tier rate. For more information, see [Azure SignalR Service pricing](https://azure.microsoft.com/pricing/details/signalr-service/).

### Configure availability zone support

Zone redundancy requires no configuration beyond selecting the Premium tier. It's automatically enabled in both of these cases:

- **Create a new zone-redundant SignalR Service resource.** Select a Premium tier SKU when you create the resource. For more information, see the quickstarts, such as [Quickstart: Create a chat room by using SignalR Service](/azure/azure-signalr/signalr-quickstart-dotnet-core).

- **Upgrade an existing resource to Premium tier.** Zone redundancy is automatically enabled when you upgrade an existing resource to a Premium tier SKU. Upgrading from Standard to Premium doesn't cause service downtime. For more information, see [Change your Azure SignalR Service tier](/azure/azure-signalr/signalr-howto-scale-signalr).

### Behavior when all zones are healthy

This section describes what to expect when you configure an Azure SignalR Service resource for zone redundancy and all availability zones are operational.

- **Cross-zone operation:** Azure SignalR Service automatically manages how connections and operations are distributed across availability zones. Infrastructure in multiple zones process traffic in an active-active model. You don't need to configure anything to take advantage of this behavior. The service routes messages between instances across zones automatically, so a message sent by a client in one zone is delivered to clients connected in any other zone.

- **Cross-zone data replication:** Azure SignalR Service doesn't persist customer data, so there's no data to replicate between zones. Connection state is ephemeral and is associated with each active connection.

### Behavior during a zone failure

This section describes what to expect when you configure an Azure SignalR Service resource for zone redundancy and there's an outage in one of the availability zones.

- **Detection and response:** The Azure SignalR Service platform is responsible for detecting a failure in an availability zone. You don't need to take any action to initiate a zone failover.

[!INCLUDE [Availability zone down notification (Service Health and Resource Health)](./includes/reliability-availability-zone-down-notification-service-resource-include.md)]

- **Active requests:** During a zone failure, active connections to nodes in the affected zone are dropped. If your clients handle [transient faults](#resilience-to-transient-faults) appropriately, such as by reconnecting after a short period of time, they typically avoid significant impact.

- **Expected data loss:** Azure SignalR Service doesn't persist messages, so a zone failure isn't expected to cause data loss within the Azure SignalR service. However, any active connections are dropped during a zone-down event and so any data that's actively being transmitted might be lost.

- **Expected downtime:** The reconnect of dropped active connections typically takes a few seconds. Clients that implement reconnect logic experience minimal disruption.

- **Redistribution:** Azure SignalR Service detects the loss of the zone and automatically redistributes traffic across the healthy zones. You don't need to take any action.

### Zone recovery

When an availability zone recovers, Azure SignalR Service automatically reintegrates it into the active service topology. You don't need to take any action for zone recovery.

After a zone recovers, new connections might be directed to infrastructure in the recovered zone. Any existing connections won't be moved or rebalanced to the recovered zone, but they'll be gradually rebalanced as the existing connections drop and reconnect over time. Connection imbalance across zones doesn't have any impact on your workload.

### Test for zone failures

Azure SignalR Service manages traffic routing, failover, and zone recovery automatically for zone-redundant Premium tier resources. You don't need to initiate anything. Because zone redundancy is fully managed, you don't need to validate availability zone failure processes.

## Resilience to region-wide failures

Azure SignalR Service is a single-region service. If the region becomes unavailable, your SignalR Service resource is also unavailable.

To protect your application against a region-wide failure, you can use *geo-replication*, which is available in the Premium tier. Alternatively, you can build a custom multiregion solution by deploying multiple SignalR Service resources in different regions.

### Geo-replication

Geo-replication enables you to add *replicas* of your SignalR Service resource in other Azure regions. Azure Traffic Manager uses DNS-based routing to direct each client to the nearest healthy regional replica. If a region fails, the Traffic Manager detects the failure through health checks and stops directing clients to that replica. New client connections are automatically routed to the nearest healthy replica.

:::image type="content" source="./media/reliability-signalr/geo-replication.svg" alt-text="Diagram that shows Azure SignalR Service configured for geo-replication across two regions." border="false":::

The region you created the Azure SignalR Service resource in is called the *primary region*, and its replica is the *primary replica*. The *control plane* of the primary resource manages the configuration of your Azure SignalR Service resource.

#### Requirements

- **Region support:** You can add replicas in any region where Azure SignalR Service is available.
- **Tier:** You must use the Premium tier to enable geo-replication.
- **Replica limit:** Each primary SignalR Service resource supports up to eight replicas.

#### Considerations

- **Configuration inheritance:** Replicas inherit most configuration settings from the primary resource. Certain settings must be configured separately on each replica. For the complete list of settings that aren't inherited, see [Geo-replication in Azure SignalR Service](/azure/azure-signalr/howto-enable-geo-replication).

- **Configuration changes:** The primary control plane, in the primary region, processes any configuration changes to the SignalR Service resource. If the primary control plane is unavailable, you will be unable to update the resource configuration, though existing replicas will continue to process data traffic without interruption.

#### Cost

Each replica is billed separately based on its own unit count and outbound message volume. Cross-region egress fees apply when messages are transferred between replicas and delivered to clients or servers in another region. For more information, see [Azure SignalR Service pricing](https://azure.microsoft.com/pricing/details/signalr-service/).

#### Configure geo-replication

To add or remove a replica to a SignalR Service resource, see [Geo-replication in Azure SignalR Service](/azure/azure-signalr/howto-enable-geo-replication).

#### Capacity planning and management

Each replica handles traffic independently. During a regional failover, clients from the failed region reconnect to the nearest healthy replica. To ensure that the surviving replicas have enough capacity to absorb this extra load, configure each replica with units that can handle the full expected traffic of the workload, not just the portion it normally serves.

Alternatively, enable autoscaling on each replica so units can scale out automatically in response to higher load. Autoscaling continues to work when a secondary replica is unavailable, but autoscaling doesn't work if the primary control plane is unavailable. For more information about autoscaling, see [Autoscaling in Azure SignalR Service](/azure/azure-signalr/signalr-howto-scale-autoscale).

For general guidance on overprovisioning as a strategy, see [Manage capacity by overprovisioning](/azure/reliability/concept-redundancy-replication-backup#manage-capacity-with-over-provisioning).

#### Behavior when all regions are healthy

This section describes what to expect when you configure Azure SignalR Service for geo-replication and all replicas are operational.

- **Cross-region operation:** Azure Traffic Manager routes each client to the nearest healthy regional replica. Clients in different geographic areas might connect to different replicas. SignalR Service synchronizes messages across replicas so that clients connected to any replica can communicate with each other.

- **Cross-region data replication:** When a message is sent to a replica, the service synchronously transfers that message to other replicas so that clients connected elsewhere can receive it. The synchronization overhead is minimal for most common messaging patterns, such as broadcasting to large groups or messaging a single connection. Messaging to small groups (fewer than 10 members) might produce a slightly higher synchronization overhead.

  Azure SignalR Service doesn't persist messages; only active delivery is synchronized across replicas.

#### Behavior during a region failure

This section describes what to expect when you configure Azure SignalR Service for geo-replication and there's an outage in one of the replica regions.

- **Detection and response:** SignalR Service is responsible for detecting a failure in a region and automatically rerouting incoming traffic to a replica in one of the other regions that you configure.

[!INCLUDE [Region down notification (Service Health only)](./includes/reliability-region-down-notification-service-include.md)]

- **Active requests:** Active connections to the replica in the failed region are dropped. Clients must reconnect after the replica fails over.

- **Expected data loss:** Azure SignalR Service doesn't persist messages. Messages that were in transit to clients in the failed region at the time of the failure might be lost. No persistent data loss is expected because the service doesn't store customer data.

- **Expected downtime:** Azure Traffic Manager performs health checks against each replica. When a region outage causes a replica to fail its health check, the Traffic Manager removes that replica's endpoint from its DNS resolution results. After removing the endpoint, the DNS TTL of 90 seconds must elapse before clients see updated DNS records. In total, the transition typically takes a few minutes. Well-designed clients that implement reconnect logic can resume normal operation after reconnecting to the healthy replica.

  If the primary control plane is unavailable, you can't make any changes to the configuration of your SignalR Service resource or its replicas. However, connections continue to work in healthy replicas.

- **Redistribution:** Azure Traffic Manager directs incoming request to healthy replicas. However, if a client attempts to reconnect before Azure Traffic Manager has detected the replica failover and the updated DNS entries have propagated to the client, then a client's reconnect attempt might continue to target the unavailable region and could fail.

   After the DNS update propagates, reconnecting clients are automatically routed to the nearest healthy replica.

#### Region recovery

When the failed region recovers, the Traffic Manager health check detects the restored replica and includes its endpoint in DNS resolution again. Clients currently connected to other replicas aren't affected and remain connected until they disconnect. New connections are again routed to the recovered region's replica when it is the nearest healthy replica.

#### Test for region failures

To simulate a region failover and test your client application's reconnect behavior, you can disable a replica's endpoint. This action causes Traffic Manager to stop routing traffic to that replica, which lets you observe how your clients behave when the replica they connect to becomes unavailable. For detailed steps, see [Disable or enable the replica endpoint](/azure/azure-signalr/howto-enable-geo-replication#disable-or-enable-the-replica-endpoint).

### Custom multi-region solutions for resiliency

If you need cross-region resiliency but aren't using geo-replication, you can deploy and manage separate SignalR Service resources in multiple regions and implement your own failover logic in your application server. This approach is more complex than geo-replication and doesn't support zero-downtime failover for client-to-client connectivity. For a detailed architecture overview, failover patterns, and testing guidance, see [Resiliency and disaster recovery in Azure SignalR Service](/azure/azure-signalr/signalr-concept-disaster-recovery).

## Backup and restore

Azure SignalR Service is a stateless messaging service. It doesn't persist customer messages and has no backup or restore capability.

To protect your resource configuration, define your SignalR Service resources using infrastructure as code (such as Bicep or ARM templates) and store those definitions in source control. If you need to recreate a resource, redeploy it from the stored configuration.

## Resilience to service maintenance

[!INCLUDE [Service maintenance (no special callouts)](includes/reliability-maintenance-include.md)]

During planned maintenance, Azure SignalR Service uses a graceful shutdown strategy to reduce the impact on connected clients. Connections are gradually disconnected over a set time window, allowing clients to reconnect progressively rather than all at once. For more information, see [Disconnections during service maintenance](/azure/azure-signalr/signalr-concept-client-disconnections#disconnections-during-service-maintenance).

Maintenance events are surfaced to your clients as connection drops. Ensure that your client applications implement reconnection logic so they can recover from maintenance-related disconnections without user-visible interruption.

## Service-level agreement

[!INCLUDE [Service-level agreement](includes/reliability-service-level-agreement-include.md)]

## Related content

- [Reliability in Azure](./overview.md)
- [Azure SignalR Service overview](/azure/azure-signalr/signalr-overview)
- [Geo-replication in Azure SignalR Service](/azure/azure-signalr/howto-enable-geo-replication)
- [Resiliency and disaster recovery in Azure SignalR Service](/azure/azure-signalr/signalr-concept-disaster-recovery)
- [Understanding client disconnections and reconnection in Azure SignalR Service](/azure/azure-signalr/signalr-concept-client-disconnections)
