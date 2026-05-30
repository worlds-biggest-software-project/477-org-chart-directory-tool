# Data Model Suggestion 4: Graph Database Model (Neo4j)

## Approach

A property graph model using Neo4j, where employees, departments, positions, and locations are nodes and organisational relationships (REPORTS_TO, BELONGS_TO, LOCATED_AT, FILLS) are first-class edges. This is the domain-specific specialty approach: org charts are literally graphs, and a graph database represents them natively rather than encoding them into tables. Cypher queries express hierarchical traversals, span-of-control analysis, and reorg simulations in a way that mirrors how people think about organisational structure.

## Why This Suits the Domain

An org chart is a directed acyclic graph. Every feature in this product -- hierarchical traversal, subtree rendering, span-of-control analysis, matrix reporting, shortest-path-to-decision-maker, reorg simulation -- maps directly to graph traversal operations. In a relational database, these require recursive CTEs, ltree extensions, or closure tables. In Neo4j, they are native operations with O(1) relationship traversal regardless of total dataset size. The "who knows whom" and "who reports through whom" questions that power AI-driven people search are graph pattern matching problems. Scenario planning becomes a matter of adding tentative edges and traversing them alongside (or instead of) current edges.

---

## Node Definitions

```cypher
// ============================================================
// Constraints and Indexes
// ============================================================

// Uniqueness constraints (also create indexes)
CREATE CONSTRAINT employee_tenant_email IF NOT EXISTS
FOR (e:Employee) REQUIRE (e.tenant_id, e.email) IS UNIQUE;

CREATE CONSTRAINT employee_tenant_number IF NOT EXISTS
FOR (e:Employee) REQUIRE (e.tenant_id, e.employee_number) IS UNIQUE;

CREATE CONSTRAINT department_tenant_code IF NOT EXISTS
FOR (d:Department) REQUIRE (d.tenant_id, d.code) IS UNIQUE;

CREATE CONSTRAINT position_id IF NOT EXISTS
FOR (p:Position) REQUIRE p.id IS UNIQUE;

CREATE CONSTRAINT location_tenant_name IF NOT EXISTS
FOR (l:Location) REQUIRE (l.tenant_id, l.name) IS UNIQUE;

CREATE CONSTRAINT tenant_slug IF NOT EXISTS
FOR (t:Tenant) REQUIRE t.slug IS UNIQUE;

CREATE CONSTRAINT pay_band_tenant_name IF NOT EXISTS
FOR (pb:PayBand) REQUIRE (pb.tenant_id, pb.name) IS UNIQUE;

CREATE CONSTRAINT scenario_id IF NOT EXISTS
FOR (s:Scenario) REQUIRE s.id IS UNIQUE;

CREATE CONSTRAINT user_tenant_email IF NOT EXISTS
FOR (u:User) REQUIRE (u.tenant_id, u.email) IS UNIQUE;

// Additional indexes for common lookups
CREATE INDEX employee_status IF NOT EXISTS FOR (e:Employee) ON (e.tenant_id, e.status);
CREATE INDEX employee_hris IF NOT EXISTS FOR (e:Employee) ON (e.tenant_id, e.hris_external_id);
CREATE INDEX department_name IF NOT EXISTS FOR (d:Department) ON (d.tenant_id, d.name);

// Full-text search index
CREATE FULLTEXT INDEX employee_search IF NOT EXISTS
FOR (e:Employee) ON EACH [e.first_name, e.last_name, e.preferred_name, e.title, e.bio];

// ============================================================
// Node: Tenant
// ============================================================
// Properties:
//   id: UUID
//   name: String
//   slug: String
//   settings: Map (feature flags, branding, privacy defaults)
//   sso_config: Map (SAML/OIDC configuration)
//   created_at: DateTime
//   updated_at: DateTime

// ============================================================
// Node: Employee
// ============================================================
// Properties:
//   id: UUID
//   tenant_id: UUID
//   email: String
//   first_name: String
//   last_name: String
//   preferred_name: String (optional)
//   employee_number: String (optional)
//   title: String
//   phone: String (optional)
//   photo_url: String (optional)
//   bio: String (optional)
//   pronouns: String (optional)
//   hire_date: Date
//   termination_date: Date (optional)
//   employment_type: String ('full_time'|'part_time'|'contractor'|'intern')
//   status: String ('active'|'inactive'|'on_leave'|'terminated')
//   hris_external_id: String (optional)
//   hris_last_sync: DateTime (optional)
//   custom_fields: Map (tenant-defined fields)
//   created_at: DateTime
//   updated_at: DateTime

// ============================================================
// Node: Department
// ============================================================
// Properties:
//   id: UUID
//   tenant_id: UUID
//   name: String
//   code: String
//   cost_centre: String (optional)
//   metadata: Map (optional)
//   created_at: DateTime
//   updated_at: DateTime

// ============================================================
// Node: Position
// ============================================================
// Properties:
//   id: UUID
//   tenant_id: UUID
//   title: String
//   is_open: Boolean
//   headcount: Integer
//   fte: Float
//   job_details: Map (description, requirements, skills)
//   created_at: DateTime
//   updated_at: DateTime

// ============================================================
// Node: Location
// ============================================================
// Properties:
//   id: UUID
//   tenant_id: UUID
//   name: String
//   country_code: String
//   timezone: String
//   is_remote: Boolean
//   address: Map (line1, line2, city, state, postal_code)
//   created_at: DateTime

// ============================================================
// Node: PayBand
// ============================================================
// Properties:
//   id: UUID
//   tenant_id: UUID
//   name: String
//   level_order: Integer
//   compensation_ranges: Map (currency-keyed salary ranges)
//   created_at: DateTime

// ============================================================
// Node: Scenario
// ============================================================
// Properties:
//   id: UUID
//   tenant_id: UUID
//   name: String
//   description: String (optional)
//   base_snapshot_date: Date
//   status: String ('draft'|'in_review'|'approved'|'archived')
//   created_by: UUID
//   created_at: DateTime
//   updated_at: DateTime

// ============================================================
// Node: Demographics (separate for privacy)
// ============================================================
// Properties:
//   employee_id: UUID
//   gender: String
//   ethnicity: String
//   age_band: String
//   disability_status: String
//   veteran_status: String
//   voluntary_self_id: Boolean
//   collected_at: DateTime
```

