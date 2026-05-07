# Org Chart & Directory Tool — Feature & Functionality Survey

> Candidate #477 · Researched: 2026-05-07

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| ChartHop | People analytics + org chart platform | Commercial SaaS (from $2/employee/month) | https://www.charthop.com |
| Workleap Pingboard | Org chart + employee engagement | Commercial SaaS | https://workleap.com/pingboard |
| Organimi | Org chart builder | Commercial SaaS ($0 free tier; paid plans) | https://www.organimi.com |
| Lucidchart | General diagramming (org chart templates) | Commercial SaaS (from ~$9/user/month) | https://www.lucidchart.com |
| OrgChart (theorgchart.com) | Org chart automation + analytics | Commercial SaaS | https://theorgchart.com |
| GoProfiles | AI-powered people directory + org chart | Commercial SaaS (free tier available) | https://www.goprofiles.io |
| Peerdom | Org chart + organisational OS | Commercial SaaS | https://peerdom.com |
| Microsoft Visio | General diagramming | Commercial (Microsoft 365 add-on) | https://www.microsoft.com/en-us/microsoft-365/visio |
| Miro | Visual collaboration (org chart templates) | Commercial SaaS (free tier available) | https://miro.com |
| OneDirectory | Active Directory / M365 directory + org chart | Commercial SaaS | https://www.onedirectory.com |

---

## Feature Analysis by Solution

### ChartHop

**Core features**
- Live bi-directional HRIS sync (Workday, BambooHR, ADP, HiBob, Rippling, UKG, Deel, Personio)
- Interactive, zoomable org chart with department/role/location filters
- Employee profiles with custom fields (cost centre, job family, compensation band, etc.)
- Animated timeline slider — drag through org history to see team evolution
- Headcount planning and scenario modelling with budget impact calculations
- Compensation review workflows
- DEI analytics (gender, ethnicity, pay-band breakdowns by team/level)
- People analytics dashboards (attrition, span of control, open headcount)
- Role-based access controls — field-level privacy (HR-only vs. company-wide)

**Differentiating features**
- Overlay multiple data points (comp, tenure, performance) directly on the org chart view
- Integrated compensation review tool that connects planning directly to the org chart
- Scenario planning with before/after comparison views for proposed reorgs

**UX patterns**
- Dashboard-centric: data visualisation is the primary UI metaphor
- Progressive complexity: chart view for casual browsing, analytics panel for power users
- Filter sidebar to narrow the org chart by department, location, or custom attribute

**Integration points**
- REST API with OAuth 2.0 authentication; webhook event notifications
- 50+ HRIS and payroll connectors via native integrations
- Merge.dev integration for additional connector coverage
- Slack and Teams integrations for directory look-up in chat

**Known gaps**
- Pricing is enterprise-oriented; cost can be prohibitive for mid-market companies
- Custom workflow automation is limited compared to full HCM platforms
- Mobile experience is functional but less polished than the desktop product

**Licence / IP notes**
- Proprietary SaaS; no open-source components exposed. No patents found in public record.

---

### Workleap Pingboard

**Core features**
- Real-time employee directory with photos, contact details, role, team, and location
- Drag-and-drop org chart builder connected to HRIS for auto-updates
- Open-role and planned-headcount mapping directly within the org chart view
- Scenario planning for modelling workforce changes before committing
- Employee engagement: peer recognition, birthday/anniversary celebrations, surveys
- One-on-one meeting facilitation tools
- Manager connect and team games to foster coworker connections

**Differentiating features**
- Org chart as an engagement surface — not just hierarchy, but culture
- "Who is out" status indicators in the directory (PTO, OOO)
- Employee onboarding journeys integrated with the directory

**UX patterns**
- Org-chart-first layout; directory search is a secondary surface
- Card-based employee profiles with an informal, social feel (photos prominent)
- Mobile-first design with a strong companion app

**Integration points**
- HRIS connectors (BambooHR, Rippling, Gusto, Workday, and others)
- Slack and Microsoft Teams integrations
- SSO via SAML 2.0 / OIDC

**Known gaps**
- Contact information can be difficult to find within the directory
- Platform expansion (recognition, surveys) may not suit buyers wanting a focused directory-only tool
- Less powerful analytics than ChartHop

