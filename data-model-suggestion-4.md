# Data Model Suggestion 4: Graph-Relational (Lineage-First)

> Project: Data Quality Monitor · Created: 2026-05-11

## Philosophy

This model places the data lineage graph at the centre of the architecture. Rather than treating lineage as an add-on (a table or two appended to a relational schema), this design uses a property graph layer — implemented via PostgreSQL's `ltree` extension and a generic graph schema (`graph_nodes` / `graph_edges`) — as the structural backbone. Every entity in the system (data source, dataset, column, check, pipeline job, team) is a graph node, and every relationship (transforms, owns, monitors, triggers, affects) is a graph edge with typed properties.

This architecture is directly motivated by the observation that the highest-value features of a data quality monitor — root-cause analysis, impact analysis, anomaly propagation tracing, and conflict-of-interest detection — are all graph traversal problems. Monte Carlo's primary differentiator is its lineage graph. When a freshness alert fires on a downstream table, the first question is always "what upstream job failed, and which other tables are affected?" In a relational model, this requires recursive CTEs. In a graph model, it is a native traversal.

The design uses PostgreSQL as the graph store (not Neo4j or a separate graph database), keeping operational simplicity. The `ltree` extension provides efficient hierarchical path queries for dataset-within-schema-within-database-within-source hierarchies, while the generic `graph_nodes`/`graph_edges` tables handle arbitrary relationship traversal. Operational CRUD data (scan results, metrics, alerts) lives in standard relational tables that reference graph nodes by ID.

**Best for:** Platforms where lineage-driven root-cause analysis, impact analysis, and relationship-aware anomaly propagation are the primary value propositions — especially when competing with Monte Carlo's lineage graph capabilities.

**Trade-offs:**
- (+) Root-cause and impact analysis are native graph traversals, not recursive SQL CTEs
- (+) Relationship-rich queries (show all datasets affected by this pipeline failure) are first-class operations
- (+) `ltree` enables fast hierarchical queries (all tables in a schema, all schemas in a database) without recursive CTEs
- (+) The generic node/edge model is infinitely extensible: new entity types and relationship types require no schema migrations
- (+) Natural fit for visualisation: the graph can be directly rendered as lineage diagrams
- (-) Higher query complexity for simple CRUD operations (fetching a dataset requires a node lookup + property extraction)
- (-) Generic `graph_nodes`/`graph_edges` tables lose type-specific indexes and constraints; application layer must enforce consistency
- (-) Graph traversal queries can be slow without careful edge indexing and depth limits
- (-) PostgreSQL is not a native graph database; very deep or complex traversals may hit performance limits
- (-) Development teams unfamiliar with graph patterns face a steeper learning curve
- (-) The `ltree` extension must be installed and maintained

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenLineage | The graph model directly represents OpenLineage's Job/Run/Dataset/Facet relationships as typed nodes and edges |
| W3C PROV | PROV's Entity/Activity/Agent model maps to graph nodes; wasGeneratedBy/wasAttributedTo/used relationships map to graph edges |
| ISO/IEC 25012 (SQuaRE) | Quality dimensions are properties on check and metric nodes |
| W3C SHACL | Graph validation shapes can be expressed as SHACL constraints over the property graph |
| W3C DCAT | Dataset catalog metadata follows DCAT vocabulary, stored as node properties |
| PostgreSQL ltree | Used for hierarchical path queries on the source > database > schema > table > column hierarchy |

---

## Graph Core (Property Graph Layer)