---

## Relationship Definitions

```cypher
// ============================================================
// Core Organisational Relationships
// ============================================================

// Employee reports to another employee (the org chart)
// (subordinate)-[:REPORTS_TO {since: date, is_dotted_line: false}]->(manager)
CREATE (emp)-[:REPORTS_TO {
    since: date('2024-03-15'),
    is_dotted_line: false
}]->(manager)

// Employee belongs to a department
// (employee)-[:BELONGS_TO {since: date, role: 'member'|'lead'|'head'}]->(department)
CREATE (emp)-[:BELONGS_TO {
    since: date('2024-03-15'),
    role: 'member'
}]->(dept)

// Employee fills a position
// (employee)-[:FILLS {since: date}]->(position)
CREATE (emp)-[:FILLS {
    since: date('2024-03-15')
}]->(pos)

// Employee is located at a location
// (employee)-[:LOCATED_AT {since: date, is_primary: true}]->(location)
CREATE (emp)-[:LOCATED_AT {
    since: date('2024-03-15'),
    is_primary: true
}]->(loc)

// Employee has a pay band
// (employee)-[:HAS_PAY_BAND {since: date, salary: 150000, currency: 'USD'}]->(payband)
CREATE (emp)-[:HAS_PAY_BAND {
    since: date('2024-03-15'),
    salary: 150000,
    currency: 'USD',
    bonus_target_pct: 15
}]->(pb)

// Employee has demographics (1:1, separate for access control)
// (employee)-[:HAS_DEMOGRAPHICS]->(demographics)
CREATE (emp)-[:HAS_DEMOGRAPHICS]->(demo)

// ============================================================
// Department Hierarchy
// ============================================================

// Department is a child of another department
// (child_dept)-[:CHILD_OF]->(parent_dept)
CREATE (engineering)-[:CHILD_OF]->(technology)

// Department is in a tenant
// (department)-[:IN_TENANT]->(tenant)
CREATE (dept)-[:IN_TENANT]->(tenant)

// ============================================================
// Position Hierarchy
// ============================================================

// Position reports to another position (structural, independent of who fills it)
// (junior_position)-[:POSITION_REPORTS_TO]->(senior_position)
CREATE (seniorEng)-[:POSITION_REPORTS_TO]->(engManager)

// Position is in a department
// (position)-[:IN_DEPARTMENT]->(department)
CREATE (pos)-[:IN_DEPARTMENT]->(dept)

// Position is at a location
// (position)-[:BASED_AT]->(location)
CREATE (pos)-[:BASED_AT]->(loc)

// ============================================================
// Scenario Planning Relationships
// ============================================================

// Proposed changes within a scenario (virtual edges)
// (scenario)-[:PROPOSES_CHANGE {change_type, details}]->(target_node)
CREATE (scenario)-[:PROPOSES_CHANGE {
    change_type: 'transfer',
    from_department: 'Engineering',
    to_department: 'Product',
    proposed_date: date('2026-07-01'),
    budget_impact: -5000
}]->(emp)

// Proposed new reporting line in a scenario
// (employee)-[:WOULD_REPORT_TO {scenario_id: UUID}]->(new_manager)
CREATE (emp)-[:WOULD_REPORT_TO {
    scenario_id: 'uuid-scenario-1'
}]->(newManager)

// ============================================================
// Succession Planning Relationships
// ============================================================

// Position is at risk
// (position)-[:HAS_SUCCESSION_RISK {level: 'critical', notes: '...'}]->()
// Candidate could fill position
// (employee)-[:SUCCESSION_CANDIDATE {readiness, priority, development_plan}]->(position)
CREATE (emp)-[:SUCCESSION_CANDIDATE {
    readiness: 'ready_1yr',
    priority: 1,
    development_plan: 'Complete leadership programme by Q4'
}]->(pos)

// ============================================================
// Historical Relationships (temporal edges)
// ============================================================

// Past reporting relationships (kept for audit/analytics)
// (employee)-[:PREVIOUSLY_REPORTED_TO {from: date, to: date}]->(former_manager)
CREATE (emp)-[:PREVIOUSLY_REPORTED_TO {
    from: date('2023-01-15'),
    to: date('2024-03-14')
}]->(formerManager)

// Past department membership
// (employee)-[:PREVIOUSLY_IN {from: date, to: date}]->(former_dept)
CREATE (emp)-[:PREVIOUSLY_IN {
    from: date('2023-01-15'),
    to: date('2024-03-14')
}]->(formerDept)
```

