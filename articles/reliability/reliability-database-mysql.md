---
title: Reliability in Azure Database for MySQL
description: Learn how to make Azure Database for MySQL resilient to various potential outages and problems, including transient faults, availability zone outages, region outages, and service maintenance, and learn about backup and restore.
ms.author: maghan
author: markingmyname
ms.topic: reliability-article
ms.custom: subject-reliability
ms.service: azure-database-mysql	
ms.subservice: flexible-server
ms.date: 03/27/2026
ai.usage: ai-assisted
---

# Reliability in Azure Database for MySQL

[Azure Database for MySQL](/azure/mysql/flexible-server/overview) is a fully managed database service that gives you granular control and flexibility over database management functions and configuration settings. The service provides high availability (HA) and disaster recovery (DR) capabilities based on your requirements.

[!INCLUDE [Shared responsibility](includes/reliability-shared-responsibility-include.md)]

This article describes how to make Azure Database for MySQL resilient to various potential outages and problems, including transient faults, availability zone outages, region outages, and service maintenance. It also describes how you can use backups to recover from other types of problems and highlights some key information about the Azure Database for MySQL service-level agreement (SLA).

## Production deployment recommendations

The Azure Well-Architected Framework provides recommendations across reliability, security, cost, operations, and performance. To understand how these areas influence each other and contribute to a reliable Azure Database for MySQL solution, see [Architecture best practices for Azure Database for MySQL](/azure/well-architected/service-guides/azure-database-for-mysql).

## Reliability architecture overview

[!INCLUDE [Introduction to reliability architecture overview section](includes/reliability-architecture-overview-introduction-include.md)]

### Logical architecture

When you work with Azure Database for MySQL, you deploy a *server*, which represents the compute and storage resources required to support your database server. You deploy one or more *databases* to the server.

You can deploy servers in the Burstable, General Purpose, and Memory Optimized *compute tiers*. Each compute tier is [optimized for different kinds of workloads](/azure/mysql/flexible-server/concepts-service-tiers-storage).

For more information about the general service architecture and deployment models, see [Azure Database for MySQL overview](/azure/mysql/flexible-server/overview).

### Physical architecture

- **Compute and storage separation:** Azure Database for MySQL uses a compute and storage separation architecture to support HA. The database engine runs on a virtual machine (VM). Data files are stored in Azure Storage, which synchronously maintains three copies of the data to protect against storage hardware failures. Depending on the server's HA configuration, data files can be stored in zone-redundant storage (ZRS) or local-redundant storage (LRS).

- **HA:** You can optionally enable a *HA configuration* on your server. When you enable the HA configuration, the service provisions and maintains a warm standby replica server. Data changes on the primary server are synchronously replicated to the standby replica server to ensure zero data loss during a failure of the primary server.

    The architecture separates the compute layer from the storage layer, which allows the service to handle different types of failures appropriately. For higher resiliency, you can spread the servers across availability zones.

    A standby replica server is deployed in the same VM configuration as the primary server, including vCores, storage, and network settings.

    You can switch between servers by performing a *failover*. Use *unplanned failovers* when the primary server fails, and use *planned failovers* when you need to minimize application downtime during a failover.

    For more information, see [HA in Azure Database for MySQL](/azure/mysql/flexible-server/concepts-high-availability).

- **Backups:** Azure Database for MySQL automatically creates server backups. For more information, see [Backup and restore](#backup-and-restore).

## Resilience to transient faults

[!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)]

Your applications must handle transient connectivity errors that can occur during maintenance, scaling operations, or network interruptions. Follow these recommendations:

> [!div class="checklist"]
> - When your application encounters transient faults, retry the operation by using exponential backoff. Increase the delay between retries and limit the number of attempts. If the operation continues to fail after the maximum number of retries, treat it as a failure.
>
> - When possible, use [client libraries](/azure/mysql/flexible-server/concepts-connection-libraries) that automatically handle retries.
>
> - Transient errors that occur during write operations require more careful consideration. Consider making your write operations idempotent so that you can run them multiple times.

## Resilience to availability zone failures

[!INCLUDE [Resilience to availability zone failures](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)]

Select your type of availability zone support through the HA configuration. When you enable HA, Azure Database for MySQL deploys a *standby replica server* alongside your primary server. This HA model helps ensure that committed data is never lost during failures. Whichever HA deployment model you choose, the service synchronously commits data to both the primary and standby replica servers. If the primary server is disrupted, the server automatically fails over to the standby replica server.

