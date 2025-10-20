---
title: Reliability in Azure Databricks
description: Find out about reliability in Azure Databricks
author: anaharris-ms
ms.author: anaharris
ms.topic: reliability-article
ms.custom: subject-reliability, references_regions
ms.service: azure-databricks
ms.date: 07/9/2025
---

# Reliability in Azure Databricks

[Azure Databricks](/azure/databricks/introduction/) is a data analytics platform optimized for the Microsoft Azure cloud services platform. Azure Databricks provides a unified environment for data engineering, data science, machine learning, and analytics workloads. The service is designed to handle large-scale data processing with built-in resilience features that help ensure your analytics workloads remain available and performant.

This article describes reliability and availability zones support in Azure Databricks. For a more detailed overview of reliability in Azure, see [Azure reliability](/azure/reliability/overview).

## Production deployment recommendations

To learn about how to deploy Azure Databricks to support your solution's reliability requirements, and how reliability affects other aspects of your architecture, see [Architecture best practices for Azure Databricks](/azure/well-architected/service-guides/azure-databricks).

## Reliability architecture overview

It's important that you understand the reliability of each primary component in Azure Databricks:

- **Control plane**: A stateless service that manages workspace metadata, user access, job scheduling, and cluster management.

- **DBFS Root:** A storage account that's automatically provisioned when you create an Azure Databricks workspace in your cloud account.

- **Compute plane**: Runs data processing workloads using clusters of virtual machines (VMs). The compute plane is designed to handle transient faults and automatically replace failed nodes without user intervention. 

    Workspace availability depends on the availability of the control plane, but compute clusters can continue processing jobs even if the control plane experiences brief interruptions.

## Transient faults

[!INCLUDE [Transient fault description](includes/reliability-transient-fault-description-include.md)]

For applications running on Azure Databricks, implement retry logic with exponential backoff when connecting to external services, or Azure services like Azure Storage, Azure SQL Database, or Azure Event Hubs. The Databricks runtime includes built-in resilience for many Azure services, but your application code should handle service-specific transient failures.

## Availability zone support

[!INCLUDE [AZ support description](includes/reliability-availability-zone-description-include.md)]

Azure Databricks supports *zone redundancy* for each component:

- *Control plane:* In regions that support availability zones, the control plane runs in multiple availability zones. The control plane is designed to handle zone failures automatically, with minimal impact and no user intervention required.

    Control plane workspace data, including Databricks Runtime Images, are stored in databases. In regions that support availability zones, the databases are configured to use zone-redundant storage (ZRS). Storage accounts used to serve Databricks Runtime images are also redundant inside the region, and all regions have secondary storage accounts that are used when the primary is down.

- *DBFS Root:* In regions that support availability zones, you can configure the storage account for DBFS Root with zone-redundant storage (ZRS). In regions that support availability zones and are paired, you can optionally use geo-zone-redundant storage (GZRS).
    
    > [!WARNING]
    > **Product team:** [The DR doc](/azure/databricks/admin/disaster-recovery) says GRS is default for DBFS Root, but what about in Mexico Central and Qatar Central? These regions don't support GRS.

- *Compute plane:* Databricks supports *automatic zone distribution* for compute resources, which means that your resources are distributed across multiple availability zones. Distribution across multiple zones helps your production workloads achieve resiliency to zone outages.

### Region support