---

## Key Cypher Queries

```cypher
// ============================================================
// Org Chart: Full subtree under a manager
// ============================================================
MATCH path = (manager:Employee {id: $managerId})<-[:REPORTS_TO*]-(report:Employee)
WHERE report.status = 'active'
RETURN report.id, report.first_name, report.last_name, report.title,
       length(path) AS depth
ORDER BY depth, report.last_name;

// ============================================================
// Span of control analysis (direct + total reports)
// ============================================================
MATCH (mgr:Employee {tenant_id: $tenantId, status: 'active'})<-[:REPORTS_TO]-(direct:Employee {status: 'active'})
WITH mgr, COUNT(direct) AS direct_reports
OPTIONAL MATCH (mgr)<-[:REPORTS_TO*]-(all_report:Employee {status: 'active'})
WITH mgr, direct_reports, COUNT(all_report) AS total_reports
RETURN mgr.id, mgr.first_name, mgr.last_name, mgr.title,
       direct_reports, total_reports
ORDER BY direct_reports DESC;

// ============================================================
// Find the reporting path between any two employees
// ============================================================
MATCH path = shortestPath(
    (emp1:Employee {id: $employeeId})-[:REPORTS_TO*]-(emp2:Employee {id: $targetId})
)
RETURN [n IN nodes(path) | {id: n.id, name: n.first_name + ' ' + n.last_name, title: n.title}] AS chain;

// ============================================================
// DEI breakdown by department
// ============================================================
MATCH (e:Employee {tenant_id: $tenantId, status: 'active'})-[:BELONGS_TO]->(d:Department)
MATCH (e)-[:HAS_DEMOGRAPHICS]->(dem:Demographics)
RETURN d.name AS department,
       dem.gender AS gender,
       dem.ethnicity AS ethnicity,
       COUNT(*) AS headcount
ORDER BY d.name, headcount DESC;

// ============================================================
// Matrix reporting: employees with dotted-line reports
// ============================================================
MATCH (e:Employee {tenant_id: $tenantId, status: 'active'})-[r:REPORTS_TO]->(mgr:Employee)
WHERE r.is_dotted_line = true
RETURN e.first_name + ' ' + e.last_name AS employee,
       mgr.first_name + ' ' + mgr.last_name AS dotted_line_manager,
       r.since AS since;

// ============================================================
// Scenario comparison: show proposed vs current structure
// ============================================================
MATCH (s:Scenario {id: $scenarioId})-[pc:PROPOSES_CHANGE]->(target)
OPTIONAL MATCH (target)-[:BELONGS_TO]->(current_dept:Department)
RETURN target.first_name + ' ' + target.last_name AS employee,
       pc.change_type,
       current_dept.name AS current_department,
       pc.to_department AS proposed_department,
       pc.budget_impact;

// ============================================================
// Succession readiness overview
// ============================================================
MATCH (pos:Position {tenant_id: $tenantId})<-[sc:SUCCESSION_CANDIDATE]-(candidate:Employee)
OPTIONAL MATCH (pos)<-[:FILLS]-(incumbent:Employee)
RETURN pos.title,
       incumbent.first_name + ' ' + incumbent.last_name AS current_holder,
       candidate.first_name + ' ' + candidate.last_name AS successor,
       sc.readiness, sc.priority
ORDER BY pos.title, sc.priority;

// ============================================================
// AI people search: "Who leads design in EMEA?"
// ============================================================
MATCH (e:Employee {tenant_id: $tenantId, status: 'active'})-[:BELONGS_TO]->(d:Department)
MATCH (e)-[:LOCATED_AT]->(l:Location)
WHERE d.name CONTAINS 'Design'
  AND l.country_code IN ['GB', 'DE', 'FR', 'NL', 'IE', 'ES', 'IT']
  AND EXISTS { (e)<-[:REPORTS_TO]-(:Employee) }
RETURN e.first_name + ' ' + e.last_name AS name,
       e.title, d.name AS department, l.name AS office
ORDER BY e.title;

// ============================================================
// Historical: org structure at a point in time
// ============================================================
MATCH (e:Employee {tenant_id: $tenantId})
WHERE e.hire_date <= date($asOfDate)
  AND (e.termination_date IS NULL OR e.termination_date > date($asOfDate))
OPTIONAL MATCH (e)-[r:REPORTS_TO]->(mgr:Employee)
WHERE r.since <= date($asOfDate)
OPTIONAL MATCH (e)-[prev:PREVIOUSLY_REPORTED_TO]->(old_mgr:Employee)
WHERE prev.from <= date($asOfDate) AND prev.to > date($asOfDate)
WITH e, COALESCE(mgr, old_mgr) AS effective_manager
RETURN e.id, e.first_name, e.last_name, e.title,
       effective_manager.id AS manager_id,
       effective_manager.first_name + ' ' + effective_manager.last_name AS manager_name;
```

