---
title: "Write a Reliability Guide"
description: Learn how to write reliability guides for Azure services. Understand the structure, required sections, and best practices for documenting resilience to transient faults, availability zone failures, and region-wide failures.
author: glynnniall
ms.author: glynnniall
ms.date: 02/04/2026
ms.service: learn
ms.topic: contributor-guide
ms.custom: internal-contributor-guide
---

# Write a reliability guide

This guide helps you:

> [!div class="checklist"]
>
> - Decide whether to write a _Reliability_ guide for a product or service.
> - Identify what content to include in a _Reliability_ guide.
> - Find the right location in the TOC for your guide.

## What is this pattern?

*Reliability* guides provide the customer with information about how a product works from a platform reliability perspective. The concept of reliability includes such topics as availability zone support, multi-region deployments, outage scenarios, backups, service maintenance, proactive preparation for disasters, and testing for different kinds of outages.

Portions of this document can link to other documents, such as articles that provide guidance about availability zone migration. But we recommend that you keep all reliability information inside one document. If a section becomes too large and difficult to include, you can move that information into a new document and provide a link to it from an appropriate section.

Standardizing this guide type has several advantages:

- **Reader**: Reliability guidance is presented in a consistent way across different Microsoft services. This consistency helps when a customer is reviewing multiple services together and needs to understand how they behave both individually and together.
- **Author**: The structure helps the author determine which content to include.
- **Portfolio managers**: The well-defined content helps us measure coverage and identify content gaps.

Reliability guides are targeted at two distinct personas:

 - **Architects**, who need to understand the details of how a service works to make informed architectural decisions, to decide on the appropriate configuration, and to plan across a whole solution.
- **Engineers responsible for business continuity**, who need to plan their high availability and disaster recovery strategies and need to understand the exact processes that Azure services follow during different kinds of situations.

## When to use this pattern

Reliability guides are conceptual content. They don't contain how-to steps. They explain how a service works from a reliability and resiliency perspective.

Reliability guides offer the following key advantages:

- Drive consumption to Microsoft-owned services through Microsoft documentation.
- Provide guidance and support to help customers take advantage of reliability offerings.
- Help customers experience the benefits of Microsoft Azure reliability support.

Use this pattern to inform the customer about:

- How the service supports availability zones, which is a key element of Azure's resiliency strategy.
- What reliability support is offered and not offered for a particular service.
- How to discover and implement reliability support for resources.
- How to use reliability offerings to achieve redundancy and resiliency during different types of outages.
- How to prevent data loss and downtime in the case of a local or regional outage.

### Examples of this pattern

| Example | Analysis |
| ----- | ----- |
| [Reliability in Azure Bastion](/azure/reliability/reliability-bastion) |  The article meets all template requirements and it's a good example of a basic service guide. The service supports zone-redundant and zonal deployments with customer-selected zones, and is a single-region service.  |
| [Reliability in Azure App Service](/azure/reliability/reliability-app-service) | The article meets all template requirements. The service supports zone-redundant deployments with Microsoft-selected zones, and is a single-region service. |
| [Reliability in Azure Kubernetes Service (AKS)](/azure/reliability/reliability-aks) | The article meets all template requirements. The service supports zone-redundant and zonal deployments at different layers, and is a single-region service. |
| [Reliability in Azure API Management](/azure/reliability/reliability-api-management) | This article meets all template requirements and it's a good example of a complex service guide. The service supports multiple types of availability zone support, and it's a multi-region service that has Microsoft-managed multi-region support. |

## Summary of how to use this pattern

In documentation, this pattern might be a single article or a series of articles that a main document links to.

Here's a quick summary of all pattern elements:

> [!div class="checklist"]
>
> - To promote consistency, [start from a template](#use-a-template).
> - Place the article under a **Reliability** [TOC node](#toc-placement) that's under a **Concepts** node.
> - Set the [metadata](#set-metadata) metadata, especially `ms.topic`, `ms.custom`, and `description`, appropriately for reliability guides.
> - Use the format "Reliability in \<product or service\>" for the [H1 headline](#h1-headline-and-introduction) and follow the instructions for the introductory section.
> - Include an H2 section for [production deployment recommendations](#production-deployment-recommendations).
> - (Optional) Include an H2 section for [reliability architecture overview](#reliability-architecture-overview).
> - Include an H2 section that discusses [resilience to transient faults](#resilience-to-transient-faults).
> - Include an H2 section that discusses [resilience to availability zone failures](#resilience-to-availability-zone-failures).
> - Include an H2 section that discusses [resilience to region-wide failures](#resilience-to-region-wide-failures).
> - (Optional) Include an H2 section for [backup and restore](#backup-and-restore) if your service supports backups.
> - (Optional) Include an H2 section for [resilience to service maintenance](#resilience-to-service-maintenance) if your service has special maintenance requirements.
> - Include an H2 section for [service-level agreement](#service-level-agreement).
> - Include a [related content](#related-content) section with a list of links to articles or actions to take.

## Details of how to use this pattern

This section repeats all the pattern elements but adds details, examples, and rationale.

### Use a template

To promote consistency, start with a template:

- **For Azure services that are deployed into a single region,** Use the [Reliability guide template](contribute-reliability-template.md).
- **For Azure services that are nonregional** (either deployed globally or deployed into a geographic area instead of a region), contact the Reliability Hub team to discuss how to proceed.
- **If your service offers multiple tiers or SKUs**, you can provide a single reliability guide that covers all tiers and use zone pivots for each tier. To see an example of how to do this, see the [reliability guide for Azure SQL Database](/azure/reliability/reliability-sql-database).

### TOC placement

- In the TOC that's accessible from [Reliability in Azure](/azure/reliability/overview), add your Reliability guide to the **Reliability by service** node, under **Reliability guidance**.
- The TOC for your service should contain a **Reliability** node, which is usually under a **Concepts** node. Add the reliability guide as a child of the **Reliability** node. The URL for the document should point to the document that the [Reliability in Azure](/azure/reliability/overview) TOC contains.

### Set metadata

- Set the value for `ms.topic` to be `reliability-article`.
- Set the value for `ms.custom` to be `subject-reliability`.
- Set the `description` metadata to summarize what the article covers. List only the H2 sections that are actually included in your article. Use this format:

  **Example (article with all optional sections):**

  ```yaml
  description: Learn how to make Azure [service-name] resilient to a variety of potential outages and problems, including transient faults, availability zone outages, region outages, and service maintenance, and learn about backup and restore.
  ```

  **Example (article without backup/restore and service maintenance sections):**

  ```yaml
  description: Learn how to make Azure [service-name] resilient to a variety of potential outages and problems, including transient faults, availability zone outages, and region outages.
  ```

### H1 (headline) and introduction

The headline (H1) is the primary heading at the top of the article.

Use the following format for the H1: "Reliability in \[service-name\]".

The introduction should consist of three paragraphs in the following order:

1. **Service introduction**: Begin with a brief 2-3 sentence introduction to the service. The FIRST mention of the service should be formatted as a link to the service overview, using "Azure \[service-name\]" format. For example:

   > [Azure Bastion](/azure/bastion/bastion-overview) is a fully managed platform as a service (PaaS) that you provision to provide high-security connections to virtual machines via a private IP address. It provides seamless RDP/SSH connectivity to your virtual machines directly over TLS from the Azure portal, or via the native SSH or RDP client that's already installed on your local computer.

2. **Shared responsibility**: Include the shared responsibility include file:

   ```markdown
   [!INCLUDE [Shared responsibility](includes/reliability-shared-responsibility-include.md)]
   ```

3. **Article scope**: Describe what the article contains, using the terms from the H2 section names. List only the topics actually covered in the article. For example:

   > This article describes how to make Azure Bastion resilient to a variety of potential outages and problems, including transient faults, availability zone failures, and region-wide failures. It also describes backup and recovery options, and highlights key information about the Azure Bastion service level agreement (SLA).

**Product naming convention**: Refer to the [Microsoft Product Style Guide](https://aka.ms/MPSG) to confirm how to refer to the service name, including on first use and subsequent uses.

### Production deployment recommendations

This section contains production deployment recommendations for your service.

- Many services have a Well-Architected Framework (WAF) service guide that contains detailed guidance about reliability configuration and other recommendations. To see if there is a WAF service guide for your service, go to [Well-Architected Framework perspective on Azure services](/azure/well-architected/service-guides/).

  If there is a WAF service guide, this section should refer to it using the following format:

  ```markdown
  The Azure Well-Architected Framework provides recommendations across reliability, security, cost, operations, and performance. To understand how these areas influence each other and contribute to a reliable \<service-name\> solution, see [Architecture best practices for \<service-name\>](URL).
  ```

- If your service does NOT have a WAF service guide, and you have guidance, try to organize it as checklist using the following format:

  ```markdown
  For production workloads, we recommend that you:

  > ![div class="checklist"]
  > - Recommendation 1
  > - Recommendation 2
  ```

### Reliability architecture overview

*Optional but strongly recommended section:* This section focuses on important elements of the service architecture that's relevant to the reliability. It doesn't provide a comprehensive review of the entire service architecture but introduces important reliability elements. Common items to include in this section are:

- **Resource model:** If there are multiple resources that a customer creates or manages, consider giving a brief description of the resources and link to more information in the product documentation. This is especially important where there might be different capabilities or guidance in different components.

- **Dependencies:** If your service requires the customer to deploy dependent resources, and the configuration of those dependent resources might affect the customer's reliability, please explicitly mention that.

  For example, you might need a customer to deploy a storage account to use the service, and that storage account's redundancy model (LRS/ZRS/GRS) will affect how resilient their overall solution is.

  Please link to the reliability guides for dependent services here too.

- **Redundancy:** Does the service have built-in redundancy across fault domains or hardware, such as by natively supporting multiple instances? If so, please mention that and describe it. Be explicit about whether redundancy requires the customer to configure the service (such as by enabling a high availability mode or adding a specific number of instances, or if redundancy handled automatically for them.

- **Infrastructure:** How does the customer's configuration map to physical infrastructure? The goal isn't to provide a lot of detail about the inner workings of a service, just to help customers understand important aspects of how the service works so they can make informed decisions about reliability configuration and so they can understand why the service behaves as it does.

Consider whether your service is simple (single or few resources for customers to deploy, limited decision points, platform-managed reliability with no configuration) or complex (customers have to deploy and configure resources at multiple layers, and configuration significantly affects the resiliency of their underlying infrastructure):

- **For simple services**, you can provide the necessary information directly within the H2.

  **Example:**

    ```markdown
    ## Reliability architecture overview

    By default, \[service-name\] achieves redundancy by spreading compute nodes and data throughout a single datacenter in the primary region. This approach protects your data in the event of a localized failure, such as a small-scale network or power failure, and even during the following events:

    - Customer initiated management operations that result in a brief downtime.
    - Service maintenance operations.

    *etc.*
    ```

- **For complex services**, we often suggest splitting the reliability architecture overview into two subsections:

  - *Logical architecture*, which describes the resources that the customer deploys and works with.

  - *Physical architecture*, which describes how these local resources map to lower-level infrastructure (such as VMs or nodes).

  We have a standard include file that introduces the subsections.

  **Example:**

    ```markdown
    ## Reliability architecture overview

    [!INCLUDE [Introduction to reliability architecture overview section](includes/reliability-architecture-overview-introduction-include.md)]

    ### Logical architecture

    The resource you deploy is an *xxx*, which represents *xyz*.

    ### Physical architecture

    Internally, when you deploy an *xxx*, \<service-name\> deploys two virtual machines (VMs), which are referred to as *instances*. You don't see or manage these VMs directly. The platform automatically manages instance creation, health monitoring, and replacement of unhealthy instances.
    ```

### Resilience to transient faults

This section explains how the service handles transient faults and provides guidance for customers.

**Structure:**

1. Begin with the transient faults include file:

   ```markdown
   ## Resilience to transient faults

   [!INCLUDE [Resilience to transient faults](includes/reliability-transient-fault-description-include.md)]
   ```

2. Provide service-specific guidance on transient fault handling, such as:
   - Built-in retry policies
   - Recommended retry strategies
   - Circuit breaker patterns
   - Timeout configurations

3. If your service hosts customer code or applications, explain how to avoid causing or propagating transient faults, and how to apply retries effectively. For example, App Service supports deployment slots, which avoid application downtime during deployments.

### Resilience to availability zone failures

This is a key section. It describes how the service works with Azure's availability zones (available in many, but not all, regions).

**Structure:**

1. Begin with the availability zone include file:

   ```markdown
   ## Resilience to availability zone failures

   [!INCLUDE [Resilience to availability zone failures](includes/reliability-availability-zone-description-include.md)]
   ```

2. Provide a high-level overview of how the service supports availability zones. For example, explain whether the service automatically spreads replicas across zones or requires customer configuration.

  **Critical requirements**: You must explicitly state which availability zone deployment model(s) the service supports:

   - **Zone redundancy**, which automatically spreads resources across multiple zones.

      For zone-redundant services, ensure the following information is clear:
      - Who picks the zones (the customer or Microsoft)
      - How many zones the resource uses (all zones in region, a specific number of zones, or a minimum number of zones)

      **Example:**

        ```markdown
        \[service-name\] can be configured to be *zone redundant*, which means your resources are spread across all of the availability zones in the region. Zone redundancy helps you achieve resiliency and reliability for your production workloads.
        ```

   - **Zonal**, in which the customer picks a single zone to use.

      For zonal services, add this include file somewhere within the section:

      ```markdown
      [!INCLUDE [Zonal resource description](includes/reliability-availability-zone-zonal-include.md)]
      ```

   - Supports **both models**.

   - Some services don't fit neatly into these categories. Please speak to the Reliability Hub team about how best to handle these situations.

> [!NOTE]
> You may be asked to provide an image here if it helps to explain how resources are distributed across availability zones. We will work with you to design and prepare the image.

#### Requirements

List any requirements that must be met to use availability zones with this service. At a minimum, zone support is only available in regions with zones. Commonly, specific SKUs are required for many services too. If availability zones are supported in all SKUs, or if the service has only one default SKU, mention this. Also mention any other requirements that must be met.

Each requirement should be a separate bullet point, with a bold lead-in.

Region support must be the FIRST bullet point.

- **If the service supports all availability zone-enabled regions**, use phrasing similar to this:

  **Example:**

    ```markdown
    - **Region support:** Zone-redundant \[service-name\] resources can be deployed into [any region that supports availability zones](/azure/reliability/regions-list).
    ```

- **If the service supports all but a small number of availability zone-enabled regions**, list the exceptions but not the full region list, like this:

  **Example:**

    ```markdown
    - **Region support:** Zone-redundant \[service-name\] resources can be deployed into any region that supports availability zones, except the following regions:

      - *XYZ*
      - *ABC*
    ```

- **If the service supports a subset of availability zone-enabled regions**, then you must:

    1. Create an include file in your repository that contains the list of supported regions. This means you can reuse the list in your own documentation, and you can modify the list without going through the Reliability Hub team.

    1. Add your region information into the file, for example:

        ```markdown
        | Americas         | Europe               | Middle East   | Africa             | Asia Pacific   |
        |------------------|----------------------|---------------|--------------------|----------------|
        | Brazil South     | France Central       | Qatar Central |                    | Australia East |
        | Canada Central   | Germany West Central |               |                    | Central India  |
        | Central US       | North Europe         |               |                    | China North 3  |
        | East US          | Sweden Central       |               |                    | East Asia      |
        | East US 2        | UK South             |               |                    | Japan East     |
        | South Central US | West Europe          |               |                    | Southeast Asia |
        | West US 2        |                      |               |                    |                |
        | West US 3        |                      |               |                    |                |
        ```

    1. In this section, include that file within the **Region support** bullet.

        **Example:**

          ```markdown
          - **Region support:** Zone-redundant \[service-name\] resources can be deployed into the following regions:

            [!INCLUDE [Azure <[service-name\] availability zone region support](../service-name/includes/availability-zone-regions-include.md)]
          ```

**Example section:**

  ```markdown
  ### Requirements

  - **Region support:** Zone-redundant \[service-name\] resources can be deployed into [any region that supports availability zones](/azure/reliability/regions-list).

  - **SKU requirements:** You must use the Standard tier or Premium tier to enable zone redundancy.
  ```

#### Considerations

Describe limitations, unsupported scenarios, and important caveats related to availability zone support.

**Include:**

- Features that aren't replicated across zones
- Workflows or scenarios that don't support zone redundancy
- Temporary limitations during zone failures
- Configuration gotchas or unexpected behaviors

**Example:**

  ```markdown
  ### Considerations

  During an availability zone outage, your application continues to run and serve traffic. However, you might be unable to use feature X or Y until the availability zone recovers.
  ```

#### Instance distribution across zones

*Optional section*: Include this section when the service allows customers to specify the number of instances/replicas. It's especially important if there's unexpected behavior in how instances are distributed across zones.

**Include:**

- How the service distributes replicas across zones
- Situations where distribution might be uneven
- Recommendations for ensuring balanced distribution

**Example:**

  ```markdown
  ### Instance distribution across zones

  Azure AI Search attempts to place replicas across different availability zones. However, there are occasionally situations where all of the replicas of a search service might be placed into the same availability zone. This situation can happen when replicas are removed from your service, such as when you *scale in* by configuring your service to use fewer replicas. The reason is that replica removal currently doesn't cause the remaining replicas to be rebalanced across the availability zones.

  To reduce the likelihood of all of your replicas being placed into a single availability zone, you can manually trigger a scale-out operation immediately after a scale-in operation. For example, suppose your search service has ten replicas and you want to scale in to seven replicas. Instead of performing a single scale operation, you can temporarily scale to six instances, then immediately scale to seven instances, to trigger zone rebalancing.
  ```

#### Cost

Explain the billing implications of enabling availability zone support.

**Include:**

- Whether there's an additional charge for zone redundancy
- Whether customers need to deploy additional instances of the service
- Links to Azure pricing documentation (never specify exact prices, or even percentages or other relative differences)

**Example:**

  ```markdown
  ### Cost

  When you enable zone redundancy, you're charged at a different rate. For more information, see [Service pricing](https://azure.microsoft.com/pricing/details/service-name).
  ```

#### Configure availability zone support

Provide links to documentation that shows customers how to work with availability zone support. Do not include step-by-step procedures here; consider creating a separate how-to article and linking to it from the reliability guide.

- **If the service provides automatic availability zone support**, and the customer has no control or ability to opt in or out, use wording like this:

  ```markdown
  ### Configure availability zone support

  When you create a \[service-name\] resource in a supported region, it's automatically zone-redundant. For more information about creating a new \[service-name\] resource, see [Create a new \[service-name\] resource that uses availability zones](link).
  ```

- **If the service requires the customer configure availability zone support in some way,** provide information about what possibilities the customer has to configure and configure the resource.

  DO NOT provide embedded detailed how-to guidance. Provide links to documents that show how to create a resource or instance with availability zone enabled. Ideally, the documents should contain examples using the Azure portal, Azure CLI, Azure PowerShell, and Bicep.

  If your service does NOT support enabling availability zone support after deployment, add an explicit statement to indicate that.

  **Example:**

  ```markdown
  ### Configure availability zone support

  - **Create a new zone-redundant \[service-name\] resource.** For more information, see [Create a new \[service-name\] resource that uses availability zones](link).

  - **Enable or disable zone redundancy on an existing \[service-name\] resource.** Zone redundancy can be configured only when a new \[service-name\] resource is created. If you have an existing \[service-name\] resource that isn't zone-redundant, replace it with a new zone-redundant \[service-name\] resource. You can't convert an existing \[service-name\] resource to use availability zones.

  - **Verify availability zone configuration for \[service-name\] resources.** For more information, see [Verify zone redundancy configuration for \[service-name\]](link).
  ```

#### Capacity planning and management

*Optional section*: Include this section if there's a risk that a zone failover can cause resource in the healthy zones to become overloaded, which could then cause problems for the customer's workload.

**Include:**
- How zone failures affect capacity in surviving zones
- Whether customers can mitigate risks through overprovisioning
- Recommendations for capacity planning, if applicable

If your service recommends overprovisioning, link to [Manage capacity by over-provisioning](/azure/reliability/concept-redundancy-replication-backup#manage-capacity-with-over-provisioning).

**Example:**

  ```markdown
  ### Capacity planning and management

  To prepare for availability zone failure, consider *overprovisioning* the number of replicas. Overprovisioning allows your \[service-name\] resource to tolerate some capacity loss and continue to function without degraded performance. Adding replicas during an outage is challenging, so overprovisioning helps ensure that your search service can handle normal request volumes, even with reduced capacity. For more information, see [Manage capacity by overprovisioning](/azure/reliability/concept-redundancy-replication-backup#manage-capacity-with-over-provisioning).
  ```

#### Behavior when all zones are healthy

Include this section to explain normal operations when all availability zones are functioning properly.

**Include two bullets:**

- **Traffic routing between zones**: Explain how traffic is distributed across zones during normal operations.

  - *For zone-redundant services*, traffic routing typically services fall into one of these models:
    - *Active/active*: Requests are automatically spread across instances in every availability zone
    - *Active/passive*: Requests go to a primary instance with standby instances in other zones

     **Example:**

      ```markdown
      - **Traffic routing between zones:** When you configure zone redundancy on \[service-name\], requests are automatically spread across the instances in each availability zone. A request might go to any instance in any availability zone.
      ```

  - *For zonal services*, clarify that customers are responsible for configuring their solution to route requests between the availability zones.

    **Example:**

      ```markdown
      - **Traffic routing between zones:** When you deploy multiple X resources in different availability zones, you need to decide how to route traffic between those resources. Commonly, you use a zone-redundant Azure Load Balancer to send traffic to resources in each zone.
      ```

- **Data replication between zones**:

  - *For zone-redundant services where the service replicates data across zones*, explain:
    - Replication method: synchronous, asynchronous, or hybrid. Most zone-redundant Azure services replicate data synchronously across zones.
    - How replication works during normal operations (not during failures)

    > [!IMPORTANT]
    > The data replication approach across zones is usually different from the approach used across regions.

    **Example:**

      ```markdown
      - **Data replication between zones:** When a client makes a change to any data in your  \[service-name\] resource, that change is applied to all instances in all zones simultaneously. This approach is referred to as synchronous replication. Synchronous replication ensures a high level of data consistency, which reduces the likelihood of data loss during a zone failure. Availability zones are located relatively close together, which means there's minimal effect on latency or throughput
      ```

    Some services replicate their data asynchronously, where changes are applied in a single zone and then propagated after some time to the other zones. Use wording similar to this to explain this approach and its tradeoffs.

    **Example:**

      ```markdown
      - **Data replication between zones:** When a client makes a changes to any data in your \[service-name\] resource, that change is applied to the primary zone. At that point, the write is considered to be complete. At some point later in time, the X resource in the secondary zone is automatically updated with the change. This approach is referred to as asynchronous replication. Asynchronous replication ensures high performance and throughput. However, any data that hasn't been replicated between availability zones could be lost if the primary zone experiences a failure.
      ```

  - *For stateless services*, clarify that no data is synchronized.

    **Example:**

      ```markdown
      - **Data replication between zones:** Because \[service-name\] doesn't store state, there's no data to replicate between zones.
      ```

> [!NOTE]
> Your service might behave differently to the examples provided above, so adjust or rewrite as much as you need.
>
> The accuracy and clarity of this information is critical to our customers, so please make sure you understand and explain the replication process thoroughly.

#### Behavior during a zone failure

Explain what happens when an availability zone fails. Be precise and clear, as customers rely on this information for their business continuity planning.

**Include the following bullets:**

- **Detection and response**: Explain who detects the zone failure and is responsible for responding, such as by initiating failover.

  - *For zone-redundant resources:* Microsoft always manages detection and failover, so use wording like this:

    **Example:**

      ```markdown
      - **Detection and response:** The \[service-name\] platform is responsible for detecting a failure in an availability zone. You don't need to do anything to initiate a zone failover.
      ```

  - *For zonal resources:* Specify whether it's customer-managed or Microsoft-managed, like this:

    **Example:**

      ```markdown
      - **Detection and response:** You need to detect the loss of an availability zone and respond appropriately. For example, you might initiate a failover to a secondary instance that you have precreated in another availability zone.
      ```

- **Notification**: Explain how customers can detect zone failures.

  All services should publish zone outage information into Azure Service Health. Some services also publish zone outage information into Azure Resource Health. Your service might also have other metrics or telemetry that customers should use to monitor the zone health.

  We have standard include files you can use for common scenarios:

  - If the service supports **both** Azure Resource Health **and** Azure Service Health:

    ```markdown
    [!INCLUDE [Availability zone down notification (Service Health and Resource Health)](./includes/reliability-availability-zone-down-notification-service-resource-include.md)]
    ```

  - If the service supports **only** Azure Service Health:

    ```markdown
    [!INCLUDE [Availability zone down notification (Service Health only)](./includes/reliability-availability-zone-down-notification-service-include.md)]
    ```

- **Active requests**: Explain what happens to any active (inflight) requests during a zone outage. For zone-redundant services, clarify whether work is automatically resumed on another node.

    **Example:**

    ```markdown
    - **Active requests**: When an availability zone is unavailable, any requests in progress that are connected to a replica in the faulty availability zone are terminated and need to be retried.
    ```

- **Expected data loss:** Explain if the customer should expect any data loss during a zone failover.

  For zone-redundant resources with synchronous replication, usually there is no data loss:

    **Example:**

    ```markdown
    - **Expected data loss:** A zone failure isn't expected to cause any data loss.
    ```

  For zonal resources, the data loss depends on the health of the zone:

    **Example:**

    ```markdown
    - **Expected data loss:** Data stored is unavailable until the zone recovers.
    ```

  For stateless services, there's no data to be lost:

    **Example:**

    ```markdown
    - **Expected data loss:** Zone failures aren't expected to cause data loss because \[service-name\] is a stateless service.
    ```

- **Expected downtime:** Explain any expected downtime, such as how long it takes for failover operations to complete.

  For most zone-redundant resources we typically suggest setting the expectation that a few seconds of downtime are normal during a zone failure, and that customers who design applications to be resilient to transient faults should be able to withstand these temporary blips:

    **Example:**

    ```markdown
    - **Expected downtime:** During zone outages, connections might experience brief interruptions that typically last a few seconds as traffic is redistributed. Ensure that your applications are prepared by following [transient fault handling guidance](#resilience-to-transient-faults).
    ```

  For zonal resources, use wording like this:

    **Example:**

    ```markdown
    - **Expected downtime:** When a zone is unavailable, your gateway is unavailable until the availability zone recovers.
    ```

- **Traffic rerouting**

  For zone-redundant services, explain how the platform recovers, including how traffic is rerouted to the surviving zones.

    **Example:**

    ```markdown
    - **Traffic rerouting:** When a zone is unavailable, \[service-name\] detects the loss of the zone and creates new instances in another availability zone. Then, any new requests are automatically spread across all active instances.
    ```

  For zonal services, explain that customers are responsible for rerouting traffic after a zone is lost.

    **Example:**

    ```markdown
    - **Traffic rerouting:** When a zone is unavailable, your \[service-name\] resource is unavailable. If you have a secondary \[service-name\] resource in another availability zone, you're responsible for rerouting traffic to that secondary \[service-name\] resource.
    ```

#### Zone recovery

Explain the zone recovery process after an availability zone comes back online.

**Include:**

- Who initiates recovery (customer or Microsoft)
  - For zone-redundant resources, recovery procedures are typically Microsoft-initiated
  - For zonal resources, customers might have to initiate recovery processes
- How the service restores normal operations
- Any data synchronization considerations (if applicable). Most services don't experience data inconsistencies between zones, but if your service does, explain what customers should do to resolve synchronization issues.

**Example:**

  ```markdown
  ### Zone recovery

  When the availability zone recovers, \[service-name\] automatically restores instances in the availability zone, removes any temporary instances created in the other availability zones, and reroutes traffic between your instances as normal.
  ```

#### Test for zone failures

Explain how customers can test their resilience to zone failures.

- *For zone-redundant services:* Most zone-redundant services are fully managed, so customers don't need to test failover. Use wording like this:

    **Example:**

    ```markdown
    #### Test for zone failures

    The Azure \[service-name\] platform manages traffic routing, failover, and zone recovery for zone-redundant resources. You don't need to initiate anything. Because this feature is fully managed, you don't need to validate availability zone failure processes.
    ```

- *For zonal services:* Explain if customers can simulate zone failures (for example, using Azure Chaos Studio). Link to specific fault injection documentation where appropriate. Only do so when there is a fault specific to the service.

    **Example:**

    ```markdown
    #### Test for zone failures

    You can simulate a zone failure by using Azure Chaos Studio. Inject the [specific fault name] to simulate the loss of an availability zone. Regularly test your responses to zone failures so that you can be ready for unexpected availability zone outages.
    ```

### Resilience to region-wide failures

This section describes any *native* multi-region capabilities the service might have, such as Microsoft-managed geo-replication and failover. It also contains a subsection that describes approaches or architectural patterns for creating custom multi-region solutions, which are referred to as *custom multi-region solutions for resiliency*.

**Organizing multi-region features:** The organization of this section is important for consistency and tooling support.

- *If the service offers any multi-region capabilities that are relevant for reliability*, such as cross-region replication, create an H3 with the name of that capability. Provide the complete set of H4 subsections (Requirements, Considerations, Cost, Configure multi-region support, etc.) under the H3 heading.
- *If the service offers multiple distinct multi-region capabilities*, such as geo-replication and cross-region read replicas, create an H3 heading for each solution and add the H4 subsections into each.
- *For all service*, then add an H3 called *Custom multi-region solutions for resiliency*. This section doesn't need to follow the subsection structure.

**Introductory paragraph:**

- *Single-region services:* Most Azure services have no native multi-region support at all. Customers can manually deploy separate resources into multiple regions, but they would have to handle replication, traffic distribution, failover, etc., which is described in the *custom multi-region solutions for resiliency* section.

  For single-region services, introduce this H2 section with something like this:

  **Example:**

  ```markdown
  ## Resilience to region-wide failures

  \[service-name\] is a single-region service. If the region becomes unavailable, your \[service-name\] resource is also unavailable. However, you can deploy separate resources into multiple regions and it is your responsibility to manage replication, traffic distribution, failover, etc. For more information on how to deploy across multiple regions, see [Custom multi-region solutions for resiliency](#custom-multi-region-solutions-for-resiliency).
  ```

- *Single-region services that provide failover to a paired region:* These services are deployed into a single region (the *home* region). However, when they are deployed into a home region that has a paired region, Microsoft replicates configuration and data to the paired region. Services might only replicate certain types of data.

  When deployed into a nonpaired region, Microsoft doesn’t replicate anything for the resource across regions, and there’s no built-in cross-region failover.

  For single-region services with paired region failover, consider using different wording depending on whether failover is fully Microsoft-controlled or if the customer can control it to some degree:

    - **Fully Microsoft controlled**: Microsoft is in total control of data replication between regions and it's enabled by default. Customers typically have no ability to opt in or out, however, there might be very limited situations where a customer can opt out due to data residency concerns.

      In the event of a catastrophic region failure in the primary region, Microsoft – at our absolute discretion – might elect to fail over all customers who use the service in that region. Introduce this section with something like:

      **Example:**

        ```markdown
        ## Resilience to region-wide failures

        \[service-name\] is a single-region service. If the region becomes unavailable, your \[service-name\] resource is also unavailable.

        > \[!IMPORTANT]
        > If your resources are in a region that's paired, you can choose to let Microsoft replicate configuration and data to the paired region. In the event of a region outage, Microsoft may perform a failover to the paired region. However, Microsoft is unlikely to initiate failover except after a significant delay and is done on a best-effort basis. Failover of \[service-name\] resources might happen at a different time to any failover of other Azure services. This process is a default option and requires no intervention from you.
        >
        > If the default replication and failover behavior doesn't meet your needs, you can use [custom multi-region solutions for resiliency] to plan for and initiate your failovers.
        ```

      Notice that we include some of the information in an IMPORTANT box, because Microsoft-managed failover is often misunderstood and customers make incorrect assumptions based on how they think the capability should work.

    - **Some degree of customer control**: A customer can decide whether to opt into replication of data to the paired region, such as by enabling a feature explicitly. A customer might be able to trigger a failover independently of Microsoft, although Microsoft might also retain the ability to trigger a failover if we want to. Introduce this section with something like:

      **Example:**

        ```markdown
        ## Resilience to region-wide failures

        \[service-name\] is a single-region service. If the region becomes unavailable, your \[service-name\] resource is also unavailable.

        However, if your resources are in *paired region*, you can:

        - **Data replication**. Choose to let Microsoft replicate configuration and data to the paired region. This process is a default option and requires no intervention from you.
        - **Failover**. Choose to let Microsoft monitor for and perform failover. Microsoft is unlikely to initiate failover except after a significant delay and is done on a best-effort basis. Failover of \[service-name\] resources might happen at a different time to any failover of other Azure services

        - **Manual failover**. Perform manual failover to the paired region yourself, whether the region is experiencing downtime or not. You can use this approach to perform planned failovers.

          If resources are in a *nonpaired region*, Microsoft doesn’t replicate configuration and data across regions, and there’s no built-in cross-region failover. However, you can deploy separate resources into  multiple regions and it is your responsibility to manage replication, traffic distribution, failover, etc.

          If you use a nonpaired region, or the default replication and failover behavior doesn't meet your needs, you can use [custom multi-region solutions for resiliency](#custom-multi-region-solutions-for-resiliency) to plan for and initiate regional failover.
        ```

- *Multi-region support*: These services are deployed to a single region (the *home region*), which is where the configuration and metadata of the service is located. However, a customer can explicitly replicate some parts of the service or its data into an arbitrary number of other regions, regardless of the region's pairing status.

    If there's a region outage in the home region, the parts of the service that are geo-replicated should continue to work. However, some other aspects of the service might have problems. For example, a customer might not be able to reconfigure the resource because the metadata can't be updated in the faulty region. Different services handle this differently.

    Introduce the capabilities with a brief explanation of what they do and their scope.

    **Example:**

    ```markdown
    ## Resilience to region-wide failures

    \[service-name\] can be configured to use multiple Azure regions. When you configure multi-region support, you choose the primary region, and \[service-name\] automatically replicates changes in your data to each selected secondary region.
    ```

The H4 sections below mirror the sections for availability zones, but have a slightly different focus for multi-region capabilities.

#### Requirements

List any requirements that must be met to use multiple regions with this service. Most commonly, specific SKUs are required. If multiple regions are supported in all SKUs, or if the service has only one default SKU, mention this. Also mention any other requirements that must be met.

**Example:**

  ```markdown
  - You can select any Azure region for your secondary instances.
  - You must use the Premium tier to enable multi-region support.
  ```

#### Considerations

Describe any workflows or scenarios that aren't supported, as well as any gotchas. For example, some services replicate only parts of the solution across regions.

Include information about any expected downtime or effects if you enable multi-region support after deployment. Provide links to any relevant information.

**Example:**

  ```markdown
  When you enable multi-region support, component Z is replicated across regions, but other components aren't replicated. After a region failover, your resource continues to work, but feature A might be unavailable until the region recovers and full service is restored.
  ```

#### Cost

Give an idea of what this does to the customer's billing meters. For example, is there an additional charge for enabling multi-region support? Do they need to deploy additional instances of the service in each region?

Don't specify prices. Link to the Azure pricing information if needed.

**Example:**

  ```markdown
  When you enable multi-region support, you're billed for each region that you select. For more information, see [Service pricing information].
  ```

#### Configure multi-region support

In this section, link to deployment guidance. If you don't have the required document, you'll need to create one.

DO NOT provide detailed how-to guidance in this article.

Provide links to documents that show how to create a resource or instance with multi-region support. Ideally, the documents should contain examples using the Azure portal, Azure CLI, Azure PowerShell, and Bicep.

**Example:**

  ```markdown
  To deploy a new multi-region \[service-name\] resource, see [Create an \[service-name\] resource with multi-region support].

  To enable multi-region support for an existing \[service-name\] resource, see [Enable multi-region support in an \[service-name\] resource].
  ```

If your service does NOT support enabling multi-region support after deployment, add an explicit statement to indicate that.

If your service supports disabling multi-region support, provide links to the relevant how-to guides for that scenario.

#### Capacity planning and management

*Optional section.* In some services, a region failover can cause instances in the surviving regions to become overloaded with requests. If that's a risk for your service's customers, explain that here, and whether they can mitigate that risk by overprovisioning capacity.

### Behavior when all regions are healthy

Add information about normal operations. Break the content down into two bullets: **Traffic routing between regions** and **data replication between regions**.

- **Traffic routing between regions**. Explains how work is divided up between instances in multiple regions, during regular day-to-day operations - NOT during a region failure.

    Commonly, services use one of these traffic routing approaches:
    - *Active/active.* Requests are spread across instances in every region. Services might use Traffic Manager or Azure Front Door behind the scenes, and you can decide whether to disclose that fact.
    - *Active/passive.* Requests always goes to the primary region.

    **Example:**

    ```markdown
    - **Traffic routing between regions:** When you configure multi-region support, all requests are routed to an instance in the primary region. The secondary regions are used only in the event of a failover.
    ```

- **Data replication between regions** is only required for services that perform data replication across regions.

    Most Azure services replicate the data across regions asynchronously, where changes are applied in a single region and then propagated after some time to the other regions. Use wording similar to this to explain this approach and its tradeoffs.

    **Example:**

    ```markdown
    - **Data replication between regions**: When a client changes any data in your \[service-name\] resource, that change is applied to the primary region. At that point, the write is considered to be complete. Later, the X resource in the secondary region is automatically updated with the change. This approach is called *asynchronous replication.* Asynchronous replication ensures high performance and throughput. However, any data that wasn't replicated between regions could be lost if the primary region experiences a failure.
    ```

    Alternatively, some services replicate their data synchronously which means that changes are applied to multiple (or all) regions simultaneously, and the change isn't considered to be completed until multiple/all regions have acknowledged the change. Use wording similar to the following to explain this approach and its tradeoffs:

    **Example:**

    ```markdown
    - **Data replication between regions**: When a client changes any data in your \[service-name\] resource, that change is applied to all instances in all regions simultaneously. This approach is called *synchronous replication.* Synchronous replication ensures a high level of data consistency, which reduces the likelihood of data loss during a region failure. However, because all changes must be replicated across regions that might be geographically distant, you might experience lower throughput or performance.
    ```

>[!NOTE]
> You may be asked to provide an image here if it helps to explain the traffic routing process.

Your service might behave differently to the examples provided above, so adjust or rewrite as much as you need. The accuracy and clarity of this information is critical to our customers, so please make sure you understand and explain the routing and replication processes thoroughly.

#### Behavior during a region failure

Explain what happens when a region is down. Be precise and clear. Avoid ambiguity in this section, because customers depend on it for their planning purposes.  Divide your content into the following bullets.

- **Detection and response.** Explain who is responsible for detecting that a region is down and for responding, such as by initiating a region failover. Whether your service has customer-managed failover or the service manages it itself, describe it here.

  If your multi-region support depends on another service, commonly Azure Storage, detecting and failing over, explicitly state that, and link to the relevant Reliability guide to understand the conditions under which that happens. Be careful with talking about GRS because that doesn't apply in nonpaired regions, so explain how things work in that case.

  *For customer initiated detection:*

    **Example:**

    ```markdown
    - **Detection and response:** \[service-name\] is responsible for detecting a failure in a region and automatically failing over to the secondary region.
    ```

  *For service initiated detection:*

    **Example:**

    ```markdown
    - **Detection and response:** \[service-name\] is responsible for detecting a failure in a region and automatically failing over to the secondary region.
    ```

  *For detection that depends on another service:*

    **Example:**

    ```markdown
    - **Detection and response:** In regions that have pairs, \<service name> depends on Azure Storage geo-redundant storage for data replication to the secondary region. Azure Storage detects and initiates a region failover, but it does so only in the event of a catastrophic region loss. This action might be delayed significantly, and during that time your resource might be unavailable. For more information, see [Link to more info].
    ```

- **Notification** Explain if there's a way for a customer to find out when a region has been lost. Are there logs? Is there a way to set up an alert?

  Similar to zone outage notifications, we have standard include files you can use for common scenarios:

  - If the service supports **both** Azure Resource Health **and** Azure Service Health:

    ```markdown
    [!INCLUDE [Region down notification (Service Health and Resource Health)](./includes/reliability-region-down-notification-service-resource-include.md)]
    ```

  - If the service supports **only** Azure Service Health:

    ```markdown
    [!INCLUDE [Region down notification (Service Health only)](./includes/reliability-region-down-notification-service-include.md)]
    ```

- **Active requests** Explain what happens to any active (inflight) requests:

    **Example:**

  ```markdown
  - **Active requests:** Any active requests are dropped and should be retried by the client.
  ```

- **Expected data loss:** Explain if the customer should expect any data loss during a region failover. Data loss is common during a region failover, so it's important to be clear here.

    **Example:**

    ```markdown
    - **Expected data loss:** You might lose some data during a region failure if that data isn't yet synchronized to another region.
    ```

- **Expected downtime:** Explain any expected downtime, such as during a failover operation.

  **Example:**

  ```markdown
  - **Expected downtime:**Your \[service-name\] resource might be unavailable for approximately 2 to 5 minutes during the region failover process.
  ```

- **Traffic rerouting** Explain how the platform recovers, including how traffic is rerouted to the surviving region. If appropriate, explain how customers should reroute traffic after a region is lost.

  **Example:**

  ```markdown
  - **Expected downtime:** When a region failover occurs, \[service-name\] updates DNS records to point to the secondary region. All subsequent requests are sent to the secondary region.
  ```

#### Region recovery

Explain who initiates a region recovery. Is it customer-initiated or Microsoft-initiated? What does region recovery involve?

If there is any possibility of data synchronization issues or inconsistencies during region recovery, explain that here, as well as what customers can/should do to resolve the situation.

**Example:**

```markdown
When the primary region recovers, [service-name] automatically restores instances in the region, removes any temporary instances created in the other regions, and reroutes traffic between your instances as normal.
```

#### Test for region failures

Can you trigger a fault to simulate a region failure, such as by using Azure Chaos Studio? If so, link to the specific fault types that simulate the appropriate failure.

**Example:**

```markdown
You can simulate a region failure by using Azure Chaos Studio. Inject the XXX fault to simulate the loss of an entire region. Regularly test your responses to region failures so that you can be ready for unexpected region outages.
```

For Microsoft-managed multi-region services, is there a way for the customer to test a region failover? If that's not possible, use wording like this:

**Example:**

```markdown
The Azure \[service-name\] platform manages traffic routing, failover, and region recovery for multi-region X resources. You don't need to initiate anything. Because this feature is fully managed, you don't need to validate region failure processes.
```

#### Custom multi-region solutions for resiliency

*Optional but strongly recommended section:* Describes how a customer might approach deploying independent instances of the service into different regions and coordinating replication, failover, etc. It needs to be clear that the customer is the one who is managing the whole process, and that there are no built-in multi-region features being used.

Consider adding this section if any of these applies to the service:

- The service does NOT have built-in multi-region support.
- The service provides built-in multi-region support, but:
  - That support relies on paired regions, which means customers in nonpaired regions can't use it.
  - The service provides Microsoft-managed replication and failover, but Microsoft decides when to fail over, which means customers with strict failover policies or stringent reliability requirements might not be well-served by the capability.
  - The built-in multi-region support only works for subset of the functionality of the service, which means customers needing regional resiliency for the other capabilities might need to use another solution as well.

You can provide multiple approaches if required.

Call out any dependency on paired regions. At least one of the approaches should work in nonpaired regions.

If you have any approaches documented in the Azure Architecture Center, summarize them and link to them from here.

**Example:**

  ```markdown
  If you need to use \[service-name\] in multiple regions, you need to deploy separate resources in each region. If you create an identical deployment in a secondary Azure region using a multi-region geography architecture, your application becomes less susceptible to a single-region disaster. When you follow this approach, you need to configure load balancing and failover policies. You also need to replicate your data across the regions so that you can recover your last application state.

  For example approaches that illustrates this architecture, see:

  - [Multi-region load balancing with Traffic Manager, Azure Firewall, and Application Gateway](/azure/architecture/high-availability/reference-architecture-traffic-manager-application-gateway)
  - [Highly available multi-region web application](/azure/architecture/web-apps/app-service/architectures/multi-region)
  - [Deploy Azure Spring Apps to multiple regions](/azure/architecture/web-apps/spring-apps/architectures/spring-apps-multi-region)
  ```

### Backup and restore

Required only if the service provides backup capabilities or similar functionality (like exporting data).

Describe any backup features the service provides. Clearly explain whether they are fully managed by Microsoft, or if customers have any control over backups. Explain where backups are stored and how they can be recovered. Note whether the backups are only accessible within the region or if they're accessible across regions, such as after a region failure.

We recommend adding the following include file, which adds an explanation about the role of backups:

```markdown
[!INCLUDE [Backups description](includes/reliability-backups-include.md)]
```

### Resilience to service maintenance

Describe how the service maintains reliability during maintenance operations. If the service has no special requirements for customers to maintain reliability during service maintenance, you can omit the section.

If the customer should take action to avoid issues during maintenance, briefly explain how they can do that:

  **Example:**

  ```markdown
  ## Resilience to service maintenance

  [service-name] performs regular service upgrades and other maintenance tasks. To ensure that sufficient capacity is available even during upgrades, you should configure your *resource type* to have additional capacity.
  ```

If the customer can control any aspect of when or how maintenance occurs, briefly explain how they can do that. Provide links to any relevant documentation.

  **Example:**

  ```markdown
  ## Resilience to service maintenance

  To reduce service disruptions during critical time periods, [service-name] provides controls so that you can specify planned maintenance times.
  ```

### Service-level agreement

This section should always start with the following include file:

  ```markdown
  ## Service-level agreement

  [!INCLUDE [Service-level agreement](includes/reliability-service-level-agreement-include.md)]
  ```

Then the section should call out:
- Whether there are different guarantees/SLAs depending on customer configurations or decisions, such as SLAs that apply to particular SKUs, configurations, or regions
- Key requirements that must be met for the SLA to take effect

Do not repeat the SLA, or provide any exact wording or numbers. Instead, aim to provide a general overview of how a customer should interpret the SLA for a service, because they often are quite specific about what needs to be done for an SLA to apply.

### Related content

Add a *Related content* section to direct the reader to a relevant task to accomplish, or to a related topic to progress to, such as relevant product documentation and [Reliability in Azure](/azure/reliability).
