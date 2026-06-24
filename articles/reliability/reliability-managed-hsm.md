---
title: Reliability in Azure Key Vault Managed HSM
description: Learn how to make Azure Key Vault Managed HSM resilient to a variety of potential outages and problems, including transient faults, partition failures, availability zone outages, region outages, and service maintenance, and learn about backup and restore.
author: msmbaldwin
ms.author: mbaldwin
ms.topic: reliability-article
ms.custom: subject-reliability
ms.service: azure-key-vault
ms.subservice: managed-hsm
ms.date: 04/22/2026
---

# Reliability in Azure Key Vault Managed HSM

[Azure Key Vault Managed HSM](/azure/key-vault/managed-hsm/overview) is a fully managed, highly available, single-tenant, standards-compliant cloud service that enables you to safeguard cryptographic keys for your cloud applications by using hardware security modules (HSMs) that are validated to FIPS 140-3 Level 3. Managed HSM provides a range of built-in reliability features to help ensure that your keys stay available.

[!INCLUDE [Shared responsibility](includes/reliability-shared-responsibility-include.md)]

This article describes how Managed HSM is resilient to various potential outages and problems, including transient faults, partition failures, and region outages. It also describes how to use backups and the security domain for disaster recovery, how recovery features protect against accidental deletion, and key information about the Managed HSM service-level agreement (SLA).

## Production deployment recommendations

For production workloads, we recommend that you:

> [!div class="checklist"]
> - Download and securely store the [security domain](/azure/key-vault/managed-hsm/security-domain) immediately after provisioning your Managed HSM. You need the security domain for disaster recovery.
> - Establish a multiperson quorum for the security domain with at least three key holders.
> - Enable [purge protection](/azure/key-vault/managed-hsm/recovery) to prevent accidental or malicious deletion.
> - Implement regular [backups](/azure/key-vault/managed-hsm/backup-restore) to an Azure Storage account, and use geo-redundant storage in supported regions.
> - Enable [multi-region replication](/azure/key-vault/managed-hsm/multi-region-replication) for mission-critical workloads that require a higher SLA.

## Reliability architecture overview

When you use Managed HSM, you deploy an *instance*, which is sometimes called a *pool*.

The architecture of Managed HSM is designed for high availability and durability.

- **Single-tenant isolation:** Each Managed HSM instance is dedicated to a single customer and consists of a cluster of multiple HSM partitions that are cryptographically isolated.

- **Triple-redundant partitions:** A Managed HSM pool consists of three load-balanced HSM partitions distributed across separate racks within a datacenter. This distribution provides redundancy against hardware failures and ensures that the loss of a single component, such as a rack's power supply or network switch, doesn't affect all partitions.

- **Confidential computing:** Each service instance runs in a trusted execution environment (TEE) that uses Intel SGX enclaves. Microsoft personnel, including individuals with physical access to the servers, can't access your cryptographic key material.

- **Automatic healing:** If a hardware failure or other issue affects one of the three partitions, the service automatically rebuilds the affected partition on healthy hardware without any customer intervention and without exposing secrets.

To learn how Managed HSM implements these capabilities, see [Key sovereignty, availability, performance, and scalability in Managed HSM](/azure/key-vault/managed-hsm/managed-hsm-technical-details).

### Security domain

The [security domain](/azure/key-vault/managed-hsm/security-domain) is a critical component of Managed HSM for disaster recovery purposes. It's an encrypted blob that contains all the credentials needed to rebuild a Managed HSM instance from scratch, including the partition owner key, the partition credentials, the data-wrapping key, and an initial backup of the HSM.

> [!IMPORTANT]
> Without the security domain, disaster recovery isn't possible. Microsoft has no way to recover the security domain and can't access your keys without it.

Security domains are a critical part of the security and reliability of your Managed HSM. We recommend that you follow these best practices:

- **Generate keys securely:** For production environments, generate the RSA key pairs that protect the security domain in an air-gapped environment, such as an on-premises HSM or an isolated workstation.

- **Store offline:** Store security domain keys on encrypted USB drives or other offline storage, with each key share on a separate device in separate geographical locations.
- **Establish a multiperson quorum:** Use at least three key holders to prevent any single person from having access to all quorum keys, and to avoid a dependency on any single person.

