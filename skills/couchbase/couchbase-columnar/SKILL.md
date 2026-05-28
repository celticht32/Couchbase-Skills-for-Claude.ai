---
name: couchbase-columnar
description: "Design and use Couchbase Capella Columnar, the columnar analytics database service. Use whenever the user asks about Capella Columnar, columnar analytics, Columnar instance, Columnar vs Analytics service, Columnar collections, Columnar links to operational data, Columnar SQL++, Columnar pricing, when to use Columnar vs the Analytics service, or 'I need to run analytical queries on my Couchbase data.' Distinct from the cb-analytics-* skills (which cover the Couchbase Analytics Service embedded in Couchbase Server / Capella operational clusters) — Capella Columnar is a separate, standalone columnar database product on Capella. Use proactively when the user needs OLAP-scale analytics, BI tool integration, or columnar storage for analytical workloads."
license: MIT
---

# Couchbase Capella Columnar

A skill for *designing and using* Couchbase Capella Columnar — the standalone columnar analytics database product on Capella.

Distinct from the `cb-analytics-*` skills, which cover the **Couchbase Analytics Service** embedded within Couchbase Server / Capella operational clusters. These are different products:

| | Analytics Service | Capella Columnar |
|---|---|---|
| **What it is** | Analytics service co-located with operational cluster | Standalone columnar database, separate instance |
| **Where it runs** | On Analytics nodes of an operational cluster | Separate Columnar instance on Capella |
| **Data ingestion** | DCP-based streaming from the operational cluster | Links to Capella operational cluster, S3, or external sources |
| **Storage** | Row-oriented (same as operational) | True columnar storage |
| **Best for** | Ad-hoc analytics alongside operational workloads | OLAP, BI tools, large analytical queries |
| **Skills** | `cb-analytics-*` | This skill |

## When this skill applies

- "What's Capella Columnar?"
- "Should I use Columnar or the Analytics Service?"
- "How do I connect Columnar to my operational Capella cluster?"
- "How do I connect a BI tool (Tableau, Power BI, Looker) to Couchbase?"
- "How do I query JSON data in Columnar?"
- "What's the difference between Columnar collections and operational collections?"

## Pick the right reference

| Question | Read |
|---|---|
| "Columnar architecture, when to use it vs Analytics Service" | `references/architecture.md` |
| "Connecting data sources, querying, BI tool integration" | `references/usage.md` |

## Core principle: Columnar is for OLAP, not OLTP

Columnar is optimized for analytical queries — large scans, aggregations, GROUP BY, window functions, JOINs across large datasets. It is not a replacement for the operational cluster. Transactional writes (your application's reads/writes) still go to the operational cluster. Columnar receives data via links and stores it in columnar format for efficient analytical access.

## Related skills

- `cb-analytics-*` — the Analytics Service embedded in operational clusters (different product)
- `couchbase-capella` — provisioning the operational Capella cluster that Columnar links to
- `couchbase-sqlpp-tuning` — SQL++ query patterns apply to Columnar with minor differences
