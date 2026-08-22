# Amazon Directory Service

<!-- API-EVANGELIST-PROVENANCE:BEGIN -->
> ### About this repository
>
> **This is not our API.** This repository is an independent, third-party profile of a company's
> **publicly available** API surface, maintained by [API Evangelist](https://apievangelist.com).
> API Evangelist does not operate, host, resell, or support this company's APIs, and is not
> affiliated with or endorsed by the company unless stated on the profile.
>
> **Where the information came from.** Everything here is assembled from material a member of the
> public can reach with a browser and no credentials — the company's own website, developer portal
> and documentation, the specifications it publishes for public use (OpenAPI, AsyncAPI, JSON Schema,
> `apis.json`, `llms.txt` and similar), its public repositories, and its public status, pricing and
> changelog pages. **Nothing here is obtained by breaching a system, defeating an access control, or
> using credentials of any kind.**
>
> **The rating is an independent assessment.** The Kin Score and Agent Readiness rating are
> independently calculated scores of a company's *public* API artifacts, produced by API Evangelist
> against a published rubric. They are not certifications, endorsements, security assessments, or
> audits, and they score published artifacts — not the quality, safety, or security of the software.
>
> **Corrections, re-scores, and removal are free.** No partnership, contract, or purchase is
> required, and you do not need to justify the request.
>
> - **Something wrong?** Open an issue on this repository, or email
>   [info@apievangelist.com](mailto:info@apievangelist.com).
> - **Published something new?** Ask for a re-score and we will re-run the rating.
> - **Want the listing taken down?** Say so and we will honor it. The profile is reduced to your
>   company name, a factual description, and a link to your own site, and the company is recorded as
>   **unrated** — never scored zero for having asked.
>
> **Response times.** Acknowledgement within **one business day**; removal or restriction within
> **two business days**; corrections and re-scores within **five business days**.
>
> **Not from the company, and here with a question?** You are welcome here — we would rather be the
> front line and point you the right way than have a good report go nowhere. What this repository
> can answer is narrow, though, so it is worth knowing who you are actually looking for:
>
> - **A question about how the API works, an account, billing, or a bug in the service** — that is
>   the company's own support, not us. We profile this API; we do not operate it and cannot see
>   your account.
> - **A bug in an open-source project we only catalog** — file it on that project's own repository.
>   This has happened with a real and correct bug report that reached us instead of the people who
>   could fix it, which helped nobody.
> - **Anything about this listing itself** — the description, the tags, the rating, a missing or
>   wrong artifact — is ours. Open an issue here.
> - **Not sure, or something general about API Evangelist or APIs.io** — open an issue on the
>   [APIs.io Inbox](https://github.com/api-search/inbox) and we will route it.
>
> **This repository contains no software, and we will never ask you to download anything.** There is
> no build, release, installer, or binary here — only text and machine-readable API descriptions, so
> there is nothing here that can be "corrupt" or need "repairing". Any issue, comment, or email
> claiming otherwise and offering a download link is not from us and is hostile. Do not follow the
> link; it is a lure. Report it to GitHub and, if you like, tell us at
> [info@apievangelist.com](mailto:info@apievangelist.com) so we can take it down.
>
> **On a security or compliance team?** Email
> [info@apievangelist.com](mailto:info@apievangelist.com) with *security* in the subject line and
> you will get a person, not a form. We will tell you exactly which public URLs this profile was
> built from so your team can see the same surface we did, and we will take the listing down on
> request while you work through it.
>
> Full detail: **[Where this data comes from](https://apievangelist.com/about/where-our-data-comes-from)**
<!-- API-EVANGELIST-PROVENANCE:END -->

AWS Directory Service for Microsoft Active Directory, also known as AWS Managed Microsoft AD, enables your directory-aware workloads and AWS resources to use managed Active Directory in AWS. It provides a fully managed, highly available Microsoft Active Directory in the AWS Cloud, with features including trust relationships, domain controllers, LDAPS, and multi-account directory sharing.

- **URL:** https://aws.amazon.com/directoryservice/
- **Run:** [![Run in Postman](https://run.pstmn.io/button.svg)](https://www.postman.com/api-evangelist/amazon-web-services/collection/)

**Tags:** Active Directory, Authentication, AWS, Directory Services, Identity Management

**Created:** 2026-03-16 | **Modified:** 2026-04-19

## APIs

- **AWS Directory Service API** - Provides programmatic access to create and manage directories, trusts, snapshots, and domain controllers for Microsoft Active Directory in the AWS Cloud.
  - **Base URL:** https://ds.amazonaws.com
  - **Version:** 2015-04-16
  - **OpenAPI:** [amazon-directory-service-openapi.yaml](openapi/amazon-directory-service-openapi.yaml)

### Common Properties

| Property | URL |
|---|---|
| Documentation | https://docs.aws.amazon.com/directoryservice/latest/devguide/what_is.html |
| Getting Started | https://aws.amazon.com/directoryservice/getting-started/ |
| Terms of Service | https://aws.amazon.com/service-terms/ |
| Privacy Policy | https://aws.amazon.com/privacy/ |
| Sign Up | https://portal.aws.amazon.com/billing/signup |
| GitHub Organization | https://github.com/aws |
| Status Page | https://health.aws.amazon.com/health/status |

### Features

| Feature | Description |
|---|---|
| Managed Microsoft AD | Fully managed AWS Managed Microsoft Active Directory with automatic patching and monitoring |
| Simple AD | Standalone managed directory powered by Samba 4 for basic AD functionality |
| AD Connector | Proxy service for connecting AWS applications to existing on-premises AD |
| Trust Relationships | One-way and two-way trust relationships between AWS and on-premises directories |
| Multi-Region Replication | Replicate your AWS Managed Microsoft AD across multiple AWS Regions |
| Directory Sharing | Share a single directory across multiple AWS accounts and VPCs |

### Use Cases

| Use Case | Description |
|---|---|
| Hybrid Identity | Extend on-premises Active Directory into AWS for unified identity management |
| Workload Authentication | Enable Windows and Linux workloads to join and authenticate against managed AD |
| AWS Application Integration | Use managed AD for AWS WorkSpaces, RDS, and other AD-aware services |
| LDAPS Encryption | Secure LDAP communications with certificates for compliance requirements |
| Disaster Recovery | Use directory snapshots for point-in-time recovery of directory data |

### Integrations

| Integration | Description |
|---|---|
| Amazon WorkSpaces | Join WorkSpaces desktops to managed AD for enterprise desktop management |
| Amazon RDS | Enable Windows Authentication for SQL Server RDS instances via managed AD |
| AWS IAM Identity Center | Use managed AD as identity source for centralized access management |
| AWS CloudTrail | Audit all Directory Service API calls for compliance and security monitoring |
| Amazon SNS | Receive directory event notifications via SNS topic subscriptions |

## Artifacts

### OpenAPI

| Name | Description | File |
|---|---|---|
| AWS Directory Service OpenAPI | OpenAPI 3.x specification for the AWS Directory Service API | [amazon-directory-service-openapi.yaml](openapi/amazon-directory-service-openapi.yaml) |

### JSON Schema

| Schema | Description | File |
|---|---|---|
| DirectoryDescription | Full description of a managed directory | [amazon-directory-service-directory-description-schema.json](json-schema/amazon-directory-service-directory-description-schema.json) |
| Trust | A trust relationship between directories | [amazon-directory-service-trust-schema.json](json-schema/amazon-directory-service-trust-schema.json) |
| Snapshot | A point-in-time snapshot of a directory | [amazon-directory-service-snapshot-schema.json](json-schema/amazon-directory-service-snapshot-schema.json) |
| Certificate | A certificate registered for directory authentication | [amazon-directory-service-certificate-schema.json](json-schema/amazon-directory-service-certificate-schema.json) |
| IpRouteInfo | An IP route entry for directory traffic | [amazon-directory-service-ip-route-info-schema.json](json-schema/amazon-directory-service-ip-route-info-schema.json) |
| ConditionalForwarder | A DNS conditional forwarder for a directory | [amazon-directory-service-conditional-forwarder-schema.json](json-schema/amazon-directory-service-conditional-forwarder-schema.json) |

### JSON Structure

| Schema | Description | File |
|---|---|---|
| DirectoryDescription | JSON Structure for DirectoryDescription | [amazon-directory-service-directory-description-structure.json](json-structure/amazon-directory-service-directory-description-structure.json) |
| Trust | JSON Structure for Trust | [amazon-directory-service-trust-structure.json](json-structure/amazon-directory-service-trust-structure.json) |
| Snapshot | JSON Structure for Snapshot | [amazon-directory-service-snapshot-structure.json](json-structure/amazon-directory-service-snapshot-structure.json) |

### JSON-LD

| Name | Description | File |
|---|---|---|
| Amazon Directory Service Context | JSON-LD 1.1 context for Amazon Directory Service types and properties | [amazon-directory-service-context.jsonld](json-ld/amazon-directory-service-context.jsonld) |

## Capabilities

### Shared API Definitions

| Name | Description | File |
|---|---|---|
| Amazon Directory Service API | Shared Naftiko capability definition for Directory Service API operations | [directory-service-api.yaml](capabilities/shared/directory-service-api.yaml) |

### Workflows

| Workflow | Description | Tools | Personas | File |
|---|---|---|---|---|
| Active Directory Management | End-to-end Active Directory lifecycle management using Amazon Directory Service | 14 | Identity Engineer, Cloud Architect | [active-directory-management.yaml](capabilities/active-directory-management.yaml) |

## Vocabulary

| Name | Description | File |
|---|---|---|
| Amazon Directory Service Vocabulary | Unified taxonomy mapping operational and capability dimensions | [amazon-directory-service-vocabulary.yaml](vocabulary/amazon-directory-service-vocabulary.yaml) |

## Rules

| Name | Description | File |
|---|---|---|
| Amazon Directory Service Spectral Rules | Spectral governance rules for AWS Directory Service API quality | [amazon-directory-service-spectral-rules.yml](rules/amazon-directory-service-spectral-rules.yml) |

## Maintainers

- Kin Lane - kin@apievangelist.com
