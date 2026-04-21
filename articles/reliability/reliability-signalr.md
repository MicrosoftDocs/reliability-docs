---
title: Reliability in Azure SignalR Service
description: Learn how to make Azure SignalR Service resilient to a variety of potential outages and problems, including transient faults, availability zone outages, region outages, and service maintenance.
author: glynnniall
ms.author: pnp
ms.topic: reliability-article
ms.custom: subject-reliability
ms.service: azure-signalr-service
ms.date: 04/20/2026
ai-usage: ai-assisted
---

# Reliability in Azure SignalR Service

[Azure SignalR Service](/azure/azure-signalr/signalr-overview) is a fully managed service that enables real-time bidirectional communication in web and mobile applications. It's built on ASP.NET Core SignalR, which abstracts the underlying transport mechanism. When WebSockets are available, the service uses them. When they aren't, it falls back to Server-Sent Events or Long Polling. This abstraction means your client and server code can communicate in real time without being tied to a specific transport protocol. The service supports a server-based default mode and a serverless mode that integrates with Azure Functions.

[!INCLUDE [Shared responsibility](includes/reliability-shared-responsibility-include.md)]

This article describes how to make Azure SignalR Service resilient to a variety of potential outages and problems, including transient faults, availability zone outages, region outages, and service maintenance. It also highlights key information about the Azure SignalR Service service-level agreement (SLA).

> [!IMPORTANT]
> In Default mode, the reliability of your app servers determines whether clients can connect and communicate. Plan the reliability of your app servers alongside your Azure SignalR Service configuration. To match the resiliency of your Azure SignalR Service resource, deploy your app servers with the same level of redundancy—for example, in availability zones or across multiple regions.

## Production deployment recommendations for reliability

For production workloads, we recommend that you:

> [!div class="checklist"]
> - Use the Premium tier to access availability zone support and geo-replication.
> - [Enable geo-replication](#geo-replication) to distribute your service across multiple Azure regions and improve regional resiliency.
> - Size each replica to handle your full expected traffic load so that the service can maintain capacity during a failover.
> - Design client applications and app servers to reconnect automatically when connections drop.

## Reliability architecture overview

In Default mode, you deploy *app servers* that act as SignalR hubs. Your app servers connect to the Azure SignalR Service resource and contain your hub logic. Clients connect to the Azure SignalR Service resource, which routes messages between clients and app servers. The reliability of your app servers directly affects whether clients can establish and maintain connections.

In Serverless mode, the service integrates with Azure Functions. Azure Functions act as event-driven message handlers, and you don't manage app servers directly.

Internally, the service distributes its compute capacity across multiple nodes. The platform manages these nodes directly, including creation, health monitoring, and replacement of unhealthy nodes. You don't see or manage these nodes.

## Resilience to transient faults

[!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)]

Azure SignalR Service uses long-lived connections between clients, app servers, and the service. These connections can be dropped due to transient faults such as network instability, load balancer reconfigurations, or browser tab suspensions. Design your client applications and app servers to handle connection drops and reconnect automatically.

The Azure SignalR Service SDKs include built-in reconnection handling for server connections to the service. Client-side reconnection depends on the client library you use. ASP.NET Core SignalR clients support stateful reconnect, which allows a client to resume its previous connection without losing state if it reconnects quickly using the same connection ID. If the reconnection results in a new connection ID—for example, after a longer outage—the service treats the client as a new connection. In this case, the client needs to rejoin any groups it was previously in and restore any session state.

For detailed guidance on designing your application to handle client disconnections and reconnections, see [Understanding client disconnections and reconnection in Azure SignalR Service](/azure/azure-signalr/signalr-concept-client-disconnections).

## Resilience to availability zone failures

[!INCLUDE [Resilience to availability zone failures](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)]

Azure SignalR Service supports zone-redundant deployments in the Premium tier. When you create a Premium tier resource in a region that supports availability zones, Azure SignalR Service automatically distributes its compute capacity evenly across all availability zones in the region. If an availability zone fails, Azure SignalR Service routes new connections to instances in the remaining healthy zones.

### Requirements

- **Region support:** Zone-redundant Azure SignalR Service resources can be deployed into [any region that supports availability zones](/azure/reliability/regions-list).

- **Tier requirement:** Zone redundancy is available only in the Premium tier. Standard tier resources don't support availability zones.

### Considerations

During an availability zone outage, connections that were using instances in the affected zone are terminated. Clients and app servers need to reconnect. The service continues to operate and accept new connections through instances in the healthy zones.

Standard tier resources don't support availability zones. If you need zone redundancy, use the Premium tier. You can upgrade an existing Standard tier resource to the Premium tier without downtime; zone redundancy is automatically enabled after the upgrade.

### Cost

Zone redundancy is included in the Premium tier. There's no separate charge for zone redundancy beyond the Premium tier cost. Each Premium tier unit has its own billing. For more information, see [Azure SignalR Service pricing](https://azure.microsoft.com/pricing/details/signalr-service/).

