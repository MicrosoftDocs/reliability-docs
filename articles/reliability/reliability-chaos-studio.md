---
title: Reliability in Azure Chaos Studio
description: Learn how to improve reliability in Azure Chaos Studio by using availability zones and zone redundancy. Understand disaster recovery and zone outage experiences.
author: glynnniall
ms.author: pnp
ms.topic: reliability-article
ms.custom: subject-reliability, references_regions
ms.service: azure-chaos-studio
ms.date: 04/14/2026
---

# Reliability in Azure Chaos Studio

This article describes reliability support in [Azure Chaos Studio](/azure/chaos-studio/chaos-studio-overview). It covers how to configure availability zones and what to expect during a zone-wide outage. For a more detailed overview of reliability in Azure, see [Azure reliability](/azure/architecture/framework/resiliency/overview).

## Availability zone support

[!INCLUDE [Availability zone description](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)]

Azure Chaos Studio supports zone redundancy as the default configuration within a region. Chaos Studio resources are automatically duplicated or distributed across different zones.

### Prerequisites

The following regions support availability zones for Chaos Studio:

| Americas         | Europe               | Asia Pacific   |
|------------------|----------------------|----------------|
| Brazil South     | France Central       | Australia East |
| Canada Central   | Germany West Central | East Asia      |
| Central US       | Italy North          | Japan East     |
| East US          | North Europe         | Southeast Asia |
| East US 2        | Sweden Central       | UAE North      |
| South Central US | UK South             |                |
| West US 2        | West Europe          |                |
| West US 3        |                      |                |

For detailed information on the regional availability model for Azure Chaos Studio see [Regional availability of Azure Chaos Studio](/azure/chaos-studio/chaos-studio-region-availability).

### Zone down experience

In the event of a zone-wide outage, you should anticipate a brief degradation in performance and availability as the service transitions to a functioning zone. This interruption does not depend on the restoration of the affected zone, as Microsoft-managed services mitigate zone losses by using capacity from alternative zones. In the event of an availability zone outage, it's possible that a chaos experiment could encounter errors or disruptions, but crucial experiment metadata, historical data, and specific details should remain accessible, and the service should not experience a complete outage.

## Cross-region disaster recovery and business continuity

Chaos Studio is a single-region service and doesn't support service-enabled cross-region failover.

## Next steps

- [Create and run your first experiment](/azure/chaos-studio/chaos-studio-quickstart-azure-portal).
- [Learn more about chaos engineering](/azure/chaos-studio/chaos-studio-chaos-engineering-overview).
- [Reliability in Azure](/azure/reliability/overview)
