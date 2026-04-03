---
title: Reliability in Azure VMware Solution
description: Learn about resiliency in Azure VMware Solution, including resilience to transient faults, availability zone failures, and region failures.
author: jjaygbay1
ms.author: jacobjaygbay
ms.topic: reliability-article
ms.custom: subject-reliability, references_regions
ms.service: azure-vmware
ms.date: 02/18/2026
ai-usage: ai-assisted
zone_pivot_groups: azure-vmware-solution-generations
---

# Reliability in Azure VMware Solution

[Azure VMware Solution](/azure/azure-vmware/introduction) provides private clouds that contain VMware vSphere clusters built from dedicated bare-metal Azure infrastructure. You can migrate workloads from your on-premises environments, deploy new virtual machines (VMs), and consume Azure services from your private clouds. You can use a combination of VMware and Azure-native capabilities to enable high availability and resiliency of your workloads.

[!INCLUDE [Shared responsibility](includes/reliability-shared-responsibility-include.md)]

This article describes how to make Azure VMware Solution resilient to potential outages and problems, including transient faults, availability zone outages, and region outages. It also describes how you can use backups to recover from other types of problems and highlights some key information about the Azure VMware Solution service-level agreement (SLA).

## Production deployment recommendations

Azure VMware Solution deployments require careful planning across a range of areas and often require multiple Azure services. For detailed guidance, see [Azure VMware Solution workloads](/azure/well-architected/azure-vmware/overview) in the Azure Well-Architected Framework.

## Reliability architecture overview

Azure VMware Solution uses a hyperconverged infrastructure with VMware vSphere clusters.

When you deploy Azure VMware Solution, you deploy a *private cloud*, which has one or more clusters. Each cluster contains ESXi hosts that provide compute, storage through virtual SAN (vSAN), and networking through VMware NSX. There are two generations of Azure VMware Solution:

- Gen 1 uses specialized bare-metal hardware for nodes and uses dedicated networking approaches. For more information about the key concepts, see [Azure VMware Solution private cloud and cluster concepts](/azure/azure-vmware/architecture-private-clouds).

- [Gen 2](/azure/azure-vmware/native-introduction) uses standard Azure VM types and Azure virtual networks. This architecture simplifies networking architecture, enhances data transfer speeds, reduces latency for workloads, and improves performance when accessing other Azure services.

### Fault tolerance

Azure VMware Solution provides several mechanisms to handle faults at both the infrastructure and application level:

- *vSphere High Availability (HA):* vSphere HA monitors ESXi hosts and VMs. If a host fails, it automatically restarts affected VMs on healthy hosts. vSphere HA is turned on by default and reserves compute and memory capacity for a single node failure.

- *vSAN fault tolerance:* vSAN storage policies protect against storage-level transient faults by maintaining multiple copies of data across hosts. If a storage path or disk experiences transient problems, vSAN automatically handles failover to healthy storage paths.

- *Network redundancy:* Azure VMware Solution provides redundant network paths and multiple VMkernel network adapters to handle network-level transient faults.

## Resilience to transient faults

[!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)]

For applications that run on Azure VMware Solution VMs, implement standard practices to handle transient faults:

- Set up appropriate retry policies with exponential backoff.

- Use circuit breaker patterns for external service calls.

- Monitor application health and implement graceful degradation.

- Design stateless applications when possible to reduce the impact of VM restarts.

## Resilience to availability zone failures

[!INCLUDE [Resilience to availability zone failures](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)]

:::zone pivot="avs-gen1"

Azure VMware Solution Gen 1 supports availability zones through *[stretched clusters](/azure/azure-vmware/architecture-stretched-clusters)*, which distribute ESXi hosts across two availability zones within a region. Microsoft selects the zones to use. Your cluster runs in an active-active configuration across the two zones, and vSAN also spans multiple zones. You can designate whether each workload is deployed into one or two zones.

A witness node is automatically deployed into a third availability zone to provide quorum for split-brain scenarios. Microsoft manages the witness node automatically.