```sql
-- ============================================================
-- PREREQUISITES
-- ============================================================

CREATE EXTENSION IF NOT EXISTS ltree;

-- ============================================================
-- GRAPH CORE: Nodes and Edges
-- ============================================================

CREATE TABLE graph_nodes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    node_type       VARCHAR(50) NOT NULL,
    -- Node types:
    --   organisation, team, user,
    --   data_source, database, schema, dataset, column,
    --   check_definition, check_instance,
    --   scan_job, scan_run,
    --   pipeline_job, pipeline_run,
    --   alert, incident,
    --   data_contract
    name            VARCHAR(500) NOT NULL,
    path            LTREE,          -- hierarchical path for fast subtree queries
    -- Example paths:
    --   'src.snowflake_prod'
    --   'src.snowflake_prod.analytics'
    --   'src.snowflake_prod.analytics.public'
    --   'src.snowflake_prod.analytics.public.orders'
    --   'src.snowflake_prod.analytics.public.orders.email'
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Properties vary by node_type. Examples:
    --
    -- data_source node:
    -- {
    --   "source_type": "snowflake",
    --   "account": "xy12345.us-east-1",
    --   "status": "active",
    --   "credentials_ref": "vault://snowflake/prod"
    -- }
    --
    -- dataset node:
    -- {
    --   "schema": "public",
    --   "table": "orders",
    --   "dataset_type": "table",
    --   "row_count": 1500000,
    --   "size_bytes": 420000000,
    --   "tags": ["production", "critical"],
    --   "freshness_hours": 1
    -- }
    --
    -- column node:
    -- {
    --   "data_type": "varchar(255)",
    --   "nullable": true,
    --   "ordinal": 2,
    --   "semantic_type": "email",
    --   "stats": {"null_rate": 0.003, "distinct_count": 1487320}
    -- }
    --
    -- check_instance node:
    -- {
    --   "check_type": "rule_based",
    --   "dimension": "completeness",
    --   "scope": "column",
    --   "config": {"column": "email", "max_null_rate": 0.05},
    --   "severity": "critical",
    --   "is_ai_generated": false,
    --   "is_enabled": true
    -- }
    --
    -- pipeline_job node:
    -- {
    --   "job_type": "dbt",
    --   "schedule_cron": "0 */6 * * *",
    --   "dbt_model": "orders_staging",
    --   "last_run_status": "success"
    -- }
    status          VARCHAR(50) DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_nodes_org ON graph_nodes(org_id);
CREATE INDEX idx_graph_nodes_type ON graph_nodes(org_id, node_type);
CREATE INDEX idx_graph_nodes_path ON graph_nodes USING GIST(path);
CREATE INDEX idx_graph_nodes_properties ON graph_nodes USING GIN(properties jsonb_path_ops);
CREATE INDEX idx_graph_nodes_name ON graph_nodes(org_id, name);

CREATE TABLE graph_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    source_node_id  UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    target_node_id  UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    edge_type       VARCHAR(50) NOT NULL,
    -- Edge types:
    --   contains        (source > schema > table > column hierarchy)
    --   transforms      (pipeline_job produces dataset from source datasets)
    --   monitors        (check_instance monitors a dataset or column)
    --   triggered_by    (scan_run triggered by scan_job)
    --   produced_by     (check_result produced by scan_run)
    --   raised          (alert raised by check_result)
    --   grouped_in      (alert grouped into incident)
    --   assigned_to     (incident assigned to user/team)
    --   owns            (team owns dataset or data_source)
    --   consumes        (team consumes dataset via data_contract)
    --   column_maps_to  (column-level lineage)
    --   depends_on      (pipeline_job depends on another pipeline_job)
    --   affected_by     (dataset affected by alert/incident)
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Properties vary by edge_type. Examples:
    --
    -- transforms edge:
    -- {
    --   "job_name": "orders_etl",
    --   "discovered_via": "dbt_manifest",
    --   "confidence": 1.0,
    --   "last_observed_at": "2026-05-10T14:35:00Z"
    -- }
    --
    -- column_maps_to edge:
    -- {
    --   "transformation": "cast(amount as numeric(10,2))",
    --   "transform_type": "cast"
    -- }
    --
    -- monitors edge:
    -- {
    --   "check_type": "null_rate",
    --   "dimension": "completeness"
    -- }
    weight          DOUBLE PRECISION DEFAULT 1.0,  -- for weighted graph algorithms
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_edges_source ON graph_edges(source_node_id);
CREATE INDEX idx_graph_edges_target ON graph_edges(target_node_id);
CREATE INDEX idx_graph_edges_type ON graph_edges(org_id, edge_type);
CREATE INDEX idx_graph_edges_source_type ON graph_edges(source_node_id, edge_type);
CREATE INDEX idx_graph_edges_target_type ON graph_edges(target_node_id, edge_type);
CREATE INDEX idx_graph_edges_properties ON graph_edges USING GIN(properties jsonb_path_ops);
```

