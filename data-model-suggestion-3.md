# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Data Quality Monitor · Created: 2026-05-11

## Philosophy

This model takes a pragmatic middle path: core structural relationships (organisations, datasets, check executions) are stored in properly normalised relational tables with foreign keys, while variable, domain-specific, and rapidly evolving data (check parameters, metric details, connection configurations, AI suggestions) are stored in JSONB columns. This "relational spine with JSONB wings" approach is the architecture used by products like Soda Cloud (SodaCL checks stored as YAML/JSON), Bigeye (Bigconfig as YAML/JSON), and OpenMetadata (JSON Schema-typed entity metadata).

The key insight is that a data quality monitor must support dozens of check types, each with different parameters, across multiple warehouse types with different connection configs, and produce results with varying structures. A fully normalised model would require either an unwieldy number of tables (one per check type) or a narrow/wide-table antipattern. JSONB columns absorb this variability while PostgreSQL's GIN indexes, containment operators (`@>`), and JSON path queries keep the variable data queryable and performant.

This approach optimises for development velocity and schema flexibility. New check types, metric types, and integration configurations can be added without database migrations — just new JSON structures validated by JSON Schema at the application layer. This makes it the ideal architecture for an MVP that needs to iterate rapidly while serving multiple warehouse backends and check paradigms.

**Best for:** MVP and growth-stage development where rapid iteration, multi-warehouse support, and flexible check definitions matter more than strict normalisation.

**Trade-offs:**
- (+) Fastest development velocity: new features often require zero schema migrations
- (+) Handles variability naturally: 50+ check types with different parameters, 6+ warehouse connection formats
- (+) PostgreSQL JSONB is battle-tested: GIN indexes, containment queries, JSON path expressions
- (+) Moderate table count (~18-20) keeps the schema comprehensible
- (+) JSON Schema validation at the application layer provides type safety without database rigidity
- (-) JSONB columns lack database-enforced referential integrity; consistency depends on application logic
- (-) Complex queries on deeply nested JSONB can be slower than joins on indexed relational columns
- (-) Schema evolution of JSONB structures requires application-level migration logic, not standard SQL ALTER TABLE
- (-) Reporting queries may need JSONB extraction functions (->>, jsonb_array_elements) that are less readable than plain SELECTs
- (-) Data warehouse teams accustomed to strict schemas may find JSONB columns less transparent

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO/IEC 25012 (SQuaRE) | Quality dimensions stored as a VARCHAR enum field on check and metric rows, not embedded in JSONB |
| JSON Schema (Draft 2020-12) | Every JSONB column has a corresponding JSON Schema definition enforced at the application layer; check parameter schemas are stored in `check_templates.parameter_schema` |
| SodaCL | Check definition YAML structure informs the `config` JSONB format for rule-based checks |
| Great Expectations | Expectation suite JSON format supported via import/export against the `config` JSONB field |
| OpenLineage | Lineage events stored with JSONB facets matching the OpenLineage facet schema |
| Open Data Contract Standard | Data contract specs stored as JSONB documents following ODCS v3 structure |
| OAuth 2.0 (RFC 6749) | Authentication config in JSONB allows different auth mechanisms per data source |

---

## Organisation & Tenancy

```sql
-- ============================================================
-- ORGANISATION & ACCESS
-- ============================================================

CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    -- Example settings:
    -- {
    --   "default_severity": "warning",
    --   "retention_days": 90,
    --   "ai_features_enabled": true,
    --   "default_schedule": "0 */6 * * *",
    --   "allowed_domains": ["acme.com"]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255),
    role            VARCHAR(50) NOT NULL DEFAULT 'viewer',
    profile         JSONB NOT NULL DEFAULT '{}',
    -- Example profile:
    -- {
    --   "auth_provider": "google",
    --   "avatar_url": "https://...",
    --   "notification_preferences": {
    --     "slack_dm": true, "email_digest": "daily"
    --   },
    --   "teams": ["data-engineering", "analytics"]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(org_id, email)
);

CREATE INDEX idx_users_org_id ON users(org_id);
```

---

## Data Sources & Datasets

