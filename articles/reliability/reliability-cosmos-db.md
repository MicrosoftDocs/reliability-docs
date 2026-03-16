---
title: Reliability in Azure Cosmos DB
description: Learn how to make Azure Cosmos DB resilient to a variety of potential outages and problems, including transient faults, availability zone outages, region outages, and service maintenance, and learn about backup and restore.
author: seesharprun
ms.author: sidandrews
ms.topic: reliability-article
ms.custom: subject-reliability
ms.service: azure-cosmos-db
ms.date: 03/13/2026
---

# Reliability in Azure Cosmos DB

[!INCLUDE [Shared responsibility](includes/reliability-shared-responsibility-include.md)]

## Production deployment recommendations

The Azure Well-Architected Framework provides recommendations across reliability, security, cost, operations, and performance. To understand how these areas influence each other and contribute to a reliable Azure Cosmos DB solution, see [Architecture best practices for Azure Cosmos DB](/azure/well-architected/service-guides/cosmos-db).

## Reliability architecture overview

<!-- TODO -->

## Resilience to transient faults

[!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)]

<!-- TODO -->

[Design resilient applications with Azure Cosmos DB SDKs](/azure/cosmos-db/conceptual-resilient-sdk-applications)

## Resilience to availability zone failures

[!INCLUDE [Resilience to availability zone failures](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)]

Azure Cosmos DB supports *zone redundancy*. When you enable zone redundancy, Azure distributes the replicas of your data across multiple availability zones, providing resiliency to datacenter problems and outages.

We recommend using zone redundancy in regions where it's supported, especially for single-region accounts. Because availability zones are physically separate and provide distinct power source, network, and cooling, the availability SLAs for Azure Cosmos DB are higher for zone-redundant accounts than accounts that don't use availability zones.

### Requirements

**Region support:** You can enable zone redundancy in Azure regions that supports availability zones. To see if your region supports availability zones, see [the list of supported regions](./regions-list.md).

### Considerations

- A single-region account with Availability Zones can maintain read-write availability when an outage affects only one availability zone. However, if multiple availability zones or the entire region is impacted, single-region accounts lose read and write access until service is restored.

- TODO interactions between multi-region and AZ support

### Cost

Regions where zone redundancy is enabled are charged at a premium. However, the premium pricing for availability zones is waived for accounts configured with multi-region writes and/or for collections configured with autoscale mode. For more information, see [Azure Cosmos DB pricing](https://azure.microsoft.com/pricing/details/cosmos-db/).

### Configure availability zone support

You can configure availability zones only when you add a new region to an Azure Cosmos DB account.

- **Create a new Azure Cosmos DB account with zone redundancy:** 

- **Add a zone-redundant region to an existing Azure Cosmos DB account:** To enable availability zone support on an existing account, you need to add a new region and enable zone redundancy on that region. TODO

    - [Azure portal](/azure/cosmos-db/how-to-manage-database-account#add-remove-regions-from-your-database-account)
    - [Azure CLI](/azure/cosmos-db/manage-with-cli#add-or-remove-regions)
    - [Azure Resource Manager templates](/azure/cosmos-db/manage-with-templates)

### Capacity planning and management

<!-- TODO -->

### Behavior when all zones are healthy

This section describes what to expect when you configure an Azure Cosmos DB account for zone redundancy, and all zones are operational.

- **Cross-zone operation:** <!-- TODO -->

- **Cross-zone data replication:** <!-- TODO -->

### Behavior during a zone failure

This section describes what to expect when you configure an Azure Cosmos DB account for zone redundancy, and there's an outage in one of the zones.

- **Detection and response:** <!-- TODO -->

[!INCLUDE [Availability zone down notification (Service Health and Resource Health)](./includes/reliability-availability-zone-down-notification-service-resource-include.md)]

- **Active requests:** <!-- TODO -->

- **Expected data loss:** <!-- TODO -->

- **Expected downtime:** <!-- TODO -->

- **Redistribution:** <!-- TODO -->

### Zone recovery

When the availability zone recovers, Azure Cosmos DB automatically restores replicas in the availability zone, and reroutes traffic between replicas as normal.

### Test for zone failures

The Azure Cosmos DB platform manages traffic routing, failover, and failback for zone-redundant accounts. Because this feature is fully managed, you don't need to initiate anything or validate availability zone failure processes.

## Resilience to region-wide failures

<!-- TODO -->

### Multi-region read and write

<!-- TODO -->

#### Requirements

- **Region support:** <!-- TODO -->

#### Considerations

<!-- TODO -->

#### Cost

<!-- TODO -->

#### Configure multi-region support

<!-- TODO -->

#### Capacity planning and management

<!-- TODO -->

#### Behavior when all regions are healthy

This section describes what to expect when you configure an Azure Cosmos DB account for multi-region support, and all regions are operational.

- **Cross-region operation:** <!-- TODO -->

- **Cross-region data replication:** <!-- TODO -->

#### Behavior during a region failure

This section describes what to expect when you configure an Azure Cosmos DB account for multi-region support, and there's an outage in one of the replica regions.

- **Detection and response:** <!-- TODO -->

- **Notification:** <!-- TODO -->

- **Active requests:** <!-- TODO -->

- **Expected data loss:** <!-- TODO -->

- **Expected downtime:** <!-- TODO -->

- **Redistribution:** <!-- TODO -->

#### Region recovery

<!-- TODO -->

#### Test for region failures

<!-- TODO -->

### Custom multi-region solutions for resiliency

<!-- TODO -->

## Backup and restore

<!-- TODO -->

## Resilience to service maintenance

<!-- TODO -->

## Service-level agreement

[!INCLUDE [Service-level agreement](includes/reliability-service-level-agreement-include.md)]

<!-- TODO -->

## Related content

<!-- TODO -->
