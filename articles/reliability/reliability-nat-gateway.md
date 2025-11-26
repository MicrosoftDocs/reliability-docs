---
title: Reliability in Azure NAT Gateway
description: Learn about reliability in Azure NAT Gateway, including availability zones and multi-region deployments.
ms.author: anaharris
author: anaharris-ms
ms.topic: reliability-article
ms.custom: subject-reliability
ms.service: azure-nat-gateway
ms.date: 10/06/2025
#Customer intent: As an engineer responsible for business continuity, I want to understand the details of how Azure NAT Gateway works from a reliability perspective and plan disaster recovery strategies in alignment with the exact processes that Azure services follow during different kinds of situations.
---

# Reliability in Azure NAT Gateway

[Azure NAT Gateway](/azure/nat-gateway/nat-overview) is a fully managed Network Address Translation (NAT) service that provides outbound internet connectivity for resources connected to your private virtual network. The service provides both source network address translation (SNAT) for outbound connections and destination network address translation (DNAT) for response packets to outbound-originated connections only. 

<!-- PG: Can we add just blurb here about the redundancy and reliability of NAT Gateway? -->

This article describes reliability and availability zones support in Azure NAT Gateway. For a more detailed overview of reliability in Azure, see [Azure reliability](/azure/reliability/overview).

## Production deployment recommendations

For production deployments, you should:

- Use the StandardV2 SKU, which automatically enables [zone redundancy](#availability-zone-support) in supported regions.
- Configure zone-redundant public IP addresses for your NAT Gateway instances.
- Configure NAT Gateway with enough public IP addresses to handle your peak connection requirements, which reduces the likelihood of availability problems due to SNAT port exhaustion.

## Reliability architecture overview
<!-- PG: I've made some assumptions here about how things work - especially how a single logical NAT Gateway can be mapped to multiple AZs. If I've misunderstood something or used incorrect terminology, we can adjust. -->

NAT Gateway operates at the subnet level, automatically becoming the default route for outbound internet traffic without requiring additional routing configurations.

A NAT Gateway is comprised of one or more *instances*, which represent the underlying compute infrastructure required to operate the service. Internally, NAT Gateway implements a distributed architecture using software-defined networking to provide high reliability and scalability. The service operates with multiple fault domains, enabling it to survive multiple infrastructure component failures without service impact. Azure manages the underlying service operations, including distribution across fault domains and infrastructure redundancy.

For more information about NAT Gateway architecture and redundancy, see [Azure NAT Gateway resource](/azure/nat-gateway/nat-gateway-resource#nat-gateway-architecture).

## Transient faults

[!INCLUDE [Transient fault description](includes/reliability-transient-fault-description-include.md)]

SNAT port exhaustion can manifest as a transient fault in your application. To reduce the likelihood of transient faults related to network address translation, you should:

- **Minimize the likelihood of SNAT port exhaustion.** Exhaustion can happen when an application makes multiple independent connections to the same IP address and port. Configure your applications to handle SNAT port exhaustion gracefully by implementing connection pooling and proper connection lifecycle management.

- **Monitor NAT Gateway's datapath availability metric.** Use Azure Monitor to detect potential connectivity issues early. Set up alerts for connection failures and SNAT port exhaustion to proactively identify and address transient fault conditions before they impact your applications' outbound connectivity. To learn more, see [What is Azure NAT Gateway metrics and alerts?](/azure/nat-gateway/nat-metrics).

- **Avoid setting high idle timeout values.** Idle timeout values that are significantly higher than the default 4 minutes for NAT Gateway connections can contribute to SNAT port exhaustion during high connection volumes.

For comprehensive guidance on connection management and troubleshooting NAT Gateway-specific issues, see [Troubleshoot Azure NAT Gateway connectivity](/azure/nat-gateway/troubleshoot-nat-connectivity).

## Availability zone support

[!INCLUDE [AZ support description](includes/reliability-availability-zone-description-include.md)]

Azure NAT Gateway supports availability zones in both zone-redundant and zonal configurations:

- *Zone-redundant:* When you use the StandardV2 SKU of NAT Gateway, zone redundancy is enabled automatically. Zone redundancy for an Azure NAT Gateway resource spreads instances across multiple [availability zones](../reliability/availability-zones-overview.md). When you use a zone-redundant configuration, you can achieve resiliency and reliability for your production workloads.

- *Zonal:* When you use the Standard (v1) SKU, you can optionally create a zonal configuration by selecting a single availability zone for the Azure NAT Gateway resource. All traffic from connected subnets is routed through the NAT gateway, even if that's in a different availability zone.

     <!-- Allen: If we retain this image, then we need to move it into reliability hub. -->
    :::image type="content" source="./media/nat-availability-zones/zonal-nat-gateway.png" alt-text="Diagram of zonal deployment of NAT gateway.":::
    
    *Figure 1: Zonal deployment of NAT gateway.*

    If a NAT gateway within an availability zone experiences an outage, all virtual machines in the connected subnets fail to connect to the internet.

    You can create *zonal stacks* by deploying subnets for each availability zone, with a dedicated NAT gateway in each subnet. You need to manually assign virtual machines to the relevant availability zone and subnet. 
    
    <!-- Allen: If we retain this image, then we need to move it into reliability hub. -->
    :::image type="content" source="./media/nat-availability-zones/multiple-zonal-nat-gateways.png" alt-text="Diagram of zonal isolation by creating zonal stacks.":::

    *Figure 2: Zonal isolation by creating zonal stacks.*
    
    > [!IMPORTANT]
    > Pinning to a single zone by deploying a zonal NAT gateway doesn’t increase resiliency unless you explicitly deploy resources into multiple zones. To improve resiliency, it's best to use a zone-redundant NAT Gateway.

    <!-- PG: Is there any situation where it might make sense to deploy a zonal NAT Gateway? -->

If you deploy a Standard (v1) NAT Gateway and don't specify an availability zone, the NAT gateway is then *nonzonal*, which means Azure selects the availability zone. A nonzonal configuration doesn't provide protection against availability zone outages.

If you deploy a Standard (v1) NAT Gateway and don't specify an availability zone, the NAT gateway is placed in **no zone** by default. When NAT gateway is placed in **no zone**, Azure places the resource in a zone for you. There isn't visibility into which zone Azure chooses for your NAT gateway. A **no zone** configuration doesn't provide protection against availability zone outages.

### Region support

Zone-redundant and zonal NAT Gateway resources can be deployed into [any region that supports availability zones](./regions-list.md).

### Requirements

- To deploy a zone-redundant NAT Gateway, you must use the StandardV2 SKU.

- The requirements for public IP addresses attached to NAT Gateway depend on whether it's deployed in a zone-redundant or zonal configuration:
    
    - *Zone-redundant:* Public IP addresses attached to a zone-redundant NAT Gateway must also be zone-redundant.
    
    - *Zonal:* Public IP addresses attached to a zonal NAT Gateway must be one of the following:
        - Zone-redundant public IP address.
        - Zonal public IP address in the same availability zone as the NAT Gateway.
        - Nonzonal public IP address, although this isn't recommended.

For guidance on configuring zone-redundant public IP addresses with NAT Gateway, see [Design NAT gateway for high resiliency](../nat-gateway/nat-availability-zones.md).

### Cost

There is no additional cost to use availability zone support for NAT Gateway. For more information about pricing, see [Azure NAT Gateway pricing](https://azure.microsoft.com/pricing/details/azure-nat-gateway/).

### Configure availability zone support

- **New resources:** To deploy a new NAT Gateway resource with availability zone support, see [Create a NAT gateway with availability zones](/azure/nat-gateway/quickstart-create-nat-gateway?tabs=portal).

<!-- Allen: Need to update the create document to reflect AZ options? -->

- **Enable availability zone support:** NAT Gateway availability zone configuration cannot be changed after deployment. To modify the availability zone configuration, you must deploy a new NAT Gateway resource with the desired zone settings.

### Normal operations

This section describes what to expect when Azure NAT Gateway resources are configured for availability zone support and all availability zones are operational.

- **Traffic routing between zones**: The way traffic is routed through your NAT Gateway depends on the availability zone configuration your NAT Gateway uses.

    - *Zone-redundant:* Traffic is routed through an instance within any availability zone.

    - *Zonal:* Each NAT Gateway instance operates independently within its assigned availability zone. Outbound traffic from subnet resources is routed through the NAT Gateway instance in the resource's availability zone.

- **Data replication between zones**: NAT Gateway doesn't perform data replication between zones as it's a stateless service for outbound connectivity. Each NAT Gateway instance operates independently within its availability zone without requiring synchronization with instances in other zones.

### Zone-down experience

This section describes what to expect when an Azure NAT Gateway resource is configured for availability zone support and there's an availability zone outage.

- **Detection and response:**

    - *Zone-redundant:* Azure NAT Gateway detects and responds to failures in an availability zone. You don't need to do anything to initiate an availability zone failover.

    - *Zonal*: You are responsible for implementing application-level failover to alternative connectivity methods or NAT Gateway instances in other zones.

- **Notification:**

    - *Zone-redundant:* Zone outages are visible through Azure Service Health notifications, and Azure Resource Health.

    - *Zonal:* Zone outages are visible through Azure Service Health notifications, Azure Resource Health, and the NAT Gateway datapath availability metric. You can configure alerts on the datapath availability metric to detect connectivity issues.

- **Active requests:**

    - *Zone-redundant:* Active outbound connections are dropped, and clients need to retry. Subsequent connection attempts flow through an instance in another availability zone.

    - *Zonal:* Active outbound connections through a failed zonal NAT Gateway are lost and must be re-established through alternative connectivity paths. Applications should implement retry logic to handle connection failures. Because the outbound public IP address changes, any TCP sessions might need to be renegotiated.

- **Expected data loss:** No data loss occurs because NAT Gateway is a stateless service for outbound connectivity. Connection state is recreated when connections are re-established.

- **Expected downtime:**

    - *Zone-redundant:* No downtime is expected during a zone failover. Clients can retry connections immediately and requests will be routed to an instance in another zone.

    - *Zonal:* Outbound connectivity is lost until you reroute traffic to alternative NAT Gateway instances in other zones, or through alternative connectivity methods.

- **Traffic rerouting:**
    
    - *Zone-redundant:* Future connection requests are routed through a NAT Gateway instance in an availability zone that's online.
    
        It's unlikely that virtual machines in the affected availability zone would still be operating. However, in the event of a partial zone failure that causes NAT Gateway to be unavailable while virtual machines continue to operate, any outbound connections from virtual machines in the affected zone are routed through a NAT Gateway instance in another zone. These connections might have slightly higher latency because they cross multiple zones.

    - *Zonal:* You are responsible for implementing application-level failover to alternative connectivity methods or NAT Gateway instances in other zones.

### Failback

No manual intervention is required for failback operations as NAT Gateway is a stateless service.

When an availability zone recovers, NAT Gateway instances in that zone automatically become available for new outbound connections. Existing connections established through NAT Gateway instances in other zones during the outage continue to use their current connectivity paths until the connections naturally terminate.

### Testing for zone failures

The Azure NAT Gateway platform manages traffic routing, failover, and failback for zone-redundant NAT Gateway resources. Because this feature is fully managed, you don't need to initiate anything or validate availability zone failure processes.

If you use a zonal NAT Gateway, you need to prepare and test failover plans in case a zone failure occurs.

## Multi-region support

Azure NAT Gateway is a single-region service that operates within the boundaries of a specific Azure region. The service doesn't provide native multi-region capabilities or automatic failover between regions. If a region becomes unavailable, NAT Gateway resources in that region are also unavailable.

If you design a networking approach with multiple regions, you should deploy independent NAT Gateway instances into each region.

## Service-level agreement

The service-level agreement (SLA) for Azure NAT Gateway describes the expected availability of the service and the conditions that must be met to achieve that availability expectation. NAT Gateway's SLA only applies when you have configured two or more virtual machine instances, and it excludes SNAT port exhaustion from downtime calculations. To understand the SLA's conditions, it's important that you review the [SLA for Online Services](https://www.microsoft.com/licensing/docs/view/Service-Level-Agreements-SLA-for-Online-Services).

### Related content

- [Azure NAT Gateway overview](/azure/nat-gateway/nat-overview)
- [Design NAT gateway for high resiliency](/azure/nat-gateway/nat-availability-zones)
- [Troubleshoot Azure NAT Gateway connectivity](/azure/nat-gateway/troubleshoot-nat-connectivity)
- [Azure NAT Gateway Resource Health](/azure/nat-gateway/resource-health)
