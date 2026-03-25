---
title: Reliability in Azure Database for MySQL
description: Learn how to make Azure Database for MySQL resilient to a variety of potential outages and problems, including transient faults, availability zone outages, region outages, and service maintenance, and learn about backup and restore.
ms.author: maghan
author: markingmyname
ms.topic: reliability-article
ms.custom: subject-reliability
ms.service: azure-database-mysql	
ms.subservice: flexible-server
ms.date: 03/24/2026
ai.usage: ai-assisted
---

# Reliability in Azure Database for MySQL

[Azure Database for MySQL](/azure/mysql/overview) is a fully managed database service designed to give you granular control and flexibility over database management functions and configuration settings. The service provides high availability and disaster recovery capabilities based on your requirements.

[!INCLUDE [Shared responsibility](includes/reliability-shared-responsibility-include.md)]

This article describes how to make Azure Database for MySQL resilient to a variety of potential outages and problems, including transient faults, availability zone outages, region outages, and service maintenance. It also describes how you can use backups to recover from other types of problems, and highlights some key information about the Azure Database for MySQL service level agreement (SLA).

## Production deployment recommendations

To learn about how to deploy Azure Database for MySQL to support your solution's reliability requirements, and how reliability affects other aspects of your architecture, see [Architecture best practices for Azure Database for MySQL in the Azure Well-Architected Framework](/azure/well-architected/service-guides/azure-database-for-mysql).

## Reliability architecture overview

[!INCLUDE [Introduction to reliability architecture overview section](includes/reliability-architecture-overview-introduction-include.md)]

### Logical architecture

When you work with Azure Database for MySQL, you deploy a *server*, which represents the compute and storage resources required to support your database server. You deploy one or more *databases* to the server.

Servers can be deployed in multiple *compute tiers*: Burstable, General Purpose, and Memory Optimized, each of which are [optimized for different kinds of workloads](/azure/mysql/flexible-server/concepts-service-tiers-storage).

For more information about the general service architecture and deployment models, see [What is Azure Database for MySQL?](/azure/mysql/overview).

### Physical architecture

- **Compute and storage separation:** Azure Database for MySQL uses a compute and storage separation architecture to support high availability. The database engine runs on a virtual machine, while data files are stored in Azure storage, which maintains three locally redundant synchronous copies of the database files to ensure data durability.

- **High availability:** You can optionally enable a *high availability configuration* on your server. When you enable the high availability configuration, the service provisions and maintains a warm standby replica server. Data changes on the primary server are synchronously replicated to the standby replica server to ensure zero data loss during a failure of the primary server.

    The architecture separates the compute layer from the storage layer, allowing the service to handle different types of failures appropriately. For higher resiliency, you can spread the servers across availability zones.

    :::image type="content" source="./media/reliability-database-mysql/high-availability.png" alt-text="Diagram showing the high availability architecture, with a primary and standby replica server." border="false" :::

    A standby replica server is deployed in the same VM configuration as the primary server, including vCores, storage, and network settings.

    You can switch between servers by performing a *failover*. There are two types of failover: *unplanned failovers*, which are used when the primary server fails, and *planned failovers*, which are used in other scenarios where you need to minimize application downtime during a failover.

    For more information, see [High availability in Azure Database for MySQL](/azure/mysql/flexible-server/concepts-high-availability).

- **Backups:** Azure Database for MySQL automatically creates server backups. For more information, see [Backup and restore](#backup-and-restore).

## Resilience to transient faults

[!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)]

Your applications must handle transient connectivity errors that can occur during maintenance, scaling operations, or network interruptions. Follow these recommendations:

> [!div class="checklist"]
> - When your application encounters transient faults, retry the operation by using exponential backoff. Increase the delay between retries and limit the number of attempts. If the operation still fails after the maximum retries, treat it as a failure.
> - Where possible, use [client libraries](/azure/mysql/flexible-server/concepts-connection-libraries) that automatically handle retries.
> - Transient errors that occur during write operations require more careful consideration. Consider making your write operations idempotent, so they can be safely executed multiple times.

