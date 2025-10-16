---
title: Reliability in Azure Virtual Network Gateway
description: Find out about reliability in Azure Virtual Network Gateway, including availability zones and multi-region deployments.
ms.author: anaharris
author: anaharris-ms
ms.topic: reliability-article
ms.custom: subject-reliability
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

TODO

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

TODO

::: zone-end

## Transient faults
<!-- TODO check this whole section -->

[!INCLUDE [Transient fault description](includes/reliability-transient-fault-description-include.md)]

Additionally, in a distributed networking environment, transient faults can occur at multiple layers. A transient fault in Azure, an edge site or device, or in your environment can temporarily disrupt connectivity.

::: zone pivot="expressroute"

ExpressRoute reduces the effect of transient faults by using redundant connection paths, fast fault detection, and automated failover. However, it's important that your on-premises components are configured correctly to be resilient to a variety of issues. For comprehensive fault handling strategies, see [Designing for high availability with ExpressRoute](../expressroute/designing-for-high-availability-with-expressroute.md).

::: zone-end

::: zone pivot="vpn"

TODO

::: zone-end

## Availability zone support

[!INCLUDE [AZ support description](includes/reliability-availability-zone-description-include.md)]

Virtual network gateways are automatically zone-redundant when they meet the requirements. Zone redundancy means that the gateway's instances distributed across three availability zones. This configuration eliminates any single zone as a point of failure and provides the highest level of zone resiliency. Zone-redundant gateways provide automatic failover within the region, and maintaining connectivity during zone maintenance or outages.

All newly created gateways are zone-redundant, and zone redundancy is recommended for all production workloads.

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

- Use a zone-enabled gateway SKU. The following SKUs support availability zones:

  [!INCLUDE [skus-with-az](../expressroute/includes/sku-availability-zones.md)]

- Ensure your gateway subnet is sized appropriately for the selected SKU and deployment requirements. <!-- TODO link -->

- Configure the gateway's public IP address to be zone-redundant.

::: zone-end

::: zone pivot="vpn"

TODO

::: zone-end

### Cost
<!-- TODO try to normalise -->

::: zone pivot="expressroute"

Zone-aware virtual network gateways follow standard gateway pricing based on the selected SKU. Zone-aware gateways require zone-enabled gateway SKUs, which have higher hourly rates compared to standard gateway SKUs due to their enhanced capabilities and performance characteristics.

There are no additional charges specifically to enable zone redundancy. Data transfer costs remain consistent regardless of zone configuration and are based on your ExpressRoute circuit bandwidth.

