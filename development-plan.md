# Data Quality Monitor — Development Plan

> Project: 035-data-quality-monitor
> Generated: 2026-05-25
> Based on: research.md, features.md, standards.md, README.md, data-model-suggestion-1..4.md

---

## 1. Technology Decisions

### 1.1 Data Model: Hybrid Relational + JSONB (Suggestion 3, adapted)

**Decision:** Adopt Data Model Suggestion 3 (Hybrid Relational + JSONB) as the foundation, with selective borrowing from Suggestion 1 (normalised schema for `dataset_columns` as a first-class table) and Suggestion 4 (graph-style lineage edges with JSONB properties).

**Rationale:**
- **MVP velocity.** The hybrid model has 17 tables vs. 28 in the fully normalised model. JSONB columns absorb check-type variability (50+ check types with different parameters) without requiring schema migrations for each new type.
- **Normalised columns.** Suggestion 3 embeds column metadata in a JSONB `profile` field on `datasets`. This saves a JOIN for simple reads but makes column-level checks, metrics, and lineage awkward to reference. Borrowing `dataset_columns` from Suggestion 1 gives each column a stable UUID for foreign-key references in check results and metrics, while keeping column stats in JSONB.
- **Graph-style lineage.** Suggestion 4's `graph_nodes`/`graph_edges` architecture is powerful but adds operational complexity prematurely. Instead, use Suggestion 3's `lineage_edges` table with JSONB `detail` (column mappings, confidence, discovery method). This can be migrated to a graph model in a later phase if root-cause analysis demands it.
- **Event sourcing deferred.** Suggestion 2's CQRS/event-sourced model provides excellent audit trails but doubles implementation complexity. For the MVP, an append-only `audit_log` table (from Suggestion 1) provides regulatory coverage. Event sourcing can be layered on in Phase 8+ for regulated-industry deployments.
- **Standards alignment.** ISO/IEC 25012 quality dimensions as a VARCHAR enum (from Suggestion 1). JSON Schema Draft 2020-12 for validating JSONB check parameters (from Suggestion 3). OpenLineage-aligned entity naming for future interoperability.

### 1.2 Technology Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| **Primary Language** | Python 3.12+ | Dominant in data engineering; direct compatibility with Great Expectations, Soda Core, dbt, and ML libraries (scikit-learn, Prophet). Enables easy import of existing GX expectation suites. |
| **API Framework** | FastAPI | Async-first, OpenAPI auto-generation, Pydantic for request/response validation. Aligns with the standards.md recommendation for REST/OpenAPI interface. |
| **Database** | PostgreSQL 16+ | JSONB with GIN indexes, table partitioning, Row-Level Security, `ltree` extension available for future graph migration. All four data model suggestions target PostgreSQL. |
| **ORM / Query** | SQLAlchemy 2.0 + Alembic | Type-safe ORM with JSONB support; Alembic for versioned migrations. Industry standard for Python+PostgreSQL. |
| **Task Scheduler** | Celery + Redis | Cron-based scan scheduling, async check execution, distributed worker support. Redis as broker and result backend. |
| **Frontend** | React 19 + TypeScript 5 | React is the standard for data-tool UIs (Monte Carlo, Bigeye, OpenMetadata all use React). TypeScript for type safety. |
| **UI Component Library** | shadcn/ui + Tailwind CSS 4 | Modern, accessible components; consistent with the data platform aesthetic. Chart library: Recharts for time-series metric visualisation. |
| **ML / Anomaly Detection** | scikit-learn (Isolation Forest, LOF), statsmodels (seasonal decomposition) | Lightweight, no GPU required. Sufficient for z-score, MAD, and seasonal baseline detection. Prophet or NeuralProphet for advanced seasonality in later phases. |
| **LLM Integration** | Anthropic Claude API (claude-sonnet-4-20250514) | For AI-generated check suggestions, natural-language alert explanations, and cross-table relationship discovery. Claude selected for structured output reliability and cost efficiency at mid-market pricing. |
| **Authentication** | OAuth 2.0 (authorization code + client credentials) | Matches the industry pattern documented in standards.md. OIDC via Google/GitHub for user login; API keys (Bearer token) for service-to-service. |
| **Containerisation** | Docker + Docker Compose | Single `docker compose up` for local development. Production deployments via Helm charts (Kubernetes) in later phases. |
| **Testing** | pytest + pytest-asyncio (backend), Vitest + Playwright (frontend) | pytest is the Python standard. Playwright for end-to-end UI tests. |
| **CI/CD** | GitHub Actions | Lint, test, build, and publish on every PR. Docker image publishing to GHCR. |
| **Licence** | Apache 2.0 | Matches Great Expectations, Soda Core, Deequ. Includes explicit patent grant. Enables commercial embedding. |

### 1.3 Project Structure

```
data-quality-monitor/
├── .github/
│   └── workflows/
│       ├── ci.yml                  # Lint + test + build on PR
│       └── release.yml             # Docker image publish on tag
├── docker/
│   ├── Dockerfile                  # Multi-stage backend image
│   ├── Dockerfile.frontend         # Frontend build image
│   └── docker-compose.yml          # Local dev stack (API + DB + Redis + frontend)
├── alembic/
│   ├── alembic.ini
│   └── versions/                   # Numbered migration files
├── src/
│   ├── dqm/                        # Main Python package
│   │   ├── __init__.py
│   │   ├── config.py               # Settings via pydantic-settings
│   │   ├── main.py                 # FastAPI app entrypoint
│   │   ├── models/                 # SQLAlchemy ORM models
│   │   │   ├── __init__.py
│   │   │   ├── organisation.py
│   │   │   ├── data_source.py
│   │   │   ├── dataset.py
│   │   │   ├── check.py
│   │   │   ├── scan.py
│   │   │   ├── metric.py
│   │   │   ├── alert.py
│   │   │   ├── incident.py
│   │   │   ├── lineage.py
│   │   │   ├── contract.py
│   │   │   └── audit.py
│   │   ├── api/                    # FastAPI routers
│   │   │   ├── __init__.py
│   │   │   ├── sources.py
│   │   │   ├── datasets.py
│   │   │   ├── checks.py
│   │   │   ├── scans.py
│   │   │   ├── alerts.py
│   │   │   ├── incidents.py
│   │   │   ├── metrics.py
│   │   │   ├── lineage.py
│   │   │   ├── contracts.py
│   │   │   └── auth.py
│   │   ├── connectors/             # Warehouse connection adapters
│   │   │   ├── __init__.py
│   │   │   ├── base.py             # Abstract connector interface
│   │   │   ├── snowflake.py
│   │   │   ├── bigquery.py
│   │   │   ├── redshift.py
│   │   │   ├── databricks.py
│   │   │   └── postgres.py         # For testing and self-monitoring
│   │   ├── checks/                 # Check execution engine
│   │   │   ├── __init__.py
│   │   │   ├── engine.py           # Check runner and result collector
│   │   │   ├── registry.py         # Check type registry
│   │   │   ├── rule_based.py       # Rule-based check implementations
│   │   │   ├── schema_drift.py     # Schema comparison logic
│   │   │   ├── freshness.py        # Freshness/currentness checks
│   │   │   ├── volume.py           # Row count checks
│   │   │   └── custom_sql.py       # User-defined SQL checks
│   │   ├── profiler/               # Data profiling engine
│   │   │   ├── __init__.py
│   │   │   ├── profiler.py         # Column statistics computation
│   │   │   └── schema_discovery.py # Schema introspection
│   │   ├── anomaly/                # ML anomaly detection (Phase 4)
│   │   │   ├── __init__.py
│   │   │   ├── detector.py         # Anomaly scoring engine
│   │   │   ├── baselines.py        # Baseline computation and storage
│   │   │   └── algorithms.py       # Z-score, MAD, Isolation Forest
│   │   ├── ai/                     # LLM-powered features (Phase 5)
│   │   │   ├── __init__.py
│   │   │   ├── check_generator.py  # AI check suggestion engine
│   │   │   ├── explainer.py        # Natural-language alert explanations
│   │   │   └── relationship.py     # Cross-table relationship discovery
│   │   ├── alerting/               # Alert dispatch
│   │   │   ├── __init__.py
│   │   │   ├── dispatcher.py       # Alert routing engine
│   │   │   ├── slack.py
│   │   │   ├── pagerduty.py
│   │   │   ├── email.py
│   │   │   └── webhook.py
│   │   ├── integrations/           # External tool integrations (Phase 6)
│   │   │   ├── __init__.py
│   │   │   ├── dbt.py              # dbt manifest/results ingestion
│   │   │   ├── airflow.py          # Airflow operator
│   │   │   └── openlineage.py      # OpenLineage event consumer
│   │   ├── scheduler/              # Scan scheduling
│   │   │   ├── __init__.py
│   │   │   └── scheduler.py        # Celery beat configuration
│   │   └── services/               # Business logic layer
│   │       ├── __init__.py
│   │       ├── scan_service.py
│   │       ├── check_service.py
│   │       ├── alert_service.py
│   │       └── metric_service.py
│   └── tests/
│       ├── conftest.py             # Shared fixtures (test DB, factories)
│       ├── unit/
│       ├── integration/
│       └── e2e/
├── frontend/
│   ├── package.json
│   ├── tsconfig.json
│   ├── vite.config.ts
│   ├── src/
│   │   ├── App.tsx
│   │   ├── main.tsx
│   │   ├── components/
│   │   │   ├── layout/
│   │   │   ├── datasets/
│   │   │   ├── checks/
│   │   │   ├── scans/
│   │   │   ├── alerts/
│   │   │   ├── metrics/
│   │   │   └── common/
│   │   ├── pages/
│   │   ├── hooks/
│   │   ├── api/                    # API client (generated from OpenAPI)
│   │   └── types/
│   └── tests/
│       ├── unit/
│       └── e2e/
├── docs/
│   ├── api.md
│   ├── check-types.md
│   └── deployment.md
├── pyproject.toml
├── README.md
└── LICENSE                         # Apache 2.0
```

---

## 2. Phase Dependency Graph

```
Phase 1: Foundation & Data Source Connectivity
    │
    ├──► Phase 2: Check Engine & Rule-Based Validation
    │       │
    │       ├──► Phase 3: Scheduling, Alerting & Web UI
    │       │       │
    │       │       ├──► Phase 4: ML Anomaly Detection
    │       │       │       │
    │       │       │       └──► Phase 5: AI-Powered Features
    │       │       │               │
    │       │       │               └──► Phase 7: Data Contracts & Governance
    │       │       │
    │       │       └──► Phase 6: Integrations (dbt, Airflow, OpenLineage)
    │       │               │
    │       │               └──► Phase 8: Lineage & Root-Cause Analysis
    │       │
    │       └──► Phase 9: Streaming & Real-Time
    │
    └──► Phase 10: Enterprise & Scale
```

**Critical path:** Phases 1 → 2 → 3 → 4 → 5 (AI features require anomaly baselines which require the check engine which requires data sources).

**Parallel tracks after Phase 3:**
- Track A: ML/AI (Phases 4, 5, 7)
- Track B: Integrations/Lineage (Phases 6, 8)
- Track C: Streaming/Enterprise (Phases 9, 10)

---

## 3. Development Phases

---

### Phase 1: Foundation & Data Source Connectivity

**Goal:** Establish the project skeleton, database schema, authentication, and the ability to connect to and introspect at least two warehouse types.

**Duration:** 4 weeks

**Prerequisites:** None

#### Task 1.1: Project Scaffolding

**What:** Create the repository, configure CI/CD, set up Docker Compose for local development (PostgreSQL 16, Redis 7), initialise the FastAPI application, and configure Alembic for database migrations.

