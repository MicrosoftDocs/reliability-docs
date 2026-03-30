---
title: How to Read a Service-Level Agreement (SLA)
description: Learn how to interpret service-level agreements (SLAs), understand what they guarantee and exclude, and use them to make better architectural decisions for your workloads.
author: glynnniall
ms.service: azure
ms.subservice: azure-reliability
ms.topic: concept-article
ms.date: 03/31/2026
ms.author: glynnniall
ms.custom: subject-reliability
ai.usage: ai-assisted
show_latex: true
#customer intent: As a cloud architect or reliability engineer, I want to understand how to read and interpret SLAs so that I can use them as engineering signals when designing resilient workloads.
---

# How to read a service-level agreement (SLA)

A service-level agreement (SLA) is one of the most commonly referenced documents in cloud architecture, but it's also one of the most frequently misunderstood. Many engineers treat an SLA as a simple availability guarantee, but it's actually a contractual document with precise definitions, conditions, and exclusions that shape its meaning. If you don't read an SLA carefully, you can make architectural decisions based on unsupported assumptions.

This article explains how to interpret SLAs methodically so that you can use them as engineering inputs when you design resilient workloads. This article describes how to:

- Explain what an SLA actually guarantees. 
- Identify how technology service providers typically define and measure availability. 
- Recognize conditions and exclusions that affect coverage. 
- Use SLAs to inform architectural decisions rather than as substitutes for resilience.

Most SLAs define commitments around uptime or availability, but service providers can also define SLAs around other requirements such as performance, recovery time, or data durability. 

> [!IMPORTANT]
> This article explains how to *interpret* SLAs. It doesn't provide legal advice or guidance about how to dispute service providers or claim service credits. It isn't specific to Microsoft SLAs. The examples and principles apply broadly to help you read any service provider's SLA critically.

## How SLAs relate to SLOs and reliability targets

Before you read an SLA, understand where SLAs fit alongside other reliability concepts:

- An **SLA** is a contractual commitment between a service provider and a customer, with defined consequences for unmet targets.

- A **service-level objective (SLO)** is an internal reliability target that you set for your own workload.

An SLA tells you the *minimum* that the service provider commits to. An SLO reflects the reliability that your *users* actually need. Use SLAs to inform your SLO, but you should rarely mirror them directly. Account for your own code, your dependencies, your operational processes, and your users' tolerance for downtime. Sometimes, you need stricter SLOs than your service providers' SLAs combined. Other times, your workload can tolerate problems that exceed what your service provider financially commits to.

For more information about how to define SLOs and reliability targets, see [Recommendations for defining reliability targets](/azure/well-architected/reliability/metrics).

## Why you should read SLAs

SLAs are engineering signals, not strict availability guarantees. They describe the conditions under which a service provider commits to a defined level of service and the financial consequences if the service provider doesn't meet those commitments. SLAs don't guarantee that your application is reliable, that your users avoid downtime, or that the service provider prevents all failures.

Assumptions about SLAs lead to poor architectural decisions. When architects treat an SLA's uptime percentage as a direct indicator of service reliability, they often overlook critical details, such as what qualifies as downtime, what conditions must be met, and what scenarios are excluded. The resulting workload design might not tolerate the failure modes that the SLA implies.

SLAs, when read carefully, also provide useful signals for designing resilient systems:

- **Transient failures are expected:** Service providers expect short-lived failures to occur. SLAs make this expectation explicit through downtime definitions and retry requirements. Design your architecture to handle transient failures as a normal condition, not an exceptional one.

- **Not all services are equivalent:** Services vary in their SLA scopes, conditions, and exclusions.

- **Uptime percentages don't apply uniformly:** A service with a 99.99% SLA for its core operations might offer a lower SLA or no SLA at all for specific features that you depend on.

- **Use SLAs alongside limits, quotas, and product documentation:** SLAs alone don't address all reliability considerations. Also review service limits, throttling behaviors, known limitations, and operational guidance in the product documentation.

## What an SLA is and isn't

An SLA is a set of conditional commitments. It states that *if* a customer meets certain requirements, *then* the service provider commits to a defined level of service. *If* the service provider fails to meet that level, *then* the customer is entitled to a financial remedy, usually in the form of service credits. 

SLAs apply only when you meet specific preconditions, such as configuration requirements, deployment topologies, or operational behaviors. The SLA specifies exactly what counts as a failure. If a problem doesn't meet the documented definition of downtime, the SLA doesn't cover it.