> [!WARNING]
> **Note to PG:** Please advise if there are any other recommendations.

## Resilience to availability zone failures

[!INCLUDE [Resilience to availability zone failures](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)]

You can select your type of availability zone support though the *high availability* configuration. Enabling high availability deploys a *standby replica server* alongside your primary server. This high availability model helps ensure that committed data is never lost during failures. Whichever high availability deployment model you choose, data is synchronously committed to both the primary and standby replica servers. If a disruption occurs to the primary server, the server automatically fails over to the standby replica server.

Data is stored on premium storage within each availability zone, with locally redundant storage (LRS) that automatically stores three data copies within each zone.

Azure Database for MySQL supports two availability zone configuration types when you use high availability:

- **Zone-redundant high availability:** Zone redundancy provides the highest level of zone resilience by deploying a primary server in one availability zone and a standby replica server in a different availability zone. The standby replica server uses similar compute, storage, and network configuration to the primary. A zone-redundant configuration provides physical isolation of the entire stack between primary and standby servers.

   You select the availability zones for the primary and standby servers.

    We recommend zone-redundant deployments for production servers.

     :::image type="content" source="./media/reliability-database-mysql/zone-redundant.png" alt-text="Diagram showing a zone-redundant server, with the primary and standby servers in different availability zones." border="false" :::

    Write operations can experience a small increase in commit latency because the service synchronously replicates data to the standby server. On average, you can expect 5-10 percent increased latency for application writes and commits, but the impact varies by workload, selected SKU, and region.

- **Zonal (same-zone) high availability:** The primary and standby servers use the same availability zone. If a disruption occurs to the primary server, but the zone is still healthy, the server automatically fails over to the standby server. A zonal deployment gives you high availability within a single availability zone. It protects you against node-level failures and also helps with reducing application downtime during planned and unplanned downtime events. However, it doesn't protect against an outage in that zone.

    :::image type="content" source="./media/reliability-database-mysql/zonal.png" alt-text="Diagram showing a zonal server, with the primary and standby servers in the same availability zone." border="false" :::

    While it's not the recommended option, you can opt into zonal (same-zone) high availability when you deploy your server. It's also the only high availability option available if the server's region doesn't support availability zones. The region effectively functions as a single zone, and so the only high availability configuration you can select is same-zone.

    Because the servers are in the same zone, it can reduce the write latency to applications you deploy within the same zone.

If you configure your server without high availability, then it runs on a single server. If that server or its zone goes down, your server is unavailable.

### Requirements

- **Region support:** Azure Database for MySQL's support for availability zone configurations differs between Azure regions. For a full list of regions, and the types of availability zone support and any specific considerations for that region, see [Azure regions](/azure/mysql/flexible-server/overview#azure-regions).

- **Service tier:** High availability requires General Purpose or Memory Optimized tiers. The Burstable tier doesn't support high availability (zone-redundant or same-zone).

### Cost

When you enable high availability, the standby server is created and billed at the same rate as the primary. The availability zone configuration doesn't affect the cost. There are no charges for data replication within or between availability zones. Depending on your backup storage volume, you might also be billed for backup storage. For detailed pricing information, see [Azure Database for MySQL pricing](https://azure.microsoft.com/pricing/details/mysql/flexible-server/).

### Considerations

