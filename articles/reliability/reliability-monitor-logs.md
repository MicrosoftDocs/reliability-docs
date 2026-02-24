---
title: Reliability in Azure Monitor Logs
description: Learn about reliability for Azure Monitor Logs (Log Analytics workspaces), including availability zones, workspace replication, multi-region strategies, and data export for continuity.
author: austinmccollum
ms.author: austinmc
ms.topic: reliability-article
ms.custom: subject-reliability, references_regions
ms.service: azure-monitor
ms.subservice: logs
ms.date: 02/16/2026
---

# Reliability in Azure Monitor Logs

[Azure Monitor Logs](/azure/azure-monitor/logs/data-platform-logs) is a centralized software as a service (SaaS) platform for collecting, analyzing, and acting on system generated data by Azure and non-Azure resources and applications.

[!INCLUDE [Shared responsibility description](includes/reliability-shared-responsibility-include.md)]

This article describes how to make Azure Monitor Logs resilient to various potential outages and problems, including transient faults, availability zone outages, and region outages.

## Production deployment recommendations

Azure Monitor Logs offers several features that you can use individually or in combination to help enhance workspaces resilience to various types of issues.

| Capability    | Description    | Relative cost |
|----|----|----|
| [Availability zones](/azure/azure-monitor/logs/availability-zones)    | Protect your Log Analytics workspace from datacenter failures through redundancy between zones in your region.    | Least expensive |
| [Data export](/azure/azure-monitor/logs/logs-data-export)    | Back up ingested logs against entire region failures by continuously exporting logs to a geo-redundant storage account.    | Somewhat expensive |
| [Workspace replication](/azure/azure-monitor/logs/workspace-replication)    | Protect your Log Analytics workspace against entire region failures in Log Analytics or downstream services through redundancy between regions.    | Most expensive |

:::image type="content" source="media/reliability-monitor-logs/resiliency-features.png" lightbox="media/reliability-monitor-logs/resiliency-features.png" alt-text="An image showing the resiliency features of Azure Monitor Log Analytics based on the previous table." border="false":::

Certain Log Analytics features aren't compatible with all reliability features. For example, Auxiliary tables aren't supported with workspace replication. For more information, review the [Reliability best practices for Log Analytics](/azure/azure-monitor/logs/best-practices-logs#reliability) for details on feature compatibility.

## Reliability architecture overview

[!INCLUDE [Introduction to reliability architecture overview section](includes/reliability-architecture-overview-introduction-include.md)]

### Logical architecture

When you use Azure Monitor Logs, you deploy a *Log Analytics workspace*, which is a data store into which you can collect any type of log data.

Log Analytics workspaces are associated with a *cluster*, which provides the computational resources for the workspace and other capabilities. Most workspaces run on a *shared cluster*, which Microsoft deploys and manages, and which are shared among multiple customers. You can optionally deploy a [dedicated cluster](/azure/azure-monitor/logs/logs-dedicated-clusters), which provides dedicated resources for your workspace. There are some differences in reliability features between shared and dedicated clusters, which are described throughout this article.

Azure Monitor Logs has two distinct data paths, each with its own reliability characteristics:

- **Ingestion** is the path through which log data flows into a Log Analytics workspace. The ingestion pipeline acknowledges receipt of data only after durably writing it to storage. The ingestion path includes several components:
  
  - *Data sources* generate telemetry. Some sources, like Azure platform logs sent through diagnostic settings, submit data directly to the ingestion pipeline through *diagnostic settings*. Other sources require an *agent* to collect and forward data. Your applications can also use the [Logs Ingestion API](/azure/azure-monitor/logs/logs-ingestion-api-overview) to send logs directly to Azure Monitor.
  
  - *Data collection rules (DCRs)* define which data to collect, how to transform it, and where to send it. DCRs route data to the ingestion pipeline through a *data collection endpoint (DCE)*.  

- **Query** is the path through which you retrieve and analyze data that's already stored in the workspace. Log queries use Kusto Query Language (KQL) and run against the workspace's stored data.

Ingestion and query are handled as distinct service operations, so a disruption to one doesn't necessarily affect the other. For example, during a degradation in the ingestion pipeline, you might still be able to query previously ingested data. Similarly, a query-side issue doesn't prevent new data from being ingested and stored.

For more information about the core components of Azure Monitor Logs, see [Azure Monitor Logs overview](/azure/azure-monitor/logs/data-platform-logs).

### Physical architecture

Internally, Azure Monitor Logs uses multiple *replicas* of both the computational and storage components of log data. These replicas are managed by Microsoft and you don't need to configure or manage them.

You're responsible for deploying and managing agents, and for the reliability of the compute resources they run on.

## Resilience to transient faults

[!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)]

