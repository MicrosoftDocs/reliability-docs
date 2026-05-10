---
title: Reliability in Azure Traffic Manager
description: Learn how to make Azure Traffic Manager resilient to a variety of potential outages and problems, including transient faults and region-wide failures, and learn about the service-level agreement.
author: glynnniall
ms.author: pnp
ms.topic: reliability-article
ms.custom: subject-reliability
ms.service: azure-traffic-manager
ms.date: 05/05/2026
---

# Reliability in Azure Traffic Manager

[Azure Traffic Manager](/azure/traffic-manager/traffic-manager-overview) is a DNS-based traffic load balancer that distributes traffic optimally across globaly distributed backends. Traffic Manager provides high availability and quick responsiveness for your public-facing applications by using DNS to direct client requests to appropriate service endpoints based on traffic-routing methods and endpoint health monitoring.

[!INCLUDE [Shared responsibility](includes/reliability-shared-responsibility-include.md)]

This article describes the reliability capabilities of Azure Traffic Manager in response to a range of potential outages, including transient faults and region-wide failures. It also highlights key considerations for maintaining resilience and preparing for recovery, and provides an overview of the Azure Traffic Manager service-level agreement (SLA).

> [!NOTE]
> This article describes how the Traffic Manager service is resilient, or how you can make it resilient, to various problems. It doesn't explain how to use Traffic Manager to perform failover between applications or regions. For an example failover architecture, see [Multitier web application built for high availability and disaster recovery](/azure/architecture/example-scenario/infrastructure/multi-tier-app-disaster-recovery).

## Production deployment recommendations

The Azure Well-Architected Framework provides recommendations for reliability, performance, security, cost, and operations. To learn how these areas influence each other and contribute to a reliable Traffic Manager solution, see [Architecture best practices for Azure Traffic Manager in the Well-Architected Framework](/azure/well-architected/service-guides/azure-traffic-manager).

## Reliability architecture overview

[!INCLUDE [Introduction to reliability architecture overview section](includes/reliability-architecture-overview-introduction-include.md)]

### Logical architecture

When you use Traffic Manager, you deploy a *profile*, which specifies your application's back-end endpoints and configures how Traffic Manager should route requests to those endpoints. For more information, see [Traffic Manager endpoints](/azure/traffic-manager/traffic-manager-endpoint-types) and [Traffic Manager routing methods](/azure/traffic-manager/traffic-manager-routing-methods).

A Traffic Manager profile presents as a DNS CNAME record. When it receives a resolution request from a client or DNS resolver, Traffic Manager dynamically resolves the IP address based on rules you specify in the profile. Traffic Manager's responsibility is to provide clients with the IP address of an endpoint to reach your service. After name resolution, none of your application's traffic flows through Traffic Manager. For more information, see [How Traffic Manager Works](/azure/traffic-manager/traffic-manager-how-it-works).

Traffic Manager monitors the health of your endpoints, and routes incoming requests to healthy endpoints while avoiding unhealthy endpoints. For more information, see [Traffic Manager endpoint monitoring](/azure/traffic-manager/traffic-manager-monitoring).

> [!IMPORTANT]
> The reliability of your overall solution depends on the configuration of the endpoints that your traffic manager routes traffic to.
>
> This article doesn't cover your endpoints, but their availability configurations directly affect your application's resilience. Review the [reliability guides for Azure services in your solution](./overview-reliability-guidance.md) to learn how each service supports your reliability requirements.

### Physical architecture

Traffic Manager operates as a nonregional service and deploys its infrastructure across multiple availability zones in multiple Azure regions worldwide. This design enables Traffic Manager to remain resilient during an availability zone or region outage, because infrastructure in another zone or region continues to respond to resolution requests.

Global internet protocols like Anycast, DNS, and BGP automatically route incoming DNS resolution requests to the nearest healthy Traffic Manager infrastructure.

## Resilience to transient faults

[!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)]

Traffic Manager operates at the DNS level and uses health probes to monitor endpoint availability. The service handles transient faults through its global DNS infrastructure and endpoint monitoring capabilities.

When you use Traffic Manager, consider the following types of transient faults separately:

- **Transient faults during DNS resolution:** If a transient fault occurs during DNS resolution, the client or intermediate resolver should retry.

- **Transient faults affecting your back-end endpoints:** [Traffic Manager endpoint monitoring](/azure/traffic-manager/traffic-manager-monitoring) checks the health of your endpoints regularly. A transient fault within an endpoint, or in the network path to an endpoint, might be detected as an unhealthy endpoint. Configure endpoint monitoring to look for consecutive problems over a period of time.

Your DNS CNAME record’s time to live (TTL) determines how your solution handles faults. If the TTL is very low, clients need to make more requests to Traffic Manager and there are more potential opportunities for transient faults to arise. If the TTL is very high, in the event of a true fault in an endpoint, clients might experience delays in failover until the TTL expires. Configure TTLs carefully to balance availability, latency, and responsiveness. For more information, see [Performance considerations for Traffic Manager](/azure/traffic-manager/traffic-manager-performance-considerations).

## Resilience to availability zone failures

[!INCLUDE [Resilience to availability zone failures](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)]

Traffic Manager operates as a nonregional service and deploys its infrastructure across multiple availability zones in multiple Azure regions worldwide. It replicates changes to your profile synchronously across these zones and regions. This design enables Traffic Manager to remain resilient during an availability zone outage, because infrastructure in another zone or region continues to respond to resolution requests.

> [!WARNING]
> **Note to PG:** Can you please confirm that configuration changes are replicated synchronously?

> [!WARNING]
> **Note to PG:** Could a zone/region outage cause Traffic View data and/or RUM data to be lost, or at least temporarily unavailable?

## Resilience to region-wide failures

Traffic Manager operates as a nonregional service and deploys its infrastructure across multiple availability zones in multiple Azure regions worldwide. This design enables Traffic Manager to remain resilient during a region outage, because infrastructure in another zone or region continues to respond to resolution requests.

## Resilience to portal and management tool outages

If you manage your Traffic Manager profile in the Azure portal, prepare for scenarios where you can’t access it, especially if you need to reconfigure your profile during a platform outage.

Like other Azure services, Traffic Manager supports deployment and management through a variety of tools. We recommend you familiarize yourself with how to use [Azure CLI](/azure/traffic-manager/quickstart-create-traffic-manager-profile-cli) or [Azure PowerShell](/azure/traffic-manager/quickstart-create-traffic-manager-profile-powershell) to manage your profile. Alternatively, deploy and configure your profile by using infrastructure as code technologies like [Bicep](/azure/traffic-manager/quickstart-create-traffic-manager-profile-bicep) or [Terraform](/azure/traffic-manager/quickstart-create-traffic-manager-profile-terraform). These tools remain operational even if the Azure portal is degraded.

## Backup and restore

Traffic Manager is a stateless DNS service. It doesn't persist your data and has no backup or restore capability.

To protect your resource configuration, define your Traffic Manager profiles and other resources using infrastructure as code (such as Bicep or ARM templates) and store those definitions in source control. If you need to recreate a resource, redeploy it from the stored configuration.

## Resilience to service maintenance

[!INCLUDE [Service maintenance (no special callouts)](includes/reliability-maintenance-include.md)]

## Service-level agreement

[!INCLUDE [Service-level agreement](includes/reliability-service-level-agreement-include.md)]

Azure Traffic Manager provides a 100% availability SLA for DNS query responses, as long as clients retry failed requests repeatedly.

## Related content

- [Azure Traffic Manager overview](/azure/traffic-manager/traffic-manager-overview)
- [Azure reliability documentation](./overview.md)
