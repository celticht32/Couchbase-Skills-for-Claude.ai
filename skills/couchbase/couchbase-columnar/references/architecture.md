# Columnar architecture

## What Capella Columnar is

Capella Columnar is a standalone columnar database on Capella. It stores data in columnar format (each field stored together, compressed), which makes analytical queries — scans, aggregations, GROUP BY, window functions — dramatically faster than row-oriented storage.

It's a separate Capella product from the operational cluster, provisioned independently and billed separately.

## How data gets into Columnar

Columnar doesn't accept direct writes from your application. Data flows in through **links**:

- **Capella operational cluster link:** streams data from your Capella cluster's collections into Columnar via a managed DCP pipeline. Changes in the operational cluster appear in Columnar with configurable lag (near-real-time to minutes).
- **S3 / Azure Blob / GCS link:** query files stored in object storage (CSV, JSON, Parquet) directly from Columnar. Data is not ingested; it's queried in place.
- **External Couchbase link:** connect to a self-managed Couchbase Server cluster.

## Columnar vs Analytics Service — decision guide

Use the **Analytics Service** (cb-analytics-* skills) when:
- Your cluster is self-managed (not on Capella)
- Your analytical queries run alongside the operational workload and sharing nodes is acceptable
- You need analytics on a single operational cluster with minimal infrastructure overhead
- Your analytical query volume is moderate (not competing with heavy OLAP workloads)

Use **Capella Columnar** when:
- You need true columnar storage for large-scale analytical performance
- Your analytical queries are heavy and isolating them from the operational cluster is important
- You want BI tool connectivity (JDBC/ODBC drivers, Tableau, Power BI, Looker)
- You need to query data from multiple sources in one place (Capella cluster + S3 + external)
- You're building a data warehouse or analytical data mart on top of operational data

## Columnar data model

Columnar uses scopes and collections (same naming as operational Couchbase), but stored in columnar format. You define **Columnar collections** that mirror or transform operational collections. The schema is inferred from the JSON data.

Unlike the operational cluster, Columnar collections can also have explicitly defined schemas (column types) for better compression and query performance on structured data.

## Isolation and performance

Because Columnar runs on a separate instance, analytical queries don't compete with operational workload for CPU, memory, or disk I/O. This is the key production argument for Columnar over the Analytics Service — no "noisy neighbor" effect from heavy analytical queries degrading your application's response time.
