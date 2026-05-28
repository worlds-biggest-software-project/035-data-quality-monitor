# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Data Quality Monitor · Created: 2026-05-11

## Philosophy

This model follows classical third-normal-form (3NF) relational design: every concept gets its own table, relationships are enforced via foreign keys, and reference data is stored once. The schema is directly aligned with the ISO/IEC 25012 data quality dimensions (completeness, accuracy, consistency, currentness, uniqueness, etc.) and the OpenLineage specification (Job, Run, Dataset as first-class entities).

The normalized approach mirrors how products like OpenMetadata and Great Expectations structure their metadata stores: separate tables for data sources, datasets, columns, check definitions, check executions, and metric results. This yields maximum referential integrity, straightforward SQL queries for reporting, and clean migration paths as the schema evolves.

This architecture is best for teams that prioritise data integrity, complex cross-entity reporting (e.g., "show me all checks that failed across all datasets owned by team X in the last 7 days"), and regulatory environments where audit queries must be simple and verifiable.

**Best for:** Regulated environments (GDPR, HIPAA, BCBS 239) where referential integrity, audit-ready reporting, and query simplicity outweigh schema flexibility.

**Trade-offs:**
- (+) Strong referential integrity via foreign keys; database enforces consistency
- (+) Straightforward SQL for all reporting and analytics queries
- (+) Clean alignment with ISO 25012 dimensions and OpenLineage entities
- (+) Standard migration tooling (Alembic, Flyway) works well with normalized schemas
- (-) Higher table count (~35-40 tables) increases JOIN complexity for some queries
- (-) Adding new check types or metric dimensions requires schema migrations
- (-) Multi-jurisdiction or domain-specific variations require new columns or tables, not just config changes
- (-) Performance can degrade on high-cardinality metric queries without careful indexing and partitioning

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO/IEC 25012 (SQuaRE) | The 15 quality characteristics map to the `quality_dimension` enum used in `check_definitions` and `metric_results` |
| ISO/IEC 25024 | Measurement functions defined in this standard inform the `metric_type` and computation logic for each quality dimension |
| ISO 8000 | Vocabulary and data quality management roles inform the `quality_dimension` reference table and team/ownership model |
| OpenLineage | Core entities (Job, Run, Dataset) map directly to `data_sources`, `datasets`, `scan_jobs`, and `scan_runs` |
| W3C PROV | Provenance model (Entity/Activity/Agent) informs the `scan_runs` (Activity), `datasets` (Entity), `users` (Agent) relationship pattern |
| JSON Schema (Draft 2020-12) | Used for validating check configuration parameters stored in the `parameters` column |
| Open Data Contract Standard (ODCS) | Data contracts table structure follows the ODCS YAML schema concepts |
| OAuth 2.0 (RFC 6749) | API key and token tables support both service tokens and OAuth flows |

---

## Organisation & Tenancy

```sql
-- ============================================================
-- ORGANISATION & TENANCY
-- ============================================================

CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',  -- free, team, enterprise
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255),
    role            VARCHAR(50) NOT NULL DEFAULT 'viewer',  -- owner, admin, editor, viewer
    auth_provider   VARCHAR(50),  -- local, google, github, saml
    auth_subject    VARCHAR(255), -- external identity provider subject ID
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(org_id, email)
);

CREATE INDEX idx_users_org_id ON users(org_id);
CREATE INDEX idx_users_email ON users(email);

CREATE TABLE api_keys (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    user_id         UUID REFERENCES users(id) ON DELETE SET NULL,  -- NULL for service keys
    name            VARCHAR(255) NOT NULL,
    key_prefix      VARCHAR(8) NOT NULL,   -- first 8 chars for identification
    key_hash        VARCHAR(128) NOT NULL,  -- bcrypt hash of the full key
    scopes          TEXT[] NOT NULL DEFAULT '{}',  -- read, write, admin
    expires_at      TIMESTAMPTZ,
    last_used_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_api_keys_org_id ON api_keys(org_id);
CREATE INDEX idx_api_keys_key_prefix ON api_keys(key_prefix);

CREATE TABLE teams (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(org_id, name)
);

CREATE TABLE team_members (
    team_id         UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',  -- lead, member
    PRIMARY KEY (team_id, user_id)
);
```

