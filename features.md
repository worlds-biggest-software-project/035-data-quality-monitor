# Data Quality Monitor — Feature & Functionality Survey

> Candidate #35 · Researched: 2026-05-02

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Great Expectations (GX Core) | Open Source + Commercial Cloud | Apache 2.0 (Core); proprietary GX Cloud | https://greatexpectations.io |
| Soda Core / Soda Cloud | Open Source + Commercial Cloud | Apache 2.0 (Core); proprietary Cloud | https://soda.io |
| Monte Carlo | Commercial (Enterprise) | Proprietary SaaS; custom pricing | https://www.montecarlodata.com |
| Anomalo | Commercial (Enterprise) | Proprietary SaaS; custom pricing | https://www.anomalo.com |
| Bigeye | Commercial (Mid-market) | Proprietary SaaS | https://www.bigeye.com |
| Metaplane | Commercial (SMB) | Proprietary SaaS | https://www.metaplane.dev |
| Deequ (AWS) | Open Source | Apache 2.0 | https://github.com/awslabs/deequ |
| dbt Tests | Open Source + Commercial Cloud | Apache 2.0 (dbt Core) | https://www.getdbt.com |
| Sparvi | Commercial (SMB) | Proprietary SaaS | https://www.sparvi.io |

## Feature Analysis by Solution

### Great Expectations (GX Core)

**Core features**
- Python-based framework for defining and running "expectations" (declarative assertions) against datasets
- Supports Pandas, Apache Spark, and SQLAlchemy backends — validates data at rest or in transit
- Expectation Suites: named collections of assertions that can be versioned and shared across teams
- Validation Results: structured JSON output for each suite run, consumable by CI/CD systems, alerting platforms, or dashboards
- Data Docs: auto-generated HTML documentation of expectations and validation history for each dataset
- Batch-level and streaming validation (Kafka, Kinesis integration for real-time use cases)
- 50+ built-in expectation types covering nullness, uniqueness, data type, range, regex pattern, referential integrity, and distribution checks
- GX Cloud: managed scheduling, centralised result storage, team collaboration, and alerting on top of GX Core

**Differentiating features**
- Longest track record in the category (founded 2017); de facto OSS standard for the unit-test-for-data paradigm
- Expectation Gallery: community-contributed expectations beyond the built-in set
- Data Docs provide always-current, shareable documentation of data quality expectations — no equivalent exists in other OSS tools
- Python-first design allows complex conditional logic, cross-column assertions, and custom statistical checks unavailable in SQL-only tools

**UX patterns**
- CLI and Python API for authoring; no drag-and-drop UI in the open-source version
- GX Cloud adds a browser UI for scheduling, viewing results, and managing suites collaboratively
- Steep initial learning curve: profiling, context configuration, and expectation suite setup require several hours for new users
- "Cold start" problem: expectation authoring is entirely manual; no suggestion engine for initial suite population

**Integration points**
- Orchestrator plugins: Airflow operator, Prefect task, Dagster sensor for embedding validations in pipeline DAGs
- Slack, PagerDuty, and webhook notifications for validation failures
- dbt integration via dbt-expectations package (community-maintained)
- GitHub Actions and GitLab CI/CD for schema and data validation on pull requests

**Known gaps**
- Every expectation must be manually authored; no ML-based inference of appropriate checks from data samples
- No built-in anomaly detection or time-series drift monitoring; purely rule-based
- No native data lineage tracking; GX validates data but does not explain where bad data originated
- GX Cloud pricing (low-thousands per month for team tier) is a significant jump from the free OSS core
- Four native dbt tests are not surfaced inside GX without additional configuration; the two ecosystems are complementary but not unified

**Licence / IP notes**
- GX Core: Apache 2.0 — fully permissive; suitable for embedding in commercial products
- GX Cloud: proprietary SaaS
- No known patent encumbrances

---

### Soda Core / Soda Cloud

**Core features**
- YAML-based data check definitions ("SodaCL" — Soda Checks Language) covering 50+ built-in check types
- Dataset-level metrics: row count, schema changes, partition counts
- Column-level metrics: missing value percentage, duplicate percentage, min/max/mean/stddev, and custom SQL metrics
- Diagnostics Warehouse: stores all scan results, failed records, and historical data quality metrics directly in the customer's own data warehouse (2025 feature)
- Collaborative Authoring UI in Soda Cloud for joint business/technical check definition
- Three-tier architecture: Soda Core (engine) + Soda Agent (deployment proxy) + Soda Cloud (managed layer for scheduling, alerting, collaboration)
- Data Contracts: schema and quality contracts enforced between data producers and consumers

