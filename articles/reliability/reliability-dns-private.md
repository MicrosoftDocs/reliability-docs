---
title: Reliability in Azure DNS private zones
description: Learn how to make Azure DNS private zones resilient to various potential outages and problems, including transient faults and region-wide failures, and learn about the service-level agreement.
author: glynnniall
ms.author: glynnniall
ms.topic: reliability-article
ms.custom: subject-reliability
ms.service: azure-dns
ms.date: 08/31/2026
---

# Reliability in Azure DNS private zones

[Azure DNS private zones](/azure/dns/private-dns-overview) provide secure name resolution within Azure virtual networks. You can scope private DNS zones to one or more virtual networks, and organizations typically use them for internal applications. The hostnames you resolve are local DNS names that aren't publicly accessible via the internet. The resolved IP addresses are often private IP addresses that aren't accessible from the internet. Azure DNS is a global service that isn't bound to any specific availability zone or single region.

[!INCLUDE [Shared responsibility](includes/reliability-shared-responsibility-include.md)]

This article describes how to make Azure DNS private zones resilient to various potential outages and problems, including transient faults and region-wide failures. It also provides key information about the Azure DNS private zones service-level agreement (SLA).

## Production deployment recommendations for reliability

For production workloads, we recommend that you follow these recommendations:

> [!div class="checklist"]
> - **Configure appropriate TTL values:** Set time-to-live (TTL) values that balance performance with recovery time. Lower TTL values enable faster failover but increase query volume. Consider 300 seconds (5 minutes) as a starting point for production workloads.
>
> - **Shard large DNS zones:** If you have a large DNS zone, consider [sharding your zone](/azure/dns/sharding-private-dns-zones) to improve your overall reliability and operational efficiency.

## Reliability architecture overview

[!INCLUDE [Introduction to reliability architecture overview section](includes/reliability-architecture-overview-introduction-include.md)]

### Logical architecture

The primary resource you deploy is a *zone*, which represents a set of DNS records that map hostnames (domain names) to IP addresses. The hostnames that the zone resolves are usually local DNS names that aren't publicly accessible through the internet.

You create private DNS zones as standalone resources and link them to specific virtual networks by creating [*virtual network links*](/azure/dns/private-dns-virtual-network-links). When DNS requests come from clients within those virtual networks, the private DNS zones participate in the resolution process. You can [manually create entries in a DNS zone](/azure/dns/dns-private-records) or configure [autoregistration of VMs](/azure/dns/private-dns-autoregistration) on virtual network links. Azure DNS private zones support DNS resolution between virtual networks across Azure regions, even without explicitly peering the virtual networks. However, all virtual networks must be linked to the private DNS zone.

The DNS [name resolution process](/azure/virtual-network/virtual-networks-name-resolution-for-vms-and-role-instances?toc=%2Fazure%2Fdns%2Ftoc.json) involves multiple components, including DNS resolvers and intermediate layers that process requests before reaching the authoritative DNS servers. Private zones use the same DNS protocols and behaviors as public zones, including TTL values and caching mechanisms.

> [!IMPORTANT]
> The reliability of your overall solution depends on the configuration of the resources that your DNS records refer to, such as virtual machines and load balancers.
>
> This article doesn't cover those resources, but their availability configurations directly affect your application's resilience. Review the [reliability guides for Azure services in your solution](./overview-reliability-guidance.md) to learn how each service supports your reliability requirements.

### Physical architecture

Azure DNS is a nonregional service. Microsoft deploys its infrastructure across multiple availability zones in multiple Azure regions worldwide. This design enables Azure DNS to remain resilient during an availability zone or region outage because infrastructure in another zone or region continues to respond to resolution requests.

Global internet protocols such as Anycast, DNS, and Border Gateway Protocol (BGP) automatically route incoming DNS resolution requests to the nearest healthy Azure DNS infrastructure.

## Resilience to transient faults

[!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)]

Azure DNS handles transient faults through its global DNS infrastructure.

If a transient fault occurs during DNS resolution, the client or intermediate resolver should retry the request. Configure timeout values appropriately. A timeout of 2 to 5 seconds is usually sufficient for a DNS client.

Each DNS record's time to live (TTL) also affects how your solution handles faults. If the TTL is very low, clients make more requests to Azure DNS, which creates more opportunities for transient faults. If the TTL is very high, in the event of a true fault in a backend server that requires you to redirect to a different IP address, clients might experience delays in failover until the TTL expires. Configure TTLs carefully to balance availability, latency, and responsiveness.

## Resilience to availability zone failures

[!INCLUDE [Resilience to availability zone failures](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)]

Azure DNS operates as a nonregional service. Microsoft distributes its infrastructure across multiple availability zones in multiple Azure regions and replicates changes to your private DNS zones across that infrastructure. You don't select availability zones or configure zone redundancy. During an availability zone outage, infrastructure in another zone or region continues to respond to resolution requests.

