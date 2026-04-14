---
title: Reliability in Azure Private Link Service
description: Learn how to make Azure Private Link service resilient to a variety of potential outages and problems, including transient faults, availability zone outages, and region outages.
author: AbdullahBell
ms.author: abell
ms.topic: reliability-article
ms.custom: subject-reliability
ai-usage: ai-assisted
ms.service: azure-private-link
ms.date: 02/20/2026
---

# Reliability in Azure Private Link service

[Azure Private Link service](/azure/private-link/private-link-service-overview) helps you privately expose your own applications, such as applications that run on virtual machines (VMs), within an Azure virtual network. Private Link service helps other Azure customers or clients on your networks connect securely without public IP addresses, which ensures that traffic remains within the Azure network.

[!INCLUDE [Shared responsibility](includes/reliability-shared-responsibility-include.md)]

This article focuses on Private Link service and the associated private endpoints as a connectivity mechanism. It describes platform-level and control-plane behavior during transient faults, availability zone outages, and region-wide outages.

> [!NOTE]
> This article focuses on Private Link service, which facilitates private connectivity to applications that you run on your own VMs. If you use private endpoints with other Azure services, for example Azure Storage or Azure SQL Database, you should instead review the reliability guides for those services for reliability information about their private endpoints.

> [!IMPORTANT]
> The reliability of your overall solution depends on the configuration of the backend servers that Private Link service connects to. These backend servers might be Azure VMs, Azure virtual machine scale sets, or external endpoints. The reliability of your solution also depends on the configuration of load balancers and other network components.
>
> Your backend servers aren't in scope for this article, but their availability configurations directly affect your application's resilience. To understand how each service supports your reliability requirements, review the reliability guides for all Azure services in your solution. You can achieve end-to-end reliability for your application by ensuring that your backend servers are also configured for high availability and zone redundancy.

## Reliability architecture overview

Private Link service helps your customers connect privately to your workloads in Azure. As the *service provider*, you deploy a *Private Link service* resource. *Service consumers* create *private endpoints* in their own Azure virtual networks. These endpoints connect securely and privately to your applications through Azure Private Link. This setup doesn't expose public IP addresses, even when a consumer uses the private endpoint from an on-premises environment through Azure ExpressRoute or another private connectivity method.

:::image type="complex" source="./media/reliability-private-link-service/architecture.svg" border="false" lightbox="./media/reliability-private-link-service/architecture.svg" alt-text="Diagram that shows a Private Link service in a service provider's virtual network. Traffic enters the Private Link service from a service consumer in a different virtual network in a separate Microsoft Entra tenant."::: 
    Diagram that shows a network connection between an on‑premises environment and two Azure virtual networks that sit in separate tenants and regions. On the left, an on‑premises network connects to Azure through ExpressRoute private peering and an ExpressRoute gateway. This path leads into a consumer virtual network. The consumer virtual network contains a subnet and a private endpoint. A network security group controls this subnet and denies outbound traffic. The diagram labels the subnet with the address range 10.0.1.0/24 and labels the virtual network with the address range 10.0.0.0/16. Traffic moves across the Microsoft network through Private Link. This connection leads to a provider virtual network on the right side of the diagram. The provider virtual network contains a Private Link service that uses a NAT IP address of 192.168.0.5. This service connects to a Standard Load Balancer that has a frontend IP address of 192.168.0.10. The load balancer distributes traffic to a virtual machine scale set that contains virtual machines with IP addresses such as 192.168.0.1 and 192.168.0.2. A network security group controls the provider virtual network and denies inbound traffic. The diagram labels the provider subnet with the IP address range 192.168.0.0/24 and labels the provider virtual network with the IP address range 192.168.0.0/16. The diagram shows that the consumer network is deployed in Region A and is within Subscription A in Microsoft Entra tenant A. The diagram shows that the provider network is deployed in Region B and is within Subscription B in Microsoft Entra tenant B.
:::image-end::: 

A Private Link service is typically attached to an Azure load balancer that fronts backend resources, like VMs or virtual machine scale sets. You can also use [Private Link service Direct Connect (preview)](/azure/private-link/configure-private-link-service-direct-connect), which facilitates connectivity to any privately routable IP address within your virtual network. If you use Private Link service Direct Connect, review the documentation carefully to understand requirements, region availability, and limitations.

> [!IMPORTANT]
> Private Link service Direct Connect is currently in preview.
>
> See the [Supplemental Terms of Use for Microsoft Azure Previews](https://azure.microsoft.com/support/legal/preview-supplemental-terms/) for legal terms that apply to Azure features that are in beta, preview, or are otherwise not generally available.

## Resilience to transient faults

[!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)]

