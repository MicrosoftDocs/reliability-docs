---
title: Reliability in Azure DNS public zones
description: Learn how to make Azure DNS public zones resilient to various potential outages and problems, including transient faults, availability zone failures, and region-wide failures.
author: glynnniall
ms.author: glynnniall
ms.topic: reliability-article
ms.custom: subject-reliability
ms.service: azure-dns
ms.date: 08/24/2026
---

# Reliability in Azure DNS public zones

[Azure DNS](/azure/dns/dns-overview) provides name resolution by using Microsoft Azure infrastructure. This article focuses on public DNS zones, which you typically create for domains that you own and use to publish records for applications and services that are available on the internet. The hostnames you resolve are publicly accessible DNS names, and the resolved IP addresses are usually public IP addresses reachable from the internet.

Azure DNS is a nonregional service that isn't bound to a specific availability zone or Azure region.

[!INCLUDE [Shared responsibility](includes/reliability-shared-responsibility-include.md)]

This article describes how Azure DNS public zones respond to transient faults, availability zone failures, region-wide failures, service outages, security threats and misconfiguration, portal and management tool outages, and service maintenance. It also describes how to protect and restore your zone configuration and explains key service-level agreement (SLA) requirements.

## Production deployment recommendations for reliability

For production deployments of Azure DNS public zones, follow these recommendations to enhance reliability:

> [!div class="checklist"]
> - **Delegate to all of the name servers:** Azure DNS assigns four name servers to each public DNS zone. Configure your domain delegation to use all four name servers. This configuration provides fault isolation and is required to qualify for the Azure DNS SLA.
>
> - **Configure appropriate TTL values:** Set time-to-live (TTL) values that balance query volume with how quickly clients receive record changes. Lower TTL values allow clients to receive changes sooner but increase query volume. Higher TTL values reduce query volume but can delay failover after you change a record.
>
> - **Use alias records for supported Azure resources:** [Alias records](/azure/dns/dns-alias) automatically reflect changes to an underlying Azure resource during DNS resolution and help prevent stale DNS records.

## Reliability architecture overview

[!INCLUDE [Introduction to reliability architecture overview section](includes/reliability-architecture-overview-introduction-include.md)]

### Logical architecture

The primary resource you deploy is a *zone*, which contains the DNS record sets for a domain. A record set associates a DNS name with a value, such as an IP address or endpoint. The names that a public DNS zone resolves are accessible through the internet.

