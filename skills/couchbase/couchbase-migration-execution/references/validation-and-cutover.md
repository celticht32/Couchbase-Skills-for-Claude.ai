# Validation and cutover

**Contents:** [Validation philosophy](#validation-philosophy) · [Count parity](#count-parity) · [Sample equivalence](#sample-equivalence) · [Aggregate parity](#aggregate-parity) · [Application-level testing](#application-level-testing) · [Continuous diff during dual-write](#continuous-diff-during-dual-write) · [The cutover](#the-cutover) · [Cutover monitoring](#cutover-monitoring) · [Rollback plan](#rollback-plan) · [Post-cutover validation](#post-cutover-validation) · [Decommissioning source](#decommissioning-source) · [Quick decision tree](#quick-decision-tree)

The two riskiest phases of a migration. Confirming source = target, and switching production traffic. Done well, these are anticlimactic. Done poorly, they're how migrations cause outages.

## Validation philosophy

There's no single "validate the migration" check. Real validation is layered:

1. **Count parity** — same number of items per table/collection
2. **Sample-level equivalence** — pick random records, compare field by field
3. **Aggregate parity** — running known aggregations (sums, counts, averages) on both and comparing
4. **Application-level test** — running the actual app against target, comparing behavior
5. **Continuous diff during dual-write** — comparing source and target in real time as writes happen

Each layer catches different failures:
- Count parity catches "we missed a whole table"
- Sample equivalence catches "transformation is dropping fields"
- Aggregate parity catches "we have all the records but some values are wrong"
- Application-level catches "queries return different results"
- Continuous diff catches drift over time

A robust migration validates at every layer.

## Count parity

The cheapest, simplest check. Just compare counts.

```python
# Source
source_count = source_conn.execute("SELECT count(*) FROM users").fetchone()[0]

# Target
target_count = couchbase.cluster.query(
    "SELECT count(*) AS c FROM users"
).rows()[0]["c"]

assert source_count == target_count, f"Source={source_count}, Target={target_count}"
```

**Catches:** missing records (whole batches), entire collections not migrated.

**Misses:** content errors. Two records with the same key but wrong content still count the same.

For partitioned tables, run count per partition / per shard, not just total. Pinpoints where any drift is.

## Sample equivalence

Pick N random records; fetch from both; compare.

```python
import random

def validate_sample(sample_size=1000):
    # Get N random IDs from source
    source_ids = source.execute(
        f"SELECT id FROM users ORDER BY random() LIMIT {sample_size}"
    ).fetchall()
    
    mismatches = []
    for (sid,) in source_ids:
        source_row = source.execute("SELECT * FROM users WHERE id = %s", sid).fetchone()
        try:
            target_doc = couchbase.collection.get(f"user::{sid}").content_as[dict]
        except DocumentNotFoundException:
            mismatches.append({"id": sid, "type": "missing_in_target"})
            continue
        
        source_doc = transform_row_to_doc(source_row)  # same transformation used in migration
        if source_doc != target_doc:
            mismatches.append({"id": sid, "type": "content_mismatch",
                                "diff": diff(source_doc, target_doc)})
    return mismatches
```

**Sample size:**
- 100 — quick sanity check
- 1,000 — reasonable confidence for small / medium datasets
- 10,000 — high confidence
- 100,000 — exhaustive for big migrations

For < 1M records, sample 10% or all. For larger: sample 10,000-100,000 random IDs.

**`diff` function:** compare maps recursively, optionally ignoring fields you expect to differ (migration timestamps, system fields). Return a list of paths that differ.

## Aggregate parity

Run statistical queries on both sides and compare.

```python
# Source
source_aggregates = source.execute("""
    SELECT
        COUNT(*) AS total,
        SUM(balance) AS total_balance,
        AVG(balance) AS avg_balance,
        MIN(created_at) AS earliest,
        MAX(created_at) AS latest
    FROM accounts
""").fetchone()

# Target
target_aggregates = couchbase.cluster.query("""
    SELECT
        COUNT(*) AS total,
        SUM(balance) AS total_balance,
        AVG(balance) AS avg_balance,
        MIN(created_at) AS earliest,
        MAX(created_at) AS latest
    FROM accounts
""").rows()[0]

assert source_aggregates == target_aggregates
```

**Catches:** "we have all the rows but the balances are wrong somewhere." If sums differ by even 1, something is off.

**Per-segment aggregates** are even better:

```sql
SELECT tier, COUNT(*), SUM(balance) FROM accounts GROUP BY tier
```

Comparing per-segment isolates which segment has the drift.

## Application-level testing

The most realistic validation: run actual application code against the target and compare its behavior to running against the source.

**Pattern: shadow read mode**

```python
def get_user(user_id):
    # Primary: read from source
    source_user = source.users.find_one({"_id": user_id})
    
    # Shadow: also read from target; compare; log differences
    if SHADOW_READS_ENABLED:
        try:
            target_user = couchbase.users.get(f"user::{user_id}").content_as[dict]
            if not users_equivalent(source_user, target_user):
                log.warning(f"Shadow read mismatch: {user_id}",
                             extra={"source": source_user, "target": target_user})
        except Exception as e:
            log.warning(f"Shadow read failed: {user_id}: {e}")
    
    return source_user
```

Production traffic exercises both systems; you see real-world drift. Don't surface target errors to users; just log.

**Pattern: replay test suite**

If your app has a comprehensive integration test suite, run it twice — once against source, once against target. Diff the responses.

## Continuous diff during dual-write

While dual-write is active, run a continuous comparator:

```python
def continuous_diff(sample_rate=0.01):
    # For sample_rate of all writes, also read back from both and compare
    while True:
        write = next_pending_write_from_log()
        if random.random() < sample_rate:
            source_state = source.read(write.key)
            target_state = couchbase.read(write.key)
            if not equivalent(source_state, target_state):
                alert.send("Dual-write divergence", details={...})
        time.sleep(0.1)
```

A 1% sample rate catches systematic drift while staying lightweight.

## What "equivalent" means

The hard question. Two documents may differ in:

- Timestamp formats (ISO 8601 vs Unix epoch)
- Numeric precision (float vs decimal)
- Field ordering (irrelevant in JSON)
- Missing fields vs null fields (`{}`  vs `{x: null}`)
- Whitespace in string values
- System-added fields (migration timestamps, internal versions)

The migration-defined `equivalent()` function should:
- Normalize both sides to a canonical form
- Ignore fields you expect to differ
- Compare what should be the same

Get this function right — it's the heart of validation.

## When to declare "migration valid"

Tier the confidence:

| Confidence level | Sufficient for |
|---|---|
| Count parity OK | "We migrated everything" |
| Count + small sample equivalence | "Migration is roughly correct" |
| Count + 10K sample + aggregate parity | Staging cutover |
| All above + shadow reads for a week | Production cutover |
| All above + soak period post-cutover | Decommission source |

Don't promote to higher confidence without doing the underlying validation. "We tested 100 records and it was fine" is not "the migration is correct."

## The cutover

Cutover = switching application traffic from source to target. The mechanics depend on how the application is wired:

### Pattern 1: configuration switch

The app reads its data store endpoint from config. Change the config; restart (or reload). Simple but blunt.

```python
# Before
DB_URL = "postgresql://pg-host:5432/mydb"

# After
DB_URL = "couchbase://cb.example.com"
```

Requires app code to support both DB types via abstraction (or a complete rewrite of data-access code).

### Pattern 2: feature flag

The app's data-access layer reads from one or the other based on a runtime flag. Flip the flag (no restart needed).

```python
def get_user(user_id):
    if feature_flag("use_couchbase"):
        return couchbase.users.get(f"user::{user_id}").content_as[dict]
    else:
        return postgres.execute("SELECT * FROM users WHERE id = %s", user_id).fetchone()
```

Better. The flag can be flipped instantly or gradually (10% of traffic to Couchbase, then 50%, then 100%).

### Pattern 3: percentage rollout

Gradually shift traffic:

```python
def get_user(user_id):
    pct = couchbase_traffic_percentage()  # 0-100
    if hash(user_id) % 100 < pct:
        return couchbase.users.get(f"user::{user_id}").content_as[dict]
    else:
        return postgres.execute("SELECT * FROM users WHERE id = %s", user_id).fetchone()
```

Start at 1%, monitor, increase to 10%, monitor, increase to 50%, 100%. Each step is reversible.

### Pattern 4: per-endpoint cutover

For phased migrations, cut over one endpoint at a time:
- `/users/profile` → Couchbase
- `/users/orders` → still source
- `/admin/...` → still source

Lower-stakes endpoints first; high-stakes endpoints last when confidence is highest.

## Cutover monitoring

During and immediately after cutover, watch:

| Metric | Why |
|---|---|
| Application error rate | First sign of "the new path doesn't work" |
| Latency p50 / p95 / p99 | Couchbase typically faster, but transformation overhead could be slower |
| Couchbase cluster health (`admin_stats_overview`) | Sudden load is the cluster's first stress test |
| Source DB load | Source should now be quiet (or only have dual-writes); if it's still heavy, traffic didn't actually switch |
| User-reported errors / support tickets | The ultimate check |

Set alerts on application error rate spiking; you want to know within minutes if the cutover broke something.

## Rollback plan

Before cutover, document the rollback plan:

**For feature-flag-based cutover:** flip the flag back. Should take seconds.

**For config-switch cutover:** change config back, restart. Takes minutes.

**For percentage rollout:** drop to 0% Couchbase traffic. Seconds.

**Soak period:** keep source running, taking dual-writes from the application, for AT LEAST a week post-cutover. Ideally 2-4 weeks. During soak, rollback is straightforward.

**After soak:** rollback is much harder. Data written to target-only mode isn't in source. You'd need a reverse-direction migration to roll back.

**Decommission source:** only after soak with no incidents. Even then, take a final backup of source before decommissioning, in case of late-discovered issues.

## Common cutover pitfalls

- **No rollback plan documented:** "we'll figure it out if something goes wrong" — works until it doesn't
- **Cutover during peak hours:** schedule for low-traffic windows
- **Cutover with no monitoring set up:** can't tell if it's working
- **Cutover without prior dry-run:** the staging cutover should mirror production exactly
- **Decommissioning source too quickly:** lose your rollback option
- **Application changes deployed with cutover:** if something breaks, you don't know if it's the code change or the data store change. Deploy app changes separately
- **All traffic at once:** gradual is safer

## Post-cutover validation

For 24-48 hours after cutover:

- Sample-equivalence check between source and target (still both up; source receives writes via the dual-write or CDC pipeline)
- Application error rate monitoring
- Manual spot-checks on key user-facing flows
- Customer support volume — sudden spike means real users hit a bug

If issues emerge: roll back. The first hour is when rollback is cheapest.

## Decommissioning source

After the soak period with no incidents:

1. Stop dual-write or CDC (target is now standalone)
2. Take final backup of source (kept for ~6 months as insurance)
3. Put source in read-only mode for a week (catches anything still relying on it)
4. Shut down source
5. Communicate decommissioning to the team / org
6. Eventually delete the backup once confidence is permanent

Don't skip step 3 — there's almost always *something* still pointing at the source (monitoring scripts, batch jobs, reports). Read-only mode catches them without breaking them.

## Quick decision tree

- **Sizing validation effort?** → Count parity is free; sample equivalence is hours; full audit is days. Match to migration size
- **Sample size?** → 10K random IDs is high confidence for most migrations; 100K for huge ones
- **Cutover mechanism?** → Feature flag (best); config switch (worst, blunt); percentage rollout (incrementally safer)
- **Cutover timing?** → Low-traffic window; never during a holiday weekend or product launch
- **How long is the soak period?** → 2-4 weeks minimum post-cutover before decommissioning source
- **When can you decommission source?** → After 4+ weeks soak with no incidents AND final backup
- **Discovered drift mid-validation?** → Stop. Investigate. Fix the transformation. Re-migrate the affected slice. Re-validate
