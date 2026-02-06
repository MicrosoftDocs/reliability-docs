---
title: Reliability in Azure Data Explorer
description: Learn how to make Azure Data Explorer resilient to a variety of potential outages and problems, including transient faults, availability zone outages, and region-wide outages.
author: glynnniall
ms.author: glynnniall
ms.topic: reliability-article
ms.custom: subject-reliability
ms.service: azure-data-explorer
ms.date: 02/06/2026
---

# Reliability in Azure Data Explorer

Azure Data Explorer (ADX) is a big data analytics service that enables you to ingest, store, and query large volumes of data with low latency. It is commonly used for log analytics, telemetry, and time-series workloads that require fast querying over very large datasets.

This article describes how to make Azure Data Explorer resilient to a variety of potential outages and problems, including transient faults, availability zone outages, and region-wide outages.

## Production deployment recommendations

To improve the reliability of Azure Data Explorer in production environments, consider the following recommendations:

- **Enable availability zone support where available.**  
  Azure Data Explorer supports availability zones. When availability zone support is enabled, compute nodes are distributed across multiple availability zones and data is stored using zone-redundant storage. This configuration improves resilience to availability zone failures.

- **Plan for reduced capacity during availability zone failures.**  
  When availability zone support is enabled, a zone outage results in the temporary loss of the compute nodes in the affected zone. This reduces the total available capacity of the cluster. If your workload can’t tolerate reduced capacity during a zone outage, deploy additional nodes to ensure sufficient headroom.

- **Design explicitly for regional failures.**  
  Azure Data Explorer clusters are deployed into a single Azure region. If that region becomes unavailable, the cluster and its data are unavailable. To mitigate region-wide failures, you must design and operate custom multi-region solutions.

## Reliability architecture overview

Azure Data Explorer has a clear separation between compute and storage, which is central to its reliability model.

The **compute layer** consists of cluster nodes. These nodes are Microsoft-managed virtual machines that handle data ingestion and query processing.

The **storage layer** is built on Azure Storage and is managed by the service. Storage is independent of the compute layer and persists data separately from the cluster nodes.

From a logical perspective, you deploy clusters, which contain databases, which in turn contain tables. This abstraction is sufficient to understand the reliability characteristics of the service without going into low-level implementation details.

<!-- DIAGRAM CALLOUT -->
<!-- Include a high-level reliability architecture diagram showing:
     - An Azure Data Explorer cluster
     - Multiple compute nodes
     - A separate Azure Storage layer
     - Clear separation between compute and storage -->

## Resilience to transient faults

Azure Data Explorer doesn’t provide service-specific guidance for handling transient faults beyond standard Azure platform behaviour.

Transient faults can occur due to brief network interruptions or temporary unavailability of compute resources. You should design client applications to handle these failures by retrying failed requests using appropriate retry logic. No Azure Data Explorer–specific retry mechanisms are currently documented.

## Resilience to availability zone failures

Azure Data Explorer supports a **zone-redundant deployment model**.

When availability zone support is enabled:
- Compute resources (cluster nodes) are distributed across multiple availability zones.
- Data is stored using Azure Storage zone-redundant storage (ZRS), which synchronously replicates data across availability zones.

Microsoft manages the distribution of resources across availability zones and handles detection and response to availability zone failures.

<!-- DIAGRAM CALLOUT -->
<!-- Include an availability zone diagram showing:
     - Three availability zones in a region
     - Azure Data Explorer cluster nodes distributed across zones
     - Zone-redundant storage spanning all zones with replicated data -->

### Requirements

- **Region support:**  
  Availability zone support is available only in Azure regions that support availability zones. It’s currently unclear whether Azure Data Explorer supports all availability zone–capable regions or only a subset. This requires confirmation from the product group.

### Considerations

- **Cost:**  
  Enabling availability zone support incurs additional costs. Based on current understanding, this is primarily driven by the use of zone-redundant storage, which is billed at a higher rate than locally redundant storage. This should be confirmed with the product group.

