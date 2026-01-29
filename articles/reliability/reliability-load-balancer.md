---
title: Reliability in Azure Load Balancer
description: Learn how to make Azure Load Balancer resilient to a variety of potential outages and problems, including transient faults, availability zone outages, and region outages.
author: glynnniall
ms.author: glynnniall
ms.topic: reliability-article
ms.custom: subject-reliability
ms.service: azure-load-balancer
ms.date: 01/08/2026
ai-usage: ai-assisted

---

# Reliability in Azure Load Balancer

Azure Load Balancer is a layer-4 load-balancing service for Transmission Control Protocol (TCP) and User Datagram Protocol (UDP) traffic that distributes incoming requests among healthy instances of your services. Load Balancer provides high availability and ultra-low-latency network performance.

[!INCLUDE [Shared responsibility description](includes/reliability-shared-responsibility-include.md)]

This article describes how to make Load Balancer resilient to a variety of potential outages and problems, including transient faults, availability zone outages, and region outages. It also highlights key information about the Load Balancer service-level agreement (SLA).

> [!IMPORTANT]
> The reliability of your overall solution depends on the configuration of the back-end instances (servers) like Azure virtual machines (VMs) or virtual machine scale sets that your load balancer routes traffic to.
>
> This article doesn't cover your back-end instances, but their availability configurations directly affect your application's resilience. Review the [reliability guides for Azure services in your solution](./overview-reliability-guidance.md) to learn how each service supports your reliability requirements. When you configure your back-end instances for high availability and zone redundancy, you can achieve complete reliability for your application.

## Production deployment recommendations

The Azure Well-Architected Framework provides recommendations across reliability, security, cost, operations, and performance. To learn how these areas influence each other and contribute to a reliable Load Balancer solution, see [Architecture best practices for Load Balancer](/azure/well-architected/service-guides/azure-load-balancer).

## Reliability architecture overview

A load balancer can be either public or internal. A public load balancer is reachable from the internet through a public IP address resource. An internal load balancer is reachable only from within your virtual network and other networks that you connect to the virtual network.

Each load balancer consists of multiple components:

- *Front-end IP configurations* that receive traffic. A public load balancer receives traffic on a public IP address. An internal load balancer receives traffic on an IP address within your virtual network.

- *Back-end pools* that contain a collection of *back-end instances* that can receive traffic, like individual VMs that run your application.

- *Load-balancing rules* that define how the load balancer distributes traffic from a front end to a back-end pool.

- *Health probes* that monitor the availability of back-end instances.

For more information, see [Load Balancer components](/azure/load-balancer/components).

