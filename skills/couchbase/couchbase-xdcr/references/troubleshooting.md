# XDCR troubleshooting

## Diagnostic workflow

1. **`admin_stats_xdcr`** — check `changes_left`, `docs_failed_cr_source`, and `rate_replication`.
2. **`admin_xdcr_list_replications`** — confirm replication status is `running`, not `paused` or `error`.
3. **`admin_eventing_list_functions`** — if lag coincides with high mutation rate, check if Eventing is also writing to the source (doubling the mutation load).
4. **Check destination cluster health** via `ping_cluster` and `get_cluster_details` on the destination.

## `changes_left` is non-zero and growing

XDCR can't keep up with the source mutation rate. Options:

- Increase `sourceNozzlePerNode` and `targetNozzlePerNode` (more parallel replication streams)
- Decrease the filter scope (filter out low-priority documents)
- Reduce source mutation rate if possible (batch writes less aggressively)
- Add Data nodes to source or destination to reduce per-node load
- Lower XDCR priority to `Medium` or `Low` during spikes, then raise back to `High` once the source catches up — this doesn't fix the root cause but protects foreground workload

## `changes_left` is stable but non-zero

This is normal during a backfill (initial replication after creating a replication or redeploying). A large `changes_left` at creation is expected — watch for the trend: if it's consistently decreasing, the backfill is progressing. Only act if it's stagnant or growing.

## `docs_failed_cr_source` is non-zero

The local cluster lost conflict resolution (the remote document was newer/higher-revision). This is expected and normal in active-active — it means the destination had a more recent version and that version won. It is not an error unless the count is growing rapidly relative to total writes, which would indicate very frequent conflicts and potential data loss.

In active-passive, any `docs_failed_cr_source` is unexpected and means something is writing to the destination when it shouldn't be.

## Replication status is `error`

Check the Couchbase UI (XDCR → replication row → hover for error message), or `admin_stats_xdcr` for error details.

Common errors:

| Error | Cause | Fix |
|---|---|---|
| `authentication failed` | Destination credentials in the reference are wrong or expired | `admin_xdcr_update_reference` with correct credentials |
| `bucket not found` | Destination bucket name is wrong, or bucket was deleted | Verify bucket name; recreate if deleted |
| `certificate expired` / `TLS handshake failed` | Destination cluster's TLS certificate has expired | Rotate certificate on destination; update reference |
| `connection refused` | Network connectivity between clusters broken | Check firewalls, VPC peering, security groups; verify port 8091 and 11210 are open |
| `too many connections` | XDCR nozzles × nodes exceeds destination's connection limit | Reduce `targetNozzlePerNode`; check destination node count |

## Replication not reaching all collections (7.1+)

If using `collectionsExplicitMapping: true`, each source collection must have an entry in `colMappingRules`. A missing collection is silently not replicated. Audit the mapping against the current collection list with `admin_scope_list` on both clusters.

## Replication lag spikes during rebalance

XDCR pauses and restarts replication streams during source or destination rebalance. `changes_left` spikes then recovers. This is normal — monitor that it recovers within minutes. If it doesn't recover after the rebalance completes, check node connectivity.

## Document count mismatch between source and destination

After a replication has been running for a while, `source_doc_count != destination_doc_count` is common and usually benign:

- TTL: documents expired on source but not yet on destination (or vice versa if TTL is different)
- Delete not replicated: `OnDelete` behavior depends on replication settings; if `replicateTo` is not `PERSIST_NONE`, deletes may not replicate
- Filter: some documents are intentionally filtered out

To verify replication fidelity for specific keys, use `cb_get` on the same key from both clusters and compare CAS values or field values.