**Differentiating features**
- SodaCL YAML syntax is more approachable than GX Python for non-engineers while remaining expressive
- Diagnostics Warehouse keeps all observability data inside the customer's cloud environment — strong data sovereignty story
- Data Contracts as a first-class concept: teams can negotiate and enforce schema/quality guarantees across pipeline boundaries
- Soda Cloud's Collaborative Authoring UI bridges technical and business user check authorship better than any other tool in this tier

**UX patterns**
- YAML file editing for check authorship in Core; cloud UI for scheduling, monitoring, and collaboration
- CLI-based scan execution; integrates into CI/CD with exit codes and structured results
- Faster onboarding than Great Expectations; YAML checks are more readable and require less Python knowledge
- Anomaly detection (Soda Cloud paid tier only): ML-based flagging without manual threshold setting

**Integration points**
- dbt integration: Soda can scan dbt model outputs as part of post-run validation
- Airflow, Prefect, Dagster operators for pipeline-embedded quality gates
- Snowflake, BigQuery, Redshift, Databricks, DuckDB, and 20+ warehouse/lake backends
- Slack, PagerDuty, Teams alerting; webhook support for custom notification routing

**Known gaps**
- ML anomaly detection gated behind the paid Soda Cloud tier; Core is rules-only
- No data lineage features; Soda identifies which tables fail checks but does not trace root causes through upstream pipelines
- Soda Cloud pricing ($500–$1,500/month for small teams) may be disproportionate for teams that only need basic rule checks
- Community ecosystem smaller than Great Expectations; fewer contributed check templates
- Time-series volume and freshness trending is available but less sophisticated than Monte Carlo's ML baseline approach

**Licence / IP notes**
- Soda Core: Apache 2.0
- Soda Cloud: proprietary SaaS
- No known patent encumbrances

---

### Monte Carlo

**Core features**
- ML-powered automatic baseline detection for table volume, freshness, schema, and distribution without manual rule authorship
- End-to-end data lineage tracking: automatically maps table-level and column-level dependencies across the data estate
- Automated root-cause analysis: when an anomaly fires, Monte Carlo traces lineage to identify probable upstream source and visualises the dependency path
- Agent Observability (2025): monitors ML model inputs, LLM data feeds, and AI agent outputs; deploys LLM-as-judge evaluations for GenAI pipelines
- Incident management workflow: alerts grouped into incidents with assignment, comments, and resolution tracking
- Field Health monitors: per-column distribution tracking using statistical baselines updated continuously

**Differentiating features**
- Pioneered the "data observability" category in 2019; best-in-class ML anomaly detection with no manual rule authorship required
- Only enterprise-grade tool with built-in AI/ML pipeline observability and LLM-as-judge evaluation capabilities (2025)
- Root-cause analysis that links anomalies to specific upstream code changes, pipeline failures, or schema changes — no other OSS or SMB tool offers this
- Metadata-only approach: Monte Carlo scans metadata and query history rather than moving data, preserving warehouse security posture

**UX patterns**
- Browser-based dashboard; anomaly feed, lineage graph, and incident tracker as primary surfaces
- No manual rule authorship required for baseline anomaly detection — onboarding primarily configuration of warehouse connections
- Initial 2–4 weeks of baseline data collection before reliable anomaly detection; zero alerts during this period
- Alert fatigue mitigation: ML models tune signal-to-noise ratio over time using feedback

**Integration points**
- Snowflake, BigQuery, Redshift, Databricks, and major warehouses via metadata API
- Slack, PagerDuty, OpsGenie for alerting
- dbt, Fivetran, Airflow, Dagster integrations for lineage enrichment
- REST API for programmatic incident and lineage data access

**Known gaps**
- No self-hosted option; cloud-only — a blocker for regulated industries with strict data sovereignty requirements
- Custom rule authorship less expressive than Great Expectations or Soda; designed to complement rather than replace rule-based checks
- Enterprise-only pricing ($100K+/year) places it out of reach for SMB and startup buyers
- Initial baseline collection period means zero coverage for newly created tables during the first weeks
- Some users report alert volume can be high before the ML model fully calibrates to seasonal patterns

