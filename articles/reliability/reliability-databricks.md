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

[Azure Databricks](/azure/databricks/introduction/) is a data analytics platform optimized for the Microsoft Azure cloud services platform. Azure Databricks provides a unified environment for data engineering, data science, machine learning, and analytics workloads. The service is designed to handle large-scale data processing with built-in resilience features that help ensure your analytics workloads remain available and performant.

This article describes reliability and availability zones support in Azure Databricks. For a more detailed overview of reliability in Azure, see [Azure reliability](/azure/reliability/overview).

## Production deployment recommendations

For production Azure Databricks deployments, use zone-redundant storage (ZRS) or geo-zone-redundant storage (GZRS) for workspace storage accounts instead of the default geo-redundant storage (GRS). This configuration provides better protection against zone-level failures within a region.

Deploy clusters across multiple availability zones when possible to improve resilience. Azure Databricks automatically distributes compute nodes across zones, but you should verify sufficient capacity exists in target zones and consider using instance pools to reduce cluster startup times during recovery scenarios.

For critical workloads requiring regional protection, implement a multi-region disaster recovery topology with separate Azure Databricks workspaces in paired regions.

**Source:** [Regional disaster recovery for Azure Databricks clusters](https://learn.microsoft.com/azure/databricks/scenarios/howto-regional-disaster-recovery)

## Reliability architecture overview

Azure Databricks reliability architecture consists of two main planes: the *control plane* and the *compute plane*. 

- **Control plane**: Manages workspace metadata, user access, job scheduling, and cluster management. It is a stateless service that can run in multiple availability zones within a region that supports availability zones. The control plane is designed to handle failures automatically, with no user intervention required.

Control plane workspace data, including Databricks Runtime Images, are stored in databases that are stored in zone-redundant storage (ZRS). In addition, all regions are configured for Geo-zone-redundant storage (GZRS), and so have a secondary storage account that is used when the primary is down.

- **Compute plane**: Runs data processing workloads using clusters of virtual machines (VMs). The compute plane is designed to handle transient faults and automatically replace failed nodes without user intervention. 

Workspace availability depends on the availability of the control plane, but compute clusters can continue processing jobs even if the control plane experiences brief interruptions.

Data on DBFS Root is not affected if its storage account is configured with ZRS or GZRS, which is the default.


**Source:** [Azure Databricks disaster recovery - Inside region high availability guarantees](https://learn.microsoft.com/azure/databricks/admin/disaster-recovery#inside-region-high-availability-guarantees)

## Transient faults

[!INCLUDE [Transient fault description](includes/reliability-transient-fault-description-include.md)]

Azure Databricks handles transient faults automatically at the infrastructure level through its cluster management system. When compute nodes become unavailable due to transient issues, the cluster manager automatically requests replacement nodes from the Azure compute provider.

For applications running on Azure Databricks, implement retry logic with exponential backoff when connecting to external services such as Azure Storage, Azure SQL Database, or Azure Event Hubs. The Databricks runtime includes built-in resilience for many Azure services, but your application code should handle service-specific transient failures.

**Source:** [Transient fault handling](https://learn.microsoft.com/azure/architecture/best-practices/transient-faults)

## Availability zone support

[!INCLUDE [AZ support description](includes/reliability-availability-zone-description-include.md)]

Azure Databricks supports *zone redundancy* for the control plane.

For the compute plane, Databricks supports *automatic zone distribution* for compute resources, which means that your resources are distributed across multiple availability zones. Distribution across multiple zones helps your production workloads achieve resiliency and reliability. 

Compute resource zone distribution in a zone-redundant deployment follows specific rules. For more information, see [Considerations](#considerations).

**Source:** [Azure Databricks disaster recovery - Inside region high availability guarantees](https://learn.microsoft.com/azure/databricks/admin/disaster-recovery#inside-region-high-availability-guarantees)

### Region support

Azure Databricks availability zone support is available in all Azure regions that support availability zones. For a complete list of regions with availability zone support, see [Azure regions with availability zones](/azure/reliability/availability-zones-service-support).

### Requirements

To use availability zone support in Azure Databricks:

- Deploy workspaces in Azure regions that support availability zones.

- Configure workspace storage accounts with zone-redundant storage (ZRS) or geo-zone-redundant storage (GZRS).

- Ensure sufficient compute capacity exists across multiple zones in your target region. Azure Databricks automatically distributes cluster nodes across zones, but you should verify that your selected instance types are available in all target zones.

**Source:** [Azure Databricks disaster recovery - Availability of the compute plane](https://learn.microsoft.com/azure/databricks/admin/disaster-recovery#inside-region-high-availability-guarantees)

### Considerations

Azure Databricks automatically distributes cluster nodes across availability zones. The distribution depends on available capacity in each zone. During high-demand periods, you might experience clusters that concentrated in fewer zones.

In a zone-down scenario, if nodes are lost, the Azure cluster manager requests replacement nodes from the Azure compute provider. If there is sufficient capacity in the remaining healthy zones to fulfill the request, the compute provider pulls nodes from the healthy zones to replace the nodes that were lost. 

The only exception to this is when the driver node is lost. 
When a driver node is lost due to zone failure, the entire cluster restarts, which may cause longer recovery times compared to losing worker nodes. Plan for this behavior in your job scheduling and monitoring strategies.


**Source:** [Azure Databricks disaster recovery - Availability of the compute plane](https://learn.microsoft.com/azure/databricks/admin/disaster-recovery#inside-region-high-availability-guarantees)

### Cost

Azure Databricks costs include both compute and storage components:

**Compute costs:** Databricks bills based on Databricks Units (DBUs), which are units of processing capability per hour based on VM instance type. Zone distribution doesn't affect compute costs, as you pay for the same number of virtual machines regardless of their availability zone placement.

**Storage costs:** When creating new Azure Databricks workspaces (after March 6, 2023), the default redundancy for the managed storage account (DBFS root) is Geo-redundant storage (GRS), not ZRS. Changing from GRS to ZRS may reduce storage costs while maintaining zone-level protection. ZRS provides protection against data center failures within a region without the additional cost of geo-replication.

For detailed pricing information, see [Azure Databricks pricing](https://azure.microsoft.com/pricing/details/databricks/) for compute costs and [Azure Storage pricing](https://azure.microsoft.com/pricing/details/storage/) for storage redundancy options.

**Source:** [Azure Databricks pricing](https://azure.microsoft.com/pricing/details/databricks/) and [Change workspace storage redundancy options](https://learn.microsoft.com/azure/databricks/admin/workspace/workspace-storage-redundancy)

### Configure availability zone support

**Control plane configuration:**
- The control plane automatically supports zone redundancy in regions with availability zones. No customer configuration is required.
- Control plane workspace data is automatically stored using zone-redundant storage (ZRS) with geo-zone-redundant storage (GZRS) for additional protection.

**Compute plane configuration:**
- Cluster nodes are automatically distributed across availability zones. No customer configuration is required for zone distribution.
- Azure Databricks handles zone placement automatically based on available capacity.

**Workspace storage configuration:**
- **Create**: When creating new Azure Databricks workspaces, you can optionally configure the associated storage account to use ZRS or GZRS instead of the default GRS during workspace creation. See [Change workspace storage redundancy options](/azure/databricks/admin/workspace/workspace-storage-redundancy).
- **Migrate**: For existing workspaces, you can change the redundancy configuration of the workspace storage account from GRS to ZRS or GZRS. See [Change how a storage account is replicated](/azure/storage/common/redundancy-migration) for migration procedures.

**Source:** [Change workspace storage redundancy options](https://learn.microsoft.com/azure/databricks/admin/workspace/workspace-storage-redundancy) and [Azure Databricks disaster recovery - Inside region high availability guarantees](https://learn.microsoft.com/azure/databricks/admin/disaster-recovery#inside-region-high-availability-guarantees)

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

Azure Databricks workspaces are single-region services that cannot span multiple regions. However, workspace storage accounts can be configured with geo-zone-redundant storage (GZRS) or geo-redundant storage (GRS) for automatic cross-region data replication.

Azure Databricks does not provide built-in multi-region disaster recovery capabilities. For comprehensive multi-region protection of your analytics workloads, you must implement [alternative multi-region approaches](#alternative-multi-region-approaches).

**Source:** [Regional disaster recovery for Azure Databricks clusters](https://learn.microsoft.com/azure/databricks/scenarios/howto-regional-disaster-recovery)

### Alternative multi-region approaches

Typical disaster recovery solutions involve two (or possibly more) workspaces. You can choose from several strategies. Consider the potential length of the disruption (hours or maybe even a day), the effort to ensure that the workspace is fully operational, and the effort to restore (fail back) to the primary region.

For workloads requiring multi-region protection, see [Databricks -Choose a recovery solution strategy](/azure/databricks/admin/disaster-recovery#choose-a-recovery-solution-strategy).


## Backups

Azure Databricks workspace metadata is automatically backed up as part of the service's managed operations. This includes notebook content, job definitions, cluster configurations, and access control settings.

For data stored in DBFS Root or external storage services, implement backup strategies appropriate to your data protection requirements. Use geo-redundant storage for automatic cross-region backup or implement scheduled backup processes using Azure Data Factory or similar services.

Workspace-level backup and restore capabilities are not directly available. Plan for workspace recreation procedures that include restoring configurations, users, and access controls from your synchronization processes.

**Source:** [Regional disaster recovery for Azure Databricks clusters](https://learn.microsoft.com/azure/databricks/scenarios/howto-regional-disaster-recovery)

## Reliability during service maintenance

Azure Databricks performs regular maintenance activities including security updates, performance improvements, and capacity management. Most maintenance operations are transparent to running workloads.

During control plane maintenance, workspace management operations may experience brief interruptions, but running clusters continue processing jobs. Cluster creation and termination operations may be delayed during maintenance windows.

For compute plane maintenance, individual nodes may be replaced automatically without affecting overall cluster availability in multi-node configurations. Single-node clusters may experience brief interruptions during node replacement.


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
