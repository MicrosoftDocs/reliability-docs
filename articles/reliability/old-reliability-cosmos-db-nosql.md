## Availability zone support

### SLA improvements

<!-- TODO decide what to do with this section given it talks about the details of SLAs so much (and we're not supposed to) -->

To increase resiliency and availability of an Azure Cosmos DB account, you can implement a layering approach, which includes:

- Enabling zone redundancy.
- Adding one or more additional regions.
- Enabling multi-region writes.

The layering approach protects the account at each step of the way - from 4 9's availability for a single-region configuration without zone redundancy, through 4 and half 9's for single region with zone redundancy, all the way to 5 9's of availability for multi-region configuration with the multi-region write option enabled.

The following table contains a summary of SLAs for each configuration:

|KPI|Single-region writes without availability zones|Single-region writes with availability zones|Multiple-region, single-region writes without availability zones|Multiple-region, single-region writes with availability zones|Multiple-region, multiple-region writes with or without availability zones|
|---------|---------|---------|---------|---------|---------|
|Write availability SLA | 99.99% | 99.995% | 99.99% | 99.995% | 99.999% |
|Read availability SLA  | 99.99% | 99.995% | 99.999% | 99.999% | 99.999% |
|Zone failures: data loss | Data loss | No data loss | No data loss | No data loss | No data loss |
|Zone failures: availability | Availability loss | No availability loss | No availability loss | No availability loss | No availability loss |
|Regional outage: data loss | Data loss |  Data loss | Dependent on consistency level. For more information, see [Consistency, availability, and performance tradeoffs](/azure/cosmos-db/consistency-levels). | Dependent on consistency level. For more information, see [Consistency, availability, and performance tradeoffs](/azure/cosmos-db/consistency-levels). | Dependent on consistency level. For more information, see [Consistency, availability, and performance tradeoffs](/azure/cosmos-db/consistency-levels).
|Regional outage: availability | Availability loss | Availability loss | No availability loss for read region failure, temporary for write region failure | No availability loss for read region failure, temporary for write region failure | No read availability loss, temporary write availability loss in the affected region |
|Price (***1***) | Not applicable | Provisioned RU/s x 1.25 rate | Provisioned RU/s x *N* regions | Provisioned RU/s x 1.25 rate x *N* regions (***2***) | Multiple-region write rate x *N* regions |

***1*** For serverless accounts, RUs are multiplied by a factor of 1.25.

***2*** The 1.25 rate applies only to regions in which you enable availability zones.

## Cross-region disaster recovery and business continuity

### Service managed failover

Service-managed failover allows Azure Cosmos DB to fail over the write region of a multiple-region account in order to preserve business continuity at the cost of data loss, as described earlier in the [Durability](#durability) section. Regional failovers are detected and handled in the Azure Cosmos DB client. They don't require any changes from the application. For instructions on how to enable multiple read regions and service-managed failover, see [Manage an Azure Cosmos DB account using the Azure portal](/azure/cosmos-db/how-to-manage-database-account).

> [!IMPORTANT]
> If you have chosen single-region write configuration with multiple read regions, we strongly recommend that you configure the Azure Cosmos DB accounts used for production workloads to *enable service-managed failover*. This configuration enables Azure Cosmos DB to fail over the account databases to available regions.
> In the absence of this configuration, the account will experience loss of write availability for the whole duration of the write region outage. Manual failover won't succeed because of a lack of region connectivity.

> [!WARNING]
> Even with service-managed failover enabled, partial outage may require manual intervention for the Azure Cosmos DB service team. In these scenarios, it may take up to 1 hour (or more) for failover to take effect. For better write availability during partial outages, we recommend enabling availability zones in addition to service-managed failover.

### Multiple write regions

You can configure Azure Cosmos DB to accept writes in multiple regions. This configuration is useful for reducing write latency  in geographically distributed applications.

When you configure an Azure Cosmos DB account for multiple write regions, strong consistency isn't supported and write conflicts might arise. For more information on how to resolve these conflicts, see [Conflict types and resolution policies when using multiple write regions](/azure/cosmos-db/conflict-resolution-policies).

> [!IMPORTANT]
> Updating same Document ID frequently (or recreating same document ID frequently after TTL or delete) will have an effect on replication performance due to increased number of conflicts generated in the system.  

#### Conflict-resolution region

When an Azure Cosmos DB account is configured with multiple-region writes, one of the regions will act as an arbiter in write conflicts.

#### Best practices for multi-region writes
<!-- TODO consider moving into the multi-region writes doc -->

Here are some best practices to consider when you're writing to multiple regions.

##### Keep local traffic local

When you use multiple-region writes, the application should issue read and write traffic that originates in the local region strictly to the local Cosmos DB region. For optimal performance, avoid cross-region calls.

It's important for the application to minimize conflicts by avoiding the following antipatterns:

* Sending the same write operation to all regions to increase the odds of getting a fast response time

* Randomly determining the target region for a read or write operation on a per-request basis

* Using a round-robin policy to determine the target region for a read or write operation on a per-request basis.

##### Avoid dependency on replication lag

You can't configure multiple-region write accounts for strong consistency. The region that's being written to responds immediately after Azure Cosmos DB replicates the data locally while asynchronously replicating the data globally.

Though it's infrequent, a replication lag might occur on one or a few partitions when you're geo-replicating data. Replication lag can occur because of a rare blip in network traffic or higher-than-usual rates of conflict resolution.

For instance, an architecture in which the application writes to Region A but reads from Region B introduces a dependency on replication lag between the two regions. However, if the application reads and writes to the same region, performance remains constant even in the presence of replication lag.

##### Evaluate session consistency usage for write operations

In session consistency, you use the session token for both read and write operations.

For read operations, Azure Cosmos DB sends the cached session token to the server with a guarantee of receiving data that corresponds to the specified (or a more recent) session token.

For write operations, Azure Cosmos DB sends the session token to the database with a guarantee of persisting the data only if the server has caught up to the provided session token. In single-region write accounts, the write region is always guaranteed to have caught up to the session token. However, in multiple-region write accounts, the region that you write to might not have caught up to writes issued to another region. If the client writes to Region A with a session token from Region B, Region A won't be able to persist the data until it catches up to changes made in Region B.

It's best to use session tokens only for read operations and not for write operations when you're passing session tokens between client instances.

##### Mitigate rapid updates to the same document

The server's updates to resolve or confirm the absence of conflicts can collide with writes triggered by the application when the same document is repeatedly updated. Repeated updates in rapid succession to the same document experience higher latencies during conflict resolution.

Although occasional bursts in repeated updates to the same document are inevitable, you might consider exploring an architecture where new documents are created instead if steady-state traffic sees rapid updates to the same document over an extended period.

### Read and write outages

Clients of single-region accounts will experience loss of read and write availability until service is restored.

<!-- TODO move this into the new document -->
Multiple-region accounts experience different behaviors depending on the following configurations and outage types.

- Single write region:
  - Read region outage:
    - **Availability impact:** All clients automatically redirect reads to other regions. There's no read or write availability loss for all configurations. The exception is a configuration of two regions with strong consistency, which loses write availability until restoration of the service. Or, *if you enable service-managed failover*, the service marks the region as failed and a failover occurs.
    - **Durability impact:** No data loss.
    - **What to do:** During the outage, ensure that there are enough provisioned Request Units (RUs) in the remaining regions to support read traffic. When the outage is over, readjust provisioned RUs as appropriate.
  - Write region outage:
    - **Availability impact:** Clients redirect reads to other regions. *Without service-managed failover*, clients experience write availability loss. Restoration of write availability occurs automatically when the outage ends. *With service-managed failover*, clients experience write availability loss until the service manages a failover to a new write region that you select.
    - **Durability impact:** If you don't select the strong consistency level, the service might not replicate some data to the remaining active regions. This replication depends on the [consistency level](/azure/cosmos-db/consistency-levels#rto) that you select. If the affected region suffers permanent data loss, you could lose unreplicated data.
    - **What to do:** During the outage, ensure that there are enough provisioned RUs in the remaining regions to support read traffic. *Don't* trigger a manual failover during the outage, because it can't succeed. When the outage is over, readjust provisioned RUs as appropriate. Accounts that use the API for NoSQL might also recover unreplicated data in the failed region from your [conflict feed](/azure/cosmos-db/how-to-manage-conflicts#read-from-conflict-feed).
- Multiple write regions
  - Outage of any write region:
    - **Availability impact:** There's a possibility of temporary loss of write availability, which is analogous to a single write region with service-managed failover. The failover of the [conflict-resolution region](#conflict-resolution-region) might also cause a loss of write availability if a high number of conflicting writes happen at the time of the outage.
    - **Durability impact:** Recently updated data in the failed region might be unavailable in the remaining active regions, depending on the selected [consistency level](/azure/cosmos-db/consistency-levels). If the affected region suffers permanent data loss, you might lose unreplicated data.
    - **What to do:** During the outage, ensure that there are enough provisioned RUs in the remaining regions to support more traffic. When the outage is over, readjust provisioned RUs as appropriate. If possible, Azure Cosmos DB automatically recovers unreplicated data in the failed region. This automatic recovery uses the conflict resolution method that you configure for accounts that use the API for NoSQL. For accounts that use other APIs, this automatic recovery uses *last write wins*.

#### Outage detection, notification, and management

<!-- TODO move this into the new document -->

Multiple-region accounts experience different behaviors, as described in the following table.

**Single write region without service-managed failover:**

- **Read region outage:** All clients redirect reads to other regions automatically. No read or write availability loss (except with strong consistency and fewer than two read regions remaining, which can affect write availability). No data loss.
  - Ensure enough provisioned RUs in remaining regions. Don't trigger manual failover.

- **Write region outage:** Clients experience write availability loss. Data replication to other regions depends on your consistency level; unreplicated data might be lost if the region suffers permanent data loss.
  - Ensure enough provisioned RUs in remaining regions. Don't trigger manual failover. Readjust RUs when service is restored.

**Single write region with service-managed failover:**

- **Read region outage:** All clients redirect reads to other regions automatically. No read or write availability loss (except with strong consistency and fewer than two read regions remaining, which can affect write availability). No data loss.
  - Ensure enough provisioned RUs in remaining regions.

- **Write region outage:** Clients experience write availability loss until Azure Cosmos DB fails over to a new write region. Data replication depends on your consistency level; unreplicated data might be lost if the region suffers permanent data loss.
  - Ensure enough provisioned RUs in remaining regions. Don't trigger manual failover. When service is restored, move the write region back to the original region and readjust RUs. For API for NoSQL accounts, recover unreplicated data from your conflict feed if possible.

**Multiple write regions:**

- **Write region outage:** Recently updated data might become unavailable in other regions. The potential data loss window depends on your consistency level (eventual, consistent prefix, and session consistency guarantee less than 15 minutes of staleness; bounded staleness depends on your configuration). Unreplicated data might be lost if the region suffers permanent data loss.
  - Ensure enough provisioned RUs in remaining regions. When service is restored, readjust RUs. Azure Cosmos DB automatically recovers unreplicated data using your configured conflict resolution method (or last write wins for non-NoSQL APIs).

> [!IMPORTANT]
> Don't invoke manual failover during an Azure Cosmos DB outage on either the source or destination region. Manual failover requires region connectivity to maintain data consistency, so it won't succeed.