---

## Data Source & Dataset Registry

```sql
-- ============================================================
-- DATA SOURCE & DATASET REGISTRY
-- ============================================================

CREATE TABLE data_sources (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    source_type     VARCHAR(50) NOT NULL,  -- snowflake, bigquery, redshift, databricks, postgres
    connection_config JSONB NOT NULL,       -- encrypted connection parameters
    -- Example connection_config for Snowflake:
    -- {
    --   "account": "xy12345.us-east-1",
    --   "warehouse": "COMPUTE_WH",
    --   "database": "ANALYTICS",
    --   "role": "DATA_QUALITY_ROLE",
    --   "credentials_ref": "vault://snowflake/analytics"
    -- }
    status          VARCHAR(50) NOT NULL DEFAULT 'active',  -- active, inactive, error
    last_synced_at  TIMESTAMPTZ,
    owner_team_id   UUID REFERENCES teams(id) ON DELETE SET NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(org_id, name)
);

CREATE INDEX idx_data_sources_org_id ON data_sources(org_id);

CREATE TABLE datasets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_source_id  UUID NOT NULL REFERENCES data_sources(id) ON DELETE CASCADE,
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    schema_name     VARCHAR(255),          -- database schema (e.g., "public", "analytics")
    table_name      VARCHAR(255) NOT NULL,  -- physical table/view name
    dataset_type    VARCHAR(50) NOT NULL DEFAULT 'table',  -- table, view, external
    description     TEXT,
    tags            TEXT[] DEFAULT '{}',
    owner_team_id   UUID REFERENCES teams(id) ON DELETE SET NULL,
    row_count       BIGINT,                -- last known row count
    size_bytes      BIGINT,                -- last known size
    last_profiled_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(data_source_id, schema_name, table_name)
);

CREATE INDEX idx_datasets_org_id ON datasets(org_id);
CREATE INDEX idx_datasets_data_source_id ON datasets(data_source_id);
CREATE INDEX idx_datasets_tags ON datasets USING GIN(tags);

CREATE TABLE dataset_columns (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dataset_id      UUID NOT NULL REFERENCES datasets(id) ON DELETE CASCADE,
    column_name     VARCHAR(255) NOT NULL,
    ordinal_position INTEGER NOT NULL,
    data_type       VARCHAR(100) NOT NULL,  -- varchar, integer, timestamp, etc.
    is_nullable     BOOLEAN NOT NULL DEFAULT true,
    is_primary_key  BOOLEAN NOT NULL DEFAULT false,
    description     TEXT,
    semantic_type   VARCHAR(100),  -- email, phone, currency, percentage, identifier, etc.
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(dataset_id, column_name)
);

CREATE INDEX idx_dataset_columns_dataset_id ON dataset_columns(dataset_id);
```

---

## Schema History & Drift Detection

```sql
-- ============================================================
-- SCHEMA HISTORY & DRIFT DETECTION
-- ============================================================

CREATE TABLE schema_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dataset_id      UUID NOT NULL REFERENCES datasets(id) ON DELETE CASCADE,
    snapshot_hash   VARCHAR(64) NOT NULL,   -- SHA-256 of the canonical column list
    columns_json    JSONB NOT NULL,          -- full column list at this point in time
    -- Example columns_json:
    -- [
    --   {"name": "id", "type": "bigint", "nullable": false, "position": 1},
    --   {"name": "email", "type": "varchar(255)", "nullable": true, "position": 2}
    -- ]
    captured_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_schema_snapshots_dataset_id ON schema_snapshots(dataset_id);
CREATE INDEX idx_schema_snapshots_captured_at ON schema_snapshots(dataset_id, captured_at DESC);

CREATE TABLE schema_changes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dataset_id      UUID NOT NULL REFERENCES datasets(id) ON DELETE CASCADE,
    previous_snapshot_id UUID REFERENCES schema_snapshots(id),
    current_snapshot_id  UUID NOT NULL REFERENCES schema_snapshots(id),
    change_type     VARCHAR(50) NOT NULL,   -- column_added, column_removed, type_changed, nullable_changed
    column_name     VARCHAR(255) NOT NULL,
    previous_value  VARCHAR(255),           -- e.g., old data type
    current_value   VARCHAR(255),           -- e.g., new data type
    detected_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    severity        VARCHAR(20) NOT NULL DEFAULT 'warning'  -- info, warning, critical
);

CREATE INDEX idx_schema_changes_dataset_id ON schema_changes(dataset_id);
CREATE INDEX idx_schema_changes_detected_at ON schema_changes(detected_at);
```

