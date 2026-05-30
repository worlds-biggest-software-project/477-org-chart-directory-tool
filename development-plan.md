# Org Chart & Directory Tool — Phased Development Plan

> Project: 477-org-chart-directory-tool · Created: 2026-05-31
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four data-model suggestions. It adopts **Data Model Suggestion 3 (Hybrid Relational + JSONB on PostgreSQL)** as the canonical schema, incorporating the `ltree` materialised-path hierarchy from Suggestion 1. The audit trail draws on the change-log concepts from Suggestion 2 without committing to full event sourcing.

The product is an open-source, self-hostable, multi-tenant SaaS that keeps org charts and employee directories current via live HRIS sync, with people analytics, headcount planning, AI-native natural-language search, and an MCP server for AI agents.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language (backend) | **TypeScript (Node.js 22 LTS)** | The product is API- and frontend-heavy (interactive chart UI, REST/GraphQL/webhooks/SCIM). A single TypeScript codebase shared across backend and frontend reduces type drift for the rich `Employee`/`OrgNode` types passed to D3. The AI layer is invoked through SDKs that have first-class TS support. |
| API framework | **NestJS** | Provides structured DI, modular feature boundaries (matching our phase structure), built-in OpenAPI 3.1 generation via decorators, guards for RBAC, and interceptors for tenant scoping and audit logging — all of which we need as cross-cutting concerns. |
| Database | **PostgreSQL 16** | Suggestion 3 hybrid model: relational integrity for the hierarchy + JSONB for HRIS raw payloads, custom fields, and scenario change-sets. `ltree` gives O(1) subtree queries. Self-hostable with mature ops tooling — required by the open-source/data-sovereignty positioning. |
| ORM / migrations | **Prisma** + raw SQL for `ltree`/recursive CTE queries | Prisma gives type-safe models and a migration workflow; ltree and GIST-indexed path queries drop to raw SQL via `prisma.$queryRaw`. |
| Task queue | **BullMQ on Redis** | HRIS syncs, webhook delivery (with retries/backoff), AI narrative generation, and export rendering are async and long-running. BullMQ gives durable jobs, repeatable cron-style sync schedules, and dead-letter handling. |
| Cache / queue store | **Redis 7** | Backs BullMQ, session/rate-limit state, and natural-language-search embedding cache. |
| Frontend | **Next.js 15 (App Router) + React 19** | Server components for fast directory/profile pages; client islands for the interactive chart. SSR helps SEO-free internal tooling load fast and supports iframe embed. |
| Org chart rendering | **D3.js (d3-hierarchy) + react-d3-tree** | MIT-licensed, the established choice for zoomable hierarchical layouts; handles tree, and we extend to matrix/flat views. Confirmed IP-safe by `features.md` legal summary. |
| AI / LLM | **Vercel AI SDK** with provider routing (Anthropic Claude primary, OpenAI fallback) | Natural-language people search, reorg suggestions, anomaly detection, and auto-narratives. Provider-agnostic so self-hosters can point at their own gateway. |
| Embeddings / semantic search | **pgvector** extension | Keeps semantic employee search inside Postgres — no extra vector DB to self-host. |
| HRIS integration | **Merge.dev Unified HRIS API** as the primary connector + native CSV importer fallback | One integration unlocks 50+ HRIS systems (Workday, BambooHR, ADP, Rippling, HiBob, Deel, Personio) per `standards.md`. Reduces per-connector maintenance. |
| Auth (app users) | **OIDC + SAML 2.0** via `@node-saml/node-saml` and `openid-client` | Enterprise SSO is table-stakes (`features.md`). |
| Provisioning | **SCIM 2.0** server (RFC 7643/7644) | Lets Okta/Entra ID push employee + manager updates directly. The `manager` Enterprise extension models reporting lines. |
| AI agent access | **MCP server** (`@modelcontextprotocol/sdk`) | No competitor exposes one — clear differentiator. Exposes org data as MCP resources + tools. |
| Chat integrations | **Slack Bolt SDK**; Teams via Bot Framework (v1.1) | In-chat directory lookup. |
| Export rendering | **Puppeteer** (PNG/PDF) + native SVG serialisation | Server-side chart export to PNG/PDF/SVG. |
| Containerisation | **Docker + docker-compose** | One-command self-host: app, Postgres, Redis. |
| Testing | **Vitest** (unit/integration) + **Playwright** (E2E) | Fast TS-native unit runner; Playwright drives the chart UI and export flows. |
| API contract testing | **Pact** for webhook/SCIM consumers | Verifies SCIM 2.0 and webhook payloads against the spec. |
| Code quality | **ESLint + Prettier + tsc** (strict) | Standard TS toolchain. |
| Package manager / monorepo | **pnpm workspaces + Turborepo** | Shared `packages/types`, `packages/db` across `apps/api`, `apps/web`, `apps/worker`. |
| Observability | **OpenTelemetry + pino** | Structured logging; traces across sync jobs and AI calls. |

### Standards alignment (from `standards.md`)

