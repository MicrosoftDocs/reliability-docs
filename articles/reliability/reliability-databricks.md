---
title: Reliability in Azure Databricks
description: Find out about reliability in Azure Databricks
author: anaharris-ms
ms.author: anaharris
ms.topic: reliability-article
ms.custom: subject-reliability, references_regions
ms.service: azure-databricks
ms.date: 07/01/2025
---

# Reliability in Azure Databricks

[Azure Databricks](/azure/databricks/introduction/) is a data analytics platform optimized for the Microsoft Azure cloud services platform. Azure Databricks provides a unified environment for data engineering, data science, machine learning, and analytics workloads. The service is designed to handle large-scale data processing with built-in resilience features that help ensure your analytics workloads remain available and performant.

This article describes reliability and availability zones support in Azure Databricks. For a more detailed overview of reliability in Azure, see [Azure reliability](/azure/reliability/overview).

## Production deployment recommendations

<!-- Please verify that these recommendations are reasonable. -->

For production Azure Databricks deployments, deploy clusters across multiple availability zones when possible to improve resilience. Azure Databricks automatically distributes compute nodes across zones, but you should verify sufficient capacity exists in target zones and consider using instance pools to reduce cluster startup times during recovery scenarios. <!-- What's the actual guidance here? -->

Use zone-redundant storage (ZRS) or geo-zone-redundant storage (GZRS) for workspace storage accounts instead of the default geo-redundant storage (GRS). This configuration provides better protection against zone-level failures within a region.

For critical workloads requiring protection agianst region outages, implement a multi-region topology with separate Azure Databricks workspaces in multiple regions.

## Reliability architecture overview

Azure Databricks reliability architecture consists of two main planes: the *control plane* and the *compute plane*. 

- **Control plane**: Manages workspace metadata, user access, job scheduling, and cluster management. It is a stateless service that can run in multiple availability zones within a region that supports availability zones. The control plane is designed to handle zone failures automatically, with no user intervention required.

    Control plane workspace data, including Databricks Runtime Images, are stored in databases that are stored in zone-redundant storage (ZRS). In addition, all regions are configured for Geo-zone-redundant storage (GZRS), and so have a secondary storage account that is used when the primary is down.

- **Compute plane**: Runs data processing workloads using clusters of virtual machines (VMs). The compute plane is designed to handle transient faults and automatically replace failed nodes without user intervention. 

    Workspace availability depends on the availability of the control plane, but compute clusters can continue processing jobs even if the control plane experiences brief interruptions.

    Data on DBFS Root is not affected if its storage account is configured with ZRS or GZRS, which is the default.

## Transient faults

[!INCLUDE [Transient fault description](includes/reliability-transient-fault-description-include.md)]

For applications running on Azure Databricks, implement retry logic with exponential backoff when connecting to external services, or Azure services like Azure Storage, Azure SQL Database, or Azure Event Hubs. The Databricks runtime includes built-in resilience for many Azure services, but your application code should handle service-specific transient failures.

## Availability zone support

[!INCLUDE [AZ support description](includes/reliability-availability-zone-description-include.md)]

Azure Databricks supports *zone redundancy* for the control plane.

For the compute plane, Databricks supports *automatic zone distribution* for compute resources, which means that your resources are distributed across multiple availability zones. Distribution across multiple zones helps your production workloads achieve resiliency to zone outages.

### Region support