**Design:**
```python
# src/dqm/config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    database_url: str = "postgresql+asyncpg://dqm:dqm@localhost:5432/dqm"
    redis_url: str = "redis://localhost:6379/0"
    secret_key: str  # JWT signing key
    cors_origins: list[str] = ["http://localhost:5173"]
    log_level: str = "INFO"

    class Config:
        env_prefix = "DQM_"

# src/dqm/main.py
from fastapi import FastAPI
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: init DB pool, Redis connection
    async with db_engine.begin() as conn:
        pass  # connection pool warmed
    yield
    # Shutdown: close pools

app = FastAPI(
    title="Data Quality Monitor",
    version="0.1.0",
    lifespan=lifespan,
)
```

**Testing:**
- `test_app_starts`: FastAPI app starts without errors and returns 200 on `/health`
- `test_db_connection`: Alembic migrations run successfully against a fresh PostgreSQL instance
- `test_redis_connection`: Redis ping succeeds
- `test_docker_compose_up`: `docker compose up` brings up all services within 60 seconds

#### Task 1.2: Core Database Schema (Migration 001)

**What:** Create the Alembic migration for the foundational tables: `organisations`, `users`, `api_keys`, `data_sources`, `datasets`, `dataset_columns`, `audit_log`.

**Design:**
```python
# alembic/versions/001_foundation.py
def upgrade():
    # organisations table
    op.create_table(
        "organisations",
        sa.Column("id", sa.dialects.postgresql.UUID, primary_key=True,
                  server_default=sa.text("gen_random_uuid()")),
        sa.Column("name", sa.String(255), nullable=False),
        sa.Column("slug", sa.String(100), nullable=False, unique=True),
        sa.Column("plan_tier", sa.String(50), nullable=False, server_default="free"),
        sa.Column("settings", sa.dialects.postgresql.JSONB, nullable=False,
                  server_default="{}"),
        sa.Column("created_at", sa.TIMESTAMP(timezone=True), nullable=False,
                  server_default=sa.text("now()")),
        sa.Column("updated_at", sa.TIMESTAMP(timezone=True), nullable=False,
                  server_default=sa.text("now()")),
    )

    # data_sources table — config as JSONB for multi-warehouse flexibility
    op.create_table(
        "data_sources",
        sa.Column("id", sa.dialects.postgresql.UUID, primary_key=True),
        sa.Column("org_id", sa.dialects.postgresql.UUID,
                  sa.ForeignKey("organisations.id", ondelete="CASCADE"), nullable=False),
        sa.Column("name", sa.String(255), nullable=False),
        sa.Column("source_type", sa.String(50), nullable=False),
        sa.Column("config", sa.dialects.postgresql.JSONB, nullable=False),
        sa.Column("status", sa.String(50), nullable=False, server_default="active"),
        sa.Column("sync_state", sa.dialects.postgresql.JSONB, nullable=False,
                  server_default="{}"),
        # ... timestamps, unique constraint on (org_id, name)
    )

    # datasets, dataset_columns, users, api_keys, audit_log
    # (following the hybrid model with dataset_columns as a first-class table)
```

**Testing:**
- `test_migration_001_upgrade`: Migration applies cleanly to an empty database
- `test_migration_001_downgrade`: Migration rolls back without errors
- `test_org_creation`: Insert an organisation row and verify all defaults are set
- `test_data_source_fk`: Inserting a data source with invalid `org_id` raises `ForeignKeyViolation`
- `test_unique_constraints`: Duplicate `(org_id, name)` on `data_sources` raises `UniqueViolation`

#### Task 1.3: Authentication & API Key Management

**What:** Implement OAuth 2.0 authorization-code flow (Google/GitHub OIDC) for user login and API key creation/validation for service-to-service calls. API keys use the `Authorization: Bearer dqm_<prefix>_<secret>` pattern.

**Design:**
```python
# src/dqm/api/auth.py
from fastapi import Depends, HTTPException, Security
from fastapi.security import HTTPBearer, OAuth2AuthorizationCodeBearer

bearer_scheme = HTTPBearer(auto_error=False)

async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Security(bearer_scheme),
    db: AsyncSession = Depends(get_db),
) -> User:
    token = credentials.credentials
    if token.startswith("dqm_"):
        # API key authentication
        prefix = token[4:12]  # 8-char prefix
        key_row = await db.execute(
            select(ApiKey).where(ApiKey.key_prefix == prefix)
        )
        api_key = key_row.scalar_one_or_none()
        if not api_key or not bcrypt.checkpw(token.encode(), api_key.key_hash.encode()):
            raise HTTPException(status_code=401, detail="Invalid API key")
        return api_key.user or SystemUser(org_id=api_key.org_id)
    else:
        # JWT token authentication (from OAuth flow)
        payload = jwt.decode(token, settings.secret_key, algorithms=["HS256"])
        user = await db.get(User, payload["sub"])
        if not user:
            raise HTTPException(status_code=401)
        return user
```

**Testing:**
- `test_api_key_creation`: Create an API key, verify prefix is stored and full key is returned once
- `test_api_key_auth_valid`: Request with valid API key returns 200
- `test_api_key_auth_invalid`: Request with invalid API key returns 401
- `test_api_key_scopes`: Request exceeding API key's scopes returns 403
- `test_jwt_auth_valid`: Request with valid JWT returns 200 and correct user
- `test_jwt_auth_expired`: Request with expired JWT returns 401
- `test_no_auth`: Request without `Authorization` header returns 401

#### Task 1.4: Data Source Connection & Introspection

**What:** Implement the warehouse connector abstraction and two concrete connectors (PostgreSQL for testing, Snowflake for production). Each connector must: test connectivity, discover schemas/tables, and introspect column metadata.

**Design:**
```python
# src/dqm/connectors/base.py
from abc import ABC, abstractmethod
from dataclasses import dataclass

@dataclass
class ColumnInfo:
    name: str
    data_type: str
    is_nullable: bool
    ordinal_position: int
    comment: str | None = None

@dataclass
class TableInfo:
    schema_name: str
    table_name: str
    table_type: str  # "table" | "view" | "external"
    row_count: int | None = None
    columns: list[ColumnInfo] | None = None

class BaseConnector(ABC):
    @abstractmethod
    async def test_connection(self) -> bool:
        """Verify the connection is valid. Returns True or raises."""

    @abstractmethod
    async def discover_tables(self, schema_filter: str | None = None) -> list[TableInfo]:
        """List all tables/views in the source, optionally filtered by schema."""

    @abstractmethod
    async def introspect_columns(self, schema_name: str, table_name: str) -> list[ColumnInfo]:
        """Get column metadata for a specific table."""

    @abstractmethod
    async def execute_query(self, sql: str, params: dict | None = None) -> list[dict]:
        """Execute a read-only SQL query and return rows as dicts."""

    @abstractmethod
    async def get_row_count(self, schema_name: str, table_name: str) -> int:
        """Get the current row count for a table."""

    @abstractmethod
    async def get_freshness(self, schema_name: str, table_name: str,
                            timestamp_column: str) -> datetime | None:
        """Get the MAX value of a timestamp column (freshness indicator)."""

# src/dqm/connectors/snowflake.py
class SnowflakeConnector(BaseConnector):
    def __init__(self, config: dict):
        self.account = config["account"]
        self.warehouse = config["warehouse"]
        self.database = config["database"]
        self.role = config.get("role")
        self._credentials_ref = config["credentials"]["vault_ref"]

    async def discover_tables(self, schema_filter=None) -> list[TableInfo]:
        query = """
            SELECT table_schema, table_name, table_type, row_count
            FROM information_schema.tables
            WHERE table_schema NOT IN ('INFORMATION_SCHEMA')
        """
        if schema_filter:
            query += " AND table_schema = %(schema)s"
        rows = await self.execute_query(query, {"schema": schema_filter})
        return [TableInfo(**row) for row in rows]
```

**Testing:**
- `test_postgres_connector_test_connection`: Connect to local PostgreSQL, verify `test_connection()` returns True
- `test_postgres_connector_discover_tables`: Create test tables, verify `discover_tables()` returns them with correct metadata
- `test_postgres_connector_introspect_columns`: Verify column names, types, and nullability match the test table definition
- `test_snowflake_connector_mock`: Mock Snowflake connection, verify query generation for `discover_tables()` and `introspect_columns()`
- `test_connector_factory`: Verify `ConnectorFactory.create("snowflake", config)` returns `SnowflakeConnector`
- `test_invalid_source_type`: Verify `ConnectorFactory.create("unknown", config)` raises `UnsupportedSourceType`
- `test_connection_failure`: Verify graceful error handling when connection parameters are invalid

#### Task 1.5: Data Source & Dataset CRUD API

**What:** Implement REST API endpoints for registering data sources, triggering metadata sync (schema/table discovery), and browsing the dataset/column inventory.

**Design:**
```python
# src/dqm/api/sources.py
router = APIRouter(prefix="/api/v1/sources", tags=["Data Sources"])

@router.post("/", response_model=DataSourceResponse, status_code=201)
async def create_source(body: CreateDataSourceRequest, user=Depends(get_current_user)):
    """Register a new data source (warehouse connection)."""

@router.post("/{source_id}/sync", response_model=SyncResponse)
async def sync_source(source_id: UUID, user=Depends(get_current_user)):
    """Trigger metadata sync: discover schemas, tables, and columns."""

@router.get("/{source_id}/datasets", response_model=list[DatasetSummary])
async def list_datasets(source_id: UUID, schema: str | None = None):
    """List all discovered datasets for a source."""

# src/dqm/api/datasets.py
router = APIRouter(prefix="/api/v1/datasets", tags=["Datasets"])

@router.get("/{dataset_id}", response_model=DatasetDetail)
async def get_dataset(dataset_id: UUID):
    """Get dataset details including columns and latest profile."""

@router.get("/{dataset_id}/columns", response_model=list[ColumnDetail])
async def list_columns(dataset_id: UUID):
    """List all columns for a dataset with stats."""
```

**Testing:**
- `test_create_source_success`: POST a valid Snowflake source config, verify 201 and returned ID
- `test_create_source_duplicate_name`: POST a source with existing name, verify 409 Conflict
- `test_sync_source`: POST `/sync`, verify datasets and columns are persisted in the database
- `test_sync_source_detects_new_tables`: Add a table to the test DB, re-sync, verify it appears
- `test_list_datasets_pagination`: Create 50 datasets, verify pagination parameters work
- `test_get_dataset_with_columns`: Verify dataset detail includes embedded column metadata
- `test_audit_log_on_create`: Verify `audit_log` row is created when a source is registered

#### Phase 1 — Definition of Done

- [ ] Repository initialised with CI/CD (GitHub Actions: lint, test, build)
- [ ] `docker compose up` starts PostgreSQL, Redis, and the API server
- [ ] Alembic migration 001 creates all foundational tables
- [ ] API key authentication works end-to-end (create key, use key, reject invalid key)
- [ ] PostgreSQL and Snowflake connectors pass unit and integration tests
- [ ] Data source CRUD API: create, list, get, delete, sync
- [ ] Dataset/column inventory populated by sync and browsable via API
- [ ] Audit log records all write operations
- [ ] Test coverage: >80% on all new code
- [ ] API documentation auto-generated at `/docs` (Swagger UI)

---

### Phase 2: Check Engine & Rule-Based Validation

**Goal:** Build the check definition framework, implement the core rule-based check types (covering ISO 25012 dimensions), and execute checks against connected datasets.

**Duration:** 4 weeks

**Prerequisites:** Phase 1 complete (data sources connected, datasets discoverable)

#### Task 2.1: Check Template & Definition Schema (Migration 002)

