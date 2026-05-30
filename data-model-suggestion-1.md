# Data Model Suggestion 1: Normalized Relational Model (PostgreSQL)

## Approach

A traditional third-normal-form (3NF+) relational schema using PostgreSQL. Core entities are fully normalized with proper foreign keys, indexes, and constraints. Hierarchical reporting relationships use an adjacency list with PostgreSQL's `ltree` extension for efficient tree traversal, combined with recursive CTEs for ad-hoc queries.

## Why This Suits the Domain

Org chart and directory tools have well-defined entities (employees, departments, positions, locations) with stable relationships. The reporting hierarchy is the central data structure, and relational databases handle this cleanly via self-referencing foreign keys enhanced by `ltree` materialized paths. HRIS sync, headcount planning, and DEI analytics all benefit from strong referential integrity, ACID transactions, and mature query optimisation. PostgreSQL is also the natural choice for a self-hostable open-source product.

---

## Schema Definition

```sql
-- ============================================================
-- Extensions
-- ============================================================
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "ltree";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- ============================================================
-- Tenants (multi-tenant support)
-- ============================================================
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    settings        JSONB NOT NULL DEFAULT '{}',
    sso_provider    VARCHAR(50),       -- 'saml' | 'oidc' | NULL
    sso_config      JSONB,
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
    address_line1   VARCHAR(255),
    address_line2   VARCHAR(255),
    city            VARCHAR(100),
    state_province  VARCHAR(100),
    postal_code     VARCHAR(20),
    country_code    CHAR(2),           -- ISO 3166-1 alpha-2
    timezone        VARCHAR(50),       -- IANA timezone
    is_remote       BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

-- ============================================================
-- Departments
-- ============================================================
CREATE TABLE departments (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(50),
    parent_id       UUID REFERENCES departments(id) ON DELETE SET NULL,
    path            LTREE,             -- materialized path for fast subtree queries
    cost_centre     VARCHAR(50),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

CREATE INDEX idx_departments_path ON departments USING GIST (path);
CREATE INDEX idx_departments_parent ON departments (parent_id);
CREATE INDEX idx_departments_tenant ON departments (tenant_id);

-- ============================================================
-- Pay Bands / Levels
-- ============================================================
CREATE TABLE pay_bands (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(100) NOT NULL,   -- e.g. 'L4', 'IC3', 'M2'
    level_order     INTEGER NOT NULL,        -- numeric sort order
    min_salary      NUMERIC(12, 2),
    max_salary      NUMERIC(12, 2),
    currency        CHAR(3) DEFAULT 'USD',   -- ISO 4217
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

-- ============================================================
-- Positions (roles that exist in the org, filled or open)
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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_positions_dept ON positions (department_id);
CREATE INDEX idx_positions_reports ON positions (reports_to_id);
CREATE INDEX idx_positions_tenant ON positions (tenant_id);

-- ============================================================
-- Employees
-- ============================================================
CREATE TABLE employees (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    external_id     VARCHAR(255),            -- HRIS source ID
    employee_number VARCHAR(50),
    email           VARCHAR(255) NOT NULL,
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    preferred_name  VARCHAR(100),
    photo_url       TEXT,
    phone           VARCHAR(30),
    position_id     UUID REFERENCES positions(id) ON DELETE SET NULL,
    manager_id      UUID REFERENCES employees(id) ON DELETE SET NULL,
    org_path        LTREE,                   -- materialized reporting path
    department_id   UUID NOT NULL REFERENCES departments(id) ON DELETE RESTRICT,
    location_id     UUID REFERENCES locations(id) ON DELETE SET NULL,
    pay_band_id     UUID REFERENCES pay_bands(id) ON DELETE SET NULL,
    hire_date       DATE,
    termination_date DATE,
    employment_type VARCHAR(20) NOT NULL DEFAULT 'full_time',
        -- 'full_time' | 'part_time' | 'contractor' | 'intern'
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
        -- 'active' | 'inactive' | 'on_leave' | 'terminated'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email),
    UNIQUE (tenant_id, employee_number)
);

CREATE INDEX idx_employees_manager ON employees (manager_id);
CREATE INDEX idx_employees_dept ON employees (department_id);
CREATE INDEX idx_employees_org_path ON employees USING GIST (org_path);
CREATE INDEX idx_employees_tenant ON employees (tenant_id);
CREATE INDEX idx_employees_status ON employees (tenant_id, status);
CREATE INDEX idx_employees_external ON employees (tenant_id, external_id);

-- ============================================================
-- Employee DEI Demographics (separated for privacy / access control)
-- ============================================================
CREATE TABLE employee_demographics (
    employee_id     UUID PRIMARY KEY REFERENCES employees(id) ON DELETE CASCADE,
    gender          VARCHAR(50),
    ethnicity       VARCHAR(100),
    age_band        VARCHAR(20),         -- '18-24', '25-34', etc.
    disability_status VARCHAR(50),
    veteran_status  VARCHAR(50),
    voluntary_self_id BOOLEAN NOT NULL DEFAULT FALSE,
    collected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- Custom Fields (tenant-configurable)
-- ============================================================
CREATE TABLE custom_field_definitions (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    entity_type     VARCHAR(30) NOT NULL,    -- 'employee' | 'department' | 'position'
    field_name      VARCHAR(100) NOT NULL,
    field_type      VARCHAR(20) NOT NULL,    -- 'text' | 'number' | 'date' | 'select' | 'multi_select'
    options         TEXT[],                  -- for select/multi_select types
    is_required     BOOLEAN NOT NULL DEFAULT FALSE,
    visibility      VARCHAR(20) NOT NULL DEFAULT 'company',
        -- 'hr_only' | 'manager' | 'company' | 'self'
    sort_order      INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, entity_type, field_name)
);

CREATE TABLE custom_field_values (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    definition_id   UUID NOT NULL REFERENCES custom_field_definitions(id) ON DELETE CASCADE,
    entity_id       UUID NOT NULL,           -- polymorphic FK to employee/dept/position
    value_text      TEXT,
    value_number    NUMERIC,
    value_date      DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (definition_id, entity_id)
);

CREATE INDEX idx_cfv_entity ON custom_field_values (entity_id);

-- ============================================================
-- Compensation Records (historical)
-- ============================================================
CREATE TABLE compensation_records (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    employee_id     UUID NOT NULL REFERENCES employees(id) ON DELETE CASCADE,
    effective_date  DATE NOT NULL,
    salary          NUMERIC(12, 2) NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    bonus_target    NUMERIC(5, 2),           -- percentage
    equity_shares   INTEGER,
    pay_band_id     UUID REFERENCES pay_bands(id),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (employee_id, effective_date)
);

CREATE INDEX idx_comp_employee ON compensation_records (employee_id, effective_date DESC);

-- ============================================================
-- Org Change History (audit trail)
-- ============================================================
CREATE TABLE org_change_log (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    entity_type     VARCHAR(30) NOT NULL,    -- 'employee' | 'department' | 'position'
    entity_id       UUID NOT NULL,
    change_type     VARCHAR(20) NOT NULL,    -- 'created' | 'updated' | 'deleted'
    changed_fields  TEXT[] NOT NULL DEFAULT '{}',
    old_values      JSONB,
    new_values      JSONB,
    changed_by      UUID,                    -- user who made the change
    source          VARCHAR(30) NOT NULL DEFAULT 'manual',
        -- 'manual' | 'hris_sync' | 'api' | 'csv_import'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ocl_entity ON org_change_log (entity_type, entity_id);
CREATE INDEX idx_ocl_tenant_time ON org_change_log (tenant_id, created_at DESC);

-- ============================================================
-- HRIS Sync Configuration
-- ============================================================
CREATE TABLE hris_connections (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    provider        VARCHAR(50) NOT NULL,    -- 'bamboohr' | 'rippling' | 'workday' | 'adp' | 'hibob'
    credentials     BYTEA,                   -- encrypted OAuth tokens / API keys
    sync_frequency  INTERVAL NOT NULL DEFAULT '15 minutes',
    field_mapping   JSONB NOT NULL DEFAULT '{}',
    last_sync_at    TIMESTAMPTZ,
    last_sync_status VARCHAR(20),            -- 'success' | 'partial' | 'failed'
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, provider)
);

CREATE TABLE hris_sync_log (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    connection_id   UUID NOT NULL REFERENCES hris_connections(id) ON DELETE CASCADE,
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    status          VARCHAR(20) NOT NULL DEFAULT 'running',
    records_created INTEGER DEFAULT 0,
    records_updated INTEGER DEFAULT 0,
    records_failed  INTEGER DEFAULT 0,
    error_details   JSONB
);

-- ============================================================
-- Headcount Planning & Scenarios
-- ============================================================
CREATE TABLE planning_scenarios (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    base_snapshot_date DATE NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
        -- 'draft' | 'in_review' | 'approved' | 'archived'
    created_by      UUID NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE scenario_changes (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    scenario_id     UUID NOT NULL REFERENCES planning_scenarios(id) ON DELETE CASCADE,
    change_type     VARCHAR(30) NOT NULL,
        -- 'new_hire' | 'termination' | 'promotion' | 'transfer' | 'reorg' | 'role_change'
    target_position_id UUID REFERENCES positions(id),
    target_employee_id UUID REFERENCES employees(id),
    proposed_department_id UUID REFERENCES departments(id),
    proposed_manager_id UUID REFERENCES employees(id),
    proposed_title  VARCHAR(255),
    proposed_pay_band_id UUID REFERENCES pay_bands(id),
    proposed_salary NUMERIC(12, 2),
    proposed_start_date DATE,
    budget_impact   NUMERIC(12, 2),
    notes           TEXT,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sc_scenario ON scenario_changes (scenario_id);

-- ============================================================
-- Succession Planning
-- ============================================================
CREATE TABLE succession_plans (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    position_id     UUID NOT NULL REFERENCES positions(id) ON DELETE CASCADE,
    risk_level      VARCHAR(20) NOT NULL DEFAULT 'medium',
        -- 'low' | 'medium' | 'high' | 'critical'
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE succession_candidates (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    plan_id         UUID NOT NULL REFERENCES succession_plans(id) ON DELETE CASCADE,
    employee_id     UUID NOT NULL REFERENCES employees(id) ON DELETE CASCADE,
    readiness       VARCHAR(20) NOT NULL DEFAULT 'developing',
        -- 'ready_now' | 'ready_1yr' | 'developing' | 'not_ready'
    priority        INTEGER NOT NULL DEFAULT 1,
    notes           TEXT,
    UNIQUE (plan_id, employee_id)
);

-- ============================================================
-- Users & Access Control
-- ============================================================
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    employee_id     UUID REFERENCES employees(id) ON DELETE SET NULL,
    email           VARCHAR(255) NOT NULL,
    password_hash   TEXT,                    -- NULL for SSO-only users
    role            VARCHAR(30) NOT NULL DEFAULT 'viewer',
        -- 'super_admin' | 'admin' | 'hr_manager' | 'manager' | 'viewer'
    scim_external_id VARCHAR(255),           -- SCIM 2.0 provisioning ID
    last_login_at   TIMESTAMPTZ,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

-- ============================================================
-- API Keys & Webhooks
-- ============================================================
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
    events          TEXT[] NOT NULL,          -- e.g. '{employee.created,employee.updated}'
    secret_hash     TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Key Hierarchical Query Examples

```sql
-- All direct and indirect reports for an employee using ltree
SELECT e.id, e.first_name, e.last_name, e.org_path
FROM employees e
WHERE e.org_path <@ (SELECT org_path FROM employees WHERE id = :manager_id)
  AND e.id != :manager_id
  AND e.tenant_id = :tenant_id;

