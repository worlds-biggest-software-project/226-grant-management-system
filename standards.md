# Standards & API Reference

> Project: Grant Management System · Generated: 2026-05-03

## Industry Standards & Specifications

### Federal Grant Compliance Regulations

**OMB 2 CFR Part 200 — Uniform Guidance for Federal Awards**
- **URL:** https://www.ecfr.gov/current/title-2/subtitle-A/chapter-II/part-200
- The primary federal framework governing administrative requirements, cost principles, and audit requirements for all federal grant recipients. The 2024 final rule (effective October 1, 2024) is the most significant update since 2013; key changes include raising the single audit threshold to $1 million, raising the de minimis indirect cost rate to 15%, and adding explicit cybersecurity requirements for protecting PII. Any grant management system handling federal awards must accommodate these requirements.

**Single Audit Act — 2 CFR Part 200 Subpart F**
- **URL:** https://www.ecfr.gov/current/title-2/subtitle-A/chapter-II/part-200/subpart-F
- Requires entities spending $1 million or more in federal awards per fiscal year (threshold raised from $750K in FY2026) to undergo an independent single audit. Grant management platforms must support Schedule of Expenditures of Federal Awards (SEFA) data collection, tracking of Assistance Listing Numbers (ALNs), and submission to the Federal Audit Clearinghouse (FAC).

**Federal Funding Accountability and Transparency Act (FFATA)**
- **URL:** https://www.usaspending.gov/
- Mandates public reporting of all federal awards on USASpending.gov. Grant management systems must support data export in formats compatible with USASpending reporting requirements, including recipient UEI, award amounts, project descriptions, and performance data.

### Data Reporting Standards

**GREAT Act — Grant Reporting Efficiency and Agreements Transparency Act (2019)**
- **URL:** https://www.grants.gov/data-standards
- Requires the federal government to adopt standardised, machine-readable data structures for all grants reporting. OMB has identified 540 core data elements for grant management (funding amounts, project dates, locations, recipient information) that must be machine-readable. Grant management platforms must align their data models to the finalised core data elements published under OMB Memorandum M-24-11.

**XBRL Grants Taxonomy**
- **URL:** https://xbrl.us/home/priorities/efficiency/grants-reporting/
- XBRL US developed a Grants Taxonomy to demonstrate how GREAT Act data fields can be expressed in machine-readable, structured format. Relevant for platforms producing federal compliance reports, single audit data, and structured expenditure reports. The taxonomy covers grant recipient data, programme information, and financial data elements.

**DATA Act — Digital Accountability and Transparency Act (2022)**
- **URL:** https://www.xbrl.org/tag/data-act/
- Requires federal agencies to publish standardised, machine-readable spending data. Grant management systems serving federal agencies or large federal grant recipients must produce DATA Act-compliant reporting outputs linking awards, financial accounts, and performance data.

**Financial Data Transparency Act (FDTA)**
- **URL:** https://xbrl.us/research/data-standards-and-fdta/
- Extends machine-readable data requirements to eight major US financial regulatory agencies. Relevant for grant management systems used by regulated financial entities (e.g., community development financial institutions receiving federal grants).

### Entity & Award Identification Standards

**SAM.gov / Unique Entity Identifier (UEI)**
- **URL:** https://open.gsa.gov/api/entity-api/
- All organisations receiving federal funds must be registered in SAM.gov and assigned a Unique Entity Identifier (UEI) by GSA, replacing the former DUNS number (from April 2022). Grant management systems must validate and store UEI numbers for all applicants and subrecipients receiving federal awards.

**Assistance Listing Numbers (ALNs) — formerly CFDA Numbers**
- Federal programme identifiers required on all SEFA schedules and federal award data submissions. Grant management systems tracking federal awards must capture and display ALNs for each award.

### W3C & Accessibility Standards

**Web Content Accessibility Guidelines (WCAG) 2.2 — W3C**
- **URL:** https://www.w3.org/TR/WCAG22/
- The de facto standard for web accessibility. Grant application portals used by government agencies must comply with WCAG 2.1 Level AA as required by the ADA Title II final rule (published April 2024), with compliance deadlines from April 2026. WCAG 2.2 added nine new success criteria addressing visual, mobility, and cognitive accessibility. Compliance is essential for any publicly funded grant portal.

**W3C ARIA (Accessible Rich Internet Applications)**
- **URL:** https://www.w3.org/WAI/ARIA/
- WAI-ARIA standards for making dynamic web content and grant application forms accessible to screen readers and assistive technologies. Required for complex multi-step application form UIs.

### API & Data Exchange Standards

**OpenAPI Specification 3.1.1**
- **URL:** https://spec.openapis.org/oas/v3.1.1.html
- The standard for documenting and describing REST APIs. All public-facing grant management APIs should be described using OpenAPI 3.1 to enable integration, SDK generation, and third-party connectivity. JSON Schema is fully aligned with OpenAPI 3.1.

**JSON Schema (Draft 2020-12)**
- **URL:** https://json-schema.org/specification
- Standard for validating and documenting JSON data structures used in API request/response payloads. Critical for defining grant application data models, award records, budget line items, and reporting outputs in a portable, interoperable format.

