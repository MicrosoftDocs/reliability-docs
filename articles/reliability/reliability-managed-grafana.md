---
title: Reliability in Azure Managed Grafana
description: Find out about reliability in Azure Managed Grafana, including transient faults, availability zones, multi-region support, backups, and service maintenance.
author: maud-lv
ms.author: malev
ms.topic: reliability-article
ms.custom: subject-reliability, references_regions
ms.service: azure-managed-grafana
ms.date: 03/02/2026
ai-usage: ai-assisted
---

# Reliability in Azure Managed Grafana

[Azure Managed Grafana](/azure/managed-grafana/overview) provides hosted Grafana workspaces for building dashboards and visualizations. Microsoft manages all underlying infrastructure, including compute, networking, storage, and service updates.

[!INCLUDE [Shared responsibility description](includes/reliability-shared-responsibility-include.md)]

This article describes how to make Azure Managed Grafana resilient to a variety of potential outages and problems, including transient faults, availability zone outages, and region outages. It also describes how you can back up and recover from other types of problems, and highlights some key information about the Azure Managed Grafana service-level agreement (SLA).

## Production deployment recommendations

To increase the reliability of production deployments using Azure Managed Grafana, we recommend that you take the following actions:

>[!div class="checklist"]
> - **Enable zone redundancy** when creating a workspace to provide resilience to availability zone failures.
> - **Store dashboards and other Grafana resources as code**, such as by exporting them from the Grafana API or CLI, and storing them in a source control repository like GitHub. Use CI/CD pipelines to deploy dashboards to Azure Managed Grafana. This approach supports recovery scenarios. It also enables deployment to multiple Grafana instances, including instances in different Azure regions if required.

## Reliability architecture overview

[!INCLUDE [Reliability architecture overview introduction](includes/reliability-architecture-overview-introduction-include.md)]

### Logical architecture

The primary Azure resource that you deploy is a *workspace*. After you deploy your workspace, you use the workspace's Grafana endpoint to configure and interact with data sources, dashboards, visualizations, and other Grafana resources.

### Physical architecture

Behind the scenes, when you create a workspace, the Azure platform provisions the following components:

- **Grafana servers**: Dedicated virtual machines that run the Grafana application. By default, two servers are provisioned for high availability and redundancy. These servers are fully managed by Microsoft. You don't see them in your subscription, you can't access them, and you aren't responsible for patching, scaling, or maintaining them.
- **Load balancer**: A network load balancer that distributes incoming browser requests across the Grafana servers. The load balancer monitors server health and automatically routes traffic away from any unhealthy server.
- **Backend database**: An Azure Database for PostgreSQL database that stores workspace configuration and other persistent data. This database is shared across all Grafana servers in the workspace.

The load balancer tracks which of the Grafana servers are available. In a dual-server setup, if one server becomes unhealthy, the load balancer sends all requests to the remaining server. That server picks up the browser sessions previously served by the other one, based on information saved in the shared database. In the meantime, the Azure Managed Grafana service works to repair the unhealthy server or bring up a new one.

:::image type="content" source="media/reliability-managed-grafana/workspace-virtual-machines.svg" alt-text="Diagram that shows a Azure Managed Grafana workspace that consists of two virtual machines and a load balancer deployed by the service." border="false":::

## Resilience to transient faults

[!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)]

You can build client applications to interact with your Grafana workspace through the Grafana API. Ensure those applications follow Azure retry guidance for any failed requests.

## Resilience to availability zone failures

[!INCLUDE [Resilience to availability zone failures](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)]

Azure Managed Grafana workspaces support zone redundancy in supported Azure regions. When zone redundancy is enabled, the workspace's Grafana servers are distributed across multiple availability zones. Microsoft selects the zones your workspace uses. Other resources, such as the network load balancer, database, and shared gateway, are also configured to use multiple availability zones.

:::image type="content" source="media/reliability-managed-grafana/zone-redundant.svg" alt-text="Diagram that shows an Azure Managed Grafana workspace with two instances, each in a separate availability zone, and a zone-redundant load balancer." border="false":::

If you don't enable zone redundancy, it's *nonzonal* or *regional*, which means that the servers and other components might be placed in any availability zone within the region or within the same zone. If any availability zone in the region has a problem, your workspace might experience downtime.