For detailed pricing information and to compare costs across different gateway SKUs and configurations, see the [Azure ExpressRoute pricing page](https://azure.microsoft.com/pricing/details/expressroute/).

::: zone-end

::: zone pivot="vpn"

TODO

::: zone-end

### Configure availability zone support

This section explains how to configure zone redundancy for your virtual network gateways.

- **Create a new virtual network gateway with availability zone support.** Any new virtual network gateways you create are automatically zone-redundant, if they meet the requirements listed above.

::: zone pivot="expressroute"

    For detailed deployment instructions, see TODO.

::: zone-end

::: zone pivot="vpn"

    For detailed deployment instructions, see TODO.

::: zone-end

- **Change the availability zone configuration of an existing virtual network gateway.** Virtual network gateways that you already created might not be zone-redundant. You can migrate a nonzonal gateway to a zone-redundant gateway with minimal downtime.

::: zone pivot="expressroute"

    For more information, see [Migrate ExpressRoute gateways to availability zone-enabled SKUs](../expressroute/gateway-migration.md).

::: zone-end

::: zone pivot="vpn"

    For more information, see TODO.

::: zone-end

- **Verify the zone redundancy status of an existing virtual network gateway.** TODO

### Capacity planning and management

TODO

### Normal operations

The following section describes what to expect when your virtual network gateway is configured for zone redundancy and all availability zones are operational.

- **Data replication between zones:** No data replication occurs between zones because the virtual network gateway doesn't store persistent customer data.

- **Traffic routing between zones:** Traffic from your on-premises environment is distributed among instances in all of the zones that your gateway uses. This active-active configuration ensures optimal performance and load distribution under normal operating conditions.

    ::: zone pivot="expressroute"

    However, if you use FastPath for optimized performance, traffic from your on-premises environment bypasses the gateway, which improves throughput and reduces latency. For more information, see TODO.

    ::: zone-end

- **Instance management:** The platform automatically manages instance placement across the zones that your gateway uses. It replaces failed instances and maintains the configured instance count. Health monitoring ensures that only healthy instances receive traffic.

### Zone-down experience

The following section describes what to expect when your virtual network gateway is configured for zone redundancy and there's an availability zone outage.

- **Detection and response:** Microsoft manages the detection of zone failures and automatically initiates failover. No customer action is required.

::: zone pivot="expressroute"

- **Notification**: ExpressRoute doesn't notify you when a zone is down. However, you can use [Azure Resource Health](/azure/service-health/resource-health-overview) to monitor for the health of your gateway. You can also use [Azure Service Health](/azure/service-health/overview) to understand the overall health of the Azure ExpressRoute service, including any zone failures.

::: zone-end

::: zone pivot="vpn"

- **Notification**: Azure VPN Gateway doesn't notify you when a zone is down. However, you can use [Azure Resource Health](/azure/service-health/resource-health-overview) to monitor for the health of your gateway. You can also use [Azure Service Health](/azure/service-health/overview) to understand the overall health of the Azure VPN Gateway service, including any zone failures.

::: zone-end

    Set up alerts on these services to receive notifications of zone-level problems. For more information, see [Create Service Health alerts in the Azure portal](/azure/service-health/alerts-activity-log-service-notifications-portal) and [Create and configure Resource Health alerts](/azure/service-health/resource-health-alert-arm-template-guide).

- **Active requests:** Any active requests connected through gateway instances in the failing zone are terminated. Clients should retry the requests by following the guidance for how to [handle transient faults](#transient-faults).

- **Expected data loss:** Zone failures aren't expected to cause data loss because virtual network gateways don't store persistent customer data.

- **Expected downtime:** During zone outages, connections might experience brief interruptions that typically last up to one minute as traffic is redistributed.

::: zone pivot="expressroute"

- **Traffic rerouting:** The platform automatically distributes traffic to healthy zones for zone-redundant deployments, with BGP route convergence typically completing within 1-3 minutes.

    FastPath-enabled circuits maintain optimized routing throughout the failover process, ensuring minimal effect on application performance. <!-- PG: If you use FastPath, does a gateway outage affect you at all? If not, we should clarify throughout the section. -->

::: zone-end

::: zone pivot="vpn"

TODO

::: zone-end

### Zone recovery

When the affected availability zone recovers, Azure automatically takes the following actions:

- Restores instances in the recovered zone

- Removes any temporary instances that were created in other zones during the outage

- Returns to normal traffic distribution across all available zones

### Testing

::: zone pivot="expressroute"

<!-- TODO PG to review this section -->

Test route convergence by withdrawing BGP advertisements from specific paths to simulate circuit or peering location failures. Since zone-redundant gateways automatically handle zone failures, focus testing on end-to-end application connectivity and monitoring the automatic failover process rather than manual intervention procedures.

These testing approaches help verify that your ExpressRoute configuration properly handles zone outage scenarios and meets your availability requirements. Document all test results and maintain regular testing schedules to ensure continued reliability as your environment evolves.

::: zone-end

::: zone pivot="vpn"

TODO

::: zone-end

## Multi-region support

A virtual network gateway is a single-region resource. If the region becomes unavailable, your gateway is also unavailable.

::: zone pivot="expressroute"

If you have Azure resources that are spread across multiple regions, you can use the ExpressRoute Premium SKU to connect across regions. Each ExpressRoute SKU enables connectivity with a different set of regions. For more information, see [What is Azure ExpressRoute?](../expressroute/expressroute-introduction.md). <!-- TODO mention this is helpful but not relared to resiliency of the gateway -->

::: zone-end

::: zone pivot="vpn"

TODO

::: zone-end

### Alternative multi-region approaches

Because a virtual network gateway is deployed into a single region, if that region has a major outage then your networks might lose connectivity.

You can create independent connectivity paths to your Azure environment by using one or more of the following approaches:

::: zone pivot="expressroute"

- Create multiple ExpressRoute circuits, which connect to gateways in different Azure regions.
- Use a site-to-site VPN as a backup for private peering traffic.
- Use Internet connectivity as a backup for Microsoft peering traffic.

For detailed guidance, see [Designing for disaster recovery with ExpressRoute private peering](../expressroute/designing-for-disaster-recovery-with-expressroute-privatepeering.md).

::: zone-end

::: zone pivot="vpn"

TODO

::: zone-end

## Reliability during service maintenance

Azure performs regular maintenance on virtual network gateways to ensure optimal performance and security. During these maintenance windows, some service disruptions can occur, but Azure designs these activities to minimize effect on your connectivity. 

::: zone pivot="expressroute"

You can configure gateway maintenance windows to align with your operational requirements, reducing the likelihood of unexpected disruptions. For more information, see [Configure customer-controlled maintenance for ExpressRoute gateways](../expressroute/customer-controlled-gateway-maintenance.md).

::: zone-end

::: zone pivot="vpn"

TODO

::: zone-end

## Service level agreement

[!INCLUDE [SLA description](includes/reliability-service-level-agreement-include.md)]

::: zone pivot="expressroute"

Azure ExpressRoute offers a strong service level agreement (SLA) that guarantees high uptime for your connections. Different availability SLAs apply if you deploy across multiple peering locations (sites), if you use ExpressRoute Metro, or if you have a single-site configuration.

::: zone-end

::: zone pivot="vpn"

TODO

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