---

## Operational Tables (Relational Layer)

The graph layer captures structure and relationships. The operational tables below capture time-series data (scan results, metrics, alerts) that reference graph nodes but need their own indexing and partitioning strategies.

```sql
-- ============================================================
-- ORGANISATION (lightweight, for RLS and FK references)
-- ============================================================

CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    node_id         UUID NOT NULL REFERENCES graph_nodes(id),  -- back-reference to graph
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- SCAN EXECUTION
-- ============================================================

CREATE TABLE scan_runs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    node_id         UUID NOT NULL REFERENCES graph_nodes(id),  -- scan_run node in graph
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    scan_job_node_id UUID NOT NULL REFERENCES graph_nodes(id),  -- scan_job node
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    trigger_type    VARCHAR(20) NOT NULL DEFAULT 'scheduled',
    triggered_by    UUID REFERENCES graph_nodes(id),  -- user node
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    summary         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_scan_runs_org ON scan_runs(org_id);
CREATE INDEX idx_scan_runs_started ON scan_runs(started_at DESC);
CREATE INDEX idx_scan_runs_job ON scan_runs(scan_job_node_id);

-- ============================================================
-- CHECK RESULTS
-- ============================================================

CREATE TABLE check_results (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scan_run_id     UUID NOT NULL REFERENCES scan_runs(id) ON DELETE CASCADE,
    org_id          UUID NOT NULL,
    check_node_id   UUID NOT NULL REFERENCES graph_nodes(id),   -- check_instance node
    dataset_node_id UUID NOT NULL REFERENCES graph_nodes(id),   -- dataset node
    column_node_id  UUID REFERENCES graph_nodes(id),            -- column node (NULL for table-level)
    status          VARCHAR(20) NOT NULL,
    observed_value  DOUBLE PRECISION,
    expected_value  DOUBLE PRECISION,
    deviation       DOUBLE PRECISION,
    result_detail   JSONB NOT NULL DEFAULT '{}',
    executed_at     TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (executed_at);

CREATE INDEX idx_check_results_scan ON check_results(scan_run_id);
CREATE INDEX idx_check_results_check ON check_results(check_node_id);
CREATE INDEX idx_check_results_dataset ON check_results(dataset_node_id);
CREATE INDEX idx_check_results_status ON check_results(org_id, status, executed_at DESC);

-- ============================================================
-- METRIC TIME SERIES
-- ============================================================

CREATE TABLE metrics (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scan_run_id     UUID NOT NULL REFERENCES scan_runs(id) ON DELETE CASCADE,
    org_id          UUID NOT NULL,
    dataset_node_id UUID NOT NULL REFERENCES graph_nodes(id),
    column_node_id  UUID REFERENCES graph_nodes(id),
    metric_type     VARCHAR(50) NOT NULL,
    dimension       VARCHAR(50) NOT NULL,
    value           DOUBLE PRECISION NOT NULL,
    context         JSONB DEFAULT '{}',
    measured_at     TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (measured_at);

CREATE INDEX idx_metrics_lookup ON metrics(dataset_node_id, column_node_id, metric_type, measured_at DESC);

-- ============================================================
-- ANOMALY BASELINES
-- ============================================================

CREATE TABLE metric_baselines (
    dataset_node_id UUID NOT NULL REFERENCES graph_nodes(id),
    column_node_id  UUID REFERENCES graph_nodes(id),
    metric_type     VARCHAR(50) NOT NULL,
    baseline_mean   DOUBLE PRECISION NOT NULL,
    baseline_stddev DOUBLE PRECISION NOT NULL,
    baseline_median DOUBLE PRECISION,
    baseline_mad    DOUBLE PRECISION,
    sample_count    INTEGER NOT NULL,
    window_days     INTEGER NOT NULL DEFAULT 30,
    last_updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (dataset_node_id, COALESCE(column_node_id, '00000000-0000-0000-0000-000000000000'::uuid), metric_type)
);

-- ============================================================
-- ALERTS
-- ============================================================

CREATE TABLE alerts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    node_id         UUID NOT NULL REFERENCES graph_nodes(id),  -- alert node in graph
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    check_result_id UUID REFERENCES check_results(id),
    severity        VARCHAR(20) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    ai_explanation  TEXT,
    status          VARCHAR(20) NOT NULL DEFAULT 'open',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at     TIMESTAMPTZ
);

CREATE INDEX idx_alerts_org_status ON alerts(org_id, status);
CREATE INDEX idx_alerts_created ON alerts(created_at DESC);

-- ============================================================
-- INCIDENTS
-- ============================================================

CREATE TABLE incidents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    node_id         UUID NOT NULL REFERENCES graph_nodes(id),  -- incident node in graph
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    title           VARCHAR(500) NOT NULL,
    severity        VARCHAR(20) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'open',
    root_cause      TEXT,
    resolution      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at     TIMESTAMPTZ
);

CREATE INDEX idx_incidents_org_status ON incidents(org_id, status);

-- ============================================================
-- AUDIT LOG
-- ============================================================

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    actor_node_id   UUID REFERENCES graph_nodes(id),
    action          VARCHAR(100) NOT NULL,
    target_node_id  UUID REFERENCES graph_nodes(id),
    changes         JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_org ON audit_log(org_id, created_at DESC);
CREATE INDEX idx_audit_target ON audit_log(target_node_id);
```