**What:** Create the `check_templates` and `checks` tables. Seed the database with built-in check templates covering the primary ISO/IEC 25012 quality dimensions: completeness, uniqueness, validity, currentness, and consistency.

**Design:**
```python
# Seed data: built-in check templates
BUILTIN_TEMPLATES = [
    {
        "name": "not_null",
        "check_type": "rule_based",
        "dimension": "completeness",
        "scope": "column",
        "parameter_schema": {
            "$schema": "https://json-schema.org/draft/2020-12/schema",
            "type": "object",
            "properties": {
                "column": {"type": "string"},
                "max_null_rate": {"type": "number", "minimum": 0, "maximum": 1},
            },
            "required": ["column", "max_null_rate"],
        },
    },
    {
        "name": "unique",
        "check_type": "rule_based",
        "dimension": "uniqueness",
        "scope": "column",
        "parameter_schema": {
            "type": "object",
            "properties": {
                "column": {"type": "string"},
                "max_duplicate_rate": {"type": "number", "minimum": 0, "maximum": 1},
            },
            "required": ["column"],
        },
    },
    {
        "name": "value_range",
        "check_type": "rule_based",
        "dimension": "validity",
        "scope": "column",
        "parameter_schema": {
            "type": "object",
            "properties": {
                "column": {"type": "string"},
                "min_value": {"type": "number"},
                "max_value": {"type": "number"},
            },
            "required": ["column"],
        },
    },
    {
        "name": "accepted_values",
        "check_type": "rule_based",
        "dimension": "validity",
        "scope": "column",
        "parameter_schema": {
            "type": "object",
            "properties": {
                "column": {"type": "string"},
                "values": {"type": "array", "items": {"type": "string"}},
            },
            "required": ["column", "values"],
        },
    },
    {
        "name": "regex_pattern",
        "check_type": "rule_based",
        "dimension": "validity",
        "scope": "column",
        "parameter_schema": {
            "type": "object",
            "properties": {
                "column": {"type": "string"},
                "pattern": {"type": "string"},
                "max_mismatch_rate": {"type": "number", "minimum": 0, "maximum": 1},
            },
            "required": ["column", "pattern"],
        },
    },
    {
        "name": "freshness",
        "check_type": "freshness",
        "dimension": "currentness",
        "scope": "table",
        "parameter_schema": {
            "type": "object",
            "properties": {
                "timestamp_column": {"type": "string"},
                "max_age_hours": {"type": "number", "minimum": 0},
            },
            "required": ["timestamp_column", "max_age_hours"],
        },
    },
    {
        "name": "row_count_range",
        "check_type": "volume",
        "dimension": "completeness",
        "scope": "table",
        "parameter_schema": {
            "type": "object",
            "properties": {
                "min_count": {"type": "integer", "minimum": 0},
                "max_count": {"type": "integer", "minimum": 0},
            },
        },
    },
    {
        "name": "schema_drift",
        "check_type": "schema_drift",
        "dimension": "consistency",
        "scope": "table",
        "parameter_schema": {
            "type": "object",
            "properties": {
                "alert_on": {
                    "type": "array",
                    "items": {"type": "string", "enum": [
                        "column_added", "column_removed", "type_changed", "nullable_changed"
                    ]},
                },
                "ignore_columns": {"type": "array", "items": {"type": "string"}},
            },
        },
    },
    {
        "name": "custom_sql",
        "check_type": "custom_sql",
        "dimension": "consistency",
        "scope": "table",
        "parameter_schema": {
            "type": "object",
            "properties": {
                "sql": {"type": "string"},
                "expected": {"type": "number"},
                "operator": {"type": "string", "enum": ["equals", "less_than", "greater_than"]},
            },
            "required": ["sql", "expected", "operator"],
        },
    },
]
```

**Testing:**
- `test_migration_002`: Migration creates `check_templates` and `checks` tables
- `test_builtin_templates_seeded`: All 9 built-in templates exist after migration
- `test_template_parameter_schema_valid`: Each template's `parameter_schema` is valid JSON Schema Draft 2020-12
- `test_check_config_validation`: Creating a check with config that violates its template's `parameter_schema` raises a validation error
- `test_check_creation_with_valid_config`: Creating a check with valid config succeeds

#### Task 2.2: Check Execution Engine

**What:** Build the core engine that takes a list of check instances, executes each one against the target dataset via the appropriate connector, and collects results.

**Design:**
```python
# src/dqm/checks/engine.py
class CheckEngine:
    def __init__(self, connector: BaseConnector):
        self.connector = connector
        self.registry = CheckRegistry()

    async def execute_check(self, check: Check, dataset: Dataset) -> CheckResult:
        """Execute a single check and return the result."""
        handler = self.registry.get_handler(check.check_type)
        try:
            result = await handler.run(self.connector, check, dataset)
            return CheckResult(
                check_id=check.id,
                dataset_id=dataset.id,
                status=result.status,  # passed, warned, failed
                result=result.to_dict(),  # JSONB-serialisable
                executed_at=datetime.now(UTC),
            )
        except Exception as e:
            return CheckResult(
                check_id=check.id,
                dataset_id=dataset.id,
                status="errored",
                result={"error": str(e), "error_type": type(e).__name__},
                executed_at=datetime.now(UTC),
            )

    async def execute_batch(self, checks: list[Check], dataset: Dataset) -> list[CheckResult]:
        """Execute multiple checks, collecting results. Continues on individual failures."""
        results = []
        for check in checks:
            result = await self.execute_check(check, dataset)
            results.append(result)
        return results

# src/dqm/checks/rule_based.py
class NotNullCheckHandler:
    async def run(self, connector: BaseConnector, check: Check, dataset: Dataset) -> RawResult:
        column = check.config["column"]
        max_null_rate = check.config["max_null_rate"]

        rows = await connector.execute_query(f"""
            SELECT
                COUNT(*) AS total_rows,
                COUNT(*) FILTER (WHERE "{column}" IS NULL) AS null_count
            FROM "{dataset.schema_name}"."{dataset.table_name}"
        """)
        total = rows[0]["total_rows"]
        null_count = rows[0]["null_count"]
        null_rate = null_count / total if total > 0 else 0.0

        return RawResult(
            status="passed" if null_rate <= max_null_rate else "failed",
            observed_value=null_rate,
            expected_value=max_null_rate,
            detail={
                "column": column,
                "observed_null_rate": null_rate,
                "threshold": max_null_rate,
                "row_count": total,
                "null_count": null_count,
            },
        )
```

**Testing:**
- `test_not_null_check_passes`: Table with 0% nulls, threshold 5% -> status "passed"
- `test_not_null_check_fails`: Table with 10% nulls, threshold 5% -> status "failed"
- `test_not_null_check_empty_table`: Empty table -> status "passed" (no nulls possible)
- `test_unique_check_passes`: Column with all distinct values -> "passed"
- `test_unique_check_fails`: Column with duplicates exceeding threshold -> "failed"
- `test_value_range_check`: Values within [0, 100] -> "passed"; value of -1 -> "failed"
- `test_accepted_values_check`: Column with only accepted values -> "passed"
- `test_regex_pattern_check`: Email column matching pattern -> "passed"
- `test_freshness_check_passes`: Table updated 1 hour ago, SLA 2 hours -> "passed"
- `test_freshness_check_fails`: Table updated 5 hours ago, SLA 2 hours -> "failed"
- `test_volume_check_in_range`: Row count within expected range -> "passed"
- `test_custom_sql_check`: Custom SQL returning expected value -> "passed"
- `test_check_engine_error_handling`: Connector raises exception -> status "errored", error details captured
- `test_batch_execution_continues_on_failure`: One check errors, others still execute

#### Task 2.3: Schema Drift Detection

**What:** Implement schema snapshot capture and diff logic. On each sync or scan, capture the current schema, compare to the previous snapshot, and generate `schema_changes` records.

**Design:**
```python
# src/dqm/checks/schema_drift.py
import hashlib, json

class SchemaDriftDetector:
    def capture_snapshot(self, columns: list[ColumnInfo]) -> SchemaSnapshot:
        """Create a canonical snapshot from column metadata."""
        canonical = sorted(
            [{"name": c.name, "type": c.data_type, "nullable": c.is_nullable,
              "position": c.ordinal_position} for c in columns],
            key=lambda c: c["position"],
        )
        snapshot_hash = hashlib.sha256(
            json.dumps(canonical, sort_keys=True).encode()
        ).hexdigest()
        return SchemaSnapshot(
            snapshot_hash=snapshot_hash,
            columns_json=canonical,
        )

    def diff_snapshots(
        self, previous: SchemaSnapshot, current: SchemaSnapshot
    ) -> list[SchemaChange]:
        """Compare two snapshots and return a list of changes."""
        changes = []
        prev_cols = {c["name"]: c for c in previous.columns_json}
        curr_cols = {c["name"]: c for c in current.columns_json}

        for name in curr_cols:
            if name not in prev_cols:
                changes.append(SchemaChange(
                    change_type="column_added", column_name=name,
                    current_value=curr_cols[name]["type"],
                ))
        for name in prev_cols:
            if name not in curr_cols:
                changes.append(SchemaChange(
                    change_type="column_removed", column_name=name,
                    previous_value=prev_cols[name]["type"],
                ))
        for name in set(prev_cols) & set(curr_cols):
            if prev_cols[name]["type"] != curr_cols[name]["type"]:
                changes.append(SchemaChange(
                    change_type="type_changed", column_name=name,
                    previous_value=prev_cols[name]["type"],
                    current_value=curr_cols[name]["type"],
                ))
            if prev_cols[name]["nullable"] != curr_cols[name]["nullable"]:
                changes.append(SchemaChange(
                    change_type="nullable_changed", column_name=name,
                    previous_value=str(prev_cols[name]["nullable"]),
                    current_value=str(curr_cols[name]["nullable"]),
                ))
        return changes
```

**Testing:**
- `test_snapshot_hash_deterministic`: Same columns in same order produce identical hash
- `test_snapshot_hash_differs_on_change`: Adding a column changes the hash
- `test_diff_column_added`: New column detected
- `test_diff_column_removed`: Missing column detected
- `test_diff_type_changed`: `integer` -> `bigint` detected
- `test_diff_nullable_changed`: `NOT NULL` -> `NULL` detected
- `test_diff_no_changes`: Identical snapshots produce empty change list
- `test_schema_drift_check_integration`: End-to-end: capture snapshot, modify table, re-capture, verify alert with correct change details

#### Task 2.4: Check CRUD API

**What:** REST API endpoints for creating, listing, updating, enabling/disabling, and deleting checks on datasets. Include a batch-create endpoint for applying a template across multiple columns.

**Design:**
```python
# src/dqm/api/checks.py
router = APIRouter(prefix="/api/v1/checks", tags=["Checks"])

@router.post("/", response_model=CheckResponse, status_code=201)
async def create_check(body: CreateCheckRequest, user=Depends(get_current_user)):
    """Create a new check on a dataset."""
    # Validate config against template's parameter_schema
    template = await db.get(CheckTemplate, body.template_id)
    validate_json_schema(body.config, template.parameter_schema)
    # Persist check
    ...

@router.post("/batch", response_model=list[CheckResponse], status_code=201)
async def batch_create_checks(body: BatchCreateChecksRequest):
    """Apply a check template to multiple columns/datasets at once."""

@router.get("/datasets/{dataset_id}", response_model=list[CheckSummary])
async def list_checks_for_dataset(dataset_id: UUID):
    """List all checks configured for a dataset."""

@router.patch("/{check_id}", response_model=CheckResponse)
async def update_check(check_id: UUID, body: UpdateCheckRequest):
    """Update check configuration or severity."""

@router.patch("/{check_id}/toggle", response_model=CheckResponse)
async def toggle_check(check_id: UUID, body: ToggleCheckRequest):
    """Enable or disable a check."""
```

