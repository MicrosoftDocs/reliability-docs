---
title: Reliability in Azure Database for PostgreSQL
description: Learn about reliability in Azure Database for PostgreSQL, including availability zones and multi-region deployments.
ms.author: maghan
author: markingmyname
ms.topic: reliability-article
ms.custom: subject-reliability
ms.service: azure-database-postgresql	
ms.subservice: flexible-server
ms.date: 03/10/2026
ai.usage: ai-assisted
---

# Reliability in Azure Database for PostgreSQL

[Azure Database for PostgreSQL](/azure/postgresql/flexible-server/overview) is a fully managed database service designed to give you granular control and flexibility over database management functions and configuration settings. The service provides flexibility and high availability based on your requirements.

[!INCLUDE [Shared responsibility](includes/reliability-shared-responsibility-include.md)]

This article describes how to make Azure Database for PostgreSQL resilient to a variety of potential outages and problems, including transient faults, availability zone outages, region outages, and service maintenance. It also describes how you can use backups to recover from other types of problems, and highlights some key information about the Azure Database for PostgreSQL service level agreement (SLA).

## Production deployment recommendations

To learn about how to deploy Azure Database for PostgreSQL to support your solution's reliability requirements, and how reliability affects other aspects of your architecture, see [Architecture best practices for Azure Database for PostgreSQL in the Azure Well-Architected Framework](/azure/well-architected/service-guides/postgresql).

## Reliability architecture overview

[!INCLUDE [Introduction to reliability architecture overview section](includes/reliability-architecture-overview-introduction-include.md)]

### Logical architecture

When you work with Azure Database for PostgreSQL, you deploy a *server*, which represents the compute and storage resources required to support your database server. You deploy one or more *databases* to the server.

Servers can be deployed in multiple *compute tiers*: Burstable, General Purpose, and Memory Optimized, each of which are optimized for different kinds of workloads. In some Azure regions, you can deploy servers with [Azure Confidential Computing](/azure/postgresql/security/security-confidential-computing).

For more information about the general service architecture and deployment models, see [What is Azure Database for PostgreSQL?](/azure/postgresql/flexible-server/overview).

### Physical architecture

- **Compute and storage separation:** Azure Database for PostgreSQL uses a compute and storage separation architecture designed to support high availability. The database engine runs on a Linux virtual machine, while data files reside on Azure storage that maintains three locally redundant synchronous copies of the database files, ensuring data durability.

- **High availability:** You can optionally enable a *high availability configuration* on your server. When you enable the high availability configuration, the service provisions and maintains a warm standby server. Data changes on the primary server are synchronously replicated to the standby server to ensure zero data loss. The architecture separates the compute layer from the storage layer, allowing the service to handle different types of failures appropriately. For higher resiliency, you can spread the servers across availability zones.

    :::image type="content" source="./media/reliability-database-postgresql/high-availability.png" alt-text="Diagram showing the high availability architecture, with a primary and standby server." border="false" :::

    A standby replica is deployed in the same VM configuration - including vCores, storage, and network settings - as the primary replica.

    Operations such as stop, start, and restart are performed on both primary and standby database replicas at the same time. Planned events such as scale computing and scale storage happen on the standby first and then on the primary replica. Currently, the server doesn't fail over for these planned operations.

    For more information, see [High availability in Azure Database for PostgreSQL](/azure/postgresql/high-availability/concepts-high-availability).

- **Backups:** Azure Database for PostgreSQL automatically creates server backups. For more information, see [Backup and restore](#backup-and-restore).

## Resilience to transient faults

[!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)]

Your applications must handle transient connectivity errors that can occur during maintenance, scaling operations, or network interruptions. Follow these recommendations:

> [!div class="checklist"]
> - When your application detects transient faults, it should typically retry the operation. Include exponential backoff logic that waits for an increasing amount of time between retries, and cap the number of retry attempts. If the maximum number of retry attempts is reached, your application should treat the operation as failed.
> - Where possible, use client libraries (also called drivers) that automatically handle retries.
> - Transient errors that occur during write operations require more careful consideration. Consider making your write operations idempotent, so they can be safely executed multiple times.