```sql
-- ============================================================
-- DATA SOURCES
-- ============================================================

CREATE TABLE data_sources (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    source_type     VARCHAR(50) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'active',
    config          JSONB NOT NULL,
    -- The config JSONB absorbs all warehouse-specific connection details:
    --
    -- Snowflake example:
    -- {
    --   "account": "xy12345.us-east-1",
    --   "warehouse": "COMPUTE_WH",
    --   "database": "ANALYTICS",
    --   "role": "DQ_ROLE",
    --   "credentials": {"type": "key_pair", "vault_ref": "vault://snowflake/prod"}
    -- }
    --
    -- BigQuery example:
    -- {
    --   "project_id": "acme-analytics-prod",
    --   "dataset": "raw_events",
    --   "credentials": {"type": "service_account", "vault_ref": "vault://gcp/sa-key"}
    -- }
    --
    -- Redshift example:
    -- {
    --   "host": "acme-cluster.abc123.us-east-1.redshift.amazonaws.com",
    --   "port": 5439,
    --   "database": "analytics",
    --   "credentials": {"type": "iam_role", "role_arn": "arn:aws:iam::role/dq-reader"}
    -- }
    sync_state      JSONB NOT NULL DEFAULT '{}',
    -- Example sync_state:
    -- {
    --   "last_synced_at": "2026-05-10T14:30:00Z",
    --   "tables_discovered": 142,
    --   "sync_duration_ms": 8500,
    --   "error": null
    -- }
    owner_info      JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(org_id, name)
);

CREATE INDEX idx_data_sources_org_id ON data_sources(org_id);

-- ============================================================
-- DATASETS
-- ============================================================

CREATE TABLE datasets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_source_id  UUID NOT NULL REFERENCES data_sources(id) ON DELETE CASCADE,
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    schema_name     VARCHAR(255),
    table_name      VARCHAR(255) NOT NULL,
    dataset_type    VARCHAR(50) NOT NULL DEFAULT 'table',
    tags            TEXT[] DEFAULT '{}',
    profile         JSONB NOT NULL DEFAULT '{}',
    -- Profile captures the latest profiling results:
    -- {
    --   "row_count": 1500000,
    --   "size_bytes": 420000000,
    --   "last_profiled_at": "2026-05-10T14:35:00Z",
    --   "columns": [
    --     {
    --       "name": "id",
    --       "type": "bigint",
    --       "nullable": false,
    --       "is_pk": true,
    --       "ordinal": 1,
    --       "stats": {"distinct_count": 1500000, "null_rate": 0.0}
    --     },
    --     {
    --       "name": "email",
    --       "type": "varchar(255)",
    --       "nullable": true,
    --       "is_pk": false,
    --       "ordinal": 2,
    --       "semantic_type": "email",
    --       "stats": {"distinct_count": 1487320, "null_rate": 0.0034, "pattern_top": "^[a-z]+@[a-z]+\\.[a-z]+$"}
    --     },
    --     {
    --       "name": "created_at",
    --       "type": "timestamptz",
    --       "nullable": false,
    --       "is_pk": false,
    --       "ordinal": 3,
    --       "stats": {"min": "2020-01-15T00:00:00Z", "max": "2026-05-10T13:22:00Z"}
    --     }
    --   ],
    --   "freshness": {
    --     "last_updated_at": "2026-05-10T13:22:00Z",
    --     "update_frequency_hours": 1
    --   }
    -- }
    schema_history  JSONB NOT NULL DEFAULT '[]',
    -- Array of schema snapshots with change detection:
    -- [
    --   {
    --     "captured_at": "2026-05-10T14:35:00Z",
    --     "hash": "sha256:abc123...",
    --     "column_count": 12,
    --     "changes_from_previous": [
    --       {"type": "column_added", "column": "discount_pct", "data_type": "numeric"}
    --     ]
    --   }
    -- ]
    owner_info      JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(data_source_id, schema_name, table_name)
);

CREATE INDEX idx_datasets_org_id ON datasets(org_id);
CREATE INDEX idx_datasets_data_source_id ON datasets(data_source_id);
CREATE INDEX idx_datasets_tags ON datasets USING GIN(tags);
CREATE INDEX idx_datasets_profile ON datasets USING GIN(profile jsonb_path_ops);
```

---