**Testing:**
- `test_create_check_success`: Valid check creation returns 201 with persisted check
- `test_create_check_invalid_config`: Config violating schema returns 422 with clear validation errors
- `test_batch_create_checks`: Apply not_null check to 10 columns, verify 10 checks created
- `test_list_checks_for_dataset`: Returns all checks for a dataset, not other datasets
- `test_update_check_threshold`: Change max_null_rate from 0.05 to 0.02, verify persisted
- `test_toggle_check_disable`: Disable a check, verify `is_enabled = false`
- `test_delete_check`: Delete a check, verify 404 on subsequent GET
- `test_audit_log_on_check_crud`: All create/update/delete operations produce audit log entries

#### Task 2.5: Scan Execution API & Check Result Storage (Migration 003)

**What:** Create the `scan_jobs`, `scan_runs`, and `check_results` tables. Implement the API for triggering a manual scan and viewing results. A scan discovers all enabled checks for the target datasets, executes them via the check engine, and persists results.

**Design:**
```python
# src/dqm/services/scan_service.py
class ScanService:
    async def execute_scan(self, scan_job: ScanJob, trigger_type: str,
                           triggered_by: UUID | None = None) -> ScanRun:
        run = ScanRun(
            scan_job_id=scan_job.id,
            org_id=scan_job.org_id,
            status="running",
            trigger_type=trigger_type,
            triggered_by=triggered_by,
            started_at=datetime.now(UTC),
        )
        await self.db.add(run)
        await self.db.flush()

        # Get all datasets in scope
        datasets = await self._get_datasets_for_job(scan_job)

        total, passed, warned, failed, errored = 0, 0, 0, 0, 0
        for dataset in datasets:
            connector = ConnectorFactory.create(
                dataset.data_source.source_type,
                dataset.data_source.config,
            )
            checks = await self._get_enabled_checks(dataset.id)
            engine = CheckEngine(connector)
            results = await engine.execute_batch(checks, dataset)

            for result in results:
                result.scan_run_id = run.id
                result.org_id = scan_job.org_id
                await self.db.add(result)
                total += 1
                match result.status:
                    case "passed": passed += 1
                    case "warned": warned += 1
                    case "failed": failed += 1
                    case "errored": errored += 1

        run.status = "completed"
        run.completed_at = datetime.now(UTC)
        run.summary = {
            "checks_total": total,
            "checks_passed": passed,
            "checks_warned": warned,
            "checks_failed": failed,
            "checks_errored": errored,
            "datasets_scanned": len(datasets),
            "duration_ms": int((run.completed_at - run.started_at).total_seconds() * 1000),
        }
        await self.db.commit()
        return run
```

**Testing:**
- `test_trigger_manual_scan`: POST `/api/v1/scans/{job_id}/run`, verify scan executes and results are persisted
- `test_scan_result_counts`: Scan with 5 checks (3 pass, 1 warn, 1 fail) -> summary reflects correct counts
- `test_scan_result_details`: Each check result contains the full JSONB detail (observed value, threshold, etc.)
- `test_scan_failure_handling`: If connector is unreachable, scan status = "failed" with error message
- `test_list_scan_runs`: GET `/api/v1/scans/{job_id}/runs` returns paginated run history
- `test_get_scan_run_results`: GET `/api/v1/scans/runs/{run_id}/results` returns all check results for that run
- `test_check_results_partitioned`: Verify `check_results` table is partitioned by `executed_at`

#### Phase 2 — Definition of Done

- [ ] 9 built-in check templates seeded in the database
- [ ] Check CRUD API: create (single and batch), list, update, toggle, delete
- [ ] Check config validated against template's JSON Schema on every create/update
- [ ] Check engine executes all rule-based check types: not_null, unique, value_range, accepted_values, regex_pattern, freshness, row_count_range, schema_drift, custom_sql
- [ ] Schema drift detection: snapshot capture, diff computation, change recording
- [ ] Scan execution: manual trigger via API, check result persistence, summary computation
- [ ] Check results stored with full JSONB detail per check type
- [ ] All check handlers have unit tests with pass/fail/edge-case coverage
- [ ] Test coverage: >80% on all new code

---

### Phase 3: Scheduling, Alerting & Web UI

**Goal:** Enable scheduled continuous monitoring, dispatch alerts when checks fail, and build the web UI for browsing datasets, check results, and alerts.

**Duration:** 5 weeks

**Prerequisites:** Phase 2 complete (checks execute and produce results)

#### Task 3.1: Scan Scheduling (Celery Beat)

**What:** Configure Celery with Redis as broker. Each `scan_job` with a `schedule_cron` expression becomes a periodic task. The scheduler creates scan runs at the configured interval.

**Design:**
```python
# src/dqm/scheduler/scheduler.py
from celery import Celery
from celery.schedules import crontab

celery_app = Celery("dqm", broker=settings.redis_url)

@celery_app.task(bind=True, max_retries=2, default_retry_delay=60)
def execute_scan_task(self, scan_job_id: str):
    """Celery task that executes a scan job."""
    import asyncio
    asyncio.run(_run_scan(scan_job_id))

async def _run_scan(scan_job_id: str):
    async with get_db_session() as db:
        scan_job = await db.get(ScanJob, UUID(scan_job_id))
        service = ScanService(db)
        await service.execute_scan(scan_job, trigger_type="scheduled")

def sync_schedules_from_db():
    """Read all enabled scan_jobs with cron expressions and register them with Celery Beat."""
    # Called on startup and when scan_jobs are modified
    ...
```

**Testing:**
- `test_cron_parsing`: Verify cron expressions are correctly parsed (every hour, daily at 6am, every 15 minutes)
- `test_scan_task_execution`: Celery task executes a scan and persists results
- `test_scan_task_retry_on_failure`: Transient failure retries up to 2 times
- `test_schedule_sync_adds_new_job`: Adding a scan_job with cron triggers schedule registration
- `test_schedule_sync_removes_disabled_job`: Disabling a scan_job removes it from the schedule
- `test_concurrent_scan_prevention`: If a scan is already running for a job, skip the new invocation

#### Task 3.2: Metric Collection Engine (Migration 004)

**What:** Create the `metrics` and `metric_baselines` tables. During each scan, compute and persist per-column metrics (null_rate, distinct_count, mean, stddev, min, max) and per-table metrics (row_count, freshness_seconds). These time-series metrics feed the dashboard charts and Phase 4's anomaly detection.

**Design:**
```python
# src/dqm/profiler/profiler.py
class DataProfiler:
    async def profile_table(self, connector: BaseConnector,
                            dataset: Dataset) -> TableProfile:
        """Compute table-level and column-level metrics."""
        row_count = await connector.get_row_count(
            dataset.schema_name, dataset.table_name
        )

        column_metrics = []
        for col in dataset.columns:
            stats = await self._profile_column(connector, dataset, col)
            column_metrics.append(ColumnMetrics(
                column_name=col.column_name,
                metrics=stats,
            ))

        return TableProfile(
            row_count=row_count,
            column_metrics=column_metrics,
        )

    async def _profile_column(self, connector, dataset, column) -> dict:
        """Compute statistics for a single column."""
        query = f"""
            SELECT
                COUNT(*) AS total_rows,
                COUNT("{column.column_name}") AS non_null_count,
                COUNT(*) - COUNT("{column.column_name}") AS null_count,
                COUNT(DISTINCT "{column.column_name}") AS distinct_count
            FROM "{dataset.schema_name}"."{dataset.table_name}"
        """
        rows = await connector.execute_query(query)
        stats = rows[0]
        stats["null_rate"] = stats["null_count"] / stats["total_rows"] if stats["total_rows"] > 0 else 0

        # For numeric columns, compute mean/stddev/min/max
        if column.data_type in ("integer", "bigint", "numeric", "float", "double precision"):
            numeric_query = f"""
                SELECT
                    AVG("{column.column_name}") AS mean,
                    STDDEV("{column.column_name}") AS stddev,
                    MIN("{column.column_name}") AS min_value,
                    MAX("{column.column_name}") AS max_value
                FROM "{dataset.schema_name}"."{dataset.table_name}"
            """
            numeric_rows = await connector.execute_query(numeric_query)
            stats.update(numeric_rows[0])

        return stats
```

**Testing:**
- `test_profile_table_row_count`: Verify row count matches actual table rows
- `test_profile_column_null_rate`: Insert known data, verify null_rate calculation
- `test_profile_column_distinct_count`: Verify distinct count for column with known duplicates
- `test_profile_numeric_column_stats`: Mean, stddev, min, max for numeric column
- `test_profile_string_column_no_numeric_stats`: String columns do not compute mean/stddev
- `test_metrics_persisted`: After scan, verify `metrics` table contains rows for each column
- `test_metrics_time_series`: Run two scans, verify two rows per metric with different `measured_at`

#### Task 3.3: Alert Dispatch System (Migration 005)

**What:** Create the `alert_channels`, `alert_rules`, and `alerts` tables. When a scan completes with failures, evaluate alert rules and dispatch notifications via configured channels (Slack webhook, email, generic webhook).

**Design:**
```python
# src/dqm/alerting/dispatcher.py
class AlertDispatcher:
    def __init__(self, db: AsyncSession):
        self.db = db
        self.channels = {
            "slack": SlackNotifier(),
            "email": EmailNotifier(),
            "webhook": WebhookNotifier(),
        }

    async def process_scan_results(self, scan_run: ScanRun,
                                    results: list[CheckResult]):
        """Evaluate alert rules against scan results and dispatch notifications."""
        failed_results = [r for r in results if r.status in ("failed", "errored")]
        if not failed_results:
            return

        rules = await self._get_matching_rules(scan_run.org_id, failed_results)

        for result in failed_results:
            for rule in rules:
                if self._rule_matches(rule, result):
                    alert = await self._create_alert(rule, result)
                    channel = await self.db.get(AlertChannel, rule.channel_id)
                    notifier = self.channels[channel.channel_type]
                    await notifier.send(channel.config, alert)
                    alert.notified_at = datetime.now(UTC)
                    alert.notifications = [
                        {"channel_id": str(channel.id),
                         "channel_type": channel.channel_type,
                         "sent_at": alert.notified_at.isoformat()}
                    ]

# src/dqm/alerting/slack.py
class SlackNotifier:
    async def send(self, config: dict, alert: Alert):
        """Send an alert to Slack via incoming webhook."""
        payload = {
            "text": f":red_circle: *{alert.title}*",
            "blocks": [
                {"type": "header", "text": {"type": "plain_text", "text": alert.title}},
                {"type": "section", "text": {"type": "mrkdwn",
                    "text": f"*Severity:* {alert.severity}\n*Dataset:* {alert.detail.get('dataset', 'unknown')}\n*Observed:* {alert.detail.get('observed')}\n*Threshold:* {alert.detail.get('threshold')}"}},
            ],
        }
        async with httpx.AsyncClient() as client:
            await client.post(config["webhook_url"], json=payload)
```

**Testing:**
- `test_alert_created_on_check_failure`: Failed check result -> alert row created
- `test_alert_not_created_on_pass`: Passing check result -> no alert
- `test_alert_severity_from_check`: Alert inherits severity from check definition
- `test_slack_notification_sent`: Mock Slack webhook, verify POST with correct payload
- `test_email_notification_sent`: Mock SMTP, verify email with subject and body
- `test_webhook_notification_sent`: Mock webhook endpoint, verify POST with alert JSON
- `test_alert_rule_filtering`: Rule scoped to "critical" severity only fires on critical failures
- `test_alert_deduplication`: Same check failing twice in succession does not create duplicate alert if previous alert is still open
- `test_alert_acknowledge_api`: PATCH `/api/v1/alerts/{id}/acknowledge` sets `acknowledged_at` and `acknowledged_by`
- `test_alert_resolve_api`: PATCH `/api/v1/alerts/{id}/resolve` sets `resolved_at`