**Licence / IP notes**
- Proprietary SaaS; no open-source components
- No known patent encumbrances on public record; Monte Carlo holds several trademark registrations for "data observability"

---

### Anomalo

**Core features**
- Unsupervised ML continuously monitors warehouse tables for volume anomalies, NULL/blank value spikes, distribution shifts, and unexpected categorical changes — no rule authoring required
- ML models retrain automatically as new data arrives, adapting to evolving data patterns
- Six Pillars of Data Quality framework (2025): enterprise security, depth of understanding, comprehensive coverage, automated detection, ease of use, and customisation
- Snowflake Native App: fully containerised deployment within the customer's Snowflake environment via Snowpark Container Services — no data leaves the Snowflake perimeter
- Agentic AI investigation layer (2025): autonomously monitors, investigates, and surfaces data quality findings without manual prompting
- Gartner Magic Quadrant recognition for Augmented Data Quality Solutions (first time, 2025)

**Differentiating features**
- No-code, no-rule approach: the only enterprise data quality tool that requires zero upfront rule specification and still generates substantive alerts from day one (after initial ML warm-up)
- Snowflake Native App deployment is unique in the category — strongest data sovereignty story for Snowflake-centric customers
- Detects cross-column and cross-row relationships that pure time-series volume/freshness monitors miss
- Agentic AI investigation is a genuine differentiator for teams without bandwidth to triage every alert manually

**UX patterns**
- Zero-configuration monitoring onboarding: connect to warehouse, select tables, and ML begins learning
- Web UI for alert review, anomaly investigation, and sensitivity tuning
- Alert explanations describe the nature of the anomaly (e.g., "25% increase in NULL values in customer_email since yesterday's load")
- Enterprise positioning means sales-assisted onboarding is the norm; no self-service sign-up

**Integration points**
- Snowflake (deepest integration including Native App), BigQuery, Redshift, Databricks
- Slack, PagerDuty, email alerting
- dbt lineage metadata ingestion for enriched root-cause context
- REST API for incident data export

**Known gaps**
- Enterprise-only pricing; no SMB or self-serve tier
- Non-Snowflake warehouse support is present but the Snowflake Native App is the product's primary differentiator; BigQuery/Redshift experiences are less differentiated
- Custom rule authorship is limited; Anomalo prioritises ML detection over user-defined assertions
- Lineage tracing less mature than Monte Carlo's end-to-end column-level lineage

**Licence / IP notes**
- Proprietary SaaS
- No known patent encumbrances on public record

---

### Bigeye

**Core features**
- Automatic anomaly detection for volume, freshness, schema changes, and distribution shifts at the column level
- Dependency-Driven Monitoring: automatically identifies upstream table dependencies and deploys monitoring only on columns actively consumed by downstream queries — reducing noise from orphaned columns
- Adaptive thresholds: anomaly detection baselines evolve alongside the data without manual tuning
- Freshness monitoring with configurable SLA windows and Slack/PagerDuty alerting
- Column-level lineage to surface which dashboards and models are affected by a detected anomaly

**Differentiating features**
- Dependency-Driven Monitoring is unique: focuses observability budget on columns that are actually in use, not all columns in a table
- Fast time-to-value positioning: measured in hours, not weeks; targeted at mid-market data teams without dedicated observability engineers
- Transparent alert explanations with historical trend context included in the alert notification

**UX patterns**
- Browser-based monitoring dashboard; alert feed with column-level drill-down
- Auto-discovery of tables and lineage from warehouse query history metadata
- Lightweight onboarding compared to Monte Carlo; no baseline waiting period claimed for common check types
- Feedback loop for alert quality: thumbs-up/down on alerts trains the detection model

**Integration points**
- Snowflake, BigQuery, Redshift, Databricks
- dbt metadata ingestion for lineage enrichment
- Slack, PagerDuty, OpsGenie alerting
- dbt Labs acquisition (2023): Bigeye functionality being progressively integrated into the dbt ecosystem

**Known gaps**
- Acquired by dbt Labs in 2023; future as an independent product uncertain — roadmap may be absorbed into dbt Cloud
- Lineage features less mature than Monte Carlo's end-to-end column lineage
- Custom assertion authoring less expressive than Great Expectations or Soda
- No self-hosted option