### Configure availability zone support

When you create an Azure SignalR Service resource using the Premium tier in a region that supports availability zones, zone redundancy is automatically enabled. No other configuration is required.

If you have an existing Standard tier resource, upgrade it to the Premium tier. Zone redundancy is automatically enabled after the upgrade. For more information, see [Change your Azure SignalR Service tier](/azure/azure-signalr/signalr-howto-scale-signalr).

### Behavior when all zones are healthy

This section describes what to expect when an Azure SignalR Service resource is configured for zone redundancy and all availability zones are operational.

- **Cross-zone operation:** Connections are distributed across instances in all availability zones in the region. A client or app server connection can be routed to an instance in any zone. The service routes messages between instances across zones automatically, so a message sent by a client in one zone is delivered to clients connected in any other zone.

- **Cross-zone data replication:** Because Azure SignalR Service doesn't store persistent data, there's no data to replicate between zones. Connection state is ephemeral and is associated with each active connection.

### Behavior during a zone failure

This section describes what to expect when there's an outage in one of the availability zones.

- **Detection and response:** The Azure SignalR Service platform detects the zone failure and routes new connections to instances in the healthy zones. You don't need to do anything to initiate a zone failover.

[!INCLUDE [Availability zone down notification (Service Health only)](./includes/reliability-availability-zone-down-notification-service-include.md)]

- **Active requests:** When an availability zone becomes unavailable, any active WebSocket connections routed through instances in that zone are terminated. Clients and app servers must reconnect. Ensure that your client applications and app servers implement automatic reconnection logic to handle this.

- **Expected data loss:** Zone failures aren't expected to cause persistent data loss because Azure SignalR Service doesn't store data. Messages that were in transit at the time of the failure might not be delivered to their recipients.

