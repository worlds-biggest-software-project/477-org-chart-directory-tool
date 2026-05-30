# Data Model Suggestion 2: Event-Sourced / CQRS Model

## Approach

An event-sourced architecture with Command Query Responsibility Segregation (CQRS). Every change to the organisational structure is captured as an immutable domain event. Commands validate business rules and emit events; read-side projections materialise optimised views for the org chart, directory, analytics, and planning features. The event store uses PostgreSQL for accessibility and self-hosting, with projections maintained in the same database.

## Why This Suits the Domain

Org chart and directory tools are fundamentally about tracking **change over time**. Who reported to whom last quarter? What did the department structure look like before the reorg? How has headcount shifted? Event sourcing makes temporal queries natural -- the audit trail is not a bolt-on feature but the primary data store. HRIS sync becomes a stream of inbound events that can be replayed, diffed, or rolled back. Scenario planning maps directly to "what-if" event streams that branch from a known state. DEI analytics can be projected from the same events with full historical context.

---

## Domain Events

```
-- ============================================================
-- Event Store (PostgreSQL)
-- ============================================================

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Global event log -- the single source of truth
CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id       UUID NOT NULL,
    aggregate_type  VARCHAR(50) NOT NULL,
        -- 'Employee' | 'Department' | 'Position' | 'Location' | 'Scenario'
    aggregate_id    UUID NOT NULL,
    event_type      VARCHAR(100) NOT NULL,
    event_version   INTEGER NOT NULL,         -- optimistic concurrency
    payload         JSONB NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',
        -- { "user_id": "...", "source": "hris_sync|manual|api", "correlation_id": "..." }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (aggregate_id, event_version)
);

CREATE INDEX idx_es_aggregate ON event_store (aggregate_type, aggregate_id, event_version);
CREATE INDEX idx_es_tenant_time ON event_store (tenant_id, created_at);
CREATE INDEX idx_es_event_type ON event_store (event_type);
```

### Employee Aggregate Events

| Event Type | Payload Fields |
|---|---|
| `EmployeeHired` | `employee_number, email, first_name, last_name, department_id, position_id, manager_id, location_id, hire_date, employment_type` |
| `EmployeeProfileUpdated` | `changed_fields: { field: { old, new } }` |
| `EmployeePhotoChanged` | `photo_url` |
| `EmployeeTransferred` | `from_department_id, to_department_id, from_manager_id, to_manager_id, effective_date` |
| `EmployeePromoted` | `from_position_id, to_position_id, from_pay_band_id, to_pay_band_id, new_title, effective_date` |
| `EmployeeCompensationChanged` | `old_salary, new_salary, currency, bonus_target, equity_shares, effective_date` |
| `EmployeeDemographicsUpdated` | `gender, ethnicity, age_band, disability_status, veteran_status` |
| `EmployeeTerminated` | `termination_date, reason, voluntary` |
| `EmployeeReactivated` | `reactivation_date, position_id, department_id` |

### Department Aggregate Events

| Event Type | Payload Fields |
|---|---|
| `DepartmentCreated` | `name, code, parent_id, cost_centre` |
| `DepartmentRenamed` | `old_name, new_name` |
| `DepartmentMoved` | `old_parent_id, new_parent_id` |
| `DepartmentMerged` | `absorbed_department_id, target_department_id` |
| `DepartmentDeactivated` | `reason` |

### Position Aggregate Events

| Event Type | Payload Fields |
|---|---|
| `PositionCreated` | `title, department_id, location_id, pay_band_id, reports_to_position_id, headcount, fte` |
| `PositionUpdated` | `changed_fields: { field: { old, new } }` |
| `PositionOpened` | `reason, budget_approved` |
| `PositionFilled` | `employee_id, start_date` |
| `PositionClosed` | `reason` |

### HRIS Sync Events

| Event Type | Payload Fields |
|---|---|
| `HrisSyncStarted` | `provider, connection_id, sync_id` |
| `HrisSyncRecordReceived` | `external_id, raw_data, mapped_data` |
| `HrisSyncConflictDetected` | `field, local_value, remote_value, resolution` |
| `HrisSyncCompleted` | `records_created, records_updated, records_failed, duration_ms` |

### Scenario Planning Events

| Event Type | Payload Fields |
|---|---|
| `ScenarioCreated` | `name, description, base_snapshot_event_id` |
| `ScenarioChangeProposed` | `change_type, target_id, proposed_values, budget_impact` |
| `ScenarioChangeReverted` | `change_id, reason` |
| `ScenarioApproved` | `approved_by, approval_date` |
| `ScenarioCommitted` | `changes_applied: [event_ids]` |

---

## Command Handlers