**Licence / IP notes**
- Proprietary SaaS; acquired by dbt Labs (2023)
- No known patent encumbrances

---

### Deequ (AWS)

**Core features**
- Apache Spark-based data unit-test library for large-scale batch pipeline validation
- Constraint definition API: specify data quality constraints (completeness, uniqueness, consistency, distribution) in Scala or Python
- ConstraintSuggestions: analyses a data sample and suggests an initial set of constraints — an early form of automated rule inference
- Anomaly detection module: tracks metric values over time in a metrics repository and flags deviations
- Metrics Repository: persists computed data quality metrics for trend tracking and historical comparison
- Runs as a native Spark job; no separate service or infrastructure required

**Differentiating features**
- The only major OSS data quality library purpose-built for Apache Spark at scale; handles petabyte-level datasets efficiently
- ConstraintSuggestions provides basic automated rule inference from data profiles — predates GX's manual-authorship model
- Zero additional infrastructure cost beyond existing Spark cluster

**UX patterns**
- Code-only interface: Scala or PySpark API; no web UI
- Integrated into data engineering pipelines as a library, not a standalone service
- Primarily used by data engineering teams comfortable with Spark; minimal accessibility to business users

**Integration points**
- Apache Spark (required dependency)
- AWS Glue, EMR, Databricks for managed Spark environments
- Custom storage backends for the metrics repository (S3, HDFS)
- No native BI or alerting integrations; results must be exported and consumed by external systems

**Known gaps**
- No web UI; purely code-driven
- Requires Spark expertise and infrastructure; high barrier for teams not already running Spark
- No real-time or streaming data quality checks; batch-only
- Community maintenance has slowed since Monte Carlo and commercial alternatives raised the bar for full-featured observability
- No lineage, incident management, or alerting built in

**Licence / IP notes**
- Apache 2.0
- Originally published by Amazon Research as part of the paper "Automating Large-Scale Data Quality Verification" (VLDB 2018)
- No known patent encumbrances; Amazon has not asserted patents over Deequ's open-source release

---

### dbt Tests

**Core features**
- Four native test types: not_null, unique, accepted_values, relationships — defined in YAML alongside model definitions
- Custom generic tests: reusable test macros applicable across multiple models
- Singular tests: one-off SQL assertions for complex business-logic checks
- Test severity levels: warn vs. error; allows pipeline to continue on warnings but halt on errors
- dbt-expectations package (community, Apache 2.0): ports Great Expectations' expectation types into dbt test macros, extending the native four to 50+
- Test results stored in dbt Cloud or surfaced via dbt docs; integration with downstream alerting via orchestrator callbacks

**Differentiating features**
- Zero additional tooling required for teams already using dbt; tests live in the same repo as transformation models
- Version-controlled quality assertions that follow the same PR/review workflow as SQL models
- Lineage graph in dbt includes test nodes, making data quality visible alongside transformation dependencies

**UX patterns**
- YAML authoring alongside model files; familiar to analytics engineers
- Runs as part of the dbt build/test command; failures output to CLI and dbt Cloud UI
- Results accessible in dbt Cloud's browser UI alongside model metadata
- No monitoring layer: dbt tests run on demand or on a schedule; no continuous background monitoring

**Integration points**
- All dbt-supported warehouses (Snowflake, BigQuery, Redshift, Databricks, DuckDB, and others)
- Airflow, Dagster, Prefect operators trigger dbt test runs as pipeline steps
- dbt Cloud CI/CD integration: tests run on pull requests before model promotion
- Elementary (OSS): extends dbt tests with monitoring, alerting, and trend dashboards without leaving the dbt ecosystem

**Known gaps**
- Purely rule-based; no anomaly detection, time-series drift monitoring, or ML-based checks
- Four native tests are limited; complex statistical assertions require third-party packages or custom SQL
- No continuous background monitoring: tests run only when dbt is explicitly invoked
- Cannot validate data not produced by dbt (e.g., raw source tables before transformation)
- Test failure notifications require external orchestrator or alerting configuration; no built-in Slack/PagerDuty integration in Core

