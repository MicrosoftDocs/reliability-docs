---
title: Reliability in Azure Database for MySQL
description: Learn about reliability in Azure Database for MySQL, including availability zones and multi-region deployments.
ms.author: anaharris
author: anaharris-ms
ms.topic: reliability-article
ms.custom: subject-reliability
ms.service: azure-database-mysql
ms.subservice: flexible-server
ms.date: 11/19/2025
ms.update-cycle: 180-days
ai-usage: ai-assisted
# Customer intent: As an engineer responsible for business continuity, I want to understand the details of how Azure Database for MySQL works from a reliability perspective and plan disaster recovery strategies in alignment with the exact processes that Azure services follow during different kinds of situations.
---

# Reliability in Azure Database for MySQL

[Azure Database for MySQL](/azure/mysql/) is a fully managed database service that provides high availability with automatic failover, allowing you to configure redundancy and recover quickly from failures. The service supports multiple reliability architectures through zone-redundant and same-zone high availability deployments, automated backups with geo-redundancy options, and built-in monitoring capabilities.

[!INCLUDE [Shared responsibility](includes/reliability-shared-responsibility-include.md)]

This article describes how to make Azure Database for MySQL resilient to a variety of potential outages and problems, including resilience to transient faults, availability zone failures, region-wide failures, and service maintenance. It also describes how you can use backup and restore to recover from other types of problems, and highlights some key information about the Azure Database for MySQL service level agreement (SLA).

**Sources:**
- [Azure Database for MySQL overview](/azure/mysql/) - Service overview and capabilities
- [What is Azure Database for MySQL - Flexible Server?](/azure/mysql/flexible-server/overview) - Flexible Server deployment model features

## Production deployment recommendations

To learn about how to deploy Azure Database for MySQL to support your solution's reliability requirements, and how reliability affects other aspects of your architecture, see [Architecture best practices for Azure Database for MySQL in the Azure Well-Architected Framework](/azure/well-architected/service-guides/azure-db-mysql-cost-optimization).

**Sources:**
- [Architecture best practices for Azure Database for MySQL in the Azure Well-Architected Framework](/azure/well-architected/service-guides/azure-db-mysql-cost-optimization) - Comprehensive guidance for production deployments

## Reliability architecture overview

Azure Database for MySQL Flexible Server is a fully managed database service that runs the MySQL database engine on Azure infrastructure with automated management of high availability, patching, and backups.

Azure Database for MySQL provides two different reliability architectures through its High Availability setting: zone-redundant architecture, which uses zone-redundant storage (ZRS) and zonal architecture, which uses locally redundant storage (LRS).

**Zone-redundant architecture:** When you enable zone-redundant in its high availability setting, Azure creates two servers: a primary server in one availability zone and a standby replica server in another availability zone within the same region. Both servers have identical configurations, and data is stored on zone-redundant storage (ZRS) which replicates information across availability zones. The standby server continuously reads and replays log files from the primary server's storage account through storage-level replication.

**Zonal (Local-redundant) architecture:** If you don't choose a zone-redundant configuration, Flexible Server creates two servers within the same availability zone: a primary server and a standby replica server with identical configurations. Data is stored on locally redundant storage (LRS) within a single datacenter, providing infrastructure redundancy with lower network latency but without protection against zone-level failures.

For both architectures, the service handles automatic failover when issues are detected, with DNS updates redirecting client connections to ensure transparent failover from the application perspective. 

<!-- This probably doesn't fit here, but i placed it here as a place holder.-->
[!INCLUDE [Availability zone and zonal architecture](./includes/reliability-availability-zone-zonal-include.md)]

<!--John: MySQL has an entire page on architecture. Probably good to keep summary here, but link to their page.  -->
**Sources:**
- [High-availability in Azure Database for MySQL](/azure/mysql/flexible-server/concepts-high-availability) - High availability architectural models and configurations
- [What is Azure Database for MySQL - Flexible Server?](/azure/mysql/flexible-server/overview#high-availability-within-and-across-availability-zones) - Service overview including reliability features

## Resilience to transient faults

[!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)]