## Check Definitions & Configuration

```sql
-- ============================================================
-- CHECK TEMPLATES (built-in and custom)
-- ============================================================

CREATE TABLE check_templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL UNIQUE,
    description     TEXT,
    check_type      VARCHAR(50) NOT NULL,   -- rule_based, ml_anomaly, schema_drift, freshness, volume, custom_sql
    dimension       VARCHAR(50) NOT NULL,    -- ISO 25012 dimension
    scope           VARCHAR(20) NOT NULL,    -- table, column
    default_config  JSONB NOT NULL DEFAULT '{}',
    parameter_schema JSONB NOT NULL,         -- JSON Schema (Draft 2020-12) for config validation
    -- Example parameter_schema for a null rate check:
    -- {
    --   "$schema": "https://json-schema.org/draft/2020-12/schema",
    --   "type": "object",
    --   "properties": {
    --     "column": {"type": "string"},
    --     "max_null_rate": {"type": "number", "minimum": 0, "maximum": 1},
    --     "severity": {"type": "string", "enum": ["info", "warning", "critical"]}
    --   },
    --   "required": ["column", "max_null_rate"]
    -- }
    is_builtin      BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- CHECK INSTANCES (applied to specific datasets)
-- ============================================================

CREATE TABLE checks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    dataset_id      UUID NOT NULL REFERENCES datasets(id) ON DELETE CASCADE,
    template_id     UUID REFERENCES check_templates(id),
    name            VARCHAR(255) NOT NULL,
    check_type      VARCHAR(50) NOT NULL,
    dimension       VARCHAR(50) NOT NULL,
    scope           VARCHAR(20) NOT NULL,
    config          JSONB NOT NULL,
    -- The config JSONB holds all check-type-specific parameters:
    --
    -- Rule-based null rate check:
    -- {
    --   "column": "email",
    --   "max_null_rate": 0.05,
    --   "severity": "critical"
    -- }
    --
    -- ML anomaly detection:
    -- {
    --   "column": "order_total",
    --   "metric": "mean",
    --   "sensitivity": 3.0,
    --   "min_samples": 30,
    --   "algorithm": "mad"
    -- }
    --
    -- Freshness check:
    -- {
    --   "timestamp_column": "updated_at",
    --   "max_age_hours": 2,
    --   "severity": "critical"
    -- }
    --
    -- Volume check:
    -- {
    --   "expected_range": {"min": 40000, "max": 60000},
    --   "comparison": "7d_rolling_avg",
    --   "deviation_pct": 20
    -- }
    --
    -- Custom SQL:
    -- {
    --   "sql": "SELECT COUNT(*) FROM orders WHERE total < 0",
    --   "expected": 0,
    --   "operator": "equals"
    -- }
    --
    -- Schema drift:
    -- {
    --   "alert_on": ["column_removed", "type_changed"],
    --   "ignore_columns": ["_dbt_updated_at"]
    -- }
    is_enabled      BOOLEAN NOT NULL DEFAULT true,
    is_ai_generated BOOLEAN NOT NULL DEFAULT false,
    ai_metadata     JSONB DEFAULT NULL,
    -- AI generation metadata (when is_ai_generated = true):
    -- {
    --   "model": "claude-opus-4-6",
    --   "confidence": 0.92,
    --   "reasoning": "Column 'email' has VARCHAR type and 'email' in the name, suggesting it should have low null rates",
    --   "generated_at": "2026-05-10T14:00:00Z",
    --   "accepted_by": "user-uuid",
    --   "accepted_at": "2026-05-10T14:05:00Z"
    -- }
    schedule_cron   VARCHAR(100),
    severity        VARCHAR(20) NOT NULL DEFAULT 'warning',
    created_by      UUID REFERENCES users(id) ON DELETE SET NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_checks_org_id ON checks(org_id);
CREATE INDEX idx_checks_dataset_id ON checks(dataset_id);
CREATE INDEX idx_checks_type ON checks(check_type);
CREATE INDEX idx_checks_config ON checks USING GIN(config jsonb_path_ops);
```

---

## Scan Execution & Results