Don't create a *composite SLA* that multiplies SLA percentages together to estimate the availability of a system that has multiple dependencies. This calculation can provide an approximate maximum limit, but it's unreliable in practice. It assumes that all services fail independently, which rarely happens. It also ignores the unique definitions, conditions, and exclusions in each SLA. Composite calculations create false confidence and can't replace proper SLO definition and failure mode analysis for your workload. For more information, see [Recommendations for defining reliability targets](/azure/well-architected/reliability/metrics).

## A practical model for reading SLAs

Don't read an SLA from start to finish. Instead, use a structured, five-pass approach that extracts the information that matters for your architecture.

### Pass 1: Definitions

Start with the definitions section. The definitions control how the SLA measures availability and what failures count.

- **Definitions matter as much as percentages:** A 99.99% SLA sounds impressive, but its value depends entirely on what the SLA counts as a failure and what units it measures. When the definition of downtime is narrow, the effective coverage is narrower than the percentage suggests.

- **Focus on key terms:** Terms like *valid request*, *downtime*, and *billable minutes* define what the service provider commits to. If an operation doesn't qualify as a valid request under the SLA, it doesn't affect the availability calculation.

    Identify what operations the SLA covers, what components it explicitly or implicitly excludes, and whether it applies per service instance or across a pool of instances.

> [!TIP]
> A 99.9% SLA doesn't guarantee the service is always available. It also doesn't mean that you should expect only 8.7 hours of downtime per year. The SLA's definition of downtime determines what gets measured. Narrow definitions mean that the service can experience problems that affect your users without violating the SLA.

### Pass 2: Measurement

Understand how the SLA calculates availability.

- **Time-based versus request-based measurement:** Time-based measurement typically tracks whether the service was available during each time interval. Request-based measurement tracks the ratio of successful responses to total valid requests. The same outage can produce different availability percentages depending on which method the SLA uses.

- **Retries and aggregation:** Some SLAs require you to retry failed requests for a specified period before they count as failures. Other SLAs aggregate availability over an entire billing period, so brief outages might not reduce the calculated availability enough to trigger a credit.

- **Aggregation scope:** Determine whether the SLA measures availability per individual service instance or across all instances in a scope, such as an account or subscription. When the SLA calculates availability across multiple instances, healthy instances can offset failed instances, which produces a higher reported availability than any single instance actually experienced.

> [!TIP]
> SLAs usually measure uptime over a billing period, not in real time. A service can have an outage that lasts several minutes and still meet its SLA for the month.

### Pass 3: Prerequisites, variations, and exclusions

Identify what must be true for the SLA to apply.

- **Required configurations and behaviors:** Many SLAs require specific deployment configurations or operational behaviors, such as implementing retry logic or staying within documented quotas. For example, a storage SLA might apply only when you use SSDs rather than HDDs because the service provider's uptime commitment depends on the performance characteristics of that storage type.

- **Configuration affects SLA terms:** Examine how your service configuration changes the SLA. Higher-priced tiers sometimes offer different SLA commitments, but the difference isn't always just a higher uptime percentage. Higher tiers might have different definitions, different measurement methods, or fewer exclusions. A premium tier SLA might define downtime more broadly (better coverage) rather than simply increase the uptime percentage. Don't rely on the percentage alone. Also read the definitions.

    Deployment models also affect SLA terms. For the same service, a single-instance deployment might have a lower SLA than a multiple-instance deployment that provides redundancy.

- **Environmental and operational exclusions:** SLAs commonly exclude downtime from planned maintenance, force majeure, customer-initiated actions, misconfiguration, exceeded quotas, malformed requests, and preview or unsupported features.

> [!TIP]
> A correct deployment doesn't guarantee SLA coverage. The SLA won't apply if you don't meet other conditions, such as implementing required retry logic or if an excluded event causes the failure. Check whether the features that you depend on are within the SLA's scope. SLAs often cover only specific operations, endpoints, or tiers. They might exclude preview features, certain APIs, and lower-tier versions.

### Pass 4: Design implications

To use the service most effectively, consider what the SLA reveals about how the service behaves and how you should design your solution around it.

- **Expected failure modes:** The SLA's definitions and conditions reveal which failures the service provider considers normal and expected. Your architecture must tolerate these failures.

- **What the service provider prioritizes:** How the service provider defines availability, measures it, and excludes certain scenarios reveals its core commitment and what it considers your responsibility.

- **How this should influence your design:** If the SLA excludes certain failure modes, decide whether to accept the risk, add redundancy, or use alternative services.

### Pass 5: How to claim service credits

The SLA defines service credits as the financial remedy, but service providers don't usually apply them automatically. They don't proactively monitor SLA violations or file claims for you. You must detect outages that breache the SLA, document them, and submit claims within the deadline. Use the following steps to prepare for and run that process.

#### Step 1: Know what to monitor

