---
title: Reliability in Azure Virtual Network Gateway
description: Find out about reliability in Azure Virtual Network Gateway, including availability zones and multi-region deployments.
ms.author: anaharris
author: anaharris-ms
ms.topic: reliability-article
ms.custom: subject-reliability
ms.service:
- azure-vpn-gateway
- azure-expressroute
ms.date: 10/16/2025
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
> This article covers the reliability of VPN gateways, which are the Azure-based parts of VPNs.
>
> When you use VPNs, it's critical that you design your whole network architecture to meet your resiliency requirements. <!-- TODO more -->

::: zone-end

[!INCLUDE [Shared responsibility description](includes/reliability-shared-responsibility-include.md)]

## Production deployment recommendations

::: zone pivot="expressroute"

To learn about how to deploy Azure ExpressRoute to support your solution's reliability requirements and how reliability affects other aspects of your architecture, see [Architecture best practices for Azure ExpressRoute in the Azure Well-Architected Framework](/azure/well-architected/service-guides/azure-expressroute).

::: zone-end

::: zone pivot="vpn"

For production deployments, you should consider the following recommendations:

> [!div class="checklist"]
> - **Enable zone redundancy** if your Azure VPN Gateway resources are in a supported region. Deploy VPN Gateway using supported SKUs (VpnGw1AZ or higher) to ensure access to zone redundancy features.
> - **Use Standard SKU public IP addresses.**
> - **Configure active-active mode** for higher availability when supported by your remote VPN devices.
> - **Implement proper monitoring** using [Azure Monitor VPN Gateway metrics](../vpn-gateway/monitor-vpn-gateway.md).

::: zone-end

## Reliability architecture overview

<!-- TODO rewrite this whole section -->

::: zone pivot="expressroute"

When you use ExpressRoute, you deploy and configure several components:

- *Circuits and connections*: An ExpressRoute *circuit* consists of two *connections* through a single peering location to the Microsoft Enterprise Edge. By using two connections, you can achieve active-active connectivity. However, this configuration doesn't protect against site-level failures.

- *Customer premises equipment* (CPE) includes your edge routers and client devices. You need to ensure that your CPE is designed to be resilient to problems, and that it can quickly recover when problems happen in other parts of your ExpressRoute infrastructure. For more information, see [Design and architect Azure ExpressRoute for resiliency](../expressroute/design-architecture-for-resiliency.md).

