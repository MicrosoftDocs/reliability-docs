# Copilot instructions for the Azure reliability documentation on Microsoft Learn

This repository contains the source data for the Azure reliability documentation articles published as official Microsoft documentation on Microsoft Learn. The data is stored mostly as Markdown files. These Markdown files get converted to HTML for presentation on Microsoft Learn. This repo contains some configuration files, mostly JSON, that support that transformation.

## Audience and how they use this data

The data in the repo helps professional cloud architects, site reliability engineers, and cloud engineers design and implement reliable workloads on Azure. It provides platform and service level information about Azure's reliability. Information in this content tends to be descriptive rather than prescriptive although there is some prescriptive content in specific places. Azure customers use the information in this repository to understand the reliability of the Azure services that they use how they configure those capabilities, and how they interact with the platform more broadly.

## Repository facts

- This data gets published at <https://learn.microsoft.com/azure/reliability>.
- This is not a repository for software development.
- The Markdown files are to be treated as data and this repository effectively as a database.
- The data in this repository is grounded in Microsoft Azure technology.
- This data must always lead to success of the person reading this.
- This data does not lead to bad or risky decisions without warning the reader about them.
- This data helps an architect avoid regret in their solution design.
- This data avoids risky decisions without proper warnings.
- This data is truthful.

## Repository structure

- This is the **private repository** (`-pr` suffix) for internal Microsoft authoring.
- A corresponding **public repository** exists at <https://github.com/MicrosoftDocs/reliability-docs>.
- The `main` branch is for development; the `live` branch reflects published content.

## Your behavior

- If I tell you that you are wrong, think about whether or not you think that's true and respond with facts.
- Avoid apologizing or making conciliatory statements.
- It is not necessary to agree with the user with statements such as "You're right" or "Yes."
- Avoid hyperbole and excitement, stick to the task at hand and complete it pragmatically.
- Never use content that would violate copyrights.
- When you use facts from existing content on Microsoft Learn, you'll link to those sources.
- You'll never invent Azure products or services.
- You'll never invent Microsoft products or services.

## How you'll write

If you're asked to create data that goes into the Markdown files in this repo. Use the following writing style constraints.

- Use clear language.
- Prefer simple sentences over those with dependent clauses.
- Be descriptive, not emotional.
- Avoid the passive voice. Use "you" as the subject where necessary.
- Avoid generalizations, marketing terms, or weasel words. Words you should generally avoid: seamless, seamlessly, fast, quick, quickly, easy, effortless, effortlessly, simple, simply, world-class, cutting-edge, cheap. You don't know the constraints of the reader or their situation, so you shouldn't make general statements like these. Instead present the facts/metrics/limits in a way that will help an architect make an informed decision.

## Folder and file structure

### Directory organization reality

- The root folder for articles is `articles/`.
- **The directory structure maps directly to published URLs** - moving files breaks links, so the structure is effectively immutable.
- Don't create new folders or move files without understanding the URL impact and redirect requirements.

### Types of content

There are several main types of content in this repository:

#### Reliability service guides

These are structured documents that describe the reliability features of specific Azure services.

- **Filename pattern:** `reliability-{service-name}.md` files in the `articles/` folder. For example, `reliability-storage-blob.md`.
- They follow the template defined in #file:./reliability-guide-template/reliability-template.md
- When drafting these files, they follow the instructions defined in #file:./reliability-guide-template/contributor-guide.md
- Note that some older reliability guides don't follow the current template, and don't meet the current quality bar. These are being progressively updated, but if you are asked to update or create a reliability guide, prefer the template-based guides and use the template for guidance on content and structure.

#### Platform reliability documentation

These are unstructured documents that describe reliability features that span across Azure services, such as availability zones, regions, and general platform capabilities.

- **Filename pattern:**
  - `availability-zones-*.md` for availability zones documentation.
  - `regions-*.md` for regions documentation.
  - `incident-response.md` for Azure incident response documentation.

Some articles provide reference material and lists:

- Some of them summarize and roll up content in other documentation:
  - `availability-zones-enable-zone-resiliency.md` - provides both guidance on enabling zone resiliency and a rollup of how each service supports availability zones, based on the individual reliability guides.
  - `availability-zones-service-support.md` - provides a rollup of availability zone support across Azure services, based on the individual reliability guides.
  - `regions-nonregional-services.md` - provides a rollup of Azure services that are not deployed into a single specific Azure region.
- Others provide unique content:
  - `availability-service-by-category.md` - provides a list of services based on their deployment category, which helps customers to understand which regions they are available in.
  - `regions-list.md` - provides a list of Azure regions and some of their core characteristics.

#### Conceptual reliability documentation

These are unstructured documents that describe reliability concepts, and are intended to provide a foundational understanding of reliability in the cloud.

- **Filename pattern:** `concept-*.md` files in the `articles/` folder.

#### Legacy content

The following content is considered legacy:

- Availability zone migration guides
  - **Filename pattern:** `migrate-*.md` files in the `articles/` folder.
  - These are being deprecated and should not be updated or added to. They will be removed.
- `regions-multi-region-nonpaired.md` - this is being deprecated and should not be updated. It will be removed.
- `overview-reliability-guidance.md` - this is a list of reliability guides and other reliability content for Azure services. We continue to update the links in the document, but plan to remove it in the future because the index.yml file provides a better user experience.
- Older reliability guides:
  - There are several reliability guides that don't conform to the current template or meet our updated quality bar.
  - We are progressively updating many of these guides, and retiring some older guides that no longer justify their maintenance cost. If you are asked to update or create a reliability guide, prefer the template-based guides and use the template for guidance on content and structure.

#### Overviews

## Multi-agent usage

- Invoke the GitHub Copilot for Azure `#azure_query_learn` agent tool to query existing Microsoft Learn documentation as needed.
- Invoke the Web Search for Copilot `#websearch` agent tool to query general knowledge from the Internet as needed.

## Sourcing policy

- Prioritize Microsoft Learn as the primary source of truth.
- Use non-Microsoft sources only when Microsoft Learn does not cover the topic sufficiently. Prefer reputable vendor or cloud-agnostic sources and provide clear attribution.

## Proactive edits (scope)

- Allowed without asking: copyedits that do not change meaning (grammar, spelling, concision), removal of weasel words, and minor structure cleanups (headings, lists) that preserve meaning.
- Not allowed without request: content rewrites and adding/removing sections

## Freshness updates

Data in this repository must be periodically updated to reflect modern approaches and modern technology, usually once a year. Data that receives a full freshness pass gets its `ms.date` metadata updated to reflect this. Do not proactively perform a full freshness pass; instead, when you detect content that appears outdated or divergent, leave files unchanged and output a message to the human-in-the-loop indicating that a freshness pass is recommended and why.

The following items must be addressed during a freshness update (when explicitly requested), no exceptions, and the person doing the update will need to self-attest to addressing these items:

- Links are not broken, they lead to the article they are supposed to lead to without redirection.
- Update the content to reflect the current state of Azure services, capabilities, and best practices. Remove any information that is no longer accurate or relevant.
- For reliability service guides, the data aligns with the template.
- The data is edited for quality.
- The `author` and `ms.author` reflect the correct durable owner of this data.