Before an incident occurs, identify what data you need to collect to substantiate a claim. Review the SLA's definitions and measurement method to determine:

- **Which metrics map to the SLA:** If the SLA measures request success rate, monitor HTTP response codes for the covered endpoints. If it measures uptime in minutes, monitor service availability at regular intervals.

- **Which instances are in scope:** SLAs often apply to specific tiers, versions, or deployment configurations. Monitor only the instances that qualify for SLA coverage.

- **What the provider considers a failure:** Some SLAs define downtime narrowly. For example, the SLA might require all requests to fail for one or more consecutive minutes, or it might require a documented outage on a public list. If necessary, configure your monitoring to capture the specific failure conditions that the SLA defines.

#### Step 2: Keep records during and after an incident

When an outage or degradation occurs, collect and retain evidence that aligns with the SLA's definitions:

- **Timestamps:** Record the start and end time of the incident, including the time zones. Note the duration of the impact and whether it was continuous or intermittent.

- **Error details:** Capture the specific error codes, error messages, or timeout behaviors that you observe. These details should correspond to the failure conditions that the SLA defines.

- **Request logs:** If the SLA requires retry attempts before it counts a failure, retain logs to show that you met that requirement. Include the number of retries, backoff intervals, and the outcome of each attempt.

- **Configuration evidence:** If the SLA requires a specific deployment configuration, such as a specific amount of redundancy or a premium tier, document how your instances met those prerequisites at the time of the incident.

- **Service provider communications:** Save any incident notifications, status page updates, or support correspondence from the service provider that acknowledges the problem.

#### Step 3: Determine whether the SLA was breached

Before you file a claim, calculate whether the incident meets the threshold for a credit:

1. Apply the SLA's availability formula by using your recorded data. Use the same units, such as minutes, requests, or another metric, that the SLA specifies.

1. Subtract any time that falls under documented exclusions, such as planned maintenance windows, excluded event types, or periods when your configuration didn't meet the SLA's prerequisites.

1. Compare the result against the SLA's committed percentage. If the calculated availability falls below the threshold, a credit might apply.

#### Step 4: File the claim within the required timeframe

Most SLAs impose a deadline to submit claims after an incident. If you miss the deadline, you forfeit the credit regardless of the severity of the outage.

- **Find the claim window:** Review the SLA document for the submission deadline. Common timeframes range from 30 to 60 days after the end of the billing month when the incident occurs, but each service provider sets its own deadline.

- **Submit through the correct channel:** Service providers typically require that you file claims through a support portal or a formal request process. Follow the documented procedure to avoid claim rejection.

- **Include supporting evidence:** Attach the records that you collected in Step 2, including timestamps, error logs, retry evidence, and configuration details. A well-documented claim is more likely to be processed without delays.

> [!NOTE]
> Service credits are typically a percentage of the monthly service fee for the affected service. If your monthly spend on a service is modest, a 10% credit might amount to a small sum, regardless of how severe the impact was on your business. **Credits don't cover lost revenue, customer attrition, or reputational damage.**

## A guided example: Decode a fictional SLA

To illustrate how to read an SLA in practice, this section uses a fictional service, *Pigeon Post as a Service (PPaaS)*. PPaaS is a hypothetical managed messaging service that delivers messages through trained carrier pigeons. An API accepts a message and sends it to a dispatcher that's responsible for releasing the pigeon and sending the message to the recipient.

:::image type="complex" border="false" source="media/concept-service-level-agreements/pigeon-posted-service.png" alt-text="Diagram showing the pigeon post service architecture. Messages are sent from a client to an API, which then connects to a dispatcher. The dispatcher sends a carrier pigeon to the recipient." lightbox="media/concept-service-level-agreements/pigeon-posted-service.png":::
Diagram showing the pigeon post service architecture. Messages are sent from a client to an API, which then connects to a dispatcher. The dispatcher sends a carrier pigeon to the recipient.
:::image-end:::

PPaaS is intentionally absurd, but the structure of its SLA reflects many real-world SLAs. This example illustrates the five-pass approach.

> [!NOTE]
> PPaaS is entirely fictional. Any resemblance to real services, avian or otherwise, is coincidental.

### The PPaaS SLA

The PPaaS SLA includes the following definitions:

- **Valid request:** A message dispatched through the PPaaS API that conforms to the documented message schema.

- **Downtime:** A period when all attempts to send a *valid request* to a provisioned PPaaS endpoint either fail with an error response or receive no response.

- **Total billable minutes:** The total number of minutes in the billing month during which at least one *valid request* submission operation was attempted.

- **Monthly uptime percentage**, calculated as:

    $$
    \text{Monthly uptime percentage} = \frac{\text{Total billable minutes} - \text{Downtime}}{\text{Total billable minutes}} \times 100
    $$

