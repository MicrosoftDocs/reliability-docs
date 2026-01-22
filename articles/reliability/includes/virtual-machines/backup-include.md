---
 title: Description of virtual machine backup options
 description: Description of virtual machine backup options
 author: anaharris-ms
 ms.service: azure
 ms.topic: include
 ms.date: 10/10/2025
 ms.author: anaharris
 ms.custom: include file
---

Azure Backup provides native backup support for VMs. Azure Backup creates and manages backups, and provides application-consistent protection for the entire VM, including all attached disks. A VM backup solution with Azure Backup is ideal when you need coordinated backup of multiple disks or application-aware backups. However, for database workloads, consider application-specific backup solutions that provide transaction-consistent protection and faster recovery options.

With Azure Backup for VMs, you can customize the backup frequency, retention duration, and storage configuration to suit your needs. For more information, see [Azure Backup for VMs](/azure/backup/backup-azure-vms-introduction).

Backup also supports disks that are attached to VMs. For more information, see [Overview of Azure Disk Backup](/azure/backup/disk-backup-overview).