:::image type="complex" source="./media/reliability-vmware-solution/gen1-availability-zones.png" alt-text="Diagram that shows a managed vSAN stretched cluster that spans two availability zones and a witness appliance in a third availability zone." border="false" lightbox="./media/reliability-vmware-solution/gen1-availability-zones.png":::
   At the top of the diagram, a legend indicates that the Microsoft Azure logo represents the Azure platform, a location pin icon labeled dual availability zones in an Azure region represents two availability zones, and a key icon represents a single Azure subscription. The diagram is divided into three main sections. On the left, availability zone one is labeled as the preferred site. On the right, availability zone two is labeled as the secondary site. At the bottom, availability zone three is labeled as the witness site. A box that represents Azure VMware Solution private cloud A spans availability zones one and two. Within availability zone one, four components are arranged horizontally: a server rack icon represents Azure bare-metal ESXi hosts, a stacked layers icon represents the VMware vSAN datastore preferred fault domain, a server icon represents the VMware vCenter server, and a network topology icon represents VMware NSX. A stretched-vSAN datastore label spans horizontally across both zones and connects the VMware vSAN datastore preferred fault domain in availability zone one to the VMware vSAN datastore secondary fault domain in availability zone two. The bottom half of availability zone one contains three components arranged horizontally. A circular badge icon represents VMware NSX Edge A. A cloud network icon represents VMware HCX. A monitor icon represents Gold SLA apps. Within availability zone two, two rows of components are arranged horizontally. The top row includes Azure bare-metal ESXi hosts and the VMware vSAN datastore secondary fault domain. The bottom row contains VMware NSX Edge B, Gold SLA apps enclosed in a dashed box, a monitor icon that represents Silver SLA apps, and a monitor icon that represents Bronze SLA apps. A dotted line labeled policy-based synchronous replication connects the Gold SLA apps in availability zone one to the Gold SLA apps box in availability zone two. This line indicates synchronization or replication between these applications across zones. Solid lines connect availability zone one and availability zone two to a box labeled availability zone three - witness site at the bottom of the diagram. Availability zone three contains a VMware vSAN witness appliance.
:::image-end:::

A *standard cluster* is a cluster that isn't stretched across zones. In a standard cluster, the cluster and all of its ESXi hosts are considered *nonzonal* or *regional*. Nonzonal clusters might be placed in any availability zone within the region, and Microsoft selects the zone. If an availability zone in the region experiences an outage, nonzonal clusters and hosts in the affected zone might experience downtime.

::: zone-end

:::zone pivot="avs-gen2"

Azure VMware Solution Gen 2 supports *zonal* deployments of private clouds. When you set up a zonal private cloud, each of its clusters and all of their ESXi hosts are deployed into a single availability zone that you select.

A zonal private cloud doesn't protect against availability zone failures. You can deploy multiple private clouds into separate availability zones for higher resiliency, but you're responsible for deploying and configuring each private cloud independently.

If you don't select an availability zone, your private cloud, its clusters, and all of their ESXi hosts are considered *nonzonal* or *regional*. Nonzonal clusters might be placed in any availability zone within the region, and Microsoft selects the zone. If an availability zone in the region experiences an outage, nonzonal clusters in the affected zone might experience downtime.

::: zone-end

To see information about availability zone support for other generations, select the appropriate generation at the beginning of this article.

### Requirements

:::zone pivot="avs-gen1"

