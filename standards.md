# Standards & API Reference

> Project: Org Chart & Directory Tool · Generated: 2026-05-07

## Industry Standards & Specifications

### ISO Standards

No ISO standard directly governs org chart data interchange. Relevant adjacent standards include:

- **ISO/IEC 27001 — Information Security Management Systems**
  URL: https://www.iso.org/standard/27001
  Relevant because org chart and directory tools store sensitive HR data (reporting lines, compensation bands, personal profiles). Compliance with ISO 27001 is increasingly required by enterprise buyers.

- **ISO/IEC 27018 — Protection of PII in Public Clouds**
  URL: https://www.iso.org/standard/76559.html
  Relevant for SaaS deployments of directory tools; governs how personally identifiable information (employee profiles) must be handled in cloud environments.

---

### W3C & IETF Standards

- **vCard 4.0 — RFC 6350 (IETF)**
  URL: https://datatracker.ietf.org/doc/html/rfc6350
  The canonical standard for representing and exchanging contact and person data. Directly applicable to employee profile records in a directory tool.

- **jCard — RFC 7095 (IETF)**
  URL: https://datatracker.ietf.org/doc/html/rfc7095
  The JSON encoding of vCard. Relevant for directory APIs that need to represent employee profile data in JSON format in a standards-compliant way.

- **vCard Ontology — W3C**
  URL: https://www.w3.org/TR/vcard-rdf/
  RDF/OWL representation of vCard, enabling semantic web compatibility and JSON-LD person representations. Useful for linked-data or knowledge-graph integrations.

- **RFC 7644 — SCIM 2.0 Protocol (IETF)**
  URL: https://datatracker.ietf.org/doc/html/rfc7644
  The System for Cross-domain Identity Management protocol standard. Defines RESTful APIs for provisioning and managing user accounts across systems — directly relevant for syncing employee records between HRIS, directory tools, and identity providers. Companion standards: RFC 7642 (Definitions) and RFC 7643 (Core Schema).

- **RFC 7643 — SCIM 2.0 Core Schema (IETF)**
  URL: https://datatracker.ietf.org/doc/html/rfc7643
  Defines the data model for SCIM user and group objects. The `manager` attribute and group membership directly model reporting-line hierarchies applicable to org charts.

- **JSON-LD 1.1 — W3C**
  URL: https://www.w3.org/TR/json-ld11/
  Linked data format based on JSON. Schema.org Person and Organization types can be expressed via JSON-LD, providing a standards-based model for employee profile data interchange.

---

### Data Model & API Specifications

- **Schema.org Person**
  URL: https://schema.org/Person
  Structured vocabulary for describing a person, including `jobTitle`, `worksFor`, `memberOf`, `email`, `telephone`, and `image`. Widely used for semantic data interoperability in HR and directory contexts.

- **HR Open Standards — HR-XML / HR Open JSON**
  URL: https://www.hropenstandards.org/
  An independent non-profit consortium (founded 1999) that publishes XML and JSON schema standards for HR data exchange, including employee records, competencies, organisational units, and reporting relationships. Free to download. Directly applicable for HRIS sync data models.

- **OpenAPI 3.x**
  URL: https://spec.openapis.org/oas/latest.html
  The de facto standard for documenting REST APIs. Any developer-facing org chart API should publish an OpenAPI 3.x specification to ease third-party integrations and SDK generation.

- **GraphQL Specification**
  URL: https://spec.graphql.org/
  Increasingly used in org chart tools (e.g. Peerdom) for flexible, developer-friendly querying of people data. Relevant where consumers need to traverse manager chains or filter employees by arbitrary attributes in a single request.

- **ECMA-378 — Open Packaging Conventions (Visio VSDX)**
  URL: https://www.ecma-international.org/publications-and-standards/standards/ecma-378/
  The open standard governing the VSDX file format used by Microsoft Visio. Implementing VSDX import/export enables interoperability with the dominant diagramming format in enterprise environments.

---

### Security & Authentication Standards

- **OAuth 2.0 — RFC 6749 (IETF)**
  URL: https://datatracker.ietf.org/doc/html/rfc6749
  The standard authorisation framework for API access delegation. Required for any API that allows third-party applications to read or update org/directory data on behalf of a user.