---

## Quality Dimensions & Check Definitions

```sql
-- ============================================================
-- QUALITY DIMENSIONS (ISO/IEC 25012 aligned)
-- ============================================================

CREATE TYPE quality_dimension AS ENUM (
    'completeness',    -- absence of missing values
    'accuracy',        -- correctness of values against a reference
    'consistency',     -- absence of contradictions within or across datasets
    'currentness',     -- freshness / timeliness of data
    'uniqueness',      -- absence of duplicates
    'validity',        -- conformance to defined formats, ranges, patterns
    'precision',       -- granularity of data values
    'traceability',    -- ability to track data provenance
    'understandability', -- clarity of data meaning
    'compliance',      -- adherence to regulations or standards
    'availability',    -- data is accessible when needed
    'portability',     -- data can be moved between systems
    'recoverability',  -- data can be restored after failure
    'confidentiality', -- data is protected from unauthorised access
    'efficiency'       -- data can be processed within time/resource constraints
);

-- ============================================================
-- CHECK DEFINITIONS
-- ============================================================

CREATE TABLE check_definitions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    check_type      VARCHAR(50) NOT NULL,   -- rule_based, ml_anomaly, schema_drift, freshness, volume
    dimension       quality_dimension NOT NULL,
    scope           VARCHAR(20) NOT NULL,   -- table, column
    template_id     UUID REFERENCES check_templates(id),
    parameters      JSONB NOT NULL DEFAULT '{}',
    -- Example parameters for a completeness check:
    -- {
    --   "column": "email",
    --   "operator": "less_than",
    --   "threshold": 0.05,
    --   "description": "Email null rate must be below 5%"
    -- }
    severity        VARCHAR(20) NOT NULL DEFAULT 'warning',  -- info, warning, critical
    is_ai_generated BOOLEAN NOT NULL DEFAULT false,
    created_by      UUID REFERENCES users(id) ON DELETE SET NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_check_definitions_org_id ON check_definitions(org_id);
CREATE INDEX idx_check_definitions_dimension ON check_definitions(dimension);
CREATE INDEX idx_check_definitions_check_type ON check_definitions(check_type);

CREATE TABLE check_templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL UNIQUE,
    description     TEXT,
    check_type      VARCHAR(50) NOT NULL,
    dimension       quality_dimension NOT NULL,
    scope           VARCHAR(20) NOT NULL,
    default_parameters JSONB NOT NULL DEFAULT '{}',
    parameter_schema JSONB NOT NULL,  -- JSON Schema defining valid parameter structure
    -- Example parameter_schema:
    -- {
    --   "type": "object",
    --   "properties": {
    --     "threshold": {"type": "number", "minimum": 0, "maximum": 1},
    --     "operator": {"type": "string", "enum": ["less_than", "greater_than", "equal"]}
    --   },
    --   "required": ["threshold"]
    -- }
    is_builtin      BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Junction table: which checks are applied to which datasets/columns
CREATE TABLE dataset_checks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dataset_id      UUID NOT NULL REFERENCES datasets(id) ON DELETE CASCADE,
    column_id       UUID REFERENCES dataset_columns(id) ON DELETE CASCADE,  -- NULL for table-level checks
    check_def_id    UUID NOT NULL REFERENCES check_definitions(id) ON DELETE CASCADE,
    is_enabled      BOOLEAN NOT NULL DEFAULT true,
    schedule_cron   VARCHAR(100),  -- cron expression for check schedule; NULL = inherited from scan job
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(dataset_id, column_id, check_def_id)
);

CREATE INDEX idx_dataset_checks_dataset_id ON dataset_checks(dataset_id);
CREATE INDEX idx_dataset_checks_check_def_id ON dataset_checks(check_def_id);
```

