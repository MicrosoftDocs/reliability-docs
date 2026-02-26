---
title: Reliability in Azure Managed Grafana
description: Find out about reliability in Azure Managed Grafana, including transient faults, availability zones, multi-region support, backups, and service maintenance.
author: glynnniall
ms.author: pnp
ms.topic: reliability-article
ms.custom: subject-reliability
ms.service: azure-managed-grafana
ms.date: 02/26/2026
ai-usage: as-assisted
#Customer intent: As an engineer responsible for business continuity, I want to understand the details of how Azure Managed Grafana works from a reliability perspective and plan disaster recovery strategies in alignment with the exact processes that Azure services follow during different kinds of situations.
---

# Reliability in Azure Managed Grafana

[Azure Managed Grafana](/azure/managed-grafana/overview) is a fully managed service that provides hosted Grafana workspaces on Azure. Microsoft manages the underlying infrastructure, including compute, networking, storage, and service maintenance, allowing you to focus on building and operating dashboards and visualizations.

[!INCLUDE [Shared responsibility description](includes/reliability-shared-responsibility-include.md)]

This article describes how to make Azure Managed Grafana resilient to a variety of potential outages and problems, including transient faults, availability zone outages, and region outages. It also describes how you can back up and recover from other types of problems, and highlights some key information about the Azure Managed Grafana service-level agreement (SLA).

## Production deployment recommendations

For production workloads, Microsoft recommends:

- **Use the Standard SKU** for production deployments. The Essentials SKU doesn't provide an SLA and is on a deprecation path.
- **Enable zone redundancy** when creating a workspace to improve resilience to availability zone failures.
- **Store dashboards as code** by exporting them via the Grafana API or CLI and storing them in a source control repository (such as GitHub). Use CI/CD pipelines to deploy dashboards to Azure Managed Grafana.
  - This approach supports recovery scenarios.
  - It also enables deployment to multiple Grafana instances, including instances in different Azure regions if required.

## Reliability architecture overview

[!INCLUDE [Reliability architecture overview introduction](includes/reliability-architecture-overview-introduction-include.md)]

### Logical architecture

The primary Azure resource that you deploy is a **Grafana workspace**. The workspace is fully managed by Microsoft. You don't interact with or manage any of the underlying infrastructure. All compute, networking, storage, and maintenance is handled by the platform on your behalf.

### Physical architecture

Behind the scenes, when you create a Standard SKU workspace, the Azure platform provisions the following components:

- **Grafana servers**: Dedicated virtual machines that run the Grafana application. By default, two servers are provisioned for redundancy. These servers are fully managed by Microsoft. You don't see them in your subscription, you can't access them, and you aren't responsible for patching, scaling, or maintaining them in any way.
- **Load balancer**: A network load balancer that distributes incoming browser requests across the Grafana servers. The load balancer monitors server health and automatically routes traffic away from any unhealthy server.
- **Backend database**: An Azure PostgreSQL Flexible Server instance that stores workspace configuration and other persistent data. This database is shared across all Grafana servers in the workspace.

The load balancer tracks which Grafana servers are available. In a dual-server setup, if one server becomes unhealthy, the load balancer sends all requests to the remaining server. That server picks up the browser sessions previously served by the other one, based on information saved in the shared database. In the meantime, the Azure Managed Grafana service works to repair the unhealthy server or bring up a new one.

![Diagram of the Azure Managed Grafana Standard tier workspace setup.](media/service-reliability/diagram.png)

## Resilience to transient faults

[!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)]

Azure Managed Grafana handles transient faults automatically. Because the workspace runs behind a load balancer with multiple servers, brief interruptions on individual servers are mitigated by routing requests to healthy instances. Clients should follow standard Azure retry guidance for any failed requests.

## Resilience to availability zone failures

[!INCLUDE [Resilience to availability zone failures](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)]

Azure Managed Grafana supports availability zone redundancy in supported Azure regions. When zone redundancy is enabled, the workspace's Grafana servers are distributed across multiple availability zones.

<!--NIALL TODO: Include similar diagram to Azure Firewall topic with no instance in az3-->

### Requirements

Zone redundancy support is enabled in the following regions:

| Americas | Europe | Africa | Asia Pacific |
| --- | --- | --- | --- |
| East US | North Europe |  | Australia East |
| South Central US |  |  | East Asia |
| West US 3 |  |  |  |