For more information, see [Handling transient connectivity errors in Azure Database for PostgreSQL](/azure/postgresql/flexible-server/concepts-connectivity).

## Resilience to availability zone failures

[!INCLUDE [Resilience to availability zone failures](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)]

Azure Database for PostgreSQL has different types of availability zone support, which you choose though the *high availability* configuration. Enabling high availability deploys a secondary *standby* replica alongside your primary. This high availability model is designed to ensure that committed data is never lost during failures. Whichever high availability deployment model you choose, data is synchronously committed to both the primary and standby replicas. If a disruption occurs to the primary replica, the server automatically fails over to the standby replica.

Data files and write-ahead logs (WALs) are stored on premium managed disks within each availabilty zone, with locally redundant storage (LRS) that automatically stores three data copies within each zone.

When you configure a server, you select one of the following configurations:

- **Zone-redundant high availability:** Zone redundancy provides the highest level of zone resilience by deploying a primary replica in one availability zone and a standby replica in a different availability zone. The standby replica uses similar compute, storage, and network configuration to the primary. A zone-redundant configuration provides physical isolation of the entire stack between primary and standby replicas.

    You can optionally select the zones for both the primary and secondary replicas, or you can let Microsoft choose zones for you. 

    We recommend zone-redundant deployments for most production servers.

     :::image type="content" source="./media/reliability-database-postgresql/zone-redundant.png" alt-text="Diagram showing a zone-redundant server, with the primary and standby servers in different availability zones." border="false" :::

    Read queries are processed within the primary replica's availability zone, so enable zone redundancy doesn't affect read latency. However, there can be some small latency impact on writes and commits due to synchronous replication between the replicas across zones. The amount of impact is specific to your workloads, the SKU type you select, and the region.

- **Zonal (same-zone) high availability:** In the single-zone configuration, the primary and standby replicas are both placed into the same availability zone. If a disruption occurs to the primary replica, but the zone is still healthy, the server automatically fails over to the standby replica. A zonal deployment gives you high availability within a single availability zone. It protects you against node-level failures and and also helps with reducing application downtime during planned and unplanned downtime events. However, it doesn't protect against an outage in that zone.

    You select which zone the primary and standby replicas are placed into. If your region doesn't support availability zones, the region effectively functions as a single zone, and so the only high availability configuration you can select is same-zone.

    :::image type="content" source="./media/reliability-database-postgresql/zonal.png" alt-text="Diagram showing a zonal server, with the primary and standby servers in the same availability zone." border="false" :::

    [!INCLUDE [Zonal resource description](includes/reliability-availability-zone-zonal-include.md)]

    Because the replicas are in the same zone, it can reduce the write latency to applications you deploy within the same zone.

