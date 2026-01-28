---
title: Reliability in Azure Virtual Machine Scale Sets
description: Learn how to make Azure Virtual Machine Scale Sets resilient to various potential outages and problems, including transient faults, availability zone outages, and region outages.
author: glynnniall
ms.author: glynnniall
ms.topic: reliability-article
ms.custom: subject-reliability
ms.service: azure-virtual-machine-scale-sets
ms.date: 01/23/2026
---

# Reliability in Azure Virtual Machine Scale Sets

Use [Azure Virtual Machine Scale Sets](/azure/virtual-machine-scale-sets/overview) to create and manage a group of virtual machine (VM) instances. The number of VM instances can automatically increase or decrease in response to demand or a defined schedule. Virtual Machine Scale Sets help make applications highly available and resilient by distributing VMs across multiple availability zones and fault domains.

[!INCLUDE [Shared responsibility description](includes/reliability-shared-responsibility-include.md)]

This article describes how to make Virtual Machine Scale Sets resilient to various potential outages and problems, including transient faults, availability zone outages, region outages, VM reconfiguration, and service maintenance. It also describes how you can use backups to recover from other types of problems, and it highlights key information about the Virtual Machine Scale Sets service-level agreement (SLA).

> [!IMPORTANT]
> When you consider the reliability of a scale set and its VMs, you also need to consider the reliability of your disks, network infrastructure, and applications that run on your VMs. Improving the resiliency of the VMs alone might have limited effect if the other components aren't equally resilient. Depending on your resiliency requirements, you might need to make configuration changes across multiple areas.

## Production deployment recommendations

The Azure Well-Architected Framework provides recommendations for reliability, performance, security, cost, and operations. To learn how these areas influence each other and contribute to a reliable Virtual Machine Scale Sets solution, see [Architecture best practices for Azure Virtual Machines and scale sets in the Well-Architected Framework](/azure/well-architected/service-guides/virtual-machines).

## Reliability architecture overview

A scale set groups multiple VM instances together and applies centralized configuration, autoscale rules, and rolling upgrades.

Scale sets support two distinct [orchestration modes](/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-orchestration-modes):

- *Flexible scale sets* (recommended) give you more flexibility to deploy and manage individual VM instances.
- *Uniform scale sets* deploy VMs that have identical configuration, and you manage them as a group.

### Fault domain spreading

*Fault domains* are fault isolation groups within a datacenter. Each fault domain is like a server rack, which is a collection of hardware nodes that share the same power, networking, cooling, and platform maintenance schedule. Because the VM instances of each scale set are spread across multiple fault domains, a planned or unplanned outage that occurs in one fault domain likely doesn't affect the VM instances in other fault domains.

When you deploy a scale set, you can control how many fault domains the VMs are spread across. For most scenarios, use the *max spreading* behavior, which uses as many fault domains as possible. For more information, see [Choose the right number of fault domains for Virtual Machine Scale Sets](/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-manage-fault-domains).

