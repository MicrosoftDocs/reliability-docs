---
title: Available Azure services by region types and categories 
description: Learn about region types and service categories in Azure.
author: glynnniall
ms.service: azure
ms.subservice: azure-reliability
ms.topic: reference
ms.date: 04/11/2025
ms.update-cycle: 1095-days
ms.author: pnp
ms.custom: subject-reliability
---

# Available services by region types and categories 

Availability of services across Azure regions depends on a region's type. There are two types of regions in Azure: *recommended* and *alternate*.

- **Recommended** regions provide the broadest range of service capabilities and currently support availability zones. In the Azure portal, recommended regions are designated as **Recommended**.
- **Alternate** regions extend Azure's footprint within a data residency boundary where a recommended region currently exists. Alternate regions help to optimize latency and provide a second region for disaster recovery needs but don't support availability zones. Azure conducts regular assessments of alternate regions to determine if they should become recommended regions. In the Azure portal, alternate regions are designated   as **Other**.

## Service categories across region types
 
Azure services are grouped into three categories: *foundational*, *mainstream*, and *strategic*. Azure's general policy on deploying services into any given region is primarily driven by region type, service categories, and customer demand.

- **Foundational**: Available in all recommended and alternate regions when the region is generally available, or within 90 days of a new foundational service becoming generally available.
- **Mainstream**: Available in all recommended regions within 90 days of the region general availability. Demand-driven in alternate regions, and many are already deployed into a large subset of alternate regions.
- **Strategic** (previously Specialized): Targeted service offerings, often industry-focused or backed by customized hardware. Demand-driven availability across regions, and many are already deployed into a large subset of recommended regions.