#### Task 3.4: Frontend — Dataset Browser & Check Results Dashboard

**What:** Build the React frontend with pages for: (a) data source list, (b) dataset inventory with column details, (c) check result timeline, (d) active alerts feed.

**Design:**
```typescript
// frontend/src/pages/DatasetsPage.tsx
export function DatasetsPage() {
  const { data: datasets, isLoading } = useQuery({
    queryKey: ["datasets"],
    queryFn: () => api.datasets.list(),
  });

  return (
    <PageLayout title="Datasets">
      <DataTable
        columns={[
          { header: "Source", accessorKey: "source_name" },
          { header: "Schema", accessorKey: "schema_name" },
          { header: "Table", accessorKey: "table_name" },
          { header: "Row Count", accessorKey: "row_count",
            cell: (v) => v.getValue()?.toLocaleString() },
          { header: "Checks", accessorKey: "check_count" },
          { header: "Last Scan", accessorKey: "last_scanned_at",
            cell: (v) => <TimeAgo date={v.getValue()} /> },
          { header: "Status", accessorKey: "latest_status",
            cell: (v) => <StatusBadge status={v.getValue()} /> },
        ]}
        data={datasets}
      />
    </PageLayout>
  );
}

// frontend/src/pages/DatasetDetailPage.tsx
// Tabs: Columns | Checks | Scan History | Metrics
// Columns tab shows column name, type, nullable, null_rate, distinct_count
// Checks tab shows check name, type, dimension, last status, last run time
// Scan History tab shows timeline of scan runs with pass/fail counts
// Metrics tab shows time-series charts (Recharts) for row_count, null_rate per column
```

**Testing:**
- `test_datasets_page_renders`: Page loads and displays dataset table with correct columns
- `test_datasets_page_pagination`: Navigating pages loads correct dataset slices
- `test_dataset_detail_columns_tab`: Column tab shows correct column metadata
- `test_dataset_detail_checks_tab`: Check tab shows all configured checks with latest status
- `test_dataset_detail_metrics_chart`: Metric chart renders with time-series data points
- `test_alerts_page_renders`: Alerts page shows active alerts with severity indicators
- `test_alert_acknowledge_action`: Clicking acknowledge button calls API and updates UI
- `test_responsive_layout`: Pages render correctly on 1024px and 1440px viewport widths

#### Task 3.5: Frontend — Alert Management & Scan Triggers

**What:** Alert list page with filtering (by severity, status, dataset), alert detail with context, and a manual "Run Scan" button on scan job pages.

**Design:**
```typescript
// frontend/src/pages/AlertsPage.tsx
export function AlertsPage() {
  const [filters, setFilters] = useState<AlertFilters>({
    status: "open",
    severity: undefined,
    datasetId: undefined,
  });

  const { data: alerts } = useQuery({
    queryKey: ["alerts", filters],
    queryFn: () => api.alerts.list(filters),
  });

  return (
    <PageLayout title="Alerts">
      <AlertFiltersBar filters={filters} onChange={setFilters} />
      <AlertList alerts={alerts} onAcknowledge={handleAcknowledge} onResolve={handleResolve} />
    </PageLayout>
  );
}
```

**Testing:**
- `test_alert_filter_by_severity`: Filtering by "critical" shows only critical alerts
- `test_alert_filter_by_status`: Filtering by "open" hides resolved alerts
- `test_manual_scan_trigger`: Clicking "Run Scan" sends POST and shows scan progress
- `test_scan_results_update_after_run`: After scan completes, check results refresh automatically
- E2E (Playwright): `test_full_flow_source_to_alert`: Create source -> sync -> create check -> run scan -> see alert

#### Phase 3 — Definition of Done

- [ ] Scan jobs execute on schedule via Celery Beat
- [ ] Metric collection runs during each scan, persisting column-level and table-level metrics
- [ ] Alerts fire on check failures and are dispatched to configured Slack/email/webhook channels
- [ ] Alert lifecycle: create -> acknowledge -> resolve, with API and UI support
- [ ] Frontend pages: data sources, datasets (with columns, checks, metrics tabs), alerts, scan history
- [ ] Time-series metric charts display per-column trends over time
- [ ] Manual scan trigger from the UI
- [ ] Concurrent scan prevention: duplicate scan invocations are skipped
- [ ] Test coverage: >80% on all new code (backend and frontend)
- [ ] Playwright E2E test: full flow from source registration to alert display

---

### Phase 4: ML Anomaly Detection

**Goal:** Add ML-based baseline learning and anomaly detection for volume and distribution metrics, eliminating the need for manual threshold authorship on supported metric types.

**Duration:** 4 weeks

**Prerequisites:** Phase 3 complete (metrics are being collected over time; at least 2 weeks of historical metric data needed for baselines)

#### Task 4.1: Baseline Computation Engine

**What:** Build the baseline computation module that calculates rolling statistical summaries (mean, stddev, median, MAD) over configurable time windows from historical metrics. Baselines are recomputed after each scan run.

**Design:**
```python
# src/dqm/anomaly/baselines.py
from scipy import stats as scipy_stats

class BaselineComputer:
    def __init__(self, db: AsyncSession, window_days: int = 30):
        self.db = db
        self.window_days = window_days

    async def compute_baseline(
        self, dataset_id: UUID, column_name: str | None, metric_type: str
    ) -> MetricBaseline:
        """Compute baseline statistics from recent metric history."""
        cutoff = datetime.now(UTC) - timedelta(days=self.window_days)

        query = select(Metric.value).where(
            Metric.dataset_id == dataset_id,
            Metric.column_name == column_name,
            Metric.metric_type == metric_type,
            Metric.measured_at >= cutoff,
        ).order_by(Metric.measured_at)

        result = await self.db.execute(query)
        values = [row[0] for row in result.fetchall()]

        if len(values) < 7:  # Minimum sample size for meaningful baseline
            return None

        return MetricBaseline(
            dataset_id=dataset_id,
            column_name=column_name,
            metric_type=metric_type,
            baseline={
                "mean": float(np.mean(values)),
                "stddev": float(np.std(values, ddof=1)),
                "median": float(np.median(values)),
                "mad": float(scipy_stats.median_abs_deviation(values)),
                "min": float(np.min(values)),
                "max": float(np.max(values)),
                "sample_count": len(values),
                "window_days": self.window_days,
            },
        )
```

**Testing:**
- `test_baseline_mean_stddev`: Known dataset produces expected mean and stddev
- `test_baseline_median_mad`: Known dataset produces expected median and MAD
- `test_baseline_minimum_samples`: Fewer than 7 data points returns None
- `test_baseline_window_filtering`: Only data within the window period is included
- `test_baseline_recompute_after_scan`: After new scan, baseline updates with new values
- `test_baseline_persistence`: Computed baseline is persisted in `metric_baselines` table

#### Task 4.2: Anomaly Detection Algorithms

**What:** Implement three anomaly scoring methods: z-score (for normally distributed metrics), MAD-based (for skewed distributions), and Isolation Forest (for multivariate anomalies). The detector auto-selects the appropriate method based on distribution characteristics.

**Design:**
```python
# src/dqm/anomaly/algorithms.py
class ZScoreDetector:
    def score(self, observed: float, baseline: dict, sensitivity: float = 3.0) -> AnomalyScore:
        if baseline["stddev"] == 0:
            return AnomalyScore(z_score=0.0, is_anomaly=False)
        z = abs(observed - baseline["mean"]) / baseline["stddev"]
        return AnomalyScore(
            z_score=z,
            is_anomaly=z > sensitivity,
            direction="above" if observed > baseline["mean"] else "below",
        )

class MADDetector:
    def score(self, observed: float, baseline: dict, sensitivity: float = 3.5) -> AnomalyScore:
        if baseline["mad"] == 0:
            return AnomalyScore(modified_z=0.0, is_anomaly=False)
        modified_z = 0.6745 * abs(observed - baseline["median"]) / baseline["mad"]
        return AnomalyScore(
            modified_z=modified_z,
            is_anomaly=modified_z > sensitivity,
            direction="above" if observed > baseline["median"] else "below",
        )

class AnomalyDetector:
    """Main detector that selects the appropriate algorithm."""
    def __init__(self):
        self.zscore = ZScoreDetector()
        self.mad = MADDetector()

    def detect(self, observed: float, baseline: dict, sensitivity: float = 3.0) -> AnomalyResult:
        # Use MAD for skewed distributions, z-score for normal
        skewness = self._estimate_skewness(baseline)
        if abs(skewness) > 1.0:
            score = self.mad.score(observed, baseline, sensitivity)
            algorithm = "mad"
        else:
            score = self.zscore.score(observed, baseline, sensitivity)
            algorithm = "zscore"

        return AnomalyResult(
            is_anomaly=score.is_anomaly,
            score=score,
            algorithm=algorithm,
            observed=observed,
            baseline_summary={
                "mean": baseline["mean"],
                "stddev": baseline["stddev"],
                "median": baseline["median"],
                "window_days": baseline["window_days"],
                "sample_count": baseline["sample_count"],
            },
        )
```

**Testing:**
- `test_zscore_normal_value`: Value within 1 stddev -> not anomalous
- `test_zscore_anomaly`: Value 4 stddev above mean -> anomalous
- `test_zscore_zero_stddev`: All identical values, then a different value -> handled without division by zero
- `test_mad_robust_to_outliers`: MAD correctly handles a dataset with existing outliers in the baseline
- `test_mad_anomaly`: Value with modified z > 3.5 -> anomalous
- `test_algorithm_selection_normal`: Symmetric distribution -> z-score selected
- `test_algorithm_selection_skewed`: Skewed distribution -> MAD selected
- `test_sensitivity_parameter`: Higher sensitivity threshold reduces false positives

#### Task 4.3: ML Check Type & Integration

**What:** Add the `ml_anomaly` check type to the check engine. ML checks do not have static thresholds; instead, they query the baseline for the target metric and apply the anomaly detector. Create the check template and integrate with the scan execution flow.

**Design:**
```python
# src/dqm/checks/ml_anomaly.py
class MLAnomalyCheckHandler:
    async def run(self, connector: BaseConnector, check: Check,
                  dataset: Dataset, db: AsyncSession) -> RawResult:
        column_name = check.config.get("column")
        metric_type = check.config.get("metric", "null_rate")
        sensitivity = check.config.get("sensitivity", 3.0)

        # Get current metric value
        profiler = DataProfiler()
        current_value = await profiler.get_metric(
            connector, dataset, column_name, metric_type
        )

        # Get baseline
        baseline_row = await db.execute(
            select(MetricBaseline).where(
                MetricBaseline.dataset_id == dataset.id,
                MetricBaseline.column_name == column_name,
                MetricBaseline.metric_type == metric_type,
            )
        )
        baseline = baseline_row.scalar_one_or_none()

        if not baseline:
            return RawResult(
                status="warned",
                detail={"message": "Insufficient baseline data; minimum 7 data points required"},
            )

        detector = AnomalyDetector()
        result = detector.detect(current_value, baseline.baseline, sensitivity)

        return RawResult(
            status="failed" if result.is_anomaly else "passed",
            observed_value=current_value,
            detail={
                "column": column_name,
                "metric": metric_type,
                "observed": current_value,
                "baseline_mean": result.baseline_summary["mean"],
                "baseline_stddev": result.baseline_summary["stddev"],
                "z_score": getattr(result.score, "z_score", None),
                "modified_z": getattr(result.score, "modified_z", None),
                "is_anomaly": result.is_anomaly,
                "algorithm": result.algorithm,
                "sample_count": result.baseline_summary["sample_count"],
                "direction": result.score.direction,
            },
        )
```

