# Capella Analytics usage

> Capella Analytics was formerly named **Capella Columnar**. Some UI labels and driver names may still read "Columnar" during the transition; treat them as Capella Analytics.

## Provisioning a Capella Analytics instance

In the Capella UI: **Analytics → Create Analytics Instance** (labeled "Columnar" in older UI builds).

Key settings:
- **Compute tier:** similar to operational cluster tiers but optimized for analytical workloads (more CPU relative to RAM)
- **Cloud and region:** should be the same region as your operational cluster to minimize link latency and data transfer costs
- **Storage:** Capella Analytics uses columnar-format storage (not the same as operational Couchbase storage)

## Creating a link to your operational cluster

Once the Analytics instance is running:

1. **Analytics UI → Data Sources → Add Data Source → Capella Operational Cluster**
2. Select your operational cluster
3. Choose the scope and collections to link
4. Configure the link — the link establishes a DCP stream from the operational cluster

The link replicates data in near-real-time. Latency from operational write to Analytics visibility is typically seconds to low minutes.

## Querying Capella Analytics with SQL++

Capella Analytics uses SQL++ (the same query language as the operational cluster and the embedded Analytics Service) with a few differences:

```sql
-- Standard aggregation
SELECT category, COUNT(*) AS count, AVG(price) AS avg_price
FROM `my-cluster`.shop.products
GROUP BY category
ORDER BY count DESC;

-- Window functions (stronger support in Capella Analytics than the embedded Analytics Service)
SELECT
    order_id,
    customer_id,
    total,
    SUM(total) OVER (PARTITION BY customer_id ORDER BY created_at) AS running_total
FROM `my-cluster`.orders.completed;

-- JOIN across collections
SELECT o.order_id, c.name AS customer_name, SUM(oi.quantity * oi.price) AS order_total
FROM `my-cluster`.orders.orders AS o
JOIN `my-cluster`.customers.profiles AS c ON o.customer_id = c.id
JOIN `my-cluster`.orders.order_items AS oi ON oi.order_id = o.order_id
GROUP BY o.order_id, c.name;
```

The collection reference format is `"cluster-link-name".scope.collection`. The cluster link name is what you configured when setting up the data source.

## Querying S3 data

```sql
-- Query JSON files directly in S3
SELECT *
FROM S3Object s
WHERE s.region = "us-east"
  AND s.year = 2026;

-- The S3 link must be configured first with AWS credentials
```

## BI tool connectivity

Capella Analytics provides JDBC and ODBC drivers for BI tool connectivity.

**Tableau:**
1. Download the Couchbase Analytics JDBC driver from docs.couchbase.com
2. In Tableau: Connect → Other Database (JDBC) → paste the Analytics JDBC URL
3. Authenticate with your Analytics database credentials

**Power BI:**
1. Download the ODBC driver
2. In Power BI: Get Data → ODBC → select the Analytics DSN
3. Browse and import collections as tables

**Looker / Looker Studio:**
Configure as a generic JDBC data source. Capella Analytics's SQL++ is compatible with standard SQL for most BI query patterns.

**Connection string format** (driver/endpoint names may still carry the `columnar` token during the rebrand transition — verify against the current driver docs):
```
jdbc:couchbase-analytics://<analytics-endpoint>:8095/
```

## Performance tips

**Partition your collections by time.** Columnar storage is most efficient when analytical queries scan a subset of columns. Partitioning by a date field (year, month) lets Capella Analytics prune entire time ranges at scan time.

**Use explicit schemas for structured data.** If your JSON documents have consistent field types, define an explicit schema. Better compression, faster type inference at query time.

**Avoid SELECT *.** The main performance advantage is columnar pruning — only reading the fields referenced in the query. `SELECT *` forces a full column scan and negates the advantage.

**Increase parallelism for large aggregations.** Capella Analytics scales analytical queries across all compute nodes. For very large datasets, a larger instance (more nodes) directly translates to faster query completion.
