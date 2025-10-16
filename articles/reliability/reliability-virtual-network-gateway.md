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

TODO

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
> When you use VPNs, it's critical that you design your whole network architecture to meet your resiliency requirements. TODO more

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

TODO

## Transient faults

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

::: zone pivot="expressroute"

ExpressRoute virtual network gateways support two availability zone deployment models:

- *Zone-redundant:* Gateway instances are automatically distributed across multiple availability zones within the region. This configuration eliminates any single zone as a point of failure and provides the highest level of zone resiliency. Zone-redundant gateways provide automatic failover within the region, and maintaining connectivity during zone maintenance or outages.

    <!-- PG: Are zone-redundant gateways always spread across 3 zones, or could it be 2-3? -->

    Zone-redundant gateways provide equivalent performance characteristics to zonal and nonzonal gateways, but they are more resilient.

    Zone-redundant deployment is the default when using the Azure portal and is recommended for production workloads.

    <!-- TODO diagram -->

- *Zonal:* Azure deploys all of the gateway instances into a single zone that you select within your chosen Azure region.

    This model is useful for latency-sensitive workloads that require colocation with other resources in the same zone. However, it doesn't provide zone resilience.

    > [!IMPORTANT]
    > Pinning to a single availability zone is only recommended when [cross-zone latency](./availability-zones-overview.md#inter-zone-latency) is too high for your needs and when you verify that the latency doesn't meet your requirements. By itself, a zonal gateway doesn't provide resiliency to an availability zone outage. To improve the resiliency of a zonal gateway deployment, you need to explicitly deploy separate gateways into multiple availability zones, and configure separate circuits for each gateway. <!-- PG: please verify this advice is accurate. -->
    
    <!-- TODO diagram -->

::: zone-end

If you don't specify availability zones to use for your gateway, your gateway is *nonzonal* or *regional*, which means it might be placed in any availability zone within the region. If any availability zone in the region has a problem, your gateway might experience downtime.

::: zone pivot="expressroute"

> [!NOTE]
> There's no availability zone configuration for circuits or connections. These resources are located in network edge facilities, which aren't designed to use availability zones.

::: zone-end

### Region support

Azure supports zone-aware virtual network gateways in [all regions that support availability zones](../reliability/availability-zones-region-support.md).

### Requirements

::: zone pivot="expressroute"

To deploy zone-aware ExpressRoute gateways, you must:

- Use a zone-enabled gateway SKU. The following SKUs support availability zones:

  [!INCLUDE [skus-with-az](../expressroute/includes/sku-availability-zones.md)]

- Ensure your gateway subnet is sized appropriately for the selected SKU and deployment requirements. <!-- TODO link -->

- Configure the gateway's public IP address correctly for the type of availability zone support you need:
    - *Zone-redundant:* The public IP address must be configured to be zone-redundant.  
    - *Zonal:* The public IP address must be configured to be zonal and located in the same zone as the gateway. <!-- TODO check can a ZR PIP be used with a zonal ExR GW -->

::: zone-end

::: zone pivot="vpn"

::: zone-end

### Cost
<!-- TODO try to normalise -->

::: zone pivot="expressroute"

Zone-aware ExpressRoute gateways follow standard gateway pricing based on the selected SKU. Zone-aware gateways require zone-enabled gateway SKUs, which have higher hourly rates compared to standard gateway SKUs due to their enhanced capabilities and performance characteristics.

There are no additional charges specifically to enable zone redundancy or other availability zone cnfiguration. Data transfer costs remain consistent regardless of zone configuration and are based on your ExpressRoute circuit bandwidth.

For detailed pricing information and to compare costs across different gateway SKUs and configurations, see the [Azure ExpressRoute pricing page](https://azure.microsoft.com/pricing/details/expressroute/).

::: zone-end

::: zone pivot="vpn"

::: zone-end

### Configure availability zone support

This section explains how to configure availability zone support for your virtual network gateways.

::: zone pivot="expressroute"

- **Create a new ExpressRoute gateway with availability zone support.** The approach that you use to configure availability zones depends on whether you want to create a zone-redundant or zonal gateway.

    - *Zone-redundant:* To deploy a new zone-redundant ExpressRoute virtual network gateway, refer to the guide for [creating a zone-redundant virtual network gateway in Azure Availability Zones](../vpn-gateway/create-zone-redundant-vnet-gateway.md).

    - *Zonal:* You must create a zonal ExpressRoute gateway by using Powershell, Bicep, or ARM templates. <!-- TODO verify CLI, Terraform --> You must explicitly create a standard public IP address, and configure to the zonal and deployed to the same zone that your gateway will be deployed to.

        > [!NOTE]
        > [!INCLUDE [Availability zone numbering](./includes/reliability-availability-zone-numbering-include.md)]

- **Change the availability zone configuration of an existing ExpressRoute gateway.** You can migrate a nonzonal gateway to a zone-redundant gateway with minimal downtime. For more information, see [Migrate ExpressRoute gateways to availability zone-enabled SKUs](../expressroute/gateway-migration.md). <!-- TODO what about zonal to ZR? -->

<!-- TODO do we need capacity planning section? -->

::: zone-end

### Normal operations

The following section describes what to expect when your virtual network gateway is configured with availability zone support and all availability zones are operational.

- **Data replication between zones:** No data replication occurs between zones because the virtual network gateway doesn't store persistent customer data.

- **Traffic routing between zones:** Traffic from your on-premises environment is distributed among instances in all of the zones that your gateway uses. This active-active configuration ensures optimal performance and load distribution under normal operating conditions.

    ::: zone pivot="expressroute"

    However, if you use FastPath for optimized performance, traffic from your on-premises environment bypasses the gateway, which improves throughput and reduces latency. For more information, see TODO.

    ::: zone-end

- **Instance management:** The platform automatically manages instance placement across the zones that your gateway uses. It replaces failed instances and maintains the configured instance count. Health monitoring ensures that only healthy instances receive traffic.

### Zone-down experience

TODO

### Zone recovery

TODO

### Testing

TODO

## Multi-region support

TODO

### Alternative multi-region approaches

TODO

## Reliability during service maintenance

TODO

## Service level agreement

[!INCLUDE [SLA description](includes/reliability-service-level-agreement-include.md)]

Azure ExpressRoute offers a strong service level agreement (SLA) that guarantees high uptime for your connections. Different availability SLAs apply if you deploy across multiple peering locations (sites), if you use ExpressRoute Metro, or if you have a single-site configuration.

## Related content

- [Azure ExpressRoute overview](../expressroute/expressroute-introduction.md)
- [Design and architect Azure ExpressRoute for resiliency](../expressroute/design-architecture-for-resiliency.md)
- [Designing for high availability with ExpressRoute](../expressroute/designing-for-high-availability-with-expressroute.md)
- [Designing for disaster recovery with ExpressRoute private peering](../expressroute/designing-for-disaster-recovery-with-expressroute-privatepeering.md)
