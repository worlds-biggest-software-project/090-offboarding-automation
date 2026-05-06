# Standards & API Reference

> Project: Offboarding Automation · Generated: 2026-05-06

## Industry Standards & Specifications

### ISO Standards

**ISO/IEC 27001:2022 — Information Security Management Systems**
- URL: https://www.iso.org/standard/27001
- Annex A Control 6.5 ("Responsibilities after termination or change of employment") and Control 5.18 ("Access rights") directly mandate that access rights be revoked upon employment termination. An offboarding automation platform is a primary technical control for satisfying these requirements. Organisations must upgrade to the 2022 version; the transition deadline from ISO 27001:2013 passed in October 2025.

**ISO/IEC 27002:2022 — Information Security Controls**
- URL: https://www.iso.org/standard/75652.html
- Provides detailed implementation guidance for ISO 27001 controls. Control 6.5 specifies that the organisation must communicate post-employment security responsibilities to departing employees and ensure return of all assets, access revocation, and knowledge transfer. An offboarding platform operationalises these controls and generates audit evidence.

**ISO/IEC 29101 — Privacy Architecture Framework**
- URL: https://www.iso.org/standard/75427.html
- Relevant where the offboarding platform processes personal data of departing employees (contact details, payroll, access history). The framework guides how privacy-respecting data flows should be designed in identity and lifecycle management systems.

---

### W3C & IETF Standards

**RFC 7642 — SCIM: Definitions, Overview, Concepts, and Requirements**
- URL: https://datatracker.ietf.org/doc/html/rfc7642
- Establishes the use cases and requirements for the System for Cross-domain Identity Management. Directly relevant: Section 3.4 covers the "Remove User" use case, which is the technical foundation for automated SaaS deprovisioning during offboarding.

**RFC 7643 — SCIM: Core Schema**
- URL: https://datatracker.ietf.org/doc/html/rfc7643
- Defines the canonical data model for user and group resources used across all SCIM-compliant systems. The `active` boolean attribute on the `User` resource is the primary mechanism for signalling deprovisioning across connected apps. Offboarding automation tools must speak this schema to integrate with enterprise identity providers.

**RFC 7644 — SCIM: Protocol**
- URL: https://datatracker.ietf.org/doc/html/rfc7644
- Defines the RESTful HTTP protocol for SCIM operations: `POST` (create), `GET` (retrieve), `PUT`/`PATCH` (modify), and `DELETE` (remove). The `PATCH active=false` and `DELETE /Users/{id}` operations are the two primary API calls made during an offboarding deprovisioning event. All major identity providers (Okta, Entra ID, Google Workspace) and SaaS applications (Slack, GitHub, Jira) expose SCIM 2.0 endpoints conforming to this RFC.

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- OAuth 2.0 is the access delegation standard used by most SaaS APIs an offboarding platform must call. Understanding token scopes (e.g., `admin.directory.user` for Google Workspace, `scim:write` for Slack) is essential. Offboarding also requires revoking OAuth tokens granted by the departing employee to third-party apps — a gap no current tool fully addresses.

**RFC 7009 — OAuth 2.0 Token Revocation**
- URL: https://datatracker.ietf.org/doc/html/rfc7009
- Defines the `/token/revoke` endpoint for cancelling OAuth access and refresh tokens. Directly relevant for offboarding: when an employee departs, all personal OAuth grants they approved (personal SaaS connections, shadow IT) should be revoked. This RFC is the technical foundation for that operation.

**RFC 8693 — OAuth 2.0 Token Exchange**
- URL: https://datatracker.ietf.org/doc/html/rfc8693
- Token exchange enables HR systems and identity providers to delegate offboarding actions to an automation platform without sharing administrative credentials. Useful for building a secure, least-privilege offboarding orchestration layer.

**OpenID Connect Core 1.0**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- The authentication layer built on OAuth 2.0. OIDC is used for SSO; when an employee departs, their OIDC session must be terminated and their identity provider (IdP) account deactivated. SCIM handles the deprovisioning; OIDC defines the session and token structures that must be invalidated.

---

### Data Model & API Specifications

