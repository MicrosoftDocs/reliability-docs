---
title: Reliability in Azure Private Link service
description: Learn how to make Azure Private Link service resilient to a variety of potential outages and problems, including transient faults, availability zone outages, and region outages.
author: glynnniall
ms.author: glynnniall
ms.topic: reliability-article
ms.custom: subject-reliability, references_regions
ms.service: azure-private-link
ms.date: 01/23/2026
---
# Reliability in Azure Private Link service

Azure Private Link service enables you to privately expose your own applications; such as those running on virtual machines, within Azure, allowing other Azure customers or your own networks to connect securely without using public IP addresses. Azure Private Link Service is typically used in combination with an Azure Load Balancer to provide private connectivity to your workloads, ensuring that traffic remains within the Azure network. For more details, see the official [Azure Private Link Service documentation](/azure/private-link/private-link-service-overview).

When you use Azure, reliability is a shared responsibility. Microsoft provides a range of capabilities to support resiliency and recovery. You're responsible for understanding how those capabilities work across all services that you use and selecting the options that meet your business and uptime requirements. Understanding your responsibilities helps ensure that you design for the appropriate level of reliability that your business requires.

This article focuses on the Azure Private Link platform and private endpoints as a connectivity mechanism. It describes platform-level and control-plane behavior during transient faults, availability zone outages, and region-wide outages. Reliability characteristics of specific Azure services accessed via private endpoints are not covered here and are instead documented in each service’s own reliability documentation.

> [!NOTE]
> This article focuses on Azure Private Link service, which you use in combination with a load balancer to enable private connectivity to applications that you run on your own VMs. If you use private endpoints with other Azure services, like Azure Storage or Azure SQL Database, you should instead review that service's reliability guide for any specific reliability information about their private endpoints.

<!-- Note for PG: Does Private Link service Direct Connect change anything or have any reliability implications? -->

[!INCLUDE [Shared responsibility](includes/reliability-shared-responsibility-include.md)]

## Reliability architecture overview

Private link service enables your customers to connect privately to your workloads in Azure. As the service provider, you deploy a Private Link service resource. Service consumers create private endpoints in their own Azure virtual networks. These endpoints connect securely and privately through Private Link to your applications. This setup does not expose public IP addresses.
A Private Link service is usually attached to an Azure Load Balancer that fronts backend resources (virtual machines or virtual machine scale sets). You can also use Private Link service Direct Connect (preview), which enables connectivity to any privately routable IP address within your virtual network.
<!-- Include diagram once approved by John/ Art department-->

Private Link service can be used for software as a service (SaaS) offerings hosted on Azure, and for hybrid environments where on-premises networks require secure, private access to Azure-hosted applications.
- PLS doesn’t operate independently. Availability and resiliency depend on the configuration of all dependent components, including the load balancer, backend virtual machines, and any additional networking services in the traffic path.

> [!IMPORTANT]
> Private Link service Direct Connect is currently in PREVIEW.
> See the [Supplemental Terms of Use for Microsoft Azure Previews](https://azure.microsoft.com/support/legal/preview-supplemental-terms/) for legal terms that apply to Azure features that are in beta, preview, or otherwise not yet released into general availability.

## Resilience to transient faults

[!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)]

Transient faults are short-lived, intermittent failures that occur in distributed cloud environments. They usually resolve themselves after a brief period. PLS does not introduce unique transient fault behaviors, so you can rely on standard Azure patterns for handling these scenarios.

Applications that connect through PLS should follow standard transient fault handling practices. Implement retry logic in client applications to handle temporary connectivity interruptions and review the transient fault guidance for all dependent services.

All cloud-hosted applications should follow the Azure transient fault handling guidance when they communicate with any cloud-hosted APIs, databases, and other components. For more information, see Recommendations for handling transient faults.

## Resilience to availability zone failures
[!INCLUDE [Resilience to availability zone failures](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)]

### Availability zone support

Private link service is automatically resilient to availability zone failures when deployed in a region that supports availability zones. You don't need to configure anything to enable this behavior.

<!-- Diagram here? -->
> [!NOTE]
> Your end-to-end resilience to availability zone failures depends on multiple components. Although Private Link services are resilient to availability zone outages, other components might not be. Review the configuration of your load balancers, VMs, and other components to ensure that you can meet your reliability requirements even during a zone failure.

### Requirements

**Region support:** You can deploy zone redundant Private Link services into [any region that supports availability zones](regions-list.md).

### Cost

There is no additional cost associated with availability zone support for Private Link service.

### Configure availability zone support

Availability zone support is enabled automatically when you deploy Private Link service in a region that supports availablity zones.

### Behavior when all zones are healthy

This section describes what to expect when Private Link services and private endpoints are configured for availability zone support and all availability zones are operational.

- **Traffic routing between zones**: Traffic through a private endpoint and Private Link service might be routed through any availability zone.

- **Data replication between zones**: Azure Private Link doesn't perform data replication between zones as it's a stateless service for connectivity.

### Behavior during a zone failure

This section describes what to expect when Private Link services and private endpoints are configured for availability zone support and there’s an availability zone outage.

- **Detection and response**: Microsoft is responsible for detecting availability zone failures and managing the service response.

- [!INCLUDE [Availability zone down notification (Service Health only)](./includes/reliability-availability-zone-down-notification-service-include.md)]

- **Active requests**: Active requests might be terminated during an availability zone failure. Applications should retry failed requests after transient interruptions, similar to [other transient faults](#resilience-to-transient-faults).

- **Expected data loss**: No data loss occurs because Azure Private Link is a stateless service for connectivity.

- **Expected downtime**: Existing connections that connect through the failed zone might go down. Service consumers can retry connections immediately and requests will be routed through infrastructure in another zone.

- **Traffic rerouting**: When a single availability zone fails, the service continues operating by routing new traffic through healthy zones.

  It's unlikely that virtual machines in the affected availability zone would still be operating. However, in the event of a partial zone failure that causes Azure Private Link to be unavailable in the affected zone while virtual machines in the zone continue to operate, any outbound connections to virtual machines in the affected zone are routed through Private Link infrastructure in another zone.

Application downtime can also occur if dependent components, such as load balancers or backend virtual machines, aren't zone-resilient.

### Zone recovery

When the affected availability zone recovers, Microsoft manages the failback process automatically. No customer action is required.

### Test for zone failures

The Private Link platform manages traffic routing, failover, and failback for Private Link services and private endpoints across availability zones. Because this feature is fully managed, you don't need to validate availability zone failure processes.

## Resilience to region-wide failures

Private Link service is a single-region service. The service doesn't provide native multi-region capabilities or automatic failover between regions. If an Azure region becomes unavailable, PLS resources in that region are also unavailable.

## Custom multi-region solutions for resiliency

If you design a networking approach with multiple regions, you should deploy independent Private Link services and private endpoints into each region. You're responsible for deploying and managing each Private Link service, for routing client traffic to the appropriate Private Link service, and for configuring private endpoints on each Private Link service as you require.

## Backup and recovery

Private Link service doesn't store customer data and doesn't require backup or restore. To recreate configurations, consider maintaining infrastructure-as-code templates for networking resources. Since PLS is configuration-only and stores no customer data, backup efforts should focus on infrastructure-as-code templates for rapid redeployment.

## Service-level agreement

[!INCLUDE [Service-level agreement](includes/reliability-service-level-agreement-include.md)]

## Related content

- [Reliability in Azure](./overview.md)