---

## Data Import Pattern

```cypher
// Bulk import from HRIS sync via CSV or APOC procedures
// Step 1: Load employees
LOAD CSV WITH HEADERS FROM 'file:///employees.csv' AS row
MERGE (e:Employee {id: row.id, tenant_id: row.tenant_id})
SET e.email = row.email,
    e.first_name = row.first_name,
    e.last_name = row.last_name,
    e.title = row.title,
    e.status = row.status,
    e.hire_date = date(row.hire_date),
    e.updated_at = datetime();

// Step 2: Create REPORTS_TO relationships
LOAD CSV WITH HEADERS FROM 'file:///reporting.csv' AS row
MATCH (emp:Employee {id: row.employee_id})
MATCH (mgr:Employee {id: row.manager_id})
MERGE (emp)-[r:REPORTS_TO]->(mgr)
SET r.since = date(row.since),
    r.is_dotted_line = CASE row.is_dotted_line WHEN 'true' THEN true ELSE false END;

// Step 3: Department membership
LOAD CSV WITH HEADERS FROM 'file:///departments.csv' AS row
MATCH (emp:Employee {id: row.employee_id})
MATCH (dept:Department {id: row.department_id})
MERGE (emp)-[r:BELONGS_TO]->(dept)
SET r.since = date(row.since),
    r.role = row.role;
```