Azure Databricks availability zone support is available in all Azure regions that support Azure Databricks and that also provide availability zones. For a list of regions that support Azure Databricks, see [Products available by region](https://azure.microsoft.com/explore/global-infrastructure/products-by-region). For a complete list of regions with availability zone support, see [Azure regions with availability zones](/azure/reliability/availability-zones-service-support).

### Requirements

To use availability zone support in Azure Databricks:

- Deploy workspaces in Azure regions that support availability zones.

- Configure workspace storage accounts with zone-redundant storage (ZRS) or geo-zone-redundant storage (GZRS).

- Ensure sufficient compute capacity exists across multiple zones in your target region. Azure Databricks automatically distributes cluster nodes across zones, but you should verify that your selected instance types are available in all target zones.

### Considerations

Azure Databricks automatically distributes cluster nodes across availability zones. The distribution depends on available capacity in each zone. During high-demand periods, a cluster's nodes might be concentrated in fewer zones.

In a zone-down scenario, if nodes are lost, the Azure cluster manager requests replacement nodes from the Azure compute provider. If there is sufficient capacity in the remaining healthy zones to fulfill the request, the compute provider pulls nodes from the healthy zones to replace the nodes that were lost. 

The only exception to this is when the driver node is lost. When a driver node is lost due to zone failure, the entire cluster restarts, which may cause longer recovery times compared to losing worker nodes. Plan for this behavior in your job scheduling and monitoring strategies.

### Cost

Azure Databricks costs include both compute and storage components:

**Compute costs:** Databricks bills based on Databricks Units (DBUs), which are units of processing capability per hour based on VM instance type. Zone distribution doesn't affect compute costs, as you pay for the same number of virtual machines regardless of their availability zone placement.

**Storage costs:** When creating new Azure Databricks workspaces (after March 6, 2023), the default redundancy for the managed storage account (DBFS root) is Geo-redundant storage (GRS), not ZRS. Changing from GRS to ZRS may reduce storage costs while maintaining zone-level protection. ZRS provides protection against data center failures within a region without the additional cost of geo-replication.

For detailed pricing information, see [Azure Databricks pricing](https://azure.microsoft.com/pricing/details/databricks/) for compute costs and [Azure Storage pricing](https://azure.microsoft.com/pricing/details/storage/) for storage redundancy options.

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

### Normal operations

This section describes what to expect when a workspace is configured for with availability zone support and all availability zones are operational.

Azure Databricks distributes cluster nodes across zones automatically during cluster creation. The service balances compute load across zones while maintaining data locality for optimal performance.

Data replication for workspace storage occurs synchronously across zones when using ZRS, ensuring strong consistency with minimal performance impact. Control plane operations route automatically to healthy zones without user intervention.

### Zone-down experience

This section describes what to expect when a workspace is configured with availability zone support and there's an availability zone outage.

- **Detection**: Microsoft automatically detects zone failures. No customer action is required for zone-level failover.

- **Control plane impact**: Automatic failover occurs within approximately 15 minutes. Workspace management operations remain available through healthy zones.

- **Compute plane impact**: Running clusters may lose nodes in the affected zone. The cluster manager automatically requests replacement nodes from remaining zones. If the driver node is lost, the cluster and job restart completely.

- **Data access**: Workspace data remains available if using ZRS or GZRS storage configurations.
- **Expected downtime**: Minimal for control plane operations. Compute workloads may experience brief interruptions while nodes are replaced.

- **Expected downtime**: Minimal for control plane operations. Compute workloads may experience brief interruptions while nodes are replaced. <!-- Can we quantify this or add any more detail? -->

### Failback

When the failed availability zone recovers, Azure Databricks automatically resumes normal operations across all zones. The cluster manager may rebalance node distribution during subsequent node creations, but existing nodes continue running in their current zones until terminated.

No customer action is required for failback operations. Normal zone distribution resumes for new cluster deployments.

### Testing for zone failures

Azure Databricks is a fully managed service where zone failover is handled automatically by Microsoft. You don't need to test zone failure scenarios for the service itself.

For your applications running on Azure Databricks, test job resilience by simulating driver node failures and monitoring cluster restart behavior. Validate that your data processing jobs can handle cluster restarts and resume from appropriate checkpoints.

## Multi-region support

Azure Databricks is a single-region service. If the region is unavailable, your workspace is also unavailable.

However, workspace storage accounts can be configured with geo-zone-redundant storage (GZRS) or geo-redundant storage (GRS) for automatic cross-region data replication.

**Source:** [Regional disaster recovery for Azure Databricks clusters](https://learn.microsoft.com/azure/databricks/scenarios/howto-regional-disaster-recovery)

### Alternative multi-region approaches

Azure Databricks does not provide built-in multi-region capabilities. For comprehensive multi-region protection of your analytics workloads, you must implement your own approach.

Typical multi-region solutions involve two (or possibly more) workspaces. You can choose from several strategies. Consider the potential length of the disruption (hours or maybe even a day), the effort to ensure that the workspace is fully operational, and the effort to restore (fail back) to the primary region.


## Backups

Azure Databricks workspace metadata is automatically backed up as part of the service's managed operations. This includes notebook content, job definitions, cluster configurations, and access control settings.

For data stored in DBFS Root or external storage services, implement backup strategies appropriate to your data protection requirements. For example, you can implement scheduled backup processes using Azure Data Factory or similar services.

Workspace-level backup and restore capabilities are not directly available. Plan for workspace recreation procedures that include restoring configurations, users, and access controls from your synchronization processes.

## Service-level agreement

The service-level agreement (SLA) for Azure Databricks describes the expected availability of the service and the conditions that must be met to achieve that availability expectation. To understand those conditions, it's important that you review the [SLAs for online services](https://www.microsoft.com/licensing/docs/view/Service-Level-Agreements-SLA-for-Online-Services).

## Related content

- [Azure Databricks disaster recovery](/azure/databricks/admin/disaster-recovery)
- [Reliability best practices for Azure Databricks](/azure/databricks/lakehouse-architecture/reliability/best-practices)
- [Azure reliability overview](/azure/reliability/overview)
