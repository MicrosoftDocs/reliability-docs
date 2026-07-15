---
title: Reliability Guides for Azure Services
description: See a list of reliability guides for Azure products and services. Learn about transient fault handling, availability zones, and multi-region support.
author: glynnniall
ms.service: azure
ms.topic: reference
ms.date: 01/15/2026
ms.author: pnp
ms.custom: subject-reliability
ms.subservice: azure-reliability
---

# Reliability guides by service

This article provides links to reliability guidance for many Azure services. Most reliability guides contain the following information:

- *Production deployment recommendations* provide guidance on how to deploy the service to meet your reliability requirements in production environments.

- *Resilience to transient faults* describes how the service handles day-to-day transient faults that can occur in the cloud. It also describes how to handle these faults in your application, including information about retry policies, timeouts, and other best practices.

- *Reliability architecture overview* is a synopsis of how the service supports reliability. It includes information about which components Microsoft manages and which components you manage, built-in redundancy features, and how to provision and manage multiple resources, if applicable.

- *Resilience to availability zone failures* describes how the service supports availability zones, requirements you need to meet to use availability zones, how traffic is routed and data is replicated between zones, what happens when a zone experiences an outage, zone recovery, and how to configure your resources for availability zone support.

- *Resilience to region-wide failures* outlines whether the service provides multi-region capabilities, requirements to use those capabilities, how traffic is routed and data is replicated between regions, the region-down experience, failover and failback support, and how to deploy custom multi-region solutions.

- *Resilience to service maintenance* describes how the service handles planned maintenance events, including how to minimize downtime and data loss during these events. It also shows you how to configure the service to improve resilience during maintenance times.

- *Service-level agreements (SLAs)*, which define and describe the expected uptime, and how the expected uptime changes based on the configuration that you use.

- *Backup and recovery* for supported services, including who controls and manages backups, where they're stored and replicated to, how they can be recovered, and whether they're accessible only within a region or across regions.

## Reliability guides by service

The following table provides links to reliability guidance for Azure services. Each guide contains information about how the service supports reliability features.

> [!NOTE]
> Some documents don't follow a single reliability guide format. These services might list more than one article that references reliability guidance.