For more information, see [Security domain in Managed HSM overview](/azure/key-vault/managed-hsm/security-domain).

## Resilience to transient faults

[!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)]

When you use Azure services that integrate with Managed HSM, those services handle transient faults automatically.

If you build custom applications that integrate with Managed HSM, consider the following best practices to handle any transient faults that might occur:

- **Use the Microsoft-provided SDKs** for Azure Key Vault, which include built-in retry mechanisms. SDKs are available for [.NET](/azure/key-vault/managed-hsm/quickstart-dotnet), [Python](/azure/key-vault/managed-hsm/quickstart-python), and [JavaScript](/azure/key-vault/managed-hsm/quickstart-javascript).

- **Implement retry logic**, including exponential backoff, for any code that interacts directly with Managed HSM.

- **Reduce the number of direct dependencies on Managed HSM.** Cache cryptographic operation results when possible to reduce direct requests to Managed HSM. Perform public-key operations, such as encryption, wrapping, and verification, locally by caching the public key material. Performing the operations locally reduces the dependency on your Managed HSM and reduces the likelihood of transient faults interrupting these operations.

If you use Managed HSM in high-throughput scenarios, Managed HSM doesn't throttle cryptographic operations. It uses its HSM hardware to full capacity. Each Managed HSM instance has three partitions. During maintenance or healing operations, one partition might be unavailable. For capacity planning, assume that two partitions are available. If you require guaranteed throughput, plan based on one partition being available. Monitor the [Managed HSM Availability metric](/azure/key-vault/managed-hsm/logging-azure-monitor) to understand the health of the service.

To scale encryption for large data volumes, use a key hierarchy. Store only the key encryption key (KEK) in Managed HSM, and use it to wrap lower-level data encryption keys stored elsewhere in secure key storage.

For more information about performance benchmarks and capacity planning, see [Azure Managed HSM scaling guidance](/azure/key-vault/managed-hsm/scaling-guidance).

## Resilience to partition failures

Managed HSM achieves high availability through its triple-redundant architecture, where each HSM pool consists of three HSM partitions distributed across separate server racks within a datacenter. This rack-level distribution provides redundancy against localized hardware failures.

:::image type="complex" source="/azure/key-vault/managed-hsm/media/managed-hsm-architecture.png" alt-text="Diagram that shows a Managed HSM pool's three partitions, each on a separate physical server and in a different server rack." border="false" lightbox="/azure/key-vault/managed-hsm/media/managed-hsm-architecture.png":::
    At the top of the diagram is the HSM fabric controller. Three arrows, labeled service healing operations, each point downward to three physical server racks. Below the three physical server boxes, a large enclosed area spans the full width of the diagram. This area is labeled customer's cryptographic boundary. Inside this boundary, three containers are arranged horizontally, each positioned directly beneath one of the physical server boxes. Each container holds a FIPS 140-2 Level 3 HSM instance, and a TEE SGX enclave. The three TEE boxes are fully interconnected by bidirectional arrows labeled attested TLS.
:::image-end:::

When hardware failures or localized outages occur, Managed HSM automatically redirects your requests to healthy partitions and rebuilds affected partitions through a process called *confidential service healing*. Failed partitions are automatically rebuilt on healthy hardware by using attested TLS and Intel SGX enclaves to protect secrets during recovery.

### Cost

The built-in high availability in Managed HSM doesn't add any extra costs. Pricing is based on the number of HSM pools and the number of operations that you perform. For more information, see [Azure Managed HSM pricing](https://azure.microsoft.com/pricing/details/key-vault).

### Behavior when all partitions are healthy

This section describes what to expect when Managed HSM pools are operational and no partitions are unavailable.

- **Traffic routing:** Managed HSM automatically manages traffic routing across its three partitions. During normal operations, it transparently distributes requests across partitions.

- **Data replication:** Managed HSM synchronously replicates all data, including keys, role assignments, and access control policies, across all three partitions. This approach ensures consistency and availability even if a partition becomes unavailable.

### Behavior during a partition failure

This section describes what to expect when one or more partitions become unavailable.