**Testing:**
- `test_ml_check_passes_normal_value`: Metric within baseline range -> "passed"
- `test_ml_check_detects_anomaly`: Metric 5 stddev above mean -> "failed" with anomaly details
- `test_ml_check_insufficient_baseline`: Fewer than 7 baseline points -> "warned" with message
- `test_ml_check_sensitivity_configurable`: Custom sensitivity threshold changes detection boundary
- `test_ml_check_result_includes_baseline`: Result JSONB includes baseline statistics for debugging
- `test_ml_check_e2e_with_real_metrics`: Insert 30 days of metric history, verify anomaly detection on day 31

#### Task 4.4: Anomaly Dashboard Enhancements

**What:** Update the frontend metrics tab to overlay anomaly baselines (mean +/- stddev band) on time-series charts and display anomaly markers. Add an "Anomalies" section to the alerts page.

**Design:**
```typescript
// frontend/src/components/metrics/MetricChart.tsx
interface MetricChartProps {
  metrics: MetricPoint[];
  baseline?: BaselineData;
  anomalies?: AnomalyMarker[];
}

export function MetricChart({ metrics, baseline, anomalies }: MetricChartProps) {
  return (
    <ResponsiveContainer width="100%" height={300}>
      <ComposedChart data={metrics}>
        <Area  // Baseline band (mean ± 2*stddev)
          dataKey="baselineUpper" stroke="none" fill="#e0f2fe" />
        <Area
          dataKey="baselineLower" stroke="none" fill="#e0f2fe" />
        <Line  // Baseline mean
          dataKey="baselineMean" stroke="#94a3b8" strokeDasharray="4 4" />
        <Line  // Observed values
          dataKey="value" stroke="#2563eb" strokeWidth={2} />
        {anomalies?.map((a) => (
          <ReferenceDot  // Anomaly markers
            key={a.id} x={a.measured_at} y={a.value}
            r={6} fill="#ef4444" stroke="#dc2626" />
        ))}
        <XAxis dataKey="measured_at" />
        <YAxis />
        <Tooltip />
      </ComposedChart>
    </ResponsiveContainer>
  );
}
```

**Testing:**
- `test_metric_chart_with_baseline`: Chart renders baseline band when baseline data is available
- `test_metric_chart_anomaly_markers`: Anomaly points are rendered as red dots
- `test_metric_chart_no_baseline`: Chart renders normally without baseline band when baseline is missing
- `test_anomaly_tooltip`: Hovering an anomaly marker shows z-score and baseline summary

#### Phase 4 — Definition of Done

- [ ] Baseline computation runs automatically after each scan, updating `metric_baselines`
- [ ] Three anomaly detection algorithms implemented: z-score, MAD, and Isolation Forest
- [ ] `ml_anomaly` check type registered and executable by the check engine
- [ ] ML checks produce "failed" alerts when anomalies exceed the sensitivity threshold
- [ ] ML check results include full baseline context (mean, stddev, z-score, algorithm, direction)
- [ ] Frontend metric charts display baseline bands and anomaly markers
- [ ] Minimum 7 data points required before baseline is considered valid
- [ ] Test coverage: >80% on all anomaly detection code, including edge cases (zero stddev, skewed data)

---

### Phase 5: AI-Powered Features

**Goal:** Use LLM capabilities to auto-generate quality check suggestions from schema metadata, produce natural-language alert explanations, and discover cross-table relationships.

**Duration:** 5 weeks

**Prerequisites:** Phase 4 complete (baselines and anomaly detection provide context for AI explanations)

#### Task 5.1: AI Check Suggestion Engine

**What:** Given a dataset's schema, column statistics, and semantic type hints, generate candidate quality checks using the Claude API. Suggestions are stored as checks with `is_ai_generated = true` and `ai_metadata` containing confidence scores and reasoning. Users accept or reject each suggestion.

**Design:**
```python
# src/dqm/ai/check_generator.py
import anthropic

class AICheckGenerator:
    def __init__(self):
        self.client = anthropic.Anthropic()

    async def generate_suggestions(
        self, dataset: Dataset, columns: list[DatasetColumn],
        existing_checks: list[Check]
    ) -> list[CheckSuggestion]:
        prompt = self._build_prompt(dataset, columns, existing_checks)

        response = self.client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=4096,
            messages=[{"role": "user", "content": prompt}],
            system="You are a data quality expert. Given a table schema and column statistics, suggest quality checks. Return a JSON array of check suggestions.",
        )

        suggestions = json.loads(response.content[0].text)
        return [CheckSuggestion(**s) for s in suggestions]

    def _build_prompt(self, dataset, columns, existing_checks) -> str:
        col_info = "\n".join([
            f"  - {c.column_name} ({c.data_type}, "
            f"nullable={c.is_nullable}, "
            f"semantic_type={c.semantic_type or 'unknown'}, "
            f"null_rate={c.stats.get('null_rate', 'N/A')}, "
            f"distinct_count={c.stats.get('distinct_count', 'N/A')})"
            for c in columns
        ])
        existing = "\n".join([f"  - {c.name} ({c.check_type})" for c in existing_checks])

        return f"""
Table: {dataset.schema_name}.{dataset.table_name}
Row count: {dataset.row_count}

Columns:
{col_info}

Existing checks (do not suggest duplicates):
{existing or "  (none)"}

For each suggestion, provide:
- template_name: one of [not_null, unique, value_range, accepted_values, regex_pattern, freshness, row_count_range]
- column: the target column name (or null for table-level)
- config: the check parameters matching the template schema
- confidence: 0.0-1.0 (how confident you are this check is appropriate)
- reasoning: a brief explanation of why this check is useful

Return as a JSON array.
"""
```

**Testing:**
- `test_generate_suggestions_for_orders_table`: Orders table with id, email, amount, created_at -> suggests not_null on id, email check, value_range on amount
- `test_no_duplicate_suggestions`: Existing not_null check on email -> suggestion for email not_null is excluded
- `test_suggestion_confidence_scores`: All suggestions have confidence between 0.0 and 1.0
- `test_suggestion_api_accept`: POST `/api/v1/ai/suggestions/{id}/accept` converts suggestion to active check
- `test_suggestion_api_reject`: POST `/api/v1/ai/suggestions/{id}/reject` marks suggestion as rejected
- `test_suggestion_with_semantic_types`: Column with `semantic_type = "email"` gets regex pattern suggestion
- `test_llm_error_handling`: API failure returns graceful error, does not crash the scan

#### Task 5.2: Natural-Language Alert Explanations

**What:** When an alert fires, enrich it with an LLM-generated natural-language explanation that describes what happened, why it might have happened, and what the downstream impact could be. The explanation uses the check result, baseline context, recent schema changes, and lineage data.

**Design:**
```python
# src/dqm/ai/explainer.py
class AlertExplainer:
    async def explain(self, alert: Alert, check_result: CheckResult,
                      dataset: Dataset, recent_changes: list[SchemaChange],
                      upstream_failures: list[CheckResult]) -> str:
        context = self._build_context(alert, check_result, dataset,
                                      recent_changes, upstream_failures)

        response = self.client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=1024,
            messages=[{"role": "user", "content": context}],
            system="""You are a data quality analyst. Given an alert with its context, write a clear 2-4 sentence explanation of:
1. What happened (the anomaly or failure)
2. A probable cause (based on schema changes, upstream failures, or historical patterns)
3. Potential downstream impact (which tables or reports might be affected)

Be specific, use the provided data, and avoid vague statements. Write for a data engineer audience.""",
        )

        return response.content[0].text
```

**Testing:**
- `test_explanation_includes_metric_details`: Explanation mentions the observed value and threshold
- `test_explanation_references_schema_change`: When a relevant schema change exists, explanation mentions it
- `test_explanation_references_upstream_failure`: When upstream check failed, explanation links the two
- `test_explanation_stored_on_alert`: After generation, `alert.ai_explanation` field is populated
- `test_explanation_displayed_in_ui`: Frontend alert detail shows the AI explanation
- `test_explanation_graceful_fallback`: If LLM is unavailable, alert is created without explanation (no blocking)

#### Task 5.3: Cross-Table Relationship Discovery

**What:** Analyse column names, data types, and join patterns across all datasets in a source to discover implicit referential integrity relationships (e.g., `orders.customer_id` references `customers.id`). Discovered relationships are presented as suggested checks.

**Design:**
```python
# src/dqm/ai/relationship.py
class RelationshipDiscoverer:
    async def discover(self, org_id: UUID, source_id: UUID) -> list[RelationshipSuggestion]:
        """Discover cross-table relationships from column name and type analysis."""
        datasets = await self._get_all_datasets(source_id)
        columns_by_name = self._index_columns(datasets)

        # Heuristic pass: find FK-like patterns
        candidates = []
        for dataset in datasets:
            for col in dataset.columns:
                if col.column_name.endswith("_id"):
                    referenced_table = col.column_name[:-3]  # e.g., "customer_id" -> "customer"
                    matches = self._find_matching_tables(referenced_table, datasets)
                    for match in matches:
                        candidates.append(RelationshipCandidate(
                            source_table=dataset,
                            source_column=col,
                            target_table=match,
                            target_column="id",
                            confidence=0.7,  # heuristic match
                        ))

        # LLM refinement pass: validate candidates and discover non-obvious relationships
        refined = await self._llm_refine(candidates, datasets)
        return refined
```

**Testing:**
- `test_discover_fk_pattern`: `orders.customer_id` + `customers.id` -> relationship discovered
- `test_discover_plural_match`: `order_items.order_id` + `orders.id` -> relationship discovered
- `test_no_false_positive_on_common_names`: `orders.created_at` does not create relationship to a `created` table
- `test_llm_refine_adds_confidence`: LLM refinement adjusts confidence scores
- `test_suggested_check_creation`: Discovered relationship generates a referential integrity check suggestion
- `test_relationship_api`: GET `/api/v1/ai/relationships/{source_id}` returns discovered relationships

#### Phase 5 — Definition of Done

- [ ] AI check suggestion engine generates relevant checks from schema metadata and column statistics
- [ ] Suggestions include confidence scores and natural-language reasoning
- [ ] Accept/reject workflow: suggestions become active checks on accept
- [ ] Alert explanations generated by LLM with metric details, probable causes, and impact
- [ ] Explanations reference recent schema changes and upstream failures when available
- [ ] Cross-table relationship discovery identifies FK-like patterns and generates check suggestions
- [ ] LLM calls are non-blocking: failures degrade gracefully without blocking core functionality
- [ ] Rate limiting and cost tracking for LLM API calls
- [ ] Test coverage: >80% on AI modules with mocked LLM responses

---

### Phase 6: Integrations (dbt, Airflow, OpenLineage)

**Goal:** Enable the platform to consume metadata and quality signals from the data engineering ecosystem: dbt manifests, Airflow DAGs, and OpenLineage events.

**Duration:** 4 weeks

**Prerequisites:** Phase 3 complete (check engine and alerting operational)

#### Task 6.1: dbt Integration

**What:** Ingest dbt `manifest.json` and `run_results.json` to discover models, tests, and their results. Map dbt models to datasets, dbt tests to checks, and dbt test results to check results. Support importing dbt test definitions as DQM checks.