- **SCIM 2.0** (RFC 7643/7644) — provisioning endpoints (Phase 8).
- **vCard 4.0 / jCard** (RFC 6350/7095) — directory profile export format (Phase 5).
- **Schema.org Person / JSON-LD 1.1** — profile API representation (Phase 5).
- **OpenAPI 3.1** — auto-generated for the public REST API (Phase 7).
- **OAuth 2.0 / OIDC / SAML 2.0** — auth and SSO (Phase 2, Phase 8).
- **OWASP API Security Top 10** — BOLA prevention via mandatory tenant + RBAC guards (Phase 2, cross-cutting).
- **GDPR / ISO 27018** — field-level privacy, right-to-erasure, demographics isolation (Phase 2, Phase 6).
- **MCP** — AI agent server (Phase 11).

### Project Structure

```
org-chart-tool/
├── package.json                      # pnpm workspace root
├── pnpm-workspace.yaml
├── turbo.json
├── docker-compose.yml                # api + web + worker + postgres + redis
├── Dockerfile
├── .env.example
├── packages/
│   ├── types/                        # shared domain types (Employee, OrgNode, Scenario…)
│   │   └── src/index.ts
│   ├── db/                            # Prisma schema, migrations, ltree raw-query helpers
│   │   ├── prisma/schema.prisma
│   │   ├── prisma/migrations/
│   │   └── src/{client.ts,ltree.ts,seed.ts}
│   └── config/                       # zod-validated env + tenant settings schemas
│       └── src/index.ts
├── apps/
│   ├── api/                          # NestJS REST + GraphQL + SCIM + webhooks
│   │   └── src/
│   │       ├── main.ts
│   │       ├── app.module.ts
│   │       ├── common/               # tenant guard, RBAC guard, audit interceptor, privacy filter
│   │       ├── auth/                 # OIDC/SAML/local, sessions, API keys
│   │       ├── employees/            # directory + profile CRUD
│   │       ├── org/                  # hierarchy, chart layout endpoints
│   │       ├── departments/
│   │       ├── positions/
│   │       ├── custom-fields/
│   │       ├── analytics/            # span of control, attrition, DEI
│   │       ├── planning/             # scenarios, succession
│   │       ├── hris/                 # connection config, sync orchestration
│   │       ├── imports/              # CSV import
│   │       ├── exports/              # PNG/PDF/SVG/vCard
│   │       ├── ai/                   # NL search, reorg assistant, narratives, anomalies
│   │       ├── scim/                 # SCIM 2.0 /Users /Groups
│   │       ├── webhooks/             # outbound delivery + subscription mgmt
│   │       ├── public-api/           # versioned REST, OpenAPI doc
│   │       └── integrations/         # Slack, Teams
│   ├── worker/                       # BullMQ processors (sync, webhook delivery, exports, AI jobs)
│   │   └── src/processors/
│   ├── web/                          # Next.js frontend
│   │   └── src/app/
│   │       ├── (chart)/              # interactive org chart views
│   │       ├── (directory)/          # search + profiles
│   │       ├── (analytics)/
│   │       ├── (planning)/
│   │       └── (admin)/              # tenant, RBAC, HRIS, custom fields
│   └── mcp/                          # MCP server exposing org data to AI agents
│       └── src/server.ts
└── tests/
    ├── fixtures/                     # sample CSVs, Merge payloads, SCIM requests
    └── e2e/                          # Playwright specs
```

---

## Phase 1: Foundation & Data Layer

### Purpose
Establish the monorepo, the PostgreSQL schema (hybrid relational + JSONB with `ltree`), shared types, and the dev/CI tooling. After this phase the database can be migrated and seeded, and every later phase builds on a stable, type-safe data layer. Nothing user-facing ships yet, but the entire domain model exists.

### Tasks

#### 1.1 — Monorepo & tooling scaffold

**What**: Stand up the pnpm/Turborepo workspace with `apps/{api,web,worker,mcp}` and `packages/{types,db,config}`, plus ESLint/Prettier/tsc-strict and Vitest.

**Design**:
- `pnpm-workspace.yaml` lists `apps/*` and `packages/*`.
- `turbo.json` pipelines: `build`, `lint`, `test`, `typecheck` with dependency-aware caching.
- Root `tsconfig.base.json` (`strict: true`, `noUncheckedIndexedAccess: true`, `target: ES2022`, `moduleResolution: bundler`).
- `packages/config` exports a zod-validated env loader:
```ts
export const env = z.object({
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url(),
  NODE_ENV: z.enum(['development','test','production']).default('development'),
  SESSION_SECRET: z.string().min(32),
  AI_PROVIDER: z.enum(['anthropic','openai']).default('anthropic'),
  AI_API_KEY: z.string().optional(),
  MERGE_API_KEY: z.string().optional(),
  ENCRYPTION_KEY: z.string().length(64), // hex, AES-256 for HRIS credentials
}).parse(process.env);
```

**Testing**:
- `Unit: env loader with all required vars → parsed object with defaults applied.`
- `Unit: env loader missing DATABASE_URL → ZodError naming DATABASE_URL.`
- `Unit: turbo build runs each workspace build with empty stub → exit 0.`

#### 1.2 — PostgreSQL schema (Prisma + raw SQL extensions)

**What**: Author the full hybrid schema from Suggestion 3, enable `uuid-ossp`, `ltree`, `pgcrypto`, `pgvector`.

