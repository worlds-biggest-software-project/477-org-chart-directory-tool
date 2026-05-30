# Data Model Suggestion 3: Hybrid Relational + JSONB Model (PostgreSQL)

## Approach

A pragmatic hybrid schema that keeps core entities and relationships in properly normalised relational tables while using PostgreSQL JSONB columns for flexible, schema-less data: custom employee fields, HRIS-specific raw payloads, integration configuration, scenario change proposals, and variable DEI dimensions. This gives strong referential integrity where it matters and schema flexibility where rigidity would slow development or limit tenant customisation.

## Why This Suits the Domain

Org chart tools face a tension: the core hierarchy (employees, managers, departments, positions) is highly structured and relational, but the surrounding data is varied and unpredictable. Different HRIS systems expose different fields. Each tenant wants different custom profile fields. DEI dimensions vary by jurisdiction. Scenario planning proposals contain heterogeneous change sets. A pure normalised schema forces EAV tables or constant migrations; a pure document model sacrifices referential integrity. The hybrid approach resolves this by using relational columns for stable, frequently-queried data and JSONB for everything that varies by tenant, integration, or over time.

---

## Schema Definition

```sql
-- ============================================================
-- Extensions
-- ============================================================
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "ltree";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";    -- trigram indexes for fuzzy search

-- ============================================================
-- Tenants
-- ============================================================
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    -- JSONB: branding, feature flags, locale, default privacy settings
    settings        JSONB NOT NULL DEFAULT '{
        "features": {
            "dei_analytics": true,
            "headcount_planning": true,
            "ai_search": false,
            "succession_planning": false
        },
        "privacy": {
            "default_field_visibility": "company",
            "dei_data_visible_to": ["hr_admin"]
        },
        "branding": {
            "primary_color": "#1a73e8",
            "logo_url": null
        }
    }',
    -- JSONB: SSO/SAML/OIDC configuration (varies by provider)
    sso_config      JSONB,
    -- JSONB: custom field schema definitions for this tenant
    custom_field_schemas JSONB NOT NULL DEFAULT '{
        "employee": [],
        "department": [],
        "position": []
    }',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- Locations
-- ============================================================
CREATE TABLE locations (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    country_code    CHAR(2),
    timezone        VARCHAR(50),
    is_remote       BOOLEAN NOT NULL DEFAULT FALSE,
    -- JSONB: full address, coordinates, office amenities, capacity
    details         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

-- GIN index for searching within location details
CREATE INDEX idx_locations_details ON locations USING GIN (details);

-- ============================================================
-- Departments
-- ============================================================
CREATE TABLE departments (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(50),
    parent_id       UUID REFERENCES departments(id) ON DELETE SET NULL,
    path            LTREE,
    cost_centre     VARCHAR(50),
    -- JSONB: department-level custom fields, budget info, goals
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

CREATE INDEX idx_dept_path ON departments USING GIST (path);
CREATE INDEX idx_dept_parent ON departments (parent_id);
CREATE INDEX idx_dept_tenant ON departments (tenant_id);
CREATE INDEX idx_dept_metadata ON departments USING GIN (metadata);

-- ============================================================
-- Pay Bands
-- ============================================================
CREATE TABLE pay_bands (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(100) NOT NULL,
    level_order     INTEGER NOT NULL,
    -- JSONB: salary ranges by currency/region, bonus targets, equity bands
    compensation_ranges JSONB NOT NULL DEFAULT '{}',
    /*  Example:
        {
            "USD": { "min": 120000, "max": 180000, "bonus_target_pct": 15 },
            "GBP": { "min": 95000, "max": 140000, "bonus_target_pct": 15 },
            "equity_band": { "min_shares": 1000, "max_shares": 5000 }
        }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE INDEX idx_pb_comp ON pay_bands USING GIN (compensation_ranges);

-- ============================================================
-- Positions
-- ============================================================
CREATE TABLE positions (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    title           VARCHAR(255) NOT NULL,
    department_id   UUID NOT NULL REFERENCES departments(id) ON DELETE RESTRICT,
    location_id     UUID REFERENCES locations(id) ON DELETE SET NULL,
    pay_band_id     UUID REFERENCES pay_bands(id) ON DELETE SET NULL,
    reports_to_id   UUID REFERENCES positions(id) ON DELETE SET NULL,
    is_open         BOOLEAN NOT NULL DEFAULT FALSE,
    headcount       INTEGER NOT NULL DEFAULT 1,
    fte             NUMERIC(3, 2) NOT NULL DEFAULT 1.00,
    -- JSONB: job description, requirements, skills, competencies
    job_details     JSONB NOT NULL DEFAULT '{}',
    /*  Example:
        {
            "description": "...",
            "requirements": ["5+ years experience", "..."],
            "skills": ["python", "leadership", "data-analysis"],
            "competencies": { "technical": 4, "leadership": 3 }
        }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pos_dept ON positions (department_id);
CREATE INDEX idx_pos_reports ON positions (reports_to_id);
CREATE INDEX idx_pos_tenant ON positions (tenant_id);
CREATE INDEX idx_pos_job_details ON positions USING GIN (job_details);

-- ============================================================
-- Employees
-- ============================================================
CREATE TABLE employees (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,

    -- Core relational fields (always queried, always present)
    email           VARCHAR(255) NOT NULL,
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    employee_number VARCHAR(50),
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    employment_type VARCHAR(20) NOT NULL DEFAULT 'full_time',

    -- Foreign keys for hierarchy and structure
    position_id     UUID REFERENCES positions(id) ON DELETE SET NULL,
    manager_id      UUID REFERENCES employees(id) ON DELETE SET NULL,
    department_id   UUID NOT NULL REFERENCES departments(id) ON DELETE RESTRICT,
    location_id     UUID REFERENCES locations(id) ON DELETE SET NULL,
    pay_band_id     UUID REFERENCES pay_bands(id) ON DELETE SET NULL,
    org_path        LTREE,

    -- Dates
    hire_date       DATE,
    termination_date DATE,

    -- JSONB: profile data that varies by tenant and HRIS source
    profile         JSONB NOT NULL DEFAULT '{}',
    /*  Example:
        {
            "preferred_name": "Phil",
            "pronouns": "he/him",
            "phone": "+1-555-0123",
            "photo_url": "https://...",
            "bio": "...",
            "social": {
                "linkedin": "https://linkedin.com/in/...",
                "slack_id": "U12345"
            },
            "emergency_contact": {
                "name": "Jane Doe",
                "phone": "+1-555-0456",
                "relationship": "spouse"
            }
        }
    */

    -- JSONB: tenant-defined custom fields
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    /*  Example:
        {
            "shirt_size": "L",
            "dietary_requirements": "vegetarian",
            "desk_number": "3F-42",
            "cost_code": "ENG-042"
        }
    */

    -- JSONB: raw HRIS data for debugging sync issues
    hris_raw        JSONB,
    hris_external_id VARCHAR(255),
    hris_last_sync  TIMESTAMPTZ,

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email),
    UNIQUE (tenant_id, employee_number)
);

-- Relational indexes
CREATE INDEX idx_emp_manager ON employees (manager_id);
CREATE INDEX idx_emp_dept ON employees (department_id);
CREATE INDEX idx_emp_tenant ON employees (tenant_id);
CREATE INDEX idx_emp_status ON employees (tenant_id, status);
CREATE INDEX idx_emp_org_path ON employees USING GIST (org_path);
CREATE INDEX idx_emp_hris ON employees (tenant_id, hris_external_id);

-- JSONB indexes for profile search
CREATE INDEX idx_emp_profile ON employees USING GIN (profile);
CREATE INDEX idx_emp_custom ON employees USING GIN (custom_fields);

-- Full-text search across name + profile
CREATE INDEX idx_emp_fts ON employees USING GIN (
    to_tsvector('english',
        first_name || ' ' || last_name || ' ' ||
        COALESCE(profile->>'preferred_name', '') || ' ' ||
        COALESCE(profile->>'bio', '')
    )
);

-- Trigram index for fuzzy name matching
CREATE INDEX idx_emp_name_trgm ON employees USING GIN (
    (first_name || ' ' || last_name) gin_trgm_ops
);

-- ============================================================
-- Employee Demographics (separate table for access control)
-- ============================================================
CREATE TABLE employee_demographics (
    employee_id     UUID PRIMARY KEY REFERENCES employees(id) ON DELETE CASCADE,
    -- JSONB: allows dimensions to vary by jurisdiction / tenant policy
    demographics    JSONB NOT NULL DEFAULT '{}',
    /*  Example:
        {
            "gender": "female",
            "gender_identity": "cisgender",
            "ethnicity": "Hispanic/Latino",
            "race": ["White"],
            "age_band": "35-44",
            "disability_status": "no_disability",
            "veteran_status": "non_veteran",
            "lgbtq_identity": "prefer_not_to_say"
        }
    */
    voluntary_self_id BOOLEAN NOT NULL DEFAULT FALSE,
    collected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_demo_data ON employee_demographics USING GIN (demographics);

-- ============================================================
-- Compensation History
-- ============================================================
CREATE TABLE compensation_records (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    employee_id     UUID NOT NULL REFERENCES employees(id) ON DELETE CASCADE,
    effective_date  DATE NOT NULL,
    -- JSONB: flexible comp structure (base, bonus, equity, allowances vary)
    compensation    JSONB NOT NULL,
    /*  Example:
        {
            "base_salary": 150000,
            "currency": "USD",
            "bonus_target_pct": 15,
            "equity_shares": 2000,
            "signing_bonus": 10000,
            "allowances": {
                "housing": 2000,
                "transport": 500
            }
        }
    */
    pay_band_id     UUID REFERENCES pay_bands(id),
    change_reason   VARCHAR(50),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (employee_id, effective_date)
);

CREATE INDEX idx_comp_emp ON compensation_records (employee_id, effective_date DESC);
CREATE INDEX idx_comp_data ON compensation_records USING GIN (compensation);

-- ============================================================
-- HRIS Connections
-- ============================================================
CREATE TABLE hris_connections (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    provider        VARCHAR(50) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    sync_frequency  INTERVAL NOT NULL DEFAULT '15 minutes',
    last_sync_at    TIMESTAMPTZ,
    last_sync_status VARCHAR(20),
    -- JSONB: credentials (encrypted at app layer), endpoints, auth config
    connection_config JSONB NOT NULL DEFAULT '{}',
    -- JSONB: field mapping from HRIS fields to local fields + custom_fields
    field_mapping   JSONB NOT NULL DEFAULT '{}',
    /*  Example:
        {
            "mappings": [
                { "source": "work_email", "target": "email", "target_type": "column" },
                { "source": "given_name", "target": "first_name", "target_type": "column" },
                { "source": "badge_number", "target": "badge_number", "target_type": "custom_field" },
                { "source": "department.name", "target": "department_id", "target_type": "lookup", "lookup_field": "name" }
            ],
            "conflict_resolution": "remote_wins",
            "ignore_fields": ["personal_email"]
        }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, provider)
);

-- ============================================================
-- Org Change Log (audit trail)
-- ============================================================
CREATE TABLE org_change_log (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    entity_type     VARCHAR(30) NOT NULL,
    entity_id       UUID NOT NULL,
    change_type     VARCHAR(20) NOT NULL,
    -- JSONB: flexible diff storage
    changes         JSONB NOT NULL,
    /*  Example:
        {
            "department_id": { "old": "uuid-1", "new": "uuid-2" },
            "manager_id": { "old": "uuid-3", "new": "uuid-4" },
            "custom_fields.desk_number": { "old": "3F-42", "new": "4A-01" }
        }
    */
    source          VARCHAR(30) NOT NULL DEFAULT 'manual',
    changed_by      UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ocl_entity ON org_change_log (entity_type, entity_id);
CREATE INDEX idx_ocl_tenant_time ON org_change_log (tenant_id, created_at DESC);
CREATE INDEX idx_ocl_changes ON org_change_log USING GIN (changes);

-- ============================================================
-- Planning Scenarios
-- ============================================================
CREATE TABLE planning_scenarios (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    base_snapshot_date DATE NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
    created_by      UUID NOT NULL,
    -- JSONB: scenario-level settings, comparison metrics, approval chain
    config          JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE scenario_changes (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    scenario_id     UUID NOT NULL REFERENCES planning_scenarios(id) ON DELETE CASCADE,
    change_type     VARCHAR(30) NOT NULL,
    -- JSONB: the entire proposed change as a flexible document
    proposal        JSONB NOT NULL,
    /*  Example (new hire):
        {
            "change_type": "new_hire",
            "position": { "title": "Senior Engineer", "department_id": "..." },
            "proposed_salary": 160000,
            "proposed_start_date": "2026-07-01",
            "budget_impact": {
                "annual_cost": 200000,
                "prorated_fy_cost": 100000,
                "includes": ["salary", "benefits", "equipment"]
            },
            "justification": "Replacing departed team lead"
        }
    */
    sort_order      INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sc_scenario ON scenario_changes (scenario_id);
CREATE INDEX idx_sc_proposal ON scenario_changes USING GIN (proposal);

-- ============================================================
-- Succession Planning
-- ============================================================
CREATE TABLE succession_plans (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    position_id     UUID NOT NULL REFERENCES positions(id) ON DELETE CASCADE,
    risk_level      VARCHAR(20) NOT NULL DEFAULT 'medium',
    -- JSONB: assessment criteria, development plans, timeline
    assessment      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE succession_candidates (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    plan_id         UUID NOT NULL REFERENCES succession_plans(id) ON DELETE CASCADE,
    employee_id     UUID NOT NULL REFERENCES employees(id) ON DELETE CASCADE,
    readiness       VARCHAR(20) NOT NULL DEFAULT 'developing',
    priority        INTEGER NOT NULL DEFAULT 1,
    -- JSONB: competency gaps, development actions, mentor assignments
    development_plan JSONB NOT NULL DEFAULT '{}',
    UNIQUE (plan_id, employee_id)
);

-- ============================================================
-- Users & API Access
-- ============================================================
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    employee_id     UUID REFERENCES employees(id) ON DELETE SET NULL,
    email           VARCHAR(255) NOT NULL,
    password_hash   TEXT,
    role            VARCHAR(30) NOT NULL DEFAULT 'viewer',
    scim_external_id VARCHAR(255),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    -- JSONB: user preferences, notification settings, saved views
    preferences     JSONB NOT NULL DEFAULT '{}',
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE TABLE api_keys (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(100) NOT NULL,
    key_hash        TEXT NOT NULL,
    scopes          TEXT[] NOT NULL DEFAULT '{read}',
    expires_at      TIMESTAMPTZ,
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE webhooks (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    url             TEXT NOT NULL,
    events          TEXT[] NOT NULL,
    secret_hash     TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    -- JSONB: delivery history, failure counts, retry config
    delivery_config JSONB NOT NULL DEFAULT '{"max_retries": 3, "timeout_ms": 5000}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Query Examples

```sql
-- Search employees by custom field value
SELECT id, first_name, last_name, custom_fields->>'desk_number' AS desk
FROM employees
WHERE tenant_id = :tenant_id
  AND custom_fields @> '{"cost_code": "ENG-042"}';