**Licence / IP notes**
- Proprietary SaaS. Acquired by Workleap (December 2023). No open-source components exposed.

---

### Organimi

**Core features**
- Drag-and-drop org chart builder (hierarchical, matrix, accountability, governance chart types)
- CSV/Excel import to auto-generate org charts from employee data
- Custom fields (text, number, URL, tag) with public/private visibility control
- iFrame embed for intranet publishing
- Scenario planning for restructuring and merger modelling
- Printing support with 50+ page-size options and PDF/image export
- Brand customisation (colours, icons, badges per role card)

**Differentiating features**
- Multiple chart types beyond simple hierarchy (matrix, accountability, governance/board)
- Particularly strong print/export layout engine
- Low barrier to entry for small organisations

**UX patterns**
- Template-driven onboarding — users pick a chart type and import data
- Simple, icon-forward card design
- Focused purely on chart visualisation; minimal analytics layer

**Integration points**
- API (REST) available; $1,000 one-time API setup fee
- HRIS API sync refreshes every four hours
- Google Drive, Slack, and Atlassian integrations
- iFrame embed capability

**Known gaps**
- No live sync — data refresh is periodic (every four hours), not real-time
- Limited people analytics beyond the chart view
- API setup fee is a barrier for small organisations
- No mobile app

**Licence / IP notes**
- Proprietary SaaS. API usage requires paid setup. No open-source components.

---

### Lucidchart

**Core features**
- General-purpose drag-and-drop diagramming with org chart templates
- Real-time multi-user collaborative editing
- Data-backed shapes: bind employee fields (name, role, image, custom) to chart nodes via CSV or API
- Export to PDF, PNG, JPEG, SVG, VDX (Visio), and VSDX
- 100+ third-party integrations (Google Workspace, Microsoft 365, Atlassian, Slack, Salesforce)
- Version history and commenting

**Differentiating features**
- Broadest diagramming surface — org charts coexist with flowcharts, ERDs, network diagrams
- Best-in-class Visio import/export compatibility
- Team collaboration patterns already familiar to knowledge workers

**UX patterns**
- Canvas-first, infinite whiteboard metaphor
- Template library for quick starts; org chart wizard for data-driven chart generation
- Filter by shape library to surface org chart templates prominently

**Integration points**
- REST API at api.lucid.co with OAuth 2.0 Bearer tokens
- SCIM 2.0 API at scim.lucid.co for user provisioning
- Lucid Developer Portal (developer.lucid.co) with SDKs
- Zapier, Salesforce, Google Sheets data connectors

**Known gaps**
- Requires manual re-import or API scripting to keep org data current — no live HRIS sync out of the box
- Not purpose-built for people data; lacks employee profiles, DEI analytics, headcount planning
- Overkill for orgs that only need an org chart, not a full diagramming suite

**Licence / IP notes**
- Proprietary SaaS. REST and SCIM APIs are documented publicly. No open-source SDK.

---

### OrgChart (theorgchart.com)

**Core features**
- Automated org chart creation from 50+ HRIS systems
- Drag-and-drop chart editing with real-time updates
- What-if scenario modelling and structural comparison views
- Workforce analytics: span of control, headcount, open roles
- Succession planning module: map employees to at-risk or open roles
- Custom fields and configurable card layouts
- Export to PNG, PDF, PowerPoint

**Differentiating features**
- Deepest HRIS connector library in the segment (50+ systems)
- Succession planning tightly embedded in the org chart view
- Strong enterprise configuration options (on-premise deployment available)

**UX patterns**
- HR-admin-centric workflow; complex configuration surface
- Wizard-driven HRIS connection setup
- Role-based access controls with field-level granularity

**Integration points**
- REST API for HRIS data sync
- SAML 2.0 SSO
- On-premise and cloud deployment options

**Known gaps**
- UI is considered dated compared to newer entrants
- Onboarding complexity for non-technical HR admins
- Less emphasis on the employee self-service / directory use case

**Licence / IP notes**
- Proprietary SaaS with on-premise option. No open-source components.

---

### GoProfiles

