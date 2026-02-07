---
title: Reliability in Azure Monitor Logs
description: Learn about reliability for Azure Monitor Logs (Log Analytics workspaces), including availability zones, workspace replication, multi-region strategies, and data export for continuity.
author: austinmccollum
ms.author: austinmc
ms.topic: reliability-article
ms.custom: subject-reliability, references_regions
ms.service: azure-monitor
ms.subservice: logs
ms.date: 02/01/2026
#Customer intent: As an engineer responsible for business continuity or an data architect, I want to understand the details of how Log Analytics works from a reliability perspective and plan disaster recovery strategies in alignment with the exact processes that Azure services follow during different kinds of situations.
---

# Reliability in Azure Monitor Logs

[Azure Monitor Logs](/azure/azure-monitor/logs/data-platform-logs) is a centralized software as a service (SaaS) platform for collecting, analyzing, and acting on system generated data by Azure and non-Azure resources and applications.

[!INCLUDE [Shared responsibility description](includes/reliability-shared-responsibility-include.md)]

This article describes how to make Azure Monitor Logs resilient to various potential outages and problems, including transient faults, availability zone outages, and region outages.

## Production deployment recommendations

Azure Monitor Logs offers several features that you can use individually or in combination to help enhance workspaces resilience to various types of issues.

| Capability    | Description    | Relative cost |
|----|----|----|
| [Availability Zones](/azure/azure-monitor/logs/availability-zones)    | Protect your Log Analytics workspace from datacenter failures through redundancy between zones in your region.    | Least expensive |
| [Data Export](/azure/azure-monitor/logs/logs-data-export)    | Back up ingested logs against entire region failures by continuously exporting logs to a geo-redundant storage account.    | Somewhat expensive |
| [Workspace Replication](/azure/azure-monitor/logs/workspace-replication)    | Protect your Log Analytics workspace against entire region failures in Log Analytics or downstream services through redundancy between regions.    | Most expensive |

:::image type="content" source="media/reliability-log-analytics/resiliency-features.png" lightbox="media/reliability-log-analytics/resiliency-features.png" alt-text="An image showing the resiliency features of Azure Monitor Log Analytics based on the previous table.":::

Certain Log Analytics features aren't compatible with all reliability features. For example, Auxiliary tables aren't supported with workspace replication. For more information, review the [Reliability best practices for Log Analytics](/azure/azure-monitor/logs/best-practices-logs#reliability) for details on feature compatibility.

## Reliability architecture overview

A healthy Log Analytics workspace relies on the following features:
- Ingestion pipeline (multitenant, throttling, and retry aware) reachable through public or private link endpoints
- Metadata and control plane that manages table schemas, table plans and retention periods (Analytics, Basic, Auxiliary), DCR mappings, replication settings, and export rules
- Redundant compute nodes that process ingestion, queries, exports, alerts, and replication

Redundancy and replication model
- Within a region, the service durably stores data with multiple replicas (and across zones when zone redundant) before it acknowledges ingestion
- Across regions (workspace replication feature), the service asynchronously propagates new log records from primary to secondary, and preexisting data isn't backfilled
- Data export asynchronously copies selected tables to Azure Storage or Event Hubs for external retention, governed by export rules

Reliability levers you control
- Region selection, zone redundancy, and dedicated cluster linkage if needed
- Enable workspace replication and choose a compliant secondary location in the same region group (see supported regions list)
- Data export scope (tables and destinations), storage redundancy, and immutability policies
- Retention tier configuration (interactive versus archive) that balances cost and recovery or query latency (archive requires search or restore for deep query scenarios)
- Daily cap and alerting on ingestion delay to prevent noise or throttling during incidents

Increase availability by combining zone redundancy with workspace replication or data export rules. Make alert rules and automation portable to the secondary region so your replicated data is usable during a regional outage.

## Resilience to transient faults 

[!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)]

For example, the ingestion pipeline, which sends collected data to the Log Analytics workspace, validates that each log record is successfully processed before it removes the record from the pipeline. If the ingestion pipeline isn't available or is throttling requests, the agents that send the data start to buffer locally, retrying the logs for many hours or exponentially backing off. However, a custom application that sends ingestion requests or queries needs to implement its own retry logic.