If you configure your server without high availability, then it runs on a single replica. If that replica or its zone go down, your server is unavailable. For more information, see [Configurations without availability zones](/azure/postgresql/high-availability/concepts-high-availability#configurations-without-availability-zones).

### Requirements

- **Region support**: Azure Database for PostgreSQL's support for availability zone configuratons differs between Azure regions. For a full list of regions, and the types of availability zone support and any specific considerations for that region, see [Azure regions](/azure/postgresql/overview#azure-regions).

- **Compute tier:** The following table lists the compute tier support for each type of availability zone support:

    | Pricing tier | Zone-redundant | Zonal (same-zone) |
    |---|---|---|
    | Burstable | Not supported | Supported |
    | General Purpose | Supported | Supported |
    | Memory Optimized | Supported | Supported |

- **Service tier**: Zone redundancy require General Purpose or Memory Optimized tiers.

    Zonal (same-zone) deployments are supported on all pricing tiers. 

### Considerations

- **Region capacity:** If a zone-redundant server's region lacks capacity for a zone-redundant configuration, you can tell Azure to create both replicas in the same zone temporarily, then automatically migrate to achieve zone redundancy once capacity is available. For more information, see [Configure Business Critical (High Availability) options](/azure/postgresql/high-availability/concepts-high-availability#configure-business-critical-high-availability-options).

- **Zone-redundant backups:** The primary database replica periodically performs automatic backups. At the same time, the standby replica continuously archives the transaction logs in the backup storage. If the region supports availability zones, backup data is stored on zone-redundant storage (ZRS). In regions that don't support availability zones, backup data is stored on local redundant storage (LRS).

> [!WARNING]
> **Note to PG:** Can you confirm that ZRS backups are used regardless of the server's HA or zone config, and it's purely about the region?
<!-- TODO if it is, it becomes an option for restoring a zonal server -->

### Cost

When you enable high availability, the secondary replica is created and billed at the same rate as the primary. The availability zone configuration doesn't affect the cost. There are no charges for data replication within or between availability zones. For detailed pricing information, see [Azure Database for PostgreSQL pricing](https://azure.microsoft.com/pricing/details/postgresql/flexible-server/).

### Configure availability zone support

To configure availability zone support for a server, you configure the high availability settings.

- **Create a server with zone-redundant or zonal (single-zone) high availability:** To learn how to create a server with availability zone configuration, see [Quickstart: Create an Azure Database for PostgreSQL in the Azure portal](/azure/postgresql/flexible-server/quickstart-create-server-portal).

- **Change the availability zone configuration for existing servers:** You can change the availability zone configuration for existing servers by changing the high availability settings. For detailed steps, see [Enable high availability for existing servers](/azure/postgresql/flexible-server/how-to-configure-high-availability#enable-high-availability-for-existing-servers).

    You can't change the zone used for either the primary or secondary replicas after they've been created. You need to recreate the server.

    > [!TIP]
    > It's a good idea to wait until the server activity is low before you change high availability configuration.

- **Disable high availability:** Disabling high availability removes the standby replica, so your server has a single replica and isn't resilient to outages in its availability zone. For more information, see [Disable high availability](/azure/postgresql/flexible-server/how-to-configure-high-availability#disable-high-availability).

### Behavior when all zones are healthy

This section describes what to expect when servers are configured with high availability and availability zone support and all availability zones are operational.

- **Cross-zone operation:** PostgreSQL client applications connect to the primary server by using the database server name. Azure Database for PostgreSQL uses an active/passive configuration where all database connections and queries are handled by the primary server in the primary availability zone. The standby server doesn't serve client traffic during normal operations.

- **Cross-zone data replication:** Changes to data are replicated synchronously between the primary and standby servers. Transactions aren't considered complete until both the primary and standby replicas acknowledge the write.

    Application transaction-triggered write and commits first log to the WAL on the primary server. The primary server streams these logs to the standby server by using the Postgres streaming protocol. When the standby server storage persists the logs, the primary server acknowledges write completion. The application commits its transaction only after this acknowledgment. This acknowledgment process doesn't wait for the logs to be applied to the standby server.
    
    The effects of replication are different depending on the availability zone configuration that your server uses:

    - *Zone-redundant:* Because the replicas are in separate zones, this approach ensures zero data loss during a zone failure. This situation is also sometimes called achieving a recovery point objective (RPO) of zero for zone failures.
    
        However, cross-zone replication might introduce a small amount of extra latency. The impact of the latency depends on the application, and for most applications it's negligible.

    - *Zonal*: Because both replicas are in the same zone, no traffic is replicated between zones.

    > [!NOTE]
    > The system replicates log data in real-time to the standby server. Any user errors on the primary server, such as an accidental drop of a table or incorrect data updates, are replicated to the standby replica. So, you can't use the standby to recover from these kinds of errors, and you must perform a point-in-time restore from the backup. For more information, see [Backup and restore](#backup-and-restore).

### Behavior during a zone failure

This section describes what to expect when servers are configured with high availability and availability zone support and there's an availability zone outage.

<!-- TODO planned vs. forced failover -->

- **Detection and response:** Azure periodically checks the health of both the primary and standby servers. After multiple pings, if health monitoring detects that a primary server isn't reachable, the service initiates an automatic failover to the standby server. The health monitoring algorithm uses multiple data points to avoid false positive situations.

    In the event of a zone failure, the behavior is different depending on the availability zone configuration that your server uses:

    - *Zone-redundant:* Azure Database for PostgreSQL automatically detects availability zone failures and initiates failover to the standby server without requiring customer action. To view the possible high availability status types, see [High availability status types](/azure/postgresql/flexible-server/how-to-monitor-high-availability).

    - *Zonal:* If the zone containing a zonal server experiences an outage, both replicas are unavailable. You're responsible for detecting the loss of the zone and performing any failover that you might require, such as manually failing over to a second server you precreated in another zone or region.

- **Notification:** High availability (HA) health status monitoring in Azure Database for PostgreSQL provides a continuous overview of the health and readiness of HA-enabled instances. The monitoring feature is built on top of [Azure Resource Health](/azure/service-health/resource-health-overview), and can detect and alert on any issues that might affect your database's failover readiness or overall availability. By assessing key metrics like connection status, failover state, and data replication health, HA health status monitoring enables proactive troubleshooting and helps maintain your database's uptime and performance.

    For a detailed guide on configuring and interpreting HA health statuses, see [High Availability (HA) health status monitoring for Azure Database for PostgreSQL](/azure/postgresql/flexible-server/how-to-monitor-high-availability).

- **Active requests:** When an availability zone is unavailable, any requests in progress that are connected to an replica in the faulty availability zone are terminated and need to be retried. If your clients handle [transient faults](#resilience-to-transient-faults) appropriately by retrying after a short period of time, they typically avoid significant impact.

- **Expected data loss:** The amount of data loss depends on the availability zone configuration that your server uses.

    - *Zone-redundant:* Zero data loss is expected during zone failover because of synchronous replication between the primary and standby servers in different zones.

    - *Zonal:* Data on servers in the affected zone is unavailable until the zone recovers.

- **Expected downtime:** The amount of downtime depends on the availability zone configuration that your server uses.

    - *Zone-redundant:* Failover typically completes within 60-120 seconds. If your clients handle [transient faults](#resilience-to-transient-faults) appropriately by retrying after a short period of time, they typically avoid significant impact.

    - *Zonal:* When a zone is unavailable, servers in that zone are unavailable until the availability zone recovers.

- **Redistribution:** The traffic rerouting behavior depends on the availability zone configuration that your server uses.

    - *Zone-redundant:* After failover, the former standby server becomes the new primary and begins accepting connections. Azure automatically establishes a new standby server in the original primary zone after it recovers.

    - *Zonal:* When a zone is unavailable, your server is unavailable. If you have a secondary server in another availability zone or region, you're responsible for rerouting traffic to that secondary instance.

### Zone recovery

The zone recovery behavior depends on the availability zone configuration that your server uses.

- *Zone-redundant:* When the availability zone recovers, Azure Database for PostgreSQL automatically rebuilds the standby server in the recovered zone and synchronizes it with the current primary. The recovered zone then serves as the standby location. The service doesn't automatically move the primary role back to the original zone to avoid unnecessary disruption. You can manually initiate a planned failover if you want to return the primary to the original zone.

- *Zonal:* After the zone is healthy, servers in the zone are available again. You're responsible for any zone recovery procedures and data synchronization that your workloads require.

### Test for zone failures

The options for testing for zone failures depend on the availability zone configuration that your instance uses.

- *Zone-redundant:* You can test your application's resilience to failover by using a *forced failover*. A forced failover lets you simulate an unplanned outage scenario while running your workload and observe your application downtime. We recommend running simulations in a non-production environment, or at a quiet time. For more information, see [Initiate a forced failover](/azure/postgresql/high-availability/how-to-configure-high-availability#initiate-a-forced-failover)..

- *Zonal:* While you can't simulate a full zone outage, you can simulate your server being unavailable in a similar way to during a zone outage. For more information, see [Stop compute of a server](/azure/postgresql/configure-maintain/how-to-stop-server).

> [!WARNING]
> **Note to PG:** Do you have any suggestions for other ways to simulate or test a zone failure scenario?

## Resilience to region-wide failures

Although Azure Database for PostgreSQL doesn't have built-in automatic regional failover, it offers robust disaster recovery capabilities that you can configure based on your specific recovery requirements. 

The service supports geo-redundant backup storage that replicates your backup data to an Azure paired region, enabling *geo-redundant backup and restore* if the primary region becomes unavailable. Additionally, you can deploy *cross-region read replicas* to maintain a synchronized copy of your database in a different region for faster recovery.

### Geo-redundant backup and restore

Geo-redundant backup and restore give you the ability to restore your server in a different region if a disaster occurs. It also provides at least 99.99999999999999 percent (16 nines) durability of backup objects over a year.

You can only configure geo-redundant backup when you create the server. When you configure the server with geo-redundant backup, the backup data and transaction logs are copied to the paired region asynchronously through storage replication.

For more information on geo-redundant backup and restore, see [geo-redundant backup and restore](/azure/postgresql/flexible-server/concepts-backup-restore#geo-redundant-backup-and-restore).

#### Requirements

- **Region support:** Geo-redundant backup storage is available for servers in paired regions that support geo-redundant storage (GRS).  For more information on Azure paired regions, see [Azure paired regions](./regions-paired.md). To learn about GRS support, see [Azure Storage redundancy](/azure/reliability/reliability-storage-blob#multi-region-support).

- **Pricing tier**: All pricing tiers support geo-redundant backups and cross-region read replicas.

#### Configure multi-region support

- **Create**: To learn how to create geo-redundant backups, see [Backup and restore concepts](/azure/postgresql/flexible-server/concepts-backup-restore). 

- **Migrate**: Geo-redundant backup configuration can't be changed after server creation

#### Considerations

- **Recovery Point Objective (RPO)**: Geo-redundant backups may have an RPO of several hours depending on the backup schedule. Cross-region read replicas typically have an RPO of up to 5 minutes under normal conditions.
- **Recovery Time Objective (RTO)**: Geo-restore operations can take significant time depending on database size. Read replica promotion offers much faster RTO, typically within minutes.
- **Connection string updates**: Applications must update connection strings when failing over to a different region, unless using virtual endpoints.


#### Cost

Geo-redundant backup storage approximately doubles your backup storage costs because data is replicated to a second region. For detailed pricing information, see [Azure Database for PostgreSQL pricing](https://azure.microsoft.com/pricing/details/postgresql/flexible-server/).


#### Behavior when all regions are healthy

- **Cross-region operation:** In normal operations, all database traffic is directed to the primary region. The geo-redundant backup feature doesn't route traffic between regions - it only ensures backup data protection. All read and write operations remain on the primary server in the primary region.

- **Cross-region data replication:** Geo-redundant backup uses asynchronous storage-level replication to copy backup data and transaction logs to the Azure paired region. Daily backups are automatically copied to the paired region as part of the managed backup process. Transaction logs (WAL files) are also continuously replicated to ensure comprehensive backup coverage. This replication happens transparently in the background with up to one hour of delay for backup data transmission to the paired region.


**Sources:**
- [Backup and restore in Azure Database for PostgreSQL](/azure/postgresql/flexible-server/concepts-backup-restore) - Geo-redundant backup configuration and normal operations
- [Geo-disaster recovery in Azure Database for PostgreSQL](/azure/postgresql/flexible-server/concepts-geo-disaster-recovery) - Backup replication behavior and regional failover procedures


#### Behavior during a region failure

- **Detection and response**: Microsoft detects region-wide outages and makes geo-redundant backups available in the paired region. Customer must initiate the geo-restore process.
- **Active requests**: All active connections and transactions in the failed region are lost and must be retried.
- **Expected data loss**: RPO depends on when the last backup was replicated to the paired region, potentially several hours.
- **Expected downtime**: RTO includes the time to restore from backup in the destination region, which can be substantial for large databases.
- **Redistribution**: Customer must update application connection strings to point to the newly restored server in the paired region.

#### Region recovery

When the primary region recovers, you must manually configure and sync any changes made during the outage if you want to return operations to the original region.

#### Test for region failures

- **Planned testing**: Regularly test your geo-restore procedures to ensure they meet your RTO and RPO requirements.
- **Application testing**: Verify that your applications can handle connection string changes and properly redirect traffic to a different region.
- **End-to-end validation**: Conduct full disaster recovery drills that include data validation, application functionality testing, and rollback procedures.


<!--Feature: Cross-region read replicas-->
### Cross-region read replicas

You can deploy cross region read replicas to protect your databases from region-level failures. Read replicas are updated asynchronously by using PostgreSQL's physical replication technology, and they can lag the primary. General purpose and memory optimized compute tiers support read replicas.

For more information on read replica features and considerations, see [Read replicas](/azure/postgresql/flexible-server/concepts-read-replicas).

#### Requirements

- **Region support**: Cross-region read replicas can be created in any region where Azure Database for PostgreSQL is available, not just paired regions. For the complete list of supported regions, see [Azure regions](https://azure.microsoft.com/global-infrastructure/services/).
- **Cross-region read replicas**: Can be configured after server creation. The primary server must be running and accessible.
- **Pricing tier**: All pricing tiers support cross-region read replicas.

#### Configure multi-region support

- **Create**: To learn how to create cross-region read replicas, see [Read replicas in Azure Database for PostgreSQL](/azure/postgresql/flexible-server/concepts-read-replicas#create-a-replica). Replicas can be configured after server creation. The primary server must be running and accessible.

- **Migrate**:  Read replicas can be added to existing servers through the Azure portal, CLI, or REST API.  However, moving read replicas to another resource group after their creation is unsupported. Additionally, moving replicas to a different subscription, and moving the primary that has read replicas to another resource group or subscription, is not supported.

#### Considerations

- **Recovery Point Objective (RPO)**:  Cross-region read replicas typically have an RPO of up to 5 minutes under normal conditions.
- **Recovery Time Objective (RTO)**: Read replica promotion occurs typically within minutes.
- **Configuration differences**: Read replicas may not inherit all configuration settings from the primary server. Plan to configure necessary settings post-failover.
- **Connection string updates**: Applications must update connection strings when failing over to a different region, unless using virtual endpoints.

#### Cost

Cross-region read replicas incur full compute and storage costs in the destination region, plus data transfer charges for replication. For detailed pricing information, see [Azure Database for PostgreSQL pricing](https://azure.microsoft.com/pricing/details/postgresql/flexible-server/).

#### Behavior when all regions are healthy

**Traffic routing between regions**: In normal operations, all database traffic is directed to the primary region. Cross-region read replicas can optionally serve read-only workloads to reduce latency for globally distributed applications or to offload read traffic from the primary server.

**Data replication between regions**: Cross-region read replicas use asynchronous replication to minimize impact on primary server performance. Backup data is copied to the paired region as part of the automated backup process, typically within hours of the backup being created.

#### Behavior during a region failure

- **Detection and response**: Customer detects regional outage and manually promotes a read replica to become a standalone read-write server.
- **Active requests**: All active connections to the primary region are lost. New connections must be directed to the promoted replica.
- **Expected data loss**: RPO is typically up to 5 minutes under normal conditions, potentially longer during severe regional failures.
- **Expected downtime**: RTO is typically within minutes for the promotion process, plus time to redirect application traffic.
- **Traffic rerouting**: Customer must update application connection strings to point to the promoted read replica, unless using virtual endpoints.

#### Region recovery

After the primary region recovers, you can establish a new read replica from the promoted server back to the original region, then perform another promotion to return operations to the preferred region.

#### Test for region failures

- **Planned testing**: Regularly test read replica promotion procedures to ensure they meet your RTO and RPO requirements.
- **Application testing**: Verify that your applications can handle connection string changes and properly redirect traffic to a different region.
- **End-to-end validation**: Conduct full disaster recovery drills that include data validation, application functionality testing, and rollback procedures.

### Custom multi-region solutions for resiliency

If you need multi-region resilience beyond the built-in geo-redundant backup and read replica capabilities, you can deploy multiple independent Azure Database for PostgreSQL instances across regions with application-level data synchronization. Consider these architectural patterns:

- **Active-passive with application-level failover**: Deploy primary and secondary database instances with application logic to handle failover and data synchronization.
- **Active-active with data partitioning**: Distribute data across multiple regions based on geographic or functional boundaries.
- **Hybrid approaches**: Combine Azure Database for PostgreSQL read replicas with application-level logic for more complex scenarios.

For detailed architectural guidance, see:
- [Geo-disaster recovery in Azure Database for PostgreSQL](/azure/postgresql/flexible-server/concepts-geo-disaster-recovery)


<!-- TODO consider whether to include this:
You can also use either of the following customer-managed data migration methods to replicate data to a nonpaired region: 

- [Dump and restore](/azure/postgresql/migrate/how-to-migrate-using-dump-and-restore)
- [Logical replication and logical decoding](/azure/postgresql/flexible-server/concepts-logical)
-->

**Sources:**
- [Geo-disaster recovery in Azure Database for PostgreSQL](/azure/postgresql/flexible-server/concepts-geo-disaster-recovery) - Regional failover options and procedures
- [Geo-replication in Azure Database for PostgreSQL](/azure/postgresql/flexible-server/concepts-read-replicas-geo) - Cross-region read replica configuration and management

## Backup and restore
<!-- TODO -->

Azure Database for PostgreSQL automatically performs backups that provide point-in-time recovery capabilities. Backups are fully managed by Microsoft and include both full backups and transaction log backups stored in zone-redundant storage where available.

The service automatically creates server backups and stores them in the region's zone-redundant storage (ZRS) if the region supports availability zones, or locally redundant storage (LRS) otherwise. The default backup retention period is 7 days, with the option to extend retention up to 35 days. All backups are encrypted using AES 256-bit encryption.

You can configure geo-redundant backup storage at server creation time to replicate backups to the Azure paired region for additional protection against regional failures. This option provides at least 99.99999999999999% (16 nines) durability of backup objects over a year.

Point-in-time restore allows you to restore your database to any moment within the backup retention period, creating a new server with the restored data. This capability is useful for recovering from accidental data modifications, application errors, or testing scenarios.
 
For more information, see [Backup and restore in Azure Database for PostgreSQL](/azure/postgresql/flexible-server/concepts-backup-restore).

## Resilience to service maintenance

Azure Database for PostgreSQL automatically handles critical servicing tasks including patching of the underlying hardware, operating system, and database engine. The service includes security updates, software updates, and minor version upgrades as part of planned maintenance.

To ensure your server remains available during maintenance windows, follow thwse recommendations:

- **Enable high availability**: During maintenance, the server may need to be restarted as part of the update process. If you have high availability enabled, maintenance operations typically use rolling updates to minimize downtime. Periodic maintenance activities such as minor version upgrades happen on the standby replica first. To reduce downtime, the standby is promoted to primary so that workloads can keep on while the maintenance tasks are applied on the remaining node. This sequencing applies whether your server uses zone-redundant or zonal high availability.

    For servers without high availability enabled, expect brief downtime during maintenance operations. With high availability enabled, maintenance operations typically complete with minimal or no downtime.

- **Configure custom maintenance windows**: You can configure the maintenance schedule to be system-managed or define a custom maintenance window to minimize the impact on your business operations. Schedule maintenance during low-activity periods to minimize business impact. For more information, see [Schedule maintenance](/azure/postgresql/configure-maintain/how-to-configure-scheduled-maintenance).

- **Implement retry logic:** Ensure your applications can handle brief connectivity interruptions that may occur during maintenance restarts. To make your applications resilient to these types of problems, see [Resilience to transient faults](#resilience-to-transient-faults) guidance.

## Service-level agreement

[!INCLUDE [SLA description](includes/reliability-service-level-agreement-include.md)]

Azure Database for PostgreSQL provides different availability SLAs based on the server's configuration:

- Servers configured with zone-redundant high availability.
- Servers configured with zonal (single-zone) high availability.
- Servers configured without high availability.

### Related content

- [Azure reliability](./overview.md)
- [Architecture best practices for Azure Database for PostgreSQL](/azure/well-architected/service-guides/postgresql)
- [Overview of business continuity with Azure Database for PostgreSQL](/azure/postgresql/backup-restore/concepts-business-continuity)