```sql
-- ============================================================
-- SCAN JOBS & RUNS
-- ============================================================

CREATE TABLE scan_jobs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    schedule_cron   VARCHAR(100),
    config          JSONB NOT NULL DEFAULT '{}',
    -- Job-level configuration:
    -- {
    --   "data_source_ids": ["..."],
    --   "dataset_filter": {"tags": ["production"], "schemas": ["public", "analytics"]},
    --   "parallelism": 4,
    --   "timeout_seconds": 300
    -- }
    is_enabled      BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_scan_jobs_org_id ON scan_jobs(org_id);

CREATE TABLE scan_runs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scan_job_id     UUID NOT NULL REFERENCES scan_jobs(id) ON DELETE CASCADE,
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    trigger_type    VARCHAR(20) NOT NULL DEFAULT 'scheduled',
    triggered_by    UUID REFERENCES users(id) ON DELETE SET NULL,
    summary         JSONB NOT NULL DEFAULT '{}',
    -- Summary is populated on completion:
    -- {
    --   "checks_total": 48,
    --   "checks_passed": 44,
    --   "checks_warned": 2,
    --   "checks_failed": 1,
    --   "checks_errored": 1,
    --   "datasets_scanned": 12,
    --   "duration_ms": 45000,
    --   "error": null
    -- }
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_scan_runs_scan_job_id ON scan_runs(scan_job_id);
CREATE INDEX idx_scan_runs_org_id ON scan_runs(org_id);
CREATE INDEX idx_scan_runs_started_at ON scan_runs(started_at DESC);

-- ============================================================
-- CHECK RESULTS
-- ============================================================

CREATE TABLE check_results (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scan_run_id     UUID NOT NULL REFERENCES scan_runs(id) ON DELETE CASCADE,
    check_id        UUID NOT NULL REFERENCES checks(id) ON DELETE CASCADE,
    org_id          UUID NOT NULL REFERENCES organisations(id),
    dataset_id      UUID NOT NULL REFERENCES datasets(id),
    status          VARCHAR(20) NOT NULL,  -- passed, warned, failed, errored
    result          JSONB NOT NULL,
    -- Result captures all check-type-specific output:
    --
    -- Rule-based null rate result:
    -- {
    --   "column": "email",
    --   "observed_null_rate": 0.034,
    --   "threshold": 0.05,
    --   "row_count": 1500000,
    --   "null_count": 51000,
    --   "passed": true
    -- }
    --
    -- ML anomaly result:
    -- {
    --   "column": "order_total",
    --   "metric": "mean",
    --   "observed": 42.50,
    --   "baseline_mean": 55.20,
    --   "baseline_stddev": 3.10,
    --   "z_score": -4.10,
    --   "is_anomaly": true,
    --   "algorithm": "mad",
    --   "baseline_window_days": 30,
    --   "sample_count": 720
    -- }
    --
    -- Freshness result:
    -- {
    --   "timestamp_column": "updated_at",
    --   "latest_value": "2026-05-10T11:00:00Z",
    --   "age_hours": 3.5,
    --   "max_age_hours": 2,
    --   "passed": false
    -- }
    --
    -- Schema drift result:
    -- {
    --   "changes": [
    --     {"type": "column_removed", "column": "legacy_flag", "previous_type": "boolean"},
    --     {"type": "type_changed", "column": "amount", "previous_type": "integer", "current_type": "numeric"}
    --   ],
    --   "previous_snapshot_hash": "sha256:abc...",
    --   "current_snapshot_hash": "sha256:def..."
    -- }
    executed_at     TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (executed_at);

-- Create monthly partitions for check results
-- CREATE TABLE check_results_2026_05 PARTITION OF check_results
--     FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

CREATE INDEX idx_check_results_scan_run ON check_results(scan_run_id);
CREATE INDEX idx_check_results_check ON check_results(check_id);
CREATE INDEX idx_check_results_dataset ON check_results(dataset_id);
CREATE INDEX idx_check_results_status ON check_results(org_id, status, executed_at DESC);
CREATE INDEX idx_check_results_result ON check_results USING GIN(result jsonb_path_ops);
```

---

## Metrics & Baselines