**Design**:
- Prisma `schema.prisma` models all core tables: `tenants`, `locations`, `departments`, `positions`, `payBands`, `employees`, `employeeDemographics`, `customFieldDefinitions`, `customFieldValues`, `compensationRecords`, `orgChangeLog`, `hrisConnections`, `hrisSyncLog`, `planningScenarios`, `scenarioChanges`, `successionPlans`, `successionCandidates`, `users`, `apiKeys`, `webhooks`.
- Hybrid columns (JSONB) where Suggestion 3 mandates: `employees.custom_fields JSONB`, `employees.raw_hris JSONB`, `hris_connections.field_mapping JSONB`, `scenario_changes.payload JSONB`, `tenants.settings JSONB`.
- Prisma cannot express `ltree`/GIST; add a follow-up raw-SQL migration:
```sql
ALTER TABLE departments ADD COLUMN path LTREE;
ALTER TABLE employees   ADD COLUMN org_path LTREE;
CREATE INDEX idx_departments_path ON departments USING GIST (path);
CREATE INDEX idx_employees_org_path ON employees USING GIST (org_path);
ALTER TABLE employees ADD COLUMN embedding vector(1536);  -- pgvector, Phase 9
```
- Hierarchy lifecycle states for `planning_scenarios.status`: `draft → in_review → approved → archived`.
- `employees.status`: `active | inactive | on_leave | terminated`.

**Testing**:
- `Integration (real Postgres via testcontainers): migrate from empty → all tables present, extensions enabled.`
- `Integration: insert department tree, assert GIST index used for subtree query (EXPLAIN contains "Index Scan").`
- `Unit: Prisma client types compile against packages/types Employee.`

#### 1.3 — ltree maintenance & hierarchy helpers

**What**: Trigger-managed `org_path`/`path` materialisation plus a typed `packages/db/ltree.ts` query module.

**Design**:
- Postgres trigger `maintain_employee_org_path()`: on INSERT/UPDATE of `manager_id`, recompute `org_path = parent.org_path || id`; cascade to descendants.
- Cycle guard: trigger raises exception if `NEW.manager_id` is in `NEW`'s own subtree.
- `ltree.ts` exports:
```ts
function getSubtree(tenantId: string, rootEmployeeId: string): Promise<EmployeeNode[]>;
function getAncestors(tenantId: string, employeeId: string): Promise<EmployeeNode[]>;
function getDirectReports(tenantId: string, managerId: string): Promise<EmployeeNode[]>;
function rebuildOrgPaths(tenantId: string): Promise<{ updated: number }>; // recovery utility
```

**Testing**:
- `Unit (real db): insert 3-level chain → org_path values are 'a', 'a.b', 'a.b.c'.`
- `Unit: reassign mid-tree manager → descendant paths recomputed.`
- `Unit: set manager to own descendant → trigger raises cycle error.`
- `Unit: getSubtree(rootId) → returns all descendants excluding root.`

#### 1.4 — Seed & fixtures

**What**: Deterministic seed of one tenant with ~200 employees across departments/locations, plus committed fixtures for later phases.

**Design**: `seed.ts` builds a realistic 5-level hierarchy with span-of-control outliers (one manager with 18 reports) and demographic spread. Fixtures in `tests/fixtures/`: `employees.csv`, `merge_employees.json`, `scim_create_user.json`.

**Testing**:
- `Integration: run seed twice → idempotent (no duplicate-key errors, same row counts).`
- `Unit: seed produces at least one span-of-control outlier (≥15 direct reports) for analytics tests.`

---

## Phase 2: Tenancy, Auth, RBAC & Audit (Cross-Cutting Core)

### Purpose
Every byte of data is tenant-scoped and access-controlled HR data. This phase builds the security spine — tenant isolation, local auth + sessions, role-based access, field-level privacy, and the audit interceptor — before any feature reads or writes data. It directly addresses OWASP API Top 10 (BOLA) and GDPR field-level privacy.

### Tasks

#### 2.1 — Tenant scoping middleware & guard

**What**: Resolve `tenantId` per request and enforce that every query is scoped to it.

**Design**:
- Tenant resolved from subdomain (`acme.app`) or `X-Tenant-Slug` header → `tenants.slug`.
- NestJS `TenantGuard` populates `req.tenantId`; a Prisma extension injects `tenant_id` into every `where` clause and rejects models that lack a resolved tenant.
- All repository methods require `tenantId` as the first argument (type-enforced).

**Testing**:
- `Integration: request for tenant A querying tenant B's employee id → 404 (not 403, to avoid existence leak).`
- `Unit: Prisma extension auto-adds tenant_id to findMany where.`
- `Integration: missing/unknown tenant slug → 400.`

#### 2.2 — Local auth, sessions, password hashing

**What**: Email/password login with Argon2id, secure session cookies. (SSO added in Phase 8.)

**Design**:
- `POST /auth/login`, `POST /auth/logout`, `GET /auth/me`.
- Sessions stored in Redis; cookie `HttpOnly; Secure; SameSite=Lax`.
- `users.password_hash` nullable (SSO-only users have null).

**Testing**:
- `Unit: correct password → session created.`
- `Unit: wrong password → 401, generic message (no user-enumeration).`
- `Integration: /auth/me without session → 401.`

#### 2.3 — RBAC guard

**What**: Role-based authorisation with roles `super_admin | admin | hr_manager | manager | viewer`.

**Design**:
- `@Roles('hr_manager','admin')` decorator + `RolesGuard`.
- `manager` role additionally limited to own subtree for write operations (checked via `ltree` ancestry).
- Permission matrix documented in `apps/api/src/common/rbac.ts` as a const map.

