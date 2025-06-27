---
title: Reliability in Azure Databricks
description: Find out about reliability in Azure Databricks
author: anaharris-ms
ms.author: anaharris
ms.topic: reliability-article
ms.custom: subject-reliability, references_regions
ms.service: azure-databricks
ms.date: 01/15/2025
---

# Reliability in Azure Databricks

Azure Databricks is a data analytics platform optimized for the Microsoft Azure cloud services platform. Azure Databricks provides a unified environment for data engineering, data science, machine learning, and analytics workloads. The service is designed to handle large-scale data processing with built-in resilience features that help ensure your analytics workloads remain available and performant.

This article describes reliability and availability zones support in [Azure Databricks](https://learn.microsoft.com/azure/databricks/). For a more detailed overview of reliability in Azure, see [Azure reliability](/azure/reliability/overview).

## Production deployment recommendations

For production Azure Databricks deployments, use zone-redundant storage (ZRS) or geo-zone-redundant storage (GZRS) for workspace storage accounts instead of the default geo-redundant storage (GRS). This configuration provides better protection against zone-level failures within a region.

Deploy clusters across multiple availability zones when possible to improve resilience. Azure Databricks automatically distributes compute nodes across zones, but you should verify sufficient capacity exists in target zones and consider using instance pools to reduce cluster startup times during recovery scenarios.

For critical workloads requiring regional protection, implement a multi-region disaster recovery topology with separate Azure Databricks workspaces in paired regions.

**Source:** [Regional disaster recovery for Azure Databricks clusters](https://learn.microsoft.com/azure/databricks/scenarios/howto-regional-disaster-recovery)

## Reliability architecture overview

Azure Databricks reliability architecture consists of two main planes: the control plane and the compute plane. The control plane manages workspace metadata, job scheduling, and cluster orchestration, while the compute plane runs the actual data processing workloads.

The control plane uses stateless services that automatically handle VM and zone failures, with workspace data stored in zone-replicated databases. Storage accounts for Databricks Runtime images are zone-redundant with automatic failover to secondary storage accounts.

For data storage, Azure Databricks uses Azure Storage accounts for DBFS Root, which can be configured with different redundancy options from locally redundant storage (LRS) to geo-zone-redundant storage (GZRS) depending on your reliability requirements.

**Source:** [Azure Databricks disaster recovery - Inside region high availability guarantees](https://learn.microsoft.com/azure/databricks/admin/disaster-recovery#inside-region-high-availability-guarantees)

## Transient faults

[!INCLUDE [Transient fault description](includes/reliability-transient-fault-description-include.md)]

Azure Databricks handles transient faults automatically at the infrastructure level through its cluster management system. When compute nodes become unavailable due to transient issues, the cluster manager automatically requests replacement nodes from the Azure compute provider.

For applications running on Azure Databricks, implement retry logic with exponential backoff when connecting to external services such as Azure Storage, Azure SQL Database, or Azure Event Hubs. The Databricks runtime includes built-in resilience for many Azure services, but your application code should handle service-specific transient failures.

Monitor cluster health metrics including node availability, job success rates, and execution times to identify patterns that might indicate recurring transient issues requiring architectural adjustments.

**Source:** [Transient fault handling](https://learn.microsoft.com/azure/architecture/best-practices/transient-faults)

## Availability zone support

[!INCLUDE [AZ support description](includes/reliability-availability-zone-description-include.md)]

Azure Databricks supports both zone-redundant deployment for the control plane and automatic zone distribution for compute clusters. The service provides zonal failure resilience that protects against single availability zone outages while maintaining service availability.

**Source:** [Azure Databricks disaster recovery - Inside region high availability guarantees](https://learn.microsoft.com/azure/databricks/admin/disaster-recovery#inside-region-high-availability-guarantees)

### Region support

Azure Databricks availability zone support is available in all Azure regions that support availability zones. For a complete list of regions with availability zone support, see [Azure regions with availability zones](/azure/reliability/availability-zones-service-support).

### Requirements

To use availability zone support in Azure Databricks:

- Deploy workspaces in Azure regions that support availability zones
- Configure workspace storage accounts with zone-redundant storage (ZRS) or geo-zone-redundant storage (GZRS)
- Ensure sufficient compute capacity exists across multiple zones in your target region

**Source:** [Azure Databricks disaster recovery - Availability of the compute plane](https://learn.microsoft.com/azure/databricks/admin/disaster-recovery#inside-region-high-availability-guarantees)

### Considerations

Azure Databricks automatically distributes cluster nodes across availability zones, but this distribution depends on available capacity in each zone. During high-demand periods, you might experience clusters concentrated in fewer zones.

When a driver node is lost due to zone failure, the entire cluster restarts, which may cause longer recovery times compared to losing worker nodes. Plan for this behavior in your job scheduling and monitoring strategies.

**Source:** [Azure Databricks disaster recovery - Availability of the compute plane](https://learn.microsoft.com/azure/databricks/admin/disaster-recovery#inside-region-high-availability-guarantees)

### Cost

Zone-redundant storage configurations incur additional costs compared to locally redundant storage. GZRS costs more than ZRS due to the additional geo-redundancy. For current pricing information, see [Azure Storage pricing](https://azure.microsoft.com/pricing/details/storage/).

Compute costs remain the same regardless of zone distribution, as you pay for the same number of virtual machines.

**Source:** [Azure Storage pricing](https://azure.microsoft.com/pricing/details/storage/)

### Configure availability zone support

- **Create**: When creating new Azure Databricks workspaces, configure the associated storage account to use ZRS or GZRS during workspace creation.
- **Migrate**: For existing workspaces, you can change the redundancy configuration of the workspace storage account. See [Change how a storage account is replicated](/azure/storage/common/redundancy-migration) for migration procedures.

**Source:** [Change how a storage account is replicated](https://learn.microsoft.com/azure/storage/common/redundancy-migration)

### Normal operations

When all availability zones are operational, Azure Databricks distributes cluster nodes across zones automatically during cluster creation. The service balances compute load across zones while maintaining data locality for optimal performance.

Data replication for workspace storage occurs synchronously across zones when using ZRS, ensuring strong consistency with minimal performance impact. Control plane operations route automatically to healthy zones without user intervention.

**Source:** [Azure Databricks disaster recovery - Availability of the Databricks control plane](https://learn.microsoft.com/azure/databricks/admin/disaster-recovery#inside-region-high-availability-guarantees)

### Zone-down experience

During an availability zone outage:

- **Detection**: Microsoft automatically detects zone failures. No customer action is required for zone-level failover.
- **Control plane impact**: Automatic failover occurs within approximately 15 minutes. Workspace management operations remain available through healthy zones.
- **Compute plane impact**: Running clusters may lose nodes in the affected zone. The cluster manager automatically requests replacement nodes from remaining zones. If the driver node is lost, the cluster and job restart completely.
- **Data access**: Workspace data remains available if using ZRS or GZRS storage configurations.
- **Expected downtime**: Minimal for control plane operations. Compute workloads may experience brief interruptions while nodes are replaced.

**Source:** [Azure Databricks disaster recovery - Inside region high availability guarantees](https://learn.microsoft.com/azure/databricks/admin/disaster-recovery#inside-region-high-availability-guarantees)

### Failback

When the failed availability zone recovers, Azure Databricks automatically resumes normal operations across all zones. The cluster manager may rebalance node distribution during subsequent cluster creations, but existing clusters continue running in their current zones until terminated.

No customer action is required for failback operations. Normal zone distribution resumes for new cluster deployments.

**Source:** [Azure Databricks disaster recovery - Inside region high availability guarantees](https://learn.microsoft.com/azure/databricks/admin/disaster-recovery#inside-region-high-availability-guarantees)

### Testing for zone failures

Azure Databricks is a fully managed service where zone failover is handled automatically by Microsoft. You don't need to test zone failure scenarios for the service itself.

For your applications running on Azure Databricks, test job resilience by simulating driver node failures and monitoring cluster restart behavior. Validate that your data processing jobs can handle cluster restarts and resume from appropriate checkpoints.

**Source:** [Azure Databricks disaster recovery - Inside region high availability guarantees](https://learn.microsoft.com/azure/databricks/admin/disaster-recovery#inside-region-high-availability-guarantees)

## Multi-region support

Azure Databricks is a single-region service. For multi-region protection, you must implement a customer-managed disaster recovery topology using separate workspaces in different Azure regions.

**Source:** [Regional disaster recovery for Azure Databricks clusters](https://learn.microsoft.com/azure/databricks/scenarios/howto-regional-disaster-recovery)

### Region support

Multi-region disaster recovery can be implemented using any two Azure regions that support Azure Databricks. For optimal recovery scenarios, use [Azure paired regions](/azure/reliability/cross-region-replication-azure#azure-paired-regions) that provide coordinated updates and physical separation.

### Requirements

To implement multi-region support:

- Create separate Azure Databricks workspaces in primary and secondary regions
- Use geo-redundant storage (GRS) or geo-zone-redundant storage (GZRS) for data protection
- Implement synchronization processes for workspace configurations, users, and access controls
- Establish documented failover and failback procedures

**Source:** [Regional disaster recovery for Azure Databricks clusters - How to create a regional disaster recovery topology](https://learn.microsoft.com/azure/databricks/scenarios/howto-regional-disaster-recovery#how-to-create-a-regional-disaster-recovery-topology)

### Considerations

Multi-region implementations require customer-managed synchronization of workspace metadata including notebooks, job definitions, cluster configurations, and user permissions. Plan for potential data loss during regional failures based on your synchronization frequency.

Network latency between regions affects data synchronization performance. Consider using Azure paired regions to minimize latency and ensure coordinated platform updates.

**Source:** [Azure paired regions](https://learn.microsoft.com/azure/reliability/cross-region-replication-azure#azure-paired-regions)

### Cost

Multi-region deployments incur costs for multiple workspace deployments and geo-redundant storage replication. Data transfer costs apply for cross-region synchronization activities.

**Source:** [Azure Databricks pricing](https://azure.microsoft.com/pricing/details/databricks/)

### Configure multi-region support

- **Create**: Deploy additional Azure Databricks workspaces in secondary regions following the same configuration patterns as your primary region.
- **Sync**: Implement processes to synchronize workspace configurations, notebooks, and job definitions between regions using the [Databricks REST API](https://learn.microsoft.com/azure/databricks/dev-tools/api/latest/).

**Source:** [Databricks REST API](https://learn.microsoft.com/azure/databricks/dev-tools/api/latest/)

### Normal operations

During normal operations, all production workloads run in the primary region. Implement regular synchronization of workspace configurations to the secondary region and test failover procedures during maintenance windows.

Monitor both regions for capacity availability and ensure secondary region compute quotas support your failover requirements.

**Source:** [Azure Databricks disaster recovery - Typical recovery workflow](https://learn.microsoft.com/azure/databricks/admin/disaster-recovery#typical-recovery-workflow)

### Region-down experience

During a regional outage:

- **Detection**: Customer responsibility to detect regional failures requiring failover
- **Impact**: Complete service unavailability in the affected region
- **Recovery action**: Manual failover to secondary region required
- **Expected downtime**: Depends on detection time and failover procedure execution
- **Data loss**: Potential data loss based on last synchronization time

**Source:** [Azure Databricks disaster recovery - Typical recovery workflow](https://learn.microsoft.com/azure/databricks/admin/disaster-recovery#typical-recovery-workflow)

### Failback

When the primary region recovers:

1. Verify primary region service availability
2. Synchronize any changes made in secondary region back to primary
3. Execute planned failover back to primary region
4. Resume normal operations and monitoring

Plan for extended failback procedures and validate data consistency between regions before resuming production workloads.

**Source:** [Azure Databricks disaster recovery - Typical recovery workflow](https://learn.microsoft.com/azure/databricks/admin/disaster-recovery#typical-recovery-workflow)

### Testing for region failures

Conduct regular disaster recovery drills to validate your multi-region procedures. Test failover and failback processes during maintenance windows to identify gaps in automation and documentation.

Validate that your data synchronization processes can handle the volume of changes made during extended outages.

**Source:** [Regional disaster recovery for Azure Databricks clusters](https://learn.microsoft.com/azure/databricks/scenarios/howto-regional-disaster-recovery)

### Alternative multi-region approaches

For workloads requiring active-active multi-region deployment, consider architecting separate data processing pipelines in each region with independent data sources and destinations. This approach provides regional independence but requires application-level coordination.

Use Azure Data Factory or other orchestration services to coordinate multi-region data processing workflows while maintaining separate Azure Databricks workspaces in each region.

**Source:** [The ingest process with cloud-scale analytics in Azure](https://learn.microsoft.com/azure/cloud-adoption-framework/scenarios/cloud-scale-analytics/best-practices/data-ingestion)

## Backups

Azure Databricks workspace metadata is automatically backed up as part of the service's managed operations. This includes notebook content, job definitions, cluster configurations, and access control settings.

For data stored in DBFS Root or external storage services, implement backup strategies appropriate to your data protection requirements. Use geo-redundant storage for automatic cross-region backup or implement scheduled backup processes using Azure Data Factory or similar services.

Workspace-level backup and restore capabilities are not directly available. Plan for workspace recreation procedures that include restoring configurations, users, and access controls from your synchronization processes.

**Source:** [Regional disaster recovery for Azure Databricks clusters](https://learn.microsoft.com/azure/databricks/scenarios/howto-regional-disaster-recovery)

## Reliability during service maintenance

Azure Databricks performs regular maintenance activities including security updates, performance improvements, and capacity management. Most maintenance operations are transparent to running workloads.

During control plane maintenance, workspace management operations may experience brief interruptions, but running clusters continue processing jobs. Cluster creation and termination operations may be delayed during maintenance windows.

For compute plane maintenance, individual nodes may be replaced automatically without affecting overall cluster availability in multi-node configurations. Single-node clusters may experience brief interruptions during node replacement.

Monitor Azure Service Health notifications for advance notice of planned maintenance that may affect service availability.

**Source:** [Azure Service Health](https://learn.microsoft.com/azure/service-health/)

## Service-level agreement

For information about the SLA for Azure Databricks, see [Service Level Agreements (SLA) for Online Services](https://www.microsoft.com/licensing/docs/view/Service-Level-Agreements-SLA-for-Online-Services).

The Azure Databricks SLA covers service availability but does not include:
- Customer application code reliability
- External service dependencies
- Network connectivity between Azure Databricks and external services
- Customer-managed disaster recovery implementations

**Source:** [Service Level Agreements (SLA) for Online Services](https://www.microsoft.com/licensing/docs/view/Service-Level-Agreements-SLA-for-Online-Services)

## Related content

- [Azure Databricks disaster recovery](/azure/databricks/admin/disaster-recovery)
- [Reliability best practices for Azure Databricks](/azure/databricks/lakehouse-architecture/reliability/best-practices)
- [Azure reliability overview](/azure/reliability/overview)
