---
title: Reliability in Azure Container Apps
description: Learn about reliability in Azure Container Apps, including availability zones and multi-region deployments.
ms.author: cshoe
author: craigshoemaker
ms.topic: reliability-article
ms.custom: subject-reliability, devx-track-azurepowershell, devx-track-azurecli
ms.service: azure-container-apps
ms.date: 06/26/2025
ms.update-cycle: 180-days
#Customer intent: As an engineer responsible for business continuity, I want to understand the details of how Azure Container Apps works from a reliability perspective and plan disaster recovery strategies in alignment with the exact processes that Azure services follow during different kinds of situations.
---

# Reliability in Azure Container Apps

This article covers reliability features in [Azure Container Apps](/azure/container-apps/overview), including availability zones and disaster recovery. For a complete overview of reliability in Azure, see [Azure reliability](/azure/reliability/overview).

## Production setup

For production workloads, consider the following setup:

- **Enable zone redundancy** when creating your Container Apps environment
- **Use 3+ replicas** for proper distribution across availability zones
- **Configure health probes** with appropriate timeouts
- **Store deployment configs** in source control for disaster recovery

## How Container Apps provides reliability

Container Apps automatically handles reliability through:

- **Replica distribution** across availability zones
- **Automatic load balancing** between healthy replicas  
- **Health monitoring** with container replacement
- **Traffic rerouting** during zone failures

When zone redundancy is enabled, Container Apps spreads your replicas across multiple zones. If one zone fails, traffic automatically routes to healthy zones without manual intervention.

### Key considerations

- **Session state**: Use stateless apps or external session storage to avoid data loss during failovers
- **Scaling costs**: Failed zones could trigger extra replicas in remaining zones
- **Deployments**: Updates apply to all zones at once - use blue-green deployments for safety

## Handle temporary failures

Container Apps automatically recovers from temporary issues like network problems or container crashes. For best results:

- **Configure health probes**:
  - Set liveness probe: `failureThreshold: 3`, `periodSeconds: 10`, `timeoutSeconds: 5`
  - Use readiness and startup probes for complete coverage
  - See [Health probes](/azure/container-apps/health-probes) for details

- **Add retry logic to your app**:
  - Use exponential backoff for external service calls
  - Implement circuit breakers to prevent cascading failures
  - Follow [transient fault guidance](/azure/well-architected/reliability/handle-transient-faults)

## Availability zones

Container Apps supports zone redundancy to protect against single zone failures. Zone redundancy automatically spreads replicas across multiple zones in a region.

### Setup requirements

- **Enable during environment creation** as you can't change this setting after you create your environment
- **Use virtual network** with available subnet
- **Available in all regions** that support availability zones ([see list](/azure/reliability/regions-list))
- **Works with all plans** including both Consumption and Dedicated workload profiles

### Configuration

Enable zone redundancy as you create your environment. For step-by-step instructions, see [Configure zone redundancy](/azure/container-apps/configure-zone-redundancy).

Verify setup with: `az containerapp env show` and check for `"zoneRedundant": true`

### Monitoring and scaling

Set up autoscaling rules based on:

- CPU and memory usage
- HTTP request queues  
- Custom metrics for your workload