-- Span of control (direct reports count per manager)
SELECT manager_id, COUNT(*) AS direct_reports
FROM employees
WHERE tenant_id = :tenant_id AND status = 'active' AND manager_id IS NOT NULL
GROUP BY manager_id
ORDER BY direct_reports DESC;

-- DEI breakdown by department
SELECT d.name AS department,
       ed.gender,
       COUNT(*) AS headcount,
       ROUND(COUNT(*)::NUMERIC / SUM(COUNT(*)) OVER (PARTITION BY d.id) * 100, 1) AS pct
FROM employees e
JOIN departments d ON e.department_id = d.id
JOIN employee_demographics ed ON e.id = ed.employee_id
WHERE e.tenant_id = :tenant_id AND e.status = 'active'
GROUP BY d.id, d.name, ed.gender
ORDER BY d.name, headcount DESC;
```

---

## Trade-offs

**Strengths:**
- Strong referential integrity ensures data consistency across HRIS sync operations
- Mature tooling for backups, replication, and monitoring
- `ltree` provides O(1) ancestor/descendant lookups without recursive joins
- Clean separation of DEI demographics for field-level privacy controls
- Well-understood migration and schema evolution via tools like Flyway or Alembic
- Excellent fit for self-hosted deployments with minimal operational overhead

**Weaknesses:**
- Custom fields require an EAV pattern (custom_field_definitions + custom_field_values), which adds query complexity
- Schema changes for new HRIS integrations require migrations
- Large-scale reorg simulations may require denormalised scenario snapshots for performance
- `ltree` paths must be maintained on every hierarchy change (trigger-managed)

**Scalability:**
- Handles 10,000-100,000 employees comfortably on a single PostgreSQL instance
- Read replicas support analytics workloads without affecting transactional performance
- Partitioning `org_change_log` and `hris_sync_log` by date keeps historical tables fast

**Migration path:**
- Can evolve toward Suggestion 3 (hybrid JSONB) by adding JSONB columns incrementally
- Event sourcing (Suggestion 2) can be layered on top via `org_change_log` expansion
- Graph-based queries can be handled by materialised views or a read-side Neo4j projection
