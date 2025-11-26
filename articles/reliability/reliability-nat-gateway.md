---
title: Reliability in Azure NAT Gateway
description: Learn about reliability in Azure NAT Gateway, including resilience to transient faults and availability zone failures.
author: anaharris-ms
ms.author: anaharris
ms.topic: reliability-article
ms.custom: subject-reliability
ms.service: azure-nat-gateway
ms.date: 11/27/2025
ai-usage: ai-assisted
#Customer intent: As an engineer responsible for business continuity, I want to understand the details of how Azure NAT Gateway works from a reliability perspective and plan disaster recovery strategies in alignment with the exact processes that Azure services follow during different kinds of situations.
---

# Reliability in Azure NAT Gateway

[Azure NAT Gateway](/azure/nat-gateway/nat-overview) is a fully managed Network Address Translation (NAT) service that provides outbound internet connectivity for resources connected to your private virtual network. The service provides both source network address translation (SNAT) for outbound connections and destination network address translation (DNAT) for response packets to outbound-originated connections only. Because it sits on your critical network paths, Azure NAT Gateway is designed to be a highly resilient service.

[!INCLUDE [Shared responsibility](includes/reliability-shared-responsibility-include.md)]

This article describes how Azure NAT Gateway can be made resilient to a variety of potential outages and problems, including transient faults and availability zone outages. It also highlights some key information about the Azure NAT Gateway service level agreement (SLA).

## Production deployment recommendations

For production deployments, you should:

- Use the StandardV2 SKU, which automatically enables [zone redundancy](#resilience-to-availability-zone-failures) in supported regions.
- Configure zone-redundant public IP addresses for your NAT gateways.
- Configure your NAT gateway with enough public IP addresses to handle your peak connection requirements, which reduces the likelihood of availability problems due to SNAT port exhaustion.

## Reliability architecture overview

> [!WARNING]
> **Note to PG:** We've made some assumptions here about how things work - especially how a single logical NAT gateway can be mapped to multiple AZs. If we've misunderstood something or used incorrect terminology, we can adjust.

Azure NAT Gateway operates at the subnet level, automatically becoming the default route for outbound internet traffic without requiring additional routing configurations.

An NAT gateway is comprised of one or more *instances*, which represent the underlying compute infrastructure required to operate the service. Internally, Azure NAT Gateway implements a distributed architecture using software-defined networking to provide high reliability and scalability. The service operates with multiple fault domains, enabling it to survive multiple infrastructure component failures without service impact. Azure manages the underlying service operations, including distribution across fault domains and infrastructure redundancy.

For more information about Azure NAT Gateway architecture and redundancy, see [Azure NAT Gateway resource](/azure/nat-gateway/nat-gateway-resource#nat-gateway-architecture).

## Resilience to transient faults

[!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)]

SNAT port exhaustion can manifest as a transient fault in your application. To reduce the likelihood of transient faults related to network address translation, you should:

- **Minimize the likelihood of SNAT port exhaustion.** Exhaustion can happen when an application makes multiple independent connections to the same IP address and port. Configure your applications to handle SNAT port exhaustion gracefully by implementing connection pooling and proper connection lifecycle management.

- **Monitor the NAT gateway's datapath availability metric.** Use Azure Monitor to detect potential connectivity issues early. Set up alerts for connection failures and SNAT port exhaustion to proactively identify and address transient fault conditions before they impact your applications' outbound connectivity. To learn more, see [What is Azure NAT Gateway metrics and alerts?](/azure/nat-gateway/nat-metrics).

- **Avoid setting high idle timeout values.** Idle timeout values that are significantly higher than the default 4 minutes for NAT gateway connections can contribute to SNAT port exhaustion during high connection volumes.

For comprehensive guidance on connection management and troubleshooting Azure NAT Gateway-specific issues, see [Troubleshoot Azure NAT Gateway connectivity](/azure/nat-gateway/troubleshoot-nat-connectivity).

## Resilience to availability zone failures

[!INCLUDE [Resilience to availability zone failures](includes/reliability-availability-zone-description-include.md)]

Azure NAT Gateway supports availability zones in both zone-redundant and zonal configurations:

- *Zone-redundant:* When you use the StandardV2 SKU of Azure NAT Gateway, zone redundancy is enabled automatically. Zone redundancy for a NAT gateway spreads instances across multiple [availability zones](../reliability/availability-zones-overview.md). When you use a zone-redundant configuration, you can achieve resiliency and reliability for your production workloads.

    :::image type="content" source="../nat-gateway/nat-availability-zones/single-nat-gw-zone-spanning-subnet.png" alt-text="Diagram of zone-redundant deployment of NAT gateway.":::

- *Zonal:* When you use the Standard (v1) SKU, you can optionally create a zonal configuration by selecting a single availability zone for the NAT gateway. All traffic from connected subnets is routed through the NAT gateway, even if that's in a different availability zone.

     <!-- Allen: If we retain this image, then we need to move it into reliability hub. -->
    :::image type="content" source="../nat-gateway/nat-availability-zones/zonal-nat-gateway.png" alt-text="Diagram of zonal deployment of NAT gateway.":::

    If a NAT gateway within an availability zone experiences an outage, all virtual machines in the connected subnets fail to connect to the internet.

    You can create *zonal stacks* by deploying subnets for each availability zone, with a dedicated NAT gateway in each subnet. You need to manually assign virtual machines to the relevant availability zone and subnet. 
    
    <!-- Allen: If we retain this image, then we need to move it into reliability hub. -->
    :::image type="content" source="../nat-gateway/media/nat-availability-zones/multiple-zonal-nat-gateways.png" alt-text="Diagram of zonal isolation by creating zonal stacks.":::
    
    [!INCLUDE [Zonal resource description](includes/reliability-availability-zone-zonal-include.md)]

    > [!WARNING]
    > **Note to PG:** Is there any situation where it might still make sense to deploy a zonal NAT Gateway (once StandardV2 is GA)?

If you deploy a NAT gateway and don't specify an availability zone, the NAT gateway is then *nonzonal*, which means Azure selects the availability zone. A nonzonal configuration doesn't provide protection against availability zone outages.

### Requirements

- **Region suport:** Zone-redundant and zonal NAT gateways can be deployed into [any region that supports availability zones](./regions-list.md).

- **SKU:** To deploy a zone-redundant NAT gateway, you must use the StandardV2 SKU.

- **Public IP addresses:** The requirements for public IP addresses attached to a NAT gateway depend on whether it's deployed in a zone-redundant or zonal configuration:

    - *Zone-redundant:* Public IP addresses attached to a zone-redundant NAT gateway must also be zone-redundant.

    - *Zonal:* Public IP addresses attached to a zonal NAT gateway must be one of the following:
        - Zone-redundant public IP address.
        - Zonal public IP address in the same availability zone as the NAT gateway.
        - Nonzonal public IP address, although this isn't recommended.

For guidance on configuring zone-redundant public IP addresses with Azure NAT Gateway, see [Design NAT gateway for high resiliency](../nat-gateway/nat-availability-zones.md).

### Cost

There is no additional cost to use availability zone support for Azure NAT Gateway. For more information about pricing, see [Azure NAT Gateway pricing](https://azure.microsoft.com/pricing/details/azure-nat-gateway/).

### Configure availability zone support

- **New resources:** To deploy a new NAT gateway with availability zone support, see [Create a NAT gateway with availability zones](/azure/nat-gateway/quickstart-create-nat-gateway?tabs=portal).

<!-- Allen: Need to update the create document to reflect AZ options? -->

- **Enable availability zone support:** Azure NAT Gateway availability zone configuration cannot be changed after deployment. To modify the availability zone configuration, you must deploy a new NAT gateway with the desired zone settings.

### Behavior when all zones are healthy

This section describes what to expect when NAT gateways are configured for availability zone support and all availability zones are operational.

- **Traffic routing between zones**: The way traffic is routed through your NAT gateway depends on the availability zone configuration your NAT gateway uses.

    - *Zone-redundant:* Traffic is routed through an instance within any availability zone.

    - *Zonal:* Each NAT gateway instance operates independently within its assigned availability zone. Outbound traffic from subnet resources is routed through the NAT gateway instance in the resource's availability zone.

- **Data replication between zones**: Azure NAT Gateway doesn't perform data replication between zones as it's a stateless service for outbound connectivity. Each NAT gateway instance operates independently within its availability zone without requiring synchronization with instances in other zones.

### Behavior during a zone failure

This section describes what to expect when a NAT gateway is configured for availability zone support and there's an availability zone outage.

- **Detection and response:**

    - *Zone-redundant:* Azure NAT Gateway detects and responds to failures in an availability zone. You don't need to do anything to initiate an availability zone failover.

    - *Zonal*: You are responsible for implementing application-level failover to alternative connectivity methods or NAT gateways in other zones.

- **Notification:** <!-- TODO -->

    - *Zone-redundant:* Zone outages are visible through Azure Service Health and Azure Resource Health.

    - *Zonal:* Zone outages are visible through Azure Service Health notifications, Azure Resource Health, and the NAT gateway datapath availability metric. You can configure alerts on the datapath availability metric to detect connectivity issues.

- **Active requests:**

    - *Zone-redundant:* Active outbound connections are dropped, and clients need to retry. Subsequent connection attempts flow through an instance in another availability zone.

    - *Zonal:* Active outbound connections through a failed zonal NAT gateway are lost and must be re-established through alternative connectivity paths. Applications should implement retry logic to handle connection failures. Because the outbound public IP address changes, any TCP sessions might need to be renegotiated.

- **Expected data loss:** No data loss occurs because Azure NAT Gateway is a stateless service for outbound connectivity. Connection state is recreated when connections are re-established.

- **Expected downtime:**

    - *Zone-redundant:* No downtime is expected during a zone failover. Clients can retry connections immediately and requests will be routed to an instance in another zone.

    - *Zonal:* Outbound connectivity is lost until you reroute traffic to alternative NAT gateway instances in other zones, or through alternative connectivity methods.

- **Traffic rerouting:**
    
    - *Zone-redundant:* Future connection requests are routed through a NAT gateway instance in an availability zone that's online.
    
        It's unlikely that virtual machines in the affected availability zone would still be operating. However, in the event of a partial zone failure that causes Azure NAT Gateway to be unavailable while virtual machines continue to operate, any outbound connections from virtual machines in the affected zone are routed through a NAT gateway instance in another zone. These connections might have slightly higher latency because they cross multiple zones.

    - *Zonal:* You are responsible for implementing application-level failover to alternative connectivity methods or NAT gateways in other zones.

### Zone recovery

No manual intervention is required for failback operations because Azure NAT Gateway is a stateless service.

When an availability zone recovers, NAT gateway in that zone automatically become available for new outbound connections. Existing connections established through NAT gateway instances in other zones during the outage continue to use their current connectivity paths until the connections naturally terminate.

### Test for zone failures

The Azure NAT Gateway platform manages traffic routing, failover, and failback for zone-redundant NAT gateways. Because this feature is fully managed, you don't need to initiate anything or validate availability zone failure processes.

If you use a zonal NAT gateway, you need to prepare and test failover plans in case a zone failure occurs.

## Resilience to region-wide failures

Azure NAT Gateway is a single-region service that operates within the boundaries of a specific Azure region. The service doesn't provide native multi-region capabilities or automatic failover between regions. If a region becomes unavailable, NAT gateways in that region are also unavailable.

If you design a networking approach with multiple regions, you should deploy independent NAT gateways into each region.

## Service-level agreement

[!INCLUDE [SLA description](includes/reliability-service-level-agreement-include.md)]

Azure NAT Gateway's SLA only applies when you have configured two or more virtual machine instances, and it excludes SNAT port exhaustion from downtime calculations.

### Related content

- [Azure NAT Gateway overview](/azure/nat-gateway/nat-overview)
- [NAT gateway and availability zones](/azure/nat-gateway/nat-availability-zones)
- [Troubleshoot Azure NAT Gateway connectivity](/azure/nat-gateway/troubleshoot-nat-connectivity)
- [Azure NAT Gateway Resource Health](/azure/nat-gateway/resource-health)