- **OpenID Connect 1.0 (OIDC)**
  URL: https://openid.net/specs/openid-connect-core-1_0.html
  Authentication layer built on OAuth 2.0. The primary standard for SSO in modern SaaS enterprise applications. Required for integrating with Okta, Azure Entra ID, Google Workspace, and other identity providers.

- **SAML 2.0**
  URL: https://docs.oasis-open.org/security/saml/v2.0/saml-core-2.0-os.pdf
  The XML-based SSO standard widely required for enterprise customers using legacy identity providers (ADFS, Shibboleth). Most org chart tools support SAML 2.0 alongside OIDC.

- **SCIM 2.0 (see RFC 7644 above)**
  Relevant here also as the standard for automated user provisioning and de-provisioning. Ensures employee records are added/removed in the directory when they are added/removed in the IdP.

- **GDPR — EU Regulation 2016/679**
  URL: https://gdpr.eu/
  Not a technical standard, but a mandatory regulatory framework for any directory tool that stores personal data for EU employees. Governs data minimisation, right to erasure, access controls, and data residency. Field-level privacy controls in the directory are partly driven by GDPR compliance requirements.

- **OWASP API Security Top 10**
  URL: https://owasp.org/www-project-api-security/
  The reference checklist for API security. Particularly relevant for org chart tools since directory APIs expose sensitive organisational metadata; broken object level authorisation (BOLA) is a common risk.

---

### MCP Server Specifications

A Model Context Protocol (MCP) server for an org chart and directory tool would allow AI agents (Claude, GPT-4o, etc.) to query organisational structure, look up employee profiles, and perform headcount analytics in natural language. Relevant MCP specifications:

- **Model Context Protocol (MCP) — Anthropic**
  URL: https://modelcontextprotocol.io/
  An open protocol for connecting AI assistants to external data sources and tools. Implementing an MCP server for the org chart tool would allow AI agents to answer questions like "Who are the direct reports of the VP of Engineering?" without custom integrations.

- **MCP Tools & Resources Specification**
  URL: https://spec.modelcontextprotocol.io/specification/
  Defines the `tools` and `resources` primitives used in MCP servers. Org chart data maps naturally to both resources (employee profiles, org nodes) and tools (search, traverse hierarchy, run analytics queries).

---

## Similar Products — Developer Documentation & APIs

### ChartHop

- **Description:** People analytics and org chart platform with live HRIS sync, headcount planning, and DEI analytics.
- **API Documentation:** https://docs.charthop.com/charthop-for-developers
- **Developer Basics:** https://docs.charthop.com/developer-basics
- **Data Sync Guide:** https://docs.charthop.com/syncing-data-tofrom-charthop
- **SDKs/Libraries:** No public SDK; REST API with JSON
- **Standards:** REST/JSON, OAuth 2.0, Webhook event notifications
- **Authentication:** OAuth 2.0 (Bearer tokens)

---

### Lucidchart (Lucid)

- **Description:** General-purpose diagramming platform with org chart templates and a data-backed shapes API for generating charts from employee data.
- **API Documentation:** https://developer.lucid.co/
- **Developer Portal:** https://lucid.co/developers
- **Standards:** REST API (api.lucid.co), SCIM 2.0 API (scim.lucid.co), OpenAPI
- **Authentication:** OAuth 2.0 Bearer tokens (REST); SCIM Bearer token (SCIM)
- **Notes:** Supports data-backed shapes with `idField`, `foreignKeyField`, `nameField`, `roleField`, `imageUrlField` parameters for automated org chart generation.

---

### Peerdom

- **Description:** Organisational operating system supporting hierarchical, circle (Holacracy), and matrix org models with a full modular app suite.
- **API Documentation:** https://peerdom.com/resources/ (developer docs via sign-in)
- **Standards:** GraphQL API, Webhooks
- **Authentication:** SSO (OIDC)
- **Integration connectors:** Zapier, Pipedream, n8n
- **Notes:** GraphQL is an architectural differentiator in this space; enables flexible traversal of role and person graphs.

---

### Microsoft Graph API (for org chart data)

