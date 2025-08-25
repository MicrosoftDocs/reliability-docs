---
title: Reliability in Azure Load Balancer
description: Learn how to improve reliability in Azure Load Balancer by using availability zones, cross-zone load balancing, cross-region load balancing, and operational best practices.
author: anaharris-ms 
ms.author: anaharris
ms.topic: reliability-article
ms.custom: subject-reliability
ms.service: azure-load-balancer
ms.date: 08/25/2025

#Customer intent: As an engineer responsible for business continuity, I want to understand the details of how Azure Load Balancer works from a reliability perspective and plan disaster recovery strategies in alignment with the exact processes that Azure services follow during different kinds of situations.
---

# Reliability in Azure Load Balancer

Azure Load Balancer is a layer-4 (TCP/UDP) load balancing service that distributes incoming traffic among healthy instances of services. Load Balancer provides high availability and network performance to your applications with ultra-low latency. This article describes reliability support in [Azure Load Balancer](../load-balancer/load-balancer-overview.md). It covers intra-regional resiliency via [availability zones](#availability-zone-support) and [cross-region load balancing](#multi-region-support).

[!INCLUDE [Shared responsibility description](includes/reliability-shared-responsibility-include.md)]

## Production deployment recommendations

To learn about how to deploy Load Balancer to support your solution's reliability requirements, and how reliability affects other aspects of your architecture, see [Architecture best practices for Load Balancer in the Azure Well-Architected Framework](/azure/well-architected/service-guides/azure-load-balancer).

## Reliability architecture overview

<!-- John: I realize load balancer supports zonal, but not clear whether you want to mention it here or not.-->
Azure Load Balancer operates with a highly available control plane and data plane architecture within a region. The service consists of frontend IP configurations that receive traffic, backend pools containing your application instances, load balancing rules that define traffic distribution, and health probes that monitor backend availability.

With Standard Load Balancer in availability zone-enabled regions, you can configure zone-redundant frontends where the load balancer's IP address is served simultaneously from independent infrastructure in multiple zones. This configuration ensures that zone failures don't impact the load balancer's ability to receive and distribute traffic. Backend instances can be distributed across zones, with health probes automatically removing unhealthy instances from rotation regardless of their zone location.

For global resilience, Cross-region Load Balancer provides a single anycast IP address that routes traffic to the closest healthy regional load balancer based on client proximity and regional health status.

**Sources:**
- [Load Balancer components](../load-balancer/components.md)
- [Cross-region Load Balancer](../load-balancer/cross-region-overview.md)

## Transient faults

[!INCLUDE [Transient fault description](includes/reliability-transient-fault-description-include.md)]

Azure Load Balancer handles transient faults through its health probe mechanism and automatic traffic rerouting. When managing transient faults with Load Balancer:

- Configure health probe intervals and failure thresholds to balance between rapid failure detection and avoiding false positives during transient issues.
- Implement connection retry logic with exponential backoff in your client applications for idempotent operations.
- Monitor backend health probe status through Azure Monitor to identify patterns of transient failures.
- Use session persistence (client IP affinity) judiciously, as it can impact fault tolerance by binding clients to potentially unhealthy instances.
- For outbound connections, monitor SNAT port allocation and configure outbound rules to prevent transient connection failures due to port exhaustion.

Virtual machines can fail over to another server in a cluster, with the VM's operating system restarting on the new server. You should refer to the failover process for disaster recovery, gathering virtual machines in recovery planning, and running disaster recovery drills to ensure their fault tolerance solution is successful.

For more information, see the [site recovery processes](../site-recovery/site-recovery-failover.md#before-you-start).


**Sources:**
- [Load Balancer health probes](../load-balancer/load-balancer-custom-probe-overview.md)
- [Monitor Load Balancer](../load-balancer/monitor-load-balancer.md)
- [Outbound connections in Azure](../load-balancer/load-balancer-outbound-connections.md)

## Availability zone support

[!INCLUDE [Availability zone support description](includes/reliability-availability-zone-description-include.md)]

Azure Standard Load Balancer supports both zone-redundant and zonal frontend configurations in regions with availability zones. The load balancer's availability zone selection is synonymous with its frontend IP's zone selection. For public load balancers, if the public IP in the load balancer's frontend is zone redundant, then the load balancer is also zone-redundant. If the public IP in the load balancer's frontend is zonal, then the load balancer is designated to the same zone. 

Standard Load Balancer supports cross-zone load balancing, allowing a frontend in any configuration (zone-redundant, zonal, or nonzonal) to reach backend instances in any zone within the region. Health probes originate from all zones and determine the health of backend instances independently, automatically removing unhealthy instances from rotation regardless of their zone.

### Zone redundant load balancer


In a region with availability zones, a Standard Load Balancer can be zone-redundant with traffic served by a single IP address. A single frontend IP address survives zone failure. The frontend IP may be used to reach all (non-impacted) backend pool members no matter the zone. Up to one availability zone can fail and the data path survives as long as the remaining zones in the region remain healthy.

The frontend's IP address is served simultaneously by multiple independent infrastructure deployments in multiple availability zones. Any retries or reestablishment will succeed in other zones not affected by the zone failure.

:::image type="content" source="../load-balancer/media/az-zonal/zone-redundant-lb-1.svg" alt-text="Figure depicts a zone-redundant standard load balancer directing traffic in three different zones to three different subnets in a zone redundant configuration.":::

>[!NOTE]
>VMs 1,2, and 3 can be belong to the same subnet and don't necessarily have to be in separate zones as the diagram suggestions.

Members in the backend pool of a load balancer are normally associated with a single zone such as with zonal virtual machines. A common design for production workloads would be to have multiple zonal resources. For example, placing virtual machines from zone 1, 2, and 3 in the backend of a load balancer with a zone-redundant frontend meets this design principle.

### Zonal load balancer


You can choose to have a frontend guaranteed to a single zone, which is known as a *zonal*. With this scenario,  a single zone in a region serves all inbound or outbound flow. Your frontend shares fate with the health of the zone. The data path is unaffected by failures in zones other than where it was guaranteed. You can use zonal frontends to expose an IP address per Availability Zone. 

Additionally, the use of zonal frontends directly for load-balanced endpoints within each zone is supported. You can use this configuration to expose per zone load-balanced endpoints to individually monitor each zone. For public endpoints, you can integrate them with a DNS load-balancing product like [Traffic Manager](../traffic-manager/traffic-manager-overview.md) and use a single DNS name.


:::image type="content" source="../load-balancer/media/az-zonal/zonal-lb-1.svg" alt-text="Figure depicts three zonal standard load balancers each directing traffic in a zone to three different subnets in a zonal configuration.":::


For a public load balancer frontend, you add a **zones** parameter to the public IP. This public IP is referenced by the frontend IP configuration used by the respective rule.

For an internal load balancer frontend, add a **zones** parameter to the internal load balancer frontend IP configuration. A zonal frontend guarantees an IP address in a subnet to a specific zone.


>[!IMPORTANT]
>A zonal configuration doesn't increase resilience - if that zone fails, the frontend becomes unavailable. For high availability, always use zone-redundant frontends with backend instances distributed across multiple zones.


### Non-zonal load balancer

Load Balancers can also be created in a non-zonal configuration by use of a "no-zone" frontend. In these scenarios, a public load balancer would use a public IP or public IP prefix, an internal load balancer would use a private IP. This option doesn't give a guarantee of redundancy. 

>[!NOTE]
>All public IP addresses that are upgraded from Basic SKU to Standard SKU will be of type "no-zone". Learn how to [Upgrade a public IP address in the Azure portal](../virtual-network/ip-services/public-ip-upgrade-portal.md).



### Region support

Zone-redundant Load Balancer resources can be deployed into [any region that supports availability zones](regions-list.md).


### Requirements

 - To use availability zones with Azure Load Balancer, you must use the Standard Load Balancer SKU. 
 - Basic Load Balancer doesn't support availability zones.
 - To create your resource, you need to have Network Contributor role or higher.


### Considerations

When using zone-redundant frontends, the Load Balancer service maintains the same frontend IP address during zone failures, eliminating the need for DNS updates or client reconfiguration. However, active connections might be reset during zone transitions.

For zonal frontends, a zone outage directly impacts frontend availability. The frontend becomes inaccessible if its assigned zone fails, requiring either pre-configured failover mechanisms or manual intervention to restore service.

Backend instance distribution is critical for resilience. Even with a zone-redundant frontend, if all backend instances are in a single zone, an outage in that zone causes complete application unavailability despite the load balancer remaining operational.

### Cost

Enabling zone redundancy for Standard Load Balancer doesn't incur additional charges beyond the standard pricing. You're billed based on the number of rules configured and data processed, regardless of zone configuration. For more information, see  [Azure Load Balancer pricing](https://azure.microsoft.com/pricing/details/load-balancer/)


### Configure availability zone support

- **Create a zone-redundant Load Balancer.** When creating a Standard Load Balancer, select *Zone-redundant* for the frontend IP configuration. For detailed steps, see [Create a zone-redundant load balancer](../load-balancer/quickstart-load-balancer-standard-public-portal.md).

- **Migrate from zonal to zone-redundant.** In the case where a region is augmented to have availability zones, any existing IPs would remain non-zonal like IPs used for load balancer frontends. To ensure your architecture can take advantage of the new zones, it's recommended that you create a new frontend IP. Once created, you can replace the existing non-zonal frontend with a new zone-redundant frontend. To learn how to migrate a VM to availability zone support, see [Migrate Load Balancer to availability zone support](./migrate-load-balancer.md).

- **Disable zone redundancy.** To move from zone-redundant to zonal configuration, you must create a new frontend with the desired zonal configuration and update your load balancing rules accordingly.


### Capacity planning and management

When planning capacity for zone resilience, ensure that each zone has sufficient backend capacity to handle the full load if other zones fail. Configure your backend pools with enough instances distributed across zones to maintain performance during zone outages.

For outbound connections, plan SNAT port allocation based on peak concurrent connections per instance. With zone failures potentially concentrating traffic on fewer instances, ensure adequate SNAT ports remain available. Configure multiple frontend IPs or outbound rules to increase available SNAT ports.

Monitor backend pool health and capacity metrics through Azure Monitor. Set alerts for unhealthy backend instance counts and high SNAT port utilization to proactively address capacity issues.

**Sources:**
- [Backend pool management](../load-balancer/backend-pool-management.md)
- [Outbound connections and SNAT](../load-balancer/load-balancer-outbound-connections.md)
- [Monitor Load Balancer metrics](../load-balancer/monitor-load-balancer.md)

### Normal operations

**Traffic routing between zones**. With zone-redundant Standard Load Balancer configured in an active/active model, incoming traffic is distributed across healthy backend instances in all availability zones based on the load balancing algorithm (default: five-tuple hash). The load balancer continuously evaluates backend health independently from each zone.

**Data replication between zones**. Azure Load Balancer is a network pass-through service that doesn't store or replicate application data. The service maintains its configuration state with synchronous replication across zones, ensuring immediate consistency of load balancing rules, health probe configurations, and backend pool membership across all zones.

**Sources:**
- [Load distribution algorithm](../load-balancer/load-balancer-distribution-mode.md)
- [How Load Balancer works](../load-balancer/load-balancer-overview.md#how-it-works)

### Zone-down experience

During an availability zone failure with zone-redundant Standard Load Balancer:

- **Detection and response**: Microsoft's monitoring systems automatically detect zone failures. The zone-redundant frontend continues operating from remaining zones without customer intervention.
- **Notification**: Zone failures are logged in Azure Service Health. Configure Service Health alerts to receive notifications about zone-level events affecting your resources.
- **Active requests**: Existing TCP/UDP flows from the failed zone are reset. New connections are automatically established through remaining healthy zones.
- **Expected data loss**: As a stateless network service, Load Balancer doesn't store application data, so no data loss occurs at the load balancer layer.
- **Expected downtime**: With zone-redundant configuration, there's no downtime for the load balancer service. Brief connection resets occur for flows originating from the failed zone.
- **Traffic rerouting**: Health probes from remaining zones detect backend failures in the affected zone. Traffic automatically redistributes to healthy instances in operational zones based on the configured load balancing algorithm.


<!-- John: This content was in the old reliability guide.But is this info in the WAF guide already? Not sure we want to mention this here or not. Your call -->
#### Multiple frontends

Using multiple frontends allow you to load balance traffic on more than one port and/or IP address. When designing your architecture, ensure you account for how zone redundancy interacts with multiple frontends. If your goal is to always have every frontend resilient to failure, then all IP addresses assigned as frontends must be zone-redundant. If a set of frontends is intended to be associated with a single zone, then every IP address for that set must be associated with that specific zone. A load balancer isn't required in each zone. Instead, each zonal front end, or set of zonal frontends, could be associated with virtual machines in the backend pool that are part of that specific availability zone.


**Sources:**
- [Zone redundant Load Balancer behavior](../load-balancer/load-balancer-standard-availability-zones.md#zone-redundant)
- [Azure Service Health](../service-health/overview.md)
- [Load Balancer health probes](../load-balancer/load-balancer-custom-probe-overview.md)

### Zone recovery

When an availability zone recovers, Standard Load Balancer automatically resumes normal operations:

The zone-redundant frontend immediately begins serving traffic from the recovered zone alongside other operational zones. Health probes from the recovered zone resume evaluating backend instances. As backend instances in the recovered zone pass health checks, they're automatically added back to the healthy backend pool. Traffic distribution rebalances across all available zones based on the load balancing algorithm and backend health status.

No customer action is required for zone recovery with zone-redundant configurations. The service handles all aspects of recovery automatically.

**Sources:**
- [Automatic recovery behavior](../load-balancer/load-balancer-standard-availability-zones.md#zone-redundant)
- [Health probe behavior](../load-balancer/load-balancer-custom-probe-overview.md)

### Testing for zone failures

You can simulate zone failure impacts on your backend instances using Azure Chaos Studio. Create experiments that shut down or isolate virtual machines in specific availability zones to validate your application's resilience and load balancer behavior during zone outages.

For comprehensive testing, use the [Virtual Machine Shutdown fault](../chaos-studio/chaos-studio-fault-library.md#virtual-machine-shutdown) targeting VMs in a specific zone, or implement [Network Security Group faults](../chaos-studio/chaos-studio-fault-library.md#network-security-group-security-rule) to block health probe traffic from specific zones.

Standard Load Balancer with zone-redundant frontend configuration doesn't require customer-initiated failover testing, as zone resilience is managed automatically by the platform.

**Sources:**
- [Azure Chaos Studio overview](../chaos-studio/chaos-studio-overview.md)
- [Chaos Studio fault library](../chaos-studio/chaos-studio-fault-library.md)
- [Test applications for availability zone failure](./availability-zones-overview.md#testing)

## Multi-region support

Azure Standard Load Balancer provides native multi-region support through Cross-region Load Balancer, which enables global load balancing across Azure regions. Cross-region Load Balancer provides a single static anycast IP address that automatically routes traffic to the optimal regional deployment based on client proximity and regional health.

With Cross-region Load Balancer, you deploy Standard public load balancers in each region as "origins" and the cross-region load balancer acts as a global frontend. This configuration provides automatic regional failover without DNS changes, as the anycast IP remains constant while traffic is rerouted to healthy regions.



**Sources:**
- [Cross-region Load Balancer overview](../load-balancer/cross-region-overview.md)
- [Cross-region Load Balancer FAQ](../load-balancer/load-balancer-faqs.yml#cross-region-load-balancer)

### Region support

You can only deploy your global load balancer or Public IP in Global tier in one of the listed Home regions.

A **Home region** is where the global load balancer or Public IP Address of Global tier is deployed. 
This region doesn't affect how the traffic is routed. If a home region goes down, traffic flow is unaffected.

### Home regions
* Central US
* East Asia
* East US 2
* North Europe
* Southeast Asia
* UK South
* US Gov Virginia
* West Europe
* West US


A **participating region** is where the global public IP of the load balancer is being advertised.

Traffic started by the user travels to the closest participating region through the Microsoft core network. 

Global load balancer routes the traffic to the appropriate regional load balancer.

:::image type="content" source="../load-balancer/media/cross-region-overview/multiple-region-global-traffic.png" alt-text="Diagram of multiple region global traffic.":::

### Participating regions
* Australia East 
* Australia Southeast 
* Central India 
* Central US 
* East Asia 
* East US 
* East US 2 
* Japan East 
* North Central US 
* North Europe 
* South Central US 
* Southeast Asia 
* UK South 
* US DoD Central
* US DoD East
* US Gov Arizona
* US Gov Texas
* US Gov Virginia
* West Central US 
* West Europe 
* West US 
* West US 2 

> [!NOTE]
> The backend regional load balancers can be deployed in any publicly available Azure Region and isn't limited to just participating regions.


### Requirements

To use Cross-region Load Balancer, you must use Standard public Load Balancers in each participating region as the origins. Basic Load Balancer isn't supported as an origin for Cross-region Load Balancer.

**Sources:**
- [Cross-region Load Balancer prerequisites](../load-balancer/cross-region-overview.md#prerequisites)
- [Supported configurations](../load-balancer/cross-region-overview.md#supported-configurations)

### Considerations

Cross-region Load Balancer operates at Layer 4 and doesn't provide application-layer features like SSL/TLS termination, cookie-based session affinity, or URL path-based routing. For these capabilities, consider using Azure Front Door in conjunction with regional load balancers.

Client connections to Cross-region Load Balancer use asymmetric routing - inbound traffic flows through the cross-region and regional load balancers, while return traffic bypasses the Cross-region Load Balancer for optimal performance.

Health probes for Cross-region Load Balancer originate from the global load balancing infrastructure and evaluate the health of regional load balancer frontends, not individual backend instances.

**Sources:**
- [Cross-region Load Balancer architecture](../load-balancer/cross-region-overview.md#architecture)
- [Traffic flow](../load-balancer/cross-region-overview.md#traffic-flow)
- [Health probes](../load-balancer/cross-region-overview.md#health-probes)

### Cost

Cross-region Load Balancer incurs charges for the global load balancer resource in addition to the regional Standard Load Balancers. Pricing includes a fixed hourly charge plus data processing charges for traffic routed through the Cross-region Load Balancer.

**Sources:**
- [Azure Load Balancer pricing](https://azure.microsoft.com/pricing/details/load-balancer/)
- [Cross-region Load Balancer pricing](../load-balancer/cross-region-overview.md#pricing)

### Configure multi-region support

- **Create Cross-region Load Balancer.** Deploy Standard public load balancers in your target regions first, then create the Cross-region Load Balancer with these regional load balancers as backends. For detailed steps, see [Create a cross-region load balancer](../load-balancer/tutorial-cross-region-portal.md).

- **Disable Cross-region Load Balancer.** To disable cross-region functionality, remove the Cross-region Load Balancer resource. Regional load balancers continue operating independently.

- **Migrate to Cross-region Load Balancer.** For existing regional deployments, create the Cross-region Load Balancer and add your existing regional Standard Load Balancers as origins. Update client endpoints to use the global anycast IP. See [Tutorial: Create a cross-region load balancer](../load-balancer/tutorial-cross-region-portal.md).

**Sources:**
- [Create cross-region load balancer - Portal](../load-balancer/tutorial-cross-region-portal.md)
- [Create cross-region load balancer - PowerShell](../load-balancer/tutorial-cross-region-powershell.md)
- [Create cross-region load balancer - CLI](../load-balancer/tutorial-cross-region-cli.md)

### Capacity planning and management

For Cross-region Load Balancer deployments, each regional deployment must have sufficient capacity to handle the total global traffic load in case other regions fail. Plan backend capacity in each region to support 100% of peak traffic.

Monitor regional load balancer metrics and health probe status through Azure Monitor. Set up alerts for regional probe health to detect regional degradation before complete failure. Configure autoscaling on regional backend pools to handle traffic surges during regional failovers.

**Sources:**
- [Monitor Cross-region Load Balancer](../load-balancer/monitor-load-balancer.md)
- [Load Balancer metrics](../load-balancer/load-balancer-standard-diagnostics.md)

### Normal operations

**Traffic routing between regions**. Cross-region Load Balancer operates in an active/active model, routing each client connection to the lowest-latency healthy regional deployment based on Microsoft's global network proximity data and regional health status. The geo-proximity load-balancing algorithm is based on the geographic location of your users and your regional deployments. 

Traffic started from a client hits the closest participating region and travel through the Microsoft global network backbone to arrive at the closest regional deployment. 

For example, you have a global load balancer with standard load balancers in Azure regions:

* West US
* North Europe

If a flow is started from Seattle, traffic enters West US. This region is the closest participating region from Seattle. The traffic is routed to the closest region load balancer, which is West US.

Azure global load balancer uses geo-proximity load-balancing algorithm for the routing decision. 

The configured load distribution mode of the regional load balancers is used for making the final routing decision when multiple regional load balancers are used for geo-proximity. 

For more information, see [Configure the distribution mode for Azure Load Balancer](../load-balancer/load-balancer-distribution-mode.md).

Egress traffic follows the routing preference set on the regional load balancers.


**Data replication between regions**. Cross-region Load Balancer doesn't replicate application data between regions. Configuration changes to the Cross-region Load Balancer are globally replicated with strong consistency. Regional load balancer configurations remain independent and must be managed separately in each region.

**Sources:**
- [Cross-region traffic distribution](../load-balancer/cross-region-overview.md#traffic-flow)
- [Global load balancing](../load-balancer/cross-region-overview.md#global-load-balancing)

### Region-down experience

During a regional failure with Cross-region Load Balancer:

- **Detection and response**: Microsoft's global monitoring automatically detects regional load balancer failures through health probes. The health probe of the global load balancer gathers information about availability of each regional load balancer every 5 seconds. If one regional load balancer drops its availability to 0, global load balancer detects the failure. The regional load balancer is then taken out of rotation. 

    :::image type="content" source="../load-balancer/media/cross-region-overview/global-region-view.png" alt-text="Diagram of global region traffic view." border="true":::

- **Notification**: Regional failures appear in Azure Service Health. Configure alerts to notify your operations team of regional events.
- **Active requests**: Existing connections to the failed region are terminated. New connections are automatically routed to the next-best healthy region based on latency.
- **Expected data loss**: As a stateless service, Load Balancer doesn't cause data loss. Application-level data loss depends on your data replication strategy.
- **Expected downtime**: With Cross-region Load Balancer, there's no service downtime. Clients experience brief connection resets while traffic redistributes to healthy regions.
- **Traffic rerouting**: The anycast IP remains constant. DNS updates aren't required. Traffic automatically flows to remaining healthy regions based on proximity and capacity.

**Sources:**
- [Automatic failover behavior](../load-balancer/cross-region-overview.md#failover)
- [Health probe detection](../load-balancer/cross-region-overview.md#health-probes)

### Region recovery

When a region recovers from failure:

Cross-region Load Balancer health probes automatically detect when the regional load balancer becomes healthy again. Traffic gradually returns to the recovered region based on its latency advantage for different clients. The anycast IP address remains unchanged throughout the failback process, requiring no client or DNS updates.

Load distribution rebalances automatically based on client proximity to all healthy regions, including the recovered region.

**Sources:**
- [Automatic recovery](../load-balancer/cross-region-overview.md#failover)
- [Health-based routing](../load-balancer/cross-region-overview.md#health-probes)

### Testing for region failures

To test regional failure scenarios with Cross-region Load Balancer, temporarily disable the health probe endpoints on all backend instances in a specific region. This simulates a regional failure and validates that Cross-region Load Balancer correctly redirects traffic to remaining healthy regions.

You can also use Azure Chaos Studio to create region-level experiments that shut down or isolate all resources in a specific region. Monitor the Cross-region Load Balancer metrics during testing to verify proper failover behavior and measure failover timing.

**Sources:**
- [Test cross-region failover](../load-balancer/tutorial-cross-region-portal.md#test-the-load-balancer)
- [Azure Chaos Studio](../chaos-studio/chaos-studio-overview.md)

### Alternative multi-region approaches

If Cross-region Load Balancer doesn't meet your requirements, consider these alternatives:

For DNS-based global load balancing across regions, use [Azure Traffic Manager](../traffic-manager/traffic-manager-overview.md) with regional Standard Load Balancers. Traffic Manager provides geographic routing, priority routing, and weighted routing methods, though it requires DNS TTL considerations for failover timing.

For HTTP/HTTPS workloads requiring application-layer features, use [Azure Front Door](../frontdoor/front-door-overview.md) as a global entry point with regional load balancers or Application Gateways as origins. This provides SSL termination, WAF capabilities, and URL-based routing alongside global load balancing.

For multi-region architectures with complex routing requirements, review [Multi-region load balancing with Traffic Manager and Application Gateway](/azure/architecture/high-availability/reference-architecture-traffic-manager-application-gateway).

**Sources:**
- [Traffic Manager overview](../traffic-manager/traffic-manager-overview.md)
- [Azure Front Door overview](../frontdoor/front-door-overview.md)
- [Multi-region architectures](/azure/architecture/guide/networking/global-web-applications/overview)

## Reliability during service maintenance

Azure Load Balancer performs automatic platform updates with zero downtime for zone-redundant configurations. The service uses safe deployment practices that roll out updates gradually across regions and availability zones.

During platform maintenance, zone-redundant Load Balancers continue operating normally with no impact to traffic flow. Health probes continue monitoring backend instances, and traffic distribution remains unaffected.

For your application maintenance, implement connection draining by removing instances from health probe rotation before updates. Configure appropriate health probe intervals to ensure smooth traffic shifting during backend maintenance.

**Sources:**
- [Azure Load Balancer updates](../load-balancer/load-balancer-overview.md#secure-by-default)
- [Safe deployment practices](../load-balancer/load-balancer-overview.md#built-in-security)

## Service-level agreement

Because availability zones are physically separate and provide distinct power source, network, and cooling, SLAs (Service-level agreements) can increase. For more information, see the [Service Level Agreements (SLA) for Online Services](https://www.microsoft.com/licensing/docs/view/Service-Level-Agreements-SLA-for-Online-Services?lang=1).


**Sources:**
- [SLA for Load Balancer](https://azure.microsoft.com/support/legal/sla/load-balancer/)
- [Azure Service Level Agreements](https://azure.microsoft.com/support/legal/sla/)

### Related content

- [Azure reliability overview](/azure/reliability/overview)
- [Availability zones overview](/azure/reliability/availability-zones-overview)
- [Azure regions and availability zones](/azure/reliability/availability-zones-service-support)
- [Cross-region replication in Azure](/azure/reliability/cross-region-replication-azure)
- [Azure Load Balancer documentation](../load-balancer/load-balancer-overview.md)
- [Azure Well-Architected Framework - Reliability pillar](/azure/well-architected/reliability/)
- [Migrate Load Balancer to availability zone support](./migrate-load-balancer.md)