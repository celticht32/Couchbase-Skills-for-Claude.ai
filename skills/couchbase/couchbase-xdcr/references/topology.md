# XDCR topology

## Active-passive (unidirectional)

One cluster is the primary writer. The secondary cluster receives replicated data and serves reads.

```
Cluster A (primary) ──XDCR──► Cluster B (DR / read replica)
```

Writes always go to A. B is eventually consistent behind A by the replication lag.

**When to use:**
- Disaster recovery (B is the failover target)
- Read offload to a secondary region (reads go to B; writes to A)
- Analytics / reporting workload isolation

**Conflict risk:** None in steady state. The only write path is A → B. Conflict only possible during a DR failover scenario where A is declared dead and B starts accepting writes, then A recovers.

## Active-active (bidirectional)

Both clusters accept writes. Each replicates to the other.

```
Cluster A ◄──XDCR──► Cluster B
```

Documents written on A replicate to B; documents written on B replicate to A. If the same document key is written on both sides before replication catches up, a conflict occurs.

**When to use:**
- Low-latency writes from multiple geographic regions (users in EU write to EU cluster; users in US write to US cluster)
- No single point of write failure

**Conflict risk:** Always present. Mitigate by:
- Partitioning writes by key prefix or collection (EU users → `eu::*` keys, US users → `us::*` keys) — eliminates conflict by design
- Using custom conflict resolution (8.x) for data types where LWW isn't acceptable
- Accepting LWW for data where the last writer wins is correct business logic (e.g. session data, presence, non-critical preferences)

**Never use active-active for:**
- Counter/increment operations (both sides incrementing the same counter will conflict and one increment will be lost)
- Transactional workflows where cross-region consistency is required

## Hub-and-spoke

One central cluster replicates to N regional clusters. Regions replicate back to hub.

```
            ◄──► Cluster EU
Hub Cluster ◄──► Cluster APAC
            ◄──► Cluster US
```

Hub is the authoritative store; regional clusters are read-local caches with local-write capability. Useful for global reference data (product catalog) distributed out from hub to regions.

## Cascade replication

A → B → C. Rarely used. Has latency multiplication and no loops protection. Prefer hub-and-spoke over cascades.

## Topology decision checklist

| Requirement | Topology |
|---|---|
| DR only, writes always from one region | Active-passive |
| Multi-region reads with central writes | Active-passive (or active-passive + read routing) |
| Multi-region writes, conflict acceptable or partitioned by key | Active-active |
| Multi-region writes, conflict NOT acceptable | Reconsider — active-active can't guarantee this; consider partitioned active-passive |
| Global reference data + regional writes | Hub-and-spoke |

## Collection-aware replication (7.1+)

By default, XDCR replicates all collections in a bucket. In 7.1+, you can specify collection mappings in the replication config:

```json
"collectionsMigrationMode": false,
"collectionsExplicitMapping": true,
"colMappingRules": {
  "scope1.col_a": "scope1.col_a",
  "scope1.col_b": "scope2.col_b_replica"
}
```

Use explicit mapping to:
- Replicate only specific collections (reduce bandwidth)
- Replicate to a differently-named collection on the destination
- Exclude internal or temporary collections from replication

## Network and bandwidth planning

XDCR bandwidth estimate:

```
bandwidth_Mbps ≈ avg_doc_size_KB × mutations_per_sec × 8 / 1000
```

Add ~20% overhead for XDCR protocol framing and metadata. In active-active, double it (both directions).

For detailed sizing, see `couchbase-sizing/references/network.md`.
