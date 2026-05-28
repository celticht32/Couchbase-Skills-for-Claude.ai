# Compaction

## What compaction does

Couchbase stores mutations as append-only writes to data files (couchstore) or LSM tree levels (Magma). Over time, deleted and superseded document versions accumulate as dead space. Compaction reclaims this space.

**Couchstore compaction:** explicit compaction cycles run per-vBucket. During compaction, Couchbase writes a new compacted copy of the vBucket file. I/O doubles temporarily (reading old, writing new). After compaction completes, the old file is deleted.

**Magma compaction:** continuous background compaction via LSM tree merging. No explicit cycles — the engine continuously merges levels. I/O load is spread over time rather than spiking.

## Compaction impact on performance

During couchstore compaction:
- Disk I/O increases 1.5-2× (reads + writes for the compaction pass)
- Disk space temporarily increases (old + new file coexist during compaction)
- CPU load increases moderately on the Data service nodes

In extreme cases (many large vBuckets compacting simultaneously), compaction can saturate disk I/O and cause:
- KV write latency increase (disk queue backs up)
- Query latency increase (Index node also competing for disk)
- XDCR lag (replication reads competing with compaction writes)

## Autocompaction settings

Configure via bucket settings or cluster-wide defaults:

```python
admin_bucket_update(
    name="orders",
    auto_compaction_defined=True,

    # Trigger compaction when fragmentation exceeds this %
    database_fragmentation_threshold_percentage=30,
    view_fragmentation_threshold_percentage=30,

    # OR when the file exceeds this size (whichever comes first)
    database_fragmentation_threshold_size=1073741824,  # 1 GB

    # Time window — only compact during this window (reduces daytime impact)
    compaction_period_from="01:00",
    compaction_period_to="05:00",

    # Allow compaction to abort if the window closes
    abort_outside_compaction_period=True,

    # Parallel compaction (more throughput, higher I/O pressure)
    parallel_db_and_view_compaction=False,

    cluster="prod"
)
```

## Tuning compaction for production

**Fragmentation threshold (30% is the default):**
- Below 20%: compaction runs too frequently, wastes I/O
- 25-35%: reasonable for most workloads
- Above 50%: disk space waste grows; reads may slow down due to fragmented files

**Time window compaction:**
For latency-sensitive applications, confine compaction to low-traffic hours. Set `compaction_period_from/to` to your lowest-traffic window (typically 01:00-05:00 local time). Set `abort_outside_compaction_period=True` so compaction stops if it runs long.

**Parallel compaction:**
`parallel_db_and_view_compaction=True` runs data and view compaction simultaneously. Faster but doubles the I/O pressure. Disable on I/O-constrained nodes; enable on fast NVMe storage.

**Manual trigger (emergency):**
If fragmentation is very high and you need to compact immediately:
```python
admin_bucket_compact(bucket="orders", cluster="prod")
```
This triggers compaction outside the scheduled window. Use after bulk deletes or large batch updates that create high fragmentation suddenly.

## Magma and compaction

Magma handles compaction continuously in the background — there are no explicit compaction cycles to tune. However:

- Magma has a **write amplification** factor: each write may trigger background compaction work. On write-heavy workloads, this shows up as sustained moderate disk I/O rather than periodic spikes.
- If Magma's background compaction falls behind (very high write rate), `ep_diskqueue_drain` will lag `ep_diskqueue_fill`. Fix: faster storage or reduce write rate.
- Magma doesn't respond to `admin_bucket_compact` the same way couchstore does — calling it on a Magma bucket triggers a forced compaction pass but it's usually unnecessary.

## Disk space planning for compaction

For couchstore buckets, plan for 2× the actual data size on disk to accommodate:
- The actual data files
- The new compacted copy during compaction (temporary)
- Log files

For Magma, plan for 1.5× the actual data size (LSM write amplification but no temporary doubling).

Fragmentation headroom: keep at least 20% free disk space on data nodes at all times. Running out of disk space during compaction (when the temporary file exists) causes compaction to fail and data to become inaccessible.