Azure Database for MySQL Flexible Server, like other cloud services, can experience transient faults during maintenance operations, network fluctuations, or brief service interruptions. The service does not provide built-in automatic retry mechanisms, so applications must implement proper retry logic to handle these temporary issues gracefully.

To manage transient faults when using Azure Database for MySQL, implement the following recommendations:

- **Configure connection retry logic** in your applications to handle brief connection failures during failover scenarios. Use exponential backoff strategies for retry attempts with appropriate delays between attempts.
- **Implement connection pooling** in your application or use external tools like ProxySQL to efficiently manage database connections and reduce the impact of connection establishment overhead.
- **Set appropriate timeout values** for database operations to balance between application responsiveness and allowing sufficient time for transient issues to resolve.
- **Monitor connection metrics** using Azure Monitor to identify patterns of transient failures and adjust retry policies accordingly.
- **Use deployment slots or blue-green deployments** for application updates to minimize application-level transient faults during deployments.

**Sources:**
- [High-availability in Azure Database for MySQL](/azure/mysql/flexible-server/concepts-high-availability) - Automatic failover and resilience mechanisms
- [What is Azure Database for MySQL - Flexible Server?](/azure/mysql/flexible-server/overview) - Service capabilities including fault handling

## Resilience to availability zone failures

[!INCLUDE [Resilience to availability zone failures](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)]

Azure Database for MySQL Flexible Server supports both zone-redundant and zonal deployment models for availability zones, through the configuration of its High Availability setting.

Zone-redundancy provides complete isolation and resiliency by deploying the primary server in one availability zone and a standby replica in another availability zone within the same region. This configuration offers the highest level of availability against any infrastructure failure in an availability zone. For zone-redundant deployments, you can optionally choose the availability zones for both the primary server and standby replica to optimize application architecture. Data is stored on zone-redundant storage (ZRS) which replicates information across availability zones, ensuring data availability even during zone failures.

<!-- not clear on choosing zones. Is this only for zr? -->

The service also supports zonal (same-zone or local-redundant) configurations, where both primary and standby servers are deployed within the same availability zone. This option provides infrastructure redundancy with lower network latency but doesn't protect against zone-level failures. Zonal configurations are typically chosen when cross-zone latency is too high for application requirements, and so they use locally redundant storage (LRS).



**Sources:**
- [High-availability in Azure Database for MySQL](/azure/mysql/flexible-server/concepts-high-availability) - Zone-redundant and same-zone high availability models
- [Azure services that support availability zones](/azure/reliability/availability-zones-service-support) - Service availability zone support confirmation

### Requirements

