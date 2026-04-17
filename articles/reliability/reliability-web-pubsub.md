---
title: Reliability in Azure Web PubSub Service
description: Learn how to make Azure Web PubSub Service resilient to a variety of potential outages and problems, including transient faults, availability zone outages, and region outages.
author: glynnniall
ms.author: glynnniall
ms.topic: reliability-article
ms.custom: subject-reliability
ms.service: azure-web-pubsub
ms.date: 04/17/2026
---

# Reliability in Azure Web PubSub Service

[Azure Web PubSub Service](/azure/azure-web-pubsub/overview) is a fully managed, real-time messaging service that enables bi-directional communication between servers and clients by using the WebSocket protocol. A single Web PubSub resource can scale to one million concurrent WebSocket connections. The service supports several messaging patterns, including server-to-client broadcasting, messaging to named groups, client-to-client pub/sub, and AI token streaming.

[!INCLUDE [Shared responsibility description](includes/reliability-shared-responsibility-include.md)]

This article describes how to make Azure Web PubSub Service resilient to a variety of potential outages and problems, including transient faults, availability zone failures, and region-wide failures.

## Production deployment recommendations for reliability

For production workloads, we recommend that you:

> [!div class="checklist"]
>
> - Use the Premium_P1 or Premium_P2 tier. These tiers enable zone redundancy and geo-replication, both of which are unavailable in the Free and Standard_S1 tiers.
> - Enable geo-replication to protect against region-wide failures. Size each replica with enough units to handle your full expected traffic load during a failover event.
> - Design your client application to detect closed WebSocket connections and reconnect automatically, because zone failovers, region failovers, and transient faults all drop active connections.
> - If you use the client-server messaging pattern and don't use geo-replication, implement a custom multi-region solution by deploying separate Web PubSub resources in each region and implementing health-check-based negotiate logic in your application server.

## Reliability architecture overview

### Logical architecture

The resource you create is a *Web PubSub resource*, which has a globally unique endpoint such as `contoso.webpubsub.azure.com`. Clients establish WebSocket connections to this endpoint. Application servers connect to the same endpoint to send messages and receive events from clients.

Web PubSub Service supports two primary messaging patterns:

- **Client-server pattern:** Application servers push messages to clients, and clients send events to application servers through the service. Your application server controls message delivery and routing logic.
- **Client-client pattern:** Clients publish and subscribe to messages through the service directly, without involving an application server. The service handles group membership and message fan-out.

