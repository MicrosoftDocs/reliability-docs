---
title: Reliability in Azure Stream Analytics
description: Learn how to improve reliability in Azure Stream Analytics by using availability zones, geo-redundancy, and operational best practices for streaming data processing.
author: anaharris-ms
ms.author: anaharris
ms.topic: reliability-article
ms.custom: subject-reliability
ms.service: azure-stream-analytics
ms.date: 10/06/2025

#Customer intent: As an engineer responsible for business continuity, I want to understand the details of how Azure Stream Analytics works from a reliability perspective and plan disaster recovery strategies in alignment with the exact processes that Azure services follow during different kinds of situations.
---

# Reliability in Azure Stream Analytics

Azure Stream Analytics is a fully managed real-time analytics service designed to process and analyze streaming data from multiple sources simultaneously. Stream Analytics lets you build complex event processing pipelines with SQL-like queries while providing built-in reliability features to ensure continuous data processing. This article describes reliability support in [Azure Stream Analytics](../stream-analytics/stream-analytics-introduction.md). It covers intra-regional resiliency via [availability zones](#availability-zone-support) and [multi-region disaster recovery options](#multi-region-support).

[!INCLUDE [Shared responsibility description](includes/reliability-shared-responsibility-include.md)]

> [!IMPORTANT]
> When you consider the reliability of Stream Analytics, you also need to consider the reliability of your data sources, including inputs and outputs. Improving the resiliency of Stream Analytics alone might have limited impact if the other components aren't equally resilient. Depending on your resiliency requirements, you might need to make configuration changes across multiple areas.

## Production deployment recommendations

To ensure high reliability in production environments with Azure Stream Analytics:

- Deploy your streaming jobs in regions that support availability zones. 
- Configure your jobs with appropriate compatibility levels (1.2 or higher recommended) to access the latest features and performance improvements. 
- Set your streaming units based on expected throughput with additional capacity for handling peak loads and potential zone failures - typically provision 20-30% above your baseline requirements.
- Implement comprehensive monitoring using Azure Monitor metrics and diagnostic logs to track job health, input/output events, and resource utilization. - 
- Configure alerts for critical metrics like watermark delay, backlogged input events, and runtime errors to detect issues before they impact data processing. Use event ordering policies that align with your business requirements while understanding the trade-offs between data completeness and processing latency.
- For *mission-critical streaming workloads*, consider implementing a multi-region deployment strategy with synchronized job configurations across regions. While Stream Analytics doesn't provide native multi-region replication, you can achieve regional redundancy by deploying identical jobs in multiple regions with appropriate data routing mechanisms.

**Sources:**
- [Introduction to Azure Stream Analytics](../stream-analytics/stream-analytics-introduction.md)
- [Azure Stream Analytics compatibility level](../stream-analytics/stream-analytics-compatibility-level.md)
- [Understand and adjust streaming units](../stream-analytics/stream-analytics-streaming-unit-consumption.md)

> [!NOTE]
> [Azure Stream Analytics on IoT Edge](../stream-analytics/stream-analytics-edge.md) enables you to run jobs on your own infrastructure. When you use Stream Analytics on IoT Edge, you're responsible for configuring it to meet your reliability requirements. Stream Analytics on IoT Edge is outside the scope of this article.

## Reliability architecture overview

TODO

### Logical architecture

Each Stream Analytics *job* consists of:
- *Inputs* that read streaming data.
- A *query* that transforms the data.
- *Outputs* that write results to various destinations.

Stream Analytics maintains job state through regular checkpointing, enabling quick recovery with minimal data reprocessing in case of failures. When processing failures occur, Stream Analytics automatically restarts from the last checkpoint, reprocessing any events that weren't fully processed. This guarantee applies to all built-in functions and user-defined functions within the job, though achieving end-to-end exactly-once delivery depends on your output destination's capabilities.

### Physical architecture

<!-- TODO verify this -->
Azure Stream Analytics operates on a distributed computing architecture where your streaming jobs run across multiple compute nodes for high availability and fault tolerance. The service automatically handles job placement, scaling, and recovery without requiring manual intervention. 

Within a region, Stream Analytics jobs are automatically distributed across fault domains to provide resilience against hardware failures.

The service continuously monitors job health and automatically relocates jobs from failed nodes to healthy ones. This architecture ensures that transient infrastructure issues don't interrupt your streaming data processing.

**Sources:**
- [Stream Analytics job reliability](../stream-analytics/stream-analytics-job-reliability.md)
- [Achieve exactly once delivery with Azure Stream Analytics](../stream-analytics/stream-analytics-exactly-once-delivery.md)
- [Azure Stream Analytics job states](../stream-analytics/job-states.md)

## Transient faults

[!INCLUDE [Transient fault description](includes/reliability-transient-fault-description-include.md)]

Azure Stream Analytics automatically handles many transient faults through built-in retry mechanisms for both input data ingestion and output writing. 

For *input operations*, the service implements exponential backoff strategies when encountering temporary failures from input sources like Event Hubs or IoT Hub, automatically retrying failed read operations without manual intervention.

For *output operations*, Stream Analytics provides configurable retry policies that determine how the service responds to temporary failures when writing to destinations like Azure SQL Database or Cosmos DB. <!-- TODO verify this. Seems like retry policy is only for conversion errors, not transient faults? -->

In addition to the built-in retry mechanisms, you can take the following steps to enhance transient fault handling in your Stream Analytics jobs:

- **Configure your Stream Analytics jobs** with appropriate error handling policies based on your data requirements. The service offers three error policy options: Drop, Retry (default), and Redirect to error output. The Retry policy automatically attempts to process failed records with exponential backoff, suitable for handling transient network issues or temporary service unavailability. For scenarios requiring zero data loss, configure error outputs to capture and separately process records that fail after retry attempts.

- **Monitor your job's transient fault handling** through Azure Monitor diagnostics. Track metrics such as input deserialization errors, output errors, and function errors to understand the frequency and impact of transient issues. Adjust your error policies and timeout settings based on observed patterns. For comprehensive guidance on implementing retry patterns and handling transient faults in your broader streaming solution, see [Handle transient faults](/azure/well-architected/reliability/handle-transient-faults).

**Sources:**
- [Stream Analytics error policy](../stream-analytics/stream-analytics-output-error-policy.md)
- [Monitor Azure Stream Analytics with Azure Monitor diagnostics logs](../stream-analytics/stream-analytics-job-diagnostic-logs.md)
- [Handle transient faults](/azure/well-architected/reliability/handle-transient-faults)

## Availability zone support

[!INCLUDE [AZ support description](includes/reliability-availability-zone-description-include.md)]

Azure Stream Analytics automatically provides zone-redundant deployments in regions that support availability zones. When you create a Stream Analytics job in a zone-enabled region, the service distributes your job's compute resources across multiple availability zones without any additional configuration required. This zone-redundant deployment model ensures that your streaming jobs continue to process data even if an entire availability zone becomes unavailable.

Stream Analytics does not support zonal deployments. You cannot pin a Stream Analytics job to a specific availability zone. The service automatically manages the distribution of job resources across zones to optimize availability and performance. This approach provides the highest level of resilience against zone failures while simplifying deployment and management for customers.

The zone-redundant architecture applies to all Stream Analytics features including query processing, checkpointing, and job management operations. Your job's state and checkpoint data are automatically replicated across zones, ensuring no data loss during zone failures. The service handles all aspects of zone redundancy transparently, requiring no special configuration or operational procedures from customers.

<!-- John: is this still true? I moved this note from their introduction. -->
>[!NOTE]
>Stream Analytics jobs that are integrated with a virtual network don't currently support availability zones.

**Sources:**
- [Stream Analytics job reliability](../stream-analytics/stream-analytics-job-reliability.md)
- [Azure Stream Analytics documentation](../stream-analytics/index.yml)

### Region support

Zone-redundancy for Azure Stream Analytics resources is supported in any region that supports availability zones.

For the complete list of regions that support availability zones, see [Azure regions with availability zones](/regions-list.md).


### Requirements

Azure Stream Analytics automatically provides zone redundancy in supported regions without any specific SKU or tier requirements. All Stream Analytics jobs created in availability zone-enabled regions benefit from zone-redundant deployment at no additional cost. There are no special configuration requirements or prerequisites to enable zone redundancy for your streaming jobs.

<!-- John: is this still true? I moved this note from their introduction. -->
>[!NOTE]
>Stream Analytics jobs that are integrated with a virtual network don't currently support availability zones.

**Sources:**
- [Azure Stream Analytics pricing](https://azure.microsoft.com/pricing/details/stream-analytics/)

### Considerations

While Stream Analytics provides automatic zone redundancy, your input sources and output destinations must also be configured for zone redundancy to achieve end-to-end resilience. Ensure that services such as Event Hubs, IoT Hub, and your chosen output destinations are deployed with zone redundancy enabled. During a zone failure, Stream Analytics continues processing, but the availability of your entire streaming pipeline depends on all components being zone-resilient.


**Sources:**
- [Stream Analytics inputs overview](../stream-analytics/stream-analytics-input-overview.md)
- [Outputs from Azure Stream Analytics](../stream-analytics/stream-analytics-define-outputs.md)

### Cost

Enabling zone redundancy for Azure Stream Analytics doesn't incur additional charges. You pay the same rate for streaming units whether your job runs in a zone-redundant configuration or not. The pricing model remains consistent across all regions, including those with availability zone support.

**Sources:**
- [Azure Stream Analytics pricing](https://azure.microsoft.com/pricing/details/stream-analytics/)

### Configure availability zone support

- **Create a zone-redundant Stream Analytics job.** Zone redundancy is automatically enabled when you create a Stream Analytics job in a supported region. No special configuration is required. For deployment instructions, see [Quickstart: Create a Stream Analytics job by using the Azure portal](../stream-analytics/stream-analytics-quick-create-portal.md).

- **Migrate to zone-redundant deployment.** <!-- John: I imagine you cannot move jobs from non ZR to ZR and you have to recreate them. The AI hallucinated here, so I leave this as a placeholder. -->

- **Disable zone redundancy.** Zone redundancy cannot be disabled for Stream Analytics jobs in regions with availability zone support. If you require a non-zone-redundant deployment, you must create your job in a region that doesn't support availability zones.

**Sources:**
- [Create a Stream Analytics job](../stream-analytics/stream-analytics-quick-create-portal.md)


### Capacity planning and management

When planning capacity for zone-redundant Stream Analytics deployments, provision sufficient streaming units to handle your peak load requirements plus additional overhead for potential zone failures. The service automatically distributes streaming units across available zones, but during a zone failure, the remaining zones must absorb the additional processing load. Plan for at least 30% additional capacity above your typical peak to ensure smooth operations during zone outages.

Configure autoscaling for your Stream Analytics jobs to automatically adjust streaming units based on CPU utilization, SU utilization percentage, or backlogged input events. This ensures your job can scale up quickly when processing load increases due to zone failures. Monitor your job's resource utilization metrics and adjust scaling thresholds based on observed patterns during testing and production operations.

**Sources:**
- [Understand and adjust streaming units](../stream-analytics/stream-analytics-streaming-unit-consumption.md)
- [Autoscale Stream Analytics jobs](../stream-analytics/stream-analytics-autoscale.md)

### Normal operations

**Traffic routing between zones**. Azure Stream Analytics operates in an active/active model where job instances run simultaneously across all availability zones. Incoming streaming data is processed by instances in any available zone, with the service automatically managing load distribution. The platform uses internal load balancing to distribute processing tasks evenly across zones, ensuring optimal resource utilization and performance.

**Data replication between zones**. Stream Analytics replicates job state and checkpoint data synchronously across availability zones. When your job processes events and updates its state, these changes are written to multiple zones before being acknowledged. This synchronous replication ensures zero data loss even if an entire zone becomes unavailable. The replication process is transparent to your application and doesn't impact processing latency under normal conditions.

**Sources:**
- [Stream Analytics job reliability](../stream-analytics/stream-analytics-job-reliability.md)

### Zone-down experience

During an availability zone failure with zone-redundant Stream Analytics:

**Detection and response**: Microsoft's monitoring systems automatically detect zone failures within the Stream Analytics infrastructure. The service immediately initiates failover procedures without requiring any customer action. Job instances in the failed zone are marked as unhealthy, and processing workload is redistributed to instances in the remaining healthy zones.

**Notification**: Zone failures and subsequent recovery actions are logged in Azure Service Health. You can configure Service Health alerts to receive notifications about infrastructure events affecting your Stream Analytics resources. The service also logs diagnostic information about job restarts and state recovery operations.

**Active requests**: Stream Analytics uses checkpointing to maintain processing state. During a zone failure, in-flight events being processed by instances in the failed zone are automatically reprocessed from the last checkpoint by instances in healthy zones. This ensures no data loss, though some events might be processed twice if they were acknowledged but not yet checkpointed.

**Expected data loss**: No data loss occurs during zone failures due to synchronous state replication across zones. The service maintains multiple copies of checkpoint data, ensuring complete state recovery.

**Expected downtime**: Stream Analytics jobs experience minimal interruption during zone failures, typically ranging from a few seconds to under a minute. The exact duration depends on checkpoint frequency and the time required to redistribute processing load.

**Traffic rerouting**: The service automatically redirects all new input data to healthy zone instances. Existing connections from input sources are re-established with instances in operational zones. Output connections are similarly re-established, ensuring continuous data flow through your streaming pipeline.

**Sources:**
- [Azure Service Health](../service-health/overview.md)
- [Stream Analytics diagnostic logs](../stream-analytics/stream-analytics-job-diagnostic-logs.md)

### Zone recovery

When the failed availability zone recovers, Azure Stream Analytics automatically reintegrates it into the active processing pool. The service gradually redistributes processing workload back across all available zones to restore optimal load distribution. This rebalancing occurs transparently without interrupting ongoing stream processing. No customer action is required for zone recovery, as the platform handles all aspects of zone recovery operations including state synchronization and workload distribution.

**Sources:**
- [Stream Analytics job reliability](../stream-analytics/stream-analytics-job-reliability.md)

### Testing for zone failures

Azure Stream Analytics is a fully managed service where zone redundancy is handled entirely by Microsoft. You cannot directly trigger zone failovers for testing purposes. To validate your end-to-end streaming solution's resilience, focus testing on the components you control - ensure your input sources and output destinations are configured for zone redundancy and test their failover capabilities. Monitor your Stream Analytics job metrics during these tests to understand how the service maintains processing continuity during upstream or downstream failures.

**Sources:**
- [Monitor Stream Analytics job with Azure Monitor logs](../stream-analytics/stream-analytics-monitoring.md)

## Multi-region support

Azure Stream Analytics is a single-region service that doesn't provide built-in multi-region replication or automatic regional failover capabilities. Each Stream Analytics job operates within a single Azure region and becomes unavailable if that region experiences a complete outage. To achieve multi-region resilience for your streaming workloads, consider [Alternative multi-region approaches](#alternative-multi-region-approaches) to separate Stream Analytics jobs in multiple regions with appropriate data routing and synchronization strategies.


### Alternative multi-region approaches

<!--John: I pulled this section from their doc geo-redundancy which should be removed when we publish. -->
Azure Stream Analytics does not provide automatic geo-failover, but you can achieve geo-redundancy by deploying identical Stream Analytics jobs in multiple Azure regions. Each job connects to a local input and local output sources. It is the responsibility of your application to both send input data into the two regional inputs and reconcile between the two regional outputs. The Stream Analytics jobs are two separate entities.

The following diagram depicts a sample geo-redundant Stream Analytics job deployment with Event Hub input and Azure Database output.

:::image type="content" source="../stream-analytics/media/geo-redundancy/geo-redundant-jobs.png" alt-text="diagram of geo-redundant stream analytics jobs":::

#### Primary/secondary strategy

Your application needs to manage which region's output database is considered the primary and which is considered the secondary. On a primary region failure, the application switches to the secondary database and starts reading updates from that database. The actual mechanism that allows minimizing duplicate reads depends on your application. You can simplify this process by writing additional information to the output. For example, you can add a timestamp or a sequence ID to each output to make skipping duplicate rows a trivial operation. Once the primary region is restored, it catches up with the secondary database using similar mechanics.


#### Active-active strategy

For multi-region resilience with Stream Analytics, implement an active-active architecture where identical jobs run in multiple regions processing the same or partitioned data streams. Use Azure Event Hubs with geo-disaster recovery or Azure Service Bus geo-disaster recovery to replicate input data across regions. For output deduplication, leverage naturally idempotent operations or implement explicit deduplication logic using unique event identifiers.

Consider using Azure Synapse Analytics or Azure Databricks for scenarios requiring more sophisticated multi-region streaming capabilities with built-in global distribution features. These services provide additional options for cross-region replication and state management that might better suit complex multi-region requirements.

For detailed architectural patterns, see [Stream processing with Azure Stream Analytics](/azure/architecture/reference-architectures/data/stream-processing-stream-analytics)

## Backups

Stream Analytics doesn't  have a built-in backup and restore feature. However if you want to move, copy or back up your Azure Stream Analytics jobs, the Azure Stream Analytics extension for Visual Studio Code allows you to export an existing job in Azure cloud to your local computer. All the configurations of your Stream Analytics job will be saved locally and you can resubmit it to another cloud region.

<!--John: I am not sure why this title includes "between regions". I suggest we remove that, unless there is something I - or the doc - is missing here. -->
To learn how to copy, back up, and move your Stream Analytics jobs, see [Copy, back up and move your Azure Stream Analytics jobs between regions](/azure/stream-analytics/copy-job).


## Reliability during service maintenance

<!--John: I pulled this section from their doc stream-analytics-job-reliability which should be removed when we publish. -->
Part of being a fully managed service is the capability to introduce new service functionality and improvements at a rapid pace. As a result, Stream Analytics can have a service update deploy on a weekly (or more frequent) basis. No matter how much testing is done there is still a risk that an existing, running job may break due to the introduction of a bug. If you are running mission critical jobs, these risks need to be avoided. You can reduce this risk by following Azure’s *[paired region](./regions-paired.md)* model. 

## How do Azure paired regions address this concern?

Stream Analytics guarantees jobs in paired regions are updated in separate batches. Each batch has one or more regions which may be updated concurrently. The Stream Analytics service ensures any new update passes rigorous internal rings to have the highest quality. The service also proactively looks for many signals after deploying to each batch to get more confidence that there are no bugs introduced. The deployment of an update to Stream Analytics would not occur at the same time in a set of paired regions. As a result there is a sufficient time gap between the updates to identify potential issues and remediate them.

The article on **[availability and paired regions](./regions-list.md)** has the most up-to-date information on which regions are paired.

We recommended that you deploy identical jobs to both paired regions. You should then [monitor these jobs](../stream-analytics/monitor-azure-stream-analytics.md) to get notified when something unexpected happens. If one of these jobs ends up in a [Failed state](../stream-analytics/job-states.md) after a Stream Analytics service update, you can contact customer support to help identify the root cause. You should also fail over any downstream consumers to the healthy job output.

## Service-level agreement

[!INCLUDE [SLA description](includes/reliability-service-level-agreement-include.md)]

Stream Analytics provides separate availability SLAs for API calls to manage jobs, and for the operations of the jobs.

### Related content

- [Azure Stream Analytics documentation](../stream-analytics/index.yml)
- [Stream Analytics monitoring](../stream-analytics/stream-analytics-monitoring.md)
- [Azure reliability overview](/azure/reliability/overview)
- [What are availability zones?](/azure/reliability/availability-zones-overview)
- [Azure Service Health](../service-health/overview.md)
- [Handle transient faults](/azure/well-architected/reliability/handle-transient-faults)
- [Stream processing architectures](/azure/architecture/reference-architectures/data/stream-processing-stream-analytics)