To see which services are deployed in a region and the future roadmap for preview or general availability of services in a region, see [Products available by region](https://azure.microsoft.com/global-infrastructure/services/).

If a service offering isn't available in a region, contact your Microsoft sales representative for more information and to explore options.

| Region type | Non-regional | Foundational | Mainstream | Strategic | Availability zones | Data residency |
| --- | --- | --- | --- | --- | --- | --- |
| Recommended | :::image type="content" source="media/icon-checkmark.svg" alt-text="Yes" border="false"::: | :::image type="content" source="media/icon-checkmark.svg" alt-text="Yes" border="false"::: | :::image type="content" source="media/icon-checkmark.svg" alt-text="Yes" border="false"::: | Demand-driven | :::image type="content" source="media/icon-checkmark.svg" alt-text="Yes" border="false"::: | :::image type="content" source="media/icon-checkmark.svg" alt-text="Yes" border="false"::: |
| Alternate | :::image type="content" source="media/icon-checkmark.svg" alt-text="Yes" border="false"::: | :::image type="content" source="media/icon-checkmark.svg" alt-text="Yes" border="false"::: | Demand-driven | Demand-driven | N/A | :::image type="content" source="media/icon-checkmark.svg" alt-text="Yes" border="false"::: |

## Available services by region category

Azure assigns service categories as foundational, mainstream, and strategic at general availability. Typically, services start as a strategic service and are upgraded to mainstream and foundational as demand and use grow.

Azure services are presented in the following lists by category. Note that some services are non-regional, which means that they're available globally regardless of region. For information and a complete list of non-regional services, see [Products available by region](https://azure.microsoft.com/global-infrastructure/services/).

## ![An icon that signifies this service is foundational.](media/icon-foundational.svg) Foundational services

- Azure Application Gateway
- Azure Backup
- Azure Blob Storage
- Azure Cosmos DB
- Azure Data Lake Storage Gen2
- Azure Disk Storage
- Azure Event Hubs
- Azure ExpressRoute
- Azure Files
- Azure Key Vault
- Azure Kubernetes Service (AKS)
- Azure Load Balancer
- Azure NAT Gateway
- Azure Service Bus
- Azure Service Fabric
- Azure Site Recovery
- Azure SQL Database
- Azure SQL Managed Instance
- Azure Stream Analytics
- Azure Virtual Machine Scale Sets
- Azure Virtual Machines
   - Av2-series
   - Bs-series
   - Ddv5 and Ddsv5-series
   - Dv2 and DSv2-series
   - Dv3 and DSv3-series
   - Dv5 and DSv5-series
   - Edv5 and Edsv5-series
   - Ev3 and Esv3-series
   - Ev5 and Esv5-series
- Azure Virtual Network
- Azure VPN Gateway
- Public IP addresses

## ![An icon that signifies this service is mainstream.](media/icon-mainstream.svg) Mainstream services 

- Azure AI Search
- Azure API Management
- Azure App Configuration
- Azure App Service
- Azure Bastion
- Azure Batch
- Azure Container Instances
- Azure Container Registry
- Azure Data Explorer
- Azure Data Factory
- Azure Database for MySQL
- Azure Database for PostgreSQL
- Azure DDoS Protection
- Azure DNS Private Resolver
- Azure Event Grid
- Azure Firewall
- Azure Firewall Manager
- Azure Functions
- Azure HDInsight
- Azure IoT Hub
- Azure Logic Apps
- Azure Managed Redis
- Azure Monitor Logs
- Azure Monitor: Application Insights
- Azure Network Watcher
- Azure Private Link
- Azure Storage: Premium Blob Storage
- Azure Virtual Machines
   - Ddsv4-series
   - Ddv4-series
   - Dsv4-series
   - Dv4-series
   - Edsv4-series
   - Edv4-series
   - Esv4-series
   - Ev4-series
   - Fsv2-series
   - M-series
- Azure Virtual WAN
- Microsoft Entra Domain Services

### ![An icon that signifies this service is strategic.](media/icon-strategic.svg) Strategic services

- Azure AI services
- Azure Analysis Services
- Azure App Testing
- Azure Attestation
- Azure Automation
- Azure Chaos Studio
- Azure Container Apps
- Azure Data Share
- Azure Database Migration Service
- Azure Databricks
- Azure Dedicated HSM
- Azure Digital Twins
- Azure Elastic SAN
- Azure HPC Cache
- Azure Key Vault Managed HSM
- Azure Kubernetes Fleet Manager
- Azure Machine Learning
- Azure Managed Grafana
- Azure Managed Instance for Apache Cassandra
- Azure NetApp Files
- Azure Red Hat OpenShift
- Azure Remote Rendering
- Azure SignalR Service
- Azure Storage: Archive Storage
- Azure Storage: Azure File Sync
- Azure Synapse Analytics
- Azure Ultra Disk Storage
- Azure Virtual Machines
   - Bsv2-series
   - Dasv5 and Dadsv5-series
   - Dav4 and Dasv4-series
   - DCsv2-series
   - Easv5 and Eadsv5-series
   - Eav4 and Easv4-series
   - FX-series
   - HBv2-series
   - HBv3-series
   - HCv1-series
   - Lsv2-series
   - Lsv3-series
   - Lsv4, Lasv4, and Laosv4-series
   - Mv2-series
   - NCasT4_v3-series
   - NCv3-series
   - NDasrA100_v4-Series
   - NDm_A100_v4-Series
   - NDv2-series
   - NP-series
   - NVv3-series
   - NVv4-series
   - SAP HANA on Azure Large Instances
- Azure VMware Solution
- Azure Web PubSub
- Durable Task Scheduler
- Microsoft Purview
- SQL Server on Azure Virtual Machines

Older generations of services or virtual machines aren't listed. For more information, see [Previous generations of virtual machine sizes](/azure/virtual-machines/sizes-previous-gen).

To learn more about preview services that aren't yet in general availability and to see a listing of these services, see [Products available by region](https://azure.microsoft.com/global-infrastructure/services/). For a complete listing of services that support availability zones, see [Azure services that support availability zones](availability-zones-service-support.md).

## Related content

- [Azure services and regions that support availability zones](availability-zones-service-support.md)