---

## Scan Execution & Results

```sql
-- ============================================================
-- SCAN EXECUTION (OpenLineage Job/Run aligned)
-- ============================================================

CREATE TABLE scan_jobs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    schedule_cron   VARCHAR(100),          -- cron expression
    data_source_id  UUID REFERENCES data_sources(id) ON DELETE SET NULL,
    dataset_filter  JSONB DEFAULT '{}',    -- filter which datasets to include
    is_enabled      BOOLEAN NOT NULL DEFAULT true,
    created_by      UUID REFERENCES users(id) ON DELETE SET NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_scan_jobs_org_id ON scan_jobs(org_id);

CREATE TABLE scan_runs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scan_job_id     UUID NOT NULL REFERENCES scan_jobs(id) ON DELETE CASCADE,
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',  -- pending, running, completed, failed, cancelled
    trigger_type    VARCHAR(20) NOT NULL DEFAULT 'scheduled', -- scheduled, manual, ci_cd, webhook
    triggered_by    UUID REFERENCES users(id) ON DELETE SET NULL,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    duration_ms     INTEGER,
    checks_total    INTEGER DEFAULT 0,
    checks_passed   INTEGER DEFAULT 0,
    checks_warned   INTEGER DEFAULT 0,
    checks_failed   INTEGER DEFAULT 0,
    checks_errored  INTEGER DEFAULT 0,
    error_message   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_scan_runs_scan_job_id ON scan_runs(scan_job_id);
CREATE INDEX idx_scan_runs_org_id_status ON scan_runs(org_id, status);
CREATE INDEX idx_scan_runs_started_at ON scan_runs(started_at DESC);

-- ============================================================
-- CHECK RESULTS
-- ============================================================

CREATE TABLE check_results (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scan_run_id     UUID NOT NULL REFERENCES scan_runs(id) ON DELETE CASCADE,
    dataset_check_id UUID NOT NULL REFERENCES dataset_checks(id) ON DELETE CASCADE,
    dataset_id      UUID NOT NULL REFERENCES datasets(id),
    column_id       UUID REFERENCES dataset_columns(id),
    status          VARCHAR(20) NOT NULL,  -- passed, warned, failed, errored
    observed_value  DOUBLE PRECISION,      -- the measured metric value
    expected_value  DOUBLE PRECISION,      -- the threshold or expected value
    deviation       DOUBLE PRECISION,      -- (observed - expected) / expected
    row_count       BIGINT,                -- rows scanned
    failing_rows    BIGINT,                -- rows that violated the check
    details         JSONB DEFAULT '{}',    -- additional context
    executed_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_check_results_scan_run_id ON check_results(scan_run_id);
CREATE INDEX idx_check_results_dataset_check_id ON check_results(dataset_check_id);
CREATE INDEX idx_check_results_dataset_id ON check_results(dataset_id);
CREATE INDEX idx_check_results_status ON check_results(status);
CREATE INDEX idx_check_results_executed_at ON check_results(executed_at DESC);
```

---

## Metric Time Series

```sql
-- ============================================================
-- METRIC TIME SERIES
-- ============================================================

CREATE TABLE metric_results (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scan_run_id     UUID NOT NULL REFERENCES scan_runs(id) ON DELETE CASCADE,
    dataset_id      UUID NOT NULL REFERENCES datasets(id),
    column_id       UUID REFERENCES dataset_columns(id),  -- NULL for table-level metrics
    metric_type     VARCHAR(50) NOT NULL,  -- row_count, null_rate, distinct_count, mean, stddev, min, max, freshness_seconds
    dimension       quality_dimension NOT NULL,
    value           DOUBLE PRECISION NOT NULL,
    measured_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_metric_results_dataset_id ON metric_results(dataset_id);
CREATE INDEX idx_metric_results_column_metric ON metric_results(dataset_id, column_id, metric_type);
CREATE INDEX idx_metric_results_measured_at ON metric_results(measured_at DESC);

-- ============================================================
-- ANOMALY DETECTION BASELINES
-- ============================================================

CREATE TABLE metric_baselines (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dataset_id      UUID NOT NULL REFERENCES datasets(id) ON DELETE CASCADE,
    column_id       UUID REFERENCES dataset_columns(id),
    metric_type     VARCHAR(50) NOT NULL,
    baseline_mean   DOUBLE PRECISION NOT NULL,
    baseline_stddev DOUBLE PRECISION NOT NULL,
    baseline_median DOUBLE PRECISION,
    baseline_mad    DOUBLE PRECISION,  -- Median Absolute Deviation
    sample_count    INTEGER NOT NULL,
    window_days     INTEGER NOT NULL DEFAULT 30,
    last_updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(dataset_id, column_id, metric_type)
);

CREATE INDEX idx_metric_baselines_dataset_id ON metric_baselines(dataset_id);
```