**OpenAPI Specification 3.1**
- URL: https://spec.openapis.org/oas/v3.1.0.html
- The standard format for documenting REST APIs. OpenAPI 3.1 added native webhook support (the `webhooks` top-level element), which is directly useful for offboarding: HRIS systems emit departure events as webhooks, and an OpenAPI 3.1 document can formally describe the inbound webhook schema, enabling auto-generated client SDKs. Any offboarding platform API should publish an OpenAPI 3.1 spec.

**JSON Schema 2020-12**
- URL: https://json-schema.org/specification
- The data-validation standard underlying OpenAPI 3.1. Offboarding event payloads (departure date, employee ID, department, jurisdiction, access inventory) should be validated against JSON Schema definitions to ensure downstream integrations receive well-formed data. JSON Schema 2020-12 is the normative reference.

**SAML 2.0 (Security Assertion Markup Language)**
- URL: https://docs.oasis-open.org/security/saml/v2.0/
- Many enterprise IdPs still use SAML 2.0 for SSO assertions. When an employee is offboarded and their IdP account is deactivated, SAML assertions from that account must be invalidated. Offboarding platforms integrated with enterprise IdPs must understand SAML session termination. XML-heavy; OIDC is the modern replacement, but SAML remains prevalent in large enterprises.

---

### Security & Compliance Standards

**NIST SP 800-53 Rev. 5 — Security and Privacy Controls**
- URL: https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-53r5.pdf
- Two controls are directly served by offboarding automation:
  - **AC-2 (Account Management)**: Requires automated tracking of account creation, modification, and removal. AC-2(1) specifies automated mechanisms; AC-2(3) requires disabling accounts when no longer needed.
  - **PS-4 (Personnel Termination)**: Requires disabling information system access within an organisation-defined timeframe upon employment termination and revoking all authenticators and credentials. Offboarding automation is the primary technical control for PS-4 compliance.

**NIST SP 800-188 — De-Identifying Government Datasets**
- URL: https://csrc.nist.gov/pubs/sp/800/188/final
- Relevant where the platform processes and retains HR data for analytics (e.g., exit interview themes, departure cohort analysis). Provides guidance on de-identification techniques applicable when offboarding data is used for aggregate retention risk analysis without re-identifying individuals.

**SOC 2 (System and Organization Controls 2) — AICPA**
- URL: https://www.aicpa-cima.com/topic/audit-assurance/audit-and-assurance-tools/soc-2
- SOC 2 Type II is the de-facto security certification required for enterprise SaaS. The offboarding domain maps directly to multiple Trust Services Criteria:
  - **CC6.2**: Manages credentials and authorised access; requires documented offboarding with credential revocation SLAs (typically 24 hours).
  - **CC6.3**: Restricts access to authorised users; offboarding audit trails provide the evidence auditors require.
  - Auditors specifically look for evidence that terminated employee accounts were revoked across all systems, with timestamp logs. Lack of documented offboarding is one of the most common SOC 2 findings.

**GDPR Article 17 — Right to Erasure ("Right to be Forgotten")**
- URL: https://gdpr-info.eu/art-17-gdpr/
- Departing employees may request deletion of their personal data. Offboarding platforms must support a data deletion workflow alongside (or after) access revocation. The platform must distinguish between data that must be retained for legal/tax purposes and data eligible for erasure. Response to erasure requests must occur within one month. Multi-jurisdiction offboarding platforms must handle both EU GDPR and UK GDPR variants.

**GDPR Article 5 — Data Minimisation and Storage Limitation**
- URL: https://gdpr-info.eu/art-5-gdpr/
- HR data retained in an offboarding platform beyond the legally required period violates storage limitation principles. Platforms must implement configurable data retention policies that automatically purge personal data after the required retention window (e.g., 6 years for payroll records under UK law, 3 years for US employment records).

**OWASP Application Security Verification Standard (ASVS) v4.0**
- URL: https://owasp.org/www-project-application-security-verification-standard/
- Section 4 (Access Control) is directly relevant: V4.1.1 requires that access controls fail securely (i.e., offboarding failures should default to denying access, not leaving it open). Section 2 (Authentication) covers credential lifecycle management applicable to the offboarding workflow.