To make Azure DNS authoritative for your domain, [delegate the domain](/azure/dns/dns-domain-delegation) to the name servers that Azure assigns when you create the zone. After delegation is in place, you create record sets for the [DNS record types that Azure DNS supports](/azure/dns/dns-zones-records#record-types). You can also create [alias records](/azure/dns/dns-alias) that reference Azure resources such as public IP addresses, Traffic Manager profiles, and Azure Front Door endpoints, so that the DNS record stays synchronized with the target resource.

During [DNS resolution](/azure/dns/dns-domain-delegation#resolution-and-delegation), recursive DNS resolvers follow the DNS hierarchy to reach the Azure DNS authoritative name servers for your zone.

> [!IMPORTANT]
> Azure DNS resolves names but doesn't monitor endpoint health or route application traffic. The reliability of your overall solution depends on the configuration of the resources that your DNS records refer to, such as virtual machines and load balancers.
>
> This article doesn't cover those resources, but their availability configurations directly affect your application's resilience. Review the [reliability guides for Azure services in your solution](./overview-reliability-guidance.md) to learn how each service supports your reliability requirements.

### Physical architecture

Azure DNS operates as a nonregional service and deploys its infrastructure across multiple availability zones in multiple Azure regions worldwide. This design enables Azure DNS to remain resilient during an availability zone or region outage because infrastructure in another zone or region continues to respond to resolution requests.

Global internet protocols like Anycast, DNS, and BGP automatically route incoming DNS resolution requests to the nearest healthy Azure DNS infrastructure.

The Azure DNS serving plane operates in an active-active configuration across two independent serving stacks: one running on Linux and one running on Windows. These stacks share no code and no underlying hardware. Because they're independent, a bug, vulnerability, or failure that affects one stack doesn't affect the other. This independence reduces the risk of a complete service outage caused by a single point of failure and helps protect against certain classes of zero-day vulnerabilities.

## Resilience to transient faults

[!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)]

Azure DNS handles transient faults through its global DNS infrastructure.

If a transient fault occurs during DNS resolution, the client or intermediate resolver should retry according to its configured DNS retry behavior. Between 2 and 5 seconds is usually a sufficient timeout for a DNS client.

Each DNS record’s time to live (TTL) also affects how your solution handles faults. If the TTL is very low, clients need to make more requests to Azure DNS and there are more potential opportunities for transient faults to arise. If the TTL is very high, in the event of a true fault in a backend server that requires you to redirect to a different IP address, clients might experience delays in failover until the TTL expires. Configure TTLs carefully to balance availability, latency, and responsiveness.

## Resilience to availability zone failures

[!INCLUDE [Resilience to availability zone failures](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)]

Azure DNS operates as a nonregional service. Microsoft distributes its infrastructure across multiple availability zones in multiple Azure regions and replicates changes to your public DNS zones across that infrastructure. You don't select availability zones or configure zone redundancy. During an availability zone outage, infrastructure in another zone or region continues to respond to resolution requests.

If a resource that you deploy to a single availability zone, such as a virtual machine (VM), becomes unavailable during a zone failure, Azure DNS continues to return the resource's configured IP address because it doesn't monitor endpoint health. If you fail over to a resource in a healthy zone, you're responsible for updating the DNS record so that clients use the healthy resource. Alternatively, place the resources behind a zone-redundant load balancer that directs traffic to VMs in healthy zones.

## Resilience to region-wide failures

DNS zones are resilient to region outages because zone data is globally available and deployed to multiple Azure regions. If a region has an outage, resources that you deployed in that region, such as virtual networks and VMs, might be unavailable, but Azure DNS continues to resolve records in your zone.

If you have a solution that needs to switch between multiple regions, such as for disaster recovery purposes, consider using [Azure Traffic Manager](/azure/traffic-manager/traffic-manager-overview) or [Azure Front Door](/azure/frontdoor/front-door-overview). These services provide automated failover capabilities, which you can use if a region is unhealthy.

## Resilience to security threats and misconfiguration

Security attacks and configuration errors are two of the most significant reliability risks for DNS zones. Several classes of attacks specifically target DNS resolution, and accidental misconfiguration can disrupt your workloads just as severely.

For comprehensive security guidance specific to public DNS zones, see [Secure your Azure DNS deployment](/azure/dns/secure-dns) and [Protecting DNS Zones and Records](/azure/dns/dns-protect-zones-recordsets).

## Resilience to service outages

Azure DNS is a highly resilient service, with a 100% availability SLA when your application meets certain conditions. Service outages are extremely unusual, but network problems or problems with other infrastructure can disrupt connectivity to the Azure DNS service.

Azure DNS resilience is partly due to its [globally distributed, active-active serving plane architecture](#physical-architecture).

### Use multiple name servers

Azure DNS assigns four name servers to each public DNS zone. When you delegate your domain, configure all four name servers. If a resolver can't reach one name server, it can query another.

### Monitor for service outages

Use [Azure Service Health](/azure/service-health/overview) to monitor the health of Azure DNS. Configure [Service Health alerts](/azure/service-health/alerts-activity-log-service-notifications-portal) to notify you about service incidents.

### Test for service outages

Azure Chaos Studio provides faults that simulate DNS resolution failures from within some types of test workloads. These faults don't trigger an outage in Azure DNS. The Chaos Studio agent provides the [DNS Failure](/azure/chaos-studio/chaos-studio-fault-library#dns-failure) fault, and AKS Chaos Mesh provides the [DNS Chaos](/azure/chaos-studio/chaos-studio-fault-library#aks-chaos-mesh-dns-chaos) capability. Use these faults to test how your applications and infrastructure respond when DNS resolution fails, such as during a partial network failure.

## Resilience to portal and management tool outages

If you manage your public DNS zone in the Azure portal, prepare an alternative management path for scenarios where you can't access the portal, especially if you might need to reconfigure the zone during an outage.

If the Azure portal is unavailable, use the [Azure CLI](/azure/dns/dns-getstarted-cli), [Azure PowerShell](/azure/dns/dns-getstarted-powershell), or infrastructure as code (IaC) such as [Bicep](/azure/dns/dns-get-started-bicep) or [Terraform](/azure/dns/dns-get-started-terraform) to manage your public DNS zone. These tools remain operational even if the Azure portal is degraded.

## Backup and restore

Azure DNS is a stateless service. It doesn't provide managed backups or point-in-time restore for public DNS zones.

To preserve the complete Azure resource configuration, define your public DNS zones by using IaC, such as Bicep or Terraform, and store the definitions in source control. Test the definitions periodically so that you can use them to redeploy your configuration.

As an additional record-level recovery option, [export a BIND-compatible zone file](/azure/dns/dns-import-export). Zone file import has limitations and doesn't preserve every Azure-specific resource setting, so don't use an exported zone file as your only recovery artifact. Review the documented import limitations and verify the records after you restore a zone.

## Resilience to service maintenance

[!INCLUDE [Service maintenance (no special callouts)](includes/reliability-maintenance-include.md)]

## Service-level agreement

[!INCLUDE [Service-level agreement](includes/reliability-service-level-agreement-include.md)]

Azure DNS provides a 100% availability SLA for valid DNS query responses, as long as certain conditions are met. These conditions include retrying failed requests repeatedly for at least 60 consecutive seconds, and using [all of the name servers](/azure/dns/dns-domain-delegation) that Azure DNS assigns to your zone. Review the SLA document for the detailed conditions.

## Related content

- [Azure DNS public zones overview](/azure/dns/public-dns-overview)
- [Azure DNS FAQ](/azure/dns/dns-faq)
- [Import and export an Azure DNS zone file](/azure/dns/dns-import-export)
- [Secure your Azure DNS deployment](/azure/dns/secure-dns)
- [Protect Azure DNS zones and records](/azure/dns/dns-protect-zones-recordsets)
- [Azure Private DNS overview](/azure/dns/private-dns-overview)
