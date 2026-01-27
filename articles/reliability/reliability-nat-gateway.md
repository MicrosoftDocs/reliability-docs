---
title: Reliability in Azure NAT Gateway
description: Learn how to make Azure NAT Gateway resilient to a variety of potential outages and problems, including transient faults, availability zone outages, and region outages.
author: glynnniall
ms.author: glynnniall
ms.topic: reliability-article
ms.custom: subject-reliability
ms.service: azure-nat-gateway
ms.date: 01/06/2026
ai-usage: ai-assisted
---

# Reliability in Azure NAT Gateway

[Azure NAT Gateway](/azure/nat-gateway/nat-overview) is a fully managed Network Address Translation (NAT) service that provides outbound internet connectivity for resources connected to your private virtual network. The service provides both Source Network Address Translation (SNAT) for outbound connections and Destination Network Address Translation (DNAT) for response packets to outbound-originated connections only. Because Azure NAT Gateway handles traffic for critical virtual network resources, it's designed to provide high resilience.

[!INCLUDE [Shared responsibility](includes/reliability-shared-responsibility-include.md)]

This article describes how to make Azure NAT Gateway resilient to a variety of potential outages and problems, including transient faults and availability zone outages. It also highlights key information about the Azure NAT Gateway service-level agreement (SLA).

> [!IMPORTANT]
> When you consider the reliability of a NAT gateway, also consider the reliability of your virtual machines (VMs), disks, other network infrastructure, and applications that run on VMs. Improving the resiliency of the NAT gateway might have limited impact if the other components aren't equally resilient. Depending on your resiliency requirements, you might make configuration changes across multiple components.