**Testing**:
- `Unit: viewer POST /employees → 403.`
- `Unit: manager edits direct report → allowed; edits peer → 403.`
- `Unit: admin accesses any tenant employee within tenant → allowed.`

#### 2.4 — Field-level privacy filter

**What**: Strip fields the requester is not permitted to see, per `custom_field_definitions.visibility` and built-in sensitivity rules.

**Design**:
- Visibility levels: `hr_only | manager | company | self`.
- `PrivacyInterceptor` rewrites serialized employee objects based on requester role + relationship (self / manager-of / hr).
- `employee_demographics` and `compensation_records` are `hr_only` always (GDPR/ISO 27018).

**Testing**:
- `Unit: viewer reads employee with comp field → comp omitted.`
- `Unit: hr_manager reads demographics → present.`
- `Unit: employee reads own self-visible custom field → present; another's → omitted.`

#### 2.5 — Audit interceptor & org_change_log

**What**: Record every create/update/delete to `org_change_log` with old/new diff and source.

**Design**:
- Interceptor captures entity snapshot pre/post, computes `changed_fields`, writes `{entity_type, entity_id, change_type, old_values, new_values, changed_by, source}`.
- `source`: `manual | hris_sync | api | csv_import | scim`.

**Testing**:
- `Integration: PATCH employee title → one org_change_log row, changed_fields=['title'], correct old/new.`
- `Integration: HRIS-sourced update → source='hris_sync'.`
- `Unit: no-op update → no log row.`

---

## Phase 3: Org Hierarchy & Chart API (Core Value)

### Purpose
The heart of the product: serve the organisational hierarchy in a shape the frontend can render as an interactive chart, with the three view types and filtering. This is the capability that defines the product, so it ships early.

### Tasks

#### 3.1 — Departments, positions, locations, pay-bands CRUD

**What**: Tenant-scoped CRUD for the structural entities.

**Design**:
- REST resources under `/departments`, `/positions`, `/locations`, `/pay-bands`.
- `departments.parent_id` self-reference maintained via the ltree trigger (mirrors employee path logic).
- DTOs validated with zod; `positions.is_open` flag and `headcount`/`fte` for planning.

**Testing**:
- `Unit: create department with parent → path set under parent.`
- `Unit: delete department with employees → 409 (RESTRICT).`
- `Integration: create open position (is_open=true) → appears in open-roles query.`

#### 3.2 — Org tree endpoint

**What**: `GET /org/tree` returns the hierarchy as nested nodes for the chart.

**Design**:
```ts
interface OrgNode {
  employeeId: string | null;   // null for an open position node
  positionId: string | null;
  name: string;                // employee name or "Open: <title>"
  title: string;
  departmentId: string;
  locationId: string | null;
  photoUrl: string | null;
  isOpen: boolean;
  directReportCount: number;
  children: OrgNode[];
}
// GET /org/tree?rootEmployeeId=&depth=&includeOpen=true&department=&location=
```
- Built from a single recursive CTE / ltree subtree query, then assembled into the tree in memory.
- Open positions (positions with `is_open` and no employee) attached under their `reports_to` position's incumbent.

**Testing**:
- `Integration: GET /org/tree → root has children matching seeded structure.`
- `Integration: ?depth=1 → only direct reports returned.`
- `Integration: ?includeOpen=true → open positions appear as isOpen nodes.`
- `Unit: privacy filter applied to each node (viewer sees no comp).`

#### 3.3 — Chart layout views (hierarchical / matrix / flat)

**What**: `GET /org/chart?view=hierarchical|matrix|flat` returns layout-ready data.

**Design**:
- `hierarchical`: same as tree.
- `matrix`: nodes plus a `dottedLineTo: string[]` secondary-reporting list (from a `matrix_reports` JSONB on employee, or custom field).
- `flat`: paginated flat list with `managerName` for table rendering.
- Filtering by department/location/custom field applied before layout.

**Testing**:
- `Unit: view=flat → flat array, no children nesting.`
- `Unit: view=matrix → node includes dottedLineTo for employee with secondary reporting.`
- `Integration: filter ?department=Engineering → only Engineering subtree nodes.`

---

## Phase 4: Frontend — Interactive Chart & Shell

### Purpose
Make the hierarchy visible and navigable. Delivers the zoomable, searchable, filterable org chart plus the app shell (auth, tenant context, navigation). This is the first end-user-visible increment. Depends on Phases 2–3.

### Tasks

#### 4.1 — App shell, auth UI, tenant context

**What**: Next.js layout with login, session-aware nav, tenant theming.

**Design**:
- App Router layout fetches `/auth/me` server-side; redirects to `/login` if unauthenticated.
- Nav sections gated by role (admin links hidden from viewers).
- React Query for client data fetching with the session cookie.

**Testing**:
- `E2E (Playwright): unauthenticated visit / → redirected to /login.`
- `E2E: login as hr_manager → analytics nav visible; as viewer → hidden.`

#### 4.2 — Interactive org chart component

**What**: D3-powered zoomable/pannable chart consuming `/org/tree`.