---

## Architecture: Neo4j + PostgreSQL Sidecar

For production, this model pairs Neo4j with a lightweight PostgreSQL instance for data that graphs handle poorly:

| Concern | Store |
|---|---|
| Org hierarchy, reporting lines, traversals | Neo4j |
| Employee profiles, custom fields, directory | Neo4j |
| DEI demographics, succession planning | Neo4j |
| Scenario planning (proposed edges) | Neo4j |
| User authentication, sessions, API keys | PostgreSQL |
| HRIS connection config, credentials | PostgreSQL |
| Webhook delivery log, sync audit log | PostgreSQL |
| File/export storage metadata | PostgreSQL |
| Full-text search (if not using Neo4j FTS) | PostgreSQL + pg_trgm |

---

## Trade-offs

**Strengths:**
- Organisational hierarchy is the native data structure -- no impedance mismatch
- Relationship traversal (subtrees, paths, span-of-control) is O(k) where k is the path length, independent of total graph size
- Matrix reporting (dotted-line relationships) is a first-class concept, not a workaround
- Scenario planning maps naturally to tentative edges alongside real edges
- Historical relationships (PREVIOUSLY_REPORTED_TO) enable point-in-time reconstruction without complex temporal joins
- Cypher queries read like natural language descriptions of org chart questions
- Graph visualisation libraries (neovis.js, D3) integrate directly with Neo4j query results

**Weaknesses:**
- Neo4j adds operational complexity -- a second database to deploy, back up, and monitor alongside PostgreSQL
- Self-hosting Neo4j Community Edition lacks some enterprise features (clustering, role-based access)
- Aggregation queries (DEI statistics, headcount totals) are less efficient than SQL GROUP BY operations
- No built-in ACID transactions across Neo4j + PostgreSQL (requires application-level coordination or saga patterns)
- Smaller talent pool familiar with Cypher compared to SQL
- Schema enforcement is weaker than relational databases -- must rely on application-level validation and constraint declarations
- Multi-tenancy requires careful tenant_id filtering on every query (no native tenant isolation)

**Scalability:**
- Neo4j handles millions of nodes and relationships on a single instance
- For this domain (10k-100k employees per tenant), a single Neo4j instance is more than sufficient
- Read replicas (Neo4j Enterprise) can serve analytics workloads
- Graph queries remain fast as the organisation grows because traversal cost depends on local neighbourhood size, not total graph size

**Migration path:**
- Can start with PostgreSQL (Suggestions 1 or 3) and add Neo4j as a read-side projection fed by change data capture
- A hybrid approach keeps PostgreSQL as the system of record and uses Neo4j exclusively for hierarchy traversal and visualization queries
- If Neo4j proves unnecessary, the same queries can be served by PostgreSQL ltree + recursive CTEs with moderate performance trade-offs
- For maximum simplicity, Apache AGE (a PostgreSQL extension for graph queries) could provide graph capabilities without a separate database, though with less mature tooling than Neo4j
