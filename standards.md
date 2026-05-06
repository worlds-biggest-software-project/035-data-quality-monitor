# Standards & API Reference

> Project: Data Quality Monitor · Generated: 2026-05-04

## Industry Standards & Specifications

### ISO Standards

#### ISO 8000 — Data Quality (multi-part series)
- **URL:** https://www.iso.org/standard/81745.html (Part 1: Overview, 2022)
- **Relevance:** The primary international standard for data quality and master data exchange. ISO 8000-1:2022 provides the overarching framework; key sub-parts include ISO 8000-2 (Vocabulary), ISO 8000-8 (Concepts and measuring), ISO 8000-61/62 (provenance of data values), and ISO 8000-150:2022 (data quality management roles and responsibilities). A Data Quality Monitor should use ISO 8000 vocabulary and dimensional definitions as the normative basis for its quality checks and reporting.

#### ISO/IEC 25012:2008 — Data Quality Model (SQuaRE)
- **URL:** https://www.iso.org/standard/35736.html
- **Relevance:** Defines 15 data quality characteristics (accuracy, completeness, consistency, credibility, currentness, accessibility, compliance, confidentiality, efficiency, precision, traceability, understandability, availability, portability, recoverability) across two perspectives — inherent quality and system-dependent quality. These 15 characteristics map directly onto the check categories a quality monitor should implement (null rates → completeness; schema drift → consistency; freshness → currentness; duplicates → uniqueness/precision). Confirmed current as of 2025 review.

#### ISO/IEC 25024:2015 — Measurement of Data Quality (SQuaRE)
- **URL:** https://www.iso.org/standard/35749.html
- **Relevance:** Companion to ISO/IEC 25012; specifies concrete measures (metrics and measurement functions) for each of the 25012 quality characteristics. Directly applicable to implementing the metric computation engine: defines what to measure, how to calculate it, and which target entities (tables, columns, records) to apply measurements to across the full data lifecycle.

---

### W3C & IETF Standards

#### W3C PROV — Provenance Data Model
- **URL:** https://www.w3.org/TR/prov-overview/ (overview); https://www.w3.org/TR/prov-dm/ (data model)
- **Relevance:** W3C Recommendation defining a vocabulary and model for representing provenance — the history of entities, activities, and agents. A data quality platform needs to record why a quality check ran, which dataset version was assessed, and who or what triggered the run. PROV-DM (entity/activity/agent triples) and PROV-O (OWL ontology) provide the semantic model; PROV-N the compact notation. Essential for lineage-aware quality reporting.

#### W3C SHACL — Shapes Constraint Language
- **URL:** https://www.w3.org/TR/shacl/
- **Relevance:** W3C Recommendation (2017) for expressing structural constraints on RDF data graphs. SHACL shapes define rules (data types, cardinality, pattern matching, value ranges) that data nodes must satisfy, and validation reports identify violations. For a DQ monitor, SHACL provides the formal language for declaring schema and value constraints on linked-data or knowledge-graph datasets; SHACL 1.2 Core is under active development.

#### W3C DCAT — Data Catalog Vocabulary (Version 3)
- **URL:** https://www.w3.org/TR/vocab-dcat-3/
- **Relevance:** W3C Recommendation (2024) for describing datasets and data services in catalogs using a standard RDF vocabulary. DCAT metadata (dataset identifier, temporal coverage, publisher, distribution format) feeds directly into freshness monitoring and lineage tracking; integrating DCAT enables a quality monitor to discover and register datasets automatically from existing open-data catalogs and governance platforms.

#### IETF / JSON Schema — Draft 2020-12
- **URL:** https://json-schema.org/draft/2020-12 ; https://json-schema.org/specification
- **Relevance:** The de-facto standard for defining the structure, constraints, and validation rules for JSON documents, tracked through IETF. Draft 2020-12 introduces `prefixItems`, dynamic references (`$dynamicRef`), and a separated format vocabulary. A Data Quality Monitor uses JSON Schema to enforce schema contracts on JSON-encoded datasets and API payloads, and to define the schema of its own expectation suite and configuration files.

#### Dublin Core Metadata Initiative (DCMI)
- **URL:** https://www.dublincore.org/specifications/dublin-core/dces/
- **Relevance:** The 15-element Dublin Core Metadata Element Set (ISO 15836 / ANSI/NISO Z39.85) provides the minimum viable vocabulary for describing data assets: title, creator, date, format, rights, etc. Useful for annotating quality reports and monitored dataset records in an interoperable way; widely adopted in data repositories, data catalogs (DCAT profiles Dublin Core), and government open-data portals.

