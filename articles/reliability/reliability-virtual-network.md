---
title: Reliability in Azure Virtual Network
description: Learn about resiliency in Azure Virtual Network, including resilience to transient faults, availability zone failures, and region-wide failures. Understand backup options and SLA details.
author: asudbring
ms.author: allensu
ms.topic: reliability-article
ms.custom: subject-reliability
ms.service: azure-virtual-network
ms.date: 06/10/2026
ms.update-cycle: 1095-days
---

# Reliability in Azure Virtual Network

[Azure Virtual Network](/azure/virtual-network/virtual-networks-overview) is a logical representation of your network in the cloud. You can use a virtual network to define your own private IP address space and segment the network into subnets. Virtual networks serve as a trust boundary to host your compute resources such as Azure Virtual Machines and load balancers.

[!INCLUDE [Shared responsibility](includes/reliability-shared-responsibility-include.md)]

This article describes how to make Virtual Network resilient to a variety of potential outages and problems, including transient faults, availability zone outages, and region outages. It also describes how you can use backups to recover from other types of problems, and highlights some key information about the Virtual Network service level agreement (SLA).

## Production deployment recommendations

For more information about how to deploy virtual networks to support your solution's reliability requirements and how reliability affects other aspects of your architecture, see [Architecture best practices for Azure Virtual Network in the Azure Well-Architected Framework](/azure/well-architected/service-guides/virtual-network).

## Reliability architecture overview

A virtual network is one of several core networking components in Azure. When you create a virtual network, you create a set of resources that collectively define your networking configuration. These resources include the following network components:

- NSGs and application security groups, which restrict communication between parts of your network

- User-defined routes, which control how traffic flows

- Load balancers, which distribute traffic within your network

- Public IP addresses, which provide connectivity to and from the internet

- Network interface cards, which provide network connectivity to Azure virtual machines (VMs)

- Private endpoints, which provide private connectivity to Azure services and to resources outside of your virtual network

A virtual network enables direct private IP communication between the resources that it hosts. To enable hybrid cloud scenarios and securely extend your datacenter into Azure, you can link a virtual network to an on-premises network through Azure VPN Gateway or Azure ExpressRoute.

You might also deploy *appliances*, such as ExpressRoute gateways, VPN gateways, and firewalls. Appliances provide services to support your networking requirements, such as connecting to on-premises environments or providing sophisticated controls on traffic flow.

Finally, you deploy your own components, like VMs that run applications or databases, and other Azure services that provide virtual network integration.

For more information about networking in Azure, see [Networking architecture design](/azure/architecture/networking/).

## Resilience to transient faults

[!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)]

Transient faults don't usually affect virtual networks. However, transient faults might affect resources deployed within a virtual network. Review the [reliability guide for each resource](./overview-reliability-guidance.md) that you use to understand their transient fault handling behaviors.

## Resilience to availability zone failures

[!INCLUDE [Resilience to availability zone failures](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)]

A virtual network and its subnets span all availability zones within the region where you deploy them, so they're automatically zone redundant. You don't need to configure anything to enable zone redundancy.

You don't need to divide your virtual networks or subnets by availability zones to accommodate zonal resources. For example, if you configure a zonal VM, you don't have to consider the virtual network when you select the availability zone for the VM. The same is true for other zonal resources.

### Requirements

**Region support:** You can deploy zone-redundant virtual networks into any [region that supports availability zones](./regions-list.md).

### Cost

There's no extra cost for zone redundancy for Azure virtual networks.

### Configure availability zone support

Zone redundancy is configured automatically when a virtual network is deployed in a region that supports availability zones.

### Behavior when all zones are healthy

This section describes what to expect when you deploy a virtual network in a region that supports availability zones, and all zones are operational.

- **Cross-zone operation:** A virtual network and its subnets span all availability zones in the region. Resources deployed in any zone use the same virtual network and address space. Private IP traffic flows directly between resources, even when the resources are in different availability zones.

- **Cross-zone data replication:** Virtual Network is a stateless networking service. Configuration is replicated synchronously across zones, which ensures consistent configuration in every zone.

### Behavior during a zone failure

This section describes what to expect when you deploy a virtual network in a region that supports availability zones and there's an outage in one of the zones.