---

## Alerting & Incidents

```sql
-- ============================================================
-- ALERTING & INCIDENTS
-- ============================================================

CREATE TABLE alert_channels (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    channel_type    VARCHAR(50) NOT NULL,  -- slack, pagerduty, email, webhook, teams
    config          JSONB NOT NULL,         -- channel-specific configuration
    -- Example config for Slack:
    -- {"webhook_url": "https://hooks.slack.com/...", "channel": "#data-quality"}
    is_enabled      BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_alert_channels_org_id ON alert_channels(org_id);

CREATE TABLE alert_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    condition_type  VARCHAR(50) NOT NULL,  -- check_failed, anomaly_detected, freshness_sla_breach
    min_severity    VARCHAR(20) NOT NULL DEFAULT 'warning',
    dataset_filter  JSONB DEFAULT '{}',
    channel_id      UUID NOT NULL REFERENCES alert_channels(id) ON DELETE CASCADE,
    is_enabled      BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE alerts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    alert_rule_id   UUID REFERENCES alert_rules(id) ON DELETE SET NULL,
    check_result_id UUID REFERENCES check_results(id) ON DELETE SET NULL,
    incident_id     UUID REFERENCES incidents(id) ON DELETE SET NULL,
    severity        VARCHAR(20) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    ai_explanation  TEXT,  -- LLM-generated natural-language explanation
    status          VARCHAR(20) NOT NULL DEFAULT 'open',  -- open, acknowledged, resolved, suppressed
    notified_at     TIMESTAMPTZ,
    acknowledged_at TIMESTAMPTZ,
    acknowledged_by UUID REFERENCES users(id) ON DELETE SET NULL,
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_alerts_org_id_status ON alerts(org_id, status);
CREATE INDEX idx_alerts_incident_id ON alerts(incident_id);
CREATE INDEX idx_alerts_created_at ON alerts(created_at DESC);

CREATE TABLE incidents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    severity        VARCHAR(20) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'open',  -- open, investigating, resolved, closed
    assigned_to     UUID REFERENCES users(id) ON DELETE SET NULL,
    assigned_team   UUID REFERENCES teams(id) ON DELETE SET NULL,
    root_cause      TEXT,
    resolution      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at     TIMESTAMPTZ
);

CREATE INDEX idx_incidents_org_id_status ON incidents(org_id, status);

CREATE TABLE incident_comments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    incident_id     UUID NOT NULL REFERENCES incidents(id) ON DELETE CASCADE,
    user_id         UUID REFERENCES users(id) ON DELETE SET NULL,
    body            TEXT NOT NULL,
    is_system       BOOLEAN NOT NULL DEFAULT false,  -- system-generated comment
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_incident_comments_incident_id ON incident_comments(incident_id);
```

---

## Data Lineage