**Design**:
- `react-d3-tree` with custom node cards (photo, name, title, direct-report badge).
- Click node → expand/collapse subtree (lazy-load deeper levels via `?rootEmployeeId&depth=1`).
- Zoom-to-fit, pan, and "centre on me" controls.
- Open-position nodes rendered with a dashed border.

**Testing**:
- `E2E: chart renders root and first level; clicking a node loads its reports.`
- `E2E: zoom controls change transform scale.`
- `Component test: open position node has dashed style + "Open" label.`

#### 4.3 — Search, filters, view switcher

**What**: Top-bar search and department/location filters; toggle hierarchical/matrix/flat.

**Design**:
- Typeahead search hits `/employees?q=` (Phase 5) and centres the chart on the chosen person.
- Filter chips re-request `/org/chart?view=&department=&location=`.

**Testing**:
- `E2E: search "EMEA design" → chart centres on matched employee.`
- `E2E: switch to flat view → table renders.`
- `E2E: apply Engineering filter → non-Engineering nodes removed.`

---

## Phase 5: Employee Directory & Profiles

### Purpose
The daily-use surface for all employees: a searchable directory with rich, privacy-filtered profiles, exportable as standards-compliant vCard/jCard and Schema.org JSON-LD. Depends on Phases 2–3; can be built in parallel with Phase 4.

### Tasks

#### 5.1 — Directory search & list

**What**: `GET /employees?q=&department=&location=&status=&page=` with filtering and pagination.

**Design**:
- Full-text search over name/title/email using Postgres `tsvector` (semantic search added in Phase 9).
- Returns privacy-filtered profile cards.

**Testing**:
- `Integration: q="jane" → matches first/preferred/last name.`
- `Integration: status=active default excludes terminated.`
- `Unit: pagination returns correct page/total.`

#### 5.2 — Employee profile CRUD + custom field values

**What**: `GET/POST/PATCH/DELETE /employees/:id` and custom-field value writes.

**Design**:
- Profile includes core fields, `custom_fields` JSONB resolved against definitions, manager, department, location.
- Writes pass through RBAC + audit + privacy filter.
- vCard 4.0 (`Accept: text/vcard`) and Schema.org Person JSON-LD (`Accept: application/ld+json`) representations:
```jsonld
{ "@context":"https://schema.org","@type":"Person",
  "name":"Jane Doe","jobTitle":"Design Lead","email":"jane@acme.com",
  "worksFor":{"@type":"Organization","name":"Acme"},
  "memberOf":{"@type":"Organization","name":"Design"} }
```

**Testing**:
- `Integration: GET profile as vCard → valid RFC 6350 with FN, TITLE, EMAIL, ORG.`
- `Integration: GET profile as JSON-LD → valid Schema.org Person.`
- `Unit: custom field with visibility=hr_only hidden from viewer.`

#### 5.3 — Custom field definitions admin

**What**: Tenant admins define custom fields per entity type with visibility and validation.

**Design**: CRUD on `custom_field_definitions`; `field_type ∈ {text,number,date,select,multi_select}`; `options[]` for selects; `visibility` enforced by the privacy filter.

**Testing**:
- `Unit: create select field without options → 400.`
- `Integration: new hr_only field → invisible to viewer, visible to hr_manager.`

#### 5.4 — Directory & profile UI

**What**: Next.js directory grid + profile pages.

**Design**: Card grid with filters; profile page with photo, contact, manager link, team, custom fields, and "report-line breadcrumb" using ancestors.

**Testing**:
- `E2E: open profile from directory → shows manager link that navigates up the chart.`
- `E2E: viewer cannot see HR-only field on a profile.`

---

## Phase 6: People Analytics & DEI

### Purpose
Turn the structured hierarchy into insight: span-of-control, attrition, headcount/open-role tracking, and DEI representation — the analytics depth that differentiates from chart-only competitors. Depends on Phases 1–3, 5.

### Tasks

#### 6.1 — Structural analytics

**What**: `GET /analytics/span-of-control`, `/analytics/headcount`, `/analytics/growth`.

**Design**:
- Span-of-control distribution (direct reports per manager, histogram + outliers > configurable threshold, default 10).
- Headcount by department/location/status; open-role count from open positions.
- Growth trend from `org_change_log` create/terminate events over time.

**Testing**:
- `Integration: span-of-control flags seeded 18-report manager as outlier.`
- `Integration: headcount excludes terminated.`
- `Unit: growth series buckets changes by month.`

#### 6.2 — Attrition analytics

**What**: `GET /analytics/attrition?from=&to=&groupBy=department|level`.

**Design**: Attrition rate = terminations in period / avg headcount, computed from `termination_date` and change log. Grouped breakdowns.

**Testing**:
- `Unit: attrition rate matches hand-computed value on fixture.`
- `Integration: groupBy=department returns per-department rates.`

#### 6.3 — DEI reporting (privacy-gated)

**What**: `GET /analytics/dei?dimension=gender|ethnicity|age_band&groupBy=department|level` — hr_only.

**Design**:
- Reads from `employee_demographics` joined to employees.
- **Small-cell suppression**: any group with count < `min_cell_size` (default 5) is suppressed/aggregated to "Other" to prevent re-identification (GDPR).
- Percentages computed within each grouping (per Suggestion 1 query pattern).

**Testing**:
- `Unit: group with 3 members suppressed below min_cell_size.`
- `Integration: viewer/manager → 403 on /analytics/dei.`
- `Integration: hr_manager → breakdown with percentages summing to ~100 per group.`