**AWS Well-Architected Framework — SEC03-BP06: Manage Access Based on Lifecycle**
- URL: https://docs.aws.amazon.com/wellarchitected/latest/framework/sec_permissions_lifecycle.html
- AWS prescriptive guidance for identity lifecycle management. Specifies that AWS IAM Identity Center deprovisioning must sequence group membership removal, then account assignment deletion, then user deletion — skipping the intermediate steps leaves active IAM role sessions alive for up to 12 hours. Relevant for any offboarding platform that automates AWS access revocation.

---

### MCP Server Specifications

**Model Context Protocol (MCP) — Anthropic**
- URL: https://modelcontextprotocol.io/
- An offboarding automation platform built with AI-native capabilities (knowledge transfer analysis, exit interview synthesis, access discovery) would benefit from exposing MCP-compatible tool interfaces. For example, an MCP server could expose: `list_employee_access(employee_id)`, `revoke_app_access(employee_id, app_id)`, `generate_knowledge_transfer_brief(employee_id)`, and `synthesise_exit_themes(cohort, date_range)`. This would allow Claude and other LLM agents to orchestrate offboarding workflows via natural language.

---

## Similar Products — Developer Documentation & APIs

### Okta (Identity & Lifecycle Management)
- **Description:** Identity provider and lifecycle management platform; the most widely deployed enterprise IdP for provisioning and deprovisioning employees across SaaS apps. Okta's SCIM 2.0 implementation is the de-facto reference for how enterprise IdPs handle offboarding deprovisioning at scale.
- **API Documentation:** https://developer.okta.com/docs/api/openapi/okta-scim/guides/scim-20
- **Lifecycle Management Overview:** https://developer.okta.com/docs/guides/oin-lifecycle-mgmt-overview/
- **SDKs/Libraries:** Okta SDKs for Node.js, Python, Java, Go, .NET: https://developer.okta.com/code/
- **Developer Guide:** https://developer.okta.com/docs/guides/scim-provisioning-integration-overview/main/
- **Standards:** SCIM 2.0 (RFC 7643/7644), OAuth 2.0, OpenID Connect
- **Authentication:** OAuth 2.0 (PKCE for public clients); API tokens for server-side automation

### Microsoft Entra ID (formerly Azure Active Directory)
- **Description:** Microsoft's cloud identity platform; the primary IdP for Microsoft 365 environments. Entra ID Lifecycle Workflows provides templated offboarding automation including account disable, group removal, and Teams offboarding. Supports SCIM 2.0 inbound provisioning and the Microsoft Graph API for custom offboarding orchestration.
- **API Documentation:** https://learn.microsoft.com/en-us/graph/api/resources/user
- **Lifecycle Workflows:** https://learn.microsoft.com/en-us/entra/architecture/governance-deployment-employee-lifecycle
- **SDKs/Libraries:** Microsoft Graph SDK for Python, JavaScript, Java, Go, .NET: https://learn.microsoft.com/en-us/graph/sdks/sdks-overview
- **Developer Guide:** https://learn.microsoft.com/en-us/entra/identity/app-provisioning/configure-workday-termination-lookahead
- **Standards:** SCIM 2.0, OAuth 2.0, OpenID Connect, SAML 2.0
- **Authentication:** OAuth 2.0 with Azure AD tokens; service principal credentials for automation

### Google Workspace Admin SDK (Directory API)
- **Description:** Google's admin API for managing users, groups, and devices within a Google Workspace domain. The primary integration point for offboarding Google Workspace accounts: suspend user (PATCH `suspended: true`), transfer Drive ownership via the Data Transfer API, then delete or retain the account.
- **API Documentation:** https://developers.google.com/workspace/admin/directory/reference/rest
- **Directory API Overview:** https://developers.google.com/workspace/admin/directory/v1/guides
- **SDKs/Libraries:** Google APIs client libraries for Python, Node.js, Java, Go: https://developers.google.com/api-client-library
- **Developer Guide:** https://developers.google.com/workspace/admin/directory/v1/guides/manage-users
- **Standards:** REST/JSON, OpenAPI, OAuth 2.0
- **Authentication:** OAuth 2.0 with `https://www.googleapis.com/auth/admin.directory.user` scope; service accounts for server-to-server automation