---

## Example Queries

### Hierarchical Queries (ltree)

```sql
-- All tables in the "public" schema of "analytics" database in "snowflake_prod" source
SELECT id, name, properties
FROM graph_nodes
WHERE org_id = '<org-id>'
  AND path <@ 'src.snowflake_prod.analytics.public'::ltree
  AND node_type = 'dataset';

-- All columns of a specific table
SELECT id, name, properties
FROM graph_nodes
WHERE path <@ 'src.snowflake_prod.analytics.public.orders'::ltree
  AND node_type = 'column'
ORDER BY (properties->>'ordinal')::int;

-- Full path of any node (what source/database/schema does this table belong to?)
SELECT id, name, node_type, path
FROM graph_nodes
WHERE path @> (SELECT path FROM graph_nodes WHERE id = '<table-node-id>')
ORDER BY nlevel(path);
```

### Lineage Traversal (Graph Queries)

```sql
-- Upstream lineage: what feeds into this dataset? (recursive traversal)
WITH RECURSIVE upstream AS (
    -- Start from the target dataset
    SELECT e.source_node_id, e.target_node_id, e.edge_type, e.properties, 1 AS depth
    FROM graph_edges e
    WHERE e.target_node_id = '<dataset-node-id>'
      AND e.edge_type = 'transforms'

    UNION ALL

    -- Walk upstream
    SELECT e.source_node_id, e.target_node_id, e.edge_type, e.properties, u.depth + 1
    FROM graph_edges e
    JOIN upstream u ON e.target_node_id = u.source_node_id
    WHERE e.edge_type = 'transforms'
      AND u.depth < 10
)
SELECT n.name, n.node_type, n.properties, u.depth,
       u.properties->>'job_name' AS via_job
FROM upstream u
JOIN graph_nodes n ON u.source_node_id = n.id
ORDER BY u.depth;

-- Downstream impact: what is affected if this dataset has a quality issue?
WITH RECURSIVE downstream AS (
    SELECT e.source_node_id, e.target_node_id, e.edge_type, 1 AS depth
    FROM graph_edges e
    WHERE e.source_node_id = '<dataset-node-id>'
      AND e.edge_type = 'transforms'

    UNION ALL

    SELECT e.source_node_id, e.target_node_id, e.edge_type, d.depth + 1
    FROM graph_edges e
    JOIN downstream d ON e.source_node_id = d.target_node_id
    WHERE e.edge_type = 'transforms'
      AND d.depth < 10
)
SELECT n.name, n.properties->>'tags' AS tags, d.depth
FROM downstream d
JOIN graph_nodes n ON d.target_node_id = n.id
ORDER BY d.depth;

-- Column-level lineage: trace a column through transformations
WITH RECURSIVE col_lineage AS (
    SELECT e.source_node_id, e.target_node_id, e.properties, 1 AS depth
    FROM graph_edges e
    WHERE e.target_node_id = '<column-node-id>'
      AND e.edge_type = 'column_maps_to'

    UNION ALL

    SELECT e.source_node_id, e.target_node_id, e.properties, cl.depth + 1
    FROM graph_edges e
    JOIN col_lineage cl ON e.target_node_id = cl.source_node_id
    WHERE e.edge_type = 'column_maps_to'
      AND cl.depth < 20
)
SELECT n.name AS column_name,
       parent.name AS table_name,
       cl.properties->>'transformation' AS transform,
       cl.depth
FROM col_lineage cl
JOIN graph_nodes n ON cl.source_node_id = n.id
JOIN graph_edges pe ON pe.target_node_id = n.id AND pe.edge_type = 'contains'
JOIN graph_nodes parent ON pe.source_node_id = parent.id AND parent.node_type = 'dataset'
ORDER BY cl.depth;
```