| Service | Reliability guide | Other reliability documentation |
|----------|---------|---------|
| <img src="https://static.docs.com/ui/media/product/azure/search.svg" alt="Azure AI Search" width="24" /> Azure AI Search| [Reliability in AI Search](reliability-ai-search.md) ||
| <img src="/azure/media/index/api-center.svg" alt="Azure API Center" width="24" /> Azure API Center| [Reliability in Azure API Center](reliability-api-center.md) ||
| <img src="https://static.docs.com/ui/media/product/azure/api-management.svg" alt="Azure API Management" width="24" /> Azure API Management | [Reliability in API Management](reliability-api-management.md) ||
| <img src="https://static.docs.com/ui/media/product/azure/app-configuration.svg" alt="Azure App Configuration" width="24" /> Azure App Configuration| [Reliability in Azure App Configuration](reliability-app-configuration.md)||
| <img src="https://static.docs.com/ui/media/product/azure/app-service.svg" alt="Azure App Service" width="24" /> Azure App Service| [Reliability in App Service](reliability-app-service.md)||
| <img src="https://static.docs.com/ui/media/product/azure/app-service.svg" alt="Azure App Service - App Service Environment" width="24" /> Azure App Service - App Service Environment| [Reliability in App Service Environment](reliability-app-service-environment.md)||
| <img src="/azure/media/index/application-gateway-containers.svg" alt="Azure Application Gateway for Containers" width="24" /> Azure Application Gateway for Containers| [Reliability in Application Gateway for Containers](reliability-app-gateway-containers.md )    ||
| <img src="https://static.docs.com/ui/media/product/azure/application-gateway.svg" alt="Azure Application Gateway v2" width="24" /> Azure Application Gateway v2|[Reliability in Azure Application Gateway](./reliability-application-gateway-v2.md)||
| <img src="/azure/media/index/automation.svg" alt="Azure Application Automation" width="24" /> Azure Automation|[Reliability in Azure Automation](./reliability-automation.md)||
| <img src="/azure/media/index/recovery-services-vaults.svg" alt="Azure Backup" width="24" /> Azure Backup| [Reliability in Backup](reliability-backup.md)||
| <img src="/azure/media/index/bastion.svg" alt="Azure Bastion" width="24" /> Azure Bastion| [Reliability in Azure Bastion](reliability-bastion.md)||
| <img src="/azure/media/index/batch-accounts.svg" alt="Azure Batch" width="24" /> Azure Batch| [Reliability in Batch](reliability-batch.md)||
| <img src="https://static.docs.com/ui/media/product/azure/storage-accounts.svg" alt="Azure Blob Storage" width="24" /> Azure Blob Storage| [Reliability in Blob Storage](reliability-storage-blob.md) ||
| <img src="/azure/media/index/bot-services.svg" alt="Azure Bot Service" width="24" /> Azure Bot Service | [Reliability in Bot Service](reliability-bot.md)||
| <img src="/azure/media/index/chaos-studio.svg" alt="Azure Chaos Studio" width="24" /> Azure Chaos Studio| [Reliability in Chaos Studio](reliability-chaos-studio.md)||
| <img src="/azure/media/index/container-apps.svg" alt="Azure Container Apps" width="24" /> Azure Container Apps| [Reliability in Container Apps](reliability-container-apps.md)||
| <img src="https://static.docs.com/ui/media/product/azure/container-instances.svg" alt="Azure Container Instances" width="24" /> Azure Container Instances| [Reliability in Container Instances](reliability-container-instances.md)||
| <img src="https://static.docs.com/ui/media/product/azure/container-registry.svg" alt="Azure Container Registry" width="24" /> Azure Container Registry|[Reliability in Container Registry](reliability-container-registry.md) ||
| <img src="https://static.docs.com/ui/media/product/azure/cosmos-db.svg" alt="Azure Cosmos DB" width="24" /> Azure Cosmos DB| [Reliability in Azure Cosmos DB](reliability-cosmos-db.md) ||
| <img src="/azure/media/index/data-box.svg" alt="Azure Data Box" width="24" /> Azure Data Box|| [Recover data if an entire region fails](/azure/databox/data-box-disk-faq#how-can-i-recover-my-data-if-an-entire-region-fails-)|
| <img src="/azure/media/index/data-explorer.svg" alt="Azure Data Explorer" width="24" /> Azure Data Explorer| [Reliability in Azure Data Explorer](./reliability-data-explorer.md) ||
| <img src="https://static.docs.com/ui/media/product/azure/data-factory.svg" alt="Azure Data Factory" width="24" /> Azure Data Factory| [Reliability in Azure Data Factory](reliability-data-factory.md)||
| <img src="/azure/media/index/data-manager-for-energy.svg" alt="Azure Data Manager for Energy" width="24" /> Azure Data Manager for Energy|| [Reliability in Azure Data Manager for Energy](/azure/energy-data-services/reliability-energy-data-services)|
| <img src="/azure/media/index/data-share.svg" alt="Azure Data Share" width="24" /> Azure Data Share|| [Disaster recovery for Data Share](/azure/data-share/disaster-recovery)|
| <img src="/azure/media/index/database-mysql-server.svg" alt="Azure Database for MySQL" width="24" /> Azure Database for MySQL| [Reliability in Azure Database for MySQL](reliability-database-mysql.md)||
| <img src="/azure/media/index/database-postgresql-server.svg" alt="Azure Database for PostgreSQL" width="24" /> Azure Database for PostgreSQL| [Reliability in Azure Database for PostgreSQL](reliability-database-postgresql.md)||
| <img src="https://static.docs.com/ui/media/product/azure/databricks.svg" alt="Azure Databricks" width="24" /> Azure Databricks | [Reliability in Azure Databricks](reliability-databricks.md)||
| <img src="/azure/media/index/ddos-protection.svg" alt="Azure DDoS Protection" width="24" /> Azure DDoS Protection| [Reliability in DDoS Protection](reliability-ddos-protection.md)||
| <img src="/azure/media/index/device-registry.svg" alt="Azure Device Registry" width="24" /> Azure Device Registry |[Reliability in Device Registry](reliability-device-registry.md)||
| <img src="/azure/media/index/devops.svg" alt="Azure DevOps" width="24" /> Azure DevOps|| [Data protection overview](/azure/devops/organizations/security/data-protection#data-availability)|
| <img src="/azure/media/index/disk-storage.svg" alt="Azure Disk Storage" width="24" /> Azure Disk Storage|[Reliability in Azure Disk Storage](./reliability-storage-disk.md)||
| <img src="/azure/media/index/dns.svg" alt="Azure DNS" width="24" /> Azure DNS| [Reliability in Azure DNS ](reliability-dns.md)||
| <img src="/azure/media/index/documentdb.svg" alt="Azure DocumentDB" width="24" /> Azure DocumentDB| [Reliability in Azure DocumentDB](reliability-documentdb.md)||
| <img src="/azure/media/index/elastic-san.svg" alt="Azure Elastic SAN" width="24" /> Azure Elastic SAN| [Reliability in Elastic SAN](reliability-elastic-san.md)||
| <img src="/azure/media/index/event-grid-domains.svg" alt="Azure Event Grid" width="24" /> Azure Event Grid| [Reliability in Event Grid](./reliability-event-grid.md)||
| <img src="https://static.docs.com/ui/media/product/azure/event-hubs.svg" alt="Azure Event Hubs" width="24" /> Azure Event Hubs| [Reliability in Azure Event Hubs](./reliability-event-hubs.md) ||
| <img src="https://static.docs.com/ui/media/product/azure/expressroute.svg" alt="Azure ExpressRoute" width="24" /> Azure ExpressRoute| [Reliability in Azure ExpressRoute](reliability-virtual-network-gateway.md?pivots=expressroute) ||
| <img src="https://static.docs.com/ui/media/product/azure/storage-azure-files.svg" alt="Azure Files" width="24" /> Azure Files| [Reliability in Azure Files](reliability-storage-files.md)||
| <img src="https://static.docs.com/ui/media/product/azure/firewall.svg" alt="Azure Firewall" width="24" /> Azure Firewall| [Reliability in Azure Firewall](./reliability-firewall.md) ||
| <img src="https://static.docs.com/ui/media/product/azure/functions.svg" alt="Azure Functions" width="24" /> Azure Functions|  [Reliability in Azure Functions ](reliability-functions.md)||
| <img src="/azure/media/index/api-for-fhir.svg" alt="Azure Health Data Services" width="24" /> Azure Health Data Services||[Disaster recovery for Health Data Services](/azure/healthcare-apis/azure-api-for-fhir/disaster-recovery) |
| <img src="/azure/media/index/api-for-fhir.svg" alt="Azure Health Data Services" width="24" /> Azure Health Data Services: De-identification service||[Reliability in the Health Data Services de-identification service](/azure/healthcare-apis/deidentification/reliability-health-data-services-deidentification)|
| <img src="/azure/media/index/api-for-fhir.svg" alt="Azure Health Data Services" width="24" /> Azure Health Data Services: Workspace services (FHIR®, DICOM®, medtech) | | [Business continuity and disaster recovery considerations](/azure/healthcare-apis/business-continuity-disaster-recovery) |
| <img src="/azure/media/index/hdinsight.svg" alt="Azure HDInsight" width="24" /> Azure HDInsight| [Reliability in HDInsight](/azure/hdinsight/reliability-hdinsight)||
| <img src="https://static.docs.com/ui/media/product/azure/iot-hub.svg" alt="Azure IoT Hub" width="24" /> Azure IoT Hub| [Reliability in IoT Hub](reliability-iot-hub.md) ||
| <img src="https://static.docs.com/ui/media/product/azure/key-vault.svg" alt="Azure Key Vault" width="24" /> Azure Key Vault| [Reliability in Key Vault](./reliability-key-vault.md) ||
| <img src="/azure/media/index/key-vault-managed-hsm.svg" alt="Azure Key Vault Managed HSM" width="24" /> Azure Key Vault Managed HSM| [Reliability in Azure Key Vault Managed HSM](reliability-managed-hsm.md) ||
| <img src="https://static.docs.com/ui/media/product/azure/kubernetes-services.svg" alt="Azure Kubernetes Service (AKS)" width="24" /> Azure Kubernetes Service (AKS)| [Reliability in AKS](reliability-aks.md)||
| <img src="https://static.docs.com/ui/media/product/azure/load-balancer.svg" alt="Azure Load Balancer" width="24" /> Azure Load Balancer| [Reliability in Load Balancer](reliability-load-balancer.md)||
| <img src="https://static.docs.com/ui/media/product/azure/logic-apps.svg" alt="Azure Logic Apps" width="24" /> Azure Logic Apps|[Reliability in Logic Apps](reliability-logic-apps.md) ||
| <img src="/azure/media/index/managed-grafana.svg" alt="Azure Managed Grafana" width="24" /> Azure Managed Grafana|[Reliability in Azure Managed Grafana](reliability-managed-grafana.md) ||
| <img src="/azure/media/index/machine-learning.svg" alt="Azure Machine Learning" width="24" /> Azure Machine Learning|| [Failover for business continuity and disaster recovery](/azure/machine-learning/how-to-high-availability-machine-learning)|
| <img src="/azure/media/index/managed-redis.svg" alt="Azure Managed Redis" width="24" /> Azure Managed Redis|[Reliability in Azure Managed Redis](./reliability-managed-redis.md) ||
| <img src="/azure/media/index/migrate.svg" alt="Azure Migrate" width="24" /> Azure Migrate | | [Azure Migrate and backup and disaster recovery](/azure/migrate/resources-faq#does-azure-migrate-offer-backup-and-disaster-recovery)|
| <img src="https://static.docs.com/ui/media/product/azure/monitor.svg" alt="Azure Monitor Logs" width="24" /> Azure Monitor Logs | [Reliability in Azure Monitor Logs](./reliability-monitor-logs.md) ||
| <img src="/azure/media/index/nat-gateway.svg" alt="Azure NAT Gateway" width="24" /> Azure NAT Gateway | [Reliability in Azure NAT Gateway](./reliability-nat-gateway.md) ||
| <img src="https://static.docs.com/ui/media/product/azure/netapp-files.svg" alt="Azure NetApp Files" width="24" /> Azure NetApp Files| [Reliability in Azure NetApp Files](reliability-netapp-files.md)||
| <img src="/azure/media/index/network-watcher.svg" alt="Azure Network Watcher" width="24" /> Azure Network Watcher|| [Network Watcher service availability and redundancy](/azure/network-watcher/frequently-asked-questions#service-availability-and-redundancy)|
| <img src="/azure/media/index/notification-hubs.svg" alt="Azure Notification Hubs" width="24" /> Azure Notification Hubs| [Reliability in Notification Hubs](reliability-notification-hubs.md)||
| <img src="/azure/media/index/private-link.svg" alt="Azure Private Link service" width="24" /> Azure Private Link service| [Reliability in Azure Private Link service](reliability-private-link-service.md)||
| <img src="/azure/media/index/public-ip-address.svg" alt="Azure public IP addresses" width="24" /> Azure public IP addresses|| [Azure public IP addresses availability zone](/azure/virtual-network/ip-services/public-ip-addresses#availability-zone) |
| <img src="https://static.docs.com/ui/media/product/azure/storage-accounts.svg" alt="Azure Queue Storage" width="24" /> Azure Queue Storage|[Reliability in Queue Storage](reliability-storage-queue.md)||
| <img src="https://static.docs.com/ui/media/product/azure/virtual-network-gateways.svg" alt="Azure Route Server" width="24" /> Azure Route Server|| [Route Server frequently asked questions (FAQs)](/azure/route-server/route-server-faq)|
| <img src="https://static.docs.com/ui/media/product/azure/service-bus.svg" alt="Azure Service Bus" width="24" /> Azure Service Bus|[Reliability in Service Bus](reliability-service-bus.md)||
| <img src="/azure/media/index/service-fabric-clusters.svg" alt="Azure Service Fabric" width="24" /> Azure Service Fabric|| [Deploy a Service Fabric cluster across availability zones](/azure/service-fabric/service-fabric-cross-availability-zones) </p> [Disaster recovery in Service Fabric](/azure/service-fabric/service-fabric-disaster-recovery) |
| <img src="https://static.docs.com/ui/media/product/azure/signalr-service.svg" alt="Azure SignalR Service" width="24" /> Azure SignalR Service|[Reliability in Azure SignalR Service](reliability-signalr.md)||
| <img src="https://static.docs.com/ui/media/product/azure/recovery-services-vaults.svg" alt="Azure Site Recovery" width="24" /> Azure Site Recovery|[Reliability in Azure Site Recovery](./reliability-site-recovery.md)||
| <img src="https://static.docs.com/ui/media/product/azure/sql-database.svg" alt="Azure SQL Database" width="24" /> Azure SQL Database|[Reliability in Azure SQL Database](reliability-sql-database.md) |
| <img src="https://static.docs.com/ui/media/product/azure/sql-managed-instance.svg" alt="Azure SQL Managed Instance" width="24" /> Azure SQL Managed Instance| [Reliability in Azure SQL Managed Instance](./reliability-sql-managed-instance.md) ||
| <img src="/azure/media/index/storage-actions.svg" alt="Azure Storage Actions" width="24" /> Azure Storage Actions| [Reliability in Storage Actions](reliability-storage-actions.md)||
| <img src="/azure/media/index/storage-discovery.svg" alt="Azure Storage Discovery" width="24" /> Azure Storage Discovery| [Reliability in Storage Discovery](reliability-storage-discovery.md)||
| <img src="/azure/media/index/storage-mover.svg" alt="Azure Storage Mover" width="24" /> Azure Storage Mover| [Reliability in Storage Mover](reliability-storage-mover.md)||
| <img src="https://static.docs.com/ui/media/product/azure/stream-analytics.svg" alt="Azure Stream Analytics" width="24" /> Azure Stream Analytics| [Reliability in Azure Stream Analytics](./reliability-stream-analytics.md) ||
| <img src="https://static.docs.com/ui/media/product/azure/storage-accounts.svg" alt="Azure Table Storage" width="24" /> Azure Table Storage| [Reliability in Table Storage](reliability-storage-table.md)||
| <img src="https://static.docs.com/ui/media/product/azure/traffic-manager.svg" alt="Azure Traffic Manager" width="24" /> Azure Traffic Manager| [Reliability in Traffic Manager](reliability-traffic-manager.md)||
| <img src="https://static.docs.com/ui/media/product/azure/virtual-machine.svg" alt="Azure Virtual Machines" width="24" /> Azure Virtual Machines| [Reliability in Virtual Machines](reliability-virtual-machines.md)||
| <img src="/azure/media/index/vm-image-builder.svg" alt="Azure VM Image Builder" width="24" /> Azure VM Image Builder| [Reliability in VM Image Builder](reliability-image-builder.md)||
| <img src="/azure/media/index/vm-scale-sets.svg" alt="Azure Virtual Machine Scale Sets" width="24" /> Azure Virtual Machine Scale Sets| [Reliability in Virtual Machine Scale Sets](reliability-virtual-machine-scale-sets.md)||
| <img src="https://static.docs.com/ui/media/product/azure/virtual-networks.svg" alt="Azure Virtual Network" width="24" /> Azure Virtual Network| [Reliability in Virtual Network](reliability-virtual-network.md) ||
| <img src="/azure/media/index/virtual-network-manager.svg" alt="Azure Virtual Network Manager" width="24" /> Azure Virtual Network Manager| [Reliability in Azure Virtual Network Manager](reliability-virtual-network-manager.md) ||
| <img src="/azure/media/index/virtual-wans.svg" alt="Azure Virtual WAN" width="24" /> Azure Virtual WAN||[Availability zones and resiliency in Virtual WAN](/azure/virtual-wan/virtual-wan-faq#how-are-availability-zones-and-resiliency-handled-in-virtual-wan)</p> [Disaster recovery design](/azure/virtual-wan/disaster-recovery-design) |
| <img src="/azure/media/index/azure-vmware.svg" alt="Azure VMware Solution" width="24" /> Azure VMware Solution| [Reliability in Azure VMware Solution](./reliability-vmware-solution.md)||
| <img src="https://static.docs.com/ui/media/product/azure/virtual-network-gateways.svg" alt="Azure VPN Gateway" width="24" /> Azure VPN Gateway| [Reliability in VPN Gateway](reliability-virtual-network-gateway.md?pivots=vpn) ||
| <img src="/azure/media/index/web-pubsub.svg" alt="Azure Web PubSub" width="24" /> Azure Web PubSub Service| [Reliability in Azure Web PubSub Service](reliability-web-pubsub.md)||
| <img src="/azure/media/index/microsoft-fabric.svg" alt="Microsoft Fabric" width="24" /> Microsoft Fabric| [Reliability in Microsoft Fabric](reliability-fabric.md)||

## Related content

- [Azure services that support availability zones](availability-zones-service-support.md)
- [List of Azure regions](regions-list.md)
- [Build solutions for high availability by using availability zones](/azure/well-architected/reliability/regions-availability-zones)