For regions where Azure Managed Grafana is available, see [Products available by region - Azure Managed Grafana](https://azure.microsoft.com/explore/global-infrastructure/products-by-region/?products=managed-grafana&amp;regions=all).

Deploy the workspace into a [region that supports availability zones](regions-list.md).

### Considerations

- The **Essentials SKU** doesn't support zone redundancy.
- Zone redundancy can't be enabled or disabled after workspace creation.

### Cost

Enabling zone redundancy comes at an extra cost. For more information, see [Azure Managed Grafana pricing](https://azure.microsoft.com/pricing/details/managed-grafana/).

### Configure availability zone support

- **Create a new resource with availability zones enabled:** Enable zone redundancy during workspace creation through the Azure portal, CLI, or ARM/Bicep templates.
- **Migration:** You can't enable zone redundancy on an existing workspace. Instead, you need to create a new workspace with zone redundancy enabled, migrate your dashboards and configuration, and then delete the existing workspace.
- **Disable availability zone support:** You can't disable zone redundancy after workspace creation. Instead, you need to create a new workspace without zone redundancy and delete the existing one.

For more information, see [Enable zone redundancy in Azure Managed Grafana](/azure/managed-grafana/how-to-enable-zone-redundancy).

### Behavior when all zones are healthy

When all availability zones are operational:

- **Traffic routing between zones:** The load balancer distributes incoming requests across Grafana servers in multiple availability zones.
- **Data replication between zones:** Azure Managed Grafana uses Azure PostgreSQL Flexible Server as its backend database. Data replication between availability zones is handled by the database service. Azure Managed Grafana doesn't implement additional custom replication logic beyond what the database platform provides.

### Behavior during a zone failure

If an availability zone becomes unavailable:

- **Detection and response:** The workspace continues to operate using the remaining healthy zone. Active requests to impacted servers might fail and should be retried by the client.
- **Expected data loss:** No data loss is expected during an availability zone outage.
- **Expected downtime:** Customer-visible downtime is minimal, typically limited to seconds while traffic is rerouted to healthy servers.
- **Traffic rerouting:** Incoming traffic is automatically routed to healthy servers. The service runs with reduced capacity for the duration of the zone outage. Replacement servers aren't provisioned in healthy zones during the outage.

[!INCLUDE [Availability zone down notification](includes/reliability-availability-zone-down-notification-service-resource-include.md)]

### Zone recovery

Microsoft manages zone recovery automatically, including restoring service capacity when the affected zone becomes healthy again.

### Test for zone failures

You can test your resiliency to availability zone failures by using [Azure Chaos Studio](/azure/chaos-studio/chaos-studio-overview) to simulate zone outages and observe how your workspace behaves.

## Resilience to region-wide failures

Azure Managed Grafana is deployed at the regional level. The service doesn't provide built-in multi-region failover.

### Custom multi-region solutions for resiliency

To achieve resilience to regional outages, you must:

- Deploy multiple Grafana workspaces in different regions.
- Replicate dashboards and configuration between regions (for example, by using CI/CD and source control).
- Implement traffic routing and failover at the application or client level.

## Backup and restore

Azure Managed Grafana doesn't provide built-in backup or restore functionality for dashboards or other data-plane entities.

To protect against accidental deletion or corruption:

- Use the **Grafana API or CLI** to export dashboards.
- Store exported dashboards in a **source control repository**.
- Use automation or CI/CD pipelines to redeploy dashboards as needed.

[!INCLUDE [Backups include](includes/reliability-backups-include.md)]

## Resilience to service maintenance

[!INCLUDE [Service maintenance description](includes/reliability-maintenance-include.md)]

You might observe brief interruptions (typically seconds), which are mitigated by backend mechanisms such as DNS-based traffic switching. You can't control when maintenance occurs.

## Service-level agreement

[!INCLUDE [SLA description](includes/reliability-service-level-agreement-include.md)]

The **Essentials SKU** has no SLA and is on a deprecation path. It isn't recommended for production use. The **Standard SKU** is covered by the Azure Managed Grafana SLA.

If cost is a concern and the Standard SKU isn't suitable, an alternative is [Azure Monitor Dashboards with Grafana](/azure/azure-monitor/visualize/visualize-grafana-overview), which is operated by the same Microsoft team.

## Related content

- [Reliability in Azure](./overview.md)
- [Azure Managed Grafana overview](/azure/managed-grafana/overview)