- *Sites:* Circuits are established through a *site*, which is a physical peering location. Sites are designed to be highly available and have built-in redundancy across all layers, but because they represent a single physical location, there is a possibility of sites having problems.

  To mitigate the risk of site outages, ExpressRoute offers three site resiliency options that vary in their level of protection:
    
    - Standard resiliency provides a single circuit at a single site and represents the least resilient configuration.
    - High resiliency, also known as ExpressRoute Metro, enables a single circuit across two sites within the same metropolitan area.
    - Maximum resiliency offers two circuits across different peering locations and is recommended for business-critical workloads.

  For more information, see [Design and architect Azure ExpressRoute for resiliency - Site resiliency for ExpressRoute](../expressroute/design-architecture-for-resiliency.md#site-resiliency-for-expressroute).

- *Gateways:* In Azure, you create an *ExpressRoute virtual network gateway*. You deploy the gateway within your virtual network, so it's also sometimes called a *virtual network gateway*. Each gateway acts as the termination point for one or more ExpressRoute circuits.

    An ExpressRoute gateway contains two or more *instances*, which represent virtual machines (VMs) that your gateway uses to process ExpressRoute traffic. You can protect against availability zone failures by distributing gateway instances across multiple zones, providing automatic failover within the region, and maintaining connectivity during zone maintenance or outages.

This diagram shows two different ExpressRoute configurations, each with a single virtual network gateway, configured for different levels of resiliency across sites:

:::image type="content" source="../expressroute/media/design-architecture-for-resiliency/standard-vs-maximum-resiliency.png" alt-text="Diagram illustrating ExpressRoute connection options between on-premises network and Azure, showing different resiliency levels.":::

::: zone-end

::: zone pivot="vpn"

An *instance* refers to a virtual machine (VM)-level unit of the VPN gateway. Each instance represents the infrastructure that handles VPN traffic and performs security checks.

To achieve high availability, Azure VPN Gateway automatically provides two instances without requiring your intervention or configuration. The gateway automatically scales out when average throughput, CPU consumption, and connection usage reach predefined thresholds. The platform automatically manages instance creation, health monitoring, and the replacement of unhealthy instances.

To achieve protection against server and server rack failures, Azure VPN Gateway automatically distributes instances across multiple fault domains within a region.

A VPN requires components to be deployed in both the on-premises environment and within Azure. The following diagram illustrates some key components in a VPN that connects from an on-premises environment to Azure:

:::image type="content" source="media/reliability-virtual-network-gateway/vpn-reliability-architecture.png" alt-text="Diagram that shows Azure VPN Gateway, on-premises site-to-site, and point-to-site networks." border="false":::

- **On-premises components**: The components you deploy depend on whether you deploy a point-to-site or site-to-site configuration. To learn more about the differences, see [VPN Gateway topology and design](/azure/vpn-gateway/design).

   - Site-to-site configurations require an on-premises VPN device, which you're responsible for deploying, configuring, and managing.
   - Point-to-site configurations require you to deploy a VPN client application in a remote desktop and associate the configuration to each client that connects to the VPN. You're responsible for deploying and configuring the client devices.

- **Azure components**: In Azure, you create a *VPN gateway*, which is sometimes also called a *VNet gateway*. The public IP address requirements depend on your configuration:

   - **Active-standby mode**: Requires one public IP address.
   - **Active-active mode (site-to-site only)**: Requires two public IP addresses.  
   - **Active-active mode with both site-to-site and point-to-site**: Requires three static public IP addresses (deployed with Standard SKU).
   
All public IP addresses must be deployed as separate resources that meet specific requirements for zone redundancy.

A VPN gateway contains two *instances*, which represent virtual machines (VMs) that your gateway uses to process VPN traffic. To learn more about the redundancy that's built into VPN Gateway, see [Design highly available gateway connectivity for cross-premises and VNet-to-VNet connections](/azure/vpn-gateway/vpn-gateway-highlyavailable).

A site-to-site VPN configuration requires a *local network gateway*, which represents the remote VPN device. The local network gateway is a mandatory component to define a connection. It has two mutually exclusive configurations: static routing (doesn't require dynamic protocol in VPN Gateway and remote devices) and dynamic routing (requires BGP in Azure and remote devices).

::: zone-end

::: zone pivot="vpn"

## Reliability of on-premises components

You're responsible for managing the reliability of your side of the VPN connection, including client devices for point-to-site configurations and remote VPN devices for site-to-site configurations. You can improve the resiliency of your VPN by deploying multiple on-premises VPN devices in an active-active or active-passive configuration. 

For site-to-site VPN, we recommend a configuration in active-active mode. However, there are some cases where active-active isn't possible due to the asymmetric nature of flows in site-to-site connections. For more information, see:

- [How does Azure VPN Gateway handle traffic flow in active-active mode and what should I consider if my on-premises setup requires symmetric routing?](/azure/vpn-gateway/vpn-gateway-vpn-faq#how-does-azure-vpn-gateway-handle-traffic-flow-in-active-active-mode-and-what-should-i-consider-if-my-on-premises-setup-requires-symmetric-routing)
- [Does Azure guarantee symmetric routing for a given flow in active-active VPN mode?](/azure/vpn-gateway/vpn-gateway-vpn-faq#does-azure-guarantee-symmetric-routing-for-a-given-flow-in-active-active-vpn-mode)

For guidance about configuring your infrastructure for high availability, see [Design highly available gateway connectivity for cross-premises and VNet-to-VNet connections](/azure/vpn-gateway/vpn-gateway-highlyavailable).

::: zone-end

## Transient faults
<!-- TODO check this whole section -->

[!INCLUDE [Transient fault description](includes/reliability-transient-fault-description-include.md)]

Additionally, in a distributed networking environment, transient faults can occur at multiple layers. A transient fault in Azure, an edge site or device, or in your environment can temporarily disrupt connectivity.

::: zone pivot="expressroute"

ExpressRoute reduces the effect of transient faults by using redundant connection paths, fast fault detection, and automated failover. However, it's important that your on-premises components are configured correctly to be resilient to a variety of issues. For comprehensive fault handling strategies, see [Designing for high availability with ExpressRoute](../expressroute/designing-for-high-availability-with-expressroute.md).

::: zone-end

::: zone pivot="vpn"

For applications that connect through Azure VPN Gateway, implement retry logic with exponential backoff to handle potential transient connection problems. Azure VPN Gateway's stateful nature ensures that legitimate connections are maintained during brief network interruptions.

VPN Gateway connections may experience transient faults due to network interruptions or service updates. Transient faults can occur in the Azure backbone before traffic reaches the internet, in the internet itself, or on-premises. Data traffic, such as TCP flows, transits through IPsec tunnels. Transient faults can impact IPsec tunnels and/or TCP data flows. In the event of a disconnection, IKE (Internet Key Exchange) renegotiates the Security Associations (SAs) for both Phase 1 and Phase 2 to re-establish the IPsec tunnel.

During scaling operations, which take 5 to 7 minutes to complete, existing connections are preserved while new gateway instances are added to handle increased load.

::: zone-end

## Availability zone support

[!INCLUDE [AZ support description](includes/reliability-availability-zone-description-include.md)]

Virtual network gateways are automatically zone-redundant when they meet the requirements. Zone redundancy means that the gateway's instances distributed across three availability zones. This configuration eliminates any single zone as a point of failure and provides the highest level of zone resiliency. Zone-redundant gateways provide automatic failover within the region, and maintaining connectivity during zone maintenance or outages.

All newly created gateways are zone-redundant, and zone redundancy is recommended for all production workloads.

The following diagram shows a zone-redundant virtual network gateway with two instances that are distributed across availability zones: <!-- TODO confirm how instance counts work for both VPN and ExR -->

:::image type="content" source="media/reliability-virtual-network-gateway/zone-redundant.png" alt-text="Diagram that shows a virtual network gateway with two instances distributed across availability zones." border="false":::

::: zone pivot="expressroute"

> [!NOTE]
> There's no availability zone configuration for circuits or connections. These resources are located in network edge facilities, which aren't designed to use availability zones.

::: zone-end

### Region support

Zone-redundant virtual network gateways are available in [all regions that support availability zones](../reliability/availability-zones-region-support.md).

### Requirements
<!-- TODO try to normalise -->

To deploy zone-redundant virtual network gateways, you must meet all of the following requirements:

::: zone pivot="expressroute"

- **SKU:** Use a zone-enabled gateway SKU. The following SKUs support availability zones:

  [!INCLUDE [skus-with-az](../expressroute/includes/sku-availability-zones.md)]

- **Subnet size:** Ensure your gateway subnet is sized appropriately for the selected SKU and deployment requirements. <!-- TODO link --> <!-- TODO is this true for VPN too? -->

::: zone-end

::: zone pivot="vpn"

- **SKU:** All tiers of Azure VPN Gateway support availability zones, except the Basic SKU, which is only for development environments.

- **Public IP address:** Use standard public IP addresses and configure them to be zone-redundant.

::: zone-end

### Cost

::: zone pivot="expressroute"

Zone-redundant gateways for ExpressRoute require specific SKUs, which have higher hourly rates compared to standard gateway SKUs due to their enhanced capabilities and performance characteristics. For pricing information, see [Azure ExpressRoute pricing](https://azure.microsoft.com/pricing/details/expressroute/).

::: zone-end

::: zone pivot="vpn"

There's no extra cost for a gateway deployed across multiple availability zones. You're billed for the gateway instances and public IP addresses. For pricing information, see [VPN Gateway pricing](https://azure.microsoft.com/pricing/details/vpn-gateway/).

::: zone-end

### Configure availability zone support

This section explains how to configure zone redundancy for your virtual network gateways.

- **Create a new virtual network gateway with availability zone support.** Any new virtual network gateways you create are automatically zone-redundant, if they meet the requirements listed above.

::: zone pivot="expressroute"

    For detailed deployment instructions, see TODO.

::: zone-end

::: zone pivot="vpn"

    For detailed configuration steps, see [Create a zone-redundant VNet gateway](../vpn-gateway/create-zone-redundant-vnet-gateway.md).

::: zone-end

- **Change the availability zone configuration of an existing virtual network gateway.** Virtual network gateways that you already created might not be zone-redundant. You can migrate a nonzonal gateway to a zone-redundant gateway with minimal downtime.

::: zone pivot="expressroute"

    For more information, see [Migrate ExpressRoute gateways to availability zone-enabled SKUs](../expressroute/gateway-migration.md).

::: zone-end

::: zone pivot="vpn"

    For more information, see [About SKU consolidation & migration](../vpn-gateway/gateway-sku-consolidation.md).

::: zone-end

- **Verify the zone redundancy status of an existing virtual network gateway.** TODO

### Capacity planning and management

TODO

### Normal operations

The following section describes what to expect when your virtual network gateway is configured for zone redundancy and all availability zones are operational.

::: zone pivot="expressroute"

- **Traffic routing between zones:** Traffic from your on-premises environment is distributed among instances in all of the zones that your gateway uses. This active-active configuration ensures optimal performance and load distribution under normal operating conditions.

    However, if you use FastPath for optimized performance, traffic from your on-premises environment bypasses the gateway, which improves throughput and reduces latency. For more information, see TODO.

::: zone-end

::: zone pivot="vpn"

- **Traffic routing between zones:** VPN connections are routed to one primary VPN gateway instance, which is selected by Azure. The traffic routing behavior depends on whether you have configured active-standby or active-active mode, not on the zonal, or zone-redundant deployment type.

::: zone-end

- **Data replication between zones:** No data replication occurs between zones because the virtual network gateway doesn't store persistent customer data.

<!-- TODO verify instance counts and try to normalise the instance management bullet -->

::: zone pivot="expressroute"

- **Instance management:** The platform automatically manages instance placement across the zones that your gateway uses. It replaces failed instances and maintains the configured instance count. Health monitoring ensures that only healthy instances receive traffic.

::: zone-end

::: zone pivot="vpn"

- **Instance management**: The platform automatically manages instance placement across the zones your VPN gateway uses, replacing failed instances and maintaining the configured instance count of two gateway instances. Health monitoring ensures that only healthy instances receive traffic.

::: zone-end

### Zone-down experience

The following section describes what to expect when your virtual network gateway is configured for zone redundancy and there's an availability zone outage.

- **Detection and response:** The Azure platform detects and responds to a failure in an availability zone. You don't need to initiate a zone failover.

::: zone pivot="expressroute"

- **Notification**: ExpressRoute doesn't notify you when a zone is down. However, you can use [Azure Resource Health](/azure/service-health/resource-health-overview) to monitor for the health of your gateway. You can also use [Azure Service Health](/azure/service-health/overview) to understand the overall health of the Azure ExpressRoute service, including any zone failures.

::: zone-end

::: zone pivot="vpn"

- **Notification**: Azure VPN Gateway doesn't notify you when a zone is down. However, you can use [Azure Resource Health](/azure/service-health/resource-health-overview) to monitor for the health of your gateway. You can also use [Azure Service Health](/azure/service-health/overview) to understand the overall health of the Azure VPN Gateway service, including any zone failures.

::: zone-end

    Set up alerts on these services to receive notifications of zone-level problems. For more information, see [Create Service Health alerts in the Azure portal](/azure/service-health/alerts-activity-log-service-notifications-portal) and [Create and configure Resource Health alerts](/azure/service-health/resource-health-alert-arm-template-guide).

- **Active requests:** Any active requests connected through gateway instances in the failing zone are terminated. Clients should retry the requests by following the guidance for how to [handle transient faults](#transient-faults).

- **Expected data loss:** Zone failures aren't expected to cause data loss because virtual network gateways don't store persistent customer data.

- **Expected downtime:** During zone outages, connections might experience brief interruptions that typically last up to one minute as traffic is redistributed. Clients should retry the requests by following the guidance for how to [handle transient faults](#transient-faults).

::: zone pivot="expressroute"

- **Traffic rerouting:** The platform automatically distributes traffic to healthy zones for zone-redundant deployments, with BGP route convergence typically completing within 1-3 minutes.

    FastPath-enabled circuits maintain optimized routing throughout the failover process, ensuring minimal effect on application performance. <!-- PG: If you use FastPath, does a gateway outage affect you at all? If not, we should clarify throughout the section. -->

::: zone-end

::: zone pivot="vpn"

- **Traffic rerouting:** Traffic automatically reroutes to healthy availability zones. If needed, the platform creates new gateway instances in healthy zones. <!-- TODO verify whether capacity is actually replaced - it isn't for ExR -->

::: zone-end

### Zone recovery

When the affected availability zone recovers, Azure automatically takes the following actions:

- Restores instances in the recovered zone

- Removes any temporary instances that were created in other zones during the outage

- Returns to normal traffic distribution across all available zones

### Testing

The Azure platform manages traffic routing, failover, and failback for zone-redundant virtual network gateways. This feature is fully managed, so you don't need to initiate or validate availability zone failure processes.

::: zone pivot="expressroute"

<!-- TODO PG to review this section -->

However, you can test your application's resiliency to ExpressRoute problems, and verify that your ExpressRoute configuration properly handles zone outage scenarios and meets your availability requirements.

Test route convergence by withdrawing BGP advertisements from specific paths to simulate circuit or peering location failures. Since zone-redundant gateways automatically handle zone failures, focus testing on end-to-end application connectivity and monitoring the automatic failover process rather than manual intervention procedures.

::: zone-end

## Multi-region support

A virtual network gateway is a single-region resource. If the region becomes unavailable, your gateway is also unavailable.

::: zone pivot="expressroute"

If you have Azure resources that are spread across multiple regions, you can use the ExpressRoute Premium SKU to connect across regions. Each ExpressRoute SKU enables connectivity with a different set of regions. For more information, see [What is Azure ExpressRoute?](../expressroute/expressroute-introduction.md). <!-- TODO mention this is helpful but not relared to resiliency of the gateway -->

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

You can deploy separate VPN Gateways in two or more different regions. However, each gateway is attached to a different virtual network, and the gateways operate independently with no interaction or configuration/state replication between them. You're also responsible for configuring clients to connect to the correct VPN, or to switch between VPNs as required.

::: zone-end

## Reliability during service maintenance

Azure performs regular maintenance on virtual network gateways to ensure optimal performance and security. During these maintenance windows, some service disruptions can occur, but Azure designs these activities to minimize effect on your connectivity. 

During planned maintenance operations on virtual netwowork gateways, the process is executed on gateway instances sequentially, never simultaneously. This process ensures that there's always one gateway instance active during maintenance, minimizing the impact on your active connections. <!-- TODO verify whether this is true for all VNet gateways or just VPN -->

You can configure gateway maintenance windows to align with your operational requirements, reducing the likelihood of unexpected disruptions.

::: zone pivot="expressroute"

For more information, see [Configure customer-controlled maintenance for ExpressRoute gateways](../expressroute/customer-controlled-gateway-maintenance.md).

::: zone-end

::: zone pivot="vpn"

For more information, see [Configure maintenance windows for your VNet gateways](/azure/vpn-gateway/customer-controlled-gateway-maintenance).

::: zone-end

## Service level agreement

[!INCLUDE [SLA description](includes/reliability-service-level-agreement-include.md)]

::: zone pivot="expressroute"

Azure ExpressRoute offers a strong availability SLA that guarantees high uptime for your connections. Different availability SLAs apply if you deploy across multiple peering locations (sites), if you use ExpressRoute Metro, or if you have a single-site configuration.

::: zone-end

::: zone pivot="vpn"

<!-- TODO rewrite -->
VPN Gateway is provided with an SLA. The Basic SKU is the only Gateway SKU without SLA, with limited capabilities and support, and it can be used only for test and development environments. For more information, see [About Gateway SKUs](/azure/vpn-gateway/about-gateway-skus#workloads).

::: zone-end

## Related content

::: zone pivot="expressroute"

- [Azure ExpressRoute overview](../expressroute/expressroute-introduction.md)
- [Design and architect Azure ExpressRoute for resiliency](../expressroute/design-architecture-for-resiliency.md)
- [Designing for high availability with ExpressRoute](../expressroute/designing-for-high-availability-with-expressroute.md)
- [Designing for disaster recovery with ExpressRoute private peering](../expressroute/designing-for-disaster-recovery-with-expressroute-privatepeering.md)

::: zone-end

::: zone pivot="vpn"

TODO

::: zone-end