**Core features**
- AI-powered people directory with natural-language Q&A ("Who reports to John in marketing?")
- Org chart auto-populated from HRIS with real-time sync
- Rich employee profiles (skills, interests, social links, custom fields)
- Peer recognition and rewards (gift cards, charitable donations)
- Employee group roles with custom badges (Team Lead, Coordinator, etc.)
- Slack as a source-of-truth for profile photos
- Compact and expanded org chart views
- Mobile accessible

**Differentiating features**
- Generative AI assistant for people-data queries embedded in the directory
- Recognition and rewards tightly coupled to directory, making engagement a first-class feature
- Free tier with broad feature coverage for small teams

**UX patterns**
- AI chat interface for directory search alongside traditional browse/filter
- Card-based profile design with emphasis on photos and personal details
- Group pages with role badges for team identity

**Integration points**
- HRIS connectors (BambooHR, Rippling, Gusto, Workday, ADP)
- Slack and Microsoft Teams integrations (expanded as of March 2026)
- API available (details undisclosed publicly)

**Known gaps**
- Recognition/rewards scope may bloat the product for orgs wanting pure directory + org chart
- Analytics depth is lighter than ChartHop
- API documentation not publicly prominent

**Licence / IP notes**
- Proprietary SaaS. Free tier. No open-source components.

---

### Peerdom

**Core features**
- Organisational map with multiple views: circle (Holacracy-style), tree, and list
- Modular app ecosystem: Goals (OKR/KPI), Projects, Directory, Journal (audit trail), Elections, Feedback, Drafts, Network, Insights, Pages, Contribution
- Role-based (not just person-based) org modelling — roles can be assigned to multiple people
- Structural drafts for testing changes before publishing
- Cross-organisational network graph
- Insights module: role concentration, workload, vacancies analytics
- Change history and audit trail via Journal module

**Differentiating features**
- Supports Holacracy and sociocracy governance models out of the box
- Role-centric rather than person-centric model — designed for self-managing organisations
- GraphQL API (not just REST)
- Webhooks, Zapier, Pipedream, and n8n connectors

**UX patterns**
- Circle view for self-managing orgs; tree view for conventional hierarchy
- Role cards rather than person cards as primary UI unit
- Configuration-heavy; best suited to orgs that have deliberately adopted a governance framework

**Integration points**
- GraphQL API
- SSO
- Webhooks
- Zapier, Pipedream, n8n automation connectors

**Known gaps**
- Steep learning curve for organisations not adopting holacratic/sociocratic governance
- Overkill for companies wanting a simple directory and reporting-line chart
- Niche audience limits market reach

**Licence / IP notes**
- Proprietary SaaS. GraphQL API documented for developers. No open-source components.

---

### Microsoft Visio

**Core features**
- General-purpose diagramming with org chart wizard
- Data-linked shapes: connect Excel/SQL/SharePoint data to diagram shapes
- Real-time co-authoring (Microsoft 365 integration)
- Export to PDF, PNG, SVG, VSDX
- Wide template library including org chart layouts
- Embedded in Microsoft 365 ecosystem (Teams, SharePoint)

**Differentiating features**
- Deep Microsoft 365 integration — org charts surface natively in Teams and SharePoint
- Best-in-class Visio file format compatibility

**UX patterns**
- Desktop-application metaphor even in the browser version
- Ribbon-based toolbar familiar to Office users

**Integration points**
- Microsoft Graph API for employee/manager data
- SharePoint and Teams embedding
- Excel and SQL data linking for chart population

**Known gaps**
- No live HRIS sync; data connection is pull-based and manual
- Requires Microsoft 365 subscription; not accessible to non-Microsoft environments
- No people analytics or directory features

**Licence / IP notes**
- Proprietary Microsoft product. VSDX format is an open ECMA standard (ECMA-378).

---

### OneDirectory

**Core features**
- Automatically generated org chart and directory from Azure Active Directory / Microsoft 365
- Live sync from AD as source of truth — no manual imports
- Searchable employee directory with profiles drawn from M365 data
- Departmental and team filtering
- Mobile app

**Differentiating features**
- Specifically designed for Microsoft environments — zero-configuration for M365 customers
- True live sync from Active Directory without manual re-import

**UX patterns**
- Directory-first (search and browse), with org chart as a secondary view
- Clean, modern card design using M365 profile photos

