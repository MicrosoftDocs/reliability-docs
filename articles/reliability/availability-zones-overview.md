---
title: What are Azure availability zones?
description: Learn about availability zones and how to use them to design resilient solutions.
ms.service: azure
ms.subservice: azure-reliability
ms.topic: conceptual
ms.date: 02/11/2026<img width="781" height="130" alt="image" src="https://github.com/user-attachments/assets/93e8c23f-c54e-4d29-a9af-1e25551985af" />

ms.author: glynnniall
author: glynnniall
ms.custom: subject-reliability, ai-video-concept
#customer intent: As a cloud architect or reliability engineer, I want a concise overview of availability zones and practical guidance for building zone-resilient architectures so that I can minimize downtime and meet performance and latency requirements.
---

# What are availability zones?

>[!VIDEO https://learn-video.azurefd.net/vod/player?id=d36b5b2d-8bd2-43df-a796-b0c77b2f82fc]

Many Azure regions provide *availability zones*, which are separated groups of datacenters within a region. Each availability zone has independent power, cooling, and networking infrastructure, so that if one zone experiences an outage, then regional services, capacity, and high availability are supported by the remaining zones. Some Azure services automatically use multiple zones, while others require you to configure multiple-zone deployment.

Availability zones don't only protect against large-scale outages that affect an entire zone. They also provide resilience against smaller-scoped failures, such as a server rack or cluster failure within a zone. In these scenarios, workloads can continue running in other zones, even if parts of the affected zone remain operational.

The following diagram shows several example Azure regions. Regions 1 and 2 support availability zones, and regions 3 and 4 don't have availability zones. Some zones have one datacenter and others have multiple.

:::image type="content" source="media/availability-zones-overview/regions-availability-zones.svg" alt-text="Diagram of physically separate availability zone locations within an Azure region." border="false":::

> [!TIP]
> To see which regions support availability zones, see [List of Azure regions](regions-list.md).

## Datacenters and availability zones

An availability zone is a *logical grouping* of one or more physically separate datacenters within a region. Each availability zone is built in a way that if something goes wrong in one (like a power outage or network issue), the others keep working. A single datacenter doesn’t offer this level of protection on its own.

Availability zones are typically separated by several kilometers, and usually are within 100 kilometers of each other. This provides low-latency connectivity while reducing the likelihood local outages or weather events impact multiple zones simultaneously.

Datacenter locations are selected by using rigorous vulnerability risk assessment criteria. This process identifies all significant datacenter-specific risks and considers shared risks between availability zones.

<a name='zonal-and-zone-redundant-services'></a>

## Types of availability zone support

Azure services can provide different types of availability zone support for their resources, and most services support deploying either *zone-redundant* or *zonal* resources:

- **Zone-redundant resources**: Zone-redundant resources are replicated or distributed across multiple availability zones by the service. For example, zone-redundant data services replicate the data across multiple zones so that a failure in one zone doesn't affect the availability of the data. Some services are automatically zone-redundant in supported regions, while other services require that you configure your resource to be zone-redundant. For most services, Microsoft selects the zones your resources use, but sometimes you can select the set of zones.

    With zone-redundant deployments, Microsoft manages spreading requests across zones and the replication of data across zones. If an outage occurs in an availability zone, Microsoft manages failover to another zone automatically.

    :::image type="content" source="media/availability-zones-overview/zone-redundant.svg" alt-text="Diagram of a zone-redundant resource deployed across three zones." border="false":::

- **Zonal resources**: A zonal resource is deployed to a single availability zone that you select yourself.

   :::image type="content" source="media/availability-zones-overview/zonal-single.svg" alt-text="Diagram of a zonal resource deployed into a single zone." border="false":::

   Zonal deployments don't automatically provide resiliency to availability zone outages. However, zonal resources are isolated from faults in other zones. They can also help you achieve unusually stringent latency or performance requirements. For example, for a chatty workload built using virtual machines, you might choose to deploy multiple virtual machines to the same zone to reduce the latency of the connections between them.

   To make zonal resources resilient to availability zone outages, you need to design an architecture with separate resources in multiple availability zones within the region. Microsoft doesn't manage the process for you. If an outage occurs in an availability zone, you're responsible for failover to another zone.

   :::image type="content" source="media/availability-zones-overview/zonal-multiple.svg" alt-text="Diagram of three zonal resources deployed into three separate zones." border="false":::

When you configure a resource to be zone redundant, or if you use multiple instances of a zonal resource in different availability zones, then your resource is considered to be *zone-resilient*: that is, it's resilient to the outage of a single availability zone. To learn more about how to use zonal deployments and maintain zone resiliency, see [Zonal resources and zone resiliency](./availability-zones-zonal-resource-resiliency.md).

When designing your reliability strategy, make sure that you understand how each service in your workload supports availability zones. For example, some services may have extra requirements to meet for availability zone support, such as certain tiers or SKUs. [Reliability guides](./overview-reliability-guidance.md) contain details of any such requirements.

If a resource isn't configured to use availability zones, either due to the region you use not supporting zones or because of your configuration choices, it's called a *nonzonal* or *regional* deployment. Azure might place nonzonal resources across any zones in the region. You don't choose which resources go into which zones. If any availability zone in the region experiences an outage, nonzonal resources might be in the affected zone and could experience downtime.

## Configuring resources for availability zone support

Each service has its own method for configuring availability zone support, and some services don't require any configuration. To learn about how each service supports availability zones and how to configure that support, see [Azure reliability guides by service](overview-reliability-guidance.md).

## Physical and logical availability zones

Each datacenter is assigned to a physical zone. Physical zones are mapped to logical zones in your Azure subscription, and different subscriptions might have a different mapping order. Azure subscriptions are automatically assigned their mapping at the time the subscription is created. Because of this, the zone mapping for one subscription could be different for other subscriptions.

For example, subscription A may have physical zone 1 mapped to logical zone 2, while subscription B has physical zone 1 mapped to logical zone 3:

:::image type="content" source="media/availability-zones-overview/availability-zones-logical-physical.svg" alt-text="Diagram of logical to physical availability zone mapping." border="false":::

To understand the mapping between logical and physical zones for your subscription, use the [Azure CLI](/cli/azure/install-azure-cli) or [Azure PowerShell](/powershell/azure/what-is-azure-powershell), or the Azure Resource Manager APIs.

# [Azure CLI](#tab/azure-cli)

Use the [az account list-locations](/cli/azure/account#az-account-list-locations) command:

```azurecli
az account list-locations \
    --query "[?availabilityZoneMappings].{availabilityZoneMappings: availabilityZoneMappings, displayName: displayName, name: name}"
```

# [Azure PowerShell](#tab/azure-powershell)

Use the [Invoke-AzRestMethod](/powershell/module/az.accounts/invoke-azrestmethod) cmdlet to retrieve the current subscription's availability zone mappings from the List Locations Resource Manager API:

```azurepowershell
$subscriptionId = (Get-AzContext).Subscription.ID
$response = Invoke-AzRestMethod -Method GET -Path "/subscriptions/$subscriptionId/locations?api-version=2022-12-01"
$locations = ($response.Content | ConvertFrom-Json).value
$locations | Where-Object {$null -ne $_.availabilityZoneMappings} | Select-Object -Property name,displayName,@{name='availabilityZoneMappings';expression={$_.availabilityZoneMappings | convertto-json}} | Format-List
```

# [Azure Resource Manager APIs](#tab/apis)

Use one of these Azure Resource Manager APIs:

- [List Locations API](/rest/api/resources/subscriptions/list-locations): Retrieves the zone mappings for a single Azure subscription.

- [Check Zone Peers API](/rest/api/resources/subscriptions/check-zone-peers): Retrieves the zone mappings for multiple Azure subscriptions.

    > [!NOTE]
    > To use the Check Zone Peers API, the feature `Microsoft.Resources/AvailabilityZonePeering` needs to be enabled on your subscription. For more information about how to enable features, see [Register features in Azure subscription](/azure/azure-resource-manager/management/preview-features).

---

## Availability zones and Azure updates

For each region, Microsoft aims to deploy updates to Azure services within a single availability zone at a time. This approach reduces the impact that updates might have on an active workload, allowing the workload to continue to run in other zones while the update is in process. To take advantage of sequenced zone updates, your workload must be already configured to run across multiple zones. For more information about how Azure deploys updates, see [Advancing safe deployment practices](https://azure.microsoft.com/blog/advancing-safe-deployment-practices/).

<a name='inter-zone-latency'></a>

## Inter-zone networking

Within each region, availability zones are connected through a high-performance network. Microsoft strives to achieve an inter-zone communication with round-trip latency of less than approximately 2 milliseconds. Low latency allows for high-performance communication within a region, and for synchronous replication of data across multiple availability zones. The target latency refers to the latency of the network links. Depending on the communication protocol you use and the network hops required for any specific network flow, the latency you observe might be different.

In most workloads, you can distribute components of your solution across availability zones without a noticeable effect on your performance. If you have a workload with a high degree of sensitivity to inter-zone latency, it's important to test the latency between your selected availability zones with your actual protocols and configuration. To reduce inter-zone traffic, it's possible to use zonal deployments, but optimally, you should use multiple availability zones in your reliability strategy plan. To learn more about how to use zonal deployments and maintain zone resiliency, see [Zonal resources and zone resiliency](./availability-zones-zonal-resource-resiliency.md).

Azure doesn't charge for data transfer between availability zones in the same region, whether you use private or public IP addressing.

## Availability zone architectural guidance

To achieve reliable workloads:

- Production workloads should be configured to use multiple availability zones if the region they are in supports availability zones.
- For mission-critical workloads, you should consider a solution that is *both* multi-region and multi-zone.

For more detailed information on how to use regions and availability zones in a solution architecture, see [Recommendations for using availability zones and regions](/azure/well-architected/resiliency/regions-availability-zones).

## Related content

- [What are Azure regions?](./regions-overview.md)
- [List of Azure regions](regions-list.md)
- [Azure services with availability zones](availability-zones-service-support.md)
- [Recommendations for using availability zones and regions](/azure/well-architected/reliability/regions-availability-zones)