---

### Data Model & API Specifications

#### OpenLineage — Open Standard for Lineage Metadata
- **URL:** https://openlineage.io/ ; https://github.com/OpenLineage/OpenLineage/blob/main/spec/OpenLineage.md
- **Relevance:** Industry-standard JSON Schema + OpenAPI specification (originally introduced 2020, now the de facto standard) for collecting lineage metadata from running data pipelines. Defines core entities — Job, Run, Dataset — and extensible facets for enriching them. Directly applicable to a DQ monitor: quality check results are a natural OpenLineage facet; integrating with Airflow, Spark, dbt, and Flink via OpenLineage means quality events flow into the broader lineage graph automatically.

#### dbt Model Contracts & Data Tests
- **URL:** https://docs.getdbt.com/docs/mesh/govern/model-contracts ; https://docs.getdbt.com/docs/build/data-tests
- **Relevance:** dbt's contract mechanism (enforcing column names, data types, and constraints at build time) and its generic/singular test framework form a widely-adopted spec-as-code pattern for pipeline data quality. A DQ monitor should be able to ingest dbt test results via the dbt Discovery API and treat dbt contracts as a schema-contract source, closing the loop between transformation-layer quality and continuous monitoring.

#### Great Expectations Expectation Suites
- **URL:** https://docs.greatexpectations.io/docs/reference/ ; https://docs.greatexpectations.io/docs/reference/api_reference/
- **Relevance:** The GX expectation suite format (JSON-serialised expectations with parameters, result format, and evaluation parameters) is a widely-adopted open standard for expressing unit-test-style data quality rules in Python. Supporting import/export of GX expectation suites allows users migrating from GX Core to reuse existing quality rule libraries without rewriting them.

---

### Security & Authentication Standards

#### OAuth 2.0 / Bearer Token (RFC 6749 / RFC 6750)
- **URL:** https://datatracker.ietf.org/doc/html/rfc6749 ; https://datatracker.ietf.org/doc/html/rfc6750
- **Relevance:** The universal standard for delegated authorisation and API access tokens. A DQ monitor's REST/GraphQL API should implement OAuth 2.0 client-credentials flow for service-to-service authentication (pipeline agents, CI/CD integrations) and authorization-code flow for human user access. Bearer tokens in the `Authorization` header are the expected pattern across all comparable products (Great Expectations, Monte Carlo, Datafold, Bigeye).

#### GDPR — Regulation (EU) 2016/679, Article 5(1)(d) — Accuracy Principle
- **URL:** https://gdpr-info.eu/art-5-gdpr/
- **Relevance:** GDPR Article 5(1)(d) mandates that personal data must be accurate and kept up to date, with inaccurate data erased or rectified without delay. This creates a legal obligation to monitor and remediate data quality for any pipeline processing EU personal data. A DQ monitor addressing GDPR-regulated data sources must produce audit-ready quality reports, track correction events, and alert on accuracy degradation. Non-compliance carries fines up to €20 million or 4% of global annual turnover.

#### HIPAA Security Rule — 45 CFR § 164.312 — Integrity Controls
- **URL:** https://www.hhs.gov/hipaa/for-professionals/security/laws-regulations/index.html
- **Relevance:** HIPAA's technical safeguard standard (§164.312) requires covered entities to implement policies and corroboration mechanisms (checksums, message authentication, digital signatures) to protect ePHI from improper alteration or destruction. A DQ monitor operating on healthcare data must implement row-level checksums, change-detection auditing, and tamper-evident quality logs to satisfy these integrity controls.

#### CCPA / CPRA — California Consumer Privacy Act & Privacy Rights Act
- **URL:** https://oag.ca.gov/privacy/ccpa ; https://cppa.ca.gov/regulations/pdf/ccpa_statute_eff_20260101.pdf
- **Relevance:** The CPRA (effective 2023, updated regulations effective 2026-01-01) grants California consumers the right to correct inaccurate personal information. Businesses must respond to correction requests and maintain data accuracy. A DQ monitor helps demonstrate CCPA/CPRA compliance by continuously tracking accuracy metrics, logging when corrections occur, and generating reports showing that accuracy obligations are being met.

