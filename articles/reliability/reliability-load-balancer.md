---
title: Reliability in Azure Load Balancer
description: Learn how to improve reliability in Azure Load Balancer by using availability zones, cross-zone load balancing, cross-region load balancing, and operational best practices.
author: anaharris-ms 
ms.author: anaharris
ms.topic: reliability-article
ms.custom: subject-reliability
ms.service: azure-load-balancer
ms.date: 09/10/2025
ai-usage: ai-assisted

#Customer intent: As an engineer responsible for business continuity, I want to understand the details of how Azure Load Balancer works from a reliability perspective and plan disaster recovery strategies in alignment with the exact processes that Azure services follow during different kinds of situations.
---

# Reliability in Azure Load Balancer

Azure Load Balancer is a layer-4 (TCP/UDP) load balancing service that distributes incoming traffic among healthy instances of services. Load Balancer provides high availability and network performance to your applications with ultra-low latency.

This article describes reliability support in [Azure Load Balancer](../load-balancer/load-balancer-overview.md). It covers intra-regional resiliency via [availability zones](#availability-zone-support) and [cross-region load balancing](#multi-region-support).

[!INCLUDE [Shared responsibility description](includes/reliability-shared-responsibility-include.md)]

> [!IMPORTANT]
> The reliability of your overall solution depends on the configuration of the backend instances (servers) that your load balancer routes traffic to, such as Azure virtual machines (VMs) or Azure virtual machine scale sets.
>
> Your backend instances aren't in scope for this article, but their availability configurations directly affect your application's resilience. Review the [reliability guides for all of the Azure services in your solution](./overview-reliability-guidance.md) to understand how each service supports your reliability requirements. By ensuring that your backend instances are also configured for high availability and zone redundancy, you can achieve end-to-end reliability for your application.

## Production deployment recommendations

To learn about how to deploy Load Balancer to support your solution's reliability requirements, and how reliability affects other aspects of your architecture, see [Architecture best practices for Load Balancer in the Azure Well-Architected Framework](/azure/well-architected/service-guides/azure-load-balancer).

## Reliability architecture overview

A load balancer can be either public or internal. A public load balancer accepts traffic from the internet through a public IP address resource. An internal load balancer is accessible only within your virtual network and other networks that you connect to the virtual network.

Each load balancer consists of multiple components, including:

- *Frontend IP configurations*, which receive traffic. A public load balancer receives traffic on a public IP address. An internal load balancer receives traffic on an IP address within your virtual network.
- *Backend pools*, which contain a collection of *backend instances* that can receive traffic, such as individual VMs that run your application.
- *Load balancing rules*, which define how traffic from a frontend should be distributed to a backend pool.
- *Health probes*, which monitor the availability of backend instances.

To learn more about how Load Balancer works, see [Load Balancer components](../load-balancer/components.md).

For globally deployed solutions, you can deploy a *global load balancer*, which is a special type of public load balancer designed to route traffic among different regional deployments of your solution. A global load balancer provides a single anycast IP address. It routes traffic to the closest healthy regional load balancer based on client proximity and regional health status. For more information, see [Multi-region support](#multi-region-support).

## Transient faults

[!INCLUDE [Transient fault description](includes/reliability-transient-fault-description-include.md)]

When you use Load Balancer, consider the following best practices to minimize the risk of transient faults affecting your application:

- **Implement retry logic.** Clients should implement appropriate retry mechanisms for transient connection failures, including exponential backoff strategies.

- **Configure health probes with tolerance.** Configure your health probes to balance between rapid failure detection and avoiding false positives during transient issues.

- **Monitor SNAT port allocation.** For outbound connections, monitor SNAT port allocation and configure outbound rules to prevent transient connection failures due to port exhaustion.

## Availability zone support

[!INCLUDE [Availability zone support description](includes/reliability-availability-zone-description-include.md)]

Load Balancer provides two types of availability zone support: zonal and zone-redundant. To configure for either configuration, you set the availability zone configuration on each frontend IP configuration that you create.

- *Zone-redundant:* A zone-redundant frontend IP configuration is served simultaneously from independent infrastructure in multiple zones. This configuration ensures that zone failures don't impact the load balancer's ability to receive and distribute traffic.

    The following diagram shows a zone-redundant public load balancer, which is configured by creating a zone-redundant public IP address:
    
    :::image type="content" source="./media/reliability-load-balancer/zone-redundant-public-load-balancer.png" alt-text="Diagram showing a zone-redundant public load balancer, with a zone-redundant public IP address, directing traffic to three different VMs in different availability zones." border="false" :::

    The following diagram shows an internal load balancer using a similar zone-redundant configuration:

    :::image type="content" source="./media/reliability-load-balancer/zone-redundant-internal-load-balancer.png" alt-text="Diagram showing a zone-redundant internal load balancer, directing traffic to three different VMs in different availability zones." border="false" :::

- *Zonal:* A zonal frontend IP configuration is deployed into a single availability zone. The inbound and outbound flows are served within that one zone. If the availability zone has a problem, the load balancer is unavailable. You should only use a zonal frontend IP configuration if you deploy your backend instances within the same zone *and* you need all your traffic to remain within the zone, which is uncommon.

    > [!IMPORTANT]
    > Pinning to a single availability zone is only recommended when [cross-zone latency](./availability-zones-overview.md#inter-zone-latency) is too high for your needs and when you verify that the latency doesn't meet your requirements. By itself, a zonal load balancer doesn't provide resiliency to an availability zone outage. To improve the resiliency of a zonal load balancer, you need to explicitly deploy separate load balancers and backend instances into multiple availability zones and configure traffic routing and failover.

    The following diagram shows a zonal internal load balancer in availability zone 1, which is configured by creating a zonal public IP address in that zone:
    
    :::image type="content" source="./media/reliability-load-balancer/zonal-public-load-balancer.png" alt-text="Diagram showing a zonal public load balancer in zone 1, with a zonal public IP address, directing traffic to three different VMs in zone 1." border="false" :::

    The following diagram shows an internal load balancer using a similar zone-redundant configuration:

    :::image type="content" source="./media/reliability-load-balancer/zonal-internal-load-balancer.png" alt-text="Diagram showing a zonal internal load balancer in zone 1, directing traffic to three different VMs in zone 1." border="false" :::

If you don't configure a load balancer to be zone-redundant or zonal, it's considered *nonzonal* or *regional*. Nonzonal load balancers can be placed in any availability zone within the region. If an availability zone in the region experiences an outage, nonzonal load balancers might be in the affected zone and could experience downtime.
<!-- PG: When will nonzonal public IP addresses be retired (per https://azure.microsoft.com/en-us/blog/azure-public-ips-are-now-zone-redundant-by-default/)? I can still create one as of September 2025. -->

> [!TIP]
> Use zone-redundant frontend IP configurations wherever possible.
>
> Only use zonal frontend IP configurations when you require a load balancer to exist within a single zone. Avoid using nonzonal frontend IP configurations, because they don't provide resiliency to zone failures.

#### Backend instances and availability zones

Regardless of the load balancer's frontend IP configuration, your backend instances can be distributed across zones. Health probes automatically removing unhealthy backend instances from rotation regardless of their zone location.

When you use virtual machines, a common design approach for production workloads is to have multiple zonal VMs in different zones. For example, you might deploy VMs into zones 1, 2, and 3. You can create a zone-redundant load balancer and configure those VMs as the backend instances within the load balancer.

> [!NOTE]
> Distributing backend instances across multiple availability zones is essential for resilience. If all backend instances are located in a single zone, an outage in that zone will make your application unavailable, even if you use a zone-redundant load balancer.

#### Multiple frontends on a single load balancer

Using multiple frontends allow you to load balance traffic on more than one port or IP address.

Because each load balancer can have multiple frontend IP configurations, it's possible to have one load balancer that includes a combination of zone-redundant, zonal, and nonzonal frontend IP configurations. However, in practice, you typically only have one type of availability zone configuration for your load balancer. <!-- PG: please verify that this is indeed the most common (and recommended) configuration. -->

When designing your architecture:

- If your goal is to always have every frontend IP configuration resilient to failure, then we recommend that you configure all of your frontend IP addresses to be zone-redundant. 

- If you watn to assiociate a frontend IP address with a single zone, then you must configure that frontend IP to be zonal.

    If you need to use multiple zonal deployments, each in different zones, you don't need to deploy a load balancer in each zone. Instead, you can deploy a single load balancer with a zonal frontend in each zone. Configure the load balancing rules to route traffic from the zonal frontends to the virtual machines in the backend pool that are part of the same availability zone.

### Considerations

- **Zone failure:** Up to one availability zone can fail and the data path survives as long as the remaining zones in the region remain healthy. <!-- PG: please verify that two simultaneous zone failures are not handled. --> 

- **Region upgrades:** If a region is upgraded to include availability zones, any existing IPs and load balancers remain as nonzonal. You need to explicitly migrate them to be zone-redundant or zonal. <!-- PG: please verify this is accurate. -->

### Region support

Zone-redundant and zonal load balancers can be deployed into [any region that supports availability zones](regions-list.md).

### Cost

Availability zone configuration doesn't change the way a load balancer is billed. Billing is based on the number of rules configured and data processed, regardless of zone configuration. For more information, see [Azure Load Balancer pricing](https://azure.microsoft.com/pricing/details/load-balancer/)

### Configure availability zone support

When you work with Load Balancer, you set the availability zone on the frontend IP configuration.

- **Create a new load balancer with availability zone support.**

    - For *public load balancers*, the frontend IP configuration automatically adopts the availability zone configuration of the public IP address resource that you associate with it.
        
        Create or select a zone-redundant public IP address to make the frontend IP configuration zone-redundant. Alternatively, create or select a zonal public IP address to make the frontend IP configuration zonal. Azure places the load balancer in the same zone.
    
        For detailed steps, see [Create a public load balancer to load balance VMs using the Azure portal](../load-balancer/quickstart-load-balancer-standard-public-portal.md).

    - For *internal load balancer*s, when you configure the frontend IP of the load balancer, you set the availability zone support type on the frontend IP configuration.
        
        For detailed steps, see [Create an internal load balancer to load balance VMs using the Azure portal](../load-balancer/quickstart-load-balancer-standard-internal-portal.md).

- **Change the availability zone configuration of an existing load balancer.** To change the availability zone configuration of an existing load balancer, you need to replace the frontend IP configuration. The high-level approach is:

    <!-- Anastasia: I know this is veering into how-to territory. I'll leave this for you to decide whether we keep it (since it's pretty high level), or if we should move this into a new migration how-to within the product docs. -->

    1. Create a new frontend IP configuration with the desired availability zone configuration.
    
        For *public load balancers*, create a new public IP address that uses your desired availability zone configuration first. Then, reconfigure your load balancer to add a frontend IP configuration that references that public IP address.

        For *internal load balancers*, reconfigure your load balancer to add a new frontend IP configuration with your desired availability configuration. This step assigns a new private IP address from within your subnet.
    
    1. Reconfigure your load balancing rules to use the new frontend IP configuration.
    
        > [!IMPORTANT]
        > This operation requires you to reconfigure your clients to send traffic to the new frontend IP address. Depending on your clients, the process might require downtime.
    
    1. Remove the old frontend IP configuration.

           You can use this approach to move from a nonzonal to a zone-redundant frontend IP configuration, or between other availability zone support types.

### Normal operations

This section describes what to expect when a load balancer's frontend IP configuration is configured with availability zone support and all availability zones are operational.

- **Traffic routing between zones:** Traffic routing behavior depends on the availability zone configuration that your frontend IP configuration uses.

    - *Zone-redundant:* Load balancing can be performed in any availability zone.

    - *Zonal:* Load balancing is performed within the selected availability zone.
    
    For both zone-redundant and zonal frontend IP configurations, traffic is then sent to healthy backend instances specified in the backend pool, without consideration of which availability zone the backend instance is in.

- **Data replication between zones**. Load Balancer is a network pass-through service that doesn't store or replicate application data. Even if you enable [session persistence](../load-balancer/distribution-mode-concepts.md#session-persistence) on the load balancer, no state is stored on the load balancer. Session persistence adjusts the hashing process to route requests to the same backend instance. However, session persistence isn't guaranteed. When the backend pool changes, the distribution of client requests is recomputed. This process is done without storing or synchronizing state.

    The service maintains its configuration state with synchronous replication across zones, ensuring immediate consistency of load balancing rules, health probe configurations, and backend pool membership across all zones. <!-- PG: Please verify. -->

### Zone-down experience

This section describes what to expect when a load balancer's frontend IP configuration is configured with availability zone support and there's an availability zone outage.

- **Detection and response**: Responsibility for detection and response depends on the availability zone configuration that your frontend IP configuration uses.

    - *Zone-redundant:* The Azure platform is responsible for detecting a failure in an availability zone and responding. You don't need to do anything to initiate a zone failover.

    - *Zonal:* You need to detect the loss of an availability zone and for responding.
    
        You might choose to initiate a failover to a secondary frontend IP configuration, another load balancer, or other infrastructure that you create in another availability zone or region. You're responsible for any of these failover activities.

- **Notification**: Azure Load Balancer doesn't notify you when a zone is down. However, you can use [Azure Resource Health](/azure/service-health/resource-health-overview) to monitor for the health of your load balancer. You can also use [Azure Service Health](/azure/service-health/overview) to understand the overall health of the Azure Load Balancer service, including any zone failures.
    
    Set up alerts on these services to receive notifications of zone-level problems. For more information, see [Create Service Health alerts in the Azure portal](/azure/service-health/alerts-activity-log-service-notifications-portal) and [Create and configure Resource Health alerts](/azure/service-health/resource-health-alert-arm-template-guide).

- **Active requests**: Any existing TCP/UDP flows within the failed zone are reset and need to be retried by the client. Your clients should have sufficient [transient fault handling](#transient-faults) implemented, including automated retries.

- **Expected data loss**: As a stateless network service, Load Balancer doesn't store application data, so no data loss occurs at the load balancer layer.

- **Expected downtime**: The expected downtime depends on the availability zone configuration that your load balancer's frontend IP configuration uses.

    - *Zone-redundant:* There's no downtime is expected for the load balancer service.

        However, if the failure affects compute services in the zone, then any VMs or other resources that are in the affected zone could be unavailable. The load balancer's health probes are designed to detect these failures and route traffic to alternative instances in another zone based on the load balancing algorithm and backend instances' health status.

    - *Zonal:* When a zone is unavailable, your load balancer's frontend IP is unavailable until the availability zone recovers.

- **Traffic rerouting**: The traffic rerouting behavior depends on the availability zone configuration that your load balancer's frontend IP configuration uses. 

    - *Zone-redundant:* The load balancer continues to operate from the surviving zones. The Load Balancer service maintains the same frontend IP address during zone failures, eliminating the need for DNS updates or client reconfiguration. New connections are automatically established through remaining healthy zones.

    - *Zonal:* When a zone is unavailable, your load balancer's frontend IP is unavailable. If you have secondary infrastructure in another availability zone, you're responsible for rerouting client traffic to that secondary infrastructure.

### Zone recovery

When an availability zone recovers, Load Balancer automatically resumes normal operations. The zone recovery behavior depends on the availability zone configuration that your frontend IP configuration uses.

- *Zone-redundant:* The zone-redundant frontend automatically begins serving traffic from the recovered zone alongside other operational zones. Health probes from the recovered zone resume evaluating backend instances.

- *Zonal:* If you've switched clients to use alternative infrastructure during the zone outage, you're responsible for reconfiguring those clients to send traffic to the load balancer's frontend IP address.

If the zone failure also affected your compute services in the zone, then as backend instances in the recovered zone pass health checks, they're automatically added back to the healthy backend pool. Traffic distribution rebalances across all available zones based on the load balancing algorithm and backend instances' health status.

### Testing for zone failures

The options for testing for zone failures depend on the availability zone configuration that your frontend IP configuration uses.

- **Zone-redundant:** The Azure platform manages traffic routing, zone-down response, and recovery. These capabilities are fully managed, so you don't need to initiate or validate availability zone failure processes.

- **Zonal:** There's no way to simulate a full outage of the availability zone that contains your load balancer's frontend IP configuration.

    However, you can use Azure Chaos Studio to simulate the failure of a virtual machine in a failed zone. Chaos Studio provides [built-in faults for VMs](/azure/chaos-studio/chaos-studio-fault-library#virtual-machines-service-direct), including to shut down a VM. You can use these capabilities to simulate zone failures and test your failover processes.
    
    You can also test your recovery processes by reconfiguring clients or upstream services to simulate the load balancer being unavailable. 

## Multi-region support

Azure Load Balancer provides native multi-region support through [Global Load Balancer](../load-balancer/cross-region-overview.md), which enables load balancing across Azure regions. Global Load Balancer provides a single static anycast IP address that automatically routes traffic to the optimal regional deployment based on client proximity and regional health. Global Load Balancer can help to improve your application's reliability and performance.

With Global Load Balancer, you deploy multiple public load balancers in different regions, and the global load balancer acts as a global frontend. If a region has a problem, traffic fails over to healthy regions automatically and without DNS changes because the anycast IP address remains constant and routes traffic to another region.

The following diagram shows a global load balancer that routes traffic among three regional load balancers in different Azure regions:

:::image type="content" source="../load-balancer/media/cross-region-overview/cross-region-load-balancer.png" alt-text="Diagram of global load balancer." border="false":::

Global Load Balancer operates at layer 4 and doesn't provide application-layer features like SSL/TLS termination, cookie-based session affinity, or URL path-based routing. For these capabilities, consider using Azure Front Door. You can also consider using Azure Traffic Manager for layer 7 HTTP load balancing across regions.

### Region support

Your regional load balancers can be deployed into any Azure region.

A global load balancer, and its global public IP address, are deployed into a single *home region*. This region doesn't affect how the traffic is routed. If the home region goes down, traffic flow is unaffected. You must [select a home region from a specific list of Azure regions](/azure/load-balancer/cross-region-overview?branch=pr-en-us-304657#home-regions-in-azure).

Azure also has a number of [*participating regions*](/azure/load-balancer/cross-region-overview?branch=pr-en-us-304657#participating-regions-in-azure), which are the locations where the global public IP address of the load balancer is advertised from. You don't need to configure the participating regions for your global load balancer because Azure preselects the participating regions. When a client initiates a request, the traffic is routed to the closest participating region to the client. Then, the traffic travels from the participating region, through the Microsoft core network, until it reaches the closest regional load balancer.


### Cost

Global Load Balancer incurs charges for the global load balancer resource in addition to the regional load balancers. For more information, see [Load Balancer pricing](https://azure.microsoft.com/pricing/details/load-balancer/).

### Configure multi-region support

This section explains how to configure a global load balancer.

- **Create a global load balancer.** Deploy public load balancers in your target regions first, then create a global load balancer that has a backend pool containing these regional load balancers. For detailed steps, see [Create an Azure Global Load Balancer](../load-balancer/tutorial-cross-region-portal.md).

- **Disable global load balancer.** To disable cross-region functionality, remove the global load balancer resource. Regional load balancers continue operating independently, but clients need to be updated to send traffic to the correct regional IP address.

### Normal operations

This section describes what to expect when you use a global load balancer and all regions are operational.

- **Traffic routing between regions**. Global Load Balancer operates in an active/active model. Each client connection is automatically routed to the lowest-latency healthy regional deployment based on Microsoft's global network proximity data and regional health status. The geo-proximity load-balancing algorithm is based on the geographic location of your users and your regional deployments. For more information, see [Global Load Balancer](../load-balancer/cross-region-overview.md#ultra-low-latency).

- **Data replication between regions**. Global Load Balancer is a network pass-through service that doesn't store or replicate application data.

    Configuration changes to your global load balancer are globally replicated with strong consistency. Regional load balancer configurations remain independent and must be managed separately in each region. <!-- PG: Please verify. -->

### Region-down experience

This section describes what to expect when you use a global load balancer and there's a region outage.

- **Detection and response**: Global Load Balancer is responsible for detecting a regional outage and responding.
    
    The health probe of the global load balancer gathers information about availability of each regional load balancer every 5 seconds. A regional load balancer is considered to be unhealthy if either of the following conditions apply:
    - The regional load balancer is unreachable or otherwise unhealthy.
    - The regional load balancer reports that there are no healthy backend instances available.
    <!-- PG: Please verify the above paragraph. -->

    If a regional load balancer is unavailable, the global load balancer detects the failure. The regional load balancer is then taken out of rotation.

    The following diagram shows an example where traffic from a client would ordinarily be routed to their closest region, but due to a failure in that region, it's routed to another regional load balancer instead:

    :::image type="content" source="../load-balancer/media/cross-region-overview/global-region-view.png" alt-text="Diagram of global region traffic view." border="false":::

    For more information, see [Health probes](../load-balancer/cross-region-overview.md#health-probes).

- **Notification**: Global Load Balancer doesn't notify you when a region is down. However, you can use [Azure Service Health](/azure/service-health/overview) to understand the overall health of the Global Load Balancer service, including any region failures.

    Set up alerts to receive notifications of region-level problems. For more information, see [Create Service Health alerts in the Azure portal](/azure/service-health/alerts-activity-log-service-notifications-portal).

    <!-- PG: Please confirm whether Azure Resource Health would also indiate the loss of a regional load balancer. -->

- **Active requests**: Any existing TCP/UDP flows within the failed region are reset and need to be retried by the client. Your clients should have sufficient [transient fault handling](#transient-faults) implemented, including automated retries.

- **Expected data loss**: As a stateless network service, Global Load Balancer doesn't store application data, so no data loss occurs at the load balancer layer.

- **Expected downtime**: No service downtime is expected during a region outage. Clients experience brief connection resets while traffic redistributes to healthy regions.

- **Traffic rerouting**: The anycast IP remains constant and is served by healthy participating regions. When clients reestablish a connection to the same IP address, their traffic is directed to a healthy region. DNS updates aren't required.

### Region recovery

When the region recovers from failure, the global load balancer health probes automatically detect when the regional load balancer becomes healthy again. The global load balancer resumes routing traffic to the regional load balancer. The anycast IP address remains unchanged throughout the failback process, requiring no client or DNS updates.

### Testing for region failures

You can simulate a region failure by temporarily making the virtual machines within the region unavailable. Use this approach to verify that the global load balancer correctly redirects traffic to remaining healthy regions, and to validate your other failover procedures. For more information, see [Test cross-region failover](../load-balancer/tutorial-cross-region-portal.md#test-the-load-balancer)

### Alternative multi-region approaches

For HTTP/HTTPS workloads requiring application-layer features, use [Azure Front Door](../frontdoor/front-door-overview.md) as a global entry point with regional load balancers or Application Gateways as origins. This provides SSL termination, WAF capabilities, and URL-based routing alongside global load balancing.

For DNS-based global load balancing across regions, use [Azure Traffic Manager](../traffic-manager/traffic-manager-overview.md). Traffic Manager provides geographic routing, priority routing, and weighted routing methods, though it requires DNS TTL considerations for failover timing. For an example architecture that combines Azure Traffic Manager and Azure Application Gateway, see [Multi-region load balancing with Traffic Manager and Application Gateway](/azure/architecture/high-availability/reference-architecture-traffic-manager-application-gateway).

## Service-level agreement

[!INCLUDE [SLA description](includes/reliability-service-level-agreement-include.md)]

The Azure Load Balancer SLA applies when there are at least two healthy VMs configured as backend instances. The SLA excludes downtime due to SNAT port exhaustion or NSGs that block traffic.

### Related content

- [Azure Load Balancer documentation](../load-balancer/load-balancer-overview.md)
- [Global Load Balancer](../load-balancer/cross-region-overview.md)
- [Azure reliability overview](/azure/reliability/overview)
 