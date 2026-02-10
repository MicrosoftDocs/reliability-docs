---
title: Reliability in [service-name]
description: Learn how to make [service-name] resilient to a variety of potential outages and problems, including transient faults, availability zone outages, region outages, and service maintenance, and learn about backup and restore. # Adjust to cover the main sections in the document, and remove any items that don't apply.
author: glynnniall # Required; your GitHub user alias, with correct capitalization.
ms.author: glynnniall # Required; Microsoft alias of author; optional team alias.
ms.topic: reliability-article # Required
ms.custom: subject-reliability # Required - add references_regions only if specific Azure regions are mentioned.
ms.service: learn # Required replace with your service meta tag. For taxonomies see https://review.learn.microsoft.com/help/platform/metadata-taxonomies/msservice?branch=main
ms.date: 02/04/2026 # Required; mm/dd/yyyy format.
---

<!-- For guidance on how to fill out this template, see https://learn.microsoft.com/help/patterns/level4/article-reliability -->

# Reliability in [service-name]

<!-- Intro paragraph -->

<!-- [!INCLUDE [Shared responsibility](includes/reliability-shared-responsibility-include.md)] -->

<!-- This article describes... -->

## Production deployment recommendations

## Reliability architecture overview

## Resilience to transient faults

<!-- [!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)] -->

## Resilience to availability zone failures

<!-- [!INCLUDE [Resilience to availability zone failures](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)] -->

### Requirements

- **Region support:**

### Considerations

### Instance distribution across zones

### Cost

### Configure availability zone support

### Capacity planning and management

### Behavior when all zones are healthy

This section describes what to expect when you configure a [resource name] for [availability zone support type], and all zones are operational.

- **Traffic routing between zones:**

- **Data replication between zones:**

### Behavior during a zone failure

This section describes what to expect when you configure a [resource name] for [availability zone support type], and there's an outage in one of the zones.

- **Detection and response:**

- **Notification:**

- **Active requests:**

- **Expected data loss:**

- **Expected downtime:**

- **Traffic rerouting:**

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

- **Traffic routing between regions:**

- **Data replication between regions:**

#### Behavior during a region failure

This section describes what to expect when you configure a [resource name] for [multi-region support type], and there's an outage in one of the replica regions.

- **Detection and response:**

- **Notification:**

- **Active requests:**

- **Expected data loss:**

- **Expected downtime:**

- **Traffic rerouting:**

#### Region recovery

#### Test for region failures

### Custom multi-region solutions for resiliency

## Backup and restore

## Resilience to service maintenance

## Service-level agreement

<!-- [!INCLUDE [Service-level agreement](includes/reliability-service-level-agreement-include.md)] -->

## Related content