Azure Databricks availability zone support is available in all Azure regions that support Azure Databricks and that also provide availability zones. For a list of regions that support Azure Databricks, see [Products available by region](https://azure.microsoft.com/explore/global-infrastructure/products-by-region). For a complete list of regions with availability zone support, see [Azure regions with availability zones](/azure/reliability/availability-zones-service-support).

### Requirements

To use availability zone support in Azure Databricks:

- Deploy workspaces [in Azure regions that support availability zones](./regions-list.md).

- Configure workspace storage accounts with zone-redundant storage (ZRS) or geo-zone-redundant storage (GZRS).

- Ensure sufficient compute capacity exists across multiple zones in your target region. Azure Databricks automatically distributes cluster nodes across zones, but you should verify that your selected instance types are available in all target zones.

### Considerations

Azure Databricks automatically distributes cluster nodes across availability zones. The distribution depends on available capacity in each zone. During high-demand periods, a cluster's nodes might be concentrated in fewer zones.

### Cost

Zone distribution doesn't affect compute costs, as you pay for the same number of virtual machines regardless of their availability zone placement. For detailed cost information, see [Azure Databricks compute pricing](https://azure.microsoft.com/pricing/details/databricks/).

The default redundancy for the managed storage account (DBFS Root) is geo-redundant storage (GRS). Changing to ZRS or GZRS might affect your storage costs. For more information, see [Azure Storage pricing](https://azure.microsoft.com/pricing/details/storage/).

> [!WARNING]
> **Product team:** As per note above, please verify the behavior in non-paired regions - they won't have GRS storage accounts, so what's the default?

### Configure availability zone support

> [!WARNING]
> **Product team:** Please verify this information.

- **Control plane:** The control plane automatically supports zone redundancy in regions with availability zones. No customer configuration is required.

- **DBFS Root:**
    - **Create new workspace with zone-redundant DBFS Root storage**: When you create a new Azure Databricks workspace, you can optionally configure the associated storage account to use ZRS or GZRS instead of the default GRS. To learn how to change workspace storage redundancy options, see [Change workspace storage redundancy options](/azure/databricks/admin/workspace/workspace-storage-redundancy).
    
    - **Enable zone redundancy on DBFS Root storage**: For existing workspaces, you can change the redundancy configuration of the workspace storage account to ZRS or GZRS. To learn how to enable zone redundancy, see [Change how a storage account is replicated](/azure/storage/common/redundancy-migration).
    
- **Compute plane:** Cluster nodes are automatically distributed across availability zones. No customer configuration is required for zone distribution.

### Normal operations

This section describes what to expect when a workspace is configured for with availability zone support and all availability zones are operational.

- **Data replication between zones:** Data replication for workspace storage occurs synchronously across zones when DBFS Root uses a ZRS or GZRS storage account. This approach ensures strong consistency with minimal performance impact.

- **Traffic routing between zones:** Azure Databricks distributes cluster nodes across zones automatically during cluster creation. The service balances compute load across zones while maintaining data locality for optimal performance.

### Zone-down experience

This section describes what to expect when a workspace is configured with availability zone support and there's an availability zone outage.

- **Detection**: Microsoft automatically detects zone failures. No customer action is required for zone-level failover.

- **Notification:** *Content pending.*
    
    > [!WARNING]
    > **Product team:** Is there any way for customers to know when a zone is out - either entirely or just for Databricks? For example, would Azure Service Health show this information?

- **Active requests:** Running clusters may lose nodes in the affected zone. The cluster manager automatically requests replacement nodes from remaining zones. If the driver node is lost, the cluster and job restart completely.

- **Expected data loss:**

    - **Control plane:** The control plane is stateless, so no data loss is expected when a zone is down.

    - **DBFS Root:** Workspace data remains available if it uses ZRS or GZRS storage configurations. 
    
    - **Compute plane:** *Content pending.*
    
        > [!WARNING]
        > **Product team:** Please clarify whether any data on VMs in the affected zone could be lost. I assume they're stateless so this wouldn't be an issue?

- **Expected downtime**:

    - **Control plane:** The Databricks control plane performs automatic failover to healthy zones within approximately 15 minutes.

    - **DBFS Root:** No downtime is expected for storage accounts that are configured to use ZRS or GZRS storage.

    - **Compute plane:** If nodes are lost because their VMs are in the affected availability zone, the Azure cluster manager requests replacement nodes from the Azure compute provider. If there is sufficient capacity in the remaining healthy zones to fulfill the request, the compute provider pulls nodes from the healthy zones to replace the nodes that were lost. 
        
        > [!WARNING]
        > **Product team:** Can we give any indication of how long it takes for the VMs to be reinstated - even if it's a ballpark?

        If the driver node is lost due to the zone failure, the entire cluster restarts, which may cause longer recovery times compared to losing worker nodes. Plan for this behavior in your job scheduling and monitoring strategies.

- **Traffic rerouting:**

    - **Control plane:** The Databricks control plane performs automatic failover to healthy zones within approximately 15 minutes.

    - **DBFS Root:** Azure Storage automatically redirects requests to storage clusters in healthy zones. 

    - **Compute plane:** *Content pending*
        
        > [!WARNING]
        > **Product team:** Please confirm how this works. I assume the cluster manager begins routing work to nodes in healthy zones as they are created?

### Failback

> [!WARNING]
> **Product team:** Please verify this information.

When the failed availability zone recovers, Azure Databricks automatically resumes normal operations across all zones. The cluster manager may rebalance node distribution during subsequent node creations, but existing nodes continue running in their current zones until terminated.

No customer action is required for failback operations. Normal zone distribution resumes for new cluster deployments.

### Testing for zone failures

Azure Databricks is a fully managed service where zone failover is handled automatically by Microsoft. You don't need to test zone failure scenarios for the service itself.

For your applications running on Azure Databricks, test job resilience by simulating driver node failures and monitoring cluster restart behavior. Validate that your data processing jobs can handle cluster restarts and resume from appropriate checkpoints.

## Multi-region support

Azure Databricks is a single-region service. If the region is unavailable, your workspace is also unavailable.

However, in [regions that are paired](./regions-list.md), DBFS Root workspace storage accounts can be configured with geo-zone-redundant storage (GZRS) or geo-redundant storage (GRS) for automatic cross-region data replication.

> [!WARNING]
> **Product team:** I assume that simply having geo-replicated storage doesn't help a lot if there's a region outage - you still need to recover the control plane and compute plane into another region. Is it worth mentioning this at all?

### Alternative multi-region approaches

Azure Databricks does not provide built-in multi-region capabilities. For comprehensive multi-region protection of your analytics workloads, you must implement your own approach.

Typical multi-region solutions involve two (or possibly more) workspaces. You can choose from several strategies, including active-passive and active-active architectures. Consider the business criticality of the work done by the Databricks cluster, the otential length of the disruption (hours or maybe even a day), the effort to ensure that the workspace is fully operational, and the effort to restore (fail back) to the primary region.

For workloads requiring multi-region protection, see [Azure Databricks - Disaster recovery](/azure/databricks/admin/disaster-recovery).

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