-- DEI analytics with flexible dimensions
SELECT
    d.name AS department,
    dem.demographics->>'gender' AS gender,
    dem.demographics->>'ethnicity' AS ethnicity,
    COUNT(*) AS headcount,
    AVG((cr.compensation->>'base_salary')::NUMERIC) AS avg_salary
FROM employees e
JOIN departments d ON e.department_id = d.id
JOIN employee_demographics dem ON e.id = dem.employee_id
LEFT JOIN LATERAL (
    SELECT compensation FROM compensation_records
    WHERE employee_id = e.id ORDER BY effective_date DESC LIMIT 1
) cr ON TRUE
WHERE e.tenant_id = :tenant_id AND e.status = 'active'
GROUP BY d.name, dem.demographics->>'gender', dem.demographics->>'ethnicity';

-- Fuzzy name search with trigram similarity
SELECT id, first_name, last_name, profile->>'preferred_name' AS preferred_name
FROM employees
WHERE tenant_id = :tenant_id
  AND (first_name || ' ' || last_name) % :search_term
ORDER BY similarity(first_name || ' ' || last_name, :search_term) DESC
LIMIT 20;

-- Scenario budget impact summary
SELECT
    s.name AS scenario,
    COUNT(*) AS total_changes,
    SUM((sc.proposal->'budget_impact'->>'annual_cost')::NUMERIC) AS total_annual_cost,
    SUM(CASE WHEN sc.change_type = 'new_hire' THEN 1 ELSE 0 END) AS new_hires,
    SUM(CASE WHEN sc.change_type = 'termination' THEN 1 ELSE 0 END) AS terminations