```sql
-- ============================================================
-- METRIC TIME SERIES
-- ============================================================

CREATE TABLE metrics (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scan_run_id     UUID NOT NULL REFERENCES scan_runs(id) ON DELETE CASCADE,
    org_id          UUID NOT NULL REFERENCES organisations(id),
    dataset_id      UUID NOT NULL REFERENCES datasets(id),
    column_name     VARCHAR(255),  -- NULL for table-level metrics
    metric_type     VARCHAR(50) NOT NULL,
    dimension       VARCHAR(50) NOT NULL,
    value           DOUBLE PRECISION NOT NULL,
    context         JSONB DEFAULT '{}',
    -- Additional metric context:
    -- {
    --   "row_count": 1500000,
    --   "sample_size": 10000,
    --   "histogram": [0, 5, 42, 128, 89, 12, 3],
    --   "percentiles": {"p25": 12.5, "p50": 34.2, "p75": 58.9, "p95": 102.3, "p99": 245.1}
    -- }
    measured_at     TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (measured_at);

CREATE INDEX idx_metrics_lookup ON metrics(dataset_id, column_name, metric_type, measured_at DESC);
CREATE INDEX idx_metrics_org ON metrics(org_id, measured_at DESC);

-- ============================================================
-- ANOMALY BASELINES
-- ============================================================

CREATE TABLE metric_baselines (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    dataset_id      UUID NOT NULL REFERENCES datasets(id),
    column_name     VARCHAR(255),
    metric_type     VARCHAR(50) NOT NULL,
    baseline        JSONB NOT NULL,
    -- Baseline statistics:
    -- {
    --   "mean": 50000,
    --   "stddev": 2500,
    --   "median": 49800,
    --   "mad": 1800,
    --   "min": 42000,
    --   "max": 58000,
    --   "sample_count": 720,
    --   "window_days": 30,
    --   "seasonality": {
    --     "detected": true,
    --     "period": "weekly",
    --     "dow_means": [45000, 52000, 53000, 51000, 50000, 38000, 35000]
    --   }
    -- }
    last_updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(dataset_id, column_name, metric_type)
);

CREATE INDEX idx_baselines_dataset ON metric_baselines(dataset_id);
```

---

## Alerts & Incidents

