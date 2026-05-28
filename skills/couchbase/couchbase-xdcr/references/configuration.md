# XDCR configuration

## Setup sequence

1. Create a **remote cluster reference** — register the destination cluster's address and credentials.
2. Create a **replication** — define the source bucket, destination bucket, and replication settings.
3. Monitor stats to confirm replication is running and `changes_left` is draining to 0.

## Creating a remote cluster reference

```python
admin_xdcr_create_reference(
    name="cluster-b-prod",
    hostname="10.0.1.10",         # IP or hostname of any node in the destination cluster
    username="xdcr-user",
    password="<generated>",
    secure="full",                # "none" | "half" | "full"
    certificate="-----BEGIN CERTIFICATE-----\n...",  # required for "full" encryption
    cluster="cluster-a"
)
```

**Encryption levels:**
- `none` — plaintext. Do not use in production.
- `half` — password encrypted, data in plaintext. Legacy; avoid.
- `full` — TLS for both auth and data. Required for cross-cloud/internet replication.

The `hostname` can be any node in the destination cluster — XDCR auto-discovers the rest.

## Creating a replication

```python
admin_xdcr_create_replication(
    from_bucket="orders",
    to_bucket="orders",
    to_cluster="cluster-b-prod",
    replication_mode="xmem",       # always "xmem" for modern Couchbase
    filter_expression="META().id LIKE 'order::%'",  # optional
    priority="High",               # "Low" | "Medium" | "High"
    cluster="cluster-a"
)
```

## Filter expressions

Filtering reduces what gets replicated. The filter expression is evaluated against each document mutation. Only mutations where the expression evaluates to true are replicated.

Filter by key pattern:
```
META().id LIKE 'eu::%'
```

Filter by document field:
```
`region` = "EU"
```

Filter to specific collection (7.1+, use collection mappings instead for precision):
```
META().id NOT LIKE 'internal::%'
```

**Important:** changing a filter on a running replication pauses and restarts replication from the DCP checkpoint. Documents already replicated are not un-replicated on the destination — it only affects new mutations going forward. If the filter change would cause documents to *stop* being replicated that previously were, those documents remain on the destination until manually removed.

## Replication settings

Key settings configurable via `admin_xdcr_update_replication`:

| Setting | Default | Notes |
|---|---|---|
| `workerBatchSize` | 500 | Mutations per batch. Increase for high-throughput, decrease for lower latency. |
| `docBatchSizeKb` | 2048 | Total KB per batch. |
| `failureRestartInterval` | 10 | Seconds before retrying after a failure. |
| `optimisticReplicationThreshold` | 256 | Documents smaller than this (bytes) are replicated without pre-fetching destination CAS. Speeds up replication of small documents. |
| `sourceNozzlePerNode` | 2 | DCP nozzles per source node. More = higher throughput at the cost of source CPU. |
| `targetNozzlePerNode` | 2 | Sending threads per target node. |
| `priority` | `High` | Throttles XDCR's CPU claim relative to other services. Use `Medium` or `Low` to protect foreground workload during replication spikes. |
| `networkUsageLimit` | 0 (unlimited) | Cap bandwidth in MB/s. |

## Monitoring a running replication

From `admin_stats_xdcr`:

- **`changes_left`** — mutations not yet replicated. Should be near 0 in steady state. A consistently non-zero and growing value indicates the replication can't keep up.
- **`docs_written`** — cumulative mutations replicated successfully.
- **`docs_failed_cr_source`** — mutations where this cluster lost the conflict resolution (the destination's version was newer). Normal in active-active; unexpected in active-passive.
- **`data_replicated`** — bytes replicated.
- **`bandwidth_usage`** — current replication bandwidth.
- **`rate_replication`** — mutations/sec being replicated.