### Root-Cause Analysis

```sql
-- Given a failing check result, find the probable root cause:
-- 1. Get the dataset the check monitors
-- 2. Traverse upstream lineage
-- 3. Check for recent failures or schema changes on upstream datasets

WITH failing_check AS (
    SELECT cr.dataset_node_id, cr.check_node_id, cr.status, cr.executed_at
    FROM check_results cr
    WHERE cr.id = '<failing-check-result-id>'
),
upstream_datasets AS (
    SELECT DISTINCT n.id AS dataset_id, n.name, u.depth
    FROM (
        WITH RECURSIVE up AS (
            SELECT e.source_node_id, e.target_node_id, 1 AS depth
            FROM graph_edges e, failing_check fc
            WHERE e.target_node_id = fc.dataset_node_id
              AND e.edge_type = 'transforms'
            UNION ALL
            SELECT e.source_node_id, e.target_node_id, up.depth + 1
            FROM graph_edges e
            JOIN up ON e.target_node_id = up.source_node_id
            WHERE e.edge_type = 'transforms' AND up.depth < 5
        ) SELECT * FROM up
    ) u
    JOIN graph_nodes n ON u.source_node_id = n.id
)
-- Find recent failures on upstream datasets
SELECT ud.name AS upstream_table,
       ud.depth AS hops_away,
       cr.status,
       cr.result_detail,
       cr.executed_at
FROM upstream_datasets ud
JOIN check_results cr ON cr.dataset_node_id = ud.dataset_id
WHERE cr.status IN ('failed', 'errored')
  AND cr.executed_at > (SELECT executed_at - INTERVAL '24 hours' FROM failing_check)
ORDER BY ud.depth, cr.executed_at DESC;
```

### Ownership & Team Queries

```sql
-- All datasets owned by a team
SELECT n.name, n.properties
FROM graph_nodes n
JOIN graph_edges e ON e.target_node_id = n.id
WHERE e.source_node_id = '<team-node-id>'
  AND e.edge_type = 'owns'
  AND n.node_type = 'dataset';

-- Which teams are affected by an incident? (traverse incident -> alerts -> checks -> datasets -> owners)
SELECT DISTINCT owner.name AS team_name
FROM graph_edges ie
JOIN graph_edges ae ON ae.source_node_id = ie.target_node_id AND ae.edge_type = 'raised'
JOIN graph_edges me ON me.source_node_id = ae.source_node_id AND me.edge_type = 'monitors'
JOIN graph_edges oe ON oe.target_node_id = me.target_node_id AND oe.edge_type = 'owns'
JOIN graph_nodes owner ON oe.source_node_id = owner.id AND owner.node_type = 'team'
WHERE ie.source_node_id = '<incident-node-id>'
  AND ie.edge_type = 'grouped_in';
```

---

## Graph Maintenance