The PPaaS SLA includes the following service credits:

| Tier | Monthly uptime percentage | Service credit |
| --- | --- | --- |
| **Standard Coop** | Less than 99.9% | 10% |
| **Standard Coop** | Less than 99% | 25% |
| **Basic Coop** | Not applicable | Not applicable |

The PPaaS SLA includes the following conditions and exclusions:

- The SLA doesn't cover failures caused by severe weather.

- The SLA excludes up to 15 minutes per week of scheduled coop maintenance.

- You must implement retry logic with at least three retries and exponential backoff before a request counts as failed.

- The SLA doesn't apply to the Basic Coop tier.

## Read the PPaaS SLA step by step

The following passes apply the five-pass approach to the PPaaS SLA. Each step surfaces information that should shape your architecture.

### Pass 1: Understand the definitions

Start by examining what each defined term means. Here are a few particular callouts:

- **Valid request:** Only messages that conform to the documented schema and don't exceed the size limit, which you need to look up separately in the product documentation. If you send an oversized message and it fails, that failure isn't measured.

- **Total billable minutes:** Only minutes during which you actually attempted to send a message count. If you don't send messages for a portion of the month, those idle minutes don't factor into the calculation, even if the service had a problem during those times.

- **Dispatch time:** This metric measures only the time to queue a message, not the flight time, delivery confirmation, or return transit. If your pigeon takes three days to deliver a message, that's outside the SLA's scope.

### Pass 2: Understand how availability is measured

Key observations:

- **Retry requirements:** You must attempt retries with exponential backoff before a failure counts. If you don't retry, the failure isn't measured.

- **Aggregation across a billing period:** The SLA calculates availability over the entire month. A 10-minute outage in a month where your service otherwise runs continuously might still leave you above the SLA threshold.

### Pass 3: Identify conditions and exclusions

Several conditions limit the SLA's applicability:

- **Severe weather conditions:** Check your product documentation, or with your service provider, how they define severe weather conditions. This exclusion removes an entire category of failures that could frequently affect a pigeon-based service.

- **Scheduled coop maintenance:** The SLA excludes up to 15 minutes per week of planned maintenance. This maintenance window doesn't count against the SLA, even though the service is unavailable during that time. You might have no control over when that maintenance happens. Check your product documentation about when maintenance occurs and plan accordingly.

- **Tier-specific commitments:** The Basic Coop tier has no SLA at all. If you use the Basic tier, you have no contractual availability commitment.

### Pass 4: Extract architectural signals

The PPaaS SLA reveals several important signals for your architecture:

- **What PPaaS considers a failure:** Only complete, sustained dispatch failures that last at least one full minute count. Brief interruptions, slow dispatches, and individual request failures are expected to be handled by your workload.

- **What it doesn't guarantee:** The SLA only covers whether you can successfully submit a message to the PPaaS API and have it accepted for dispatch. It doesn't guarantee delivery time, delivery success, or message integrity once the pigeon is released. If your workload depends on confirmed delivery, that falls outside the SLA entirely.

- **Where architects should introduce resilience:** You should design for intermittent dispatch failures by implementing retry logic (which is required anyway). You should also have a fallback mechanism for situations where delivery guarantees matter, because the SLA doesn't cover actual delivery. Weather-related outages are excluded, so if your workload is sensitive to those outages, you need an alternative path. Scheduled maintenance is also excluded from the SLA, so your architecture must tolerate planned unavailability that occurs outside your control.

### Pass 5: Prepare to claim service credits

If PPaaS fails to meet its SLA, you need to be ready to substantiate a claim. The PPaaS SLA reveals several specific requirements:

- **Keep detailed logs:** Track dispatch success or failure at per-minute granularity, and retain logs that show each failed attempt included the required three retries with exponential backoff. Also retain independent weather data for your deployment region, because PPaaS could attribute downtime to severe weather conditions rather than a service issue.

- **File within the claim window:** The PPaaS SLA doesn't specify a claim deadline in this example, but real SLAs typically require claims within 30 to 60 days after the end of the billing month. Identify and add the deadline do your calendar so you don't forfeit a valid claim.

## Next steps

- Review the SLAs for the services you depend on. For Microsoft services, start with the [SLAs for online services](https://www.microsoft.com/licensing/docs/view/Service-Level-Agreements-SLA-for-Online-Services). Your workload might also depend on services from other service providers; apply the same five-pass approach to those SLAs.
- Compare SLA guarantees with your workload's reliability goals to identify gaps.
- Learn how to define SLOs that go beyond service provider SLAs. See [Recommendations for defining reliability targets](/azure/well-architected/reliability/metrics).