In Azure Monitor Logs, transient faults are primarily a concern for ingestion:

- **Transient faults within the ingestion pipeline:** The ingestion pipeline that sends collected data to the Log Analytics workspace verifies that each log record is successfully processed before removing it from the pipeline. If the pipeline is unavailable or throttles requests, diagnostic settings and Azure Monitor Agents begin buffering it locally and retrying for many hours, using exponential backoff. In contrast, a custom application that submits ingestion requests or queries must implement its own retry logic.

    > [!WARNING]
    > **Note to PG:** Is it safe to say that Azure resources (diagnostic settings) will also retry during transient faults in the same way that the Azure Monitor Agent does?

- **Transient faults that affect connectivity to the pipeline:** Agents buffer data locally and retry delivery when the ingestion endpoint is temporarily unavailable, which makes the ingestion path more resilient to transient faults.

- **Transient faults that affect custom application logging:** To validate your custom application is handling transient errors properly, monitor these metrics:
    - Ingestion latency
    - Failed ingestion count
    - Export failures
    - Query failure rate

    If you see sustained ingestion latency over five minutes, it might indicate a more significant and non-transient problem. Review the guidance in the other sections of this document to understand how to be resilient to other failure types.

If transient faults occur during queries or when performing other operations, clients are responsible for retrying.

## Resilience to availability zone failures

[!INCLUDE [Resilience to availability zone failures](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)]

Azure Monitor Logs offers two types of availability zone support, depending on the region and cluster type of your workspace:

- **Data resilience** provides zone redundancy for your log data by replicating it across multiple availability zones.

    Data resilience is available in all regions that support availability zones. Most regions that support availability zones require your workspace to be deployed in a [dedicated cluster](/azure/azure-monitor/logs/logs-dedicated-clusters). However, some regions support it with the default workspace configuration of a shared cluster. Moving to a dedicated cluster in a region that supports availability zones protects data ingested after the move, not historical data.

- **Service resilience** provide zone redundancy for ingestion and query continuity during an availability zone outage.

    Only some regions offer service resilience.

If an incident affects one zone, Microsoft manages failover to a different availability zone in the region automatically. You don't need to take any action because switching between zones is seamless.

### Requirements

