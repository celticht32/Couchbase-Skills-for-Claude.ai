# XDCR conflict resolution

## What is a conflict?

A conflict occurs in active-active (bidirectional) XDCR when the same document key is written on both clusters before those writes are replicated to each other. When replication delivers the remote write, the local and remote versions of the document have diverged, and one must win.

In active-passive (unidirectional) XDCR, conflicts only arise if the destination cluster is written to directly (e.g. during a DR failover scenario).

## Default resolution: Last-Write-Wins (LWW)

In Couchbase 7.x, the default conflict resolution mode is **sequence number** (revision number) based, not timestamp. The version with the higher sequence number wins. This is deterministic but doesn't reflect wall-clock time.

For **timestamp-based LWW** (the write with the more recent `hlc` timestamp wins), set `conflictResolutionType: "lww"` on both source and destination buckets **at creation time**. This requires synchronized clocks across clusters (NTP, < 500ms skew recommended).

**Important:** conflict resolution mode is set at the bucket level and cannot be changed after creation. Plan before you create the bucket.

## 8.x: Custom conflict resolution

Couchbase 8.0 adds custom conflict resolution via a JavaScript function. The function receives both document versions and returns the winner:

```javascript
function resolveConflict(key, localDoc, remoteDoc, localMeta, remoteMeta) {
    // Return "local" to keep the local version, "remote" to keep the remote version.
    // Business logic example: keep the version with the higher bid amount.
    if (localDoc.bid_amount >= remoteDoc.bid_amount) {
        return "local";
    }
    return "remote";
}
```

Configure via `admin_xdcr_create_replication` with `conflictResolutionFunction: "<function code>"`.

Custom conflict resolution is evaluated on the destination cluster. The function runs synchronously per-conflict — keep it fast (microseconds, not milliseconds).

## 8.x: Conflict logging

Instead of (or in addition to) automatic resolution, Couchbase 8.x can log conflicts to a dedicated collection so they can be audited or manually resolved:

```python
admin_xdcr_get_conflict_log(
    bucket="orders",
    scope="my-scope",
    collection="orders",
    cluster="prod"
)
```

Conflict log entries contain both versions of the document plus metadata. Use this to:
- Audit data loss in active-active deployments
- Build a manual conflict resolution workflow for business-critical data
- Alert on conflict rate anomalies

To enable conflict logging, set `enableCrossClusterVersioning: true` on the replication.

## Minimizing conflicts

**Strategy 1: Key partitioning.**
Assign each region ownership of a key prefix. EU users write `eu::user::<id>`, US users write `us::user::<id>`. No conflicts because the key spaces don't overlap.

**Strategy 2: Collection partitioning.**
Replicate collections in one direction only, even in an otherwise active-active topology. The EU cluster owns the `eu-orders` collection; the US cluster owns `us-orders`. Cross-region data access goes through application-layer routing, not conflicting writes.

**Strategy 3: Idempotent, commutative writes.**
Design document updates to be safe under LWW. If you're tracking event counts with a field like `event_count: 5`, any version being the "winner" is equally wrong. Use design patterns that are commutative: sets (not counters), append-only logs (not in-place updates), or use Couchbase's sub-document increment operation which is serialized at the Data service level (but only on one cluster).

**Strategy 4: Read-your-own-writes to the local cluster.**
If clients only read from their local cluster's replicated data, they see eventual consistency but no read-your-own-writes violations. Application code in `couchbase-app-integration/references/xdcr-app-aware.md` covers the patterns.