- **Detection and response:** The Managed HSM service detects partition failures and automatically responds to them. You don't need to take any action during a partition failure.

- **Active requests:** During a partition failure, requests in flight to the affected partition might fail and require client applications to retry them. To minimize the effects of partition outages, client applications should follow [transient fault handling practices](#resilience-to-transient-faults).

- **Expected data loss:** No data loss is expected during a partition failure because of synchronous replication across partitions.

- **Expected downtime:** For read operations and most cryptographic operations, there should be minimal or no downtime during a partition failure. The remaining healthy partitions continue to serve requests.

- **Traffic rerouting:** Managed HSM automatically reroutes traffic away from the affected partition to healthy partitions without requiring any customer intervention.

### Partition recovery

When the affected partition recovers, Managed HSM automatically restores operations through confidential service healing. This process:

1. Creates a new service instance on healthy hardware.
1. Establishes an attested TLS connection with the primary partition.
1. Securely exchanges credentials and cryptographic material.
1. Seals the service data to the new CPU.

The Azure platform fully manages this process and doesn't require any customer intervention.

## Resilience to availability zone failures

High availability in Managed HSM is based on rack-level distribution within a datacenter, not explicit availability zone deployment. Each partition runs on a separate server in a different rack, which protects against rack-level failures such as power supply or network switch problems.

To protect against datacenter-wide or availability zone-wide outages, use one of the approaches described in [Resilience to region-wide failures](#resilience-to-region-wide-failures).

## Resilience to region-wide failures

Managed HSM resources are deployed into a single Azure region. If the region becomes unavailable, your Managed HSM is also unavailable. However, there are approaches you can use to help ensure resilience to region outages.

### Multi-region replication

Managed HSM supports optional [multi-region replication](/azure/key-vault/managed-hsm/multi-region-replication), which you can use to extend a Managed HSM pool from one Azure region (the *primary region*) to a second Azure region (the *extended region*). When you configure this feature:

- Both regions are active and can serve requests.
- Key material, roles, and permissions automatically replicate between regions.
- Azure Traffic Manager routes requests to the closest available region.
- The combined SLA increases.

#### Requirements

- **Region support:** All regions that support Managed HSM can be used as primary regions. There's no dependency on Azure region pairings.

    Managed HSM doesn't support all regions as extended regions. For more information, see [Azure region support](/azure/key-vault/managed-hsm/multi-region-replication#azure-region-support).

- **Maximum number of regions:** You can add one extended region, for a maximum of two regions in total.

#### Cost

Multi-region replication incurs extra billing because the extended region consumes a second HSM pool. For more information, see [Azure Managed HSM pricing](https://azure.microsoft.com/pricing/details/key-vault).

#### Configure multi-region replication

- **Add an extended region:** For details about adding an extended region to an existing primary region, see [Extend a primary HSM into an extended region](/azure/key-vault/managed-hsm/multi-region-replication#azure-cli-commands).

    Extending a Managed HSM to another region might take up to 30 minutes.

- **Remove an extended region:** For details about removing an extended region from an existing primary region, see [Remove an extended region from the primary HSM](/azure/key-vault/managed-hsm/multi-region-replication#remove-an-extended-region-from-the-primary-hsm).

#### Behavior when all regions are healthy

This section describes what to expect when you configure multi-region replication, and both regions are operational.

- **Traffic routing:** All regions can serve requests. Azure Traffic Manager routes requests to the region with the closest geographical proximity or lowest latency.

    If you use Azure Private Link to access Managed HSM, configure private endpoints in both regions for optimal routing during failover. For more information, see [Private link behavior with multi-region replication](/azure/key-vault/managed-hsm/multi-region-replication#private-link-behavior-with-multi-region-replication).

- **Data replication:** All changes to keys, role definitions, and role assignments are replicated asynchronously to the extended region within six minutes. Wait six minutes after creating or updating a key before using it in the extended region.

#### Behavior during a region failure

This section describes what to expect when you configure multi-region replication, and there's an outage in one of the replica regions.

- **Detection and response:** Azure Traffic Manager detects the unhealthy region and routes future requests to the healthy region. DNS records have a five-second time to live (TTL), though clients caching DNS lookups might experience slightly longer failover times.

[!INCLUDE [Region down notification (Service Health only)](./includes/reliability-region-down-notification-service-include.md)]

- **Active requests:** Requests in flight to the affected region might fail and require retrying.

- **Expected data loss:** Changes made within six minutes before the region failure might not be replicated to the extended region. Those changes could be lost if the primary region is unrecoverable.

- **Expected downtime:** Both read and write operations remain available in the healthy region during failover.

    Client applications that are close to the unhealthy region might continue to be directed to that region until the DNS records are updated, but this update takes place within approximately five seconds. To minimize the failover time, clients should avoid caching DNS lookups for longer than the DNS record's TTL.

- **Rerouting:** Azure Traffic Manager automatically reroutes requests to the healthy region.

#### Region recovery

When the affected region recovers, Managed HSM automatically resumes operations. Azure Traffic Manager begins routing requests to both regions again based on proximity.

#### Test for region failures

Managed HSM fully manages traffic routing, failover, and failback for region failures, so you don't need to validate region failure processes or provide any further input.

### Custom multi-region solutions for resiliency

If multi-region replication isn't suitable for your needs, you can implement manual disaster recovery. This approach requires:

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

Managed HSM supports full backup and restore of all keys, versions, attributes, tags, and role assignments. Backups are stored in an Azure Storage account. If your region supports it, we recommend that you back up your Managed HSM to an Azure Storage account that has geo-redundant storage (GRS) enabled.

The HSM encrypts backups by using cryptographic keys associated with the HSM's security domain. You can only restore backups to an HSM with the same security domain.

Managed HSM doesn't support scheduling backups, but you can build your own scheduler by using a service like Azure Functions or Azure Automation.

While a backup is in progress, the HSM might not operate at full throughput because some partitions are busy performing the backup operation.

For detailed backup and restore procedures, see [Full backup and restore](/azure/key-vault/managed-hsm/backup-restore).

## Resilience to accidental deletion

Managed HSM provides two key recovery features to prevent accidental or malicious deletion.

- **Soft-delete:** Deleted HSMs and keys aren't immediately purged. They remain recoverable for a configurable retention period of 7 to 90 days (default: 90 days). Soft-delete is always enabled and can't be disabled.

    > [!NOTE]
    > Soft-deleted Managed HSM resources continue to incur billing until they're purged.

- **Purge protection:** When enabled, it prevents permanent deletion of your Managed HSM and its keys until the retention period elapses. Purge protection can't be disabled or overridden by anyone, including Microsoft.

We strongly recommend enabling purge protection for production environments. For more information, see [Managed HSM soft-delete and purge protection](/azure/key-vault/managed-hsm/recovery).

## Resilience to service maintenance

Managed HSM handles service maintenance, including firmware updates, patching, and hardware healing, without customer intervention. During maintenance:

- The service might temporarily make partitions unavailable while it applies updates.
- At least two of three partitions remain available during routine maintenance.
- Your client applications should implement retry logic to handle brief interruptions.

The confidential service healing process ensures that the service never exposes secrets during maintenance operations.

## Service-level agreement

[!INCLUDE [SLA description](includes/reliability-service-level-agreement-include.md)]

Managed HSM provides a standard availability SLA for single-region deployments. Enabling [multi-region replication](/azure/key-vault/managed-hsm/multi-region-replication) raises the overall expected uptime, because requests can be served from either region if one becomes unavailable.

## Related content

- [What is Azure Key Vault Managed HSM?](/azure/key-vault/managed-hsm/overview)
- [Enable multi-region replication](/azure/key-vault/managed-hsm/multi-region-replication)
- [Managed HSM disaster recovery](/azure/key-vault/managed-hsm/disaster-recovery-guide)
- [Full backup and restore](/azure/key-vault/managed-hsm/backup-restore)
- [Security domain in Managed HSM overview](/azure/key-vault/managed-hsm/security-domain)
- [Managed HSM soft-delete and purge protection](/azure/key-vault/managed-hsm/recovery)
- [Azure Managed HSM Scaling Guidance](/azure/key-vault/managed-hsm/scaling-guidance)
- [What is Azure reliability documentation?](./overview.md)