#### 6.4 — Analytics dashboard UI

**What**: Charts for the above (bar/line/distribution).

**Testing**:
- `E2E: hr_manager sees DEI dashboard; suppressed cells labelled "<5 — suppressed".`
- `E2E: span-of-control chart highlights outliers.`

---

## Phase 7: Public REST API, Webhooks & Export

### Purpose
Open the platform to developers and integrators: a versioned, OpenAPI-documented REST API, API-key auth, outbound webhooks, and chart/data export. Enables Zapier/n8n automation and presentation workflows. Depends on Phases 2–6.

### Tasks

#### 7.1 — Versioned public API + OpenAPI 3.1

**What**: `/api/v1/*` read/write endpoints with API-key auth and auto-generated OpenAPI spec at `/api/v1/openapi.json` and Swagger UI.

**Design**:
- API keys hashed (`api_keys.key_hash`), scoped (`read|write`), checked by `ApiKeyGuard`; subject to the same tenant + RBAC + privacy layers.
- NestJS Swagger decorators generate OpenAPI 3.1; schema names match `packages/types`.

**Testing**:
- `Integration: read-scoped key POST → 403.`
- `Integration: /openapi.json validates against OpenAPI 3.1 meta-schema.`
- `Integration: API call returns privacy-filtered data for the key's role.`

#### 7.2 — Outbound webhooks

**What**: Subscriptions to events; reliable delivery with HMAC signing and retries.

**Design**:
- Events: `employee.created|updated|terminated`, `department.changed`, `scenario.approved`, `sync.completed`.
- On a relevant `org_change_log` write, enqueue a BullMQ delivery job per matching `webhooks` row.
- Payload signed with `X-Signature: sha256=HMAC(secret, body)`; retries with exponential backoff; dead-letter after N attempts.

**Testing**:
- `Integration (mock receiver): employee update → POST with valid HMAC signature.`
- `Integration: receiver returns 500 → job retried; after max attempts → dead-lettered.`
- `Unit: subscription filters events not in its events[].`

#### 7.3 — Export (PNG / PDF / SVG / CSV / vCard)

**What**: `GET /exports/chart?format=png|pdf|svg` and `GET /exports/directory?format=csv|vcard`.

**Design**:
- Chart export rendered by a worker via Puppeteer against a headless chart route; large charts paginated for PDF.
- SVG produced by serialising the D3 layout server-side.
- Directory CSV/vCard streamed, privacy-filtered to the requester's role.

**Testing**:
- `Integration: format=png → image/png, non-empty buffer.`
- `Integration: format=svg → well-formed SVG containing node labels.`
- `Integration: directory vcard export → concatenated valid vCards.`

---

## Phase 8: HRIS Sync, SSO & SCIM Provisioning

### Purpose
The headline differentiator: keep data live without manual re-import, and integrate with enterprise identity. Covers Merge.dev HRIS sync + CSV import, SAML/OIDC SSO, and a SCIM 2.0 server. Depends on Phases 1–2, 5.

### Tasks

#### 8.1 — CSV import

**What**: `POST /imports/csv` mapping uploaded columns to employee fields, with validation and a dry-run preview.

**Design**:
- Column mapping UI/config; dry-run returns create/update/error counts before commit.
- Manager linking resolved by email/employee_number; deferred second pass so order-independent.
- Writes tagged `source='csv_import'` in audit log.

**Testing**:
- `Integration: import employees.csv fixture → correct create count, hierarchy linked.`
- `Unit: row with unknown manager email → flagged error in dry-run, not committed.`
- `Integration: re-import with changed titles → updates, not duplicates (matched on employee_number).`

#### 8.2 — Merge.dev unified HRIS sync

**What**: Connect an HRIS via Merge, normalise into our schema, and sync on a schedule.

**Design**:
- `hris_connections` stores encrypted Merge account token (AES-256 via `ENCRYPTION_KEY`).
- BullMQ repeatable job per connection (default 15 min); fetches Merge `Employee/Department/Team` objects, maps via `field_mapping` JSONB, stores raw payload in `employees.raw_hris`.
- Diff engine: upsert by `external_id`; mark missing-as-terminated (configurable); record per-record outcomes in `hris_sync_log`.
- `manager_id` resolved from Merge manager reference; ltree paths recomputed after batch.

**Testing**:
- `Integration (mock Merge API): sync merge_employees.json → employees created, manager chain set.`
- `Integration: second sync with one departure → employee status=terminated, sync log records it.`
- `Unit: field_mapping remaps Merge "work_email" → email.`
- `Integration: sync failure mid-batch → hris_sync_log status=partial, records_failed counted.`

#### 8.3 — SSO (OIDC + SAML 2.0)

**What**: Tenant-configurable SSO login.

**Design**:
- `tenants.sso_provider` + `sso_config` JSONB (issuer, client id/secret or SAML cert/entrypoint).
- OIDC via `openid-client`; SAML via `@node-saml/node-saml`. On callback, match/JIT-provision `users` by email, create session.

**Testing**:
- `Integration (mock IdP): OIDC callback with valid token → session created, user JIT-provisioned.`
- `Integration: SAML assertion with bad signature → 401.`

#### 8.4 — SCIM 2.0 server

