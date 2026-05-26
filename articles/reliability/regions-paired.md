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
- West US 3 is paired in one direction with East US. East US is bidirectionally paired with West US.

For a list of regions that includes all asymmetrical region pairs, see [Azure region pairs](./regions-list.md).

## Nonpaired regions

Azure continues to expand globally. Many of the newer regions provide multiple [availability zones](./availability-zones-overview.md) for higher resiliency and don't have a region pair.

Many Azure services support geo-replication and geo-redundancy between any arbitrary set of regions, and don't rely on region pairs. It's important to understand how multi-region support works for the particular services you use. For details about each service, see [Azure service reliability guides](./overview-reliability-guidance.md).

For a list of regions that includes all nonpaired regions, see [Azure region pairs](./regions-list.md).

### Choosing a secondary region for compliance-driven scenarios

When you choose a secondary region for geo-redundancy or disaster recovery, consider data residency and regulatory requirements alongside your reliability goals. Because most Azure services support replication to a region of your choice rather than only to a paired region, you can select a secondary region that meets your compliance requirements even when that region isn't within a Microsoft-defined pair. If no in-boundary secondary region is available, plan in-region redundancy and backup-based recovery instead, and adjust your recovery objectives to reflect the longer recovery time.
 
When you use your own encryption keys, also called customer-managed keys, with replication or backup to a region of your choice, consider how encryption keys are managed and distributed. By using Azure Key Vault Managed HSM you can deploy keys across selected regions, align key availability with the regions where data is replicated, and improve availability during regional disruptions.  

When backups or replicated data are stored outside of the primary compliance boundary, ensure that data is encrypted and that key placement is carefully planned. If encryption keys are stored only in the primary region, they might not be accessible during a regional outage. Conversely, storing keys in secondary regions can improve availability but must be evaluated against compliance and security requirements. 

## Related content

- [What are Azure regions?](./regions-overview.md)
- [List of Azure regions](regions-list.md)
- [Reliability guidance](./reliability-guidance-overview.md)
