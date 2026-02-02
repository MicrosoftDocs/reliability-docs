---
title: Reliability in Azure Private Link service
description: Learn how to make Azure Private Link service resilient to a variety of potential outages and problems, including transient faults, availability zone outages, and region outages.
author: glynnniall
ms.author: glynnniall
ms.topic: reliability-article
ms.custom: subject-reliability
ai-usage: ai-assisted
ms.service: azure-private-link
ms.date: 01/30/2026
---

# Reliability in Azure Private Link service

[Azure Private Link service](/azure/private-link/private-link-service-overview) enables you to privately expose your own applications, such as those running on virtual machines, within an Azure virtual network. Private Link service enables other Azure customers or clients on your own networks to connect securely without using public IP addresses, ensuring that traffic remains within the Azure network.

[!INCLUDE [Shared responsibility](includes/reliability-shared-responsibility-include.md)]

This article focuses on the Azure Private Link service and the associated private endpoints as a connectivity mechanism. It describes platform-level and control-plane behavior during transient faults, availability zone outages, and region-wide outages.

> [!NOTE]
> This article focuses on Azure Private Link service, which you use in combination with a load balancer to enable private connectivity to applications that you run on your own VMs. If you use private endpoints with other Azure services, for example Azure Storage or Azure SQL Database, you should instead review those services' reliability guides for any specific reliability information about their private endpoints.

> [!IMPORTANT]
> The reliability of your overall solution depends on the configuration of the backend servers that Private Link service connects to. Depending on your solution, these might be Azure virtual machines (VMs), Azure virtual machine scale sets, or external endpoints. It also includes load balancers and other network components.
>
> Your backend servers aren't in scope for this article, but their availability configurations directly affect your application's resilience. Review the reliability guides for all of the Azure services in your solution to understand how each service supports your reliability requirements. By ensuring that your backend servers are also configured for high availability and zone redundancy, you can achieve end-to-end reliability for your application.

## Reliability architecture overview

Private Link service enables your customers to connect privately to your workloads in Azure. As the *service provider*, you deploy a *Private Link service* resource. *Service consumers* create *private endpoints* in their own Azure virtual networks. These endpoints connect securely and privately through Private Link to your applications. This setup does not expose public IP addresses.

A Private Link service is usually attached to an Azure Load Balancer that fronts backend resources (virtual machines or virtual machine scale sets). You can also use [Private Link service Direct Connect (preview)](/azure/private-link/configure-private-link-service-direct-connect), which enables connectivity to any privately routable IP address within your virtual network.

> [!WARNING]
> **Note for PG:** Are there any reliability-specific recommendations related to Direct Connect, or does it change anything about the standard behavior (like zone distribution)? We've assumed no changes.

> [!IMPORTANT]
> Private Link service Direct Connect is currently in PREVIEW.
> See the [Supplemental Terms of Use for Microsoft Azure Previews](https://azure.microsoft.com/support/legal/preview-supplemental-terms/) for legal terms that apply to Azure features that are in beta, preview, or otherwise not yet released into general availability.

:::image type="content" source="/azure/private-link/media/private-link-service-overview/private-link-service-workflow.png" alt-text="Diagram showing a Private Link service deployed by a service provider into their virtual network, with incoming traffic from a service consumer in a different virtual network in a separate Microsoft Entra tenant." border="false" :::

Private Link service can be used for software as a service (SaaS) offerings hosted on Azure, and for hybrid environments where on-premises networks require secure, private access to Azure-hosted applications.

## Resilience to transient faults

[!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)]