```python
# Pseudocode -- command handler pattern

class TransferEmployeeCommandHandler:
    """
    Command: TransferEmployee
    Validates: employee exists, is active, target department exists,
               new manager exists and is in target department
    Emits: EmployeeTransferred
    """
    def handle(self, cmd: TransferEmployeeCommand) -> list[DomainEvent]:
        employee = self.repository.load("Employee", cmd.employee_id)

        # Business rule validation
        assert employee.status == "active", "Cannot transfer inactive employee"
        assert self.department_exists(cmd.to_department_id), "Target department not found"
        assert self.employee_active(cmd.to_manager_id), "Target manager not active"

        # Emit domain event
        return [EmployeeTransferred(
            aggregate_id=cmd.employee_id,
            from_department_id=employee.department_id,
            to_department_id=cmd.to_department_id,
            from_manager_id=employee.manager_id,
            to_manager_id=cmd.to_manager_id,
            effective_date=cmd.effective_date,
        )]


class ProposeScenarioChangeHandler:
    """
    Command: ProposeScenarioChange
    Validates: scenario is in 'draft' status, change is internally consistent
    Emits: ScenarioChangeProposed
    """
    def handle(self, cmd: ProposeScenarioChangeCommand) -> list[DomainEvent]:
        scenario = self.repository.load("Scenario", cmd.scenario_id)
        assert scenario.status == "draft", "Scenario is locked"

        budget_impact = self.calculate_budget_impact(cmd)

        return [ScenarioChangeProposed(
            aggregate_id=cmd.scenario_id,
            change_type=cmd.change_type,
            target_id=cmd.target_id,
            proposed_values=cmd.proposed_values,
            budget_impact=budget_impact,
        )]
```

---

## Read-Side Projections

Projections consume events and maintain denormalised read models. Each projection is independently rebuildable from the event store.

```sql
-- ============================================================
-- Projection: Current Org Directory (flat read model)
-- ============================================================
CREATE TABLE proj_employee_directory (
    employee_id     UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    employee_number VARCHAR(50),
    email           VARCHAR(255) NOT NULL,
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    preferred_name  VARCHAR(100),
    photo_url       TEXT,
    phone           VARCHAR(30),
    title           VARCHAR(255),
    department_id   UUID,
    department_name VARCHAR(255),
    department_path TEXT,                     -- 'Engineering > Platform > Backend'
    location_id     UUID,
    location_name   VARCHAR(255),
    manager_id      UUID,
    manager_name    VARCHAR(200),
    pay_band        VARCHAR(100),
    hire_date       DATE,
    employment_type VARCHAR(20),
    status          VARCHAR(20),
    last_event_id   UUID NOT NULL,           -- cursor for projection rebuild
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ped_tenant ON proj_employee_directory (tenant_id);
CREATE INDEX idx_ped_dept ON proj_employee_directory (department_id);
CREATE INDEX idx_ped_manager ON proj_employee_directory (manager_id);
CREATE INDEX idx_ped_search ON proj_employee_directory
    USING GIN (to_tsvector('english', first_name || ' ' || last_name || ' ' || COALESCE(title, '')));

-- ============================================================
-- Projection: Org Chart Hierarchy (adjacency + depth cache)
-- ============================================================
CREATE TABLE proj_org_hierarchy (
    employee_id     UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    manager_id      UUID,
    depth           INTEGER NOT NULL DEFAULT 0,
    path            TEXT NOT NULL,            -- 'ceo_id/vp_id/dir_id/emp_id'
    direct_reports  INTEGER NOT NULL DEFAULT 0,
    total_reports   INTEGER NOT NULL DEFAULT 0,
    department_id   UUID,
    is_leaf         BOOLEAN NOT NULL DEFAULT TRUE,
    last_event_id   UUID NOT NULL,
    PRIMARY KEY (tenant_id, employee_id)
);

CREATE INDEX idx_poh_manager ON proj_org_hierarchy (manager_id);
CREATE INDEX idx_poh_path ON proj_org_hierarchy (path text_pattern_ops);

-- ============================================================
-- Projection: DEI Analytics (pre-aggregated)
-- ============================================================
CREATE TABLE proj_dei_snapshot (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id       UUID NOT NULL,
    snapshot_date   DATE NOT NULL,
    department_id   UUID,                    -- NULL = company-wide
    dimension       VARCHAR(50) NOT NULL,    -- 'gender' | 'ethnicity' | 'age_band' | 'pay_band'
    dimension_value VARCHAR(100) NOT NULL,
    headcount       INTEGER NOT NULL,
    percentage      NUMERIC(5, 2) NOT NULL,
    avg_salary      NUMERIC(12, 2),
    avg_tenure_days INTEGER,
    attrition_rate  NUMERIC(5, 2),
    last_event_id   UUID NOT NULL,
    UNIQUE (tenant_id, snapshot_date, department_id, dimension, dimension_value)
);

CREATE INDEX idx_pds_tenant_date ON proj_dei_snapshot (tenant_id, snapshot_date);

-- ============================================================
-- Projection: Headcount Timeline (for trend charts)
-- ============================================================
CREATE TABLE proj_headcount_timeline (
    tenant_id       UUID NOT NULL,
    period_date     DATE NOT NULL,           -- first of month
    department_id   UUID,                    -- NULL = company-wide
    active_count    INTEGER NOT NULL,
    hires_count     INTEGER NOT NULL DEFAULT 0,
    terms_count     INTEGER NOT NULL DEFAULT 0,
    transfers_in    INTEGER NOT NULL DEFAULT 0,
    transfers_out   INTEGER NOT NULL DEFAULT 0,
    open_positions  INTEGER NOT NULL DEFAULT 0,
    last_event_id   UUID NOT NULL,
    PRIMARY KEY (tenant_id, period_date, department_id)
);

-- ============================================================
-- Projection: Scenario Comparison View
-- ============================================================
CREATE TABLE proj_scenario_summary (
    scenario_id     UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    status          VARCHAR(20) NOT NULL,
    base_date       DATE NOT NULL,
    total_changes   INTEGER NOT NULL DEFAULT 0,
    new_hires       INTEGER NOT NULL DEFAULT 0,
    terminations    INTEGER NOT NULL DEFAULT 0,
    promotions      INTEGER NOT NULL DEFAULT 0,
    transfers       INTEGER NOT NULL DEFAULT 0,
    total_budget_impact NUMERIC(14, 2) NOT NULL DEFAULT 0,
    created_by_name VARCHAR(200),
    last_event_id   UUID NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- Projection tracking (for rebuild / catch-up)
-- ============================================================
CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
        -- 'active' | 'rebuilding' | 'paused'
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Event Processing Pipeline

```
                    +-----------------+
                    |  HRIS Webhook   |
                    |  API Request    |
                    |  CSV Import     |
                    +--------+--------+
                             |
                             v
                    +--------+--------+
                    | Command Handler |
                    | (validate +     |
                    |  emit events)   |
                    +--------+--------+
                             |
                             v
                    +--------+--------+
                    |   Event Store   |
                    |   (append-only) |
                    +--------+--------+
                             |
              +--------------+--------------+
              |              |              |
              v              v              v
     +--------+--+  +-------+---+  +-------+----+
     | Directory  |  | Org Chart |  | DEI        |
     | Projection |  | Hierarchy |  | Analytics  |
     +------------+  | Projection|  | Projection |
                      +-----------+  +------------+