In regions that have availability zones, each zone has a distinct set of fault domains. When you create a [zone-spanning scale set](#resilience-to-availability-zone-failures), instances are spread across fault domains in each zone that your scale set uses.

### Load balancing

Scale sets can integrate with Azure load balancing services, including Azure Load Balancer and Azure Application Gateway. When the scale set adds or removes instances, the built-in load balancer integration automatically updates the load balancer configuration. For more information, see [Networking for Virtual Machine Scale Sets](/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-networking).

Scale sets include many other controls and capabilities that affect how you deploy, scale, distribute, and update instances. For more information, see [Virtual Machine Scale Sets overview](/azure/virtual-machine-scale-sets/overview).

## Resilience to transient faults

[!INCLUDE [Virtual machines transient faults](includes/virtual-machines/transient-faults-include.md)]

## Resilience to instance problems

When a scale set initiates a VM instance creation or deletion task, it's possible that the operation fails. To automatically retry failed VM instance creation or deletion tasks, consider using [Resilient create and delete for Virtual Machine Scale Sets (preview)](/azure/virtual-machine-scale-sets/resilient-vm-create-delete).

Problems might occur while instances run. For example, an instance might become unresponsive because of application crashes or resource exhaustion. By using [automatic instance repairs](/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-automatic-instance-repairs), the platform monitors your application's health and automatically does repair actions, like restarting, reimaging, or replacing a VM instance when needed.

## Resilience to availability zone failures

[!INCLUDE [Resilience to availability zone failures](~/reusable-content/ce-skilling/azure/includes/reliability/reliability-availability-zone-description-include.md)]

Virtual Machine Scale Sets supports availability zones in both zone-spanning and zonal configurations.

- A *zone-spanning* scale set spreads instances across multiple availability zones that you select.

    Spreading VM instances across availability zones gives you the highest SLA. We recommend that you use zone-spanning scale sets for most VM-based workloads in Azure.

    In a zone-spanning scale set, each VM instance and its disks are tied to a specific availability zone. When all zones are healthy, instances can communicate across zones by using a high-performance, low-latency network. If a zone experiences an outage or connectivity problem, instances in the other zones remain unaffected.

    By default, the scale set has a best-effort approach to evenly spread instances across selected zones. However, if you require strict balancing, you can choose to [change the zone balancing configuration](/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-use-availability-zones#zone-balancing).

    The following diagram shows a zone-spanning scale set spread across three zones, with one instance in each zone:

    :::image type="complex" source="media/reliability-virtual-machine-scale-sets/zone-spanning.svg" alt-text="Diagram that shows a zone-spanning scale set that has three instances, each deployed into a separate availability zone." border="false" lightbox="media/reliability-virtual-machine-scale-sets/zone-spanning.svg":::
       The diagram shows three boxes labeled availability zone 1, availability zone 2, and availability zone 3 from left to right. A rectangular box spans these three boxes. That box contains an icon labeled scale set on the far left and a VM instance icon under each availability zone heading.
    :::image-end:::

    Zone-spanning is similar to [zone redundancy](./availability-zones-overview.md#types-of-availability-zone-support) in other Azure services, but scale sets don't provide automatic replication of data across zones or failover when zones are down. A zone-spanning scale set might also have all of its instances deployed in a single zone, like when you choose to attach individual VMs to a zone-spanning flexible scale set.

    > [!NOTE]
    > If you use the flexible orchestration mode and attach, detach, or remove individual VMs, ensure that the VMs are spread across multiple zones. If the VMs are all in a single zone, your scale set might not be resilient to an outage in that zone.

- A *zonal* scale set, also called *zone-aligned*, places all its instances in a single availability zone that you specify. Each VM and its disks are zonal, so they're pinned to that specific zone.

    [!INCLUDE [Zonal resource description](includes/reliability-availability-zone-zonal-include.md)]

    The following diagram shows a zonal scale set in a single zone, with three instances in that zone:

    :::image type="complex" source="media/reliability-virtual-machine-scale-sets/zonal.svg" alt-text="Diagram that shows a zonal scale set with three instances, each deployed into the same availability zone." border="false" lightbox="media/reliability-virtual-machine-scale-sets/zonal.svg":::
       The diagram shows three boxes labeled availability zone 1, availability zone 2, and availability zone 3 from left to right. A rectangular box overlaps with the availability zone 1 box. It contains an icon labeled scale set on the far left and three VM instance icons under the availability zone heading. The boxes for availability zone 2 and availability zone 3 are empty.
    :::image-end:::    

If you don't specify availability zones for your scale set, it's *nonzonal* or *regional*. In this scenario, instances might be placed in any zone within the region and might not be evenly distributed or located in the same zone. When you use a nonzonal scale set, disk colocation in the same zone is guaranteed for Ultra and Premium v2 disks. Colocation is provided on a best-effort basis for Premium V1 disks and not guaranteed for Standard SKU disks, including solid-state drive (SSD) or hard disk drive (HDD) disks. If any zone in the region fails, your scale set might experience downtime.

### Requirements

- **Region support:** You can deploy zone-spanning and zonal scale sets into [any region that supports availability zones](./regions-list.md).

    [!INCLUDE [VM - Virtual machines zone region support](includes/virtual-machines/zone-region-support-include.md)]

    If a specific VM SKU isn't available in any of the zones that you select for your scale set, then your scale set might not be able to scale out to meet your capacity requirements.

- **Dedicated hosts:** Azure Dedicated Host deployments don't support zone-spanning or zonal scale sets.

- **Types:** Availability zone support is available for all types of scale sets, including flexible and uniform scale sets.

### Considerations

- **Fault domain spreading:** When your scale set uses availability zones, you must select from specific fault domain spreading approaches. We recommend that you use max spreading, which uses as many fault domains as possible, for most workloads. For more information, see [Choose the right number of fault domains for Virtual Machine Scale Sets](/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-manage-fault-domains).

- **Zone balancing:** [Zone balancing](/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-zone-balancing) determines whether VM instances in a scale set are evenly distributed across the zones that you select. A scale set is considered balanced if each zone has the same number of VMs, plus or minus one VM. You can set the zone balancing mode to best-effort or strict. This setting controls whether the scale set can scale out unevenly, including in zone outage scenarios.

- **Placement groups:** For uniform scale sets, if you configure multiple [placement groups](/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-placement-groups), Azure deploys multiple placement groups in each zone that your scale set uses.

### Cost

There's no cost difference between a zone-spanning, zonal, and nonzonal scale set that has the same number and type of VM instances.

### Configure availability zone support

This section explains how to configure availability zone support for your scale set.

- **Create a zone-spanning or zonal scale set.** You can configure availability zones when you create a new scale set. For more information, see [Create a virtual machine scale set that uses availability zones](/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-use-availability-zones).

    > [!NOTE]
    > [!INCLUDE [Availability zone numbering](./includes/reliability-availability-zone-numbering-include.md)]

- **Convert existing scale sets to use availability zones.** You can convert an existing nonzonal (regional) scale set to use availability zones. For more information, see [Update scale sets to add availability zones](/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-use-availability-zones#update-scale-set-to-add-availability-zones).

- **Change the availability zone configuration of an existing scale set.** You can add zones to an existing scale set, but you can't remove zones. For more information, see [Update scale sets to add availability zones](/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-use-availability-zones#update-scale-set-to-add-availability-zones).

    > [!IMPORTANT]
    > When you expand a scale set to more zones, the original VM instances don't immediately migrate or change. When you scale out, new instances are created and spread evenly across the selected availability zones. If you need data from the original instances, you're responsible for migrating the data to instances in the new zones. When you scale in the scale set, any regional instances are prioritized for removal first. Then, instances are removed based on the [scale-in policy](/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-scale-in-policy).

### Capacity planning and management

To prepare for availability zone failure, consider *over-provisioning* the number of VM instances in your scale set. This approach allows the solution to tolerate some capacity loss and continue to function without degraded performance and ensures that the remaining zones have sufficient capacity to handle full production load. For more information, see [Manage capacity by using over-provisioning](/azure/reliability/concept-redundancy-replication-backup#manage-capacity-with-over-provisioning).

### Behavior when all zones are healthy

This section describes what to expect when scale sets are configured with availability zone support and all availability zones are operational.

- **Traffic routing between zones:** You're responsible for routing traffic between VMs in the scale set, including VMs that are in different availability zones. Common approaches include Load Balancer and Application Gateway, which provide built-in integration with scale sets. For more information, see [Networking for Virtual Machine Scale Sets](/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-networking).

- **Data replication between zones:** You're responsible for any data replication that needs to happen between VMs, including across VMs in different availability zones. Databases and other similar stateful applications that run on VMs often provide capabilities to replicate data.

### Behavior during a zone failure

This section describes what to expect when scale sets are configured with availability zone support and there's an outage in their availability zones.

- **Detection and response:** You're responsible for detecting the loss of an availability zone and deciding how to respond.

    For zone-spanning scale sets, any VM instances in the affected zone might be unavailable. Instances in the healthy zones remain operational.

    For zonal scale sets deployed in the affected zone, all of the VM instances might be unavailable. You need to plan how you respond to a zone failure. For example, you might redirect traffic to another scale set in a different zone or region.

[!INCLUDE [Availability zone down notification (Service Health and Resource Health)](./includes/reliability-availability-zone-down-notification-service-resource-include.md)]

- **Active requests:** Any active requests or other work that occurs on VMs in the affected availability zone are likely to be terminated.

- **Expected data loss:** Zonal VM disks might be unavailable during a zone failure.

    If you use zone-redundant storage (ZRS) disks and an outage affects your VM, you can [force detach](/rest/api/compute/virtual-machines/attach-detach-data-disks?tabs=HTTP#diskdetachoptiontypes) your ZRS disks from the failed VM. This approach allows you to attach the ZRS disks to another VM.

- **Expected downtime:** Any VMs in the affected zone remain down until the availability zone recovers. When you use zone-spanning scale sets, VMs located in healthy zones continue to work.

- **Traffic rerouting:** You're responsible for rerouting traffic to other VMs in healthy zones.

    If you configure a zone-resilient load balancer that does health checks, the load balancer typically detects failed VMs and can route traffic to other VM instances in healthy zones.

- **Instance replacement:** Virtual Machine Scale Sets isn't guaranteed to automatically add new instances into healthy zones.

    If you have a zone-spanning scale set, you can scale out to add more instances. If the zone failure is restricted to specific sets of servers within the zone, the scale-out operation might add healthy instances into the same zone, or it might add instances into other zones. However, if the scale set uses [strict zone balancing](/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-use-availability-zones#zone-balancing), the scale set blocks scale-out operations that cause an imbalance.

   > [!TIP] 
   > It's a good practice to configure [autoscale](/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-autoscale-overview) rules based on CPU or memory usage. The autoscale rules can allow the scale set to respond to a loss of the VM instances in a zone by scaling out to add new instances in the remaining operational zones.

### Zone recovery

[!INCLUDE [VM - Virtual machines zone recovery](includes/virtual-machines/zone-recovery-include.md)]

If you add temporary instances to your scale set during a zone failure, when the zone is restored, you might need to scale down your scale set to the original capacity.

### Test for zone failures

You can use Azure Chaos Studio to simulate the loss of VMs in one or more availability zones as part of an experiment. Chaos Studio provides [built-in faults for scale sets](/azure/chaos-studio/chaos-studio-fault-library#virtual-machine-scale-set), including the ability to shut down VMs in specific zones. You can use these capabilities to simulate zone-level failures and test your failover processes.

## Resilience to region-wide failures

Scale sets are single-region resources. If the region is unavailable, any scale sets in the region are also unavailable.

### Custom multi-region solutions for resiliency

You can deploy multiple scale sets into different regions, but you need to implement replication, load balancing, and failover processes. For example, you might deploy identical scale sets in multiple regions and use Azure Front Door or Azure Traffic Manager with health probes to route traffic. You're responsible for replicating state by using application mechanisms or managed data services.

## Backup and restore

[!INCLUDE [VM - Virtual machines backups](includes/virtual-machines/backup-include.md)]

[!INCLUDE [Backups include](includes/reliability-backups-include.md)]

## Resilience to VM reconfiguration

Scale sets give you control over how you apply configuration changes to your VMs, including changing your VM SKU, changing the image that each VM uses, and adding or removing VM extensions. You can control the *upgrade policy mode*, which determines how upgrades are applied. For more information, see [Upgrade policy modes for Virtual Machine Scale Sets](/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-upgrade-policy).

Some upgrade types can require reimaging or redeploying an instance. If you have specific instances that must be excluded from automatic upgrades, such as instances that contain state that you need to preserve or configuration that you can't replicate on other instances, consider using [instance protection](/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-instance-protection).

## Resilience to service maintenance

Azure periodically performs updates to improve the reliability, performance, and security of the host infrastructure for VMs. Scale sets provide multiple ways to understand and control planned maintenance:

- [Planned maintenance notifications](/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-maintenance-notifications) tell you when maintenance is due and let you control when maintenance happens.

- [Maintenance configurations](/azure/virtual-machines/maintenance-configurations) let you schedule a maintenance window at a time that suits your business needs.

- [Scheduled Events for Linux VMs](/azure/virtual-machines/linux/scheduled-events) and for [Windows VMs](/azure/virtual-machines/windows/scheduled-events) gives your application time to prepare for VM maintenance. It provides information about upcoming maintenance events (for example, reboot) so that your application can prepare for them and limit disruption.

## Service-level agreement

[!INCLUDE [SLA description](includes/reliability-service-level-agreement-include.md)]

Virtual machine scale sets share the availability SLA for VMs. You can achieve a higher uptime percentage for your VMs by using a scale set that meets both of the following criteria:

- The scale set contains two or more instances.
- The scale set spreads those instances across two or more availability zones.

## Related content

- [What are availability zones?](/azure/reliability/availability-zones-overview)
- [What are Virtual Machine Scale Sets?](/azure/virtual-machine-scale-sets/overview)