**Licence / IP notes**
- dbt Core: Apache 2.0
- dbt-expectations: Apache 2.0
- No known patent encumbrances

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Rule-based validation for the standard data quality dimensions: completeness (nullness), uniqueness, referential integrity, value range, and format
- Schema drift detection: alert when columns are added, removed, or change type
- Freshness monitoring: alert when a table has not been updated within a defined SLA window
- Volume monitoring: alert when row counts deviate unexpectedly from historical norms
- Scheduling and continuous monitoring (not just on-demand test runs)
- Alerting via Slack and PagerDuty (the two dominant channels in data engineering teams)
- Integration with dbt and major orchestrators (Airflow, Dagster, Prefect)
- Support for Snowflake, BigQuery, Redshift, and Databricks as primary targets

### Differentiating Features
- ML-based anomaly detection that learns baselines without manual threshold authorship (Monte Carlo, Anomalo)
- Automated root-cause analysis linking anomaly back to lineage, recent pipeline changes, or upstream failures
- No-rule onboarding: system generates initial quality checks from data profiles without engineer input
- Data sovereignty deployment (Snowflake Native App, Diagnostics Warehouse in customer environment)
- AI/LLM pipeline observability: monitoring data fed to and generated by GenAI systems
- Natural-language alert explanations that narrate the "why" rather than only the metric deviation
- Data Contracts: formalised schema and quality agreements between data producers and consumers

### Underserved Areas / Opportunities
- Rule authorship cold-start: all rule-based OSS tools require manual specification; no OSS tool infers initial rules from data semantics
- Cross-table relationship discovery: implicit business rules (every order must have a matching customer) are not automatically detected by any current tool
- SMB/startup segment: open-source tools demand engineering investment; enterprise tools are priced out of reach; the mid-ground is poorly served
- Natural-language alert explanations with lineage context are available only in the highest-cost commercial tier
- Unified rule-based + ML-based quality in a single open-source product does not exist; teams must compose multiple tools

### AI-Augmentation Candidates
- Automated expectation/check generation: infer appropriate quality rules from schema metadata, column semantics, and data distributions
- Cold-start bootstrapping: use LLM-based semantic analysis of column names and table documentation to generate plausible initial checks without historical data
- Natural-language alert narration: translate metric anomalies into English explanations with probable cause and downstream impact
- Cross-table referential integrity discovery: analyse join patterns and column name similarity to auto-generate relationship checks
- Root-cause reasoning: given a quality failure, trace lineage and link to recent code changes, pipeline failures, or upstream schema changes with AI-generated narrative

## Legal & IP Summary

The open-source tools surveyed (Great Expectations Core, Soda Core, Deequ, dbt Core) are all Apache 2.0 licensed. Apache 2.0 includes an explicit patent grant, meaning contributors grant users a royalty-free licence to any patents necessarily infringed by their contributions. Deequ originated at Amazon; Amazon has not asserted patents over its open-source release. The commercial tools (Monte Carlo, Anomalo, Bigeye/dbt Labs) are proprietary SaaS; their ML anomaly detection algorithms and user interfaces are not available for reuse. Monte Carlo holds trademark registrations around "data observability" terminology, though this affects branding, not technical implementation. No data quality-specific patents have been identified in public records that would encumber common rule-based validation or time-series anomaly detection implementations.

## Recommended Feature Scope

**Must-have (MVP)**:
- Connection to Snowflake, BigQuery, Redshift, and Databricks
- Rule-based checks for nullness, uniqueness, value ranges, freshness, and volume (covering the primary ISO/IEC 25012 quality dimensions)
- Schema drift detection with configurable alerting (email + Slack)
- Scheduled continuous monitoring (not just on-demand runs)
- Integration with dbt: run quality checks as post-transformation validation steps
- Web UI for viewing check results, alert history, and per-column metrics over time

**Should-have (v1.1)**:
- AI-generated initial check suggestions from data profiles and column semantics (eliminating cold-start friction)
- ML-based baseline anomaly detection for volume and distribution drift without manual threshold authorship
- Natural-language alert explanations narrating probable root cause and downstream impact
- Data lineage integration to trace anomalies to their upstream origin pipeline or code change
- Slack and PagerDuty native alerting with alert suppression and acknowledgement workflows

**Nice-to-have (backlog)**:
- Cross-table referential integrity discovery from column name and join pattern analysis
- Data Contracts: formalised schema and quality agreements between producers and consumers
- Agentic AI investigation mode: autonomous anomaly triage with no human prompting required
- Streaming data quality validation (Kafka, Kinesis)
- Diagnostics Warehouse: persist all scan results in the customer's own warehouse for full data sovereignty