```sql
-- ============================================================
-- ALERT CHANNELS & ALERTS
-- ============================================================

CREATE TABLE alert_channels (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    channel_type    VARCHAR(50) NOT NULL,
    config          JSONB NOT NULL,
    -- Channel config varies by type:
    --
    -- Slack: {"webhook_url": "https://hooks.slack.com/...", "channel": "#data-quality", "mention_on_critical": "@oncall"}
    -- PagerDuty: {"routing_key": "R0...", "severity_map": {"critical": "critical", "warning": "warning"}}
    -- Email: {"recipients": ["team@acme.com"], "digest_interval_minutes": 60}
    -- Webhook: {"url": "https://api.acme.com/webhooks/dq", "headers": {"X-API-Key": "vault://..."}, "method": "POST"}
    is_enabled      BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE alerts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    check_result_id UUID REFERENCES check_results(id) ON DELETE SET NULL,
    incident_id     UUID REFERENCES incidents(id) ON DELETE SET NULL,
    severity        VARCHAR(20) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    detail          JSONB NOT NULL DEFAULT '{}',
    -- Alert detail captures context:
    -- {
    --   "description": "Null rate on orders.email exceeded threshold",
    --   "ai_explanation": "The email column null rate jumped from 0.3% to 12% in the last 6 hours. This correlates with a schema change detected on the upstream user_registrations table, where the email field was made nullable at 2026-05-10T08:00Z. The upstream change likely introduced null values that propagated through the ETL pipeline.",
    --   "dataset": "public.orders",
    --   "check_name": "orders_email_completeness",
    --   "observed": 0.12,
    --   "threshold": 0.05,
    --   "lineage_context": {
    --     "upstream_tables": ["public.user_registrations"],
    --     "recent_schema_changes": [{"table": "user_registrations", "change": "email made nullable", "at": "2026-05-10T08:00Z"}]
    --   }
    -- }
    status          VARCHAR(20) NOT NULL DEFAULT 'open',
    acknowledged_by UUID REFERENCES users(id) ON DELETE SET NULL,
    acknowledged_at TIMESTAMPTZ,
    resolved_at     TIMESTAMPTZ,
    notifications   JSONB DEFAULT '[]',
    -- Notification log:
    -- [
    --   {"channel_id": "...", "channel_type": "slack", "sent_at": "2026-05-10T14:35:00Z", "message_ts": "1715350500.000100"},
    --   {"channel_id": "...", "channel_type": "pagerduty", "sent_at": "2026-05-10T14:35:01Z", "incident_key": "pd-123"}
    -- ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_alerts_org_status ON alerts(org_id, status);
CREATE INDEX idx_alerts_created_at ON alerts(created_at DESC);
CREATE INDEX idx_alerts_incident ON alerts(incident_id);
CREATE INDEX idx_alerts_detail ON alerts USING GIN(detail jsonb_path_ops);

-- ============================================================
-- INCIDENTS
-- ============================================================

CREATE TABLE incidents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    title           VARCHAR(500) NOT NULL,
    severity        VARCHAR(20) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'open',
    assigned_to     UUID REFERENCES users(id) ON DELETE SET NULL,
    detail          JSONB NOT NULL DEFAULT '{}',
    -- Incident detail:
    -- {
    --   "root_cause": "Upstream Airflow DAG orders_etl failed at 03:00 UTC",
    --   "resolution": "DAG restarted; backfill completed for missing partition",
    --   "affected_datasets": ["public.orders", "analytics.order_summary"],
    --   "timeline": [
    --     {"at": "2026-05-10T03:00Z", "event": "DAG failure"},
    --     {"at": "2026-05-10T06:00Z", "event": "Freshness alert fired"},
    --     {"at": "2026-05-10T06:15Z", "event": "Incident created, assigned to @alice"},
    --     {"at": "2026-05-10T07:30Z", "event": "Root cause identified"},
    --     {"at": "2026-05-10T08:00Z", "event": "DAG restarted"},
    --     {"at": "2026-05-10T09:00Z", "event": "Backfill complete, incident resolved"}
    --   ],
    --   "comments": [
    --     {"by": "alice", "at": "2026-05-10T06:20Z", "body": "Investigating upstream DAG"},
    --     {"by": "bob", "at": "2026-05-10T07:30Z", "body": "Found: orders_etl DAG has a new dependency on a table that was dropped"}
    --   ]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at     TIMESTAMPTZ
);

CREATE INDEX idx_incidents_org_status ON incidents(org_id, status);
```

---

## Data Lineage

```sql
-- ============================================================
-- DATA LINEAGE
-- ============================================================

CREATE TABLE lineage_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    source_dataset_id UUID NOT NULL REFERENCES datasets(id) ON DELETE CASCADE,
    target_dataset_id UUID NOT NULL REFERENCES datasets(id) ON DELETE CASCADE,
    edge_type       VARCHAR(50) NOT NULL DEFAULT 'transforms',
    detail          JSONB NOT NULL DEFAULT '{}',
    -- Edge detail:
    -- {
    --   "job_name": "orders_etl",
    --   "discovered_via": "dbt_manifest",
    --   "confidence": 1.0,
    --   "column_mappings": [
    --     {"source": "user_id", "target": "customer_id", "transform": "direct"},
    --     {"source": "amount", "target": "order_total", "transform": "cast(amount as numeric(10,2))"},
    --     {"source": ["quantity", "unit_price"], "target": "line_total", "transform": "quantity * unit_price"}
    --   ],
    --   "last_observed_at": "2026-05-10T14:35:00Z"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(source_dataset_id, target_dataset_id, edge_type)
);

CREATE INDEX idx_lineage_source ON lineage_edges(source_dataset_id);
CREATE INDEX idx_lineage_target ON lineage_edges(target_dataset_id);
CREATE INDEX idx_lineage_org ON lineage_edges(org_id);
```

---

## Data Contracts

