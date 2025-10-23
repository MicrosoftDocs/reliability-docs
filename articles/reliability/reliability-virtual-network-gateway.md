---
title: Reliability in Azure Virtual Network Gateway
description: Find out about reliability in Azure Virtual Network Gateway, including availability zones and multi-region deployments.
ms.author: anaharris
author: anaharris-ms
ms.topic: reliability-article
ms.custom: subject-reliability
ms.service: azure-vpn-gateway
ms.date: 10/23/2025
zone_pivot_groups: virtual-network-gateway-types
ai-usage: ai-assisted
---

# Reliability in Azure virtual network gateway

A virtual network gateway is the component you deploy into your Azure virtual network to enable private connectivity to Microsoft resources through ExpressRoute or a VPN. This article describes reliability support in virtual network gateways, including transient fault handling, availability zone support, and multi-region deployments.

::: zone pivot="expressroute"

> [!IMPORTANT]
> This article covers the reliability of ExpressRoute virtual network gateways, which are the Azure-based parts of the ExpressRoute system.
>
> When you use ExpressRoute, it's critical that you design your whole ExpressRoute deployment to meet your resiliency requirements. Typically, this requires that you use multiple sites (peering locations) and that your on-premises components are configured to enable high availability and fast failover. For more information, see [Design and architect Azure ExpressRoute for resiliency](../expressroute/design-architecture-for-resiliency.md).

::: zone-end

::: zone pivot="vpn"

> [!IMPORTANT]
> This article covers the reliability of virtual network gateways, which are the Azure-based parts of the Azure VPN Gateway service.
>
> When you use VPNs, it's critical that you design your whole network architecture to meet your resiliency requirements. You're responsible for managing the reliability of your side of the VPN connection, including client devices for point-to-site configurations and remote VPN devices for site-to-site configurations. For guidance about configuring your infrastructure for high availability, see [Design highly available gateway connectivity for cross-premises and VNet-to-VNet connections](../vpn-gateway/vpn-gateway-highlyavailable.md).

::: zone-end

[!INCLUDE [Shared responsibility description](includes/reliability-shared-responsibility-include.md)]

## Production deployment recommendations

::: zone pivot="expressroute"

To learn about how to deploy Azure ExpressRoute to support your solution's reliability requirements and how reliability affects other aspects of your architecture, see [Architecture best practices for Azure ExpressRoute in the Azure Well-Architected Framework](/azure/well-architected/service-guides/azure-expressroute).

::: zone-end

::: zone pivot="vpn"

To ensure high reliability for your production virtual network gateways, we recommend that you:

> [!div class="checklist"]
> - **Enable zone redundancy** if your Azure VPN Gateway resources are in a supported region. Deploy VPN Gateway using supported SKUs (VpnGw1AZ or higher) to ensure access to zone redundancy features.
> - **Use Standard SKU public IP addresses.**
> - **Configure active-active mode** for higher availability, when supported by your remote VPN devices.
> - **Implement proper monitoring** using [Azure Monitor VPN Gateway metrics](../vpn-gateway/monitor-vpn-gateway.md).

::: zone-end

## Reliability architecture overview

::: zone pivot="expressroute"

ExpressRoute requires components to be deployed in the on-premises environment, peering locations, and within Azure:

- *Circuits and connections*: An ExpressRoute *circuit* consists of two *connections* through a single peering location to the Microsoft Enterprise Edge. By using two connections, you can achieve active-active connectivity. However, this configuration doesn't protect against site-level failures.

- *Customer premises equipment* (CPE) includes your edge routers and client devices. You need to ensure that your CPE is designed to be resilient to problems, and that it can quickly recover when problems happen in other parts of your ExpressRoute infrastructure. For more information, see [Design and architect Azure ExpressRoute for resiliency](../expressroute/design-architecture-for-resiliency.md).

