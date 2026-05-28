# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Data Quality Monitor · Created: 2026-05-11

## Philosophy

This model uses event sourcing as its foundational architecture: every state change in the system is recorded as an immutable event in a central event store. The current state of any entity (dataset, check, alert, incident) is derived by replaying its event history. Read-optimised materialised views serve the web UI and API, while the event store remains the single source of truth.

This pattern is directly inspired by the audit trail requirements of GDPR Article 5(1)(d), HIPAA Section 164.312, and BCBS 239 — regulations that demand complete, tamper-evident records of how data quality was monitored, when failures were detected, and what actions were taken. Rather than bolting an audit log onto a mutable relational schema, this design makes auditability intrinsic: every fact that was ever true is preserved, and temporal queries ("what checks were passing on January 15th?") are first-class operations.

The CQRS (Command Query Responsibility Segregation) split means writes go to the event store and reads come from materialised projections. This decouples write throughput (appending events) from read performance (querying denormalised views), and allows different read models to be built for different consumers — a dashboard view, an API view, a compliance export view — all from the same event stream.

**Best for:** Regulated industries (financial services, healthcare, government) where complete audit trails, temporal queries, and tamper-evident records are non-negotiable requirements.

**Trade-offs:**
- (+) Complete, immutable audit trail as an intrinsic property of the architecture — not an afterthought
- (+) Temporal queries are trivial: replay events to any point in time
- (+) Write performance is excellent: appending to an event log is the fastest write pattern
- (+) Multiple read models can be built independently without affecting the source of truth
- (+) Natural fit for streaming architectures (Kafka, Pulsar) if the platform scales to real-time
- (-) Higher implementation complexity: event replay, projection rebuilds, and eventual consistency require engineering discipline
- (-) Read model staleness: materialised views may lag behind the event store by seconds or minutes
- (-) Schema evolution of events is harder than ALTER TABLE; requires event versioning and upcasting
- (-) Storage grows monotonically; compaction and archival strategies are needed for long-running deployments
- (-) Debugging is harder: "what is the current state?" requires understanding which events contributed

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO/IEC 25012 (SQuaRE) | Quality dimensions encoded as event metadata; each quality check event carries its ISO 25012 dimension |
| GDPR Art. 5(1)(d) | The event store is a tamper-evident accuracy audit trail by design |
| HIPAA § 164.312 | Integrity controls satisfied by the append-only, hash-chained event store |
| BCBS 239 | Timeliness and completeness of quality monitoring are provably recorded in the event stream |
| OpenLineage | Each scan execution emits OpenLineage-compatible Run events with quality facets |
| W3C PROV | Event store entries naturally map to PROV activities (scans), entities (datasets), and agents (users/services) |
| OCSF | Alert and incident events follow the Open Cybersecurity Schema Framework event structure patterns |

---

## Event Store (Core Write Model)

```sql
-- ============================================================
-- EVENT STORE — the single source of truth
-- ============================================================

CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type  VARCHAR(50) NOT NULL,  -- organisation, data_source, dataset, check, scan, alert, incident, contract
    aggregate_id    UUID NOT NULL,          -- ID of the entity this event belongs to
    org_id          UUID NOT NULL,          -- tenant partition key
    event_type      VARCHAR(100) NOT NULL,  -- e.g., 'dataset.registered', 'check.created', 'scan.completed', 'alert.fired'
    event_version   INTEGER NOT NULL DEFAULT 1,  -- schema version of this event type
    sequence_number BIGINT NOT NULL,        -- per-aggregate ordering (optimistic concurrency)
    payload         JSONB NOT NULL,          -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',  -- actor, IP, correlation ID, causation ID
    -- Example metadata:
    -- {
    --   "actor_id": "user-uuid",
    --   "actor_type": "user",
    --   "correlation_id": "req-uuid",
    --   "causation_id": "previous-event-uuid",
    --   "ip_address": "10.0.0.1",
    --   "user_agent": "Mozilla/5.0..."
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(aggregate_type, aggregate_id, sequence_number)
);

-- Primary read path: get all events for an aggregate in order
CREATE INDEX idx_event_store_aggregate ON event_store(aggregate_type, aggregate_id, sequence_number);

-- Temporal queries: find all events in a time range
CREATE INDEX idx_event_store_created_at ON event_store(org_id, created_at);

-- Event type filtering: replay all events of a given type
CREATE INDEX idx_event_store_event_type ON event_store(org_id, event_type, created_at);

-- Partition by month for storage management
-- In production, use pg_partman or native declarative partitioning:
-- CREATE TABLE event_store (...) PARTITION BY RANGE (created_at);

-- ============================================================
-- AGGREGATE VERSION TRACKING (optimistic concurrency control)
-- ============================================================

CREATE TABLE aggregate_versions (
    aggregate_type  VARCHAR(50) NOT NULL,
    aggregate_id    UUID NOT NULL,
    current_version BIGINT NOT NULL DEFAULT 0,
    PRIMARY KEY (aggregate_type, aggregate_id)
);

-- ============================================================
-- EVENT SNAPSHOTS (avoid replaying full history for long-lived aggregates)
-- ============================================================

CREATE TABLE aggregate_snapshots (
    aggregate_type  VARCHAR(50) NOT NULL,
    aggregate_id    UUID NOT NULL,
    snapshot_version BIGINT NOT NULL,
    state           JSONB NOT NULL,  -- serialised aggregate state at this version
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (aggregate_type, aggregate_id, snapshot_version)
);
```