```sql
-- ============================================================
-- DATA CONTRACTS (ODCS aligned, stored as JSONB)
-- ============================================================

CREATE TABLE data_contracts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    dataset_id      UUID NOT NULL REFERENCES datasets(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    version         VARCHAR(50) NOT NULL DEFAULT '1.0.0',
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
    spec            JSONB NOT NULL,
    -- Full contract specification as JSONB (follows ODCS v3 structure):
    -- {
    --   "producer": {"team": "data-engineering", "contact": "data-eng@acme.com"},
    --   "consumers": [
    --     {"team": "analytics", "contact": "analytics@acme.com", "subscribed_at": "2026-05-01"}
    --   ],
    --   "schema": {
    --     "columns": [
    --       {"name": "id", "type": "bigint", "nullable": false, "is_pk": true},
    --       {"name": "email", "type": "varchar", "nullable": true, "pii": true}
    --     ]
    --   },
    --   "quality": {
    --     "freshness_sla_minutes": 60,
    --     "completeness_sla_pct": 99.5,
    --     "checks": [
    --       {"type": "not_null", "column": "id"},
    --       {"type": "unique", "column": "id"},
    --       {"type": "max_null_rate", "column": "email", "threshold": 0.05}
    --     ]
    --   },
    --   "sla": {
    --     "availability": "99.9%",
    --     "support_hours": "business_hours",
    --     "notification_channels": ["slack:#data-contracts"]
    --   }
    -- }
    created_by      UUID REFERENCES users(id) ON DELETE SET NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_contracts_dataset ON data_contracts(dataset_id);
CREATE INDEX idx_contracts_org ON data_contracts(org_id);
CREATE INDEX idx_contracts_spec ON data_contracts USING GIN(spec jsonb_path_ops);
```

---

## Audit Log

```sql
-- ============================================================
-- AUDIT LOG (append-only)
-- ============================================================

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    actor_id        UUID,
    actor_type      VARCHAR(20) NOT NULL DEFAULT 'user',  -- user, system, api_key
    action          VARCHAR(100) NOT NULL,
    resource_type   VARCHAR(50) NOT NULL,
    resource_id     UUID NOT NULL,
    changes         JSONB DEFAULT '{}',
    context         JSONB DEFAULT '{}',
    -- Context:
    -- {"ip_address": "10.0.0.1", "user_agent": "...", "api_key_prefix": "dqm_abc1"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_org ON audit_log(org_id, created_at DESC);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
```

---

## Example Queries

