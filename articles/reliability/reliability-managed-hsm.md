---
title: Reliability in Azure Key Vault Managed HSM
description: Learn how to make Azure Key Vault Managed HSM resilient to a variety of potential outages and problems, including transient faults, hardware failures, region outages, and service maintenance. Understand backup and restore options, recovery features, and SLA details.
author: msmbaldwin
ms.author: mbaldwin
ms.topic: reliability-article
ms.custom: subject-reliability
ms.service: azure-key-vault
ms.subservice: managed-hsm
ms.date: 04/22/2026
---

# Reliability in Azure Key Vault Managed HSM

[Azure Key Vault Managed HSM](/azure/key-vault/managed-hsm/overview) is a fully managed, highly available, single-tenant, standards-compliant cloud service that enables you to safeguard cryptographic keys for your cloud applications by using FIPS 140-3 Level 3 validated hardware security modules (HSMs). Managed HSM provides a range of built-in reliability features to help ensure that your keys remain available.

[!INCLUDE [Shared responsibility](includes/reliability-shared-responsibility-include.md)]

This article describes how Managed HSM is resilient to a variety of potential outages and problems, including transient faults, hardware failures, and region outages. It also describes how you can use backups and the security domain to recover from other types of problems, recovery features to prevent accidental deletion, and highlights some key information about the Managed HSM service level agreement (SLA).

## Production deployment recommendations for reliability

For production workloads, we recommend that you:

> [!div class="checklist"]
> - [Download and securely store the security domain](/azure/key-vault/managed-hsm/security-domain) immediately after provisioning your Managed HSM. The security domain is required for disaster recovery.
> - Establish a multi-person quorum for the security domain with at least three key holders.
> - [Enable purge protection](/azure/key-vault/managed-hsm/recovery) to prevent accidental or malicious deletion.
> - Implement regular [backups](/azure/key-vault/managed-hsm/backup-restore) to an Azure Storage account, and use geo-redundant storage in supported regions.
> - For mission-critical workloads that require a higher SLA, enable [multi-region replication](/azure/key-vault/managed-hsm/multi-region-replication).

## Reliability architecture overview

When you use Managed HSM, you deploy an *instance*, which is also sometimes called a *pool*.

Managed HSM is designed for high availability and durability through its architecture:

- **Single-tenant isolation**: Each Managed HSM instance is dedicated to a single customer and consists of a cluster of multiple HSM partitions that are cryptographically isolated.

- **Triple-redundant partitions**: A Managed HSM pool consists of three load-balanced HSM partitions distributed across separate racks within a datacenter. This distribution provides redundancy against hardware failures and ensures that the loss of a single component (such as a rack's power supply or network switch) doesn't affect all partitions.

- **Confidential computing**: Each service instance runs in a trusted execution environment (TEE) that uses Intel SGX enclaves. Microsoft personnel, including those with physical access to the servers, can't access your cryptographic key material.

- **Automatic healing**: If a hardware failure or other issue affects one of the three partitions, the service automatically rebuilds the affected partition on healthy hardware without any customer intervention and without exposing secrets.

For more information about how Managed HSM implements these capabilities, see [Key sovereignty, availability, performance, and scalability in Managed HSM](/azure/key-vault/managed-hsm/managed-hsm-technical-details).

### Security domain

The [security domain](/azure/key-vault/managed-hsm/security-domain) is a critical component for disaster recovery. It's an encrypted blob that contains all the credentials needed to rebuild a Managed HSM instance from scratch, including the partition owner key, the partition credentials, the data-wrapping key, and an initial backup of the HSM.

> [!IMPORTANT]
> Without the security domain, disaster recovery isn't possible. Microsoft has no way to recover the security domain and can't access your keys without it.

Security domains are a critical part of the security and reliability of your Managed HSM. We recommend you follow these best practices:

- **Generate keys securely**: For production environments, generate the RSA key pairs that protect the security domain in an air-gapped environment (such as an on-premises HSM or an isolated workstation).
- **Store offline**: Store security domain keys on encrypted USB drives or other offline storage, with each key share on a separate device in separate geographical locations.
- **Establish a multi-person quorum**: Use at least three key holders to prevent any single person from having access to all quorum keys, and to avoid a dependency on any single person.

For more information, see [Security domain in Managed HSM overview](/azure/key-vault/managed-hsm/security-domain).

## Resilience to transient faults

[!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)]