When you deploy a Private Link service with Standard Load Balancer, review the [transient fault handling recommendations for Azure Load Balancer](./reliability-load-balancer.md#resilience-to-transient-faults) and ensure that your load balancer is configured to handle transient faults.

## Resilience to availability zone failures

Private Link service is automatically resilient to availability zone failures when deployed in a region that supports availability zones. Service providers don't need to configure anything to enable this behavior.

:::image type="complex" source="./media/reliability-private-link-service/zone-redundant.svg" border="false" alt-text="Diagram that shows three availability zones with a public load balancer and Private Link service distributed across all zones. The load balancer directs traffic to VMs."::: 
    Diagram that shows three vertical sections arranged side by side that represent three separate availability zones. A zone-redundant internal load balancer and Private Link service span all three zones. Each zone has a backend instance. Private Link service connects to the load balancer, which connects to all backend instances. 
:::image-end::: 

Private endpoints are automatically distributed across availability zones in the region. Service consumers don't need to create separate private endpoints in different zones.

### Requirements

- **Region support:** You can deploy zone-redundant Private Link services in [any region that supports availability zones](regions-list.md).

- **Load balancer dependency:** If you use Private Link service with a backend load balancer, also configure the load balancer as zone redundant to ensure end-to-end zone resiliency. For more information, see [Reliability in Load Balancer](./reliability-load-balancer.md).

### Cost

There's no extra cost associated with availability zone support for Private Link service.

### Configure availability zone support

Availability zone support is automatically enabled when you deploy Private Link service in a region that supports availability zones.

### Behavior when all zones are healthy

This section describes what to expect when Private Link services and private endpoints support availability zones, and all zones are operational.

- **Cross-zone operation:** Traffic through a private endpoint and Private Link service might be routed through any availability zone.

- **Cross-zone data replication:** Private Link doesn't perform data replication between zones because it's a stateless service for connectivity.

### Behavior during a zone failure

This section describes what to expect when you Private Link services and private endpoints support availability zones, and there’s an outage in one of the zones.

- **Detection and response:** Microsoft is responsible for detecting availability zone failures and managing the service response.

[!INCLUDE [Availability zone down notification (Service Health only)](./includes/reliability-availability-zone-down-notification-service-include.md)]

- **Active requests:** Active requests might be terminated during an availability zone failure. Service consumers should retry failed requests after transient interruptions, similar to [other transient faults](#resilience-to-transient-faults).

- **Expected data loss:** No data loss occurs because Private Link is a stateless service for connectivity.

- **Expected downtime:** Existing connections that connect through the failed zone might fail. If backend components such as the load balancer and application servers remain available, service consumers can immediately retry the connection, and requests are routed through infrastructure in another zone.

- **Redistribution:** When a single availability zone fails, the service routes new traffic through healthy zones, which continues operation.

  VMs in the affected availability zone are unlikely to remain operational during a zone outage. However, if a partial zone failure makes Private Link unavailable in the affected zone while VMs in that zone continue to operate, outbound connections to VMs in the affected zone are routed through Private Link infrastructure in another zone.

Application downtime can also occur if dependent components, such as load balancers or backend VMs, aren't zone resilient.

### Zone recovery

When the affected availability zone recovers, Microsoft automatically manages the failback process. No customer action is required.

### Test for zone failures

The Private Link platform manages traffic routing, failover, and failback for Private Link services and private endpoints across availability zones. Because this feature is fully managed, you don't need to validate availability zone failure processes.

## Resilience to region-wide failures

Private Link service is a single-region service. The service doesn't provide native multi-region capabilities or automatic failover between regions. If an Azure region becomes unavailable, Private Link services in that region are also unavailable.

### Custom multi-region solutions for resiliency

If you design a networking approach that spans multiple regions, you should deploy independent Private Link services into each region. You're responsible for Private Link service deployment and management. Service consumers are responsible for private endpoint configuration on each region's Private Link services. Service consumers are also responsible for routing traffic to the appropriate Private Link service.

## Backup and restore

Private Link service doesn't store customer data and doesn't require backup or restore. To recreate configurations, consider maintaining infrastructure as code (IaC) templates for networking resources. Private Link services are configuration-only and store no customer data, so backup efforts should focus on IaC templates for rapid redeployment.

## Service-level agreement

[!INCLUDE [Service-level agreement](includes/reliability-service-level-agreement-include.md)]

## Related content

- [Reliability in Azure](./overview.md)
- [Reliability in Load Balancer](./reliability-load-balancer.md)