> [!IMPORTANT]
> Consider each resource within the virtual network individually, because each resource might have a different set of behaviors during the loss of an availability zone. Review the [reliability guide for each resource](./overview-reliability-guidance.md) that you use to understand their availability zone support and behavior when a zone is unavailable.

- **Detection and response:** The Azure platform detects availability zone failures and responds automatically. You don't need to do anything to initiate a zone failover for the virtual network itself.

[!INCLUDE [Availability zone down notification (Service Health only)](./includes/reliability-availability-zone-down-notification-service-include.md)]

- **Active requests:** Virtual Network is a logical construct and doesn't terminate connections. Active flows between resources in healthy zones continue to operate. Flows to or from resources in the affected zone are interrupted because the underlying resources are unavailable, not because of the virtual network. Clients should implement [transient fault handling](#resilience-to-transient-faults), including automated retries.

- **Expected data loss:** Virtual Network doesn't store data, so no data loss occurs at the virtual network layer.

- **Expected downtime:** No downtime is expected for the virtual network itself. The virtual network continues to operate from the healthy zones, and its configuration remains available.

    However, resources deployed within the virtual network might be unavailable depending on their own availability zone configuration.

- **Redistribution:** When a zone becomes unavailable, Virtual Network automatically reroutes virtual network requests to the remaining zones. This process doesn't require any action from you.

    Redistribution behavior for resources deployed within the virtual network depends on each resource type. For more information, see the [reliability guide for each resource](./overview-reliability-guidance.md).

### Zone recovery

When the zone recovers, Microsoft initiates a failback process to ensure that virtual networks continue to work in the recovered zone. The failback process is automatic and doesn't require any action from you.

However, you should verify the failback behaviors of any resources that you deploy within the virtual network. For more information, see the [reliability guide for each resource](./overview-reliability-guidance.md).

### Test for zone failures

The Virtual Network platform manages traffic routing, failover, and failback for virtual networks across availability zones. Because this feature is fully managed, you don't need to validate availability zone failure processes.

## Resilience to region-wide failures

Virtual Network is a single-region service. If the region becomes unavailable, your virtual network is also unavailable.

### Custom multi-region solutions for resiliency

You can create virtual networks in multiple regions. You can also choose to connect those networks by *peering* them together.

By creating virtual networks and other resources in multiple regions, you can be resilient to regional outages. However, you need to consider the following factors:

- **Traffic routing:** If you host internet-facing services in the virtual network, you need to decide how to route incoming traffic among your regions and components. With services such as Azure Traffic Manager and Azure Front Door, you can route internet traffic based on rules that you specify.

- **Failover:** If an Azure region is unavailable, you typically need to fail over by processing traffic in healthy regions. Traffic Manager and Azure Front Door provide failover capabilities for internet applications.      

- **Management:** Each virtual network is a separate resource and needs to be configured and managed independently from other virtual networks.

- **IP address space:** Determine how to allocate IP addresses when you create multiple virtual networks. You can create multiple virtual networks by using the same private IP address space in different regions. However, you can't peer, or connect, two virtual networks with the same address space to your on-premises network because it causes routing problems. If you plan to create a multi-network design, IP address planning is an important consideration.

Virtual networks don't require a lot of resources to run. You can invoke Azure APIs to create a virtual network with the same address space in a different region. However, to recreate a similar environment to the one that exists in the affected region, you must redeploy the VMs and other resources. If you have on-premises connectivity, such as in a hybrid deployment, you have to deploy a new VPN Gateway instance and connect to your on-premises network.

For more information about a multi-region networking architecture for web applications, see [Multi-region load balancing with Traffic Manager, Azure Firewall, and Azure Application Gateway](/azure/architecture/high-availability/reference-architecture-traffic-manager-application-gateway).

## Backup and restore

Virtual networks don't store any data that requires backup. However, you can use Bicep, Azure Resource Manager templates, or Terraform to take a snapshot of the configuration of a virtual network if you need to recreate it. For more information, see [Create an Azure virtual network](/azure/virtual-network/quickstart-create-virtual-network).

## Resilience to service maintenance

[!INCLUDE [Service maintenance (no special callouts)](includes/reliability-maintenance-include.md)]

## Service-level agreement

[!INCLUDE [SLA description](includes/reliability-service-level-agreement-include.md)]

Because of the nature of the service provided, there isn't a defined service-level agreement for Virtual Network.

## Related content

- [Availability zones](availability-zones-overview.md)