When you use Azure services that integrate with Managed HSM, those services handle transient faults automatically.

If you build custom applications that integrate with Managed HSM, consider the following best practices to handle any transient faults that might occur:

- **Use the Microsoft-provided SDKs** for Azure Key Vault, which include built-in retry mechanisms. SDKs are available for [.NET](/azure/key-vault/managed-hsm/quickstart-dotnet), [Python](/azure/key-vault/managed-hsm/quickstart-python), and [JavaScript](/azure/key-vault/managed-hsm/quickstart-javascript).

- **Implement retry logic when they interact directly with Managed HSM**, including exponential backoff retry policies.

- **Reduce the number of direct dependencies on Managed HSM.**  Cache cryptographic operation results when possible to reduce direct requests to Managed HSM. For public-key operations such as encryption, wrapping, and verification, perform these operations locally by caching the public key material. Performing the operations locally reduces the dependency on your Managed HSM and avoids transient faults from interrupting these operations.

If you use Managed HSM in high-throughput scenarios, note that Managed HSM doesn't throttle cryptographic operations. It uses its HSM hardware to full capacity. Each Managed HSM instance has three partitions. During maintenance or healing operations, one partition might be unavailable. For capacity planning, assume that two partitions are available. If you require guaranteed throughput, plan based on one partition being available. [Monitor the *Managed HSM Availability* metric](/azure/key-vault/managed-hsm/logging-azure-monitor) to understand the health of the service.

For scaling the encryption of large amounts of data, use a key hierarchy where only the key encryption key (KEK) is stored in Managed HSM and used to wrap lower-level keys that are stored in another secure key storage location.

For detailed performance benchmarks and capacity planning guidance, see [Azure Managed HSM Scaling Guidance](/azure/key-vault/managed-hsm/scaling-guidance).

## Resilience to partition failures

Managed HSM achieves high availability through its triple-redundant architecture, where each HSM pool consists of three HSM partitions distributed across separate server racks within a datacenter. This rack-level distribution provides redundancy against localized hardware failures.

:::image type="complex" source="/azure/key-vault/managed-hsm/media/managed-hsm-architecture.png" alt-text="Diagram that shows a Managed HSM pool's three partitions, each on a separate physical server and in a different server rack." border="false":::
    Diagram that shows a Managed HSM pool's three partitions, each on a separate physical server and in a different server rack.
:::image-end:::

When hardware failures or localized outages occur, Managed HSM automatically redirects your requests to healthy partitions and rebuilds affected partitions through a process called *confidential service healing*. Failed partitions are automatically rebuilt on healthy hardware using attested TLS and Intel SGX enclaves to protect secrets during recovery.

### Cost