```sql
-- ============================================================
-- DATA LINEAGE (OpenLineage aligned)
-- ============================================================

CREATE TABLE lineage_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    source_dataset_id  UUID NOT NULL REFERENCES datasets(id) ON DELETE CASCADE,
    target_dataset_id  UUID NOT NULL REFERENCES datasets(id) ON DELETE CASCADE,
    edge_type       VARCHAR(50) NOT NULL DEFAULT 'transforms',  -- transforms, copies, aggregates, joins
    job_name        VARCHAR(255),          -- the ETL/dbt job that creates this relationship
    discovered_via  VARCHAR(50),           -- query_log, dbt_manifest, openlineage, manual
    confidence      DOUBLE PRECISION DEFAULT 1.0,  -- 0.0-1.0; lower for inferred relationships
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(source_dataset_id, target_dataset_id, job_name)
);

CREATE INDEX idx_lineage_edges_source ON lineage_edges(source_dataset_id);
CREATE INDEX idx_lineage_edges_target ON lineage_edges(target_dataset_id);
CREATE INDEX idx_lineage_edges_org_id ON lineage_edges(org_id);

CREATE TABLE column_lineage_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    lineage_edge_id UUID NOT NULL REFERENCES lineage_edges(id) ON DELETE CASCADE,
    source_column_id UUID NOT NULL REFERENCES dataset_columns(id) ON DELETE CASCADE,
    target_column_id UUID NOT NULL REFERENCES dataset_columns(id) ON DELETE CASCADE,
    transformation  VARCHAR(50),  -- direct, cast, aggregate, expression
    expression      TEXT,         -- SQL expression if available
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_column_lineage_source ON column_lineage_edges(source_column_id);
CREATE INDEX idx_column_lineage_target ON column_lineage_edges(target_column_id);
```

---

## Data Contracts

```sql
-- ============================================================
-- DATA CONTRACTS (ODCS aligned)
-- ============================================================

CREATE TABLE data_contracts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    dataset_id      UUID NOT NULL REFERENCES datasets(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    version         VARCHAR(50) NOT NULL DEFAULT '1.0.0',  -- semver
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',  -- draft, active, deprecated
    producer_team_id UUID REFERENCES teams(id) ON DELETE SET NULL,
    schema_spec     JSONB NOT NULL,  -- expected column definitions
    quality_checks  JSONB NOT NULL DEFAULT '[]',  -- quality SLAs embedded in contract
    sla_freshness_minutes INTEGER,
    sla_completeness_pct  DOUBLE PRECISION,  -- e.g., 99.5
    created_by      UUID REFERENCES users(id) ON DELETE SET NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_data_contracts_dataset_id ON data_contracts(dataset_id);
CREATE INDEX idx_data_contracts_org_id ON data_contracts(org_id);

CREATE TABLE contract_consumers (
    contract_id     UUID NOT NULL REFERENCES data_contracts(id) ON DELETE CASCADE,
    consumer_team_id UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    subscribed_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (contract_id, consumer_team_id)
);
```

---

## Audit Log

```sql
-- ============================================================
-- AUDIT LOG
-- ============================================================

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    user_id         UUID REFERENCES users(id) ON DELETE SET NULL,
    action          VARCHAR(100) NOT NULL,  -- check.created, scan.triggered, alert.acknowledged, etc.
    resource_type   VARCHAR(100) NOT NULL,  -- check_definition, scan_job, dataset, alert, incident
    resource_id     UUID NOT NULL,
    changes         JSONB DEFAULT '{}',     -- before/after diff
    ip_address      INET,
    user_agent      VARCHAR(500),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_log_org_id ON audit_log(org_id);
CREATE INDEX idx_audit_log_resource ON audit_log(resource_type, resource_id);
CREATE INDEX idx_audit_log_created_at ON audit_log(created_at DESC);
```

---

## Example Queries