**Design:**
```python
# src/dqm/integrations/dbt.py
class DbtIntegration:
    async def ingest_manifest(self, org_id: UUID, source_id: UUID,
                               manifest: dict) -> DbtIngestionResult:
        """Parse a dbt manifest.json and sync models/tests to DQM datasets/checks."""
        models = self._extract_models(manifest)
        tests = self._extract_tests(manifest)

        datasets_synced = 0
        for model in models:
            dataset = await self._upsert_dataset(org_id, source_id, model)
            datasets_synced += 1

        checks_synced = 0
        for test in tests:
            check = await self._upsert_check(org_id, test)
            checks_synced += 1

        return DbtIngestionResult(
            datasets_synced=datasets_synced,
            checks_synced=checks_synced,
        )

    async def ingest_run_results(self, org_id: UUID,
                                  run_results: dict) -> int:
        """Parse dbt run_results.json and create check_results for each test."""
        results_created = 0
        for result in run_results.get("results", []):
            if result["unique_id"].startswith("test."):
                await self._create_check_result(org_id, result)
                results_created += 1
        return results_created
```

**Testing:**
- `test_ingest_manifest_creates_datasets`: Manifest with 10 models -> 10 datasets created
- `test_ingest_manifest_creates_checks`: Manifest with 4 tests per model -> checks created
- `test_ingest_manifest_idempotent`: Second ingestion does not create duplicates
- `test_ingest_run_results`: Run results with pass/fail tests -> check_results created with correct status
- `test_dbt_test_to_check_mapping`: `not_null` dbt test maps to `not_null` check template
- `test_dbt_api_endpoint`: POST `/api/v1/integrations/dbt/manifest` accepts file upload

#### Task 6.2: OpenLineage Event Consumer

**What:** Implement an HTTP endpoint that receives OpenLineage events (RunEvent with DataQualityMetrics facets) and maps them to DQM entities. This enables any OpenLineage-compatible tool (Airflow, Spark, Flink) to send lineage and quality data to DQM.

**Design:**
```python
# src/dqm/integrations/openlineage.py
@router.post("/api/v1/integrations/openlineage/events")
async def receive_openlineage_event(event: dict, user=Depends(get_current_user)):
    """Receive an OpenLineage RunEvent and process it."""
    event_type = event.get("eventType")  # START, RUNNING, COMPLETE, FAIL, ABORT
    job = event.get("job", {})
    run = event.get("run", {})
    inputs = event.get("inputs", [])
    outputs = event.get("outputs", [])

    # Update lineage edges from inputs -> outputs
    for input_ds in inputs:
        for output_ds in outputs:
            await upsert_lineage_edge(
                source=input_ds, target=output_ds,
                job_name=job.get("name"),
                discovered_via="openlineage",
            )

    # Extract data quality facets if present
    for output_ds in outputs:
        facets = output_ds.get("facets", {})
        dq_facet = facets.get("dataQualityMetrics")
        if dq_facet:
            await ingest_quality_metrics(output_ds, dq_facet)
```

**Testing:**
- `test_openlineage_run_event_creates_lineage`: RunEvent with inputs and outputs creates lineage edges
- `test_openlineage_quality_facet_creates_metrics`: DataQualityMetrics facet creates metric rows
- `test_openlineage_idempotent`: Same event received twice does not create duplicate edges
- `test_openlineage_event_validation`: Malformed event returns 422 with error details

#### Task 6.3: Airflow Operator

**What:** Publish a Python package (`dqm-airflow`) containing an Airflow operator that triggers a DQM scan as an Airflow task. The operator blocks until the scan completes and fails the task if critical checks fail.

**Design:**
```python
# integrations/airflow/dqm_airflow/operators.py
from airflow.models import BaseOperator
from airflow.utils.decorators import apply_defaults

class DQMScanOperator(BaseOperator):
    @apply_defaults
    def __init__(self, dqm_url: str, api_key: str, scan_job_id: str,
                 fail_on_critical: bool = True, **kwargs):
        super().__init__(**kwargs)
        self.dqm_url = dqm_url
        self.api_key = api_key
        self.scan_job_id = scan_job_id
        self.fail_on_critical = fail_on_critical

    def execute(self, context):
        # Trigger scan
        response = requests.post(
            f"{self.dqm_url}/api/v1/scans/{self.scan_job_id}/run",
            headers={"Authorization": f"Bearer {self.api_key}"},
        )
        run_id = response.json()["id"]

        # Poll until complete
        while True:
            status = requests.get(
                f"{self.dqm_url}/api/v1/scans/runs/{run_id}",
                headers={"Authorization": f"Bearer {self.api_key}"},
            ).json()
            if status["status"] in ("completed", "failed"):
                break
            time.sleep(10)

        if self.fail_on_critical and status["summary"]["checks_failed"] > 0:
            raise AirflowException(
                f"DQM scan failed: {status['summary']['checks_failed']} checks failed"
            )
```

**Testing:**
- `test_operator_triggers_scan`: Operator calls POST and receives run_id
- `test_operator_waits_for_completion`: Operator polls until status is "completed"
- `test_operator_fails_on_critical`: Critical check failure -> AirflowException raised
- `test_operator_succeeds_on_warnings`: Warnings without failures -> task succeeds

#### Phase 6 — Definition of Done

- [ ] dbt manifest ingestion: models -> datasets, tests -> checks, run_results -> check_results
- [ ] OpenLineage event endpoint: lineage edge creation from RunEvents
- [ ] Airflow operator published as installable Python package
- [ ] Lineage edges persisted with `discovered_via` field for provenance tracking
- [ ] Integration tests using real dbt manifest fixtures
- [ ] Test coverage: >80% on all integration code

---

### Phase 7: Data Contracts & Governance

**Goal:** Implement data contracts that formalise schema and quality SLAs between data producers and consumers, with automated violation detection and compliance reporting.

**Duration:** 4 weeks

**Prerequisites:** Phase 5 complete (AI features provide smart contract suggestions), Phase 3 (alerting for violation notifications)

#### Task 7.1: Data Contract Schema & CRUD (Migration 006)

**What:** Create the `data_contracts` table (JSONB spec following ODCS structure). Implement API endpoints for creating, versioning, publishing, and subscribing to contracts.

**Design:**
```python
# Contract spec structure (ODCS-aligned)
CONTRACT_SPEC_SCHEMA = {
    "$schema": "https://json-schema.org/draft/2020-12/schema",
    "type": "object",
    "properties": {
        "producer": {
            "type": "object",
            "properties": {
                "team": {"type": "string"},
                "contact": {"type": "string", "format": "email"},
            },
            "required": ["team"],
        },
        "schema": {
            "type": "object",
            "properties": {
                "columns": {
                    "type": "array",
                    "items": {
                        "type": "object",
                        "properties": {
                            "name": {"type": "string"},
                            "type": {"type": "string"},
                            "nullable": {"type": "boolean"},
                            "description": {"type": "string"},
                        },
                        "required": ["name", "type"],
                    },
                },
            },
        },
        "quality": {
            "type": "object",
            "properties": {
                "freshness_sla_minutes": {"type": "integer", "minimum": 1},
                "completeness_sla_pct": {"type": "number", "minimum": 0, "maximum": 100},
                "checks": {"type": "array"},
            },
        },
    },
    "required": ["producer", "schema"],
}
```

**Testing:**
- `test_create_contract`: Create a contract with valid spec -> 201
- `test_contract_spec_validation`: Invalid spec -> 422 with JSON Schema validation errors
- `test_contract_versioning`: Publishing a new version increments semver
- `test_contract_subscription`: Consumer team subscribes to a contract
- `test_list_contracts_for_dataset`: GET endpoint returns all contracts for a dataset

#### Task 7.2: Contract Violation Detection

**What:** After each scan, check whether the results violate any active contracts on the scanned datasets. Violations (schema mismatch, freshness SLA breach, quality SLA breach) generate alerts tagged with the contract ID.

**Design:**
```python
# src/dqm/services/contract_service.py
class ContractService:
    async def check_violations(self, dataset: Dataset,
                                scan_results: list[CheckResult]) -> list[ContractViolation]:
        contracts = await self._get_active_contracts(dataset.id)
        violations = []

        for contract in contracts:
            # Schema check: current columns match contract spec
            schema_violations = self._check_schema_compliance(
                dataset.columns, contract.spec["schema"]["columns"]
            )

            # Freshness SLA check
            if "freshness_sla_minutes" in contract.spec.get("quality", {}):
                freshness_violations = self._check_freshness_sla(
                    dataset, contract.spec["quality"]["freshness_sla_minutes"]
                )
                violations.extend(freshness_violations)

            # Quality SLA check (completeness)
            if "completeness_sla_pct" in contract.spec.get("quality", {}):
                quality_violations = self._check_quality_sla(
                    scan_results, contract.spec["quality"]["completeness_sla_pct"]
                )
                violations.extend(quality_violations)

            violations.extend(schema_violations)

        return violations
```

**Testing:**
- `test_schema_violation_missing_column`: Contract requires column "email"; dataset lacks it -> violation
- `test_schema_violation_type_mismatch`: Contract says "integer"; column is "varchar" -> violation
- `test_freshness_sla_violation`: Table updated 120 mins ago, SLA is 60 mins -> violation
- `test_completeness_sla_violation`: Null rate 2% when SLA is 99.5% completeness -> violation
- `test_no_violation_when_compliant`: All SLAs met -> empty violation list
- `test_violation_creates_alert`: Violation generates an alert tagged with contract ID

#### Task 7.3: Compliance Reporting

**What:** Generate compliance reports showing data quality posture across the organisation. Reports align with ISO/IEC 25012 quality dimensions and support GDPR/HIPAA/BCBS 239 audit requirements.

**Design:**
```python
# src/dqm/api/reports.py
@router.get("/api/v1/reports/quality-scorecard")
async def quality_scorecard(org_id: UUID, date_from: date, date_to: date):
    """Generate a quality scorecard broken down by ISO 25012 dimension."""
    return {
        "period": {"from": date_from, "to": date_to},
        "dimensions": [
            {"dimension": "completeness", "checks_total": 120, "checks_passed": 115,
             "pass_rate": 95.8, "trend": "improving"},
            {"dimension": "currentness", "checks_total": 30, "checks_passed": 28,
             "pass_rate": 93.3, "trend": "stable"},
            # ...
        ],
        "contract_compliance": {
            "total_contracts": 12,
            "compliant": 10,
            "violated": 2,
            "compliance_rate": 83.3,
        },
    }
```

**Testing:**
- `test_scorecard_by_dimension`: Scorecard breaks down pass rates by ISO 25012 dimension
- `test_scorecard_date_range_filter`: Only includes results within the specified date range
- `test_contract_compliance_summary`: Compliance rate calculated correctly
- `test_scorecard_trend_calculation`: Trend computed by comparing current vs. previous period

#### Phase 7 — Definition of Done

- [ ] Data contracts: create, version, publish, subscribe, deprecate
- [ ] Contract specs validated against JSON Schema (ODCS-aligned)
- [ ] Automated violation detection on every scan: schema, freshness SLA, quality SLA
- [ ] Violations generate tagged alerts with contract context
- [ ] Quality scorecard API with ISO 25012 dimensional breakdown
- [ ] Contract compliance reporting
- [ ] Test coverage: >80% on contract and reporting code

---

### Phase 8: Lineage & Root-Cause Analysis

**Goal:** Build comprehensive data lineage tracking and automated root-cause analysis that traces quality failures back to their upstream origin.

**Duration:** 5 weeks

**Prerequisites:** Phase 6 complete (lineage edges populated from dbt/OpenLineage), Phase 5 (AI explanations provide narrative layer)

#### Task 8.1: Lineage Graph API & Visualisation

**What:** REST API for querying upstream and downstream lineage. Recursive CTE-based traversal with depth limits. Frontend lineage graph visualisation using a directed graph layout.