There are no extra costs associated with the built-in high availability in Managed HSM. The pricing is based on the number of HSM pools and the number of operations performed. For more information, see [Azure Managed HSM pricing](https://azure.microsoft.com/pricing/details/key-vault).

### Behavior when all partitions are healthy

This section describes what to expect when Managed HSM pools are operational and no partitions are unavailable.

- **Traffic routing**: Managed HSM automatically manages traffic routing across its three partitions. During normal operations, requests are distributed across partitions transparently.

- **Data replication**: All data, including keys, role assignments, and access control policies, is synchronously replicated across all three partitions. This ensures consistency and availability even if a partition becomes unavailable.

### Behavior during a partition failure

This section describes what to expect when one or more partitions become unavailable.

- **Detection and response**: The Managed HSM service is responsible for detecting partition failures and automatically responding to them. You don't need to take any action during a partition failure.

- **Active requests**: During a partition failure, requests in flight to the affected partition might fail and require client applications to retry them. To minimize effects of partition outages, client applications should follow [transient fault handling practices](#resilience-to-transient-faults).

- **Expected data loss**: No data loss is expected during a partition failure because of synchronous replication across partitions.

- **Expected downtime**: For read operations and most cryptographic operations, there should be minimal to no downtime during a partition failure. The remaining healthy partitions continue to serve requests.

- **Traffic rerouting**: Managed HSM automatically reroutes traffic away from the affected partition to healthy partitions without requiring any customer intervention.

### Partition recovery

When the affected partition recovers, Managed HSM automatically restores operations through confidential service healing. This process:

1. Creates a new service instance on healthy hardware.
2. Establishes an attested TLS connection with the primary partition.
3. Securely exchanges credentials and cryptographic material.
4. Seals the service data to the new CPU.

The Azure platform fully manages this process and doesn't require any customer intervention.

## Resilience to availability zone failures

Managed HSM's high availability is based on rack-level distribution within a datacenter, not explicit availability zone deployment. Each partition runs on a separate server in a different rack, which protects against rack-level failures such as power supply or network switch issues.

If you need to be resilient to datacenter- or availability zone-wide outages, consider using one of the approaches for [resilience to region-wide failures](#resilience-to-region-wide-failures).

## Resilience to region-wide failures

Managed HSM resources are deployed into a single Azure region. If the region becomes unavailable, your Managed HSM is also unavailable. However, there are approaches you can use to help ensure resilience to region outages.

### Multi-region replication

Managed HSM supports optional [multi-region replication](/azure/key-vault/managed-hsm/multi-region-replication), which allows you to extend a Managed HSM pool from one Azure region (the *primary region*) to a second Azure region (the *extended region*). Once configured:

- Both regions are active and able to serve requests.
- Key material, roles, and permissions are automatically replicated between regions.
- Requests are routed to the closest available region using Azure Traffic Manager.
- The combined SLA increases.

#### Requirements

- **Region support**: All Azure Managed HSM regions are supported as primary regions. There's no dependency on Azure region pairings.

    Managed HSM doesn't support all regions as extended regions. For more information, see [Azure region support](/azure/key-vault/managed-hsm/multi-region-replication#azure-region-support).

- **Maximum number of regions:** You can add one extended region, for a maximum of two regions in total.

#### Cost

Multi-region replication incurs extra billing because a second HSM pool is consumed in the extended region. For more information, see [Azure Managed HSM pricing](https://azure.microsoft.com/pricing/details/key-vault).

#### Configure multi-region replication

- **Add an extended region:** For details about adding an extended region to an existing primary region, see [Extend a primary HSM into an extended region](/azure/key-vault/managed-hsm/multi-region-replication#azure-cli-commands).

    Extending a Managed HSM to another region might take up to 30 minutes.

- **Remove an extended region:** For details about removing an extended region from an existing primary region, see [Remove an extended region from the primary HSM](/azure/key-vault/managed-hsm/multi-region-replication#remove-an-extended-region-from-the-primary-hsm).

#### Behavior when all regions are healthy

When multi-region replication is enabled and both regions are operational:

- **Traffic routing**: All regions can serve requests. Azure Traffic Manager routes requests to the region with the closest geographical proximity or lowest latency.

    If you use Private Link, configure private endpoints in both regions for optimal routing during failover. For more information, see [Private link behavior with multi-region replication](/azure/key-vault/managed-hsm/multi-region-replication#private-link-behavior-with-multi-region-replication).

- **Data replication**: All changes to keys, role definitions, and role assignments are replicated asynchronously to the extended region within six minutes. Wait six minutes after creating or updating a key before using it in the extended region.

#### Behavior during a region failure

When multi-region replication is enabled and one region experiences an outage:

- **Detection and response**: Azure Traffic Manager detects the unhealthy region and routes future requests to the healthy region. DNS records have a five-second TTL, though clients caching DNS lookups might experience slightly longer failover times.

[!INCLUDE [Region down notification (Service Health only)](./includes/reliability-region-down-notification-service-include.md)]

- **Active requests**: Requests in flight to the affected region might fail and require retrying.

- **Expected data loss**: There might be data loss for changes made within six minutes before the region failure if those changes hadn't completed replication.

- **Expected downtime:** Both read and write operations remain available in the healthy region during failover.

    Client applications that are close to the unhealthy region might continue to be directed to that region until the DNS records are updated, but this update takes place within approximately five seconds. To minimize the failover time, clients should avoid caching DNS lookups for longer than the DNS record's TTL.

- **Rerouting:** Azure Traffic Manager automatically reroutes requests to the healthy region.

#### Region recovery

When the affected region recovers, Managed HSM automatically resumes operations. The Traffic Manager begins routing requests to both regions again based on proximity.

#### Test for region failures

Managed HSM fully manages traffic routing, failover, and failback for region failures, so you don't need to validate region failure processes or provide any further input.

### Custom multi-region solutions for resiliency

If multi-region replication isn't suitable for your needs, you can implement manual disaster recovery. This requires:

- **The security domain** of the source HSM.
- **The private keys** (at least the quorum number) that encrypt the security domain.
- **A recent full HSM backup** from the source HSM.

To perform disaster recovery:

1. Create a new Managed HSM instance in a different region.
1. Activate security domain recovery mode and upload the security domain.
1. Take a backup of the new HSM (required before restore).
1. Restore the backup from the source HSM.

> [!IMPORTANT]
> The new HSM has a different name and service endpoint URI. You must update your application configuration to use the new location.

For detailed disaster recovery procedures, see [Managed HSM disaster recovery](/azure/key-vault/managed-hsm/disaster-recovery-guide).

## Backup and restore

Managed HSM supports full backup and restore of all keys, versions, attributes, tags, and role assignments. Backups are stored in an Azure Storage account. If your region supports it, we recommend you back up your Managed HSM to an Azure Storage account that has geo-redundant storage (GRS) enabled.

Backups are encrypted using cryptographic keys associated with the HSM's security domain and can only be restored to an HSM with the same security domain.

Managed HSM doesn't support scheduling backups, but you can build your own scheduler by using a service like Azure Functions or Azure Automation.

While a backup is in progress, the HSM might not operate at full throughput because some partitions are busy performing the backup operation.

For detailed backup and restore procedures, see [Full backup and restore](/azure/key-vault/managed-hsm/backup-restore).

## Resilience to accidental deletion

Managed HSM provides two key recovery features to prevent accidental or malicious deletion:

- **Soft-delete**: When you delete an HSM or a key, it remains recoverable for a configurable retention period (7 to 90 days, default 90 days). Soft-delete is always enabled and can't be disabled.

    > [!NOTE]
    > Soft-deleted Managed HSM resources continue to incur billing until they're purged.

- **Purge protection**: When enabled, prevents permanent deletion of your Managed HSM and its keys until the retention period elapses. Purge protection can't be disabled or overridden by anyone, including Microsoft.

We strongly recommend enabling purge protection for production environments. For more information, see [Managed HSM soft-delete and purge protection](/azure/key-vault/managed-hsm/recovery).

## Resilience to service maintenance

Managed HSM handles service maintenance, including firmware updates, patching, and hardware healing, without customer intervention. During maintenance:

- Partitions might be temporarily unavailable while updates are applied.
- At least two of three partitions remain available during routine maintenance.
- Your client applications should implement retry logic to handle brief interruptions.

The confidential service healing process ensures that secrets are never exposed during maintenance operations.

## Service-level agreement

[!INCLUDE [SLA description](includes/reliability-service-level-agreement-include.md)]

Managed HSM provides a standard SLA for single-region deployments. When you enable [multi-region replication](/azure/key-vault/managed-hsm/multi-region-replication), the combined SLA for both regions increases.

## Related content

- [Managed HSM overview](/azure/key-vault/managed-hsm/overview)
- [Multi-region replication](/azure/key-vault/managed-hsm/multi-region-replication)
- [Managed HSM disaster recovery](/azure/key-vault/managed-hsm/disaster-recovery-guide)
- [Full backup and restore](/azure/key-vault/managed-hsm/backup-restore)
- [Security domain in Managed HSM overview](/azure/key-vault/managed-hsm/security-domain)
- [Managed HSM soft-delete and purge protection](/azure/key-vault/managed-hsm/recovery)
- [Azure Managed HSM Scaling Guidance](/azure/key-vault/managed-hsm/scaling-guidance)
- [Reliability in Azure](./overview.md)