The service stores data on Azure Files premium storage. Depending on your server's HA configuration, it either uses ZRS or LRS, which stores three data copies within or across availability zones.

Azure Database for MySQL supports two availability zone configuration types when you use HA:

- **Zone-redundant HA:** Zone redundancy provides the highest level of zone resilience by deploying a primary server in one availability zone and a standby replica server in a different availability zone. The standby replica server uses similar compute, storage, and network configuration to the primary server. A zone-redundant configuration provides physical isolation of the entire stack between primary and standby servers.

   You select the availability zones for the primary and standby servers.

    Use zone-redundant deployments for production servers.

    :::image type="complex" source="./media/reliability-database-mysql/zone-redundant.svg" alt-text="Diagram that shows a zone-redundant server, with the primary and standby servers in different availability zones." border="false" lightbox="./media/reliability-database-mysql/zone-redundant.svg":::
        The diagram consists of three sections. The first section lists client applications like Azure VMs, Azure Kubernetes Service (AKS), and Azure App Service. The next section is labeled availability zone 1. It contains the primary server and two subsections, compute and storage, separated by a dotted line. The third section is labeled availability zone 2. It mirrors availability zone 1, but it contains the standby server. Arrows point from the client applications to a standard load balancer. One arrow points from the load balancer to a VM on the primary server in availability zone 1. Bidirectional arrows connect the VM and premium file share and solid-state drive (SSD). Another arrow points from premium file share to backup blob storage. An arrow labeled storage-based asynchronous replication points from availability zone 1 to availability zone 2.
    :::image-end:::

    Write operations can experience a small increase in commit latency because the service synchronously replicates data to the standby server. On average, you can expect 5% to 10% increased latency for application writes and commits. The impact varies by workload, selected SKU, and region.

- **Local-redundant HA:** The primary and standby servers use the same availability zone. If a disruption occurs to the primary server, but the zone is still healthy, the server automatically fails over to the standby server.

    A local-redundant deployment gives you HA within a single availability zone. It protects you against node-level failures and also helps reduce application downtime during planned and unplanned downtime events. However, it doesn't protect against an outage in that zone. In regions that have availability zones, this kind of configuration is also sometimes called *zonal* or *single-zone*.

    :::image type="complex" source="./media/reliability-database-mysql/local-redundant.svg" alt-text="Diagram showing a local-redundant server, with the primary and standby servers in the same availability zone." border="false" lightbox="./media/reliability-database-mysql/local-redundant.svg":::
        The diagram consists of three sections. The first section lists client applications like Azure VMs, AKS, and App Service. The next section is labeled availability zone 1. It contains two separate sections for the primary server and the standby server. A dotted line divides each server's section into two subsections, compute and storage. Arrows point from the client applications to a standard load balancer. One arrow points from the load balancer to a VM on the primary server in availability zone 1. Bidirectional arrows connect the VM and premium file share and solid-state drive (SSD). Another arrow points from premium file share to backup blob storage. An arrow labeled storage-based asynchronous replication points from the primary server to the standby server. The third section, labeled availability zone 2, is empty.
    :::image-end:::
    
    Use local-redundant HA only in the following scenarios:

    - When you have unusually latency-sensitive applications, you validate the need to minimize latency between your primary and secondary replica, and you plan for zone resilience by using other architectural approaches.

    - When you deploy to a region that doesn't support availability zones. In this scenario, the region functions as a single zone, which makes local-redundant HA the only HA option.

    The servers are in the same zone, which can reduce the write latency to applications that you deploy within that zone.

If you configure your server without HA, then it runs on a single server. If that server or its zone fails, your server is unavailable.

### Requirements

