# Data Quality Monitor

**Candidate #35** — An AI-native data quality platform that automatically detects, explains, and remediates data anomalies without requiring manual rule authorship, targeting SMB and mid-market data teams.

## Market Opportunity

The data quality tools market is **$4.16 billion in 2024**, projected to reach **$12–13 billion by 2033 at 12–13% CAGR**. The data observability sub-segment is even faster-growing at **15–16% CAGR**.

**Market dynamics:**
- Data pipelines fail silently: 41% of data teams report undetected data quality issues
- Rule authorship is a bottleneck: Great Expectations and Soda require engineers to manually enumerate constraints
- Enterprise tools (Monte Carlo, Anomalo) cost $100K+/year; open-source tools (GX, Soda) require SQL/Python expertise
- Mid-market gap: No tool serves teams with $5M–$100M data budgets who need more than dbt Tests but can't afford enterprise pricing

## What This Platform Solves

An **AI-native data quality platform** that learns baseline data distributions and automatically flags anomalies without requiring engineers to write rules. It combines:

- **No-code rule generation**: Inspect sample data and auto-generate quality expectations
- **Anomaly detection**: ML models learn baselines and flag deviations without manual threshold setting
- **Natural-language alerts**: Anomalies are explained in plain English with probable causes
- **Lineage-aware diagnostics**: Trace anomalies to their upstream origin in pipelines
- **Data sovereignty**: Self-hostable for regulated industries; no cloud-only lock-in
- **Affordable**: Free self-hosted; cloud tier from $1K–$3K/month

## Competitive Differentiation

| Aspect | This Platform | Great Expectations | Soda Cloud | Monte Carlo | Anomalo |
|--------|---------------|-------------------|-----------|------------|---------|
| **ML Anomaly Detection** | Yes | No | Limited | Yes | Yes |
| **Rule Authorship** | AI-generated | Manual | YAML (easier) | Auto | Auto |
| **Open Source** | Yes (Apache) | Yes (Apache) | Yes (Apache) | No | No |
| **Self-Hostable** | Yes | Yes | Yes (via Agent) | No | No |
| **Natural-Language Alerts** | Yes | No | No | No | Partial |
| **Price** | Free/Cloud | Free/GX Cloud | $500+/mo | $100K+/yr | Enterprise |

## Key Features

### Must-Have (MVP)
- Connection to Snowflake, BigQuery, Redshift, Databricks
- Rule-based checks for completeness, uniqueness, freshness, volume (ISO 25012 dimensions)
- Schema drift detection with configurable alerting
- Scheduled continuous monitoring (not on-demand)
- dbt integration: run quality checks as post-transformation validation
- Web UI for viewing check results and metric trends

### Should-Have (v1.1)
- AI-generated check suggestions from data profiles
- ML-based baseline anomaly detection without manual thresholds
- Natural-language alert explanations with probable root causes
- Data lineage integration to trace anomalies upstream
- Slack and PagerDuty native alerting
- Suppression and acknowledgement workflows

### Nice-to-Have (Backlog)
- Cross-table referential integrity discovery from join patterns
- Data Contracts: formalized schema and quality agreements
- Agentic AI investigation: autonomous triage without human prompting
- Streaming data quality (Kafka, Kinesis)
- Diagnostics Warehouse: persist all scan results in customer's warehouse

## Technology Stack

**Backend**: Python or Go, dbt integrations  
**ML/Anomaly Detection**: scikit-learn, isolation forest, or prophet  
**Frontend**: React, TypeScript  
**Data Access**: SQL API to warehouse  
**Licensing**: Apache 2.0 (fully permissive)

## Market Entry Strategy

1. **MVP Launch** (months 1–4): Rule-based checks + schema drift detection
2. **AI Features** (months 5–8): Auto-generated rule suggestions, ML anomaly detection
3. **Lineage & Explanation** (months 9–12): Root-cause tracing, natural-language alerts
4. **Monetization**: Open-source core + managed cloud tier ($1K–$3K/month), enterprise support ($5K+/month)

## Why This Matters

- **Cold-start problem**: All rule-based tools (GX, Soda, dbt Tests) require manual constraint enumeration. AI-native approach eliminates friction entirely.
- **Enterprise cost barrier**: Monte Carlo ($100K+/year) and Anomalo are enterprise-only. SMB/mid-market segment (data teams with $1–5M budgets) has no good option.
- **Open-source gap**: Great Expectations and Soda Core are rule-only. No open-source tool offers ML anomaly detection + natural-language explanations + self-hosting.
- **AI opportunity**: Rule generation from schema metadata and historical distributions is feasible with LLMs; no current tool does this end-to-end.

## Success Metrics

- **Year 1**: 500+ active deployments, $100K ARR from cloud + support contracts
- **Year 2**: 2,000+ active deployments, $750K ARR; featured in G2 Leaders
- **Year 3**: 5,000+ active deployments, $2.5M+ ARR; adopted by 50+ mid-market data platforms