**Integration points**
- Microsoft Graph API for all data
- Azure AD / Entra ID
- SAML/OIDC SSO via Microsoft identity platform

**Known gaps**
- Limited to Microsoft / Azure AD environments — not usable for non-Microsoft orgs
- No headcount planning, scenario modelling, or DEI analytics
- Minimal customisation of data fields

**Licence / IP notes**
- Proprietary SaaS targeting Microsoft customers. No open-source components.

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Interactive, zoomable org chart with search and department/location filtering
- Employee profiles with photo, title, team, manager, and contact details
- HRIS data import (CSV at minimum; API sync preferred)
- Export to PNG, PDF, and/or SVG for presentations and reports
- Role-based access controls (HR-only fields vs. company-wide visibility)
- SSO support (SAML 2.0 / OIDC)
- Slack and/or Teams integration for directory look-up

### Differentiating Features
- Live (real-time or near-real-time) HRIS sync without manual re-import
- Headcount planning and scenario modelling with budget impact calculations
- DEI analytics (representation breakdowns by gender, ethnicity, level)
- AI-powered natural-language people search ("Who leads design in EMEA?")
- Succession planning embedded in the org chart view
- Multiple chart types (hierarchical, matrix, circle/Holacracy, governance/board)
- Custom fields with field-level privacy controls
- GraphQL or webhook-based API for developer integration

### Underserved Areas / Opportunities
- Affordable mid-market option combining live HRIS sync, headcount planning, and directory — ChartHop is the only full-featured tool but is expensive
- AI-assisted reorg planning — suggesting span-of-control improvements or role consolidations
- Informal network graph — overlaying communication or collaboration patterns on the formal hierarchy
- Succession bench-strength visualisation in a cost-effective package
- Open-source or self-hostable alternative (no fully-featured open-source tool exists in this space)
- Multi-model org visualisation in one product (traditional hierarchy, circle, matrix, and accountability charts)
- Employee skills and competency mapping integrated with the org chart

### AI-Augmentation Candidates
- Natural-language directory search (already pioneered by GoProfiles)
- Reorg scenario suggestions (identify span-of-control outliers, managerial bottlenecks)
- Automatic field enrichment from public sources (LinkedIn, GitHub) with privacy controls
- Anomaly detection in headcount data (unexpected attrition patterns, duplicate roles)
- Auto-generated org chart narratives for board decks and all-hands presentations

---

## Legal & IP Summary

No patents were found for any features described in publicly available documentation across these products. All solutions are proprietary commercial SaaS. Core features — org chart visualisation, HRIS sync, employee directory, scenario planning, and CSV import/export — are well-established and unpatented. The VSDX file format is covered by an open ECMA standard (ECMA-378), and SCIM 2.0 is an IETF open standard (RFC 7643, 7644), meaning implementations of these standards carry no IP risk. Open-source libraries (e.g. D3.js for visualisation, React, and various graphing libraries) are commonly used in this space under MIT or Apache 2.0 licences and are safe to adopt. No copyright or licensing concerns were identified with the feature list described here.

---

## Recommended Feature Scope

**Must-have (MVP)**
- Interactive hierarchical org chart (zoomable, searchable, filterable by department/location)
- Searchable employee directory with rich profiles (photo, title, team, manager, contact, custom fields)
- CSV import and HRIS sync (BambooHR, Rippling, Workday as priority connectors)
- Export to PNG/PDF/SVG
- Role-based access controls and field-level privacy
- SSO via SAML 2.0 / OIDC
- Slack integration for in-chat directory look-up

**Should-have (v1.1)**
- Headcount planning and scenario modelling (draft future org states with budget impact)
- Open-role visualisation within the org chart
- AI natural-language people search
- DEI breakdown analytics (representation by gender, level, department)
- Webhook and REST API for developer integrations
- Multiple chart view types (matrix, flat/list)
- Audit trail / change history

**Nice-to-have (backlog)**
- Succession planning module (bench-strength mapping to open/at-risk roles)
- AI reorg assistant (span-of-control analysis and structural recommendations)
- Informal network graph overlay (collaboration pattern analysis)
- Employee skills and competency mapping
- Mobile apps (iOS and Android)
- On-premise / self-hosted deployment option