>[!NOTE]
> The Consumption only environment requires a dedicated subnet with a CIDR range of `/23` or larger. The workload profiles environment requires a dedicated subnet with a CIDR range of `/27` or larger. To learn more about subnet sizing, see the [networking architecture overview](../container-apps/custom-virtual-networks.md#subnet).

For more information, see [Set scaling rules](/azure/container-apps/scale-app) for configuration options.

### What happens during zone failures

When a failure occurs traffic is routed to a replica running in a different zone.

| Aspect | Description |
|--------|-------------|
| **Detection and response** | Azure automatically detects zone failures and starts traffic rerouting. No action needed from you. |
| **Notification** | Zone failures show up in Azure Monitor metrics and can trigger alerts. Watch *Replica count* and *Request success rate* metrics. |
| **Active requests** | Requests to containers in the failed zone can time out or get connection errors. New requests go to healthy replicas in other zones. |
| **Expected data loss** | No data loss at the platform level since Container Apps is for stateless workloads. Any app data loss depends on your external data services. |
| **Expected downtime** | Minimal downtime as traffic reroutes to healthy zones within seconds. Some requests might see brief latency spikes during failover. |
| **Traffic rerouting** | The ingress controller automatically removes failed zone replicas from load balancing and sends traffic to healthy replicas. |

### Zone recovery

When a failed zone comes back online, Container Apps automatically restores proper zone distribution. New replicas are created in the recovered zone through normal autoscaling. All new replicas must pass health checks before getting traffic.

### Availability zone migration

To take advantage of availability zones, enable zone redundancy as you create the Container Apps environment. The environment must include a virtual network with an available subnet. You can't migrate an existing Container Apps environment from nonavailability zone support to availability zone support.

### Testing zone failures

Test your app's resilience by monitoring these metrics during zone outages:

- Replica count
- Request success rate
- Response time

Make sure to check autoscaling behavior and resource usage patterns and how well your scaling rules handle reduced zone capacity

Use Azure Monitor to track performance during both real and simulated zone failures.

## Cross-region disaster recovery and business continuity

[!INCLUDE [introduction to disaster recovery](includes/reliability-disaster-recovery-description-include.md)]

If you configure your container app to run in a single region that goes down, your apps are unavailable. You have two options:

1. **Manual recovery**: Wait for the region to recover, then redeploy everything
2. **Multi-region setup**: Deploy to multiple regions with traffic routing for automatic failover

> [!TIP]
> Always keep deployment configs in source control for easy redeployment.

### Multi-region setup requirements

- Separate Container Apps environments in each region
- Azure Front Door or Traffic Manager for traffic routing
- Azure Container Registry with geo-replication
- Multi-region data services (Cosmos DB, SQL Database with geo-replication)

### Setup steps

1. **Create environments** in each target region with consistent naming
2. **Configure traffic routing** with Azure Front Door or Traffic Manager ([setup guide](/azure/frontdoor/quickstart-create-front-door))
3. **Enable geo-replication** for Container Registry ([instructions](/azure/container-registry/container-registry-geo-replication))

### Costs and considerations

Multi-region deployment increases costs through:

- Multiple Container Apps environments
- Data transfer between regions
- Geo-replication for supporting services

Plan for configuration management across multiple regions. Container Apps doesn't replicate data automatically. Design your apps to handle eventual data consistency with global data services.

### When regions fail

Prepare your app to recover from an unlikely region failure using the following approach:

| Aspect | Description |
|--------|-------------|
| **Detection** | Traffic routing services detect outages through health probes and route traffic to healthy regions |
| **Downtime** | Downtime duration depends on traffic routing service configuration and health probe settings. Typical failover times range from 30 seconds to several minutes for Container Apps endpoints. |
| **Recovery** | Manually test recovered region before routing traffic back |

### Deployment patterns

Consider these approaches for multi-region apps:

- **Active-passive**: Primary region handles traffic, secondary takes over during outages
- **Active-active**: Traffic distributed across all regions for performance
- **Blue-green across regions**: Zero-downtime updates with immediate rollback

For detailed patterns, see [Geode pattern](/azure/architecture/patterns/geodes) and [Multi-region load balancing](/azure/architecture/high-availability/reference-architecture-traffic-manager-application-gateway).

## Data backup and maintenance

**Backups**: Container Apps doesn't provide built-in backup for app data. Store important data in external Azure services like Cosmos DB, Azure Database, or Azure Storage - each has their own backup features. Keep deployment configs and container images in source control and Azure Container Registry with geo-replication.

**Maintenance**: Container Apps handles planned maintenance with rolling updates to minimize disruption. You can set maintenance windows for preferred update times. During maintenance, at least one healthy replica stays available in zone-redundant deployments.

For maintenance configuration, see [Azure Container Apps planned maintenance](/azure/container-apps/planned-maintenance).

## Related content

- [Azure Container Apps overview](/azure/container-apps/overview)
- [Health probes in Azure Container Apps](/azure/container-apps/health-probes)
- [Autoscaling in Azure Container Apps](/azure/container-apps/scale-app)
- [Observability in Azure Container Apps](/azure/container-apps/observability)
- [Azure reliability documentation](/azure/reliability/)
- [Azure Well-Architected Framework reliability guidance](/azure/well-architected/reliability/)