```sql
-- ============================================================
-- GRAPH STATISTICS (for query optimisation)
-- ============================================================

CREATE TABLE graph_stats (
    org_id          UUID NOT NULL,
    node_type       VARCHAR(50) NOT NULL,
    node_count      BIGINT NOT NULL DEFAULT 0,
    edge_count      BIGINT NOT NULL DEFAULT 0,  -- edges where this type is the source
    last_updated    TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (org_id, node_type)
);

-- ============================================================
-- GRAPH CHANGE LOG (track graph mutations for CDC/streaming)
-- ============================================================

CREATE TABLE graph_changelog (
    id              BIGSERIAL PRIMARY KEY,
    org_id          UUID NOT NULL,
    change_type     VARCHAR(20) NOT NULL,  -- node_created, node_updated, node_deleted, edge_created, edge_deleted
    entity_type     VARCHAR(10) NOT NULL,  -- node, edge
    entity_id       UUID NOT NULL,
    previous_state  JSONB,
    new_state       JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_changelog_org ON graph_changelog(org_id, created_at DESC);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Core | 2 | graph_nodes, graph_edges |
| Graph Infrastructure | 2 | graph_stats, graph_changelog |
| Organisation | 1 | organisations (with back-reference to graph node) |
| Scan Execution | 1 | scan_runs |
| Results & Metrics | 3 | check_results (partitioned), metrics (partitioned), metric_baselines |
| Alerts & Incidents | 2 | alerts, incidents |
| Audit | 1 | audit_log (partitioned) |
| **Total** | **12** | All entity types and relationships live in graph_nodes/graph_edges |

---

## Key Design Decisions

1. **Generic property graph over typed tables.** Every entity — data source, dataset, column, check, pipeline, user, team, alert, incident — is a row in `graph_nodes` with a `node_type` discriminator. Every relationship is a row in `graph_edges` with an `edge_type` discriminator. This makes the graph infinitely extensible: adding a new entity type (e.g., "dashboard", "ML model") requires zero schema changes. The trade-off is that type-specific constraints must be enforced at the application layer.

2. **ltree for hierarchical navigation.** The `path` column on `graph_nodes` uses PostgreSQL's `ltree` extension to encode the containment hierarchy: source > database > schema > table > column. This enables fast subtree queries (`WHERE path <@ 'src.snowflake_prod.analytics.public'`) without recursive CTEs, making "show all tables in this schema" a single indexed lookup.

3. **Operational data in dedicated relational tables.** While entity definitions and relationships live in the graph, high-volume time-series data (check results, metrics, audit log) lives in separate relational tables with proper partitioning. This avoids bloating the graph with millions of ephemeral result nodes and keeps time-range queries efficient.

4. **Graph nodes referenced by operational tables.** Check results reference `check_node_id`, `dataset_node_id`, and `column_node_id` in the graph. This means a dashboard query for "latest results for dataset X" is a simple relational query on `check_results`, but following up with "what upstream datasets could have caused this failure?" is a graph traversal from the same `dataset_node_id`.

5. **Edge types encode relationship semantics.** The `edge_type` column distinguishes between `transforms` (data lineage), `monitors` (check-to-dataset), `owns` (team ownership), `contains` (hierarchy), `column_maps_to` (column lineage), and others. Combined with `properties` JSONB, edges carry rich context (transformation SQL, confidence scores, discovery method) without additional tables.

6. **Graph changelog for CDC.** The `graph_changelog` table records all mutations to the graph. This enables Change Data Capture (CDC) to stream graph changes to Kafka/Pulsar for real-time lineage updates, and provides a time-travel capability for the graph itself.

7. **Weight column for graph algorithms.** The `weight` column on `graph_edges` supports weighted graph algorithms: shortest path for root-cause analysis, PageRank for identifying critical datasets, community detection for team boundary analysis. These are computed by the application layer or a graph analytics library (NetworkX, Apache AGE).

8. **Back-references from relational to graph.** Tables like `organisations`, `scan_runs`, `alerts`, and `incidents` include a `node_id` column that links back to their graph node. This allows the application to seamlessly transition between relational queries (for performance) and graph queries (for traversal) on the same entity.

9. **12 tables total.** The graph-relational model achieves the lowest table count because the generic node/edge tables absorb all entity type variability. The normalised model's 28 tables and the hybrid model's 17 tables are compressed into 2 graph tables + 10 operational tables.

10. **No separate lineage tables.** Unlike the other models which have dedicated `lineage_edges` and `column_lineage_edges` tables, this model stores all lineage relationships as graph edges with `edge_type = 'transforms'` and `edge_type = 'column_maps_to'`. Lineage is not a feature bolted onto the schema — it is the schema.