- **Region support:** Azure Database for MySQL supports different availability zone configurations depending on your Azure region. For a complete list of regions, including types of availability zone support and specific considerations for each region, see [Azure regions](/azure/mysql/flexible-server/overview#azure-regions).

- **Service tier:** HA requires General Purpose or Memory Optimized tiers. The Burstable tier doesn't support HA (zone-redundant or local-redundant).

### Cost

When you enable HA, you create and pay for the standby server at the same rate as the primary server. The availability zone configuration doesn't affect the cost. Data replication incurs no charges within or between availability zones. Depending on your backup storage volume, you might also be billed for backup storage. For detailed pricing information, see [Azure Database for MySQL pricing](https://azure.microsoft.com/pricing/details/mysql/).

### Considerations

- **Primary keys:** Use primary keys on all tables because this approach reduces replication and failover time.

- **Limitations and known problems:** Review the list of [limitations](/azure/mysql/flexible-server/concepts-high-availability#limitations) and [known problems](/azure/mysql/flexible-server/concepts-high-availability#known-problems).

### Configure availability zone support

To configure availability zone support for a server, configure the HA settings.

> [!NOTE]
> [!INCLUDE [Availability zone numbering](./includes/reliability-availability-zone-numbering-include.md)]

- **Create a zone-redundant server.** To learn how to create a server with HA and zone redundancy enabled, see the following articles:

    - Azure portal: [Manage zone-redundant HA in Azure Database for MySQL by using the Azure portal](/azure/mysql/flexible-server/how-to-configure-high-availability#enable-high-availability-during-server-creation).

    - Azure CLI: [Manage zone-redundant HA in Azure Database for MySQL by using the Azure CLI](/azure/mysql/flexible-server/how-to-configure-high-availability-cli#enable-high-availability-during-server-creation).

- **Create a local-redundant server.** To create a server with local-redundant HA in a single availability zone, you must use the Azure CLI or another programmatic deployment method. For the Azure CLI instructions, see [Enable HA during server creation](/azure/mysql/flexible-server/how-to-configure-high-availability-cli#enable-high-availability-during-server-creation).

- **Change the availability zone configuration for existing servers.** If you have an existing server, the approach that you follow to enable availability zone support depends on the server's initial configuration.

    - To change an existing server to zone-redundant HA, you need to migrate to a new server. For more information, see [Migrate from an existing server to a zone-redundant server](/azure/mysql/flexible-server/concepts-high-availability#migrate-from-an-existing-server-to-a-zone-redundant-server).

    - To change an existing server to local-redundant HA:

      1. [Disable HA](/azure/mysql/flexible-server/how-to-configure-high-availability#disable-high-availability), if it's enabled.

      1. Enable local-redundant HA. You must use the Azure CLI or another programmatic deployment method. For Azure CLI instructions, see [Manage zone-redundant HA in Azure Database for MySQL by using the Azure CLI](/azure/mysql/flexible-server/how-to-configure-high-availability-cli).

- **Disable HA.** Disabling HA removes the standby replica server, so your server isn't resilient to zone-level outages. However, if geo-redundant backups are enabled, you can still recover the server in a different region by using those backups. For more information, see [Disable HA](/azure/mysql/flexible-server/how-to-configure-high-availability#disable-high-availability).

### Behavior when all zones are healthy

This section describes what to expect when you configure servers with HA and availability zone support, and all availability zones are operational.

- **Cross-zone operation:** MySQL client applications connect to the primary server by using the database server's fully qualified domain name (FQDN). Avoid using the IP address of the primary server because the IP address can change, including during failovers.

    Azure Database for MySQL uses an active-passive configuration in which the primary server handles all database connections and queries in the primary availability zone. The standby replica server doesn't serve client traffic during normal operations.

- **Cross-zone data replication:** Writes are committed on the primary server and synchronously written to logs for the standby server by using ZRS. The primary server doesn't wait for the standby server to apply the logs, but because the logs are in ZRS, they're available even if a replica or zone failure occurs.
    
    The effects of replication are different depending on the availability zone configuration that your server uses:

    - *Zone-redundant:* Because the servers are in separate zones, this approach ensures zero data loss during a zone failure. This situation is also known as achieving a recovery point objective (RPO) of zero for zone failures.
    
        However, cross-zone replication might introduce a small amount of extra latency. On average, you can expect 5% to 10% increased latency for application writes and commits, but the impact varies by workload, selected SKU, and region.

    - *Local-redundant:* No traffic is replicated between zones.

    > [!NOTE]
    > The system replicates all changes in real time to the standby replica server, including unintended user errors like an accidental drop of a table or incorrect data updates. Because of the immediate replication, you can't use the standby replica for recovery. To recover from user errors, you must perform a point‑in‑time restore from a backup. For more information, see [Backup and restore](#backup-and-restore).

### Behavior during a zone failure

This section describes what to expect when you configure servers with HA and availability zone support, and there's an availability zone outage.

- **Detection and response:** Azure periodically checks the health of both the primary and standby servers. After multiple pings, if health monitoring detects that a primary server isn't reachable, the service initiates an automatic failover to the standby server. The health monitoring algorithm uses multiple data points to avoid false positive situations.

    If a zone failure occurs, the behavior differs depending on the availability zone configuration that your server uses:

    - *Zone-redundant:* Azure Database for MySQL automatically detects availability zone failures by continuously monitoring multiple server endpoints. For more information, see [How automatic failover detection works in HA-enabled servers](/azure/mysql/flexible-server/concepts-high-availability#how-automatic-failover-detection-works-in-ha-enabled-servers).
    
        To see the possible HA status types, see [Monitor HA](/azure/mysql/flexible-server/concepts-high-availability#monitor-high-availability). When a zone fails, Azure initiates an [unplanned failover](/azure/mysql/flexible-server/concepts-high-availability#unplanned-automatic-failover) to the standby server without requiring you to take action.

    - *Local-redundant:* Both primary and standby servers are unavailable if the availability zone that hosts a local-redundant server becomes unavailable. In this scenario, the service doesn't provide automatic failover. You're responsible for detecting the zone outage and performing recovery actions, such as restoring zone‑redundant backups to a separate server in another availability zone or region.

- **Notification:** [!INCLUDE [Availability zone down notification partial bullet (Service Health and Resource Health)](./includes/reliability-availability-zone-down-notification-service-resource-partial-include.md)]

    Azure Database for MySQL generates an Azure Resource Health event when an unplanned failover occurs.

- **Active requests:** When an availability zone becomes unavailable, in‑progress requests to servers in the affected zone might terminate. Applications must retry these requests. If your clients handle [transient faults](#resilience-to-transient-faults) appropriately by retrying after a short period of time, they typically avoid significant impact.

- **Expected data loss:** The amount of data loss depends on your server's availability zone configuration.

    - *Zone-redundant:* Zero data loss is expected during zone failover because of synchronous replication between the primary and standby servers in different zones.

    - *Local-redundant:* Data on servers in the affected zone is unavailable until the zone recovers.

- **Expected downtime:** The amount of downtime depends on the availability zone configuration that your server uses.

    - *Zone-redundant:* Failover typically completes within 60 to 120 seconds. If your clients handle [transient faults](#resilience-to-transient-faults) appropriately by retrying after a short period of time, they typically avoid significant impact.

    - *Local-redundant:* Servers in an affected zone are unavailable until the availability zone recovers.

- **Redistribution:** The traffic rerouting behavior depends on the availability zone configuration that your server uses.

    - *Zone-redundant:* After failover, the standby server becomes the new primary server and begins to accept new connections. Azure automatically establishes a standby server in the original primary zone after it recovers. For more information, see [Unplanned failover](/azure/mysql/flexible-server/concepts-high-availability#unplanned-automatic-failover).

    - *Local-redundant:* When a zone is unavailable, your server is unavailable. If you have a separate server that you precreated in another availability zone or region, you're responsible for rerouting traffic to that server.

### Zone recovery

The zone recovery behavior depends on the availability zone configuration that your server uses.

- *Zone-redundant:* When the availability zone recovers, Azure Database for MySQL automatically rebuilds the standby server in the recovered zone and syncs it with the current primary server. The recovered zone then serves as the standby location. The service doesn't automatically move the primary role back to the original zone to avoid unnecessary disruption. If you want to return the primary to the original zone, you can [manually initiate a planned failover](/azure/mysql/flexible-server/concepts-high-availability#planned-forced-failover).

- *Local-redundant:* After the zone is healthy, servers in the zone are available again. You're responsible for zone recovery procedures and data synchronization that your workloads require.

### Test for zone failures

The options for testing for zone failures depend on the availability zone configuration that your instance uses.

- *Zone-redundant:* You can test your application's resilience to failover by initiating a planned failover. Use a planned failover to simulate an unplanned outage while your workload runs and observe your application's downtime. Run simulations in nonproduction environments, or at a quiet time. For more information, see [Planned failover](/azure/mysql/flexible-server/concepts-high-availability#planned-forced-failover).

- *Local-redundant:* You can't simulate a full zone outage, but you can simulate your server being unavailable in a similar way to what occurs during a zone outage. For more information, see the following articles:

    - Azure portal: [Stop and start an Azure Database for MySQL flexible server instance](/azure/mysql/flexible-server/how-to-stop-start-server-portal).

    - Azure CLI: [Restart, stop, and start an Azure Database for MySQL flexible server instance](/azure/mysql/flexible-server/how-to-restart-stop-start-server-cli).

## Resilience to region-wide failures

Azure Database for MySQL supports *cross-region read replicas*, which you can use to maintain a synced copy of your database in a different region for faster recovery.

You can also use geo-redundant backups, in supported regions, to provide cross-region recovery. However, backups typically involve more downtime and data loss than replication. For more information, see [Backup and restore](#backup-and-restore).

### Cross-region read replicas

Deploy read replicas to protect your databases from region-level failures. Each read replica is a separate Azure Database for MySQL server. When you place a read replica in a second Azure region, your database server can provide resilience to a region-wide problem. You can deploy up to 10 read replicas, which can optionally be in different Azure regions.

MySQL physical replication technology updates read replicas asynchronously from the source server in the primary region, which means that replicas can lag behind the source. Cross-region read replicas can optionally serve read-only workloads to reduce latency for globally distributed applications or to offload read traffic from the source server. For more information about read replica features and considerations, see [Read replicas](/azure/mysql/flexible-server/concepts-read-replicas).

:::image type="complex" source="./media/reliability-database-mysql/read-replica.svg" alt-text="Diagram that shows a read replica in a secondary Azure region, with the application directing read-write traffic to the source server." border="false":::
    The diagram consists of a primary region, a secondary region, and an icon labeled application. A solid arrow points from the application icon to the primary server in the primary region. A dotted arrow labeled asynchronous replication points from the primary server to the read replica in the secondary region.
:::image-end:::

If your primary region fails, you can manually fail over to make your secondary replica the primary server. To manually fail over, you stop the replication process, which promotes the read replica to a read-write server. Because of the asynchronous replication, failover can result in data loss. Your application needs to connect to the new primary server, and you're responsible for application reconfiguration.

:::image type="complex" source="./media/reliability-database-mysql/read-replica-failure.svg" alt-text="Diagram that shows a read replica in a second Azure region that has been failed over to become the primary server, with the application now directing read-write traffic to the secondary region." border="false":::
    The diagram consists of a primary region, a secondary region, and an icon labeled application. An x inside of a circle over the primary server in the primary region represents a failure in this region. A solid arrow points from the application icon to the failed-over primary server in the secondary region.
:::image-end:::

> [!NOTE]
> This section summarizes some of the important information about how read replicas can support resilience to region-wide failures. You can also use [read replicas](/azure/mysql/flexible-server/concepts-read-replicas) to improve performance and support high-scale geographically distributed user bases.

#### Requirements

- **Region support:** You can create cross-region read replicas in any region that supports Azure Database for MySQL. You aren't limited to Azure paired regions.

- **Compute tiers:** The General Purpose and Memory Optimized compute tiers support read replicas. The Burstable tier doesn't support read replicas.

#### Considerations

- **Configuration differences:** When you create a replica, it inherits several settings from the source server, including the compute generation, vCores, and storage. You can customize these values on the read replica after you create it, but it's best to use equal or greater values to ensure that the replica can keep up with changes in the source.

- **Monitoring replication lag:** The asynchronous replication process requires a replication lag, which can vary depending on several factors. When the replication lag is very high, your server might experience problems. It's important to monitor the replication lag so that you can mitigate problems before they escalate. For more information, see [Monitor replication](/azure/mysql/flexible-server/concepts-read-replicas#monitor-replication).

- **HA:** Read replicas can't have HA enabled, and when they're failed over to become the primary server, they also don't have HA. You're responsible for configuring HA after failing over to a replica.

#### Cost

Read replicas incur compute and storage costs, as well as cross-region data transfer charges for replication. For detailed pricing information, see [Azure Database for MySQL pricing](https://azure.microsoft.com/pricing/details/mysql/) and [Bandwidth pricing](https://azure.microsoft.com/pricing/details/bandwidth/).

#### Configure multi-region support

- **Create a read replica:** To learn how to create a read replica, see the following articles:

    - Azure portal: [How to create and manage read replicas in Azure Database for MySQL flexible server by using the Azure portal](/azure/mysql/flexible-server/how-to-read-replicas-portal)

    - Azure CLI: [How to create and manage read replicas in Azure Database for MySQL flexible server by using the Azure CLI](/azure/mysql/flexible-server/how-to-read-replicas-cli)
    
    You can configure replicas after you create the source server only if the source server is running and reachable.

- **Stop replication:** To learn how to stop replication, see [Stop replication to a replica server](/azure/mysql/flexible-server/how-to-read-replicas-portal#stop-replication-to-a-replica-server).

- **Delete a read replica:** To learn how to delete a read replica, see [Delete a replica server](/azure/mysql/flexible-server/how-to-read-replicas-portal#delete-a-replica-server).

#### Behavior when all regions are healthy

This section describes what to expect when you configure your server with a read replica in another region, and all regions are operational:

- **Traffic routing between regions:** During normal operations, your application should direct read-write traffic to the source server in the primary region. You can optionally direct read requests to your read replica.

- **Data replication between regions:** Cross-region read replicas use asynchronous replication to minimize the impact on source server performance. The amount of replication lag depends on several factors, including the write load and the latency between the source server and replicas. Replication lag is typically at least several minutes, but it can be much longer. For more information, see [Monitor replication](/azure/mysql/flexible-server/concepts-read-replicas#monitor-replication), and for detailed instructions, see [Monitor replication in the Azure portal](/azure/mysql/flexible-server/how-to-read-replicas-portal#monitor-replication).

#### Behavior during a region failure

This section describes what to expect when you configure a server for cross-region read-replica support, and there's an outage in the primary region.

- **Detection and response:** You're responsible for detecting an outage in the primary region and manually triggering a failover. This action can result in the loss of unreplicated data.

    > [!IMPORTANT]
    > You're responsible for triggering failover. Azure doesn't fail over to read replicas automatically, even if a region failure occurs.

    Failover requires you to take the following steps:

    1. Stop replication. This procedure is irreversible, and the server can't be made into a replica again. The process results in data loss. For more information about the implications of this action, see [Stop replication](/azure/mysql/flexible-server/concepts-read-replicas#stop-replication).

    1. Reconfigure your application to use the new primary server.

    For more information, see [Failover](/azure/mysql/flexible-server/concepts-read-replicas#failover).

- **Notification:** [!INCLUDE [Region down notification partial bullet (Azure Service Health only)](./includes/reliability-region-down-notification-service-partial-include.md)]

- **Active requests:** All active connections to the source region are dropped if the source server is unavailable. Applications need to retry making connections to the new primary server after the failover process completes.

- **Expected data loss:** During a region outage, you must perform a failover that stops replication. This process results in permanent loss of unreplicated data.

    The amount of data loss depends on the replication lag at the time of the outage. Replication lag is typically at least several minutes, but it can be much longer. For more information, see [Monitor replication](/azure/mysql/flexible-server/concepts-read-replicas#monitor-replication).

- **Expected downtime:** Stopping replication typically completes within two minutes after you trigger the action. You're responsible for reconfiguring your applications to connect to the new primary server. The time it takes for you to perform the reconfiguration also contributes to your overall downtime.

- **Traffic rerouting:** You're responsible for reconfiguring your applications to connect to the new primary server.

    > [!NOTE]
    > After you fail over a read replica to become the primary server, the server doesn't have HA enabled. You must enable HA manually or include it in your automation.

#### Region recovery

When the region recovers, you're responsible for failback activities to resume operations in the primary region. Microsoft doesn't automatically move the primary server. You can create a new read replica in the primary region, then perform another failover process to restore operations in the primary region. Consider one of the following approaches, depending on whether your application can tolerate downtime or data loss:

- Take your application offline and wait for the replication to catch up with all of the changes. This approach requires application downtime that's approximately the same as the replication lag.

- Perform the failover and accept the loss of unreplicated data.

Remember that you're also responsible for reconfiguring your applications to connect to the new primary server as required.

#### Test for region failures

Regularly test read replica failover procedures to ensure that your processes are valid and that the capabilities meet your recovery time objective (RTO) and RPO requirements.

You can fail over a read replica to become the primary server at any time, even when all regions are healthy. We recommend that you perform these tests in a nonproduction environment because it can cause data loss and requires manual failback.

As part of your DR strategy, regularly run full recovery drills. These drills include data validation, application functionality testing, and documented rollback procedures.

## Backup and restore

Azure Database for MySQL automatically backs up your data, so you can restore it to any point in time within the backup retention period. This protection helps you avoid accidental corruption and deletion of data. Microsoft fully manages the backups without interrupting the server's availability. The backups include both full backups and transaction log backups.

- **Backup storage:** If you configure the server with zone-redundant HA, the system stores backups in ZRS. For servers configured without HA or with local-redundant HA, the system stores backups in LRS.

    In Azure regions that have pairs, you can configure geo-redundant storage (GRS) for backups when you create the server. This approach replicates backups to the Azure paired region for extra protection against region failures. The system replicates backups asynchronously.

    The default backup retention period is 7 days, but you can extend retention up to 35 days. All backups are encrypted.

- **Restore:** Point-in-time restore (PITR) lets you restore your database to any moment within the backup retention period. The restore process creates a new database server with a new user-provided server name. You can use the new server as-is or copy data from it.

    When you restore a geo-redundant backup, you create a new server in the paired region. In some regions, you can use Universal Geo-Restore to restore a geo-redundant backup to a region that isn't your primary region's paired region.

    Use this capability to recover from accidental data modifications, application errors, or testing scenarios.

[!INCLUDE [Backups description](includes/reliability-backups-include.md)]

For more information, see [Backup and restore in Azure Database for MySQL](/azure/mysql/flexible-server/concepts-backup-restore).

## Resilience to service maintenance

Azure Database for MySQL automatically handles critical servicing tasks, including patching of the underlying hardware, operating system, and database engine. The service includes security updates, software updates, and minor version upgrades as part of planned maintenance. For more information, see [Scheduled maintenance in Azure Database for MySQL](/azure/mysql/flexible-server/concepts-maintenance).

To ensure that your server remains available during maintenance windows, follow these recommendations:

> [!div class="checklist"]
> - **Avoid management operations during maintenance periods.** Don't perform server management operations while maintenance is underway because these operations can affect your server's reliability.
>
> - **Use near-zero-downtime maintenance.** If your server has HA enabled and meets other eligibility criteria, maintenance operations typically complete within 10 to 30 seconds. If you enable HA, maintenance operations typically use rolling updates to minimize downtime. Periodic maintenance activities such as minor version upgrades occur on the standby replica first. To reduce downtime, the standby server is promoted to primary so that workloads can continue to run while the maintenance tasks are applied on the remaining node. This sequencing applies whether your server uses zone-redundant or local-redundant HA. For more information, see [Near-zero-downtime maintenance](/azure/mysql/flexible-server/concepts-maintenance#near-zero-downtime-maintenance).
>
> - **Configure custom maintenance windows.** You can configure the maintenance schedule to be system-managed or define a custom maintenance window to minimize the impact on your business operations. You can also reschedule planned maintenance operations. Schedule maintenance during low-activity periods to minimize business impact. For more information, see [Manage scheduled maintenance settings for Azure Database for MySQL](/azure/mysql/flexible-server/how-to-maintenance-portal).
>
> - **Implement retry logic.** Ensure that your applications can handle brief connectivity interruptions that might occur during maintenance restarts. To make your applications resilient to these types of problems, see [Resilience to transient faults](#resilience-to-transient-faults).
>
> - **Enable Virtual Canary maintenance on development and test servers.** Virtual Canary maintenance provides early access to updates. By enabling it on development and test servers, you can verify that upcoming updates don't affect your workload before they reach your production servers. For more information, see [Virtual Canary maintenance](/azure/mysql/flexible-server/concepts-maintenance#virtual-canary-maintenance).

## Service-level agreement

[!INCLUDE [SLA description](includes/reliability-service-level-agreement-include.md)]

Azure Database for MySQL provides different availability SLAs based on the server's configuration:

- Servers configured with zone-redundant HA.
- Servers configured with local-redundant HA.
- Servers configured without HA.

## Related content

- [Azure reliability](./overview.md)
- [Architecture best practices for Azure Database for MySQL](/azure/well-architected/service-guides/azure-database-for-mysql)
- [Overview of business continuity with Azure Database for MySQL](/azure/mysql/flexible-server/concepts-business-continuity)