### Workday (HRIS — Source of Departure Events)
- **Description:** Enterprise HRIS widely used as the system of record for employee status. The primary event source for offboarding workflows: a termination entered in Workday should trigger downstream deprovisioning. Workday exposes both SOAP (primary HCM surface) and REST APIs, plus configurable webhooks for lifecycle events. Important implementation detail: termination effective date and account disable are independent operations requiring separate API calls.
- **API Documentation:** https://community.workday.com/sites/default/files/file-hosting/productionapi/
- **Webhook Integration:** https://www.getknit.dev/blog/workday-api-integration-in-depth
- **User Management Guide:** https://www.stitchflow.com/user-management/workday/api
- **Standards:** SOAP (primary HCM), REST (growing subset), JSON, XML
- **Authentication:** OAuth 2.0 (REST); WS-Security (SOAP); integration system user credentials

### BambooHR (HRIS — Webhook Source)
- **Description:** Popular SMB HRIS; a common departure-event source for offboarding automation. BambooHR webhooks fire on `employmentStatus` field changes (including termination), delivering employee data to a registered endpoint. Permissioned webhooks (created via API) limit data scope to the creating user's permissions.
- **API Documentation:** https://documentation.bamboohr.com/docs
- **Webhooks Reference:** https://documentation.bamboohr.com/docs/webhooks
- **SDKs/Libraries:** No official SDK; REST JSON API with API key authentication
- **Developer Guide:** https://documentation.bamboohr.com/docs/getting-started
- **Standards:** REST/JSON; webhook payloads in JSON
- **Authentication:** API key passed as HTTP Basic Auth username (password left empty)

### Rippling (Unified HR+IT — Webhooks)
- **Description:** Unified HR, IT, and payroll platform. Exposes webhooks for `employee.terminated` events, enabling external systems to trigger deprovisioning when Rippling records a termination. Rippling's own App Shop handles deprovisioning for its connected apps; custom webhook integrations cover external systems. API key or OAuth 2.0 authentication.
- **API Documentation:** https://www.rippling.com/blog/enterprise-grade-apis
- **Integration Guide:** https://www.stitchflow.com/user-management/rippling/api
- **Standards:** REST/JSON; webhooks
- **Authentication:** OAuth 2.0; API key for server-side access

### Slack SCIM API
- **Description:** Slack's SCIM 2.0 endpoint for user lifecycle management. Deactivating a departing employee deactivates their Slack account (blocks sign-in, removes from channels, revokes app tokens) while preserving message history for compliance. The SCIM API requires Business+ or Enterprise Grid plan.
- **API Documentation:** https://docs.slack.dev/admins/scim-api/
- **SCIM Reference:** https://docs.slack.dev/reference/scim-api/
- **Standards:** SCIM 2.0 (RFC 7643/7644), REST/JSON
- **Authentication:** OAuth token with `admin` scope; token-generating account must remain an admin/owner

### GitHub Enterprise (SCIM)
- **Description:** GitHub's SCIM 2.0 endpoint for user provisioning and deprovisioning in Enterprise Cloud and Enterprise Server. Supports both "soft deprovision" (suspend account, preserve data) and "hard deprovision" (irreversible deletion). All SCIM operations are logged in the enterprise audit log.
- **API Documentation:** https://docs.github.com/en/enterprise-cloud@latest/admin/managing-iam/provisioning-user-accounts-with-scim/deprovisioning-and-reinstating-users
- **SCIM REST API:** https://docs.github.com/en/enterprise-server@3.20/admin/managing-iam/provisioning-user-accounts-with-scim/provisioning-users-and-groups-with-scim-using-the-rest-api
- **Standards:** SCIM 2.0, REST/JSON
- **Authentication:** Personal access token or GitHub App installation token with `admin:org` scope