- **Expected downtime:** Connections using instances in the affected zone experience a brief interruption. Design your client applications and app servers to handle dropped connections by following the [transient fault handling guidance](#resilience-to-transient-faults).

- **Traffic rerouting:** After a zone failure, the platform routes new connections to instances in the remaining healthy zones. The service continues to operate with reduced capacity until the zone recovers.

### Zone recovery

When the availability zone recovers, Azure SignalR Service automatically restores instances in that zone and redistributes connections across all zones as normal.

### Test for zone failures

The Azure SignalR Service platform manages traffic routing, failover, and zone recovery for zone-redundant resources. Because this feature is fully managed, you don't need to validate zone failure processes.

## Resilience to region-wide failures

Azure SignalR Service is a single-region service. If the region becomes unavailable, your Azure SignalR Service resource is also unavailable. Azure SignalR Service supports two approaches for regional resiliency: geo-replication, which is managed by the platform, and custom multi-region solutions using the service SDK.

### Geo-replication

With geo-replication, you create replicas of your Azure SignalR Service resource in other Azure regions. Azure Traffic Manager routes clients to the nearest healthy replica based on the client's location. If a region becomes unavailable, Traffic Manager detects the failure through health checks and automatically reroutes new and reconnecting clients to a healthy replica in another region after a DNS time-to-live (TTL) period of 90 seconds.

#### Requirements

- **Tier requirement:** Geo-replication requires the Premium tier.

- **Replica count:** You can create up to eight replicas per primary resource, in any combination of Azure regions.

#### Considerations

Replicas inherit most configuration from the primary resource. The following settings aren't inherited and must be configured separately on each replica: SKU and unit size, autoscale settings, shared private endpoint approvals, log destination settings, and alert rules.

When a region becomes unavailable, clients connected to the replica in that region experience connection drops. After the DNS cache expires (up to 90 seconds), those clients reconnect to a healthy replica in another region. Implement client-side reconnection logic to handle this interruption. Existing connections to healthy replicas are unaffected during a regional outage.

Geo-replication handles failover for Azure SignalR Service. The reliability of your app servers in each region is a separate concern. If a region's app servers become unavailable, clients can still connect to other replicas, but the hub logic in that region is inaccessible.

#### Cost

Each replica is billed separately according to its own unit size and outbound traffic. Cross-region egress fees apply when messages are transferred between replicas and delivered to clients or servers in another region. For more information, see [Azure SignalR Service pricing](https://azure.microsoft.com/pricing/details/signalr-service/).

#### Configure geo-replication

For information about how to create and manage replicas, see [Geo-replication in Azure SignalR Service](/azure/azure-signalr/howto-enable-geo-replication).

#### Capacity planning and management

Size each replica to handle your full expected traffic load. This approach allows the service to continue operating at full capacity if another replica becomes unavailable due to a regional outage. As an alternative, enable autoscaling on each replica so that the service can scale out automatically when it receives additional traffic after a failover. For more information, see [Autoscaling in Azure SignalR Service](/azure/azure-signalr/signalr-howto-scale-autoscale).

#### Behavior when all regions are healthy

This section describes what to expect when you configure geo-replication and all replicas are operational.

- **Cross-region operation:** Traffic Manager resolves the Azure SignalR Service fully qualified domain name (FQDN) to the nearest healthy replica based on the client's geographic location. All replicas accept connections simultaneously (active/active).

- **Cross-region data replication:** Replicas synchronize messaging data with each other. Messages sent to one replica are forwarded to other replicas when necessary, so a message from a client in one region is delivered to clients connected in other regions.

#### Behavior during a region failure

This section describes what to expect when the region hosting a replica becomes unavailable.

- **Detection and response:** Azure Traffic Manager monitors the health of each replica. When a replica fails health checks, Traffic Manager excludes it from DNS resolution. Failover is automatic and requires no action from you.

[!INCLUDE [Region down notification (Service Health only)](./includes/reliability-region-down-notification-service-include.md)]

- **Active requests:** Clients connected to the replica in the failed region experience a connection drop. After the DNS TTL expires (up to 90 seconds), reconnecting clients are directed to a healthy replica in another region. Implement client-side reconnection logic to handle this interruption.

- **Expected data loss:** Because Azure SignalR Service doesn't store message history, persistent data loss isn't expected. Messages that were in transit within the failed region at the time of the outage might not be delivered.

- **Expected downtime:** Clients affected by the failed region experience a disruption of up to 90 seconds while the DNS cache expires and reconnection occurs. Clients with automatic reconnection logic reconnect to a healthy replica after this period.

- **Traffic rerouting:** Traffic Manager updates DNS resolution to route new connections and reconnecting clients to the nearest healthy replica.

#### Region recovery

When the failed region comes back online, Traffic Manager detects that the replica passes health checks and includes it again in DNS resolution. New clients are routed to the recovered replica based on proximity. Clients with existing connections to other replicas remain on those replicas until they disconnect.

#### Test for region failures

To test geo-replication failover, disable the endpoint for a replica in the Azure portal. Disabling a replica's endpoint removes it from Traffic Manager's DNS resolution. After the DNS TTL expires, clients reconnect to another region. Re-enable the endpoint to restore the replica. For more information, see [Geo-replication in Azure SignalR Service](/azure/azure-signalr/howto-enable-geo-replication).

### Custom multi-region solutions for resiliency

If geo-replication doesn't meet your needs, or if you're using the Standard tier, you can configure multiple Azure SignalR Service instances in different regions and use the Azure SignalR Service SDK to manage failover between them.

In this approach, you configure your app servers or Azure Functions to connect to multiple Azure SignalR Service instances, designating each instance as either primary or secondary. The SDK automatically routes negotiate requests to an available primary instance. If all primary instances are unavailable, the SDK falls back to secondary instances. When a primary instance recovers, the SDK routes new clients back to it.

This approach requires you to manage the deployment and reliability of your app servers across regions. You're responsible for routing traffic to healthy app servers when a region becomes unavailable. Clients experience connection drops during a regional failover and must reconnect.

For more information, see [Resiliency and disaster recovery in Azure SignalR Service](/azure/azure-signalr/signalr-concept-disaster-recovery).

## Resilience to service maintenance

The Azure SignalR Service team regularly performs maintenance for improvements in performance, reliability, security, and features. During planned maintenance, Azure SignalR Service uses a graceful shutdown strategy to reduce the impact on connected clients. Connections are gradually disconnected over a set time window, allowing clients to reconnect progressively rather than all at once.

Maintenance events are surfaced to your clients as connection drops. Ensure that your client applications implement reconnection logic so they can recover from maintenance-related disconnections without user-visible interruption. For more information, see [Understanding client disconnections and reconnection in Azure SignalR Service](/azure/azure-signalr/signalr-concept-client-disconnections#disconnections-during-service-maintenance).

## Service-level agreement

[!INCLUDE [Service-level agreement](includes/reliability-service-level-agreement-include.md)]

The SLA for Azure SignalR Service varies by tier. The Premium tier provides a higher availability commitment than the Standard tier. The Free tier has no SLA guarantee.

For more information, see the [SLA for Azure SignalR Service](https://azure.microsoft.com/support/legal/sla/signalr-service/).

## Related content

- [Reliability in Azure](./overview.md)
- [Azure SignalR Service overview](/azure/azure-signalr/signalr-overview)
- [Geo-replication in Azure SignalR Service](/azure/azure-signalr/howto-enable-geo-replication)
- [Resiliency and disaster recovery in Azure SignalR Service](/azure/azure-signalr/signalr-concept-disaster-recovery)
- [Availability zones support in Azure SignalR Service](/azure/azure-signalr/availability-zones)
- [Understanding client disconnections and reconnection in Azure SignalR Service](/azure/azure-signalr/signalr-concept-client-disconnections)