- **Description:** Microsoft's unified REST API for accessing Microsoft 365 data, including user profiles, manager relationships, and group memberships from Azure Entra ID.
- **API Documentation:** https://learn.microsoft.com/en-us/graph/identity-network-access-overview
- **Manager chain endpoint:** `GET /users/{id}?$expand=manager($levels=n)`
- **Organisation endpoint:** `GET /organization`
- **SDKs/Libraries:** .NET, JavaScript, Python, Java, Go, PHP (https://learn.microsoft.com/en-us/graph/sdks/sdks-overview)
- **Standards:** REST/JSON, OData v4, OpenAPI
- **Authentication:** OAuth 2.0 / OpenID Connect via Microsoft Entra ID

---

### Merge.dev Unified HRIS API

- **Description:** Unified API layer abstracting 50+ HRIS, payroll, and directory platforms behind a single REST interface. Allows org chart tools to integrate once and support Workday, BambooHR, ADP, Rippling, HiBob, Deel, Personio, and others.
- **API Documentation:** https://docs.merge.dev/hris/
- **ChartHop integration docs:** https://docs.merge.dev/merge-unified/hris/integrations/charthop
- **SDKs/Libraries:** JavaScript, Python, Ruby, Java, Go (https://docs.merge.dev/merge-unified/hris/merge-api-basics/sdks)
- **Standards:** REST/JSON, OAuth 2.0
- **Authentication:** API key + OAuth 2.0
- **Notes:** Merge normalises disparate HRIS data models into a common schema including `Employee`, `Department`, `Team`, `Division` objects — directly applicable to org chart data ingestion.

---

### HR Open Standards (HR-XML / HR Open JSON)

- **Description:** Open, non-profit consortium standards for HR data interchange covering employee records, competency frameworks, and organisational units.
- **Documentation:** https://www.hropenstandards.org/documentationinformation
- **Standards Downloads:** https://www.hropenstandards.org/standards-downloads
- **Standards:** XML Schema (HR-XML) and JSON Schema (HR Open JSON)
- **Authentication:** N/A (data model standard, not an API product)
- **Notes:** Defines a standard `OrganizationalUnit` object and person-role relationships. Implementing HR Open schema compatibility improves interoperability with HR vendors.

---

### SCIM 2.0 (simplecloud.info)

- **Description:** The industry-standard REST protocol for automated user provisioning and de-provisioning between identity providers, HRIS platforms, and SaaS applications.
- **Specification:** https://simplecloud.info/
- **RFC 7642:** https://datatracker.ietf.org/doc/html/rfc7642
- **RFC 7643:** https://datatracker.ietf.org/doc/html/rfc7643
- **RFC 7644:** https://datatracker.ietf.org/doc/html/rfc7644
- **Standards:** REST/JSON, standardised `/Users` and `/Groups` endpoints
- **Authentication:** Bearer token
- **Notes:** SCIM's `manager` attribute (in the `EnterpriseUser` schema extension) models reporting-line hierarchy. Implementing a SCIM-compliant endpoint allows identity providers like Okta and Entra ID to push employee updates directly to the org chart tool.

---

### Okta SCIM API

- **Description:** Okta's implementation of SCIM 2.0 for user provisioning — the most widely deployed IdP in enterprise environments. Implementing Okta SCIM compatibility unlocks automatic employee sync for a large share of enterprise customers.
- **API Documentation:** https://developer.okta.com/docs/api/openapi/okta-scim/guides/scim-20
- **Standards:** SCIM 2.0 (RFC 7643, 7644)
- **Authentication:** Bearer token (generated in Okta admin)

---

## Notes

- **Unified HRIS API services** (Merge.dev, Truto, Unified.to, Finch) are emerging as a practical alternative to building native connectors for each HRIS platform. For an AI-native org chart tool, using a unified API layer significantly reduces integration maintenance overhead.
- **GraphQL is underutilised** in this segment — only Peerdom exposes a GraphQL API. For a tool that needs to traverse complex org hierarchies (manager chains, cross-functional relationships, succession paths), GraphQL's recursive query capability offers meaningful developer ergonomics over REST.
- **SCIM's `manager` attribute** in the Enterprise User schema extension (RFC 7643 §4.3) is the closest thing to a standard data model for org hierarchy. However, most HRIS vendors implement it inconsistently — normalisation is still a practical necessity.
- **MCP server opportunity**: No major org chart tool currently exposes an MCP server, leaving a clear gap for an AI-native tool to make people data accessible to AI agents via a standard protocol.
