# Org Chart & Directory Tool

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An open-source, AI-native platform that keeps org charts and employee directories accurate, searchable, and actionable through live HRIS synchronisation and intelligent people analytics.

Org Chart & Directory Tool gives HR teams, people-ops leaders, and every employee a single source of truth for organisational structure. It combines an interactive, zoomable org chart with a rich employee directory, headcount planning, and DEI analytics -- all kept current through real-time HRIS sync rather than manual CSV re-imports. Designed as an affordable, self-hostable alternative to enterprise platforms like ChartHop, it targets mid-market organisations that need live data sync and planning capabilities without enterprise-tier pricing.

---

## Why Org Chart & Directory Tool?

- **Enterprise tools are overpriced for mid-market.** ChartHop is the only platform offering live HRIS sync, headcount planning, and DEI analytics in one product, but its per-employee pricing puts it out of reach for many mid-market companies.
- **General-purpose diagramming tools require manual upkeep.** Lucidchart and Microsoft Visio can draw org charts but have no live HRIS connection, meaning every hire, departure, or reorg requires a manual re-import to keep charts current.
- **No fully-featured open-source option exists.** Every solution in this space is proprietary commercial SaaS. There is no self-hostable, open-source alternative for organisations with data-sovereignty requirements or budget constraints.
- **Focused tools lack analytics depth.** Organimi and Workleap Pingboard offer clean chart and directory experiences but provide limited or no people analytics, DEI reporting, or headcount planning.
- **AI-powered people search is still rare.** Only GoProfiles offers natural-language queries against people data; most tools still rely on filter-and-browse patterns that scale poorly in large organisations.

---

## Key Features

### Interactive Org Chart & Directory

- Zoomable, searchable hierarchical org chart with department, location, and role filters
- Multiple chart view types: hierarchical, matrix, flat/list
- Rich employee profiles with photo, title, team, manager, contact details, and custom fields
- Field-level privacy controls (HR-only vs. company-wide visibility)

### Live HRIS Synchronisation

- Real-time or near-real-time sync with BambooHR, Rippling, Workday, ADP, HiBob, and other HRIS systems
- CSV import as a fallback for systems without API connectors
- Open-role visualisation directly within the org chart

### Headcount Planning & Scenario Modelling

- Draft future-state org structures and compare scenarios before committing
- Budget impact calculations for proposed reorgs
- Succession planning module mapping high-potential employees to open or at-risk roles

### People Analytics & DEI Reporting

- DEI breakdowns by gender, ethnicity, pay band, level, and department
- Attrition rates, span-of-control distribution, team growth trends, and open-headcount tracking
- Audit trail and change history for organisational structure

### Integrations & Export

- Slack and Microsoft Teams integration for in-chat directory look-up
- SSO via SAML 2.0 / OIDC
- REST API and webhooks for developer integrations
- Export to PNG, PDF, and SVG for presentations and reports

---

## AI-Native Advantage

AI capabilities move this tool beyond static chart rendering into an intelligent organisational planning assistant. Natural-language people search lets any employee ask questions like "Who leads design in EMEA?" instead of navigating filter trees. An AI reorg assistant can analyse span-of-control outliers and managerial bottlenecks to suggest structural improvements. Anomaly detection flags unexpected attrition patterns or duplicate roles before they become problems. Auto-generated org chart narratives produce ready-to-use summaries for board decks and all-hands presentations.

---

## Tech Stack & Deployment

The tool is designed for both cloud-hosted and self-hosted deployment to accommodate organisations with data-sovereignty requirements. Org chart visualisation leverages established open-source libraries (D3.js and related graphing libraries under MIT/Apache 2.0 licences). HRIS integration uses REST APIs with OAuth 2.0 authentication, and user provisioning follows the SCIM 2.0 open standard (IETF RFC 7643/7644). SSO is handled via SAML 2.0 and OIDC. A webhook and REST API layer enables third-party automation through tools like Zapier and n8n.

---

## Market Context

The org chart and people-analytics segment spans HR teams, Finance, and People Operations across mid-market and enterprise organisations. ChartHop, the incumbent leader, prices from $2/employee/month, with costs scaling significantly at enterprise tiers. Lucidchart (from ~$9/user/month) and Organimi (free tier with paid plans) serve adjacent needs but lack live data sync and analytics. The primary buyers are HR and People Ops teams seeking a consolidated view of workforce structure, headcount plans, and diversity metrics -- with the directory component serving all employees as a daily reference tool.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