**RFC 6749 — OAuth 2.0 Authorization Framework**
- **URL:** https://datatracker.ietf.org/doc/html/rfc6749
- The standard authorisation protocol for delegated access to APIs. All grant management platforms with REST APIs use OAuth 2.0 (typically the Authorization Code flow) to authenticate third-party integrations and programmatic access.

**OpenID Connect Core 1.0**
- **URL:** https://openid.net/specs/openid-connect-core-1_0.html
- Identity layer built on OAuth 2.0 that enables Single Sign-On (SSO) for applicant portals, reviewer dashboards, and administrative interfaces. Standard requirement for enterprise grant management systems integrating with organisational identity providers (Azure AD, Okta, etc.).

### Security & Compliance Standards

**ISO/IEC 27001:2022 — Information Security Management**
- **URL:** https://www.iso.org/standard/27001
- The international standard for information security management systems (ISMS). ISO 27001:2022 is increasingly required by enterprise customers deploying SaaS grant management platforms; Annex A.5.23 specifically addresses cloud service security. Certification demonstrates that sensitive applicant and grantee data is managed with appropriate controls.

**NIST Cybersecurity Framework 2.0**
- **URL:** https://www.nist.gov/cyberframework
- US government framework for managing cybersecurity risk. 2 CFR Part 200's 2024 update explicitly requires grant recipients to take "reasonable cybersecurity measures to safeguard information including protected PII." NIST CSF provides a practical implementation reference for these requirements.

**OWASP Application Security Verification Standard (ASVS)**
- **URL:** https://owasp.org/www-project-application-security-verification-standard/
- Defines security requirements for web application design, development, and testing. Relevant for grant portal development to prevent injection attacks, authentication bypass, and data exposure of sensitive applicant financial and organisational data.

**GDPR — General Data Protection Regulation (EU)**
- **URL:** https://gdpr.eu/
- Applies to any grant management system collecting data on EU residents (including EU-based applicants or international programme participants). Requires lawful basis for processing, transparency, data subject rights, and appropriate security measures. Critical for platforms serving international funders or applicants.

---

## Similar Products — Developer Documentation & APIs

### Grants.gov / Simpler.Grants.gov

