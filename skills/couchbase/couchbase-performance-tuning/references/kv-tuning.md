# KV tuning

## Understanding KV latency sources

KV operation latency has three layers:

```
Client → Network → Data Service → Storage
                        ↓
                  [Memory lookup]    ← fast (microseconds)
                        or
                  [Disk read]        ← slow (milliseconds)
```

The biggest single factor in KV latency is whether the document is in RAM (hot) or requires a disk read (cold / ejected). Everything else is secondary.

## Working set and ejection

When `mem_used` exceeds the high watermark (85% of bucket quota by default), Couchbase begins **ejecting** documents from RAM. Future reads of ejected documents require a disk fetch — `ep_bg_fetched` increments.

Ejection policies:
- **`valueOnly`** (default): only the value is ejected from RAM; the key and metadata stay. Disk read on miss, but key operations remain fast.
- **`fullEviction`**: both key and value ejected. Lower RAM footprint but key operations also require disk on miss. Use only when dataset is much larger than RAM and you're OK with higher read latency.

**Target:** keep `ep_cache_miss_rate` below 5% for latency-sensitive workloads. Above 5%, most latency problems trace back to working-set overflow.

## KV thread pools

The KV engine uses several thread pools. The key one for performance:

**Reader threads** (`ep_num_readers`): threads that fetch items from disk for background fetches.
**Writer threads** (`ep_num_writers`): threads that flush mutations to disk.

Default values are set automatically based on CPU count. You can tune via bucket settings:

```python
admin_bucket_update(
    name="orders",
    num_reader_threads="default",   # or a specific count like 8
    num_writer_threads="default",
    cluster="prod"
)
```

When to increase reader threads: `ep_bg_fetched` is high and disk utilization is below saturation — more threads can read in parallel. Don't increase beyond the storage subsystem's I/O parallelism (e.g., NVMe SSDs handle many parallel reads; spinning disks don't).

When to increase writer threads: `ep_diskqueue_drain` is consistently below `ep_diskqueue_fill` and disk write throughput is below the storage device's capacity.

## DCP performance (replication, XDCR, Eventing, Analytics)

DCP (Database Change Protocol) is the internal feed that powers XDCR, Analytics, Eventing, and replica replication. Each DCP consumer creates a backpressure-aware streaming connection.

Signs of DCP pressure:
- `ep_dcp_replica_items_remaining` > 0 and growing (internal replicas falling behind)
- XDCR `changes_left` growing
- Analytics `ingestion_status` shows pending operations
- Eventing `dcp_backlog` growing

Root causes and fixes:

| Symptom | Root cause | Fix |
|---|---|---|
| All DCP consumers behind after rebalance | Rebalance generates a burst of mutations | Normal — wait for it to drain |
| XDCR behind, disk I/O high | Disk I/O saturated by XDCR reads | Throttle XDCR with `networkUsageLimit`; use Magma |
| Eventing backlog growing without rebalance | Handler too slow | Increase `worker_count`; optimize handler |
| Analytics behind | Analytics nodes under-resourced | Add Analytics nodes or increase RAM quota |

## Subdocument vs full document

For updates that touch a small number of fields, `mutate_in` (subdocument operation) is faster than a full `replace`:
- Full `replace`: read-modify-write of the entire document JSON
- `mutate_in`: server-side field update, no client round-trip of the full document

For documents > 1 KB where you're changing < 10% of fields, `mutate_in` can be 2-5× faster and reduces network bandwidth. See `couchbase-app-integration/references/performance-patterns.md` for SDK code.

## Durability and write latency

Higher durability = higher write latency. The tradeoff:

| Durability level | When write acks | Latency | Survives |
|---|---|---|---|
| `NONE` | In-memory on active node | ~0.1ms | Nothing (async) |
| `MAJORITY` (default) | In-memory on majority of nodes | ~1-2ms | Node restart |
| `MAJORITY_AND_PERSIST_ACTIVE` | Persisted on active + in-memory majority | ~5ms | Active disk failure |
| `PERSIST_TO_MAJORITY` | Persisted on majority | ~10ms+ | Multi-node disk failure |

For write-heavy workloads where durability can be relaxed: use `NONE` or `MAJORITY`. Only use `PERSIST_TO_MAJORITY` where data loss is absolutely unacceptable (financial records, etc.) — the latency cost is significant.

## Bulk operations

Single-document operations have per-operation overhead (TCP round-trip, Couchbase internal processing). For bulk workloads:

- **Multi-get** (`get_multi` / `cb_get_multi`): batch up to 16 gets in one call. Significant throughput improvement for read-heavy bulk workloads.
- **Async operations**: SDK async patterns pipeline multiple operations without waiting for each ack. Most SDKs support async KV — use it for bulk loads.
- **Pipeline**: don't wait for each write to ack before sending the next. Send N operations, collect N responses. Most SDKs handle this automatically in async mode.

Target: 5,000-50,000 KV ops/sec per node is typical. Above 100,000 ops/sec per node, you're usually CPU-limited — add nodes.