```

---

## Snapshot Strategy

For aggregates with many events, periodic snapshots prevent slow rehydration:

```sql
CREATE TABLE aggregate_snapshots (
    aggregate_type  VARCHAR(50) NOT NULL,
    aggregate_id    UUID NOT NULL,
    event_version   INTEGER NOT NULL,
    state           JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (aggregate_type, aggregate_id, event_version)
);

-- Keep only the latest snapshot per aggregate; older ones can be pruned
CREATE INDEX idx_snap_latest ON aggregate_snapshots (aggregate_type, aggregate_id, event_version DESC);
```

---

## Trade-offs

**Strengths:**
- Complete, immutable audit trail -- every change is recorded with who, when, why, and from which source (manual, HRIS sync, API)
- Natural fit for temporal queries: "show me the org chart as of March 1st" is just replaying events to that point
- HRIS sync conflicts become explicit events that can be reviewed, not silent overwrites
- Scenario planning is a branching event stream -- approved scenarios commit their events to the main stream
- Projections can be rebuilt at any time without data loss, enabling safe schema evolution of read models
- Read models are independently scalable and tuneable for specific query patterns

**Weaknesses:**
- Higher implementation complexity -- requires event store infrastructure, projection workers, and rebuild tooling
- Eventually consistent read models introduce a small delay between write and read availability
- Debugging requires correlating events rather than inspecting a single row
- Schema evolution of events (not projections) requires careful versioning (upcasting)
- More storage than a mutable state model -- every change is kept forever (though compression and archival mitigate this)
- Steeper learning curve for teams unfamiliar with event sourcing

**Scalability:**
- Event store scales to millions of events per tenant on PostgreSQL with partitioning by `tenant_id` or `created_at`
- Projections can be split across read replicas or separate databases
- Event publishing can be parallelised by aggregate type for independent projection updates
- For very large organisations (100k+ employees), the event rate remains manageable (org changes are infrequent relative to, say, e-commerce transactions)

**Migration path:**
- Start with a simpler relational model (Suggestion 1) and add event sourcing incrementally by capturing change events alongside mutable state
- The `org_change_log` table in Suggestion 1 is effectively a stepping stone toward full event sourcing
- Projections can initially be materialised views before graduating to dedicated projection tables
- Can adopt a full event streaming platform (Kafka, EventStoreDB) later if throughput demands grow