FROM planning_scenarios s
JOIN scenario_changes sc ON s.id = sc.scenario_id
WHERE s.tenant_id = :tenant_id AND s.status = 'draft'
GROUP BY s.id, s.name;
```

---

## JSONB Validation Strategy

Custom field schemas are defined per tenant and validated at the application layer:

```sql
-- The custom_field_schemas column on tenants defines the allowed schema:
/*
{
    "employee": [
        { "key": "shirt_size", "type": "select", "label": "Shirt Size",
          "options": ["XS","S","M","L","XL","XXL"], "required": false,
          "visibility": "self" },
        { "key": "desk_number", "type": "text", "label": "Desk Number",
          "required": false, "visibility": "company" },
        { "key": "cost_code", "type": "text", "label": "Cost Code",
          "required": true, "visibility": "hr_only" }
    ]
}
*/
-- Application layer validates employee.custom_fields against this schema
-- on every insert/update. CHECK constraints can enforce top-level type:

ALTER TABLE employees ADD CONSTRAINT chk_custom_fields_object
    CHECK (jsonb_typeof(custom_fields) = 'object');

ALTER TABLE employees ADD CONSTRAINT chk_profile_object
    CHECK (jsonb_typeof(profile) = 'object');
```

---

## Trade-offs

**Strengths:**
- Best of both worlds: relational integrity for hierarchy and structure; JSONB flexibility for profiles, custom fields, and integration data
- No EAV tables -- custom fields live in a single JSONB column with GIN indexes, making queries simpler and faster
- HRIS field mapping is configuration, not code -- new fields from a new HRIS can be mapped to `custom_fields` keys without schema migrations
- DEI demographics support arbitrary dimensions without schema changes when regulations or policies change
- GIN indexes on JSONB provide performant containment queries (`@>`) for filtering
- Compensation structure can accommodate global pay variations (allowances, equity, currency) without normalisation overhead
- Single PostgreSQL database keeps self-hosting simple

**Weaknesses:**
- JSONB columns lack database-level referential integrity -- validation moves to the application layer
- Complex JSONB queries can be harder to optimise than pure relational queries
- No database-enforced schema for custom fields (must rely on application-level validation or CHECK constraints)
- JSONB columns can accumulate inconsistent data if validation is not rigorous
- Reporting tools may struggle with deeply nested JSONB structures compared to flat relational columns

**Scalability:**
- Handles 10,000-100,000 employees on a single PostgreSQL instance
- GIN indexes add write overhead but keep read performance strong
- JSONB compression in PostgreSQL TOAST handles large profile documents efficiently
- Can partition `org_change_log` by date for time-series performance

**Migration path:**
- Natural evolution from Suggestion 1: add JSONB columns to existing relational tables incrementally
- Can extract frequently-queried JSONB fields into relational columns as usage patterns emerge (progressive normalisation)
- If event sourcing (Suggestion 2) is desired later, the `org_change_log` can be extended to a full event store
- Graph database projections (Suggestion 4) can be fed from this schema via change data capture