---

## Event Type Catalogue

Each event type has a defined payload schema. Below are the key event types grouped by aggregate.

```sql
-- ============================================================
-- EVENT TYPE DEFINITIONS (for documentation; enforced by application layer)
-- ============================================================

-- ORGANISATION EVENTS
-- org.created        { "name": "Acme Corp", "slug": "acme", "plan_tier": "team" }
-- org.plan_changed   { "old_tier": "free", "new_tier": "team" }
-- org.settings_updated { "changes": { "default_severity": "critical" } }

-- DATA SOURCE EVENTS
-- source.connected    { "name": "Prod Snowflake", "source_type": "snowflake", "config_ref": "vault://..." }
-- source.sync_started { "tables_discovered": 42 }
-- source.sync_completed { "tables_discovered": 42, "columns_discovered": 312, "duration_ms": 5400 }
-- source.disconnected { "reason": "user_action" }

-- DATASET EVENTS
-- dataset.registered  { "source_id": "...", "schema": "public", "table": "orders", "columns": [...] }
-- dataset.schema_changed { "changes": [{"type": "column_added", "column": "discount_pct", "data_type": "numeric"}] }
-- dataset.profiled    { "row_count": 1500000, "size_bytes": 4200000, "column_stats": {...} }
-- dataset.ownership_changed { "old_team": "...", "new_team": "..." }

-- CHECK EVENTS
-- check.created       { "name": "orders_email_not_null", "type": "rule_based", "dimension": "completeness", "params": {...} }
-- check.updated       { "changes": {"threshold": {"old": 0.05, "new": 0.02}} }
-- check.enabled       {}
-- check.disabled      { "reason": "too noisy" }
-- check.deleted       { "reason": "replaced by ML check" }
-- check.ai_suggested  { "model": "gpt-4", "confidence": 0.87, "suggestion": {...} }
-- check.ai_suggestion_accepted { "original_suggestion_event_id": "..." }
-- check.ai_suggestion_rejected { "original_suggestion_event_id": "...", "reason": "not applicable" }

-- SCAN EVENTS
-- scan.scheduled      { "job_name": "hourly_orders", "cron": "0 * * * *", "datasets": ["..."] }
-- scan.started        { "run_id": "...", "trigger": "scheduled", "checks_count": 24 }
-- scan.check_passed   { "run_id": "...", "check_id": "...", "dataset_id": "...", "observed": 0.001, "threshold": 0.05 }
-- scan.check_failed   { "run_id": "...", "check_id": "...", "dataset_id": "...", "observed": 0.12, "threshold": 0.05, "failing_rows": 1800 }
-- scan.check_warned   { "run_id": "...", "check_id": "...", "dataset_id": "...", "observed": 0.045, "threshold": 0.05 }
-- scan.check_errored  { "run_id": "...", "check_id": "...", "error": "connection timeout" }
-- scan.completed      { "run_id": "...", "passed": 20, "warned": 2, "failed": 1, "errored": 1, "duration_ms": 45000 }
-- scan.failed         { "run_id": "...", "error": "source unreachable" }

-- METRIC EVENTS
-- metric.recorded     { "run_id": "...", "dataset_id": "...", "column_id": "...", "type": "null_rate", "dimension": "completeness", "value": 0.034 }
-- metric.baseline_updated { "dataset_id": "...", "column_id": "...", "type": "null_rate", "mean": 0.031, "stddev": 0.008, "sample_count": 720 }
-- metric.anomaly_detected { "dataset_id": "...", "column_id": "...", "type": "row_count", "observed": 15000, "expected_mean": 50000, "z_score": -4.37 }

-- ALERT EVENTS
-- alert.fired         { "severity": "critical", "title": "...", "check_result_event_id": "...", "ai_explanation": "..." }
-- alert.notified      { "channel": "slack", "channel_id": "...", "message_ts": "..." }
-- alert.acknowledged  { "by_user": "..." }
-- alert.resolved      { "resolution": "upstream pipeline fixed" }
-- alert.suppressed    { "reason": "known maintenance window", "until": "2026-05-12T06:00:00Z" }

-- INCIDENT EVENTS
-- incident.created    { "title": "...", "severity": "critical", "alert_ids": ["..."] }
-- incident.assigned   { "to_user": "...", "to_team": "..." }
-- incident.commented  { "body": "Investigating upstream Airflow DAG failure", "by_user": "..." }
-- incident.root_cause_identified { "cause": "Airflow DAG orders_etl failed at 03:00 UTC" }
-- incident.resolved   { "resolution": "DAG restarted and backfill completed" }
-- incident.reopened   { "reason": "Issue recurred" }

-- CONTRACT EVENTS
-- contract.created    { "name": "orders_v2", "version": "1.0.0", "schema_spec": {...}, "sla": {...} }
-- contract.published  { "producer_team": "..." }
-- contract.subscribed { "consumer_team": "..." }
-- contract.violated   { "violations": [{"type": "freshness_sla", "expected_minutes": 60, "actual_minutes": 180}] }
-- contract.deprecated { "successor_contract_id": "..." }
```

