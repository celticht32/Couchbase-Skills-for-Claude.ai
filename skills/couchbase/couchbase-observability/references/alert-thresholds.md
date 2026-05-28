# Alert thresholds

These are starting-point thresholds. Baseline your cluster for at least one week before finalizing — workloads vary significantly.

## Severity levels

- **Warning:** investigate during business hours. Not impacting users yet but trending badly.
- **Page (P2):** wake someone up if this persists > 5 minutes.
- **Critical (P1):** immediate response; likely user-impacting.

## Memory

| Metric | Warning | Page | Critical | Notes |
|---|---|---|---|---|
| `kv_mem_used / ep_mem_high_wat` per node | > 80% | > 90% | > 95% | Ejections start at 85%; above 95% expect OOM errors |
| `ep_tmp_oom_errors` rate | Any > 0 | Sustained > 0 | — | Writes being rejected temporarily |
| `ep_oom_errors` | — | Any > 0 | — | Permanent write rejection; immediate action |
| GSI `index_ram_percent` | > 75% | > 85% | > 95% | Index scans degrade as disk I/O increases |

## Disk

| Metric | Warning | Page | Critical | Notes |
|---|---|---|---|---|
| `hdd_used / hdd_total` per node | > 65% | > 80% | > 90% | Leave room for compaction and temp files |
| Fragmentation (`1 - actual_disk / total_disk`) | > 25% | > 40% | — | Check compaction is running; I/O issue if not draining |
| `ep_diskqueue_fill - ep_diskqueue_drain` | > 1000 | > 10000 | — | Disk write queue growth; disk I/O bottleneck |

## Replication and DCP

| Metric | Warning | Page | Critical | Notes |
|---|---|---|---|---|
| `ep_dcp_replica_items_remaining` | > 1000 | > 50000 | Growing indefinitely | Internal replica lag |
| XDCR `changes_left` | > 10000 | > 100000 | Growing indefinitely | Tune worker count or check network |
| XDCR `docs_failed_cr_source` rate | — | Sustained growth | — | Unexpected in active-passive |

## Query service

| Metric | Warning | Page | Critical | Notes |
|---|---|---|---|---|
| `n1ql_request_time_seconds` p99 | > 2× baseline | > 5× baseline | > 10× baseline | Baseline after stable week |
| `n1ql_queued_requests` | > 0 for > 30s | > 10 | > 50 | Query service saturated |
| `n1ql_errors / n1ql_requests` error rate | > 1% | > 5% | > 10% | |
| `n1ql_active_requests` | > 50 | > 100 | > 200 | Rule of thumb; adjust for your concurrency profile |

## Node health

| Metric | Warning | Page | Critical | Notes |
|---|---|---|---|---|
| Node `status` | — | Any non-`healthy` | Any `unhealthy` | Check system events for cause |
| Node `clusterMembership` | — | Any `inactiveFailed` | — | Node has failed over |
| `sys_cpu_utilization_rate` per node | > 70% for > 5 min | > 85% for > 2 min | > 95% for > 1 min | Sustained high CPU degrades all services |
| `curr_connections` per node | > 2× baseline | > 5× baseline | — | Connection leak or retry storm |

## FTS / Search

| Metric | Warning | Page | Critical | Notes |
|---|---|---|---|---|
| FTS `num_mutations_to_index` | Growing | > 100K | > 1M | Index falling behind; add FTS nodes or reduce indexing scope |
| FTS RAM usage | > 80% of quota | > 90% | > 95% | |

## Cache miss ratio (by workload type)

| Workload | Warning | Page |
|---|---|---|
| Hot KV (sessions, user profiles) | > 5% | > 10% |
| Mixed transactional | > 20% | > 40% |
| Read-heavy catalog | > 10% | > 25% |

## Alert fatigue prevention

**Require duration before paging.** A single spike isn't a page. Require a metric to breach a threshold for 2–5 minutes before sending a page.

**Use warning-before-page.** A warning fires 15 minutes before the page threshold (e.g. warn at 70% disk, page at 80%). This gives the team time to act before things break.

**Correlate across metrics.** High memory AND rising cache miss ratio AND rising bg_fetched = a real memory pressure incident. High memory alone may be normal if it's stable and below the high watermark.

**Review and tune quarterly.** Stale thresholds that never fire are useless; thresholds that fire constantly are ignored. Both are alert failures.
