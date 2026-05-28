# Key metrics

## Memory metrics

### `mem_used` / `ep_mem_high_wat` / `ep_mem_low_wat`

The most critical cluster health signal. When `mem_used` crosses the **high watermark** (`ep_mem_high_wat`, default 85% of bucket quota), the data service starts **ejecting** documents from RAM to disk to reclaim memory. Ejections increase read latency (cache misses require disk reads).

When `mem_used` drops below the **low watermark** (`ep_mem_low_wat`, default 75%), ejections stop.

| Metric | Meaning | Threshold |
|---|---|---|
| `mem_used / ep_mem_high_wat` | How close to ejection threshold | Page at > 90% |
| `ep_tmp_oom_errors` | Temporary out-of-memory errors (writes rejected) | Alert at any non-zero |
| `ep_oom_errors` | Permanent OOM errors (writes permanently rejected) | Critical — immediate action |

### `ep_cache_miss_rate`

Percentage of KV reads that required a disk read (document wasn't in RAM). For latency-sensitive workloads, keep this low.

| Workload type | Acceptable cache miss rate |
|---|---|
| Session / user profile (hot data) | < 1% |
| E-commerce product catalog | < 5% |
| General mixed workload | < 20% |
| Analytics / reporting | Higher acceptable — data is cold by design |

A rising cache miss rate with stable document count usually means the working set has grown larger than available RAM. Time to add nodes or increase quota.

## Disk metrics

### `ep_diskqueue_drain` / `ep_diskqueue_fill`

The disk write queue. `fill` is the rate at which mutations enter the queue; `drain` is the rate at which they're written to disk. If `fill > drain` consistently, the queue grows and eventually causes memory pressure.

### `couch_total_disk_size` / `couch_docs_actual_disk_size`

Total vs actual data on disk. The difference is fragmentation (deleted/updated documents that haven't been compacted yet). High fragmentation wastes disk and can slow reads.

Auto-compaction (configured in bucket settings) should keep fragmentation below 30%. If it's higher, compaction isn't keeping up — check compaction settings and node I/O load.

| Metric | Threshold |
|---|---|
| `hdd_used / hdd_total` (node disk) | Warn at 70%, page at 85% |
| Fragmentation ratio | Warn at > 30% |

## Operations metrics

### `ops` (operations per second)

Total KV operations/sec on the bucket. Your baseline. Sudden drops (service degradation) or unexpected spikes (traffic anomaly, infinite retry loop) are both signals.

### `cmd_get` / `cmd_set` / `delete_hits`

Read/write/delete breakdown. Useful for capacity planning and anomaly detection.

### `ep_bg_fetched`

Background fetches — KV reads that had to go to disk because the document wasn't in RAM. Directly related to cache miss rate. A spike here means workload exceeded working set.

## Replication and DCP metrics

### `ep_dcp_replica_items_remaining`

Items in the DCP replication queue for internal replicas. Should be near 0 in steady state. Non-zero and growing means replica replication is falling behind — often due to a slow replica node or disk I/O pressure.

### XDCR `changes_left`

See `couchbase-xdcr/references/troubleshooting.md`. Alert when growing over time.

## Query service metrics

### `query_requests` / `query_errors` / `query_warnings`

Request rate, error rate, warning rate. A rising error rate relative to request rate indicates query problems.

### `query_request_time` (p50, p95, p99)

Query latency percentiles. Baseline these and alert on sustained p99 > your SLO threshold.

### `query_active_requests` / `query_queued_requests`

Currently running and queued queries. Queued requests indicate the query service is saturated. Alert when `query_queued_requests > 0` for more than 30 seconds.

## Index (GSI) metrics

### `index_ram_percent`

Percentage of GSI memory quota used. GSI indexes need to fit in RAM for fast scan performance. When this approaches 100%, index scans start reading from disk.

| Metric | Threshold |
|---|---|
| `index_ram_percent` | Warn at 80%, page at 90% |
| `index_disk_size` | Track for growth; no hard threshold |

### `index_num_rows_scanned` / `index_num_rows_returned`

Rows scanned vs returned. A high ratio (many rows scanned, few returned) indicates a poorly selective index. Use this to find indexes that could be improved with partial index predicates.

## Node health

### Node status (`clusterMembership`, `status`)

From `admin_stats_nodes`, each node has:
- `clusterMembership`: `active` (healthy), `inactiveFailed` (failed over), `inactiveAdded` (being added)
- `status`: `healthy`, `warmup` (loading data), `unhealthy`

Alert on any node not `active` + `healthy`.

### `curr_connections`

Open connections to the data service on a node. Unexpected spikes may indicate a connection leak in an SDK client or a retry storm.