**What**: RFC 7643/7644 `/scim/v2/Users` and `/Groups` for IdP-driven provisioning.

**Design**:
- Bearer-token auth (per-tenant SCIM token).
- `User` maps to `users` + `employees`; `EnterpriseUser.manager` → `employees.manager_id` (reporting line, per RFC 7643 §4.3).
- Supports `POST/GET/PUT/PATCH/DELETE`, filtering (`filter=userName eq ...`), and pagination per spec; `DELETE`/`active:false` → deactivate.
- Writes tagged `source='scim'`.

**Testing**:
- `Integration: POST scim_create_user.json fixture → 201 with SCIM User schema response.`
- `Integration: PATCH active:false → user deactivated, employee status updated.`
- `Integration: manager attribute set → employees.manager_id linked, org_path updated.`
- `Pact: SCIM responses conform to RFC 7643 schema.`

---

## Phase 9: AI-Native Search & Profile Enrichment

### Purpose
Deliver the AI differentiators that move the tool beyond static rendering: natural-language people search and optional profile enrichment. Depends on Phases 3, 5, 6.

### Tasks

#### 9.1 — Natural-language people search

**What**: `POST /ai/search { query }` answering questions like "Who leads design in EMEA?"

**Design**:
- Hybrid retrieval: pgvector semantic match over employee embeddings (title + department + location + skills) + structured filters extracted by the LLM (department/location/role intents) via tool-calling.
- LLM returns a ranked answer with the matched employees; never invents people — answers grounded in retrieved rows only.
- Embeddings backfilled by a worker job on employee create/update; cached.
- Results pass through the privacy filter for the requesting user.

**Testing**:
- `Integration (mock LLM + real db): "who leads design in EMEA" → returns EMEA design lead from seed.`
- `Unit: extracted filters from query map to {department:'Design',location:'EMEA'}.`
- `Integration: query referencing hr_only data as a viewer → answer excludes restricted fields.`
- `Unit: no match → graceful "no one found" (no hallucinated person).`

#### 9.2 — AI profile enrichment (opt-in, privacy-gated)

**What**: Suggest field values (e.g., skills) from provided sources; admin-approved before save.

**Design**:
- Enrichment is suggestion-only into a staging area; never auto-writes PII. Tenant setting gates it; GDPR note: only processes data the tenant supplies.

**Testing**:
- `Unit: enrichment disabled by tenant setting → endpoint 403.`
- `Integration: suggestions returned to staging, not written until approved.`

---

## Phase 10: Headcount Planning, Scenarios & Succession

### Purpose
Planning capability that, with live sync, defines the affordable-ChartHop-alternative positioning: draft future-state structures, compute budget impact, compare scenarios, and map succession bench strength. Depends on Phases 1–3, 6.

### Tasks

#### 10.1 — Scenario modelling

**What**: Create scenarios branching from a base snapshot; add proposed changes; compute budget impact.

**Design**:
- `planning_scenarios` + `scenario_changes` (change_type: `new_hire|termination|promotion|transfer|reorg|role_change`; heterogeneous `payload` JSONB per Suggestion 3).
- Scenario applied non-destructively over the current org to produce a projected tree (no writes to live employees).
- `budget_impact` summed from proposed salaries/pay bands vs current.
- Lifecycle: `draft → in_review → approved → archived`; approval can emit `scenario.approved` webhook.

**Testing**:
- `Unit: scenario with one new_hire → projected tree includes new node, budget_impact = salary.`
- `Integration: scenario does not mutate live employees table.`
- `Unit: promotion change → projected pay band updated, delta in budget_impact.`

#### 10.2 — Scenario comparison & projected chart

**What**: `GET /planning/scenarios/:id/projection` returns the projected `OrgNode` tree; UI before/after diff.

**Design**: Diff highlights added/removed/moved nodes vs base.

**Testing**:
- `Integration: projection endpoint returns tree reflecting all scenario changes.`
- `E2E: before/after view marks new hires green, terminations red.`

#### 10.3 — Succession planning

**What**: Map candidates to key positions with readiness/risk.

**Design**: `succession_plans` (risk_level) + `succession_candidates` (readiness: `ready_now|ready_1yr|developing|not_ready`, priority). Bench-strength view per position.

**Testing**:
- `Integration: add candidate to plan → appears in bench view ordered by priority.`
- `Unit: position with no ready_now candidate flagged "at risk".`

---

## Phase 11: Integrations — Slack, Teams & MCP Server

### Purpose
Meet users where they work and open org data to AI agents — the latter being a clear market gap. Depends on Phases 3, 5, 9.

### Tasks

#### 11.1 — Slack directory lookup

**What**: Slack app with `/who` slash command and app-home directory search.

**Design**:
- Slack Bolt app; `/who is the design lead in EMEA` proxies to `/ai/search`; results as Block Kit cards (photo, title, manager, contact).
- Per-tenant Slack install via OAuth; privacy filter applied to the looking-up user's mapped role.

**Testing**:
- `Integration (mock Slack): /who command → Block Kit response with matched employee.`
- `Unit: unmapped Slack user → limited viewer-level visibility.`

#### 11.2 — Microsoft Teams lookup

**What**: Teams messaging-extension equivalent of 11.1.

**Testing**:
- `Integration (mock Bot Framework): query → adaptive card with employee.`

#### 11.3 — MCP server

