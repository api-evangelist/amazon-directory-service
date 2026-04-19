# Amazon Directory Service

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