- **Region support:** Stretched clusters are only available in Azure regions that support the stretched cluster configuration. Check the [Azure region availability zone to host type mapping table](/azure/azure-vmware/architecture-private-clouds#azure-region-availability-zone-to-host-type-mapping-table) for current region support.

- **Minimum hosts:** Deploy a minimum of six hosts across two availability zones (three hosts for each zone) to enable stretched-cluster configuration. When you scale in or out, you must scale in pairs so that each zone has an equal number of hosts.

- **Host SKUs:** AV36, AV36P, and AV52 host types support stretched clusters. The AV64 SKU doesn't support stretched clusters.

::: zone-end

:::zone pivot="avs-gen2"

- **Region support:** You can deploy zonal private clouds in regions that support both [Azure VMware Solution Gen 2](/azure/azure-vmware/native-introduction#regional-availability) and [availability zones](./regions-list.md).

::: zone-end

:::zone pivot="avs-gen1"

### Considerations

Each availability zone in a region can support specific host types. For a detailed list of the host types available in each zone, see [Azure region availability zone to host type mapping table](/azure/azure-vmware/architecture-private-clouds#azure-region-availability-zone-to-host-type-mapping-table).

::: zone-end

### Cost

You incur costs for each node in the cluster, regardless of the cluster's availability zone configuration. For detailed pricing information, see [Azure VMware Solution pricing](https://azure.microsoft.com/pricing/details/azure-vmware/).

### Set up availability zone support

:::zone pivot="avs-gen1"

- **Deploy a new cluster:** When you create a new Azure VMware Solution private cloud in a supported region, you can configure it as a stretched cluster during deployment. This configuration distributes hosts across two availability zones automatically. For more information, see [Deploy vSAN stretched clusters](/azure/azure-vmware/deploy-vsan-stretched-clusters).

- **Existing clusters:** You can't convert a standard cluster to a stretched cluster, and you can't convert a stretched cluster to a standard cluster. Instead, you need to deploy a new cluster and migrate your workloads.

::: zone-end

:::zone pivot="avs-gen2"

- **Deploy a new cluster:** When you create a new Azure VMware Solution private cloud in a supported region, you can select its availability zone.

- **Existing clusters:** You can't change the availability zone configuration of an existing cluster. Instead, you need to deploy a new cluster and migrate your workloads.

::: zone-end

### Behavior when all zones are healthy

:::zone pivot="avs-gen1"

This section describes what to expect when your cluster is stretched and all availability zones are operational.

- **Cross-zone operation:** VMs can run on hosts in either availability zone. You can control VM placement by using vSphere Distributed Resource Scheduler (DRS) affinity and anti-affinity rules to optimize for performance or availability requirements.

- **Cross-zone data replication:** vSAN replicates data synchronously across availability zones. Both zones confirm each write operation before completion to ensure consistent data integrity.

::: zone-end

:::zone pivot="avs-gen2"

This section describes what to expect when your cluster is deployed in a zonal private cloud, and all availability zones are operational.

- **Cross-zone operation:** VMs run on hosts within the cluster's availability zone.

- **Cross-zone data replication:** No data is replicated to another zone.

::: zone-end

### Behavior during a zone failure

:::zone pivot="avs-gen1"

This section describes what to expect when your cluster is stretched and an availability zone outage occurs.

- **Detection and response:** Azure VMware Solution manages the infrastructure-level response to zone failures. vSphere HA automatically detects zone failures and initiates VM restart procedures if necessary.

[!INCLUDE [Availability zone down notification (Service Health and Resource Health)](./includes/reliability-availability-zone-down-notification-service-resource-include.md)]

- **Active requests:** Any VMs that run in the failed availability zone are restarted on hosts in the surviving availability zone. Active requests and connections to affected VMs are terminated, and clients are responsible for retrying them.

- **Expected downtime:** The time to restart failed VMs in the healthy zone is typically a few minutes, depending on the VM configuration and startup procedures. The stretched cluster remains operational with reduced capacity.

  If the failed availability zone contains the witness node, the witness becomes unreachable. As long as sufficient data replicas remain available, the data hosts and running workloads continue to operate without immediate data loss. However, vSAN loses quorum awareness in this state. This loss prevents it from safely making placement and recovery decisions. It also causes certain operations, such as VM power-on after failures, rebalancing, and repairs, to be blocked.

- **Expected data loss:** Because vSAN uses synchronous replication between zones, there's no data loss expected during a zone failure.

- **Redistribution:** vSphere DRS automatically redistributes VM workloads to the surviving availability zone. Network traffic routing through VMware NSX adapts to the new VM placement automatically.

::: zone-end

:::zone pivot="avs-gen2"

This section describes what to expect when your cluster is deployed in a zonal private cloud, and an availability zone outage occurs.

- **Detection and response:** You need to detect the loss of an availability zone. If necessary, you can initiate a failover to a secondary cluster that you create earlier in another availability zone.

[!INCLUDE [Availability zone down notification (Service Health and Resource Health)](./includes/reliability-availability-zone-down-notification-service-resource-include.md)]

- **Active requests:** Active requests and connections to affected VMs are terminated, and clients are responsible for retrying them.

- **Expected downtime:** When a zone is unavailable, your cluster and its workloads are unavailable until the availability zone recovers.

- **Expected data loss:** Data in the affected zone is unavailable until the zone recovers.

- **Redistribution:** You're responsible for switching traffic to other clusters in healthy zones, if necessary.

::: zone-end

### Zone recovery

:::zone pivot="avs-gen1"

When the availability zone recovers, vSphere DRS can optionally redistribute VMs back to the recovered zone based on your DRS configuration and affinity rules. You can also manually control VM placement by using vMotion operations.

::: zone-end

:::zone pivot="avs-gen2"

When the availability zone recovers, clusters and hosts in the zone are available again. You're responsible for any zone recovery procedures and data synchronization that your workloads require.

::: zone-end

### Test for zone failures

:::zone pivot="avs-gen1"

To prepare for zone failures, test your application's resilience to VM restarts and network path changes, especially when you have stretched clusters or deploy applications across separate clusters in different zones.

Because Azure VMware Solution manages the infrastructure response to zone failures, you primarily need to test your application's response to VM restarts.

::: zone-end

:::zone pivot="avs-gen2"

You're responsible for any infrastructure response to zone failures, such as failover to another cluster in a different zone or region. Ensure that you thoroughly test your response processes.

::: zone-end

## Resilience to region-wide failures

Each Azure VMware Solution cluster is deployed within a single Azure region. If the region becomes unavailable, your private cloud and all resources within it become unavailable.

However, you can also design custom multi-region solutions that combine different approaches or integrate with your existing infrastructure to meet your specific business requirements and recovery objectives.

### Custom multi-region solutions for resiliency

To achieve multi-region resilience with Azure VMware Solution, you need to deploy separate private clouds in multiple regions and implement failover and other disaster recovery (DR) solutions.

There are a range of options that support different requirements. For more information, see [Non-Microsoft backup and DR solutions for Azure VMware Solution](/azure/azure-vmware/ecosystem-disaster-recovery-vms).

## Backup and restore

Azure VMware Solution automatically backs up management components, such as vCenter Server, NSX Manager, and HCX Manager if enabled. To restore components from these management backups, create an Azure support request.

For your VM workloads, Azure VMware Solution supports multiple backup approaches. For more information, see [Backup solutions for Azure VMware Solution VMs](/azure/azure-vmware/ecosystem-back-up-vms).

## Resilience to service maintenance

Azure does automatic platform maintenance to apply security updates, deploy new features, and improve service reliability.

To learn about the effect that maintenance can have on the components of Azure VMware Solution, and to understand the components that you're responsible for maintaining and the components that Microsoft maintains, see [Azure VMware Solution private cloud maintenance](/azure/azure-vmware/azure-vmware-solution-private-cloud-maintenance).

You can set up the maintenance windows for your cluster to reduce the likelihood that maintenance affects your production workloads. For more information, see [Plan self-service maintenance for Azure VMware Solution](/azure/azure-vmware/self-service-maintenance-orchestration).

## Service-level agreement

[!INCLUDE [SLA description](includes/reliability-service-level-agreement-include.md)]

Azure VMware Solution provides different availability SLAs for workload infrastructure and for management operations.

:::zone pivot="avs-gen1"

Clusters that you configure as stretched clusters have a higher workload infrastructure availability SLA.

::: zone-end

However, to qualify for the availability SLAs, you must set up your cluster in specific ways. For more information, see the SLA text.

## Related content

- [Reliability in Azure](./overview.md)
- [Overview of Azure VMware Solution](/azure/azure-vmware/introduction)
- [Deploy vSAN stretched clusters](/azure/azure-vmware/deploy-vsan-stretched-clusters)
- [Deploy DR by using VMware HCX](/azure/azure-vmware/deploy-disaster-recovery-using-vmware-hcx)
- [Business continuity and disaster recovery (BCDR) for Azure VMware Solution](/azure/cloud-adoption-framework/scenarios/azure-vmware/eslz-business-continuity-and-disaster-recovery)
- [Azure VMware Solution workloads](/azure/well-architected/azure-vmware/overview)