---

## Read Model Projections (Materialised Views)

These tables are rebuilt from the event store. They can be dropped and reconstructed at any time.

```sql
-- ============================================================
-- PROJECTION: Current Organisation State
-- ============================================================

CREATE TABLE proj_organisations (
    id              UUID PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL,
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL,
    projection_version BIGINT NOT NULL  -- last event sequence processed
);

-- ============================================================
-- PROJECTION: Current Dataset Inventory
-- ============================================================

CREATE TABLE proj_datasets (
    id              UUID PRIMARY KEY,
    org_id          UUID NOT NULL,
    source_id       UUID NOT NULL,
    source_name     VARCHAR(255) NOT NULL,
    source_type     VARCHAR(50) NOT NULL,
    schema_name     VARCHAR(255),
    table_name      VARCHAR(255) NOT NULL,
    dataset_type    VARCHAR(50) NOT NULL,
    row_count       BIGINT,
    size_bytes      BIGINT,
    column_count    INTEGER,
    owner_team_name VARCHAR(255),
    last_profiled_at TIMESTAMPTZ,
    last_schema_change_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL,
    projection_version BIGINT NOT NULL
);

CREATE INDEX idx_proj_datasets_org_id ON proj_datasets(org_id);

-- ============================================================
-- PROJECTION: Latest Check Results (denormalised for dashboard)
-- ============================================================

CREATE TABLE proj_latest_check_results (
    dataset_check_id UUID PRIMARY KEY,
    org_id          UUID NOT NULL,
    dataset_id      UUID NOT NULL,
    dataset_name    VARCHAR(255) NOT NULL,
    column_name     VARCHAR(255),
    check_name      VARCHAR(255) NOT NULL,
    check_type      VARCHAR(50) NOT NULL,
    dimension       VARCHAR(50) NOT NULL,
    last_status     VARCHAR(20) NOT NULL,
    last_observed   DOUBLE PRECISION,
    last_expected   DOUBLE PRECISION,
    last_run_id     UUID,
    last_executed_at TIMESTAMPTZ,
    consecutive_failures INTEGER DEFAULT 0,
    projection_version BIGINT NOT NULL
);

CREATE INDEX idx_proj_latest_checks_org_id ON proj_latest_check_results(org_id);
CREATE INDEX idx_proj_latest_checks_status ON proj_latest_check_results(org_id, last_status);
CREATE INDEX idx_proj_latest_checks_dataset ON proj_latest_check_results(dataset_id);

-- ============================================================
-- PROJECTION: Metric Time Series (append-only, rebuilt from metric.recorded events)
-- ============================================================

CREATE TABLE proj_metric_series (
    id              UUID PRIMARY KEY,
    org_id          UUID NOT NULL,
    dataset_id      UUID NOT NULL,
    column_id       UUID,
    metric_type     VARCHAR(50) NOT NULL,
    dimension       VARCHAR(50) NOT NULL,
    value           DOUBLE PRECISION NOT NULL,
    measured_at     TIMESTAMPTZ NOT NULL
) PARTITION BY RANGE (measured_at);

-- Create monthly partitions
-- CREATE TABLE proj_metric_series_2026_05 PARTITION OF proj_metric_series
--     FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

CREATE INDEX idx_proj_metrics_lookup ON proj_metric_series(dataset_id, column_id, metric_type, measured_at DESC);

-- ============================================================
-- PROJECTION: Active Alerts
-- ============================================================

CREATE TABLE proj_active_alerts (
    id              UUID PRIMARY KEY,
    org_id          UUID NOT NULL,
    severity        VARCHAR(20) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    ai_explanation  TEXT,
    status          VARCHAR(20) NOT NULL,
    dataset_name    VARCHAR(255),
    check_name      VARCHAR(255),
    incident_id     UUID,
    notified_at     TIMESTAMPTZ,
    acknowledged_by VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL,
    projection_version BIGINT NOT NULL
);

CREATE INDEX idx_proj_alerts_org_status ON proj_active_alerts(org_id, status);

-- ============================================================
-- PROJECTION: Open Incidents
-- ============================================================

CREATE TABLE proj_incidents (
    id              UUID PRIMARY KEY,
    org_id          UUID NOT NULL,
    title           VARCHAR(500) NOT NULL,
    severity        VARCHAR(20) NOT NULL,
    status          VARCHAR(20) NOT NULL,
    assigned_to     VARCHAR(255),
    assigned_team   VARCHAR(255),
    alert_count     INTEGER DEFAULT 0,
    root_cause      TEXT,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL,
    projection_version BIGINT NOT NULL
);

CREATE INDEX idx_proj_incidents_org_status ON proj_incidents(org_id, status);

-- ============================================================
-- PROJECTION: Anomaly Detection Baselines
-- ============================================================

CREATE TABLE proj_metric_baselines (
    dataset_id      UUID NOT NULL,
    column_id       UUID,
    metric_type     VARCHAR(50) NOT NULL,
    baseline_mean   DOUBLE PRECISION NOT NULL,
    baseline_stddev DOUBLE PRECISION NOT NULL,
    baseline_median DOUBLE PRECISION,
    baseline_mad    DOUBLE PRECISION,
    sample_count    INTEGER NOT NULL,
    window_days     INTEGER NOT NULL DEFAULT 30,
    last_updated_at TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (dataset_id, column_id, metric_type)
);
```

