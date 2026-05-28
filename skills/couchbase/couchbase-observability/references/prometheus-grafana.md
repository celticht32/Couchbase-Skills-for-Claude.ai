# Prometheus and Grafana

## Couchbase Prometheus integration

Couchbase 7.0+ exposes a Prometheus-compatible `/metrics` endpoint on port 8091 (HTTP) or 18091 (HTTPS).

### Scrape configuration

```yaml
# prometheus.yml
scrape_configs:
  - job_name: couchbase
    scheme: https
    tls_config:
      ca_file: /etc/prometheus/couchbase-ca.crt
      insecure_skip_verify: false
    basic_auth:
      username: prometheus-reader
      password: <service-account-password>
    static_configs:
      - targets:
          - "couchbase-node1.example.com:18091"
          - "couchbase-node2.example.com:18091"
          - "couchbase-node3.example.com:18091"
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
      - target_label: cluster
        replacement: "prod"
    metric_relabel_configs:
      # Drop high-cardinality per-request metrics if not needed
      - source_labels: [__name__]
        regex: "cm_.*"
        action: drop
```

Create a dedicated read-only Couchbase user for Prometheus scraping:
```python
admin_user_upsert_user(
    domain="local",
    username="prometheus-reader",
    roles="cluster_admin",   # needs cluster_admin to read all stats; no data access
    password="<generated>",
    full_name="Prometheus scrape account",
    cluster="prod"
)
```

Note: `cluster_admin` for Prometheus is standard practice — the stats API requires this role. It has no data read/write capability.

### Using admin_stats_prometheus_targets

`admin_stats_prometheus_targets` returns the current list of Prometheus scrape targets with their URLs and authentication hints. Use this to auto-configure or verify your scrape config.

## Key metric names in the Prometheus exposition

Couchbase metrics are prefixed by service. Common ones:

```
# Bucket memory
kv_ep_mem_high_wat_ratio{bucket="orders"}
kv_mem_used_bytes{bucket="orders"}
kv_ep_cache_miss_ratio{bucket="orders"}

# Bucket ops
kv_ops{bucket="orders", op="get"}
kv_ops{bucket="orders", op="set"}
kv_ep_bg_fetched_total{bucket="orders"}

# Query service
n1ql_requests_total
n1ql_request_time_seconds{quantile="0.99"}
n1ql_errors_total

# GSI indexes
gsi_memory_used_bytes
gsi_memory_quota_bytes

# XDCR
xdcr_changes_left_total{pipeline="orders/to-dr"}

# Node
sys_mem_actual_free_bytes
sys_cpu_utilization_rate
couch_total_disk_size_bytes
```

## Grafana dashboard structure

A practical production dashboard has four sections:

### 1. Cluster overview (top of page)

Single-stat panels:
- Cluster health (derived: all nodes active+healthy? → green/red)
- Total ops/sec
- Memory utilization (highest node)
- Disk utilization (highest node)
- Active connections

### 2. Per-node resource heatmap

Time series for each node:
- `kv_mem_used_bytes` grouped by `node`
- `sys_cpu_utilization_rate` grouped by `node`
- Disk used/total grouped by `node`

### 3. Bucket performance

Time series per bucket:
- Ops breakdown (get / set / delete)
- Cache miss ratio
- DGM (disk greater than memory) ratio
- Background fetches

### 4. Service health

Per-service panels:
- Query: p99 latency, active requests, queued requests, error rate
- Index: RAM utilization %, scanned vs returned ratio
- FTS: active queries, indexing rate
- XDCR: `changes_left` per pipeline

## Recording rules (Prometheus)

Pre-compute expensive queries to keep dashboards fast:

```yaml
groups:
  - name: couchbase
    interval: 60s
    rules:
      - record: couchbase:cluster:ops_total
        expr: sum(rate(kv_ops_total[5m])) by (cluster)

      - record: couchbase:bucket:cache_miss_ratio
        expr: |
          rate(kv_ep_bg_fetched_total[5m])
          / (rate(kv_ops_total{op="get"}[5m]) + 1)

      - record: couchbase:node:mem_utilization
        expr: |
          kv_mem_used_bytes
          / kv_ep_mem_high_wat_bytes

      - record: couchbase:xdcr:changes_left_max
        expr: max(xdcr_changes_left_total) by (cluster, pipeline)
```

## Couchbase Exporter (open source)

The community [couchbase-exporter](https://github.com/couchbase/couchbase-exporter) provides richer metric coverage than the built-in `/metrics` endpoint, including per-node breakdowns and additional internal metrics. It runs as a sidecar or standalone service.

Use it if the built-in metrics don't give you enough granularity — particularly for per-vBucket stats or detailed compaction metrics.
