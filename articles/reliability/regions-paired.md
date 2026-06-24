---
title: Azure region pairs and nonpaired regions
description: Learn about Azure region pairs and regions without a pair.
author: glynnniall
ms.service: azure
ms.subservice: azure-reliability
ms.topic: concept-article
ms.date: 03/19/2025
ms.update-cycle: 1095-days
ms.author: pnp
ms.custom: references_regions, subject-reliability
---

# Azure region pairs and nonpaired regions

This article describes how Azure uses region pairs and nonpaired regions.

Azure regions are independent of each other. However, Microsoft associates some Azure regions with another region, where both regions are usually in the same geography. Together, the regions form a *region pair*. A small number of Azure services use these region pairs to support geo-replication and geo-redundancy. The pairs also support some aspects of disaster recovery in the unlikely event that a region experiences a catastrophic and unrecoverable failure.

However, many regions aren't paired, and instead use availability zones as their primary means of redundancy. In addition, many Azure services support geo-redundancy whether regions are paired or not.

You can design a highly resilient solution whether you use paired regions, nonpaired regions, or a combination.

## Paired regions

Some Azure services use paired regions to build their multiregion geo-replication and geo-redundancy strategy. For example, [Azure geo-redundant storage](/azure/storage/common/storage-redundancy#geo-redundant-storage) (GRS) can automatically replicate data to a paired region.

If you're in a region that's paired, using its pair as a secondary region provides several benefits:

- **Region recovery sequence**. In the unlikely event of a geography-wide outage, one region in every region pair is prioritized for recovery. Components that are deployed across paired regions use one of the regions as the prioritized region for recovery.
- **Sequential updating**. Azure strives to stagger any planned system updates across region pairs. This approach minimizes the impact of bugs or logical failures in the rare event of a faulty update, and it prevents downtime to solutions that are designed to use paired regions together for resiliency.
- **Data residency**. To meet data residency requirements, almost all regions reside within the same geography as their pair. To learn about the exceptions, see the [list of Azure regions](./regions-list.md).

> [!IMPORTANT]
> Deploying resources to a region in a pair doesn't automatically make them more resilient, nor does it provide automatic high availability, disaster recovery capabilities, or failover. Develop your own high availability and disaster recovery plans, regardless of whether you use paired regions or not.
>
> Even if you configure service features to use region pairs, don't rely on Microsoft-managed failover between those pairs as your primary disaster recovery approach. For example, Microsoft-managed failover of GRS-enabled storage accounts is only performed in catastrophic situations and after repeated failed recovery attempts.

You're not limited to using services within a single region or within your region's pair. Although an Azure service might rely upon a specific regional pair for some of its reliability capabilities, you can host your services in any region that satisfies your business needs. For example, an Azure solution can use Azure Storage in the Canada Central region with GRS storage to replicate data to the paired region, Canada East, while using Azure compute resources located in East US, and Azure OpenAI resources located in West US.

To see a list of regions that includes all region pairs, see [List of Azure regions](./regions-list.md).

### Asymmetrically paired regions

Most region pairs are *symmetrical*, which means that each region is bidirectionally paired with another region. For example, West US is paired with East US, and East US is paired with West US.

*Asymmetrical region pairs* involve regions that aren't bidirectionally paired. The following list includes public asymmetrical region pairs:

- Brazil South is paired with South Central US, which is outside of the Brazil geography. South Central US isn't paired with Brazil South.
- West India is paired with South India, but South India is paired with Central India.
- India South Central is paired with Central India, but Central India is paired with South India.
- West US 3 is paired in one direction with East US. East US is bidirectionally paired with West US.

For a list of regions that includes all asymmetrical region pairs, see [Azure region pairs](./regions-list.md).

## Azure region pairs list

The following tables list Azure regions and their corresponding paired region, organized by geography. Regions without a pair show **N/A** in the paired region column.

> [!NOTE]
> The :::image type="content" source="media/icon-region-restricted.svg" alt-text="Icon that shows that access to this region is restricted to support specific customer scenarios, such as disaster recovery within a specific geographic area." border="false"::: icon indicates a restricted-access region. To request access, see [Azure region access request process](/troubleshoot/azure/general/region-access-request-process#reserved-access-regions).

#### [All](#tab/all)

| Region | Paired region |
|--------|---------------|
| Australia Central | :::image type="content" source="media/icon-region-restricted.svg" alt-text="Icon that shows that access to this region is restricted to support specific customer scenarios, such as disaster recovery within a specific geographic area." border="false"::: Australia Central 2 |
| :::image type="content" source="media/icon-region-restricted.svg" alt-text="Icon that shows that access to this region is restricted to support specific customer scenarios, such as disaster recovery within a specific geographic area." border="false"::: Australia Central 2 | Australia Central |
| Australia East | Australia Southeast |
| Australia Southeast | Australia East |
| Austria East | N/A |
| Belgium Central | N/A |
| Brazil South | South Central US |
| :::image type="content" source="media/icon-region-restricted.svg" alt-text="Icon that shows that access to this region is restricted to support specific customer scenarios, such as disaster recovery within a specific geographic area." border="false"::: Brazil Southeast | Brazil South |
| Canada Central | Canada East |
| Canada East | Canada Central |
| Central India | South India |
| Central US | East US 2 |
| Chile Central | N/A |
| Denmark East | N/A |
| East Asia | Southeast Asia |
| East US | West US |
| East US 2 | Central US |
| France Central | :::image type="content" source="media/icon-region-restricted.svg" alt-text="Icon that shows that access to this region is restricted to support specific customer scenarios, such as disaster recovery within a specific geographic area." border="false"::: France South |
| :::image type="content" source="media/icon-region-restricted.svg" alt-text="Icon that shows that access to this region is restricted to support specific customer scenarios, such as disaster recovery within a specific geographic area." border="false"::: France South | France Central |
| :::image type="content" source="media/icon-region-restricted.svg" alt-text="Icon that shows that access to this region is restricted to support specific customer scenarios, such as disaster recovery within a specific geographic area." border="false"::: Germany North | Germany West Central |
| Germany West Central | :::image type="content" source="media/icon-region-restricted.svg" alt-text="Icon that shows that access to this region is restricted to support specific customer scenarios, such as disaster recovery within a specific geographic area." border="false"::: Germany North |
| Indonesia Central | N/A |
| Israel Central | N/A |
| Italy North | N/A |
| Japan East | Japan West |
| Japan West | Japan East |
| Korea Central | Korea South |
| Korea South | Korea Central |
| Malaysia West | N/A |
| Mexico Central | N/A |
| New Zealand North | N/A |
| North Central US | South Central US |
| North Europe | West Europe |
| Norway East | :::image type="content" source="media/icon-region-restricted.svg" alt-text="Icon that shows that access to this region is restricted to support specific customer scenarios, such as disaster recovery within a specific geographic area." border="false"::: Norway West |
| :::image type="content" source="media/icon-region-restricted.svg" alt-text="Icon that shows that access to this region is restricted to support specific customer scenarios, such as disaster recovery within a specific geographic area." border="false"::: Norway West | Norway East |
| Poland Central | N/A |
| Qatar Central | N/A |
| South Africa North | :::image type="content" source="media/icon-region-restricted.svg" alt-text="Icon that shows that access to this region is restricted to support specific customer scenarios, such as disaster recovery within a specific geographic area." border="false"::: South Africa West |
| :::image type="content" source="media/icon-region-restricted.svg" alt-text="Icon that shows that access to this region is restricted to support specific customer scenarios, such as disaster recovery within a specific geographic area." border="false"::: South Africa West | South Africa North |
| South Central US | North Central US |
| South India | Central India |
| Southeast Asia | East Asia |
| Spain Central | N/A |
| Sweden Central | :::image type="content" source="media/icon-region-restricted.svg" alt-text="Icon that shows that access to this region is restricted to support specific customer scenarios, such as disaster recovery within a specific geographic area." border="false"::: Sweden South |
| Switzerland North | :::image type="content" source="media/icon-region-restricted.svg" alt-text="Icon that shows that access to this region is restricted to support specific customer scenarios, such as disaster recovery within a specific geographic area." border="false"::: Switzerland West |
| :::image type="content" source="media/icon-region-restricted.svg" alt-text="Icon that shows that access to this region is restricted to support specific customer scenarios, such as disaster recovery within a specific geographic area." border="false"::: Switzerland West | Switzerland North |
| :::image type="content" source="media/icon-region-restricted.svg" alt-text="Icon that shows that access to this region is restricted to support specific customer scenarios, such as disaster recovery within a specific geographic area." border="false"::: UAE Central | UAE North |
| UAE North | :::image type="content" source="media/icon-region-restricted.svg" alt-text="Icon that shows that access to this region is restricted to support specific customer scenarios, such as disaster recovery within a specific geographic area." border="false"::: UAE Central |
| UK South | UK West |
| UK West | UK South |
| West Central US | West US 2 |
| West Europe | North Europe |
| West India | South India |
| West US | East US |
| West US 2 | West Central US |
| West US 3 | East US |

#### [Americas](#tab/americas)

| Region | Paired region |
|--------|---------------|
| Brazil South | South Central US |
| :::image type="content" source="media/icon-region-restricted.svg" alt-text="Icon that shows that access to this region is restricted to support specific customer scenarios, such as disaster recovery within a specific geographic area." border="false"::: Brazil Southeast | Brazil South |
| Canada Central | Canada East |
| Canada East | Canada Central |
| Central US | East US 2 |
| Chile Central | N/A |
| East US | West US |
| East US 2 | Central US |
| Mexico Central | N/A |
| North Central US | South Central US |
| South Central US | North Central US |
| West Central US | West US 2 |
| West US | East US |
| West US 2 | West Central US |
| West US 3 | East US |

#### [Europe](#tab/europe)

| Region | Paired region |
|--------|---------------|
| Austria East | N/A |
| Belgium Central | N/A |
| Denmark East | N/A |
| France Central | :::image type="content" source="media/icon-region-restricted.svg" alt-text="Icon that shows that access to this region is restricted to support specific customer scenarios, such as disaster recovery within a specific geographic area." border="false"::: France South |
| :::image type="content" source="media/icon-region-restricted.svg" alt-text="Icon that shows that access to this region is restricted to support specific customer scenarios, such as disaster recovery within a specific geographic area." border="false"::: France South | France Central |
| :::image type="content" source="media/icon-region-restricted.svg" alt-text="Icon that shows that access to this region is restricted to support specific customer scenarios, such as disaster recovery within a specific geographic area." border="false"::: Germany North | Germany West Central |
| Germany West Central | :::image type="content" source="media/icon-region-restricted.svg" alt-text="Icon that shows that access to this region is restricted to support specific customer scenarios, such as disaster recovery within a specific geographic area." border="false"::: Germany North |
| Italy North | N/A |
| North Europe | West Europe |
| Norway East | :::image type="content" source="media/icon-region-restricted.svg" alt-text="Icon that shows that access to this region is restricted to support specific customer scenarios, such as disaster recovery within a specific geographic area." border="false"::: Norway West |
| :::image type="content" source="media/icon-region-restricted.svg" alt-text="Icon that shows that access to this region is restricted to support specific customer scenarios, such as disaster recovery within a specific geographic area." border="false"::: Norway West | Norway East |
| Poland Central | N/A |
| Spain Central | N/A |
| Sweden Central | :::image type="content" source="media/icon-region-restricted.svg" alt-text="Icon that shows that access to this region is restricted to support specific customer scenarios, such as disaster recovery within a specific geographic area." border="false"::: Sweden South |
| Switzerland North | :::image type="content" source="media/icon-region-restricted.svg" alt-text="Icon that shows that access to this region is restricted to support specific customer scenarios, such as disaster recovery within a specific geographic area." border="false"::: Switzerland West |
| :::image type="content" source="media/icon-region-restricted.svg" alt-text="Icon that shows that access to this region is restricted to support specific customer scenarios, such as disaster recovery within a specific geographic area." border="false"::: Switzerland West | Switzerland North |
| UK South | UK West |
| UK West | UK South |
| West Europe | North Europe |

#### [Middle East](#tab/middle-east)

| Region | Paired region |
|--------|---------------|
| Israel Central | N/A |
| Qatar Central | N/A |
| :::image type="content" source="media/icon-region-restricted.svg" alt-text="Icon that shows that access to this region is restricted to support specific customer scenarios, such as disaster recovery within a specific geographic area." border="false"::: UAE Central | UAE North |
| UAE North | :::image type="content" source="media/icon-region-restricted.svg" alt-text="Icon that shows that access to this region is restricted to support specific customer scenarios, such as disaster recovery within a specific geographic area." border="false"::: UAE Central |

#### [Africa](#tab/africa)

| Region | Paired region |
|--------|---------------|
| South Africa North | :::image type="content" source="media/icon-region-restricted.svg" alt-text="Icon that shows that access to this region is restricted to support specific customer scenarios, such as disaster recovery within a specific geographic area." border="false"::: South Africa West |
| :::image type="content" source="media/icon-region-restricted.svg" alt-text="Icon that shows that access to this region is restricted to support specific customer scenarios, such as disaster recovery within a specific geographic area." border="false"::: South Africa West | South Africa North |

#### [Asia Pacific](#tab/asia-pacific)

| Region | Paired region |
|--------|---------------|
| Australia Central | :::image type="content" source="media/icon-region-restricted.svg" alt-text="Icon that shows that access to this region is restricted to support specific customer scenarios, such as disaster recovery within a specific geographic area." border="false"::: Australia Central 2 |
| :::image type="content" source="media/icon-region-restricted.svg" alt-text="Icon that shows that access to this region is restricted to support specific customer scenarios, such as disaster recovery within a specific geographic area." border="false"::: Australia Central 2 | Australia Central |
| Australia East | Australia Southeast |
| Australia Southeast | Australia East |
| Central India | South India |
| East Asia | Southeast Asia |
| Indonesia Central | N/A |
| Japan East | Japan West |
| Japan West | Japan East |
| Korea Central | Korea South |
| Korea South | Korea Central |
| Malaysia West | N/A |
| New Zealand North | N/A |
| South India | Central India |
| Southeast Asia | East Asia |
| West India | South India |

---

## Nonpaired regions

Azure continues to expand globally. Many of the newer regions provide multiple [availability zones](./availability-zones-overview.md) for higher resiliency and don't have a region pair.

Many Azure services support geo-replication and geo-redundancy between any arbitrary set of regions and don't rely on region pairs. Others might require that you design and implement your own multiregion approaches. For a list of service multiregion capabilities, including those that work between nonpaired regions, see [Azure services that support multiple regions](./regions-multiregion-support.md). For detailed information about each service, see its [reliability guide](./overview-reliability-guidance.md).

For a list of regions that includes all nonpaired regions, see [Azure region pairs](./regions-list.md).

## Related content

- [What are Azure regions?](./regions-overview.md)
- [List of Azure regions](regions-list.md)
- [Reliability guidance](./reliability-guidance-overview.md)
