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

## Resilience to availability zone failures

[!INCLUDE [Resilience to availability zone failures](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)]

<!-- TODO -->

### Requirements

- **Region support:** <!-- TODO -->

<!-- TODO -->

### Considerations

<!-- TODO -->

### Cost

<!-- TODO -->

### Configure availability zone support

<!-- TODO -->

### Capacity planning and management

<!-- TODO -->

### Behavior when all zones are healthy

This section describes what to expect when you configure an Azure Cosmos DB account for zone redundancy, and all zones are operational.

- **Cross-zone operation:** <!-- TODO -->

- **Cross-zone data replication:** <!-- TODO -->

### Behavior during a zone failure

This section describes what to expect when you configure an Azure Cosmos DB account for zone redundancy, and there's an outage in one of the zones.

- **Detection and response:** <!-- TODO -->

- **Notification:** <!-- TODO -->

- **Active requests:** <!-- TODO -->

- **Expected data loss:** <!-- TODO -->

- **Expected downtime:** <!-- TODO -->

- **Redistribution:** <!-- TODO -->

### Zone recovery

<!-- TODO -->

### Test for zone failures

<!-- TODO -->

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