---

## Event Processing & Projection Rebuild

```sql
-- ============================================================
-- PROJECTION CHECKPOINTS (track which events each projection has processed)
-- ============================================================

CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_event_id   UUID,
    last_processed_at TIMESTAMPTZ,
    status          VARCHAR(20) NOT NULL DEFAULT 'active'  -- active, rebuilding, paused
);

-- Seed checkpoints for all projections
INSERT INTO projection_checkpoints (projection_name, status) VALUES
    ('proj_organisations', 'active'),
    ('proj_datasets', 'active'),
    ('proj_latest_check_results', 'active'),
    ('proj_metric_series', 'active'),
    ('proj_active_alerts', 'active'),
    ('proj_incidents', 'active'),
    ('proj_metric_baselines', 'active');
```

---

## Example Queries

```sql
-- Temporal query: What checks were failing on a specific date?
-- Replay scan.check_failed events up to that point
SELECT payload->>'check_id' AS check_id,
       payload->>'dataset_id' AS dataset_id,
       payload->>'observed' AS observed_value,
       created_at
FROM event_store
WHERE event_type = 'scan.check_failed'
  AND org_id = '<org-id>'
  AND created_at <= '2026-03-15T23:59:59Z'
  AND created_at > '2026-03-15T00:00:00Z'
ORDER BY created_at;

-- Full history of an incident (from creation to resolution)
SELECT event_type, payload, metadata->>'actor_id' AS actor, created_at
FROM event_store
WHERE aggregate_type = 'incident'
  AND aggregate_id = '<incident-id>'
ORDER BY sequence_number;

-- Compliance export: all quality-related events for a dataset in a date range
SELECT event_type, payload, metadata, created_at
FROM event_store
WHERE org_id = '<org-id>'
  AND event_type LIKE 'scan.%'
  AND created_at BETWEEN '2026-01-01' AND '2026-03-31'
  AND payload->>'dataset_id' = '<dataset-id>'
ORDER BY created_at;

-- Rebuild a projection (pseudocode — executed by application layer)
-- 1. TRUNCATE proj_latest_check_results;
-- 2. SELECT * FROM event_store
--    WHERE event_type IN ('check.created', 'check.updated', 'scan.check_passed', 'scan.check_failed', ...)
--    ORDER BY created_at;
-- 3. For each event, apply projection logic
-- 4. UPDATE projection_checkpoints SET last_event_id = <latest>, last_processed_at = now()
--    WHERE projection_name = 'proj_latest_check_results';

-- Dashboard query: current status overview (reads from projection)
SELECT last_status, COUNT(*) AS check_count
FROM proj_latest_check_results
WHERE org_id = '<org-id>'
GROUP BY last_status;

-- Anomaly detection: z-score calculation from projected baselines
SELECT ms.dataset_id, ms.column_id, ms.metric_type,
       ms.value AS observed,
       mb.baseline_mean,
       mb.baseline_stddev,
       CASE WHEN mb.baseline_stddev > 0
            THEN ABS(ms.value - mb.baseline_mean) / mb.baseline_stddev
            ELSE 0
       END AS z_score
FROM proj_metric_series ms
JOIN proj_metric_baselines mb
  ON ms.dataset_id = mb.dataset_id
  AND COALESCE(ms.column_id, '00000000-0000-0000-0000-000000000000') =
      COALESCE(mb.column_id, '00000000-0000-0000-0000-000000000000')
  AND ms.metric_type = mb.metric_type
WHERE ms.measured_at > now() - INTERVAL '1 hour'
  AND CASE WHEN mb.baseline_stddev > 0
           THEN ABS(ms.value - mb.baseline_mean) / mb.baseline_stddev
           ELSE 0
      END > 3.0;  -- flag z-scores above 3
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store (Write Model) | 3 | event_store, aggregate_versions, aggregate_snapshots |
| Projection: Organisation | 1 | proj_organisations |
| Projection: Datasets | 1 | proj_datasets |
| Projection: Check Results | 1 | proj_latest_check_results |
| Projection: Metrics | 2 | proj_metric_series (partitioned), proj_metric_baselines |
| Projection: Alerts & Incidents | 2 | proj_active_alerts, proj_incidents |
| Projection Infrastructure | 1 | projection_checkpoints |
| **Total** | **11** | 3 core + 8 projections; projections are rebuildable |

---

## Key Design Decisions

1. **Single event store table, not one per aggregate.** A single `event_store` table with `aggregate_type` and `aggregate_id` columns simplifies infrastructure (one table to partition, back up, and replicate) while still allowing per-aggregate event replay via the composite index. This follows the Marten/EventStoreDB pattern used in production event-sourced systems.

2. **Optimistic concurrency via aggregate_versions.** Before appending events, the application reads `current_version` from `aggregate_versions`, increments it, and attempts the insert. If two concurrent writers conflict on the same aggregate, the unique constraint on `(aggregate_type, aggregate_id, sequence_number)` causes one to fail, triggering a retry. This prevents lost updates without heavy locking.

3. **Snapshots for long-lived aggregates.** Datasets and organisations accumulate thousands of events over their lifetime. Periodic snapshots (e.g., every 100 events) allow the application to hydrate aggregate state from the snapshot + recent events, rather than replaying the full history. Snapshots are stored in `aggregate_snapshots`.

4. **Projections are disposable.** Every `proj_*` table can be dropped and rebuilt from the event store. This means schema changes to read models are zero-risk: add a new column, rebuild the projection, done. The `projection_checkpoints` table tracks which events each projection has consumed, enabling incremental catch-up after rebuilds.

5. **Event metadata captures provenance.** Every event includes a `metadata` JSONB field with `actor_id`, `actor_type`, `correlation_id`, `causation_id`, `ip_address`, and `user_agent`. This satisfies W3C PROV requirements (who did what, when, and why) and HIPAA integrity controls (tamper-evident activity logging).

6. **Partitioning by time.** The event store and `proj_metric_series` are designed for time-range partitioning. Monthly partitions keep query performance stable as the event store grows, and allow archival of old partitions to cold storage without affecting active queries.

7. **Rich event type catalogue as documentation.** The ~40 defined event types serve as a living specification of system behaviour. Adding new functionality means defining new event types — no schema migrations required. Event versioning (`event_version` field) handles payload evolution over time.

8. **Compliance exports are trivial.** A regulator asking "show me all quality monitoring activity for dataset X between January and March" is a single SELECT on the event store with time-range and payload filters. No complex JOINs across normalised tables — the event stream contains everything.

9. **CQRS enables independent read optimisation.** The dashboard needs denormalised, pre-aggregated data. The API needs paginated, filterable results. The compliance team needs raw event exports. Each projection can be independently optimised for its consumer without compromising the write model.

10. **Natural migration path to streaming.** The event store can be mirrored to Kafka/Pulsar via CDC (Change Data Capture) on the `event_store` table. This enables real-time streaming of quality events to external systems (data catalogs, incident management, BI tools) without changing the core architecture.
