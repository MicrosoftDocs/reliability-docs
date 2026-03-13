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

## Reliability architecture overview

## Resilience to transient faults

[!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)]

## Resilience to availability zone failures

[!INCLUDE [Resilience to availability zone failures](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)]

### Requirements

- **Region support:**

### Considerations

### Instance distribution across zones

### Cost

### Configure availability zone support

### Capacity planning and management

### Behavior when all zones are healthy

This section describes what to expect when you configure a [resource name] for [availability zone support type], and all zones are operational.

- **Cross-zone operation:**

- **Cross-zone data replication:**

### Behavior during a zone failure

This section describes what to expect when you configure a [resource name] for [availability zone support type], and there's an outage in one of the zones.

- **Detection and response:**

- **Notification:**

- **Active requests:**

- **Expected data loss:**

- **Expected downtime:**

- **Redistribution:**

### Zone recovery

### Test for zone failures

## Resilience to region-wide failures

### Multi-region support type 1

#### Requirements

- **Region support:**

#### Considerations

#### Cost

#### Configure multi-region support

#### Capacity planning and management

#### Behavior when all regions are healthy

This section describes what to expect when you configure a [resource name] for [multi-region support type], and all regions are operational.

- **Cross-region operation:**

- **Cross-region data replication:**

#### Behavior during a region failure

This section describes what to expect when you configure a [resource name] for [multi-region support type], and there's an outage in one of the replica regions.

- **Detection and response:**

- **Notification:**

- **Active requests:**

- **Expected data loss:**

- **Expected downtime:**

- **Redistribution:**

#### Region recovery

#### Test for region failures

### Custom multi-region solutions for resiliency

## Backup and restore

## Resilience to service maintenance

## Service-level agreement

[!INCLUDE [Service-level agreement](includes/reliability-service-level-agreement-include.md)]

## Related content