- *Sites:* Circuits are established through a *site*, which is a physical peering location. Sites are designed to be highly available and have built-in redundancy across all layers, but because they represent a single physical location, there is a possibility of sites having problems. To mitigate the risk of site outages, ExpressRoute offers different site resiliency options that vary in their level of protection. For more information, see [Design and architect Azure ExpressRoute for resiliency - Site resiliency for ExpressRoute](../expressroute/design-architecture-for-resiliency.md#site-resiliency-for-expressroute).

- *Azure virtual network gateway:* In Azure, you create a gateway that acts as the termination point for one or more ExpressRoute circuits. You deploy a gateway within your virtual network, so it's sometimes called a *virtual network gateway*.

This diagram shows two different ExpressRoute configurations, each with a single virtual network gateway, configured for different levels of resiliency across sites:

:::image type="content" source="../expressroute/media/design-architecture-for-resiliency/standard-vs-maximum-resiliency.png" alt-text="Diagram illustrating ExpressRoute connection options between on-premises network and Azure, showing different resiliency levels." border="false":::

::: zone-end

::: zone pivot="vpn"

A VPN requires components to be deployed in both the on-premises environment and within Azure:

- *On-premises components*: The components you deploy depend on whether you deploy a point-to-site or site-to-site configuration. To learn more about the differences, see [VPN Gateway topology and design](../vpn-gateway/design.md).

   - Site-to-site configurations require an on-premises VPN device, which you're responsible for deploying, configuring, and managing.
   - Point-to-site configurations require you to deploy a VPN client application in a remote desktop and associate the configuration to each client that connects to the VPN. You're responsible for deploying and configuring the client devices.

- *Azure virtual network gateway*: In Azure, you create a *VPN gateway*, also called a *virtual network gateway*, which acts as the termination point for VPN connections.

- *Local network gateway:* A site-to-site VPN configuration also requires a local network gateway, which represents the remote VPN device. Microsoft manages the resiliency of a local network gateway.

The following diagram illustrates some key components in a VPN that connects from an on-premises environment to Azure:

:::image type="content" source="media/reliability-virtual-network-gateway/vpn-reliability-architecture.png" alt-text="Diagram that shows Azure VPN Gateway, on-premises site-to-site, and point-to-site networks." border="false":::

::: zone-end

### Virtual network gateway

::: zone pivot="expressroute"

An ExpressRoute gateway contains two or more *instances*, which represent virtual machines (VMs) that your gateway uses to process ExpressRoute traffic.

::: zone-end

::: zone pivot="vpn"

A VPN virtual network gateway contains exactly two *instances*, which represent virtual machines (VMs) that your gateway uses to process VPN traffic.

::: zone-end

You don't see or manage the VMs directly.  The platform automatically manages instance creation, health monitoring, and the replacement of unhealthy instances. To achieve protection against server and server rack failures, Azure automatically distributes gateway instances across multiple fault domains within a region.

::: zone pivot="expressroute"

You configure the gateway SKU. Each SKU supports a different level of throughput, and different numbers of circuits. For more information, see [About ExpressRoute virtual network gateways](../expressroute/expressroute-about-virtual-network-gateways.md).

A gateway runs in *active-active* mode by default, which supports high availability of your connection. You can optionally switch to use *active-passive* mode, but this configuration increases the risk of a failure affecting your connectivity. For more information, see [Design highly available gateway connectivity for cross-premises and VNet-to-VNet connections](../vpn-gateway/vpn-gateway-highlyavailable.md).

Ordinarily, traffic is routed through your virtual network gateway. However, if you use [FastPath](../expressroute/about-fastpath.md), traffic from your on-premises environment bypasses the gateway, which improves throughput and reduces latency.

::: zone-end

::: zone pivot="vpn"

You configure the gateway SKU. Each SKU supports a different level of throughput, and different numbers of VPN connections. For more information, see [About gateway SKUs](../vpn-gateway/about-gateway-skus.md).

Depending on your high availability requirements, you can configure your gateway as *active-standby*, which means that one instance processes traffic and the other is a standby instance, or as *active-active*, which means that both instances process traffic. Active-active isn't always possible due to the asymmetric nature of connection flows. For more information, see [Design highly available gateway connectivity for cross-premises and VNet-to-VNet connections](../vpn-gateway/vpn-gateway-highlyavailable.md).

::: zone-end

You can protect against availability zone failures by distributing gateway instances across multiple zones, providing automatic failover within the region, and maintaining connectivity during zone maintenance or outages. For more information, see [Availability zone support](#availability-zone-support).

## Transient faults

[!INCLUDE [Transient fault description](includes/reliability-transient-fault-description-include.md)]

For applications that connect through a virtual network gateway, implement retry logic with exponential backoff to handle potential transient connection problems. The stateful nature of virtual network gateways ensures that legitimate connections are maintained during brief network interruptions.

In a distributed networking environment, transient faults can occur at multiple layers, including:

- In your on-premises environment.

::: zone pivot="expressroute"

- In an edge site.

::: zone-end

::: zone pivot="vpn"

- In the internet.

::: zone-end

- Within Azure.

::: zone pivot="expressroute"

ExpressRoute reduces the effect of transient faults by using redundant connection paths, fast fault detection, and automated failover. However, it's important that your applications and on-premises components are configured correctly to be resilient to a variety of issues. For comprehensive fault handling strategies, see [Designing for high availability with ExpressRoute](../expressroute/designing-for-high-availability-with-expressroute.md).

::: zone-end

::: zone pivot="vpn"

Data traffic, such as TCP flows, transits through IPsec tunnels. Transient faults can impact IPsec tunnels or TCP data flows. In the event of a disconnection, IKE (Internet Key Exchange) renegotiates the Security Associations (SAs) for both Phase 1 and Phase 2 to re-establish the IPsec tunnel.

::: zone-end

## Availability zone support

[!INCLUDE [AZ support description](includes/reliability-availability-zone-description-include.md)]

Virtual network gateways are automatically *zone-redundant* when they meet the requirements. Zone redundancy means that the gateway's instances distributed across three availability zones. This configuration eliminates any single zone as a point of failure and provides the highest level of zone resiliency. Zone-redundant gateways provide automatic failover within the region, and maintaining connectivity during zone maintenance or outages.

All newly created gateways are zone-redundant, and zone redundancy is recommended for all production workloads.

The following diagram shows a zone-redundant virtual network gateway with two instances that are distributed across availability zones:

:::image type="content" source="media/reliability-virtual-network-gateway/zone-redundant.png" alt-text="Diagram that shows a virtual network gateway with two instances distributed across availability zones." border="false":::

::: zone pivot="expressroute"

> [!NOTE]
> There's no availability zone configuration for circuits or connections. These resources are located in network edge facilities, which aren't designed to use availability zones.

::: zone-end

### Region support

Zone-redundant virtual network gateways are available in [all regions that support availability zones](../reliability/availability-zones-region-support.md).

### Requirements

For a virtual network gateway to be zone-redundant, it must use a SKU that supports zone redundancy.

::: zone pivot="expressroute"

The following table shows which SKUs support zone redundancy:

[!INCLUDE [skus-with-az](../expressroute/includes/sku-availability-zones.md)]

::: zone-end

::: zone pivot="vpn"

All tiers of Azure VPN Gateway support zone redundancy except the Basic SKU, which is only for development environments. For more information about SKU options, see [About Gateway SKUs](../vpn-gateway/about-gateway-skus.md#workloads)

You must also use standard public IP addresses and configure them to be zone-redundant.

::: zone-end

### Cost

::: zone pivot="expressroute"

Zone-redundant gateways for ExpressRoute require specific SKUs, which have higher hourly rates compared to standard gateway SKUs due to their enhanced capabilities and performance characteristics. For pricing information, see [Azure ExpressRoute pricing](https://azure.microsoft.com/pricing/details/expressroute/).

::: zone-end

::: zone pivot="vpn"

There's no extra cost for a gateway deployed across multiple availability zones, as long as you use a supported SKU. For pricing information, see [VPN Gateway pricing](https://azure.microsoft.com/pricing/details/vpn-gateway/).

::: zone-end

### Configure availability zone support

This section explains how to configure zone redundancy for your virtual network gateways.

::: zone pivot="expressroute"

- **Create a new virtual network gateway with availability zone support.** Any new virtual network gateways you create are automatically zone-redundant, if they meet the requirements listed above. For detailed configuration steps, see [Create a zone-redundant virtual network gateway in availability zones](../vpn-gateway/create-zone-redundant-vnet-gateway.md?toc=%2Fazure%2Fexpressroute%2Ftoc.json).

::: zone-end

::: zone pivot="vpn"

- **Create a new virtual network gateway with availability zone support.** Any new virtual network gateways you create are automatically zone-redundant, if they meet the requirements listed above. For detailed configuration steps, see [Create a zone-redundant virtual network gateway in availability zones](../vpn-gateway/create-zone-redundant-vnet-gateway.md).

::: zone-end

::: zone pivot="expressroute"

- **Change the availability zone configuration of an existing virtual network gateway.** Virtual network gateways that you already created might not be zone-redundant. You can migrate a nonzonal gateway to a zone-redundant gateway with minimal downtime. For more information, see [Migrate ExpressRoute gateways to availability zone-enabled SKUs](../expressroute/gateway-migration.md).

::: zone-end

::: zone pivot="vpn"

- **Change the availability zone configuration of an existing virtual network gateway.** Virtual network gateways that you already created might not be zone-redundant. You can migrate a nonzonal gateway to a zone-redundant gateway with minimal downtime. For more information, see [About SKU consolidation & migration](../vpn-gateway/gateway-sku-consolidation.md).

::: zone-end

- **Verify the zone redundancy status of an existing virtual network gateway.** <!-- PG: Please confirm whether a customer can confirm whether an existing gateway is ZR, zonal, or nonzonal -->

### Normal operations

The following section describes what to expect when your virtual network gateway is configured for zone redundancy and all availability zones are operational.

::: zone pivot="expressroute"

- **Traffic routing between zones:** Traffic from your on-premises environment is distributed among instances in all of the zones that your gateway uses. This active-active configuration ensures optimal performance and load distribution under normal operating conditions.

    However, if you use FastPath for optimized performance, traffic from your on-premises environment bypasses the gateway, which improves throughput and reduces latency. For more information, see [About ExpressRoute FastPath](../expressroute/about-fastpath.md).

::: zone-end

::: zone pivot="vpn"

- **Traffic routing between zones:** VPN connections are routed to a primary VPN gateway instance in one zone, which is selected by Azure. The traffic routing behavior depends on whether you have configured active-standby or active-active mode, and doesn't depend on availability zone configuration.

::: zone-end

- **Data replication between zones:** No data replication occurs between zones because the virtual network gateway doesn't store persistent customer data.

- **Instance management:** The platform automatically manages instance placement across the zones that your gateway uses. Health monitoring ensures that only healthy instances receive traffic.

### Zone-down experience

The following section describes what to expect when your virtual network gateway is configured for zone redundancy and there's an availability zone outage.

- **Detection and response:** The Azure platform detects and responds to a failure in an availability zone. You don't need to initiate a zone failover.

::: zone pivot="expressroute"

- **Notification**: ExpressRoute doesn't notify you when a zone is down. However, you can use [Azure Resource Health](/azure/service-health/resource-health-overview) to monitor for the health of your gateway. You can also use [Azure Service Health](/azure/service-health/overview) to understand the overall health of the Azure ExpressRoute service, including any zone failures.

    Set up alerts on these services to receive notifications of zone-level problems. For more information, see [Create Service Health alerts in the Azure portal](/azure/service-health/alerts-activity-log-service-notifications-portal) and [Create and configure Resource Health alerts](/azure/service-health/resource-health-alert-arm-template-guide).

::: zone-end

::: zone pivot="vpn"

- **Notification**: Azure VPN Gateway doesn't notify you when a zone is down. However, you can use [Azure Resource Health](/azure/service-health/resource-health-overview) to monitor for the health of your gateway. You can also use [Azure Service Health](/azure/service-health/overview) to understand the overall health of the Azure VPN Gateway services, including any zone failures.

    Set up alerts on these services to receive notifications of zone-level problems. For more information, see [Create Service Health alerts in the Azure portal](/azure/service-health/alerts-activity-log-service-notifications-portal) and [Create and configure Resource Health alerts](/azure/service-health/resource-health-alert-arm-template-guide).

::: zone-end

- **Active requests:** Any active requests connected through gateway instances in the failing zone are terminated. Client applications should retry the requests by following the guidance for how to [handle transient faults](#transient-faults).

- **Expected data loss:** Zone failures aren't expected to cause data loss because virtual network gateways don't store persistent customer data.

- **Expected downtime:** During zone outages, connections might experience brief interruptions that typically last up to one minute as traffic is redistributed. Client applications should retry the requests by following the guidance for how to [handle transient faults](#transient-faults). <!-- PG: Please confirm -->

::: zone pivot="expressroute"

- **Traffic rerouting:** The platform automatically distributes traffic to instances in healthy zones.

    FastPath-enabled circuits maintain optimized routing throughout the failover process, ensuring minimal effect on application performance.

::: zone-end

::: zone pivot="vpn"

- **Traffic rerouting:** Traffic automatically reroutes to the other instance, which is in a different availability zone.

::: zone-end

### Zone recovery

When the affected availability zone recovers, Azure automatically takes the following actions:

- Restores instances in the recovered zone

- Removes any temporary instances that were created in other zones during the outage

- Returns to normal traffic distribution across all available zones

### Testing

The Azure platform manages traffic routing, failover, and failback for zone-redundant virtual network gateways. This feature is fully managed, so you don't need to initiate or validate availability zone failure processes.

## Multi-region support

A virtual network gateway is a single-region resource. If the region becomes unavailable, your gateway is also unavailable.

::: zone pivot="expressroute"

> [!NOTE]
> You can use the Premium ExpressRoute SKU when you have Azure resources that are spread across multiple regions. However, the Premium SKU doesn't affect how your gateway is configured, and it's still deployed into one region. For more information, see [What is Azure ExpressRoute?](../expressroute/expressroute-introduction.md).

::: zone-end

### Alternative multi-region approaches

::: zone pivot="expressroute"

You can create independent connectivity paths to your Azure environment by using one or more of the following approaches:

- Create multiple ExpressRoute circuits, which connect to gateways in different Azure regions.
- Use a site-to-site VPN as a backup for private peering traffic.
- Use Internet connectivity as a backup for Microsoft peering traffic.

For detailed guidance, see [Designing for disaster recovery with ExpressRoute private peering](../expressroute/designing-for-disaster-recovery-with-expressroute-privatepeering.md).

::: zone-end

::: zone pivot="vpn"

You can deploy separate VPN Gateways in two or more different regions. However, each gateway is attached to a different virtual network, and the gateways operate independently. There's no interaction, or replication of configuration or state between them. You're also responsible for configuring your clients and remote devices to connect to the correct VPN, or to switch between VPNs as required.

::: zone-end

## Reliability during service maintenance

Azure performs regular maintenance on virtual network gateways to ensure optimal performance and security. During these maintenance windows, some service disruptions can occur, but Azure designs these activities to minimize effect on your connectivity. 

During planned maintenance operations on virtual network gateways, the process is executed on gateway instances sequentially, never simultaneously. This process ensures that there's always one gateway instance active during maintenance, minimizing the impact on your active connections.

You can configure gateway maintenance windows to align with your operational requirements, reducing the likelihood of unexpected disruptions.

::: zone pivot="expressroute"

For more information, see [Configure customer-controlled maintenance for ExpressRoute gateways](../expressroute/customer-controlled-gateway-maintenance.md).

::: zone-end

::: zone pivot="vpn"

For more information, see [Configure maintenance windows for your VNet gateways](../vpn-gateway/customer-controlled-gateway-maintenance.md).

::: zone-end

## Service level agreement

[!INCLUDE [SLA description](includes/reliability-service-level-agreement-include.md)]

::: zone pivot="expressroute"

The availability SLA for your virtual network gateway increases when you use a gateway that is deployed with any SKU other than Basic.

Additionally, Azure ExpressRoute offers a strong availability SLA that guarantees high uptime for your connections. Different availability SLAs apply if you deploy across multiple peering locations (sites), if you use ExpressRoute Metro, or if you have a single-site configuration.

::: zone-end

For VPN Gateway, the Basic SKU provides a lower availability SLA and limited capabilities, and should only be used for testing and development. To increase your gateway's availability SLA, use any other SKU. For more information, see [Gateway SKUs - Production vs. Dev-Test workloads](../vpn-gateway/about-gateway-skus.md#workloads)

## Related content

::: zone pivot="expressroute"

- [Azure ExpressRoute overview](../expressroute/expressroute-introduction.md)
- [Design and architect Azure ExpressRoute for resiliency](../expressroute/design-architecture-for-resiliency.md)
- [Designing for high availability with ExpressRoute](../expressroute/designing-for-high-availability-with-expressroute.md)
- [Designing for disaster recovery with ExpressRoute private peering](../expressroute/designing-for-disaster-recovery-with-expressroute-privatepeering.md)

::: zone-end

::: zone pivot="vpn"

- [About zone-redundant VNet gateway in Azure availability zones](../vpn-gateway/about-zone-redundant-vnet-gateways.md)
- [Create a zone-redundant VNet gateway](../vpn-gateway/create-zone-redundant-vnet-gateway.md)
- [Monitor VNet gateway](../vpn-gateway/monitor-vpn-gateway.md)
- [Design highly available gateway connectivity for cross-premises and VNet-to-VNet connections](../vpn-gateway/vpn-gateway-highlyavailable.md)

::: zone-end