- **Region support:** Consult the availability zones [supported regions](/azure/azure-monitor/logs/availability-zones#supported-regions) for the list of regions that support data resilence and service resilience, and whether data resilience is supported in shared or dedicated clusters.

- **Dedicated cluster:** Some regions require a [dedicated cluster](/azure/azure-monitor/logs/logs-dedicated-clusters) for data resilience.

### Considerations

- **Event Hubs ingestion:** In some regions, Event Hubs ingestion into the workspace isn't resilient to a zone outage. For a list of these regions, see the footnotes in the [supported regions list](/azure/azure-monitor/logs/availability-zones#supported-regions). Evaluate alternate ingestion paths for critical data.

- **Move to dedicated clusters:** Dedicated clusters only protect new data. When a workspace is moved to a dedicated cluster, previously ingested data remains in the shared cluster. Under normal operations everything is available, but during a zone outage only the new data in the dedicated cluster might be accessible.

### Cost

Zone redundancy doesn't affect pricing. Workspaces on shared clusters are charged at the standard workspace price, and workspaces on dedicated clusters must meet the cluster's commitment tier. Neither pricing model changes when zone redundancy is enabled. For more information about pricing, see [Azure Monitor pricing](https://azure.microsoft.com/pricing/details/monitor/).

### Configure availability zone support

For workspaces using shared clusters, when you [create a workspace](/azure/azure-monitor/logs/quick-create-workspace) in a region that supports availability zones for Azure Monitor Logs then zone redundancy is enabled automatically.

If the region requires a dedicated cluster for zone support, deploy the dedicated cluster, and then [link the workspace to the dedicated cluster](/azure/azure-monitor/logs/logs-dedicated-clusters#link-a-workspace-to-a-cluster).

### Capacity planning and management

During a zone failure, your workspace might experience greater load because of ingestion and query retries. If your solution is sensitive to delays in log ingestion or queries, even during and after a zone failure, consider overprovisioning the capacity of any daily caps you configure on your workspace. For dedicated clusters, select a commitment tier with headroom for temporary ingestion spikes. For more information, see [Manage capacity by overprovisioning](concept-redundancy-replication-backup.md#manage-capacity-with-over-provisioning).

### Behavior when all zones are healthy

This section describes what to expect when a Log Analytics workspace is zone-redundant, and all zones are operational.

- **Cross-zone operation:** For clusters that have service resilience, ingestion and queries can use infrastructure in any zone. The service automatically distributes work across zones, optimizing for locality and load.

- **Cross-zone data replication:** For clusters that have data resilience, The ingestion pipeline commits all writes to multiple replicas in separate zones before acknowledging.

### Behavior during a zone failure

This section describes what to expect when a Log Analytics workspace is zone-redundant, and there's an outage in one of the zones.

- **Detection and response:** The platform detects a zone failure and responds automatically. For clusters that have service resilience, the platform reallocates ingestion and query compute to surviving zones. For clusters that have data resilience, the platform switches to use replicas in healthy zones. You don't need to initiate a zone failover.

- **Notification:** [!INCLUDE [Availability zone down notification partial bullet (Azure Service Health only)](./includes/reliability-availability-zone-down-notification-service-partial-include.md)]

    You can also monitor the workspace's health metrics, and look for sustained ingestion latency or elevated query failure rate. You can configure alerts on these metrics.

- **Active requests:** During a zone outage, any active requests might fail.

    Any active queries might be terminated. Clients can retry the queries.

    For clusters that don't have service resilience, ingestion pipelines might stop processing data that hasn't been committed. Agents or client applications can retry when the zone is healthy.

- **Expected data loss:** For clusters that have data resilience, no data loss is expected for data that has been fully committed. Unacknowledged batches need to be retried by the agent or client.

- **Expected downtime:** For clusters that have service resilience, no downtime is expected. Temporary latency increases or throttling might occur as the service rebalances infrastructure across zones.

- **Rerouting:** For clusters that have service resilience, internal load balancers within the Azure Monitor Logs service redirect traffic to healthy zone nodes. No endpoint change is required.

### Zone recovery

When an availability zone recovers, Azure Monitor Logs automatically reintegrates the zone into the active service topology. The recovered zone begins processing the ingestion pipeline and queries alongside the other zones. Data that had been replicated to surviving zones during the outage remains intact, and normal synchronous replication resumes across all zones. You don't need to take action for zone recovery and reintegration.

### Test for zone failures

Azure manages traffic routing, failover, and zone recovery for zone failures, so you don't need to validate availability zone failure processes or provide further input.

## Resilience to region-wide failures

Azure Monitor Logs provides multi-region support through workspace replication.

### Workspace replication

Workspace replication creates a secondary workspace in a supported secondary region, and then asynchronously replicates new logs as well as schema and configuration changes. Preexisting data before configuring replication isn't backfilled.

You don't see the secondary workspace as a separate workspace that you can manage. Access remains through the same workspace resource endpoint.

If the primary region fails, you must trigger a switchover by using an API. Switchover reroutes ingestion and queries to the secondary workspace. There's no automatic failover. Switchover is a manual decision. A customer-initiated switch back must be triggered to restore the primary.

:::image type="content" source="media/reliability-monitor-logs/workspace-replication-ingestion-flows.png" alt-text="Diagram that shows ingestion flows during normal and switchover modes." lightbox="media/reliability-monitor-logs/workspace-replication-ingestion-flows.png" border="false":::

This section summarizes important aspects of workspace replication. Review the full documentation to understand exactly how it works. For more information, see [Enhance resilience by replicating your Log Analytics workspace across regions](/azure/azure-monitor/logs/workspace-replication).

> [!NOTE]
> Azure Monitor Logs workspace replication uses the term *switchover* because it best represents the process of manually switching to the secondary workspace. You might also see the term *failover* used to describe the general process.

#### Requirements

- **Region support:** Workspace replication requires selecting a secondary region within the same *region group*. Some region combinations can't point to each other directly. See [supported regions list](/azure/azure-monitor/logs/workspace-replication#supported-regions) for permissible primary/secondary pairs and region groups. If your workspace's region isn't in the list, it doesn't currently support replication.

- **Dedicated clusters:** If your workspace is linked to a dedicated cluster, enable replication on the cluster first.

#### Considerations

- **Alert rules:** Alert rules aren't automatically replicated between the primary and secondary. To maintain identical rule sets in both regions, use infrastructure as code to export alert rules from one workspace and import them the other.
    
    > [!WARNING]
    > **Note to PG:** The workspace replication doc seems to say that (a) the alert rules typically *do* work in the secondary, and (b) it implies there's no way to configure alerts just on the secondary workspace since it's not individually addressable. Can you confirm whether the preceding statement is accurate, given it seems to conflict with the other doc?

- **Manual switchover and switchback:** You're responsible for deciding when to switch over and switch back, and for triggering the switchover and switchback actions.

    To prepare for a region failure, you should:
    - Establish monitoring (replication health, ingestion latency, workspace health metrics) for the active region. Audit the inactive workspace region before a switchover.
    - Consider whether to create runbooks to define switchover criteria, which might form part of your disaster recovery planning. For example, you might monitor region-wide ingestion latency, SLO breaches, sustained query failure rates, or Azure Service Health advisories. Test your runbooks regularly.
    - Document your switchover and switchback criteria.
    - Verify that any users or service principals that trigger switchover have the necessary [role permissions](/azure/azure-monitor/logs/workspace-replication#permissions-required).

- **Auxiliary tables:** Auxiliary tables aren't replicated. As a best practice, don't enable replication on workspaces that include Auxiliary tables.

- **Microsoft Sentinel:** Microsoft Sentinel watchlists and threat intelligence entries only replicate when they're refreshed, so they can take up to 12 days to achieve full parity after initial activation.

- **Purge operations:** Purge operations remove records from both primary and secondary. If either of the workspaces is unavailable, the purge fails. This means you need to plan your compliance timelines accordingly.

For more considerations, see [Deployment considerations](/azure/azure-monitor/logs/workspace-replication#deployment-considerations).

#### Cost

When you enable workspace replication, you're charged for the replication of all data you ingest to your workspace. For more information about pricing, see [Azure Monitor pricing](https://azure.microsoft.com/pricing/details/monitor/).

#### Configure multi-region support

- **Enable workspace replication:** The general steps required to enable workspace replication are:

    1. [Enable replication and specify the secondary region.](/azure/azure-monitor/logs/workspace-replication#enable-and-disable-workspace-replication).
    1. [Validate the provisioning state.](/azure/azure-monitor/logs/workspace-replication#check-workspace-provisioning-state)
    1. [Associate data collection rules (DCRs)](/azure/azure-monitor/logs/workspace-replication#associate-data-collection-rules-with-the-workspace-data-collection-endpoint) with the workspace data collection endpoint (DCE) after enabling replication to replicate those streams during switchover.

- **Disable workspace replication:** For detailed steps to disable workspace replication, see [Enable and disable workspace replication](/azure/azure-monitor/logs/workspace-replication#enable-and-disable-workspace-replication).

    Disabling replication stops new logs from being replicated, but doesn't remove already replicated copies until the feature is fully disabled.

    > [!WARNING]
    > **Note to PG:** Please clarify what "fully disabled" means.

#### Capacity planning and management

During a region failure or another switchover or switchback event, your workspace might experience greater load because of ingestion and query retries. If your solution is sensitive to delays in log ingestion or queries, even during and after a region failure, consider overprovisioning the capacity of any daily caps you configure on your workspace. For dedicated clusters, select a commitment tier with headroom for temporary ingestion spikes. For more information, see [Manage capacity by overprovisioning](concept-redundancy-replication-backup.md#manage-capacity-with-over-provisioning).

#### Behavior when all regions are healthy

This section describes what to expect when you configure a Log Analytics workspace for workspace replication, and all regions are operational.

- **Cross-region operation:** All ingestion and queries target the primary region's workspace while healthy. The secondary region's workspace holds a passive replica of data, schemas, and configuration until switchover.

- **Cross-zone data replication:** New logs, schema, and configuration updates are replicated asynchronously in batches to the secondary. Because it's asynchronous, this replication process doesn't affect ingestion latency.

    > [!WARNING]
    > **Note to PG:** can we give an idea of the replication lag (even if it's vague, like "a few minutes")?

    You can save on replication costs by choosing which DCRs participate in replication.

#### Behavior during a region failure

This section describes what to expect when you configure a Log Analytics workspace for workspace replication, and there's an outage in one of the regions.

- **Detection and response:** You're responsible for deciding when to switch the secondary workspace to become the new primary. Microsoft doesn't make this decision or initiate the process for you, even if there's a region outage. Switchover and switchbakc are manual actions.

    For information about how to decide when to switch over, see [When should I switch over?](/azure/azure-monitor/logs/workspace-replication#when-should-i-switch-over), and for information about how to monitor the health of your workspace when making the decision to switch over, see [Monitor workspace performance using queries](/azure/azure-monitor/logs/workspace-replication#monitor-workspace-performance-using-queries).

    For details about how to promote a secondary region to the new primary, see [Switch over to your secondary workspace](/azure/azure-monitor/logs/workspace-replication#switch-over-to-your-secondary-workspace).

- **Notifications:** [!INCLUDE [Region down notification partial bullet (Service Health only)](./includes/reliability-region-down-notification-service-partial-include.md)]

    For information about how to monitor the health of your workspace when making the decision to switch over, see [TMonitor workspace performance using queriesO](/azure/azure-monitor/logs/workspace-replication#monitor-workspace-performance-using-queries).
    
    Replication status helps validate secondary freshness.

    > [!WARNING]
    > **Note to PG:** Please expand on the preceding statement.

- **Active requests:** Any active ingestion to the failed region might fail, and any queries in progress in the failed region also might fail.

    After the switchover process changes the DNS entries for the workspace, the secondary workspace is available for ingestion and query. Microsoft's agents automatically retry failed log ingestion attempts. Applications using the log ingestion API should also retry.

- **Expected data loss:** Any logs or other changes that haven't yet been replicated is unavailable in the secondary region.

    After you switch over to the secondary region, if the primary region can't process incoming log data, Azure Monitor buffers the data in the secondary region for up to 11 days. During the first four days, Azure Monitor automatically reattempts to replicate the data periodically.

    The total amount of data loss is sometimes called the recovery point objective (RPO).

- **Expected downtime:** Thew switchover process involves the secondary workspace pipeline being activated, and an update to the workspace's DNS records.

    Although the DNS record change happens quickly, DNS propagation can take longer. If agents or other clients don't honor the DNS time-to-live (TTL), they might continue to attempt ingestion against the primary workspace. Ensure clients don't cache DNS entries for longer than their TTL.

    Queries resume against the secondary after the switchover completes.

    The total time it takes before the secondary workspace is available is sometimes called the recovery time objective (RTO).

    > [!WARNING]
    > **Note to PG:** Can we give an approximate idea of how long this takes (understanding DNS propagation will be an unknown factor)?

- **Redistribution:** The switchover process updates the DNS records for the ingestion endpoints to point to the secondary region. The platform routes queries internally to the active region.

    While the workspace is in the switched-over state, log replication happens back to primary region using asynchronous replication.

#### Region recovery

Switchback is manual and you're responsible for deciding when to switch back. After the primary is stable and synchronized, you can trigger a switch back. Validate no backlog of logs remains in the secondary, and that replication successfully resumes from the restored primary.

For information about how to decide when to switch back, see [When should I switch back?](/azure/azure-monitor/logs/workspace-replication#when-should-i-switch-back), and for detailed switchback steps, see [Switch back to the primary workspace](/azure/azure-monitor/logs/workspace-replication#switch-back-to-your-primary-workspace).

#### Test for region failures

You can trigger a switchover and switchback at any time, including to perform tests or disaster recovery drills.

If you perform drills, we recommend you follow these best practices:

> [!div class="checklist"]
> - Use a nonproduction environment, or if you test in production, run the process in a low-risk time window.
> - If possible, simulate the criteria that triggers your region failover criteria based on your own policies. This approach enables you to test your detection and automation as well as the switchover and switchback process.
> - Measure the RTO (switchover completion) and RPO (maximum unreplicated interval). Use exported datasets to validate sample record parity between the primary and secondary workspaces.

### Custom multi-region solutions for resiliency

If workspace replication isn't available for your region, or you must use a table type that isn't supported by workspace replication, consider these alternative approaches for multi-region resilience:

- **Dual write:** Configure sources, such as diagnostic settings, agents, and DCR-based ingestion, to send logs to two independent workspaces in different regions.

    This approach provides near real-time analytics in both regions. However, the ingestion cost approximately doubles, and it introduces the risk of configuration drift between the workspaces.

- **Export and rehydrate:** If your region is paired with another Azure region, you can use [data export](/azure/azure-monitor/logs/logs-data-export) to continuously export your logs to Azure Blob Storage. Configure the storage account to use one of the geo-redundant storage (GRS) types. Azure Storage automatically and asynchronously replicates the exported logs to the paired region.

    During a disaster, deploy a new workspace and selectively ingest recent critical data. You can also manually query long-term history directly from Blob Storage by using tools like [Azure Storage Explorer](/azure/storage/storage-explorer/vs-azure-tools-storage-manage-with-storage-explorer).

## Backup and restore

Azure Monitor Logs doesn't provide a traditional point-in-time backup or restore capability. Instead, durability and recovery rely on intra-region replication (including zone redundancy), optional workspace replication, and data export to maintain an external copy. For rapid analytics recovery after a failure, rely on platform replication rather than restore operations.

You can enable [data export](/azure/azure-monitor/logs/logs-data-export), which automatically exports copies of your logs to Azure Storage. If your Azure region is paired, you can consider enabling geo-redundant storage (GRS)to replicate the log data to the paired region.

To meet stringent compliance requirements and for tamper protection, consider using [immutability policies](/azure/storage/blobs/immutable-storage-overview) to prevent log data deletion from the storage account.

## Resilience to service maintenance

[!INCLUDE [Service maintenance](includes/reliability-maintenance-include.md)]

To minimize the effect of maintenance, Azure Monitor performs rolling maintenance across workspace replicas and availability zones.

If your solution is sensitive to ingestion and query latency, monitor ingestion latency and workspace health metrics during and after scheduled maintenance events.

Some long-running configuration changes can be resource-intensive, such as enabling replication, linking clusters, and adding large schemas. If you need to make these changes, we recommend you avoid scheduling them when platform maintenance is also scheduled. Use Azure Service Health planned maintenance to understand when maintenance is scheduled.

## Service level agreement

[!INCLUDE [Service-level agreement](includes/reliability-service-level-agreement-include.md)]

Azure Monitor Logs is unique among online services because it's the recommended platform to provide essential data and insights needed to submit a claim to Microsoft support for an SLA issue.

There are two SLAs that cover the service:

- The **Log Analytics (Query Availability SLA)** defines availability expectations for querying the data from a Log Analytics workspace.

- The **Azure Monitor SLA** defines availability expectations for alert rules and action groups, including for analyzing telemetry signals and for notification delivery.

## Related content

- [Reliability best practices for Azure Monitor Logs](/azure/azure-monitor/logs/best-practices-logs#reliability)
- [Availability zones in Azure Monitor](/azure/azure-monitor/logs/availability-zones)
- [Enhance resilience by replicating your Log Analytics workspace](/azure/azure-monitor/logs/workspace-replication)
- [Log Analytics workspace data export](/azure/azure-monitor/logs/logs-data-export)
- [Data collection rules](/azure/azure-monitor/data-collection/data-collection-rule-overview)
- [Dedicated clusters](/azure/azure-monitor/logs/logs-dedicated-clusters)