```sql
-- All failed checks for a dataset in the last 24 hours
SELECT cr.*, cd.name AS check_name, cd.dimension, dc.column_name
FROM check_results cr
JOIN dataset_checks dck ON cr.dataset_check_id = dck.id
JOIN check_definitions cd ON dck.check_def_id = cd.id
LEFT JOIN dataset_columns dc ON cr.column_id = dc.id
WHERE cr.dataset_id = '...'
  AND cr.status = 'failed'
  AND cr.executed_at > now() - INTERVAL '24 hours'
ORDER BY cr.executed_at DESC;

-- Upstream lineage traversal (recursive CTE)
WITH RECURSIVE upstream AS (
    SELECT source_dataset_id, target_dataset_id, 1 AS depth
    FROM lineage_edges
    WHERE target_dataset_id = '<failing-dataset-id>'
  UNION ALL
    SELECT le.source_dataset_id, le.target_dataset_id, u.depth + 1
    FROM lineage_edges le
    JOIN upstream u ON le.target_dataset_id = u.source_dataset_id
    WHERE u.depth < 10
)
SELECT DISTINCT d.schema_name, d.table_name, u.depth
FROM upstream u
JOIN datasets d ON u.source_dataset_id = d.id
ORDER BY u.depth;

-- Quality score by dimension across an organisation
SELECT cd.dimension,
       COUNT(*) AS total_checks,
       COUNT(*) FILTER (WHERE cr.status = 'passed') AS passed,
       ROUND(100.0 * COUNT(*) FILTER (WHERE cr.status = 'passed') / COUNT(*), 1) AS pass_rate_pct
FROM check_results cr
JOIN dataset_checks dck ON cr.dataset_check_id = dck.id
JOIN check_definitions cd ON dck.check_def_id = cd.id
WHERE cr.scan_run_id = '<latest-run-id>'
GROUP BY cd.dimension
ORDER BY pass_rate_pct;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation & Tenancy | 5 | organisations, users, api_keys, teams, team_members |
| Data Source & Registry | 3 | data_sources, datasets, dataset_columns |
| Schema History | 2 | schema_snapshots, schema_changes |
| Check Definitions | 3 | check_templates, check_definitions, dataset_checks |
| Scan Execution | 2 | scan_jobs, scan_runs |
| Results & Metrics | 3 | check_results, metric_results, metric_baselines |
| Alerting & Incidents | 5 | alert_channels, alert_rules, alerts, incidents, incident_comments |
| Data Lineage | 2 | lineage_edges, column_lineage_edges |
| Data Contracts | 2 | data_contracts, contract_consumers |
| Audit | 1 | audit_log |
| **Total** | **28** | |

---

## Key Design Decisions

1. **ISO/IEC 25012 as the quality dimension taxonomy.** The `quality_dimension` enum directly encodes all 15 SQuaRE characteristics, ensuring every check and metric is categorised using an internationally recognised vocabulary. This makes compliance reporting straightforward and enables cross-organisation benchmarking.

2. **Separate check_definitions from dataset_checks.** Check definitions are reusable templates parameterised per dataset/column via the junction table `dataset_checks`. This avoids duplicating check logic when the same null-rate check is applied to 500 columns.

3. **OpenLineage-aligned entity structure.** `data_sources` maps to OpenLineage namespaces, `datasets` to OpenLineage Datasets, `scan_jobs` to Jobs, and `scan_runs` to Runs. This makes bidirectional OpenLineage event emission and consumption natural.

4. **Metric baselines stored separately from raw metrics.** The `metric_baselines` table holds rolling statistical summaries (mean, stddev, median, MAD) used for anomaly scoring. Raw metric values in `metric_results` are never mutated; baselines are recomputed periodically.

5. **Schema snapshots for drift detection.** Rather than diff-on-the-fly, the model captures full schema snapshots and materialises individual changes into `schema_changes`. This supports historical queries ("what did the schema look like on March 15?") and makes alerting on specific change types efficient.

6. **Multi-tenant via org_id foreign keys.** Every tenant-scoped table includes `org_id` with a foreign key to `organisations`. Row-Level Security policies can be added atop this for defence-in-depth: `CREATE POLICY org_isolation ON datasets USING (org_id = current_setting('app.current_org_id')::uuid)`.

7. **Alert-to-incident grouping.** Multiple related alerts (e.g., three checks failing on the same table) can be grouped into a single incident for triage, mirroring the Monte Carlo incident management workflow.

8. **Audit log as a simple append-only table.** For this normalized model, the audit log captures who-did-what with before/after diffs. It is not the source of truth (unlike the event-sourced model in Suggestion 2) but provides regulatory audit coverage for GDPR, HIPAA, and BCBS 239.

9. **Data contracts follow ODCS structure.** The `data_contracts` table captures producer/consumer agreements with embedded schema specs and quality SLAs, enabling "data quality as a contract" workflows between teams.

10. **Foreign keys everywhere.** Every relationship is enforced at the database level. This adds overhead on writes but guarantees referential integrity — critical for a platform whose purpose is ensuring data quality.