The messaging pattern you use affects which multi-region resiliency approaches are available. For more information, see [Resilience to region-wide failures](#resilience-to-region-wide-failures).

### Physical architecture

Azure Web PubSub Service manages WebSocket connection state and message routing across a set of compute nodes. Microsoft manages the underlying infrastructure. You don't directly see or interact with individual nodes.

When you use the Premium tier in a region that supports availability zones, the service automatically distributes compute nodes and connection state across all availability zones in the region. This distribution is transparent to you; no configuration is required.

## Resilience to transient faults

[!INCLUDE [Transient fault description - resilience](includes/reliability-transient-fault-description-include.md)]

WebSocket is a long-lived connection protocol. A connection to Azure Web PubSub Service may be dropped during transient network events, back-end node restarts, or service maintenance operations. Design your client applications to detect a closed connection and reconnect with exponential backoff.

The client SDKs for C#, JavaScript, Java, and Python include built-in reconnect handling. If you build a custom WebSocket client, add reconnect logic to your application code.

Azure Web PubSub Service exposes a health check endpoint at `https://<resource-name>.webpubsub.azure.com/api/health` that returns HTTP 200 when the service is healthy. Application servers can use this endpoint to monitor service health and to decide which Web PubSub endpoint to return during the WebSocket negotiate step.

## Resilience to availability zone failures

[!INCLUDE [Resilience to availability zone failures](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)]

Zone redundancy is available only in the Premium_P1 and Premium_P2 tiers. When you create or upgrade a Web PubSub resource to one of these tiers in a region that supports availability zones, zone redundancy is automatically enabled. The service distributes its compute nodes across all availability zones in the region. If one zone fails, the service routes traffic to nodes in the healthy zones.

### Requirements

- **Region support:** Zone-redundant Azure Web PubSub Service resources can be deployed into [any region that supports availability zones](./regions-list.md).

- **SKU requirements:** You must use the Premium_P1 or Premium_P2 tier. Zone redundancy is not available in the Free or Standard_S1 tiers. Standard_S1 resources can be upgraded to Premium_P1 without service downtime.

### Considerations

- During a zone failure, active WebSocket connections routed through nodes in the affected zone are dropped. Clients must reconnect. If you design your client applications to reconnect automatically, as described in [Resilience to transient faults](#resilience-to-transient-faults), the disruption to end users is minimal.

- Scaling your Web PubSub resource from Free to Standard, or from Free or Standard to Premium, changes its public IP address. This IP change can take 30 to 60 minutes to propagate through DNS, during which the service is temporarily unreachable. Plan tier changes during maintenance windows and avoid frequent tier changes.

### Cost

Zone redundancy is included in the Premium tier at no additional cost. For more information, see [Azure Web PubSub service pricing](https://azure.microsoft.com/pricing/details/web-pubsub/).

### Configure availability zone support

When you create a Web PubSub resource in the Premium_P1 or Premium_P2 tier in a supported region, zone redundancy is automatically enabled. No further configuration is required.

To upgrade an existing Standard_S1 resource to Premium_P1, see [Scale an Azure Web PubSub Service instance](/azure/azure-web-pubsub/howto-scale-manual-scale). This upgrade does not cause service downtime.

### Capacity planning and management

Azure Web PubSub Service unit capacity applies to the resource as a whole, not to individual zones. When a zone fails and connections from the affected zone reconnect to nodes in the remaining zones, check that the combined connection load remains within the resource's unit capacity. If you run close to maximum capacity under normal conditions, consider adding units to ensure the resource can absorb the reconnection surge that occurs after a zone failure.

### Behavior when all zones are healthy

- **Cross-zone operation:** Azure Web PubSub Service operates in an active-active model across availability zones. Compute nodes in all availability zones are active and accept connections. Incoming connections are distributed across nodes in all zones.

- **Cross-zone data replication:** Azure Web PubSub Service is a message relay. Messages are delivered to connected clients in real time and are not persisted. Connection state is distributed across zones so that the service can route messages to the correct connections regardless of which zone handles a request.

### Behavior during a zone failure

- **Detection and response:** The Azure Web PubSub Service platform is responsible for detecting a failure in an availability zone. You don't need to take any action to initiate a zone failover.

[!INCLUDE [Availability zone down notification (Service Health only)](./includes/reliability-availability-zone-down-notification-service-include.md)]

- **Active requests:** When a zone is unavailable, WebSocket connections on nodes in that zone are terminated. Clients must reconnect. Ensure your client applications handle connection drops and reconnect automatically, as described in [Resilience to transient faults](#resilience-to-transient-faults).

- **Expected data loss:** Azure Web PubSub Service does not persist messages. Messages that were in flight to connections on nodes in the unavailable zone at the time of the failure may be lost.

- **Expected downtime:** Active connections in the affected zone are dropped. The reconnect typically takes a few seconds. Clients that implement reconnect logic experience minimal disruption.

- **Traffic rerouting:** The Azure Web PubSub Service platform detects the zone failure and stops routing new connections to nodes in the affected zone. New and reconnecting clients connect to nodes in the healthy zones.

### Zone recovery

When the availability zone recovers, Azure Web PubSub Service automatically restores nodes in that zone. You don't need to take any action. New connections may be routed to nodes in the recovered zone.

### Test for zone failures

Azure Web PubSub Service manages traffic routing, failover, and zone recovery automatically for zone-redundant Premium tier resources. You don't need to initiate anything. Because zone redundancy is fully managed, you don't need to validate availability zone failure processes.

## Resilience to region-wide failures

Azure Web PubSub Service is a single-region service. If the region becomes unavailable, your Web PubSub resource is also unavailable.

To protect your application against a region-wide failure, you can use *geo-replication* (a managed multi-region feature available in the Premium tier), or you can build a custom multi-region solution using separate Web PubSub resources.

### Geo-replication

Geo-replication lets you add replicas of your Web PubSub resource in other Azure regions. All replicas share a single endpoint (`contoso.webpubsub.azure.com`). Behind this endpoint, Azure Traffic Manager performs DNS-based routing to direct each client to the nearest healthy regional replica. If a region fails, the Traffic Manager detects the failure through health checks and stops directing clients to that replica. After the DNS TTL of 90 seconds, clients that reconnect are routed to the nearest healthy replica.

Geo-replication is a Premium tier feature. For the client-client pub/sub pattern, geo-replication is the only supported approach for cross-region resiliency. For the client-server pattern, geo-replication and the custom multi-region approach are both options.

#### Requirements

- **SKU requirements:** You must use the Premium_P1 or Premium_P2 tier to enable geo-replication.
- **Region support:** You can add replicas in any Azure region.
- **Replica limit:** Each primary Web PubSub resource supports up to 8 replicas.

#### Considerations

- Replicas share most configuration with the primary resource. The following configurations must be set separately on each replica:
  - SKU name and unit count (and autoscale rules, if used)
  - Shared private endpoint approvals
  - Log destination settings
  - Alerts

- After a regional failover, only new connections and reconnection attempts are routed to healthy replicas after the 90-second DNS TTL elapses. Active connections that are already established are not affected until they disconnect.

- After the failed region recovers, Traffic Manager adds it back to DNS resolution. Connected clients are not disrupted. New connections are again routed to the recovered region when it is the nearest healthy option.

- If you use the client-client (pub/sub) pattern, geo-replication is required for cross-region resiliency. A custom multi-region approach does not support zero-downtime failover for this pattern.

#### Cost

Each replica is billed separately based on its own unit count and outbound message volume. If a message is transferred between replicas and then delivered to a client or server in another region, it is billed as an outbound message. For more information, see [Azure Web PubSub service pricing](https://azure.microsoft.com/pricing/details/web-pubsub/).

#### Configure geo-replication

To add a replica to a Web PubSub resource, see [Geo-replication in Azure Web PubSub](/azure/azure-web-pubsub/howto-enable-geo-replication). Instructions are provided for the Azure portal, Azure CLI, and Bicep.

Geo-replication is not available on Free or Standard_S1 tier resources. To add geo-replication to an existing Standard_S1 resource, first upgrade it to Premium_P1.

#### Capacity planning and management

Each replica handles traffic independently. During a regional failover, clients from the failed region reconnect to the nearest healthy replica. To ensure that the surviving replicas have enough capacity to absorb this additional load, configure each replica with units that can handle the full expected traffic of the workload, not just the portion it normally serves.

Alternatively, enable autoscaling on each replica so units can scale out automatically in response to higher load. For more information about autoscaling, see [Automatically scale units of an Azure Web PubSub service](/azure/azure-web-pubsub/howto-scale-autoscale).

For general guidance on overprovisioning as a strategy, see [Manage capacity by overprovisioning](/azure/reliability/concept-redundancy-replication-backup#manage-capacity-with-over-provisioning).

#### Behavior when all regions are healthy

- **Cross-region operation:** Azure Traffic Manager routes each client to the nearest healthy regional replica. Clients in different geographic areas connect to different replicas. Web PubSub Service synchronizes messages across replicas so that clients connected to any replica can communicate with each other.

- **Cross-region data replication:** When a message is sent to a replica, the service transfers that message to other replicas so that clients connected elsewhere can receive it. The synchronization overhead is minimal for most common messaging patterns, such as broadcasting to large groups or messaging a single connection. Messaging to small groups (fewer than 10 members) may produce a slightly higher synchronization overhead. Azure Web PubSub Service does not persist messages; only active delivery is synchronized across replicas.

#### Behavior during a region failure

- **Detection and response:** Azure Traffic Manager performs health checks against each replica. When a regional outage causes a replica to fail its health check, the Traffic Manager removes that replica's endpoint from its DNS resolution results. You don't need to initiate anything.

[!INCLUDE [Region down notification (Service Health only)](./includes/reliability-region-down-notification-service-include.md)]

- **Active requests:** Active WebSocket connections to the replica in the failed region are dropped. Clients must reconnect. After the DNS TTL of 90 seconds elapses and DNS records are updated, reconnecting clients are automatically routed to the nearest healthy replica.

- **Expected data loss:** Messages that were in flight to clients connected to the failed replica and not yet delivered at the time of failure may be lost.

- **Expected downtime:** After Traffic Manager detects the failure and removes the failed replica from DNS resolution, clients that reconnect are routed to a healthy replica. This process takes up to the DNS TTL of 90 seconds. Well-designed clients that implement reconnect logic can resume normal operation after reconnecting to the healthy replica.

- **Traffic rerouting:** After the DNS TTL period, Azure Traffic Manager excludes the failed replica from DNS resolution. Clients that reconnect are directed to the nearest healthy replica.

#### Region recovery

When the failed region recovers, the Traffic Manager health check detects the restored replica and includes its endpoint in DNS resolution again. Clients currently connected to other replicas are not affected and remain connected until they disconnect. New connections are again routed to the recovered region's replica when it is the nearest healthy option.

#### Test for region failures

To simulate a regional failover and test your client application's reconnect behavior, you can disable public network access on a replica (or use access control rules to deny all traffic) and restart the resource. For detailed steps, see [Resiliency and disaster recovery in Azure Web PubSub Service](/azure/azure-web-pubsub/concept-disaster-recovery#how-to-test-a-failover).

### Custom multi-region solutions for resiliency

If you use the client-server messaging pattern and need more control over failover timing—or if you don't want to use geo-replication—you can build a custom multi-region solution.

In a custom multi-region setup, you deploy separate Web PubSub resources in two or more regions, each paired with an application server in the same region. Your application server controls which Web PubSub endpoint it returns during the WebSocket negotiate step, and it checks service health to decide which endpoint is healthy.

> [!NOTE]
> A custom multi-region solution does not support zero-downtime failover for the client-client (pub/sub) messaging pattern. For that pattern, use geo-replication.

Key responsibilities when you build a custom multi-region solution:

- **Health monitoring:** Periodically poll the health check endpoint (`https://<resource-name>.webpubsub.azure.com/api/health`) for each Web PubSub resource to detect when a primary or secondary resource is unavailable.

- **Negotiate logic:** During the WebSocket negotiate step, your application server returns the endpoint of the healthy primary Web PubSub resource. When the primary is unavailable, return the endpoint of a healthy secondary resource so that clients can connect there instead.

- **Broadcast logic:** Your application server broadcasts messages to all healthy Web PubSub endpoints, because clients are distributed across multiple resources.

- **Client reconnection:** When a resource in one region becomes unavailable, active WebSocket connections to that resource are dropped. You must implement reconnect logic in your clients. When a client reconnects and negotiates with your application server, the server returns the healthy secondary endpoint.

Both active/passive (one active resource, one standby backup) and active/active (all resources receiving traffic and serving as backup for each other) patterns are supported.

For a detailed architecture overview, failover sequence, and testing guidance, see [Resiliency and disaster recovery in Azure Web PubSub Service](/azure/azure-web-pubsub/concept-disaster-recovery).

## Resilience to service maintenance

[!INCLUDE [Service maintenance (no special callouts)](includes/reliability-maintenance-include.md)]

## Service-level agreement

[!INCLUDE [Service-level agreement](includes/reliability-service-level-agreement-include.md)]

The SLA applies to Standard_S1 and Premium tier Web PubSub resources. The Free tier has no SLA. The SLA percentage increases when zone redundancy is enabled, which is automatically the case for Premium tier resources deployed in regions that support availability zones. For more information, see [SLAs for online services](https://www.microsoft.com/licensing/docs/view/Service-Level-Agreements-SLA-for-Online-Services).

## Related content

- [What is Azure Web PubSub service?](/azure/azure-web-pubsub/overview)
- [Geo-replication in Azure Web PubSub](/azure/azure-web-pubsub/howto-enable-geo-replication)
- [Resiliency and disaster recovery in Azure Web PubSub Service](/azure/azure-web-pubsub/concept-disaster-recovery)
- [Availability zones support in Azure Web PubSub Service](/azure/azure-web-pubsub/concept-availability-zones)
- [Scale an Azure Web PubSub Service instance](/azure/azure-web-pubsub/howto-scale-manual-scale)
- [Automatically scale units of an Azure Web PubSub service](/azure/azure-web-pubsub/howto-scale-autoscale)
- [Reliability in Azure](/azure/reliability/overview)