### Requirements

**Region support:** Zone redundancy support is available in the following regions:

| Americas | Europe | Asia Pacific |
| --- | --- | --- |
| East US | North Europe | Australia East |
| South Central US |  | East Asia |
| West US 3 |  |  |

### Cost

Enabling zone redundancy comes at an extra cost. For more information, see [Azure Managed Grafana pricing](https://azure.microsoft.com/pricing/details/managed-grafana/).

### Configure availability zone support

- **Create a new workspace with availability zones enabled:** Enable zone redundancy during workspace creation through the Azure portal, CLI, or ARM/Bicep templates.

  For more information, see [Enable zone redundancy in Azure Managed Grafana](/azure/managed-grafana/how-to-enable-zone-redundancy).

- **Configure zone redundancy on an existing workspace:** You can't enable or disable zone redundancy on an existing workspace. Instead, you need to create a new workspace with your desired zone redundancy configuration, migrate your dashboards and configuration, and then delete the existing workspace.

### Behavior when all zones are healthy

This section describes what to expect when a workspace is configured to be zone-redundant, and all availability zones are operational.

- **Traffic routing between zones:** The zone-redundant load balancer automatically distributes incoming requests across the Grafana servers. Both servers can process traffic.

- **Data replication between zones:** Changes to the workspace's data are replicated synchronously across multiple availability zones. Data replication is performed by Azure Database for PostgreSQL. For more information, see [Reliability in Azure Database for PostgreSQL](./reliability-azure-database-postgresql.md). Azure Managed Grafana doesn't implement additional custom replication logic beyond what the database platform provides.

### Behavior during a zone failure

This section describes what to expect when a workspace is configured to be zone-redundant, and an availability zone becomes unavailable.

- **Detection and response:** the Azure platform detects and responds to a failure in an availability zone. You don't need to initiate a zone failover.

[!INCLUDE [Availability zone down notification (Service Health and Resource Health)](./includes/reliability-availability-zone-down-notification-service-resource-include.md)]

- **Expected data loss:** No data loss is expected during an availability zone outage.

- **Expected downtime:** Your workspace might experience a small amount of downtime, typically limited to a few seconds, while traffic is rerouted to healthy servers. Ensure any client applications handle [transient faults](#resilience-to-transient-faults) appropriately to minimize the effects of any downtime.

- **Traffic rerouting:** Incoming traffic is automatically routed to the server in the healthy zone. The service runs with reduced capacity for the duration of the zone outage. Replacement servers aren't provisioned in healthy zones during the outage.

### Zone recovery

Microsoft manages zone recovery automatically, including restoring service capacity when the affected zone becomes healthy again.

### Test for zone failures

The Azure platform manages traffic routing, failover, and failback for zone-redundant workspaces. This feature is fully managed, so you don't need to initiate or validate availability zone failure processes.

## Resilience to region-wide failures

Azure Managed Grafana is a single-region service. If the region is unavailable, your workspace is also unavailable.

### Custom multi-region solutions for resiliency

To achieve resilience to regional outages, you can deploy multiple Grafana workspaces in different regions. In this type of solution, you're responsible for:

- Replication of dashboards and configuration between regions. For example, you can apply consistent configuration across multiple workspaces by using CI/CD and source control.
- Implementing traffic routing and failover at the application or client level.

## Backup and restore

Azure Managed Grafana doesn't provide built-in backup or restore functionality for dashboards or other data-plane entities. To protect against accidental deletion or corruption:

- Use the Grafana API or CLI to export dashboards and other Grafana configuration.
- Store exported dashboards in a source control repository, such as GitHub.
- Use automation or CI/CD pipelines to redeploy dashboards and other Grafana configuration.

[!INCLUDE [Backups include](includes/reliability-backups-include.md)]

## Resilience to service maintenance

[!INCLUDE [Service maintenance description - transient fault](includes/reliability-maintenance-transient-fault-include.md)]

## Service-level agreement

[!INCLUDE [SLA description](includes/reliability-service-level-agreement-include.md)]

## Related content

- [Reliability in Azure](./overview.md)
- [Azure Managed Grafana overview](/azure/managed-grafana/overview)
