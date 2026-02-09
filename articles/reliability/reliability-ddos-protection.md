---
title: Reliability in Azure DDoS Protection
description: Learn how Azure DDoS Protection is resilient to a variety of potential outages and problems, including transient faults, availability zone outages, and region outages.
author: AbdullahBell
ms.author: abell
ms.topic: reliability-article
ms.custom: subject-reliability
ms.service: azure-ddos-protection
ms.date: 02/09/2026
---

# Reliability in Azure DDoS Protection

[Azure DDoS Protection](/azure/ddos-protection/ddos-protection-overview) is a foundational Azure networking capability that helps protect applications from distributed denial-of-service (DDoS) attacks. DDoS attacks attempt to overwhelm applications with traffic in order to deny service to legitimate users. Azure DDoS Protection helps safeguard applications by monitoring network traffic patterns and automatically mitigating abnormal traffic that could impact availability.

[!INCLUDE [Shared responsibility](includes/reliability-shared-responsibility-include.md)]

This article describes how Azure DDoS Protection contributes to workload resilience, including how the service behaves during transient faults, availability zone failures, and region-wide failures.

## Reliability architecture overview
<!-- TODO -->

Azure DDoS Protection operates as part of the Azure networking fabric rather than as a customer-deployed resource. When you enable the service, Microsoft reconfigures the underlying Azure network infrastructure rather than provisioning dedicated customer-isolated compute.


Key architectural characteristics include:

- **Built into the Azure platform fabric.** Microsoft manages the DDoS Protection infrastructure, making it a highly resilient foundational service.
- **Shared, Microsoft-managed infrastructure.** The service uses shared infrastructure rather than customer-isolated compute.
- **Largely stateless request processing.** Traffic analysis and model training occur outside the hot path, which means there's minimal state to replicate or recover during failures.

<!-- TODO IP vs. network -->

## Resilience to transient faults

[!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)]

Azure DDoS Protection runs at the network fabric layer, and the source material doesn't identify service-specific transient fault scenarios that require customer action.

- Transient faults may occur in the broader Azure networking stack, but no DDoS-specific retry or mitigation logic is required from customers.
- Customers don't implement retry logic directly against Azure DDoS Protection.

## Resilience to availability zone failures

[!INCLUDE [Resilience to availability zone failures](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)]

Azure DDoS Protection is zone-redundant by default in regions that support availability zones. The service spans all availability zones automatically and requires no customer configuration to enable zone redundancy. Microsoft manages the distribution of DDoS Protection infrastructure across zones.

![Diagram that shows a DDoS Protection plan that spans multiple availability zones](./media/reliability-ddos-protection/zone-redundant.png)

### Requirements

**Region support:** Azure DDoS Protection is zone-redundant in [any region that supports availability zones](/azure/reliability/regions-list).

### Cost

There's no additional cost to enable zone redundancy for Azure DDoS Protection. Zone redundancy is automatically included. For more information about pricing, see [Azure DDoS Protection pricing](https://azure.microsoft.com/pricing/details/ddos-protection/).

### Configure availability zone support

Azure DDoS Protection is automatically zone-redundant in supported regions. There's no configuration required to enable zone redundancy, and you can't disable it.

### Behavior when all zones are healthy

This section describes what to expect when you use Azure DDoS Protection in a region with availability zones, and all availability zones are operational.

- **Traffic routing between zones:** Traffic inspection can occur in all zones, and traffic is routed between zones transparently as part of Azure networking operations.

- **Data replication between zones:** Azure DDoS Protection doesn't replicate customer data between zones because the service is stateless and doesn't maintain customer data.

### Behavior during a zone failure

This section describes what to expect when you use Azure DDoS Protection in a region with availability zones, and there's an outage in one of the availability zones.

- **Detection and response:** Microsoft detects availability zone failures and manages all response actions. You don't need to take any action to initiate a zone failover.

[!INCLUDE [Availability zone down notification (Service Health only)](./includes/reliability-availability-zone-down-notification-service-include.md)]

- **Active requests:** Active traffic continues to be processed automatically with no customer action required.

- **Expected data loss:** No data loss is expected because the service is stateless and doesn't store customer data.

- **Expected downtime:** No downtime is expected to the DDoS Protection, and it continues operating using the remaining healthy zones.


### Zone recovery

When a failed availability zone recovers, Azure DDoS Protection automatically restores normal operations without customer intervention.

### Test for zone failures

Azure DDoS Protection is a fully Microsoft-managed, zone-redundant service. Because Microsoft manages zone redundancy, you don't need to test availability zone failover scenarios. 

## Resilience to region-wide failures

Azure DDoS Protection plans are deployed into a single Azure region. However, the protection applies at the platform level and protects public IP addresses regardless of which region those IP addresses are in.


### Requirements

- **Region support:** DDoS Protection plans can protect resources across regions, independent of the region where the plan itself is deployed. 

### Considerations

- If a region hosting a protected resource becomes unavailable, that resource is unavailable regardless of DDoS Protection.
- DDoS Protection continues to operate for protected resources in other regions. 

### Behavior when all regions are healthy


## Service-level agreement

[!INCLUDE [Service-level agreement](includes/reliability-service-level-agreement-include.md)]

Azure DDoS Protection provides an SLA that covers the availability of the DDoS mitigation service to protect against an attack.

## Related content

- [Azure DDoS Protection overview](/azure/ddos-protection/ddos-protection-overview)
- [Reliability in Azure](/azure/reliability/overview)