```sql
-- Find all checks on a dataset with their latest result status
SELECT c.name, c.check_type, c.dimension, c.config, cr.status, cr.result, cr.executed_at
FROM checks c
LEFT JOIN LATERAL (
    SELECT status, result, executed_at
    FROM check_results
    WHERE check_id = c.id
    ORDER BY executed_at DESC
    LIMIT 1
) cr ON true
WHERE c.dataset_id = '<dataset-id>'
  AND c.is_enabled = true;

-- Find all datasets with freshness violations (JSONB query)
SELECT d.schema_name, d.table_name,
       (d.profile->'freshness'->>'last_updated_at')::timestamptz AS last_updated,
       EXTRACT(EPOCH FROM now() - (d.profile->'freshness'->>'last_updated_at')::timestamptz) / 3600 AS hours_stale
FROM datasets d
WHERE d.org_id = '<org-id>'
  AND d.profile->'freshness' IS NOT NULL
  AND (now() - (d.profile->'freshness'->>'last_updated_at')::timestamptz) >
      ((d.profile->'freshness'->>'update_frequency_hours')::integer * INTERVAL '2 hours');

-- Search for checks with a specific config pattern (GIN index)
SELECT c.name, c.config
FROM checks c
WHERE c.org_id = '<org-id>'
  AND c.config @> '{"column": "email"}'::jsonb;

-- AI-generated checks awaiting acceptance
SELECT c.name, c.dataset_id, d.table_name,
       c.ai_metadata->>'confidence' AS confidence,
       c.ai_metadata->>'reasoning' AS reasoning
FROM checks c
JOIN datasets d ON c.dataset_id = d.id
WHERE c.org_id = '<org-id>'
  AND c.is_ai_generated = true
  AND c.ai_metadata->>'accepted_at' IS NULL
ORDER BY (c.ai_metadata->>'confidence')::float DESC;

-- Upstream lineage traversal (recursive CTE)
WITH RECURSIVE upstream AS (
    SELECT source_dataset_id, target_dataset_id, 1 AS depth,
           detail->'column_mappings' AS mappings
    FROM lineage_edges
    WHERE target_dataset_id = '<dataset-id>'
  UNION ALL
    SELECT le.source_dataset_id, le.target_dataset_id, u.depth + 1,
           le.detail->'column_mappings'
    FROM lineage_edges le
    JOIN upstream u ON le.target_dataset_id = u.source_dataset_id
    WHERE u.depth < 10
)
SELECT d.schema_name, d.table_name, u.depth, u.mappings
FROM upstream u
JOIN datasets d ON u.source_dataset_id = d.id
ORDER BY u.depth;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation & Access | 2 | organisations, users |
| Data Sources & Datasets | 2 | data_sources, datasets (columns embedded as JSONB) |
| Check Configuration | 2 | check_templates, checks |
| Scan Execution | 2 | scan_jobs, scan_runs |
| Results & Metrics | 3 | check_results (partitioned), metrics (partitioned), metric_baselines |
| Alerts & Incidents | 3 | alert_channels, alerts, incidents |
| Lineage | 1 | lineage_edges (column mappings in JSONB) |
| Data Contracts | 1 | data_contracts (full spec as JSONB) |
| Audit | 1 | audit_log (partitioned) |
| **Total** | **17** | Several tables partitioned by time |

---

## Key Design Decisions

1. **Columns embedded in dataset profile, not a separate table.** In the normalised model, `dataset_columns` is a separate table with foreign keys. Here, column metadata (names, types, stats) lives in the `profile` JSONB column of `datasets`. This eliminates a JOIN for the most common query pattern ("show me this table and its columns") and naturally handles column statistics that vary by type (numeric stats vs. string pattern stats vs. temporal range stats).

2. **Check config as JSONB with JSON Schema validation.** Each check template defines a `parameter_schema` (JSON Schema Draft 2020-12) that validates the `config` JSONB on the corresponding `checks` row. This gives type safety without database rigidity: adding a new check type means adding a new template row, not a new table or ALTER TABLE.

3. **Check results capture full context in a single JSONB column.** A null-rate result, an anomaly detection result, and a schema drift result have completely different structures. Rather than forcing them into a single set of relational columns (with many NULLs) or creating per-type result tables, the `result` JSONB column holds the complete, typed output. The GIN index on `result` enables efficient filtering.

4. **Column lineage embedded in lineage edge detail.** Rather than a separate `column_lineage_edges` table (as in the normalised model), column-level mappings are stored in the `detail` JSONB of each `lineage_edges` row. This reflects the reality that column mappings are always queried in the context of their parent edge, never independently.

5. **Incident timeline and comments in JSONB, not separate tables.** Incidents are typically short-lived (hours to days) with a handful of comments. Storing the timeline and comments as JSONB arrays in the `detail` column eliminates two tables and keeps incident rendering a single-row read. For high-volume incident management (thousands of comments), a separate table would be warranted — but that is not the expected usage pattern for a data quality monitor.

6. **Partitioning for time-series data.** `check_results`, `metrics`, and `audit_log` are partitioned by `executed_at`/`measured_at`/`created_at`. This ensures query performance remains stable as the platform accumulates months of history, and enables efficient retention policies (drop old partitions).

7. **Data contracts as full JSONB documents.** The ODCS (Open Data Contract Standard) defines contracts as YAML/JSON documents. Storing the entire contract spec as JSONB preserves the original structure, enables import/export without transformation, and allows GIN-indexed queries across contract fields.

8. **AI metadata as a dedicated JSONB column.** AI-generated checks carry metadata (model, confidence, reasoning) that rule-based checks do not. Rather than adding nullable columns or a separate table, the `ai_metadata` JSONB column is only populated for AI-generated checks. This cleanly separates AI-specific concerns from core check definition.

9. **Notifications logged inline on alerts.** Each alert's notification history (which channels were notified, when, with what message IDs) is stored as a JSONB array on the alert row itself. This avoids a separate notification log table while keeping a complete delivery record for debugging.

10. **17 tables vs. 28 in the normalised model.** The JSONB approach absorbs variability that would otherwise require additional junction tables, type-specific tables, and reference data tables. This lower table count accelerates development and simplifies deployment, at the cost of weaker database-enforced integrity.