---

### Data Quality Framework Standards

#### TDQM — Total Data Quality Management Framework
- **URL:** https://mitiq.mit.edu/ (MIT Information Quality Program)
- **Relevance:** The academic framework originating from MIT's TDQM program defines data quality as "fitness for use" and categorises quality dimensions across product (correctness, completeness, currency) and service (accessibility, interpretability) perspectives. The TDQM IQ Assessment methodology — define, measure, analyse, improve — provides the conceptual model for a continuous monitoring platform's assessment and remediation cycle.

#### ISO/IEC 5259 — Data Quality for Analytics and Machine Learning (series)
- **URL:** https://www.iso.org/standard/81088.html (Part 1)
- **Relevance:** Emerging ISO series (Parts 1–5 published 2022–2024) specifically addressing data quality for AI/ML workloads. Covers quality model, quality processes, quality requirements, and data quality governance (Part 5). Particularly relevant as DQ monitors are increasingly deployed to gate ML feature pipelines; aligning with ISO/IEC 5259 positions the product for AI-governance use cases.

---

## Similar Products — Developer Documentation & APIs

### Great Expectations (GX Core / GX Cloud)
- **Description:** Open-source Python framework and managed cloud platform for defining, running, and sharing data quality expectations. GX Core is the free library; GX Cloud adds a managed UI, collaborative workflows, and a REST API layer.
- **API Documentation:** https://docs.greatexpectations.io/docs/reference/ (Python API reference); https://docs.greatexpectations.io/docs/cloud/ (GX Cloud REST API)
- **SDKs/Libraries:** Python library (`pip install great-expectations`); PyPI: https://pypi.org/project/great-expectations/ ; GitHub: https://github.com/great-expectations/great_expectations
- **Developer Guide:** https://docs.greatexpectations.io/docs/home/
- **Standards:** Python-native; GX Cloud exposes REST over HTTPS; expectation suites serialised as JSON (compatible with JSON Schema); OpenLineage integration available
- **Authentication:** GX Core — no auth (local); GX Cloud — user access tokens and organization access tokens (Bearer token in `Authorization` header), generated from Settings > Tokens in the GX Cloud UI

### Monte Carlo
- **Description:** Commercial data observability platform that automatically monitors tables for volume, freshness, schema changes, field health, and distribution anomalies using ML-based anomaly detection, without requiring manual rule authoring.
- **API Documentation:** https://docs.getmontecarlo.com/docs/api ; Full GraphQL reference: https://apidocs.getmontecarlo.com/
- **SDKs/Libraries:** Python SDK (`pip install pycarlo`); CLI tool; GitHub org: https://github.com/monte-carlo-data
- **Developer Guide:** https://docs.getmontecarlo.com/docs/developer-resources
- **Standards:** GraphQL API (not REST); HTTPS transport; Python SDK wraps GraphQL calls
- **Authentication:** API key pair (`x-mcd-id` and `x-mcd-token` HTTP headers). Keys generated from the Monte Carlo dashboard (Settings > API); key_secret shown once only. Both personal and account-level keys supported.

### Soda Core (v3)
- **Description:** Free, open-source Python library and CLI for data quality as code, powered by SodaCL (Soda Checks Language), a YAML-based DSL for writing data quality checks that run directly against databases.
- **API Documentation:** https://docs.soda.io/soda-core/overview-main.html ; Python API: https://docs.soda.io/soda-documentation/soda-v3/overview-main
- **SDKs/Libraries:** Python library (`pip install soda-core-<datasource>`); CLI tool; GitHub: https://github.com/sodadata/soda-core
- **Developer Guide:** https://docs.soda.io/
- **Standards:** YAML-based SodaCL DSL; Python API; integrates with dbt, Airflow, Spark; no proprietary REST API in the open-source tier
- **Authentication:** Soda Core (OSS) — no auth; Soda Cloud (managed) — API key via environment variables (`SODA_CLOUD_API_KEY` / `SODA_CLOUD_API_SECRET`)