If a resource that you deploy to a single availability zone, such as a virtual machine (VM), becomes unavailable during a zone failure, Azure DNS continues to return the resource's configured IP address because it doesn't monitor endpoint health. If you fail over to a resource in a healthy zone, you're responsible for updating the DNS record so that clients use the healthy resource. Alternatively, place the resources behind a zone-redundant load balancer that directs traffic to VMs in healthy zones.

## Resilience to region-wide failures

Azure DNS private zones are resilient to region outages because zone data is globally available. If a region has an outage, its virtual networks and resources such as VMs might be unavailable, but name resolution continues to work.

The following example shows how private zone data remains available across multiple regions. The private zone `azure.contoso.com` is linked to virtual networks in three regions: region A, region B, and region C. Autoregistration is enabled in regions A and B. The diagram shows region A experiencing an outage:

:::image type="complex" source="./media/reliability-dns-private/region-outage.svg" border="false" lightbox="./media/reliability-dns-private/region-outage.svg" alt-text="Diagram that shows a private DNS zone linked to virtual networks in three regions while region A is unavailable.":::
   The private DNS zone `azure.contoso.com` contains records for VM1 in region A and VM2 in region B. Autoregistration is enabled for the virtual networks in regions A and B, and the zone is also linked to the virtual network in region C. Region A and VM1 are unavailable, but VMs in regions B and C can still query the zone and resolve VM1's IP address.
:::image-end:::

Suppose a temporary outage occurs in region A. VMs in regions B and C can still query DNS names in the private zone, including names that are autoregistered from region A. They can continue to resolve the IP address of VM1 in region A, even though VM1 isn't available. Service interruption in region A doesn't affect name resolution in the other regions.

The preceding example doesn't show a disaster recovery scenario in which your solution fails over to a replacement for VM1 in another region. However, because private zones are global, you can recreate VM1 in another region's virtual network to take over the workload.

If you create virtual networks and networking resources across multiple regions, you need to plan and implement your multiregion strategy for applications that require cross-region failover.

## Resilience to security threats and misconfiguration

Security attacks and configuration errors are two of the most significant reliability risks for DNS zones. Several classes of attacks specifically target DNS resolution, and accidental misconfiguration can disrupt your workloads just as severely.

For comprehensive security guidance specific to private DNS zones, see [Protecting private DNS Zones and Records](/azure/dns/dns-protect-private-zones-recordsets).

## Resilience to service outages

Azure DNS is a highly resilient service, with a 100% availability SLA when your application meets certain conditions. Service outages are extremely unusual, but network or other infrastructure problems can disrupt connectivity to the Azure DNS service.

### Monitor for service outages

[!INCLUDE [Service down notification partial (Azure Service Health and Resource Health)](./includes/reliability-region-down-notification-service-partial-include.md)]

### Test for service outages

Azure Chaos Studio provides a set of faults to simulate problems with DNS resolution. For example, the Chaos Studio agent provides the [DNS Failure](/azure/chaos-studio/chaos-studio-fault-library#dns-failure) fault type, and Azure Kubernetes Service (AKS) Chaos Mesh provides the [DNS Chaos](/azure/chaos-studio/chaos-studio-fault-library#aks-chaos-mesh-dns-chaos) capability. You can use these fault types to test how your applications and infrastructure respond when DNS resolution requests fail, which might occur during a partial network failure.

## Resilience to portal and management tool outages

If you manage your DNS zone in the Azure portal, prepare for scenarios where you can't access it, especially if you need to reconfigure your DNS zone during a platform outage.

You can use various tools to deploy and manage Azure DNS private zones. Learn how to use [Azure CLI](/azure/dns/private-dns-getstarted-cli) or [Azure PowerShell](/azure/dns/private-dns-getstarted-powershell) to manage your private zone. Alternatively, use infrastructure as code (IaC), such as Bicep or [Terraform](/azure/dns/dns-private-zone-terraform), to deploy and configure your private zone. These tools remain operational even if the Azure portal is degraded.

## Backup and restore

Azure DNS is a stateless service. It doesn't provide managed backups or point-in-time restore for private DNS zones.

To preserve the complete Azure resource configuration, define your private DNS zones by using IaC, such as Bicep or Terraform, and store the definitions in source control. Periodically test the definitions so that you can use them to redeploy your configuration.

## Resilience to service maintenance

[!INCLUDE [Service maintenance (no special callouts)](includes/reliability-maintenance-include.md)]

## Service-level agreement

[!INCLUDE [Service-level agreement](includes/reliability-service-level-agreement-include.md)]

Azure DNS provides a 100% availability SLA for valid DNS query responses when you meet certain conditions. These conditions include retrying failed requests for at least 60 consecutive seconds. Review the SLA document for the detailed conditions.

## Related content

- [Azure DNS private zones documentation](/azure/dns/private-dns-overview)
- [Azure DNS FAQ](/azure/dns/dns-faq)
- [Protecting private DNS Zones and Records - Azure DNS](/azure/dns/dns-protect-private-zones-recordsets)
- [Azure DNS public zones reliability](reliability-dns-public.md)