**What**: An MCP server exposing org data as resources + tools for AI agents (Claude, etc.).

**Design**:
- Resources: `employee/{id}` profile (privacy-filtered to the connected principal), `org/tree`.
- Tools: `search_people(query)`, `get_direct_reports(employeeId)`, `get_manager_chain(employeeId)`, `run_analytics(metric, params)`.
- Auth via per-tenant API key mapped to a role; every tool result passes the privacy filter — no MCP path bypasses tenant/RBAC/privacy.

**Testing**:
- `Integration: MCP search_people tool → grounded results, privacy-filtered.`
- `Integration: get_manager_chain → ordered ancestors from ltree.`
- `Unit: MCP key for tenant A cannot read tenant B resource → error.`

---

## Phase 12: AI Reorg Assistant, Anomaly Detection & Narratives

### Purpose
The advanced AI layer: structural recommendations, anomaly flags, and auto-generated narratives — the highest-value differentiators, built last because they depend on analytics, planning, and AI search. Depends on Phases 6, 9, 10.

### Tasks

#### 12.1 — AI reorg assistant

**What**: `POST /ai/reorg/suggest { scenarioId? }` proposing span-of-control fixes and bottleneck reductions within constraints.

**Design**:
- Feeds span-of-control outliers, vacant criticals, and budget constraints (from analytics + scenario) to the LLM with tool access to read structure.
- Output is a set of proposed `scenario_changes` (suggestion-only) with rationale; user reviews before applying.

**Testing**:
- `Integration (mock LLM): outlier manager (18 reports) → suggestion to split team, emitted as draft scenario changes.`
- `Unit: suggestions never auto-write to live org (staged only).`

#### 12.2 — Anomaly detection

**What**: Scheduled detection of attrition spikes, duplicate roles, orphaned employees, and broken reporting lines.

**Design**:
- Worker job over `org_change_log` + current state: z-score on attrition by department; duplicate detection on near-identical title+department; employees with null/cyclic manager.
- Flags written to a `anomalies` view/table surfaced in admin UI.

**Testing**:
- `Unit: department with attrition > mean+2σ flagged.`
- `Integration: employee with missing manager flagged as orphaned.`
- `Unit: two identical-title same-dept positions flagged duplicate.`

#### 12.3 — Auto-generated org narratives

**What**: `POST /ai/narrative { scope }` producing board/all-hands summaries.

**Design**: LLM summarises analytics + recent changes into prose ("Engineering grew 12% this quarter; span of control improved from 9.2 to 7.8 average"). Numbers are computed first and passed in — the LLM only narrates, preventing fabricated figures.

**Testing**:
- `Integration (mock LLM): narrative includes the exact headcount figures supplied (no invented numbers).`
- `Unit: narrative request as viewer for hr_only scope → 403.`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Data Layer          ─── required by everything
    │
Phase 2: Tenancy / Auth / RBAC / Audit    ─── requires 1; cross-cutting spine
    │
Phase 3: Org Hierarchy & Chart API (CORE) ─── requires 1, 2
    ├── Phase 4: Frontend Chart & Shell    ─── requires 2, 3   ┐ parallel
    └── Phase 5: Directory & Profiles      ─── requires 2, 3   ┘
            │
Phase 6: Analytics & DEI                  ─── requires 1-3, 5
            │
            ├── Phase 7: Public API / Webhooks / Export ─── requires 2-6
            ├── Phase 8: HRIS Sync / SSO / SCIM         ─── requires 1, 2, 5 (parallel w/ 7)
            └── Phase 9: AI Search & Enrichment         ─── requires 3, 5, 6 (parallel w/ 7, 8)
                    │
                    ├── Phase 10: Planning / Scenarios / Succession ─── requires 1-3, 6
                    └── Phase 11: Slack / Teams / MCP               ─── requires 3, 5, 9
                            │
Phase 12: AI Reorg / Anomalies / Narratives ─── requires 6, 9, 10
```

**Parallelism opportunities:**
- Phases **4 and 5** can be developed concurrently once Phase 3 lands.
- Phases **7, 8, and 9** can proceed in parallel after Phase 6 (8 needs only 1, 2, 5 and may even start alongside 6).
- Phases **10 and 11** can run in parallel after Phase 9.

---

## Definition of Done (per phase)

Every phase must satisfy all of the following before it is considered complete:

1. All tasks in the phase implemented.
2. All unit and integration tests pass (`pnpm test` green across affected workspaces).
3. ESLint and Prettier pass with no warnings; `tsc --noEmit` (strict) passes.
4. New/changed endpoints have request/response DTOs validated by zod and appear in the auto-generated OpenAPI 3.1 spec (where public).
5. Prisma migration(s) created, named, and reversible; `ltree`/raw-SQL migrations included where applicable.
6. Tenant scoping, RBAC, and field-level privacy enforced on every new data path (OWASP BOLA check — no endpoint returns cross-tenant or over-privileged data).
7. Any write path emits an `org_change_log` entry with the correct `source`.
8. New config/env vars added to `.env.example` and the zod env schema.
9. `docker compose up` builds and runs the affected services successfully.
10. New user-facing capability verified end-to-end (Playwright spec for UI phases; integration test for API/worker phases).
11. AI features: outputs are grounded in retrieved/computed data (no hallucinated people or figures) and pass the privacy filter.
```