### dbt (dbt Labs)
- **Description:** SQL-first transformation framework with built-in data testing via generic and singular tests, model contracts enforcing schema at build time, and a Discovery API for querying project metadata and test results programmatically.
- **API Documentation:** dbt Cloud Admin API v2/v3: https://docs.getdbt.com/dbt-cloud/api-v2 ; Discovery API: https://docs.getdbt.com/docs/dbt-cloud-apis/discovery-use-cases-and-examples ; APIs overview: https://docs.getdbt.com/docs/dbt-cloud-apis/overview
- **SDKs/Libraries:** dbt-core Python library; dbt Cloud Terraform provider; Postman collection: https://www.postman.com/dbtlabs/dbt-cloud-api-documentation/overview
- **Developer Guide:** https://docs.getdbt.com/
- **Standards:** REST (Admin API) and GraphQL (Discovery API); OpenAPI-documented endpoints; model contracts defined in YAML; integrates with OpenLineage
- **Authentication:** Personal Access Tokens (PATs) or Service Account Tokens; passed as `Authorization: Token <token>` header. OAuth 2.0 supported for SSO integrations.

### Datafold
- **Description:** Data diff and observability platform specialising in comparing datasets across environments (dev vs. prod, pre- vs. post-migration), detecting regressions in dbt CI pipelines, and monitoring for row-level and column-level anomalies.
- **API Documentation:** https://docs.datafold.com/api-reference/datafold-api ; REST API: https://docs.datafold.com/reference/cloud/rest_api/
- **SDKs/Libraries:** `data-diff` open-source CLI/library (GitHub: https://github.com/datafold/data-diff ; docs: https://data-diff.readthedocs.io/); Python client
- **Developer Guide:** https://docs.datafold.com/
- **Standards:** REST API (HTTPS); supports composite primary keys; column-level diffing; integrates with dbt CI
- **Authentication:** API key passed as `Authorization: Bearer <api_key>` header; keys generated from the Datafold application settings

### Bigeye
- **Description:** API-first data observability platform with automated metric monitoring, anomaly detection across 40+ data quality dimensions (freshness, volume, uniqueness, completeness, distribution, validity), and a Bigconfig infrastructure-as-code layer for managing monitors declaratively.
- **API Documentation:** https://docs.bigeye.com/docs/api-user-guide ; API key management: https://docs.bigeye.com/docs/using-api-keys
- **SDKs/Libraries:** Python SDK; CLI (`bigeye-cli`); Bigconfig YAML IaC layer; MCP server: https://github.com/bigeyedata/bigeye-mcp-server
- **Developer Guide:** https://docs.bigeye.com/
- **Standards:** REST API (HTTPS); OpenAPI-documented; YAML-based Bigconfig for monitors-as-code
- **Authentication:** Personal API Key (`apikey <key>` in `Authorization` header). Keys verified at `https://<company>.bigeye.com/api/v1/personal-api-keys/verify`. Agent API Keys also supported for embedded agents.

### AWS Deequ
- **Description:** Open-source library built on Apache Spark for computing data quality metrics and verifying constraints (unit tests for data) at scale on large datasets stored in S3 or other Hadoop-compatible stores; used internally at Amazon.
- **API Documentation:** GitHub (primary): https://github.com/awslabs/deequ ; PyDeequ (Python): https://github.com/awslabs/python-deequ ; AWS blog guide: https://aws.amazon.com/blogs/big-data/test-data-quality-at-scale-with-deequ/
- **SDKs/Libraries:** Scala/Java library (Maven/SBT); PyDeequ Python wrapper (`pip install pydeequ`; docs: https://pydeequ.readthedocs.io/)
- **Developer Guide:** https://aws.amazon.com/blogs/big-data/testing-data-quality-at-scale-with-pydeequ/
- **Standards:** Spark-native (no REST API); integrates with AWS Glue Data Quality; metrics exported as Spark DataFrames; Deequ 2.x requires Spark 3.1+
- **Authentication:** AWS IAM roles and policies (for S3/Glue access); no separate API key system — inherits Spark cluster credentials

### Talend Data Quality (now Qlik Talend)
- **Description:** Enterprise ETL and data quality suite (acquired by Qlik) providing data profiling, cleansing, standardisation, and matching components, with a REST API for the Talend Data Catalog and Talend Data Preparation products.
- **API Documentation:** Qlik Talend API portal: https://talend.qlik.dev/apis/ ; Data Catalog REST API: https://help.qlik.com/talend/en-US/talend-data-catalog/8.1/Subsystems/UserGuide/Content/REST-API.htm ; Component Kit HTTP API: https://talend.github.io/component-runtime/main/latest/documentation-rest.html
- **SDKs/Libraries:** Talend Component Kit (Java); REST API (Swagger/OpenAPI); Talend Studio (GUI-based ETL designer)
- **Developer Guide:** https://help.qlik.com/talend/ ; Qlik Talend developer docs: https://talend.qlik.dev/
- **Standards:** REST (OpenAPI / Swagger); SOAP for legacy connectors; Talend Data Preparation exposes a Swagger UI for interactive API exploration
- **Authentication:** Talend Cloud uses OAuth 2.0 bearer tokens via Qlik's IAM; on-premise installations use basic auth or LDAP integration

### Atlan
- **Description:** Active metadata platform and data catalog with built-in data quality integrations, lineage, and governance workflows; exposes a comprehensive REST and GraphQL API for automating asset management, quality metadata tagging, and policy enforcement.
- **API Documentation:** https://developer.atlan.com/ ; REST API reference: https://developer.atlan.com/sdks/raw/ ; API endpoints: https://developer.atlan.com/endpoints/ ; Full API docs: https://apidocs.atlan.com/
- **SDKs/Libraries:** Python SDK (`pip install pyatlan`); Java SDK; Go SDK; all available via https://developer.atlan.com/
- **Developer Guide:** https://developer.atlan.com/ ; Metadata type reference: https://developer.atlan.com/models/api/
- **Standards:** REST API (OpenAPI); JSON Schema for type definitions; Atlan's open API architecture documented at https://atlan.com/open-api-architecture/
- **Authentication:** Personal Access Tokens (PAT) or API tokens; passed as `Authorization: Bearer <token>` header; tokens generated from the Atlan UI settings

### OpenMetadata
- **Description:** Open-source, unified metadata platform providing data discovery, data quality test suites, column-level lineage, and team collaboration, with a schema-first REST API and native SDKs.
- **API Documentation:** https://docs.open-metadata.org/latest/main-concepts/metadata-standard/apis ; SDK/API reference: https://docs.open-metadata.org/v1.12.x/api-reference
- **SDKs/Libraries:** Python SDK (`pip install openmetadata-ingestion`); Java SDK; GitHub: https://github.com/open-metadata/OpenMetadata
- **Developer Guide:** https://docs.open-metadata.org/
- **Standards:** REST API (OpenAPI/Swagger, schema-first using JSON Schema for all entity types); OpenLineage integration; DCAT-aligned metadata model
- **Authentication:** JWT tokens; passed as `Authorization: Bearer <jwt>` header; also supports SSO via SAML 2.0 and Google/Okta OAuth

---

## Notes

1. **Standards convergence:** ISO/IEC 25012 and 25024 provide the most rigorous dimensional model for what to measure and how. OpenLineage is the de-facto wire protocol for lineage events. JSON Schema (Draft 2020-12) covers structural validation of JSON datasets and API payloads. These three together form the recommended normative core.

2. **GraphQL vs REST split:** Monte Carlo uses GraphQL exclusively; most other products (Datafold, Bigeye, Atlan, OpenMetadata) use REST with OpenAPI documentation. A new DQ monitor should offer a REST/OpenAPI interface for broad tooling compatibility, with optional GraphQL for complex traversal queries (lineage, impact analysis).

3. **Authentication pattern:** All commercial products use API keys or Bearer tokens (OAuth 2.0). Service-to-service flows use long-lived service tokens; user-interactive flows use short-lived access tokens. Implementing both patterns (service token + OAuth 2.0 client credentials) is the minimum viable authentication surface.

4. **Open-source vs commercial:** Great Expectations (GX Core), Soda Core, AWS Deequ, and OpenMetadata are freely available and can be studied for expectation/check schema design. Monte Carlo, Bigeye, Datafold, and Atlan are commercial SaaS products whose public API docs reveal interface patterns without requiring a license.

5. **Regulatory coverage:** GDPR Art. 5(1)(d), HIPAA § 164.312, and CCPA/CPRA together create a legal basis for selling a DQ monitor into financial services, healthcare, and any business handling EU or Californian personal data. Compliance reporting templates targeting these three regulations would be a significant product differentiator.

6. **ISO/IEC 5259 watch:** This emerging series (2022–2024) covering data quality for AI/ML is gaining adoption in AI governance frameworks. Aligning the product's quality dimensions and reporting outputs with ISO/IEC 5259 vocabulary positions it well for AI Act compliance workflows, particularly in the EU.