For globally deployed solutions, you can deploy a *global load balancer*, which is a unique type of public load balancer for routing traffic among different regional deployments of your solution. A global load balancer provides a single anycast IP address and routes traffic to the closest healthy regional load balancer based on client proximity and regional health status. For more information, see [Resilience to region-wide failures](#resilience-to-region-wide-failures).

## Resilience to transient faults

[!INCLUDE [Transient fault description](includes/reliability-transient-fault-description-include.md)]

When you use Load Balancer, consider the following best practices to minimize the risk of transient faults affecting your application:

- **Implement retry logic.** Clients should implement appropriate retry mechanisms for transient connection failures, which include exponential backoff strategies.

- **Configure health probes with tolerance.** Configure your health probes to balance rapid failure detection with the need to avoid false positives during transient problems.

- **Monitor SNAT port allocation.** For outbound connections, monitor Source Network Address Translation (SNAT) port allocation and configure outbound rules to prevent transient connection failures that occur because of port exhaustion.

## Resilience to availability zone failures

[!INCLUDE [Resilience to availability zone failures](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)]

You can deploy Load Balancer as *zone redundant* by configuring each front-end IP configuration that you create. A zone-redundant front-end IP configuration uses independent infrastructure in multiple zones to serve traffic simultaneously. This configuration ensures that zone failures don't affect the load balancer's ability to receive and distribute traffic.

The following diagram shows a zone-redundant public load balancer, which you configure by creating a zone-redundant public IP address.

:::image type="complex" border="false" source="./media/reliability-load-balancer/zone-redundant-public-load-balancer.svg" alt-text="Diagram that shows a zone-redundant public load balancer with a zone-redundant public IP address that directs traffic to three VMs in different availability zones." lightbox="./media/reliability-load-balancer/zone-redundant-public-load-balancer.svg":::
   Architecture diagram that shows a zone-redundant public load balancer deployed across three availability zones. A zone-redundant public IP address connects to the zone-redundant front-end IP configuration. The public load balancer section includes a zone-redundant front-end IP configuration, a load-balancing rule, and a back-end pool. The back-end pool connects to three VMs, one VM in each availability zone.
:::image-end:::

The following diagram shows an internal load balancer that uses a similar zone-redundant configuration.

:::image type="complex" border="false" source="./media/reliability-load-balancer/zone-redundant-internal-load-balancer.svg" alt-text="Diagram that shows a zone-redundant internal load balancer with a zone-redundant public IP address that directs traffic to three VMs in different availability zones." lightbox="./media/reliability-load-balancer/zone-redundant-internal-load-balancer.svg":::
   Architecture diagram that shows a zone-redundant internal load balancer deployed across three availability zones. The load balancer includes a zone-redundant front-end IP configuration, a load balancing rule, and a back-end pool that spans all three zones. Traffic routes from the internal front-end IP configuration through the load balancer to the VMs across all zones.
:::image-end:::

> [!NOTE]
> You can deploy zonal load balancers, but we recommend that you use zone-redundant load balancers for all workloads, including workloads that you deploy into a single zone. Microsoft is migrating all public IP addresses and load balancers to zone-redundant configurations.

In regions without availability zones, Azure creates all load balancers in a *nonzonal* or *regional* configuration by using a front-end configuration with no zone configured. If the region experiences an outage, your nonzonal load balancers might experience downtime.

#### Back-end instances and availability zones

The availability zone configuration of your back-end instances operates independently of your load balancer's front-end IP configuration.

To distribute your back-end instances across zones, configure the relevant service according to its reliability features and your architecture requirements.

> [!NOTE]
> To achieve resilience, distribute back-end instances across multiple availability zones. If all back-end instances run in a single zone, an outage in that zone makes your application unavailable, even if you use a zone-redundant load balancer.

For example, when you use VMs, a common design approach for production workloads is to place multiple zonal VMs in zones 1, 2, and 3 to achieve zone resiliency. For load balancing, you can then create a zone-redundant load balancer and configure those VMs as the back-end instances within the load balancer. The load balancer's health probes automatically remove unhealthy VMs from rotation regardless of their zone location.

If you deploy your VMs in the same availability zone, you can deploy a zone‑redundant front-end IP configuration on your load balancer, as the following diagram shows.

:::image type="complex" border="false" source="./media/reliability-load-balancer/zone-redundant-load-balancer-zonal-virtual-machines.svg" alt-text="Diagram that shows a zone-redundant public load balancer directing traffic to two different VMs in zone 1." lightbox="./media/reliability-load-balancer/zone-redundant-load-balancer-zonal-virtual-machines.svg":::
   Architecture diagram that shows a zone-redundant public load balancer deployed across three availability zones, but with both VMs located only in zone 1. A zone-redundant public IP address connects to a zone-redundant front-end IP configuration. The load balancer includes a zone-redundant front-end IP configuration, load-balancing rule, and a back-end pool that spans all three zones. The back-end pool contains two VMs that are both deployed in zone 1 only, and zone 2 and zone 3 have no VMs. This configuration creates a single point of failure (SPoF) because all VMs reside in one zone.
:::image-end:::

### Requirements

**Region support:** You can deploy zone-redundant load balancers into [any region that supports availability zones](regions-list.md).

### Cost

Availability zone configuration doesn't change the way Azure bills a load balancer. Azure bases billing on the number of rules that you configure and the data that it processes, regardless of zone configuration. For more information, see [Load Balancer pricing](https://azure.microsoft.com/pricing/details/load-balancer/).

### Configure availability zone support

When you use Load Balancer, you set the availability zone support on the front-end IP configuration.

- **Create a new load balancer with availability zone support.**

    - For *public load balancers*, the front-end IP configuration automatically adopts the availability zone configuration of the public IP address resource that you associate with it. To make the front-end IP configuration zone redundant, create or choose a zone-redundant public IP address. Azure configures public IP addresses as zone redundant by default. For more information, see [Create a public load balancer to load balance VMs by using the Azure portal](/azure/load-balancer/quickstart-load-balancer-standard-public-portal).

    - For *internal load balancers*, when you configure the front-end IP address of the load balancer, you set the availability zone support type on the front-end IP configuration. For more information, see [Create an internal load balancer to load balance VMs by using the Azure portal](/azure/load-balancer/quickstart-load-balancer-standard-internal-portal).

- **Change the availability zone configuration of an existing load balancer.** To change the availability zone configuration of an existing load balancer, replace the front-end IP configuration. You can use this approach to move from a zonal configuration to a zone-redundant front-end IP configuration:

    1. Create a new front-end IP configuration that has the desired availability zone configuration.
    
        For *public load balancers*, create a new public IP address that uses your desired availability zone configuration. Then reconfigure your load balancer to add a front-end IP configuration that references that public IP address.

        For *internal load balancers*, reconfigure your load balancer to add a new front-end IP configuration with your desired availability configuration. This step assigns a new private IP address from within your subnet.
    
    1. Reconfigure your load-balancing rules to use the new front-end IP configuration.
    
        > [!IMPORTANT]
        > This operation requires you to reconfigure your clients to send traffic to the new front-end IP address. Depending on your clients, the process might require downtime.
    
    1. Remove the old front-end IP configuration.

### Behavior when all zones are healthy

This section describes what to expect when a load balancer uses a zone-redundant front-end IP configuration and all availability zones operate normally.

- **Traffic routing between zones:** Load balancing can occur in any availability zone. The load balancer sends traffic to healthy back-end instances that you specify in the back-end pool, without regard for which availability zone contains the back-end instance.

- **Data replication between zones:** Load Balancer operates as a network pass-through service that doesn't store or replicate application data. Even if you set up [session persistence](/azure/load-balancer/distribution-mode-concepts#session-persistence) on the load balancer, the load balancer stores no state. Session persistence adjusts the hashing process to route requests to the same back-end instance, but the load balancer doesn't guarantee session persistence. When the back-end pool changes, the load balancer recomputes the distribution of client requests and doesn't store or sync state.

    The service maintains its configuration state through synchronous replication across zones, which ensures immediate consistency of load-balancing rules, health probe configurations, and back-end pool membership across all zones.

### Behavior during a zone failure

This section describes what to expect when a load balancer uses a zone-redundant front-end IP configuration and an availability zone outage occurs.

- **Detection and response:** The Azure platform is responsible for detecting a failure in an availability zone and responding. You don't need to do anything to initiate a zone failover.

[!INCLUDE [Availability zone down notification (Service Health and Resource Health)](./includes/reliability-availability-zone-down-notification-service-resource-include.md)]

- **Active requests:** When a zone fails, any existing TCP and UDP flows within the zone are automatically reset, and your clients must retry those flows. Your clients should implement sufficient [transient fault handling](#resilience-to-transient-faults), including automated retries.

- **Expected data loss:** Load Balancer is a stateless network service, so it doesn't store application data and no data loss occurs at the load balancer layer.

- **Expected downtime:** No downtime is expected for zone-redundant load balancers because the load balancer continues to work from healthy zones.

    If the failure affects compute services in the zone, any VMs or other resources in the affected zone might be unavailable. The load balancer's health probes detect these failures and route traffic to alternative instances in another zone based on the load-balancing algorithm and the back-end instances' health status.

- **Traffic rerouting:** The load balancer continues to operate from the healthy zones. Load Balancer maintains the same front-end IP address during zone failures. This behavior means that you don't need to apply Domain Name System (DNS) updates or reconfigure clients. The Azure platform establishes new client connections through remaining healthy zones.

### Zone recovery

When an availability zone recovers, Load Balancer automatically resumes normal operations. The zone-redundant front end automatically begins to serve traffic from the recovered zone along with other operational zones. Health probes from the recovered zone resume evaluating the back-end instances.

If the zone failure also affects your compute services in that zone, Load Balancer automatically adds back-end instances to the healthy back-end pool after they recover and pass health checks. Traffic distribution rebalances across all available zones based on the load-balancing algorithm and the back-end instances' health status.

### Test for zone failures

The Azure platform manages traffic routing, zone-down response, and recovery, so you don't need to initiate or validate availability zone failure processes.

You can use Azure Chaos Studio to simulate the failure of a VM in a single zone. Chaos Studio provides [built-in faults for VMs](/azure/chaos-studio/chaos-studio-fault-library#virtual-machines-service-direct), including a fault that shuts down a VM. You can use these capabilities to simulate zone failures and test your failover processes.

## Resilience to region-wide failures

Public and internal load balancers are deployed into a single Azure region. If the region becomes unavailable, your load balancers in that region also become unavailable. Load Balancer provides native multi-region support through a *global load balancer*, which supports load balancing across Azure regions. You can also deploy other load balancing services to route and fail over across Azure regions

### Global load balancers

Global load balancer provides a single static anycast IP address that automatically routes traffic to the optimal regional deployment based on client proximity and regional health. Global load balancer improves your application's reliability and performance.

With global load balancer, you deploy multiple public load balancers in different regions, and the global load balancer serves as a global front end. If your back-end servers in one region have a problem, traffic automatically switches to healthy regions without DNS changes because the anycast IP address remains constant and routes traffic to another region.

For more information, see [Global load balancer](/azure/load-balancer/cross-region-overview).

### Custom multi-region solutions for resiliency

Azure provides a range of load-balancing services that suit different requirements. Choose a load balancer that meets your resiliency requirements and suits your application type. For more information, see [Load-balancing options](/azure/architecture/guide/technology-choices/load-balancing-overview).

## Service-level agreement

[!INCLUDE [SLA description](includes/reliability-service-level-agreement-include.md)]

The Load Balancer SLA applies when you have at least two healthy VMs configured as back-end instances. The SLA excludes downtime because of SNAT port exhaustion or network security groups (NSGs) that block traffic.

### Related content

- [Load Balancer documentation](/azure/load-balancer/load-balancer-overview)
- [Azure reliability overview](overview.md)