To validate your custom application is handling transient errors properly, monitor these metrics: 
- ingestion latency
- failed ingestion count
- export failures
- query failure rate 

Initiate an investigation or a switchover for sustained latency longer than five minutes.

## Resilience to availability zone failures

Azure Monitor Logs offers zone-redundant **data resilience** (stored data) for all regions that support availability zones. Most regions that support availability zones require your workspace to be deployed in a d[dedicated cluster](/azure/azure-monitor/logs/logs-dedicated-clusters), however, some regions support it with the default workspace configuration of a shared cluster. Moving to a dedicated cluster in a region that supports availability zones protects data ingested after the move, not historical data.

Only some regions offer zone-redundant **service resilience** (ingestion and query continuity). 

If an incident affects one zone, Microsoft manages failover to a different availability zone in the region automatically. You don't need to take any action because switching between zones is seamless.

### Requirements

- **Region support** - Consult the availability zones [supported regions](/azure/azure-monitor/logs/availability-zones#supported-regions) for current coverage.
- **Dedicated cluster** - Some regions require a [dedicated cluster](/azure/azure-monitor/logs/logs-dedicated-clusters) for certain features of zone redundancy. If a dedicated cluster is required for zone-redundant **data resilience** or **service resilience** in a region,  workspace must be linked to a dedicated cluster with a minimum commitment tier ≥ 100 GB/day.
- **Historical data** - When an existing workspace is moved to a dedicated cluster, historical data isn't retroactively protected. Only new data ingested after the move benefits from new zone redundancy features.

### Considerations 

- Distinguish data resilience (replicated storage) from service resilience (ingestion/query continuity). Some regions currently offer only data resilience.
- In some regions, Event Hubs ingestion into the workspace isn't resilient to a zone outage (see footnotes in availability zones article). Evaluate alternate ingestion paths for critical data.
- Dedicated clusters only protect new data. When a workspace is moved to a dedicated cluster, previously ingested data remains in the shared cluster. Under normal operations everything is available, but during a zone incident only the new data in the dedicated cluster might be accessible.

### Instance distribution across zones

Replica sets for table shards are placed across up to three availability zones where zone data resilience is supported. Placement and rebalancing are platform-managed Transient skew can occur after scale-in or maintenance but rebalancing restores even spread. Customers can't pin or force zone placement.

### Cost

Zone redundancy has no separate pricing meter for the workspace. Dedicated clusters have commitment tier charges independent of zone redundancy.

### Configure availability zone support 

Actions:
1. [Create a workspace](/azure/azure-monitor/logs/quick-create-workspace) in an availability zone-capable region. Zone redundancy is auto applied where supported on shared clusters.
1. If the region requires a dedicated cluster for zone support, deploy the dedicated cluster, and then [link the workspace](/azure/azure-monitor/logs/logs-dedicated-clusters#link-a-workspace-to-a-cluster).

### Capacity planning and management 

Monitor ingestion latency and daily cap usage. Maintain a buffer between peak daily volume and the daily cap so retry bursts during a zone incident don't cause throttling. For dedicated clusters, select a commitment tier with headroom for temporary ingestion spikes.

### Behavior when all zones are healthy 

- **Traffic routing:** Ingestion and queries use active/active routing to nodes in any zone, optimizing for locality and load.
- **Data replication:** Writes are committed to multiple zone-isolated replicas before acknowledgment (reducing exposure to single-zone loss).



### Behavior during a zone failure

- **Detection and response:** The platform detects a zone failure and reallocates ingestion and query compute to surviving zones automatically.
- **Notification:** Use Service Health and workspace health metrics. Alert on sustained ingestion latency or elevated query failure rate.
- **Active requests:** In-flight ingestion batches or queries to failed zone nodes are retried.
- **Data loss:** Not expected for acknowledged records. Unacknowledged batches are retried.
- **Downtime:** No expected data unavailability. Temporary latency increases or throttling might occur during rebalancing.
- **Traffic rerouting:** Load balancers steer to healthy zone nodes so no endpoint change is required.


### Zone recovery

Returning zone capacity is phased back in. The platform rebalances shards to restore uniform distribution. No user action is required.

### Test for zone failures 

You can't directly simulate a zone failure. Use [Azure Chaos Studio](/azure/chaos-studio/chaos-studio-overview) to test upstream components (for example, network constraints) and run ingestion/query load tests with reduced capacity to validate operational service level objectives (SLO) and alerting.


## Resilience to region-wide failures

Azure Monitor Logs provides **workspace replication** for cross-region resilience. Replication creates a secondary workspace instance in a supported secondary region in the same region group and asynchronously copies **new** logs (after enablement) plus schema and configuration changes. A switchover must be manually triggered by the customer through an API. Switchover reroutes ingestion and queries to the secondary workspace. A customer-initiated switch back must be triggered to restore the primary. Preexisting data before configuring replication isn't backfilled. 

Key characteristics of workspace replication:
- Active/passive architecture
- Asynchronous data replication
- Configuration (schemas, custom tables, retention settings) propagated automatically
- The secondary workspace isn't independently visible or manageable in the portal - access remains through the same workspace resource endpoint.
- Save costs by choosing which DCRs participate in replication

[**Data Export**](/azure/azure-monitor/logs/logs-data-export) is a solution that can provide dual write in the same region until you enable replication. It can also provide geo-redundant backups when the Azure storage account target for data export is configured with geo-redundancy.

### Requirements 

- Region support for workspace replication requires selecting a secondary region within the same region group. Some region combinations can't point to each other directly. See [supported regions list](/azure/azure-monitor/logs/workspace-replication#supported-regions) for permissible primary/secondary pairs and region groups.
- Replication is configured with the Log Analytics management [**workspace** REST API](/azure/azure-monitor/logs/workspace-replication#enable-workspace-replication).
- Switchover and switch back actions are manual and require [proper role permissions](/azure/azure-monitor/logs/workspace-replication#permissions-required).
- If linked to a dedicated cluster, enable replication at cluster first.
- Associate **data collection rules** (DCRs) with the workspace data collection endpoint (DCE) after enabling replication to replicate those streams during switchover.

### Considerations

- There's no automatic failover. Switchover is a manual decision.
- Configure runbooks to define switchover criteria (for example, region-wide ingestion latency, SLO breach, sustained query failure rate, Service Health advisory). Test regularly.
- Switchover time includes DNS propagation. Some agents or clients with sticky DNS might continue to attempt primary ingestion briefly.
- Purge operations remove records from both primary and secondary. If one is unavailable, the purge fails so plan compliance timelines accordingly.
- Alert rules aren't autoreplicated. Use IaC export/import to maintain identical rule sets in both regions.
- Microsoft Sentinel watchlists and threat intelligence entries only replicate when they're refreshed, so they can take up to 12 days for full parity after initial activation.
- Auxiliary tables aren't replicated. As a best practice, don't enable replication on workspaces that include Auxiliary tables.

### Cost

Replication adds per-GB replicated charges. Disabling replication stops new replication charges but doesn't remove already replicated copies until the feature is fully disabled.


### Configure multi-region support 

For the complete steps to enable workspace replication, see [Enhance resilience by replicating your Log Analytics workspace](/azure/azure-monitor/logs/workspace-replication).
1. Enable replication and specify the secondary region through the API.
1. Validate the provisioning state.
1. Associate DCRs with the workspace DCE.
1. Establish monitoring (replication health, ingestion latency, workspace health metrics) for the active region. Audit the inactive workspace region before a switchover.
1. Document switchover and switchback criteria and automation.

To configure data export for geo-redundant backups, see [Log Analytics workspace data export](/azure/azure-monitor/logs/logs-data-export).

### Capacity planning and management 

Plan for ingestion surges post-switchover (buffered retries). Ensure the daily cap leaves headroom for peak plus backlog. For dedicated clusters, select a commitment tier with headroom for temporary ingestion spikes.

### Behavior when all regions are healthy
 
- **Traffic routing:** All ingestion and queries target the primary region while healthy. The secondary holds a passive replica of data, schemas, and configuration until switchover.
- **Data replication:** Asynchronous batches propagate new logs, schema, and configuration updates to the secondary.

### Behavior during a region failure

- **Detection and response:** Customer or automation triggers switchover when regional outage or severe degradation is confirmed.
- **Notification:** Use Service Health plus custom ingestion and query health alerts. Replication status helps validate secondary freshness.
- **Active requests:** In-flight ingestion to the failed region is retried. After a DNS change, clients send to the secondary.
- **Expected data loss:** Not expected for acknowledged data. Unreplicated in-flight data is buffered (up to documented limits) and retried.
- **Expected downtime:** Limited to the switchover window (DNS and pipeline activation). Queries resume against the secondary afterward.
- **Traffic rerouting:** DNS update redirects ingestion endpoints. The platform routes queries internally to the active region.

### Region recovery

Switch back is manual. After the primary is stable and synchronized, trigger a switch back. Validate no backlog remains and replication resumes from the restored primary. See [Switch back to the primary workspace](/azure/azure-monitor/logs/workspace-replication#switch-back-to-your-primary-workspace) for details.

### Testing for region failures 

See [Monitor workspace and service health](/azure/azure-monitor/logs/workspace-replication#monitor-workspace-and-service-health) for guidance. 
Perform DR drills: simulate criteria (for example, inject latency via test alerts) then execute scripted switchover and switch-back in a nonproduction or low-risk window; measure RTO (switchover completion) and RPO (max unreplicated interval). Use exported datasets to validate sample record parity.

### Alternative multi-region approaches 

If workspace replication isn't enabled or a table type isn't supported, consider these alternatives for multi-region resilience:
- **Dual write:** Configure sources (diagnostic settings, agents, DCR-based ingestion) to send to two independent workspaces in different regions. This approach provides near real-time analytics in both at roughly double ingestion cost and with higher configuration drift risk.
- **Export and rehydrate:** Use continuous data export to Azure storage (GRS or GZRS). During a disaster, deploy a new workspace and selectively ingest recent critical data while querying long-term history directly in Azure storage using Azure Data Explorer.

## Backups

Azure Monitor Logs doesn't provide a traditional point-in-time backup and restore mechanism. Durability and recovery rely on intra-region replication (and zone redundancy), workspace replication, and data export for an external long-term copy. For compliance and tamper protection, use data export to Azure storage with immutability policies. For rapid analytics recovery, rely on replication Purge operations propagate to replicated copies and fail if one instance is unavailable. Export immutability provides an extra safeguard.

## Reliability during service maintenance

Azure Monitor performs rolling maintenance across workspace replicas and zones. Zone redundancy minimizes customer visible impact. Some configuration changes such as enabling replication, linking clusters, and adding large schemas are long running operations. Schedule them outside critical monitoring windows. Monitor ingestion latency and workspace health metrics during and after scheduled maintenance events and related Service Health notifications.

## Service level agreement

[!INCLUDE [Service-level agreement](includes/reliability-service-level-agreement-include.md)]

Azure Monitor Logs is unique among online services because it's the recommended platform to provide essential data and insights needed to submit a claim to Microsoft support for an SLA issue.

The **Log Analytics (Query Availability SLA)** defines availability expectations for data in a Log Analytics workspace. The **Azure Monitor SLA** defines availability expectations for Alert Rules and Action Groups. 

For the formal SLA, see [Service Level Agreements for Online Services https://aka.ms/CSLA](https://aka.ms/CSLA).

## Related content

- [Reliability best practices for Azure Monitor Logs](/azure/azure-monitor/logs/best-practices-logs#reliability)
- [Availability zones in Azure Monitor](/azure/azure-monitor/logs/availability-zones)
- [Enhance resilience by replicating your Log Analytics workspace](/azure/azure-monitor/logs/workspace-replication)
- [Log Analytics workspace data export](/azure/azure-monitor/logs/logs-data-export)
- [Data collection rules](/azure/azure-monitor/data-collection/data-collection-rule-overview)
- [Dedicated clusters](/azure/azure-monitor/logs/logs-dedicated-clusters)