- **Description:** The primary US federal grant opportunity database. Grants.gov hosts all federal funding opportunities; the Simpler.Grants.gov initiative is rebuilding the platform with a modern API-first architecture.
- **API Documentation:** https://www.grants.gov/api and https://simpler.grants.gov/developers
- **API Guide:** https://grants.gov/api/api-guide
- **SDKs/Libraries:** Python client available via dltHub (https://dlthub.com/context/source/grantsgov); REST API usable from any HTTP client
- **Developer Guide:** https://wiki.simpler.grants.gov/product/api
- **Standards:** REST/JSON; OpenAPI described; unauthenticated public search endpoint (`/v1/api/search2`)
- **Authentication:** Public search: no authentication required. System-to-system (S2S) grantor access requires API key obtained via Help Desk ticket.
- **Rate Limits:** 60 requests/minute, 10,000 requests/day per key (Simpler.Grants.gov)

### USASpending.gov API

- **Description:** Comprehensive US federal spending data API covering all contracts, grants, loans, and other financial assistance awards from FY2001 onwards. Provides public access to all FFATA-mandated award transparency data.
- **API Documentation:** https://api.usaspending.gov/
- **SDKs/Libraries:** No official SDK; REST/JSON API with standard HTTP clients; Python examples in GitHub
- **Developer Guide:** https://www.usaspending.gov/federal-spending-guide
- **Standards:** REST/JSON; v2 API; query by award type, agency, recipient, location, time period, ALN
- **Authentication:** No authentication required; completely public API

### SAM.gov Entity Management API (GSA)

- **Description:** Official GSA API for entity registration data, exclusion lists, and contract/grant award data. Required integration point for validating UEI registration status of federal grant applicants and subrecipients.
- **API Documentation:** https://open.gsa.gov/api/entity-api/
- **Opportunity Management API:** https://open.gsa.gov/api/opportunities-api/
- **SDKs/Libraries:** No official SDK; REST/JSON
- **Standards:** REST/JSON; daily updated for active notices, weekly for archived
- **Authentication:** Public access: API key required (available on SAM.gov Account Details page). 10 requests/day public; 1,000/day for registered entities. System accounts require application via SAM.gov workspace.

### Candid Grants API (Foundation Directory)

- **Description:** The most comprehensive database of private, corporate, and community foundation grants in the US. The Grants API provides programmatic access to funder profiles, grant transactions, recipient data, and aggregate funding summaries. Essential for grant prospecting functionality.
- **API Documentation:** https://developer.candid.org/
- **Getting Started:** https://developer.candid.org/reference/get-started-with-grants-api
- **SDKs/Libraries:** REST/JSON API; no official SDKs documented
- **Developer Guide:** https://developer.candid.org/reference/welcome
- **Standards:** REST/JSON; new grants published every business day
- **Key Endpoints:** `/transactions` (grant detail), `/funders` (funder profiles), `/recipients` (nonprofit funding activity), `/summary` (aggregate totals)
- **Authentication:** API key required; access tiers based on subscription level

### Blackbaud SKY API

- **Description:** REST API platform providing access to all Blackbaud products including Raiser's Edge NXT, Blackbaud Grantmaking, and Financial Edge NXT. Enables bi-directional integration between grant management and constituent/fundraising data.
- **API Documentation:** https://developer.blackbaud.com/skyapi/docs
- **API Reference:** https://developer.sky.blackbaud.com/api
- **SDKs/Libraries:** .NET, JavaScript SDKs available via Blackbaud developer portal
- **Developer Guide:** https://developer.blackbaud.com/skyapi/docs/basics
- **Standards:** REST/JSON; OpenAPI described
- **Authentication:** OAuth 2.0 Authorization Code flow; application registration required at developer.blackbaud.com

### Fluxx Grantmaker API

- **Description:** REST API for Fluxx's grantmaker platform, enabling custom integrations with financial systems, CRMs, and business intelligence tools. Supports read/write access to grant records, application data, workflow states, and reporting.
- **API Documentation:** Available at `https://[fluxx-instance]/api/rest/v2/doc` (per-instance documentation)
- **SDKs/Libraries:** Java toolkit available at https://github.com/fluxxlabs/fluxx_api_toolkit_java; open-source Fluxx Exporter at https://github.com/RockefellerArchiveCenter/fluxx_exporter
- **Standards:** REST/JSON; v1 and v2 API versions available
- **Authentication:** OAuth 2.0 (client credentials); API key and secret generated via `[instance]/oauth/applications`

### Foundant GLM API

- **Description:** Data export API for Foundant's Grant and Scholarship Lifecycle Manager. Enables one-way data pulls from Foundant into external CRMs, accounting systems, or BI tools, with HMAC-SHA256 authenticated POST requests.
- **API Documentation:** https://help.foundant.com/39942-advanced-features/316039-api-data-sets
- **Support Portal:** https://support.foundant.com/hc/en-us/articles/4404296012951-API-Data-Sets
- **Standards:** REST/JSON; POST with `application/json` content type
- **Authentication:** HMAC-SHA256 signature; API key ID (public GUID) + 32-byte cryptographic secret; keys configured in GLM Integrations menu

### Submittable API

- **Description:** REST API for Submittable's application and submission management platform. Provides read/write access to submissions, reviewers, forms, and applicant communications. Used for grants, awards, and fellowship programmes.
- **API Documentation (v4):** https://submittable-api.submittable.com/docs/v4/index.html
- **API Documentation (v3):** https://submittable-api.submittable.com/docs/v3/index.html
- **SDKs/Libraries:** Python client: https://submittable-api-client.readthedocs.io/
- **Standards:** REST/JSON; v4 is current version; Swagger/OpenAPI documented
- **Authentication:** Basic Authentication (access token as password)

### Salesforce Nonprofit Cloud / NPSP API

- **Description:** Salesforce REST and SOAP APIs provide access to NPSP grant opportunity records, grantmaker accounts, constituent data, and workflow automation. The Nonprofit Cloud Fundraising Business API provides structured endpoints for grant and fundraising lifecycle management.
- **API Documentation:** https://developer.salesforce.com/docs/atlas.en-us.nonprofit_cloud.meta/nonprofit_cloud/npc_fundraising_business_api_rest_reference.htm
- **NPSP GitHub:** https://github.com/SalesforceFoundation/NPSP
- **SDKs/Libraries:** Salesforce provides official SDKs for Java, Python, JavaScript, .NET, PHP, Ruby via Salesforce DX
- **Developer Guide:** https://developer.salesforce.com/docs/
- **Standards:** REST/JSON and SOAP; Bulk API 2.0 for large data operations; OpenAPI 3.0 for newer endpoints
- **Authentication:** OAuth 2.0 (Authorization Code, Client Credentials, JWT Bearer flows)

---

## Notes

**Grants.gov S2S modernisation:** The Simpler.Grants.gov project (HHS-led, open source at https://github.com/HHS/simpler-grants-gov) is rebuilding the legacy SOAP-based system-to-system interface with a modern REST/JSON API. Grant management platforms should plan migration to the new API as the legacy SOAP interface is phased out.

**GREAT Act data element standardisation:** As of 2026, OMB has finalised the initial core data elements under the GREAT Act (published at grants.gov/data-standards), but GAO has reported that 501 of 540 identified data elements are still not machine-readable. Platforms building federal grant management tooling should monitor OMB guidance for required schema updates as implementation continues.

**Subrecipient monitoring gap:** No industry-standard API or data format currently exists for subrecipient risk data exchange between pass-through entities and subrecipients. This represents a significant opportunity for an open standard that could reduce compliance burden across the sector.

**Emerging MCP / AI agent integration:** As AI-native grant management tools emerge, Model Context Protocol (MCP) server implementations wrapping Grants.gov, USASpending.gov, and Candid APIs would enable LLM agents to perform grant prospecting, eligibility screening, and compliance checking autonomously. No published MCP servers for these APIs were identified as of May 2026.