### Atlassian (Jira/Confluence — User Management API)
- **Description:** Atlassian's cloud user management REST API for deactivating and removing users from Jira and Confluence. Deactivation (`POST /users/{account_id}/manage/lifecycle/disable`) stops sign-in and frees the licence seat while preserving issue history and attribution for audit compliance.
- **API Documentation:** https://developer.atlassian.com/cloud/admin/user-management/rest/api-group-lifecycle/
- **SDKs/Libraries:** Atlassian Python API, JS API client; also accessible via the Atlassian SDK
- **Standards:** REST/JSON, OpenAPI
- **Authentication:** OAuth 2.0 (3LO); API token for basic auth (deprecated for user management in favour of OAuth)

### AWS IAM Identity Center (Formerly AWS SSO)
- **Description:** AWS's central access management service for workforce identities. Supports SCIM 2.0 for automated provisioning from external IdPs (Okta, Entra ID). Direct deprovisioning via the AWS SCIM API requires sequencing: remove group memberships, delete SSO account assignments, then delete the SCIM user — skipping intermediate steps leaves active IAM role sessions alive for up to 12 hours.
- **API Documentation:** https://docs.aws.amazon.com/singlesignon/latest/userguide/users-groups-provisioning.html
- **IAM User Removal:** https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_remove.html
- **Well-Architected Guidance:** https://docs.aws.amazon.com/wellarchitected/latest/framework/sec_permissions_lifecycle.html
- **Standards:** SCIM 2.0, REST/JSON
- **Authentication:** AWS SigV4 (SCIM API); IAM roles for programmatic access

### Merge.dev (Unified HRIS API)
- **Description:** Unified API aggregator that normalises data from 40+ HRIS platforms (BambooHR, Workday, Rippling, ADP, etc.) behind a single REST API. Relevant for offboarding platforms that need to receive departure events from any HRIS without building individual integrations. The `employees` endpoint surfaces `employment_status` and `termination_date` fields standardised across all connected HRIS systems.
- **API Documentation:** https://docs.merge.dev/hris/
- **HRIS Unified API Reference:** https://docs.merge.dev/merge-unified/
- **SDKs/Libraries:** JavaScript, Python, Java, Go, Ruby: https://docs.merge.dev/merge-unified/hris/merge-api-basics/sdks
- **Developer Guide:** https://www.merge.dev/blog/guide-to-hris-api-integrations
- **Standards:** REST/JSON, OpenAPI 3.0
- **Authentication:** API key (linked account token per HRIS integration)

---

## Notes

**SCIM 2.0 is the central protocol.** All major SaaS platforms (Slack, GitHub, Atlassian, AWS, Google, Microsoft) expose SCIM 2.0 endpoints. An offboarding automation platform should consume SCIM 2.0 as its primary deprovisioning protocol, supplementing with vendor-specific REST APIs only where SCIM is unavailable or insufficient (e.g., transferring file ownership in Google Drive before deleting an account).

**No formal offboarding data standard exists.** There is no ISO or IETF standard defining the schema of an "offboarding event" (departure date, reason, asset inventory, access list, knowledge transfer status). This is an open opportunity: a well-designed open-source offboarding platform could publish a normative OpenAPI 3.1 specification for offboarding event payloads that becomes a de-facto community standard — analogous to how OpenTelemetry became the default observability wire format.

**OAuth token revocation is a gap.** RFC 7009 defines how to revoke OAuth tokens, but no current offboarding tool systematically discovers and revokes all OAuth tokens a departing employee granted to third-party apps (personal integrations, shadow IT). This is an area where AI-assisted discovery (scanning email, calendar, and IdP logs for OAuth grant patterns) combined with RFC 7009 revocation calls could address a genuine security gap.

**Deprovisioning sequencing matters.** As documented in the AWS Well-Architected guidance and the Workday integration notes, deprovisioning is not a single API call — it is an ordered sequence of operations (suspend account, transfer data, revoke group memberships, delete access assignments, delete user). Race conditions and incomplete sequences leave security gaps. An offboarding platform must model deprovisioning as a stateful workflow with rollback and retry, not a single fire-and-forget API call.