- **Region support.** Zone-redundant Azure Database for MySQL resources can be deployed [in regions that support availability zones for Azure Database for MySQL](/azure/mysql/flexible-server/overview#azure-regions), where the region supports multiple availability zones and zone-redundant Premium file shares are available. 

- **Compute tier requirements.** You must use the General Purpose or Business Critical compute tiers to enable zone redundancy or zonal configurations. The Burstable compute tier doesn't support high availability features.


**Sources:**
- [High-availability in Azure Database for MySQL](/azure/mysql/flexible-server/concepts-high-availability#limitations) - Compute tier requirements and limitations
- [What is Azure Database for MySQL - Flexible Server?](/azure/mysql/flexible-server/overview#azure-regions) - Regional availability for high availability features

### Considerations

- The service enables GTID mode for zone-redundant and zonal configurations, so you should verify that your workload doesn't have restrictions or limitations on replication with GTIDs.

- Storage autogrow is automatically enabled for servers with zone-redundant and zonal configurations and cannot be disabled.

- Restarting the primary database server to apply static parameter changes also restarts the standby replica.

**Sources:**
- [High-availability in Azure Database for MySQL](/azure/mysql/flexible-server/concepts-high-availability#limitations) - High availability considerations and limitations

### Cost

When you enable zone-redundancy or a zonal configuration, you're charged for the provisioned compute and storage for both the primary and standby replica servers. This effectively doubles your compute and storage costs for the high availability benefit.

**Sources:**
- [High-availability in Azure Database for MySQL](/azure/mysql/flexible-server/concepts-high-availability) - Billing for primary and secondary replicas

### Configure availability zone support

- **To create a zone-redundant MySQL Flexible server**, go to [Enable high availability during server creation](/azure/mysql/flexible-server/how-to-configure-high-availability#enable-high-availability-during-server-creation).

<!-- John: There is nothing here in this how-to describing how or when to set the Availability Zone. In the portal you, the customer can choose a zone to deploy to. -->

- **Disable zone redundancy**: To disable zone redundancy for an existing server, got to [Disable high availability](/azure/mysql/flexible-server/how-to-configure-high-availability#disable-high-availability).

<!--John: When you disable zone redundancy, does it default to zonal? -->

- **Migrate to zone-redundant support**: You cannot enable zone-redundant high availability after server creation. To migrate from a single-instance server to zone-redundant support, you must create a new server with zone-redundancy enabled during server creation and then migrate your data.

<!--John: Seems like the migrate guide supplies a method to "nigrate". Not sure if that info is valid or we want to move it. -->

**Sources:**
- [Manage zone redundant high availability in Azure Database for MySQL with the Azure portal](/azure/mysql/flexible-server/how-to-configure-high-availability) - Portal configuration procedures
- [Manage zone redundant high-availability in Azure Database for MySQL with Azure CLI](/azure/mysql/flexible-server/how-to-configure-high-availability-cli) - CLI configuration procedures

### Behavior when all zones are healthy

**Traffic routing between zones.** For zone-redundant deployments, client connections are routed to the primary server in the designated availability zone. The standby replica continuously synchronizes with the primary through storage-level replication but doesn't serve client traffic during normal operations.

**Data replication between zones.** Data replication occurs synchronously through zone-redundant storage (ZRS). The standby server continuously reads and replays log files from the primary server's storage account. Commits and writes are acknowledged after log files are flushed at the primary server's ZRS storage, which may result in 5-10 percent increased latency for application writes and commits due to the synchronous replication technology.

**Sources:**
- [High-availability in Azure Database for MySQL](/azure/mysql/flexible-server/concepts-high-availability#zone-redundant-high-availability-ha-architecture) - Normal operations for zone-redundant high availability

### Behavior during a zone failure

- **Detection and response**: Microsoft-managed. Azure Database for MySQL detects zone failures and automatically initiates failover to the standby replica in the healthy availability zone.
- [!INCLUDE [Availability zone down notification (Service Health only)](./includes/reliability-availability-zone-down-notification-service-include.md)]
- **Active requests**: Active connections to the failed zone are terminated and must be re-established. Applications should implement retry logic to handle connection interruptions.
- **Expected data loss**: RPO=0. No data loss expected due to synchronous replication through zone-redundant storage.
- **Expected downtime**: RTO is expected to be 60-120 seconds during automatic failover to the standby replica.
- **Traffic rerouting**: DNS updates redirect new connections to the standby replica, which becomes the new primary. Client applications reconnect automatically using the same connection string.

**Sources:**
- [Overview of business continuity with Azure Database for MySQL - Flexible Server](/azure/mysql/flexible-server/concepts-business-continuity#unplanned-downtime-mitigation) - Zone failure recovery processes and RTOs/RPOs

### Zone Recovery

When the availability zone recovers, Azure Database for MySQL automatically initiates the process to restore the standby replica in the recovered zone. The high availability solution brings back the old primary server when possible and places it as the new standby. This process is fully managed by Microsoft and doesn't require customer intervention.

**Sources:**
- [High-availability in Azure Database for MySQL](/azure/mysql/flexible-server/concepts-high-availability#zone-redundant-high-availability-ha-architecture) - Automatic zone recovery process

### Test for zone failures

You can test failover scenarios by manually initiating a forced failover through the Azure portal or Azure CLI. The portal provides a forced failover option in the High Availability section that allows you to validate your application's handling of failover events. This testing helps ensure that your application properly handles connection interruptions and reconnects appropriately after failover completes.

The service doesn't provide Azure Chaos Studio integration for zone failure simulation, but the [forced failover](/azure/mysql/flexible-server/how-to-configure-high-availability#forced-failover) functionality provides adequate testing capabilities for zone-level failure scenarios.

**Sources:**
- [Forced failover](/azure/mysql/flexible-server/how-to-configure-high-availability#forced-failover) - Manual failover testing procedures

## Resilience to region-wide failures

Azure Database for MySQL Flexible Server doesn't provide native multi-region support with automatic failover. The service is deployed in a single region and doesn't offer built-in cross-region replication or automatic regional failover capabilities. However, the service offers robust disaster recovery capabilities through multiple features: [geo-redundant backups](/azure/mysql/flexible-server/concepts-backup-restore#geo-restore) and [read replicas](/azure/mysql/flexible-server/concepts-read-replicas) for disaster recovery. Each option provides different recovery characteristics and cost implications.

### Geo-redundant backup and restore

Azure Database for MySQL provides [geo-redundant backup capabilities](/azure/mysql/flexible-server/concepts-backup-restore#backup-redundancy-options) that replicate backups to the paired region for regional resiliency.

#### Requirements

- **Region support**: Geo-redundant backup is available in regions that support [Azure paired regions](./regions-list.md).
- **Configuration timing**: Geo-redundant backup storage must be configured during server creation for new servers, but can be upgraded after server creation for existing servers.
- **Backup retention**: Geo-redundant backups support the same retention periods as locally redundant backups (1 to 35 days).
- **Zone-redundant HA servers**: For servers with zone-redundant high availability, geo-redundancy can only be enabled or disabled at server creation time.

#### Configure multi-region support

- **Create**: Configure geo-redundant backup storage during server creation through [Azure portal, CLI, or ARM templates](/azure/mysql/flexible-server/concepts-backup-restore#backup-redundancy-options).
- **Enable**: Upgrade existing servers from locally redundant to geo-redundant backup storage after deployment through [Compute + Storage configuration settings](/azure/mysql/flexible-server/concepts-backup-restore#move-from-other-backup-storage-options-to-geo-redundant-backup-storage).

#### Considerations

Geo-redundant backup storage provides at least 99.99999999999999% (16 nines) durability. For servers with zone-redundant high availability, backup storage is automatically set to zone-redundant within the primary region. Geo-redundant backups enable [geo-restore](/azure/mysql/flexible-server/concepts-backup-restore#geo-restore) to recover from regional failures by creating a new server in the paired region or any other supported region where the service is available.

**Universal Geo-restore**: You can restore to any non-paired Azure supported region (except Brazil South, USGov Virginia, and West US 3) using the Universal Geo-restore capability. Geo-restore can also be performed on [stopped servers using Azure CLI](/azure/mysql/flexible-server/how-to-restore-server-cli).

#### Cost

Geo-redundant backup storage is charged at a different rate than locally redundant storage. Backup storage costs apply for consumption beyond 100% of total provisioned server storage.

#### Behavior during a region failure

- **Detection and response**: Customer-managed. During a regional outage, you must manually initiate [geo-restore](/azure/mysql/flexible-server/concepts-backup-restore#geo-restore) to create a new server in another region.
- **Expected data loss**: RPO is less than 1 hour, depending on when the last geo-redundant backup was performed. There's a delay between when a backup is taken and when it's replicated to different region, which can be up to an hour.
- **Expected downtime**: RTO varies based on the database size, transaction log size, network bandwidth, and the total number of databases recovering in the same region at the same time.
- **Traffic rerouting**: Customer-managed. You must update application connection strings to point to the new server created through geo-restore.
- **Primary region down**: When the primary region is down, you can't create geo-redundant servers in the respective geo-paired region because storage can't be provisioned in the primary region. You can still geo-restore by disabling geo-redundancy option in the restore portal experience.

#### Region recovery

When the primary region recovers, you can manually switch back by creating a new server in the original region using point-in-time restore and updating application connection strings.

#### Test for region failures

You can test regional disaster recovery by performing geo-restore operations to secondary regions during planned testing windows.

### Cross-region read replicas for disaster recovery

Azure Database for MySQL supports [read replicas](/azure/mysql/flexible-server/concepts-read-replicas) that can be deployed in different regions to create cross-region disaster recovery capabilities.

#### Requirements

- **Region support**: Read replicas can be created in any [Azure supported region where Azure Database for MySQL Flexible Server is available](https://azure.microsoft.com/explore/global-infrastructure/products-by-region/table).
- **Source server requirements**: The source server must be running and accessible.
- **Compute tier requirements**: Read replicas are available only for General Purpose and Business Critical compute tiers. The Burstable compute tier doesn't support read replicas.
- **Replica limits**: You can replicate from a source server to up to five replicas.

#### Configure multi-region support

- **Create**: Deploy read replicas in different regions through the [Azure portal](/azure/mysql/flexible-server/how-to-read-replicas-portal), CLI, or ARM templates. You can optionally [choose the availability zone](/azure/mysql/flexible-server/how-to-read-replicas-portal#create-a-read-replica) for the read replica when creating it.
- **Promote**: Read replicas can be [promoted to independent servers](/azure/mysql/flexible-server/how-to-read-replicas-portal#stop-replication-to-a-replica-server) during regional outages by stopping replication, which converts the replica into a standalone read-write server.

#### Considerations

Read replicas use [MySQL asynchronous replication](/azure/mysql/flexible-server/concepts-read-replicas), which may result in some data loss during regional failures. The replica promotion process is manual and requires updating application connection strings. There's a measurable delay between the source and the replica, and data on the replica eventually becomes consistent with the source data. The promotion process is irreversible - once replication is stopped, it cannot be reestablished.

#### Cost

You're charged for compute and storage resources for each read replica in addition to the primary server costs.

#### Behavior during a region failure

- **Detection and response**: Customer-managed. You must manually [promote a read replica](/azure/mysql/flexible-server/how-to-read-replicas-portal#stop-replication-to-a-replica-server) to become an independent server.
- **Expected data loss**: Limited data loss possible due to asynchronous replication lag between regions. RTO in most cases is a few minutes and RPO < 1 hour, but can be much higher depending on various factors including latency between sites, the amount of data to be transmitted, and the primary database write workload.
- **Expected downtime**: RTO depends on the time required to promote the replica and update application connections.
- **Traffic rerouting**: Customer-managed. You must update application connection strings to point to the promoted replica.

#### Region recovery

When the primary region recovers, you can establish a new replication relationship or continue using the promoted replica as the primary server.

#### Test for region failures

You can test read replica promotion scenarios during planned maintenance windows to validate disaster recovery procedures.

### Custom multi-region solutions for resiliency

If you need to use Azure Database for MySQL in multiple regions with more control over failover timing and data consistency, you need to deploy separate resources in each region and implement application-level patterns.

When you create identical deployments in multiple Azure regions, your application becomes less susceptible to single-region disasters. This approach requires you to configure load balancing and failover policies at the application level. You also need to implement custom data replication logic to synchronize critical information across regions.

For architecture examples that illustrate multi-region patterns, see:

- [Multi-region load balancing with Traffic Manager and Application Gateway](/azure/architecture/high-availability/reference-architecture-traffic-manager-application-gateway)
- [Highly available multi-region web application](/azure/architecture/web-apps/app-service/architectures/multi-region)
- [Enterprise integration using queues and events](/azure/architecture/reference-architectures/enterprise-integration/queues-events)

**Sources:**
- [Overview of business continuity with Azure Database for MySQL - Flexible Server](/azure/mysql/flexible-server/concepts-business-continuity) - Comprehensive business continuity planning and recovery strategies
- [Backup and restore in Azure Database for MySQL](/azure/mysql/flexible-server/concepts-backup-restore) - Complete backup and restore capabilities including geo-restore procedures
- [Read replicas in Azure Database for MySQL](/azure/mysql/flexible-server/concepts-read-replicas) - Cross-region replication and disaster recovery using read replicas
- [How to create and manage read replicas using Azure portal](/azure/mysql/flexible-server/how-to-read-replicas-portal) - Step-by-step procedures for read replica management

## Backup and restore

Azure Database for MySQL Flexible Server provides automated backup capabilities with flexible retention periods and multiple redundancy options to protect your data against planned and unplanned events.

The service automatically performs daily backups of database files and continuously backs up transaction logs. Backups can be retained for any period between 1 to 35 days, allowing point-in-time restore to any moment within the backup retention period. Recovery time depends on the size of data to restore plus the time required for log recovery.

Three backup redundancy options are available: locally redundant backup storage provides 11 nines durability within a single datacenter, zone-redundant backup storage provides 12 nines durability across availability zones within a region, and geo-redundant backup storage provides 16 nines durability by replicating backups to the geo-paired region.

For servers with zone-redundant high availability, backup storage is automatically set to zone-redundant. For servers with same-zone high availability or no high availability, backup storage defaults to locally redundant but can be upgraded to geo-redundant storage after server creation. Geo-redundant backups enable geo-restore to recover from regional failures by creating a new server in the paired region or any other supported region.

For more information, see [Backup and restore in Azure Database for MySQL](/azure/mysql/flexible-server/concepts-backup-restore).


## Resilience to service maintenance

Azure Database for MySQL Flexible Server performs automated patching of the underlying hardware, operating system, and database engine including security and software updates. For the MySQL engine, planned maintenance releases also include minor version upgrades.

You can configure the patching schedule to be system-managed or define your own custom schedule to minimize business impact. During maintenance, the patch is applied and the server might require a restart. With zone-redundant high availability enabled, maintenance operations are performed in a rolling manner to minimize service disruption. The standby replica receives updates first, followed by a failover to maintain service availability while the original primary receives updates.

Custom maintenance windows allow you to make your patching cycle predictable and choose a maintenance window that has minimum impact on your business operations. The service follows a monthly release schedule for continuous integration and release.

For more information, see [What is Azure Database for MySQL - Flexible Server?](/azure/mysql/flexible-server/overview#automated-patching-with-a-managed-maintenance-window).

## Service-level agreement

The service-level agreement (SLA) for Azure Database for MySQL describes the expected availability of the service, and the conditions that must be met to achieve that availability expectation.

The SLA provides different uptime guarantees based on the configuration tier and high availability settings. Servers configured with zone-redundant high availability achieve the highest availability targets, while single-instance deployments have lower SLA guarantees. The Business Critical and General Purpose tiers both support high availability configurations that enhance the SLA, while the Burstable tier operates without high availability and has corresponding SLA limitations.

For the most current SLA information and specific uptime guarantees for each configuration tier, see [SLA for Azure Database for MySQL](https://azure.microsoft.com/support/legal/sla/mysql/).

**Sources:**
- [SLA for Azure Database for MySQL](https://azure.microsoft.com/support/legal/sla/mysql/) - Official service level agreement details

## Related content

- [What are availability zones?](/azure/reliability/availability-zones-overview)
- [Azure reliability](/azure/reliability/overview)
- [High-availability in Azure Database for MySQL](/azure/mysql/flexible-server/concepts-high-availability)
- [Overview of business continuity with Azure Database for MySQL - Flexible Server](/azure/mysql/flexible-server/concepts-business-continuity)
- [Backup and restore in Azure Database for MySQL](/azure/mysql/flexible-server/concepts-backup-restore)