Additionally, when you deploy a Private Link service with a Standard Load Balancer, review the [transient fault handling recommendations for Azure Load Balancer](./reliability-load-balancer.md#resilience-to-transient-faults) and ensure your load balancer is configured correctly to handle transient faults.

## Resilience to availability zone failures

Private Link service is automatically resilient to availability zone failures when deployed in a region that supports availability zones. Service providers don't need to configure anything to enable this behavior.

:::image type="content" source="./media/reliability-private-link-service/private-link-service-zone-redundant.png" alt-text="Diagram showing a zone-redundant Private Link service and public load balancer, directing traffic to three different VMs in different availability zones." border="false" :::

Private endpoints are automatically distributed across availability zones in the region. Service consumers don't need to create separate private endpoints in different zones.

### Requirements

- **Region support:** You can deploy zone redundant Private Link services into [any region that supports availability zones](regions-list.md).

- **Load balancer dependency:** Your backend load balancer should also be configured as zone-redundant to ensure end-to-end zone resiliency. For more information, see [Reliability in Azure Load Balancer](./reliability-load-balancer.md).

### Cost

There is no additional cost associated with availability zone support for Private Link service.

### Configure availability zone support

Availability zone support is enabled automatically when you deploy Private Link service in a region that supports availability zones.

### Behavior when all zones are healthy

This section describes what to expect when Private Link services and private endpoints are configured for availability zone support and all availability zones are operational.

- **Traffic routing between zones**: Traffic through a private endpoint and Private Link service might be routed through any availability zone.

- **Data replication between zones**: Azure Private Link doesn't perform data replication between zones as it's a stateless service for connectivity.

### Behavior during a zone failure

This section describes what to expect when Private Link services and private endpoints are configured for availability zone support and there’s an availability zone outage.

- **Detection and response**: Microsoft is responsible for detecting availability zone failures and managing the service response.

[!INCLUDE [Availability zone down notification (Service Health only)](./includes/reliability-availability-zone-down-notification-service-include.md)]

- **Active requests**: Active requests might be terminated during an availability zone failure. Service consumers should retry failed requests after transient interruptions, similar to [other transient faults](#resilience-to-transient-faults).

- **Expected data loss**: No data loss occurs because Azure Private Link is a stateless service for connectivity.

- **Expected downtime**: Existing connections that connect through the failed zone might go down. As long as backend components, like the load balancer and application servers, are still available, service consumers can retry connections immediately and requests will be routed through infrastructure in another zone.

- **Traffic rerouting**: When a single availability zone fails, the service continues operating by routing new traffic through healthy zones.

  It's unlikely that virtual machines in the affected availability zone would still be operating. However, in the event of a partial zone failure that causes Azure Private Link to be unavailable in the affected zone while virtual machines in the zone continue to operate, any outbound connections to virtual machines in the affected zone are routed through Private Link infrastructure in another zone.

Application downtime can also occur if dependent components, such as load balancers or backend virtual machines, aren't zone-resilient.

### Zone recovery

When the affected availability zone recovers, Microsoft manages the failback process automatically. No customer action is required.

### Test for zone failures

The Private Link platform manages traffic routing, failover, and failback for Private Link services and private endpoints across availability zones. Because this feature is fully managed, you don't need to validate availability zone failure processes.

## Resilience to region-wide failures

Private Link service is a single-region service. The service doesn't provide native multi-region capabilities or automatic failover between regions. If an Azure region becomes unavailable, Private Link services in that region are also unavailable.

### Custom multi-region solutions for resiliency

If you design a networking approach with multiple regions, you (the service provider) should deploy independent Private Link services into each region. You're responsible for deploying and managing each Private Link service. Service consumers are responsible for configuring private endpoints on each Private Link service as required, and for routing traffic to the appropriate Private Link service.

## Backup and recovery

Private Link service doesn't store customer data and doesn't require backup or restore. To recreate configurations, consider maintaining infrastructure-as-code templates for networking resources. Since Private Link services are configuration-only and store no customer data, backup efforts should focus on infrastructure-as-code templates for rapid redeployment.

## Service-level agreement

[!INCLUDE [Service-level agreement](includes/reliability-service-level-agreement-include.md)]

## Related content

- [Reliability in Azure](./overview.md)
- [Reliability in Azure Load Balancer](./reliability-load-balancer.md)