- **Primary keys:** We recommend you use primary keys on all tables, because this approach reduces replication and failover time.
- **Limitations and known problems:** Review the list of [limitations](/azure/mysql/flexible-server/concepts-high-availability#limitations) and [known problems](/azure/mysql/flexible-server/concepts-high-availability#known-problems).

### Configure availability zone support

To configure availability zone support for a server, you configure the high availability settings.

- **Create a zone-redundant server:** To learn how to create a server with high availability and zone redundancy enabled, see:
    - Azure portal: [Manage zone redundant high availability in Azure Database for MySQL with the Azure portal](/azure/mysql/flexible-server/how-to-configure-high-availability#enable-high-availability-during-server-creation)
    - Azure CLI: [Manage zone redundant high-availability in Azure Database for MySQL with Azure CLI](/azure/mysql/flexible-server/how-to-configure-high-availability-cli#enable-high-availability-during-server-creation)

- **Create a zonal server:** To create a server with same-zone (zonal) high availability, you must use the Azure CLI or another programmatic deployment method. For Azure CLI instructions, see [Enable high-availability during server creation](/azure/mysql/flexible-server/how-to-configure-high-availability-cli#enable-high-availability-during-server-creation).

- **Change the availability zone configuration for existing servers:** To move from zone-redundant to same-zone high availability, first disable high availability, and then enable same-zone high availability. You must use the Azure CLI or another programmatic deployment method. For Azure CLI instructions, see [Manage zone redundant high-availability in Azure Database for MySQL with Azure CLI](/azure/mysql/flexible-server/how-to-configure-high-availability-cli).

    You can't move from same-zone to zone-redundant high availability, and you can't enable high availability on an existing server.

- **Disable high availability:** Disabling high availability removes the standby relica server, so your server isn't resilient to outages in its availability zone. For more information, see [Disable high availability](/azure/mysql/flexible-server/how-to-configure-high-availability#disable-high-availability).

### Behavior when all zones are healthy

This section describes what to expect when servers are configured with high availability and availability zone support and all availability zones are operational.

- **Cross-zone operation:** MySQL client applications connect to the primary server by using the database server's fully qualified domain name (FQDN). Avoid using the IP address of the primary server because the IP address can change, including during failovers. Azure Database for MySQL uses an active/passive configuration where all database connections and queries are handled by the primary server in the primary availability zone. The standby replica server doesn't serve client traffic during normal operations.

- **Cross-zone data replication:** Changes to data are replicated synchronously between the primary and standby servers. Transactions aren't considered complete until both the primary and standby servers acknowledge the write.

    Application transaction-triggered write and commits first write to the primary server. The primary server streams these logs to the standby server by using the MySQL replication protocol. When the standby replica server storage persists the logs, the primary server acknowledges write completion. The application commits its transaction only after this acknowledgment. This acknowledgment process doesn't wait for the logs to be applied to the standby replica server.
    
    The effects of replication are different depending on the availability zone configuration that your server uses:

    - *Zone-redundant:* Because the servers are in separate zones, this approach ensures zero data loss during a zone failure. This situation is also sometimes called achieving a recovery point objective (RPO) of zero for zone failures.
    
        However, cross-zone replication might introduce a small amount of extra latency. On average, you can expect 5-10 percent increased latency for application writes and commits, but the impact varies by workload, selected SKU, and region.

    - *Zonal*: Because both servers are in the same zone, no traffic is replicated between zones.

    > [!NOTE]
    > The system replicates log data in real-time to the standby replica server. Any user errors on the primary server, such as an accidental drop of a table or incorrect data updates, are replicated to the standby replica server. So, you can't use the standby replica server to recover from these kinds of errors, and you must perform a point-in-time restore from the backup. For more information, see [Backup and restore](#backup-and-restore).

### Behavior during a zone failure

This section describes what to expect when servers are configured with high availability and availability zone support and there's an availability zone outage.

- **Detection and response:** Azure periodically checks the health of both the primary and standby servers. After multiple pings, if health monitoring detects that a primary server isn't reachable, the service initiates an automatic failover to the standby server. The health monitoring algorithm uses multiple data points to avoid false positive situations.

    In the event of a zone failure, the behavior is different depending on the availability zone configuration that your server uses:

    - *Zone-redundant:* Azure Database for MySQL automatically detects availability zone failures by continuously monitoring multiple server endpoints. For more information, see [How automatic failover detection works in HA enabled servers](/azure/mysql/flexible-server/concepts-high-availability#how-automatic-failover-detection-works-in-ha-enabled-servers).
    
        To view the possible high availability status types, see [Monitor high availability](/azure/mysql/flexible-server/concepts-high-availability#monitor-high-availability). When a zone fails, Azure initiates an [unplanned failover](/azure/mysql/flexible-server/concepts-high-availability#unplanned-automatic-failover) to the standby server without requiring you to take action.

    - *Zonal:* If the availability zone that hosts a zonal server becomes unavailable, both the primary and standby servers are unavailable. In this scenario, the service doesn't provide automatic failover. You're responsible for detecting the zone outage and performing recovery actions, such as restoring zone‑redundant backups to a separate server in another availability zone or region.

- **Notification**: [!INCLUDE [Availability zone down notification partial bullet (Service Health and Resource Health)](./includes/reliability-availability-zone-down-notification-service-resource-partial-include.md)]

    Azure Database for MySQL generates an Azure Reosuce Health event when an unplanned failover occurs.

- **Active requests:** When an availability zone becomes unavailable, any in‑progress requests to servers in the affected zone might be terminated. Applications must retry these requests. If your clients handle [transient faults](#resilience-to-transient-faults) appropriately by retrying after a short period of time, they typically avoid significant impact.

- **Expected data loss:** The amount of data loss depends on the availability zone configuration that your server uses.

    - *Zone-redundant:* Zero data loss is expected during zone failover because of synchronous replication between the primary and standby servers in different zones.

    - *Zonal:* Data on servers in the affected zone is unavailable until the zone recovers.

- **Expected downtime:** The amount of downtime depends on the availability zone configuration that your server uses.

    - *Zone-redundant:* Failover typically completes within 60-120 seconds. If your clients handle [transient faults](#resilience-to-transient-faults) appropriately by retrying after a short period of time, they typically avoid significant impact.

    - *Zonal:* Servers in an impacted zone are unavailable until the availability zone recovers.

- **Redistribution:** The traffic rerouting behavior depends on the availability zone configuration that your server uses.

    - *Zone-redundant:* After failover, the former standby server becomes the new primary and begins accepting new connections. Azure automatically establishes a new standby server in the original primary zone after it recovers. For full details, see [Unplanned failover](/azure/mysql/flexible-server/concepts-high-availability#unplanned-automatic-failover).

    - *Zonal:* When a zone is unavailable, your server is unavailable. If you have a separate server that you precreated in another availability zone or region, you're responsible for rerouting traffic to that server.

### Zone recovery

The zone recovery behavior depends on the availability zone configuration that your server uses.

- *Zone-redundant:* When the availability zone recovers, Azure Database for MySQL automatically rebuilds the standby server in the recovered zone and synchronizes it with the current primary. The recovered zone then serves as the standby location. The service doesn't automatically move the primary role back to the original zone to avoid unnecessary disruption. You can [manually initiate a planned failover](/azure/mysql/flexible-server/concepts-high-availability#planned-forced-failover) if you want to return the primary to the original zone.

- *Zonal:* After the zone is healthy, servers in the zone are available again. You're responsible for any zone recovery procedures and data synchronization that your workloads require.

### Test for zone failures

The options for testing for zone failures depend on the availability zone configuration that your instance uses.

- *Zone-redundant:* You can test your application's resilience to failover by initiating a planned failover. A planned failover lets you simulate an unplanned outage scenario while running your workload and observe your application downtime. We recommend running simulations in a non-production environment, or at a quiet time. For more information, see [Planned failover](/azure/mysql/flexible-server/concepts-high-availability#planned-forced-failover).

- *Zonal:* While you can't simulate a full zone outage, you can simulate your server being unavailable in a similar way to what happens during a zone outage. For more information, see:
    - Azure portal: [Stop/Start an Azure Database for MySQL - Flexible Server instance](/azure/mysql/flexible-server/how-to-stop-start-server-portal)
    - Azure CLI: [Restart/stop/start an Azure Database for MySQL - Flexible Server instance](/azure/mysql/flexible-server/how-to-restart-stop-start-server-cli)

## Resilience to region-wide failures

Azure Database for MySQL supports *cross-region read replicas*, which you can use to maintain a synchronized copy of your database in a different region for faster recovery.

You can also use geo-redundant backups, in supported regions, to provide cross-region recovery. However, backups typically involve more downtime and data loss than replication. For more information, see [Backup and restore](#backup-and-restore).

### Cross-region read replicas

You can deploy read replicas to protect your databases from region-level failures. Each read replica is a separate Azure Database for MySQL server. When you place a read replica in a second Azure region, your database server can provide resilience to a region-wide problem. You can deploy up to ten read replicas, which can optionally be in different Azure regions. MySQL's physical replication technology updates read replicas asynchronously from the source server in the primary region, and they can lag the source. Cross-region read replicas can optionally serve read-only workloads to reduce latency for globally distributed applications or to offload read traffic from the source server. For more information on read replica features and considerations, see [Read replicas](/azure/mysql/flexible-server/concepts-read-replicas).

:::image type="content" source="./media/reliability-database-mysql/read-replica.png" alt-text="Diagram showing a read replica in a second Azure region, with the application directing read-write traffic to the source server." border="false" :::

If your primary region fails, you can trigger a *failover* so that your secondary replica becomes the primary server. You do this by stopping the replication process. Because of the asynchronous replication, failover can result in data loss. Your application then needs to connect to the new primary server, and you're responsible for this application reconfiguration.

:::image type="content" source="./media/reliability-database-mysql/read-replica-failure.png" alt-text="Diagram showing a read replica in a second Azure region that has been failed over to become the primary server, with the  application now directing read-write traffic to the secondary region." border="false" :::

> [!NOTE]
> This section summarizes some of the important information about how read replicas can support resilience to region-wide failures. You can also use read replicas to improve performance and support high-scale geographically distributed user bases. For more information, see [Read replicas](/azure/mysql/flexible-server/concepts-read-replicas).

#### Requirements

- **Region support:** You can create cross-region read replicas in any region that supports Azure Database for MySQL. You aren’t limited to Azure paired regions.

- **Compute tiers:** The General Purpose and Memory Optimized compute tiers support read replicas. The Burstable tier doesn't support read replicas.

#### Considerations

- **Configuration differences:** When you create a replica, it inherits several settings from the source server, including the compute genration, vCores, and storage. You can customize these values on the read replica after it's created. However, it's best to use equal or greater values to ensure that the replica can keep up with changes in the source.

- **Monitoring replication lag:** The asynchronous replication process requires a replication lag, which can vary depending on a number of factors. When the replication lag is very high, your server might experience problems. It's important to monitor the replication lag so that you can mitigate problems before they escalate. For more information, see [Monitor replication](/azure/mysql/flexible-server/concepts-read-replicas#monitor-replication).

- **High availability:** Read replicas can't have high availability enabled, and when they're failed over to become the primary server, they also don't have high availability. You're responsible for configuring high availability after failing over to a replica.

#### Cost

Read replicas incur compute and storage costs, as well as cross-region data transfer charges for replication. For detailed pricing information, see [Azure Database for MySQL pricing](https://azure.microsoft.com/pricing/details/mysql/flexible-server/) and [Bandwidth pricing](https://azure.microsoft.com/pricing/details/bandwidth/).

#### Configure multi-region support

- **Create a read replica:** To learn how to create a read replica, see:
    - Azure portal: [How to create and manage read replicas in Azure Database for MySQL - Flexible Server by using the Azure portal](/azure/mysql/flexible-server/how-to-read-replicas-portal)
    - Azure CLI: [How to create and manage read replicas in Azure Database for MySQL - Flexible Server using the Azure CLI](/azure/mysql/flexible-server/how-to-read-replicas-cli)
    
    Replicas can be configured after the source server is created, as long as the source server is running and accessible.

- **Stop replication:** To learn how to stop replication, see [Stop replication to a replica server](/azure/mysql/flexible-server/how-to-read-replicas-portal#stop-replication-to-a-replica-server).

- **Delete a read replica:** To learn how to delete a read replica, see [Delete a replica server](/azure/mysql/flexible-server/how-to-read-replicas-portal#delete-a-replica-server).

#### Behavior when all regions are healthy

This section describes what to expect when your server is configured with a read replica in another region and a virtual endpoint, and all regions are operational:

- **Traffic routing between regions:** In normal operations, your application should direct read-write traffic to the source server in the primary region. You can optionally direct read requests to your read replica.

- **Data replication between regions:** Cross-region read replicas use asynchronous replication to minimize impact on source server performance. The amount of replication lag depends on a number of factors, including the write load and the latency between the source server and replicas. Replication lag is typically at least several minutes, but it can be much longer. For more information, see [Monitor replication](/azure/mysql/flexible-server/concepts-read-replicas#monitor-replication), and for detailed instructions, see [Monitor replication in the Azure portal](/azure/mysql/flexible-server/how-to-read-replicas-portal#monitor-replication).

#### Behavior during a region failure

This section describes what to expect when your server is configured with a read replica in another region and a virtual endpoint, and there's an outage in the primary region:

- **Detection and response:** You're responsible for detecting an outage in the primary region, and manually triggering a failover. This action can result in the loss of any unreplicated data.

    > [!IMPORTANT]
    > You're responsible for triggering failover. Azure doesn't fail over to read replicas automatically, even if there's a region failure.

    Failover requires that you:
    1. Stop replication. This is an irreversable procedure, and the server can't be made into a replica again. The process results in data loss. For more detail on the implications of this action, see [Stop replication](/azure/mysql/flexible-server/concepts-read-replicas#stop-replication).
    1. Reconfigure your application to use the new primary.

    For more information, see [Failover](/azure/mysql/flexible-server/concepts-read-replicas#failover).

- **Notification:** [!INCLUDE [Region down notification partial bullet (Azure Service Health only)](./includes/reliability-region-down-notification-service-partial-include.md)]

- **Active requests:** All active connections to the source region are dropped if the source server is unavailable. Applications need to retry making connections to the new primary server after the failover process completes.

- **Expected data loss:** During a region outage, you must perform a failover that stops replication. This process results in the permanent loss of any unreplicated data.

    The amount of data loss depends on the replication lag at the time of the outage. Replication lag is typically at least several minutes, but it can be much longer. For more information, see [Monitor replication](/azure/mysql/flexible-server/concepts-read-replicas#monitor-replication).

- **Expected downtime:** Stopping replication typically completes within 2 minutes of being triggered. You're responsible for reconfiguring your applications to connect to the new primary server, and the time it takes for you to perform the reconfiguration also contributes to your overall downtime.

- **Traffic rerouting:** You're responsible for reconfiguring your applications to connect to the new primary server.

    > [!NOTE]
    > After a read replica is failed over to be the primary server, it doesn't have high availability configuration enabled. You need to enable high availability configuration manually, or add it to your own automation processes.

#### Region recovery

When the region recovers, you're responsible for any failback activities to resume operations in the primary region. Microsoft doesn't automatically move the primary server. You can create a new read replica in the primary region, then perform another failover process to restore operations in the primary region. Consider one of the following approches, depending on whether your application can tolerate downtime or data loss:

- Take your application offline, and wait for the replication to catch up with all of the changes. This approach requires application downtime, roughly equal to the replication lag.
- Perform the failover and accept the loss of any unreplicated data.

Remember you're also responsible for reconfiguring your applications to connect to the new primary server as required.

#### Test for region failures

Regularly test read replica failover procedures to ensure your processes are valid, and that the capabilities meet your RTO and RPO requirements.

You can fail over a read replica to become the primary server at any time, even when all regions are healthy. We recommend you perform these tests in a non-production environment as it can result in data loss and requires manual failback.

As part of your disaster recovery strategy, regularly run full recovery drills. These drills should include data validation, application functionality testing, and documented rollback procedures.

## Backup and restore

Azure Database for MySQL automatically performs backups that provide point-in-time recovery capabilities, and help to protect you against accidental corruption and deletion of data. Backups are fully managed by Microsoft, don't interrupt the availability of the server, and include both full backups and transaction log backups.

- **Backup storage:** If the server is configured with zone-redundant high availability, backups are stored in zone-redundant storage (ZRS). For servers configured without high availability, or with zonal (single-zone) high availability, backups are stored in locally redundant storage (LRS).

    In Azure regions with pairs, you can configure geo-redundant (GRS) backup storage at server creation time to replicate backups to the Azure paired region for additional protection against region failures. Backups are replicated asynchronously.

    The default backup retention period is 7 days, with the option to extend retention up to 35 days. All backups are encrypted.

- **Restore:** Point-in-time recovery allows you to restore your database to any moment within the backup retention period. The restore process creates a new database server with a new user-provided server name, which you can then use as-is or copy data from.

    When you restore a geo-redundant backup, you create a new server in the paired region. In some regions, you can use Universal Geo-Restore to restore a geo-redundant backup to a region that isn't your primary region's paired region.

    This capability is useful for recovering from accidental data modifications, application errors, or testing scenarios.

[!INCLUDE [Backups description](includes/reliability-backups-include.md)]

For more information, see [Backup and restore in Azure Database for MySQL](/azure/mysql/flexible-server/concepts-backup-restore).

## Resilience to service maintenance

Azure Database for MySQL automatically handles critical servicing tasks including patching of the underlying hardware, operating system, and database engine. The service includes security updates, software updates, and minor version upgrades as part of planned maintenance. For more information, see [Scheduled maintenance in Azure Database for MySQL](/azure/mysql/flexible-server/concepts-maintenance).

To ensure your server remains available during maintenance windows, follow these recommendations:

> [!div class="checklist"]
> - **Avoid management operations during maintenance periods:** Don't perform server management operations while maintenance is underway, because these operations can affect your server's reliability.
>
> - **Use near-zero-downtime maintenance:** If your server has high availability enabled and meets other eligibility criteria, maintenance operations often complete within 10-30 seconds. If you have high availability enabled, maintenance operations typically use rolling updates to minimize downtime. Periodic maintenance activities such as minor version upgrades happen on the standby replica first. To reduce downtime, the standby is promoted to primary so that workloads can keep on while the maintenance tasks are applied on the remaining node. This sequencing applies whether your server uses zone-redundant or zonal high availability. For more information, see [Near-zero-downtime maintenance](/azure/mysql/flexible-server/concepts-maintenance#near-zero-downtime-maintenance).
>
> - **Configure custom maintenance windows:** You can configure the maintenance schedule to be system-managed or define a custom maintenance window to minimize the impact on your business operations. You can also reschedule planned maintenance operations. Schedule maintenance during low-activity periods to minimize business impact. For more information, see [Manage scheduled maintenance settings for Azure Database for MySQL](/azure/mysql/flexible-server/how-to-maintenance-portal).
>
> - **Implement retry logic:** Ensure your applications can handle brief connectivity interruptions that may occur during maintenance restarts. To make your applications resilient to these types of problems, see [Resilience to transient faults](#resilience-to-transient-faults) guidance.
>
> - **Enable Virtual Canary maintenance on development and test servers:** Virtual Canary maintenance offers early access to updates. By enabling it on development and test servers, you can verify that your workload isn't affected by upcoming updates before they reach your production servers. For more information, see [Virtual Canary maintenance](/azure/mysql/flexible-server/concepts-maintenance#virtual-canary-maintenance).

## Service-level agreement

[!INCLUDE [SLA description](includes/reliability-service-level-agreement-include.md)]

Azure Database for MySQL provides different availability SLAs based on the server's configuration:

- Servers configured with zone-redundant high availability.
- Servers configured with zonal (single-zone) high availability.
- Servers configured without high availability.

## Related content

- [Azure reliability](./overview.md)
- [Architecture best practices for Azure Database for MySQL](/azure/well-architected/service-guides/azure-database-for-mysql)
- [Overview of business continuity with Azure Database for MySQL](/azure/mysql/flexible-server/concepts-business-continuity)