- **Zone selection:**  
  Customers can choose which availability zones to use for compute resources. Storage zone placement is managed by Microsoft. This behaviour should be validated with the product group.

### Capacity planning and management

When an availability zone becomes unavailable, the nodes in that zone are temporarily unavailable, reducing overall cluster capacity.

If your workload can’t tolerate this reduction, you should overprovision capacity so that sufficient resources remain available during a zone outage. For more information, see [Manage capacity by over-provisioning](/azure/reliability/concept-redundancy-replication-backup#manage-capacity-with-over-provisioning).

### Behavior when all zones are healthy

- **Traffic routing between zones:**  
  During normal operation, Azure Data Explorer uses compute nodes across all available zones for ingestion and query processing. Work is distributed across nodes regardless of their availability zone. This behaviour aligns with other Azure compute services and should be confirmed with the product group.

- **Data replication between zones:**  
  Data is synchronously replicated across availability zones using Azure Storage zone-redundant storage. This provides a high level of data consistency and minimises the risk of data loss during a zone failure.

### Behavior during a zone failure

- **Detection and response:**  
  Microsoft detects availability zone failures and manages the response for Azure Data Explorer.

- **Notification:**  
  You can use Azure Service Health to receive notifications about availability zone outages. Azure Data Explorer doesn’t expose per-node health information through Azure Resource Health.

- **Active requests:**  
  Active requests that rely on compute or storage resources in the failed zone might be terminated and should be retried by the client. This behaviour should be confirmed with the product group.

- **Expected data loss:**  
  No data loss is expected during an availability zone outage because data is synchronously replicated across zones.

- **Expected downtime:**  
  A brief service interruption might occur while traffic is redirected to healthy availability zones.

- **Traffic rerouting:**  
  After a zone failure, Azure Data Explorer routes new requests to compute and storage resources in the remaining healthy zones.

### Zone recovery

When the failed availability zone recovers, Microsoft recreates the cluster nodes in that zone and restores normal traffic distribution across all zones. No customer action is required.

### Test for zone failures

Availability zone failover and recovery for Azure Data Explorer are fully managed by Microsoft. You don’t need to initiate or validate availability zone failure processes.

## Resilience to region-wide failures

Azure Data Explorer is a **single-region service**. Clusters are deployed into a single Azure region, and if that region becomes unavailable, the cluster and its data are unavailable.

To minimise the business impact of a regional outage, you can deploy Azure Data Explorer clusters in multiple regions and implement custom data replication and failover strategies. Existing business continuity and disaster recovery documentation covers these scenarios.

Azure paired regions can be used where appropriate, but they aren’t mandatory. Some regions aren’t paired, and regulatory or geopolitical considerations might make paired regions unsuitable.

### Custom multi-region solutions for resiliency

If you need regional resiliency, deploy independent Azure Data Explorer clusters in multiple regions. You’re responsible for coordinating data replication, traffic routing, and failover between regions.

This approach works in both paired and non-paired regions and provides flexibility for customers with strict regulatory or availability requirements.

## Backup and recovery

Azure Data Explorer doesn’t provide a native backup capability. This design aligns with its role as an analytics service, where data is typically retained in upstream systems such as data lakes and re-ingested into Azure Data Explorer as needed.

> For most solutions, you shouldn’t rely exclusively on backups. Instead, use the other capabilities described in this guide to support your resiliency requirements. However, backups protect against some risks that other approaches don’t. For more information, see [Designing resilient applications for Azure](/azure/reliability).

## Resilience to service maintenance

No customer actions are required to maintain reliability during service maintenance. Maintenance activities for Azure Data Explorer are managed by Microsoft.

## Service-level agreement

Azure Data Explorer provides a service-level agreement (SLA). To qualify for the SLA, you must meet the documented deployment and configuration requirements. For details, see the official Azure Data Explorer SLA documentation.

## Related content

- [Reliability in Azure](/azure/reliability)
- [Azure Data Explorer overview](/azure/data-explorer/data-explorer-overview)

[!INCLUDE [security-reminder](../../includes/security-reminder.md)]