**Design:**
```python
# src/dqm/api/lineage.py
@router.get("/api/v1/lineage/{dataset_id}/upstream")
async def get_upstream_lineage(dataset_id: UUID, max_depth: int = 5):
    """Traverse upstream lineage via recursive CTE."""
    query = text("""
        WITH RECURSIVE upstream AS (
            SELECT source_dataset_id, target_dataset_id,
                   detail->>'job_name' AS job_name,
                   detail->>'discovered_via' AS discovered_via,
                   1 AS depth
            FROM lineage_edges
            WHERE target_dataset_id = :dataset_id
          UNION ALL
            SELECT le.source_dataset_id, le.target_dataset_id,
                   le.detail->>'job_name', le.detail->>'discovered_via',
                   u.depth + 1
            FROM lineage_edges le
            JOIN upstream u ON le.target_dataset_id = u.source_dataset_id
            WHERE u.depth < :max_depth
        )
        SELECT DISTINCT d.id, d.schema_name, d.table_name,
               u.job_name, u.discovered_via, u.depth
        FROM upstream u
        JOIN datasets d ON u.source_dataset_id = d.id
        ORDER BY u.depth
    """)
    result = await db.execute(query, {"dataset_id": dataset_id, "max_depth": max_depth})
    return [LineageNode(**row._mapping) for row in result.fetchall()]
```

**Testing:**
- `test_upstream_lineage_single_hop`: A -> B, query B upstream -> returns A
- `test_upstream_lineage_multi_hop`: A -> B -> C, query C upstream -> returns B (depth 1) and A (depth 2)
- `test_downstream_lineage`: A -> B -> C, query A downstream -> returns B and C
- `test_lineage_depth_limit`: Chain of 10 tables, max_depth=3 -> only first 3 returned
- `test_lineage_no_cycles`: Circular reference A -> B -> A does not cause infinite recursion
- `test_lineage_graph_ui`: Frontend renders directed graph with correct node positions and edge labels

#### Task 8.2: Automated Root-Cause Analysis

**What:** When a check fails, automatically traverse upstream lineage to find correlated failures (same time window), schema changes, or freshness issues on upstream tables. Combine with AI explanation from Phase 5 for a comprehensive root-cause narrative.

**Design:**
```python
# src/dqm/services/root_cause_service.py
class RootCauseService:
    async def analyse(self, failing_result: CheckResult) -> RootCauseReport:
        # 1. Get upstream datasets
        upstream = await self.lineage_service.get_upstream(
            failing_result.dataset_id, max_depth=5
        )

        # 2. Find correlated failures on upstream datasets (within 24h window)
        upstream_failures = []
        for node in upstream:
            failures = await self._get_recent_failures(
                node.id, since=failing_result.executed_at - timedelta(hours=24)
            )
            upstream_failures.extend(failures)

        # 3. Find recent schema changes on upstream datasets
        schema_changes = []
        for node in upstream:
            changes = await self._get_recent_schema_changes(
                node.id, since=failing_result.executed_at - timedelta(hours=48)
            )
            schema_changes.extend(changes)

        # 4. Generate AI explanation with full context
        explanation = await self.explainer.explain(
            check_result=failing_result,
            upstream_failures=upstream_failures,
            schema_changes=schema_changes,
        )

        return RootCauseReport(
            failing_check=failing_result,
            upstream_failures=upstream_failures,
            schema_changes=schema_changes,
            probable_cause=self._rank_causes(upstream_failures, schema_changes),
            ai_explanation=explanation,
        )
```

**Testing:**
- `test_root_cause_finds_upstream_failure`: Upstream table has a check failure in the same window -> included in report
- `test_root_cause_finds_schema_change`: Upstream table had a column removed recently -> included
- `test_root_cause_ranks_by_proximity`: Closer upstream failure ranked higher than distant one
- `test_root_cause_ai_explanation`: Report includes AI-generated narrative referencing the upstream cause
- `test_root_cause_no_upstream`: No lineage data -> report shows "no upstream context available"
- `test_root_cause_api`: GET `/api/v1/checks/results/{id}/root-cause` returns full root-cause report

#### Phase 8 — Definition of Done

- [ ] Upstream and downstream lineage traversal APIs with configurable depth
- [ ] Lineage graph visualisation in the frontend with interactive exploration
- [ ] Automated root-cause analysis on check failures: upstream failures, schema changes, freshness issues
- [ ] Root-cause reports combine heuristic ranking with AI-generated explanations
- [ ] Column-level lineage display (from OpenLineage/dbt data)
- [ ] Test coverage: >80% on lineage and root-cause code

---

### Phase 9: Streaming & Real-Time

**Goal:** Extend the platform from batch-only monitoring to support real-time quality validation on streaming data sources (Kafka, Kinesis).

**Duration:** 5 weeks

**Prerequisites:** Phase 3 complete (check engine and alerting), Phase 4 (anomaly baselines)

#### Task 9.1: Streaming Connector Framework

**What:** Extend the `BaseConnector` abstraction to support streaming data sources. Implement a Kafka consumer that reads messages, applies configured checks in micro-batches, and emits results.

**Design:**
```python
# src/dqm/connectors/kafka.py
from aiokafka import AIOKafkaConsumer

class KafkaStreamingConnector:
    def __init__(self, config: dict):
        self.bootstrap_servers = config["bootstrap_servers"]
        self.topic = config["topic"]
        self.group_id = config.get("group_id", "dqm-quality-monitor")
        self.batch_size = config.get("batch_size", 1000)
        self.batch_timeout_seconds = config.get("batch_timeout_seconds", 30)

    async def consume_and_validate(self, checks: list[Check],
                                    callback: Callable):
        consumer = AIOKafkaConsumer(
            self.topic,
            bootstrap_servers=self.bootstrap_servers,
            group_id=self.group_id,
        )
        await consumer.start()

        batch = []
        try:
            async for msg in consumer:
                batch.append(json.loads(msg.value))
                if len(batch) >= self.batch_size:
                    results = await self._validate_batch(batch, checks)
                    await callback(results)
                    batch = []
        finally:
            await consumer.stop()
```

**Testing:**
- `test_kafka_consumer_reads_messages`: Consumer reads from topic and accumulates batch
- `test_kafka_batch_validation`: Batch of 1000 messages validated against checks
- `test_kafka_result_callback`: Results are passed to callback function
- `test_kafka_reconnect_on_failure`: Consumer reconnects after broker disconnect
- `test_batch_timeout`: Partial batch is flushed after timeout period

#### Task 9.2: Streaming Dashboard

**What:** Real-time dashboard panel showing streaming check results, throughput, and error rates with auto-refreshing via WebSocket.

**Testing:**
- `test_websocket_connection`: Frontend connects to WebSocket and receives events
- `test_streaming_metrics_update`: Metrics update in real-time as new batches are processed
- `test_streaming_alert_display`: Streaming check failure appears in alerts within 5 seconds

#### Phase 9 — Definition of Done

- [ ] Kafka connector: consume, validate in micro-batches, emit results
- [ ] Streaming checks use the same check engine as batch checks
- [ ] Real-time dashboard with WebSocket-driven updates
- [ ] Streaming anomaly detection using the same baseline infrastructure
- [ ] Test coverage: >80% on streaming code

---

### Phase 10: Enterprise & Scale

**Goal:** Hardening for enterprise deployments: multi-tenancy via Row-Level Security, Helm charts for Kubernetes, horizontal scaling, data retention policies, and SSO (SAML 2.0).

**Duration:** 6 weeks

**Prerequisites:** Phases 1-8 substantially complete

#### Task 10.1: Row-Level Security & Multi-Tenancy

**What:** Enable PostgreSQL Row-Level Security (RLS) policies on all tenant-scoped tables. Every query automatically filters by `org_id` based on the authenticated user's organisation.

**Design:**
```sql
-- Enable RLS on all tenant tables
ALTER TABLE datasets ENABLE ROW LEVEL SECURITY;
CREATE POLICY org_isolation ON datasets
    USING (org_id = current_setting('app.current_org_id')::uuid);
-- Repeat for all tenant-scoped tables
```

**Testing:**
- `test_rls_prevents_cross_tenant_access`: User from org A cannot see org B's datasets
- `test_rls_admin_bypass`: Superuser can access all organisations
- `test_rls_performance`: RLS does not add >5% overhead on typical queries

#### Task 10.2: Kubernetes Deployment (Helm Charts)

**What:** Create Helm charts for deploying the full stack (API, workers, scheduler, frontend, PostgreSQL, Redis) on Kubernetes with configurable resource limits, autoscaling, and health checks.

**Testing:**
- `test_helm_template_renders`: `helm template` renders valid Kubernetes manifests
- `test_helm_install_minikube`: Full stack deploys on minikube and passes health checks
- `test_horizontal_pod_autoscaler`: API scales from 2 to 4 replicas under load

#### Task 10.3: Data Retention & Archival

**What:** Implement configurable retention policies. Old check results and metrics are archived to cold storage (S3) and partitions are dropped from PostgreSQL.

**Testing:**
- `test_retention_policy_drops_partitions`: Partitions older than retention period are dropped
- `test_archived_data_accessible`: Archived data can be queried via an archive API
- `test_retention_respects_org_settings`: Different orgs can have different retention periods

#### Task 10.4: SSO (SAML 2.0 / OIDC)

**What:** Implement SAML 2.0 SSO for enterprise customers. Support Okta, Azure AD, and OneLogin as identity providers.

**Testing:**
- `test_saml_login_flow`: SAML assertion creates/updates user and issues JWT
- `test_saml_attribute_mapping`: SAML attributes mapped to user roles
- `test_saml_logout`: Single logout invalidates session

#### Phase 10 — Definition of Done

- [ ] Row-Level Security active on all tenant-scoped tables
- [ ] Helm charts published and tested on minikube
- [ ] Horizontal autoscaling for API and worker pods
- [ ] Data retention policies with configurable periods per organisation
- [ ] SAML 2.0 SSO with Okta/Azure AD/OneLogin
- [ ] Performance benchmarks: API p99 latency <200ms, scan throughput >100 tables/minute
- [ ] Security audit: OWASP top 10 mitigated
- [ ] Documentation: deployment guide, operator manual, API reference

---

## 4. Summary Timeline

| Phase | Name | Duration | Cumulative |
|-------|------|----------|------------|
| 1 | Foundation & Data Source Connectivity | 4 weeks | Month 1 |
| 2 | Check Engine & Rule-Based Validation | 4 weeks | Month 2 |
| 3 | Scheduling, Alerting & Web UI | 5 weeks | Month 3–4 |
| 4 | ML Anomaly Detection | 4 weeks | Month 4–5 |
| 5 | AI-Powered Features | 5 weeks | Month 5–6 |
| 6 | Integrations (dbt, Airflow, OpenLineage) | 4 weeks | Month 6–7* |
| 7 | Data Contracts & Governance | 4 weeks | Month 7–8* |
| 8 | Lineage & Root-Cause Analysis | 5 weeks | Month 8–9* |
| 9 | Streaming & Real-Time | 5 weeks | Month 9–10* |
| 10 | Enterprise & Scale | 6 weeks | Month 10–12* |

*Phases 6-10 can be parallelised across tracks (see dependency graph). With two parallel tracks, total calendar time is approximately 9-10 months.

**MVP (Phases 1-3):** Functional data quality monitoring platform with rule-based checks, scheduling, alerting, and web UI. Sufficient for initial deployments and user feedback. Target: **Month 4**.

**v1.0 (Phases 1-5):** AI-native platform with ML anomaly detection, AI-generated checks, and natural-language explanations. The full competitive differentiator against Great Expectations and Soda Core. Target: **Month 6**.

**v2.0 (Phases 1-10):** Enterprise-ready platform with integrations, lineage, contracts, streaming, and multi-tenant hardening. Target: **Month 10-12**.