> [!IMPORTANT]
> The StandardV2 SKU for Azure NAT Gateway is currently in preview.
>
> For legal terms that apply to Azure features in beta, preview, or not yet generally available, see [Supplemental terms of use for Microsoft Azure previews](https://azure.microsoft.com/support/legal/preview-supplemental-terms/).

## Production deployment recommendations

For production workloads, we recommend the following practices:

> [!div class="checklist"]
> - **Use the StandardV2 SKU** to get automatic [zone redundancy](#resilience-to-availability-zone-failures) in supported regions.
>
>   > [!NOTE]
>   > Before deployment, review the [key limitations of StandardV2 Azure NAT Gateway](/azure/nat-gateway/nat-overview#key-limitations-of-standardv2-nat-gateway) to ensure that it supports your configuration.
>
> - **Allocate enough public IP addresses** for your peak connection requirements. Insufficient IP addresses can cause SNAT port exhaustion and availability problems.
>
> - **Use StandardV2 SKU public IP addresses** with StandardV2 Azure NAT Gateway. StandardV2 Azure NAT Gateway doesn't support Standard SKU public IP addresses.

## Reliability architecture overview

[!INCLUDE [Introduction to reliability architecture overview section](includes/reliability-architecture-overview-introduction-include.md)]

### Logical architecture

A *NAT gateway* is a resource that you deploy. To use the NAT gateway as the default route for outbound internet traffic, attach it to one or more subnets in your virtual network. You don't need to configure any custom routes or other routing.

### Physical architecture

Internally, a NAT gateway consists of one or more *instances*, which represent the underlying infrastructure required to operate the service.

Azure NAT Gateway implements a distributed architecture by using software-defined networking (SDN) to provide high reliability and scalability. The service operates across multiple fault domains so that it can withstand multiple infrastructure component failures without service impact. Azure manages the underlying service operations, including distribution across fault domains and infrastructure redundancy.

For more information about Azure NAT Gateway architecture and redundancy, see [Azure NAT Gateway resources](/azure/nat-gateway/nat-gateway-resource#nat-gateway-architecture).

## Resilience to transient faults

[!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)]

*SNAT port exhaustion* occurs when applications make multiple independent connections to the same IP address and port, which exhausts the SNAT ports available for the outbound IP address. SNAT port exhaustion can appear as a transient fault in your application. To reduce the likelihood of transient faults related to NAT, do the following tasks:

- **Minimize the likelihood of SNAT port exhaustion.** Configure your applications to handle SNAT gracefully by implementing connection pooling and proper connection life cycle management.

- **Deploy sufficient public IP addresses.** A single NAT gateway supports multiple public IP addresses, and each public IP address provides a separate set of SNAT ports.

- **Monitor the NAT gateway's datapath availability metric.** Use Azure Monitor to detect potential connectivity problems early. Set up alerts for connection failures and SNAT port exhaustion to proactively identify and address transient fault conditions before they affect your applications' outbound connectivity. For more information, see [Azure NAT Gateway metrics and alerts](/azure/nat-gateway/nat-metrics).

- **Avoid setting high idle timeout values.** When you set idle timeout values significantly higher than the default 4 minutes for NAT gateway connections, SNAT port exhaustion can occur during high connection volumes.

For more information about connection management and troubleshooting problems in Azure NAT Gateway, see [Troubleshoot Azure NAT Gateway connectivity](/azure/nat-gateway/troubleshoot-nat-connectivity).

## Resilience to availability zone failures

[!INCLUDE [Resilience to availability zone failures](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)]

Azure NAT Gateway supports availability zones in both zone-redundant and zonal configurations:

- **Zone-redundant:** The StandardV2 SKU for Azure NAT Gateway provides automatic zone redundancy. Zone redundancy spreads NAT gateway instances across all availability zones in a region. A zone-redundant configuration improves the resiliency and reliability of production workloads.

    :::image type="complex" source="media/reliability-nat-gateway/zone-redundant.svg" alt-text="Diagram of zone-redundant deployment of Azure NAT Gateway." border="false":::
    The diagram shows the internet at the top. Below the internet is a NAT Gateway resource that spans the width of the virtual network. A subnet in the virtual network contains three VMs. Each VM is positioned in a different availability zone. The first VM is in availability zone 1, the second VM is in availability zone 2, and the third VM is in availability zone 3. Three separate arrows in each availability zone indicate the flow of outbound traffic from NAT Gateway to the internet.
    :::image-end:::

- **Zonal:** When you use the Standard (v1) SKU, you can optionally create a zonal configuration. You deploy a zonal NAT gateway in an availability zone that you select. When you deploy a NAT gateway to a specific zone, it provides outbound connectivity to the internet explicitly from that zone. Zonal public IP addresses from a different availability zone aren't allowed. All traffic from connected subnets routes through the NAT gateway, even if subnet resources reside in a different availability zone.

    :::image type="complex" source="media/reliability-nat-gateway/zonal.svg" alt-text="Diagram of a zonal Azure NAT Gateway deployment." border="false":::
    The diagram shows the internet at the top. Below the internet is a NAT Gateway resource deployed within availability zone 1 only. NAT Gateway connects to a subnet that contains a single VM. A virtual network contains the subnet, all located in availability zone 1. Availability zone 2 and availability zone 3 are empty. An arrow indicates the flow of outbound traffic from NAT Gateway to the internet.
    :::image-end:::
    
    If a NAT gateway within an availability zone experiences an outage, all VMs in the connected subnets fail to connect to the internet, even if those VMs reside in healthy availability zones.

    [!INCLUDE [Zonal resource description](includes/reliability-availability-zone-zonal-include.md)]

    If you deploy VMs in several availability zones and need to use zonal NAT gateways, you can create *zonal stacks* in each availability zone. To create zonal stacks, deploy the following resources:

    - *Multiple subnets:* Create a separate subnet for each availability zone rather than using one subnet that spans zones.

    - *Zonal NAT gateways:* Deploy a NAT gateway in the same availability zone as its attached subnet.

    - *Manual VM assignment:* Place each VM in the correct availability zone and the availability zone's corresponding subnet.

    :::image type="complex" source="media/reliability-nat-gateway/zonal-stacks.svg" alt-text="Diagram of zonal isolation by creating zonal stacks." border="false":::
     The diagram shows the internet at the top and three availability zones below the internet. Each availability zone contains its own dedicated NAT Gateway instance, subnet, and VM. All these resources reside in a shared virtual network. Arrows show an outbound traffic flow from each availability zone's NAT Gateway instance to the internet.
    :::image-end:::
    
If you deploy a Standard (v1) NAT gateway and don't specify an availability zone, the NAT gateway is *nonzonal*, so Azure selects the availability zone. If any availability zone in the region has an outage, it might affect your NAT gateway. We don't recommend a nonzonal configuration because it doesn't provide protection against availability zone outages.

### Requirements

- **Region support:** You can deploy zone-redundant and zonal NAT gateways into [any region that supports availability zones](./regions-list.md).

- **SKU:** To deploy a zone-redundant NAT gateway, use the StandardV2 SKU. To deploy a zonal NAT gateway, use the Standard SKU. We recommend the StandardV2 SKU.

- **Public IP addresses:** The requirements for public IP addresses attached to a NAT gateway depend on the SKU and deployment configuration.

    | Azure NAT Gateway SKU | Availability zone support type | Public IP address requirements |
    | --- | --- | --- |
    | StandardV2 | Zone-redundant | StandardV2 public IP address |
    | Standard | Zonal | Standard public IP address that's zone redundant or in the same zone as the NAT gateway |
    | Standard | Nonzonal | Standard public IP address that's zone redundant or in any zone |

### Cost

Availability zone support for Azure NAT Gateway has no extra cost. For more information, see [Azure NAT Gateway pricing](https://azure.microsoft.com/pricing/details/azure-nat-gateway/).

### Configure availability zone support

- **New resources:** The deployment steps depend on your availability zone configuration:

    - *Zone-redundant:* To deploy a new zone-redundant NAT gateway by using the StandardV2 SKU, see [Create a StandardV2 Azure NAT Gateway](/azure/nat-gateway/quickstart-create-nat-gateway-v2).

    - *Zonal:* To deploy a new zonal NAT gateway by using the Standard SKU, see [Create a NAT gateway](/azure/nat-gateway/quickstart-create-nat-gateway). When you create the NAT gateway, select its availability zone instead of selecting **No zone**.

- **Turn on availability zone support:** You can't change the availability zone configuration for Azure NAT Gateway after deployment. To modify the availability zone configuration, you must deploy a new NAT gateway with the desired zone settings.

    To upgrade from a Standard to StandardV2 NAT gateway, you must create a new public IP address that uses the StandardV2 SKU.

### Behavior when all zones are healthy

This section describes what to expect when NAT gateways are configured for availability zone support and all availability zones are operational.

- **Traffic routing between zones:** The way traffic from your VM routes through your NAT gateway depends on the availability zone configuration that your NAT gateway uses.

    - *Zone-redundant:* Traffic can route through a NAT gateway instance within any availability zone.

    - *Zonal:* Each NAT gateway instance operates independently within its assigned availability zone. Outbound traffic from subnet resources routes through the NAT gateway's zone, even if the VM resides in a different zone.

- **Data replication between zones:** Azure NAT Gateway doesn't replicate data between zones because it's a stateless service for outbound connectivity. Each NAT gateway instance operates independently within its availability zone and doesn't require synchronization with instances in other zones.

### Behavior during a zone failure

This section describes what to expect when a NAT gateway is configured for availability zone support and there's an availability zone outage.

- **Detection and response:** Responsibility for detection and response depends on the availability zone configuration that your NAT gateway uses.

    - *Zone-redundant:* Azure NAT Gateway detects and responds to failures in an availability zone. You don't need to do anything to initiate an availability zone failover.

    - *Zonal:* You must implement application-level failover to alternative connectivity methods or NAT gateways in other zones.

- **Notification:** [!INCLUDE [Availability zone down notification partial bullet (Service Health and Resource Health)](./includes/reliability-availability-zone-down-notification-service-resource-partial-include.md)]

    You can also use the NAT gateway's datapath availability metric to monitor the health of your NAT gateway. Configure alerts on the datapath availability metric to detect connectivity problems.

- **Active requests:** Active request behavior depends on the availability zone configuration that your NAT gateway uses.

    - *Zone-redundant:* Instances in the faulty zone drop active outbound connections. Clients must retry their connection requests, and subsequent attempts route through a NAT gateway instance in another availability zone.

    - *Zonal:* A failed zonal NAT gateway drops active outbound connections. You must decide whether and how to re-establish connectivity through alternative connectivity paths. Applications should implement retry logic to handle connection failures.
    
        If you reroute traffic, the outbound public IP address changes, so any Transmission Control Protocol (TCP) sessions might need to be re-established.

- **Expected data loss:** No data loss occurs because Azure NAT Gateway is a stateless service for outbound connectivity. Connection state is re-created when connections are re-established.

- **Expected downtime:** The expected downtime depends on the availability zone configuration that your NAT gateway uses.

    - *Zone-redundant:* Existing connections from the failed zone might disconnect. Clients can retry connections immediately, and requests route to an instance in another zone. All remaining connections from healthy zones persist.

    - *Zonal:* Outbound connectivity remains unavailable until the zone recovers or until you reroute traffic through alternative connectivity methods or NAT gateways in other zones.

- **Traffic rerouting:** The traffic rerouting behavior depends on the availability zone configuration that your NAT gateway uses. 
    
    - *Zone-redundant:* New connection requests route through a NAT gateway instance in a healthy availability zone.
    
        During a zone failure, VMs in that zone also typically stop operating. But if a partial zone failure affects only the NAT gateway while VMs continue to run, outbound connections from those VMs route through a NAT gateway instance in another zone.

    - *Zonal:* You must implement application-level failover, like using alternative connectivity methods or redirecting traffic to NAT gateways in other zones.

### Zone recovery

Azure NAT Gateway is a stateless service, so failback operations don't require manual intervention.

When an availability zone recovers, NAT gateway instances in that zone automatically become available for new outbound connections. Connections established through NAT gateway instances in other zones during the outage continue to use their existing connectivity paths until those connections close.

### Test for zone failures

The options for testing for zone failures depend on the availability zone configuration that your instance uses.

- *Zone-redundant:* The Azure NAT Gateway platform manages traffic routing, failover, and failback for zone-redundant NAT gateways. These managed features don't require you to initiate any manual actions or validate availability zone failure processes.

- *Zonal:* You must prepare and test failover plans to handle a potential zone failure.

## Resilience to region-wide failures

Azure NAT Gateway is a single-region service that operates within the boundaries of a specific Azure region. The service doesn't provide native multi-region capabilities or automatic failover between regions. If a region becomes unavailable, NAT gateways in that region are also unavailable.

If you design a networking approach that spans multiple regions, deploy independent NAT gateways in each region.

## Service-level agreement

[!INCLUDE [SLA description](includes/reliability-service-level-agreement-include.md)]

The Azure VNet NAT SLA covers Azure NAT Gateway. The availability SLA only applies when you have two or more healthy VMs, and it excludes SNAT port exhaustion from downtime calculations.

### Related content

- [Azure NAT Gateway overview](/azure/nat-gateway/nat-overview)
- [Reliability in Azure NAT Gateway](/azure/reliability/reliability-nat-gateway)
- [Troubleshoot Azure NAT Gateway connectivity](/azure/nat-gateway/troubleshoot-nat-connectivity)
- [Azure NAT Gateway and Resource Health](/azure/nat-gateway/resource-health)
