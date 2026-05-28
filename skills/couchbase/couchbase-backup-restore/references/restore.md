# Restore

## Full cluster restore

Restore all buckets from the most recent full backup + increments:

```bash
cbbackupmgr restore \
    --archive /backup/couchbase-archive \
    --repo prod-cluster \
    --cluster couchbase://10.0.0.1 \
    --username Administrator \
    --password <pass> \
    --start 2026-05-19T00:00:00Z \
    --end 2026-05-20T02:00:00Z
```

`--start` and `--end` define the backup window. cbbackupmgr applies the full backup within the window plus all increments up to `--end`. Omit both for "latest."

Key options:
- `--force-updates` — overwrite existing documents even if the destination has a newer CAS. Use when restoring over a live cluster (prevents the restore from losing to existing versions).
- `--no-indexes` — skip GSI/FTS index restore (useful if indexes already exist and you only need data)
- `--threads N` — parallel restore threads
- `--resume` — if a restore was interrupted, resume from where it stopped
- `--purge` — delete documents in the destination that aren't in the backup (restore to an exact state, not an additive merge)

## Restore a single bucket

```bash
cbbackupmgr restore \
    --archive /backup/couchbase-archive \
    --repo prod-cluster \
    --cluster couchbase://10.0.0.1 \
    --username Administrator \
    --password <pass> \
    --include-buckets orders
```

The `orders` bucket must already exist on the destination cluster. cbbackupmgr does not create buckets. If you're restoring to a new cluster, create the bucket first with matching settings (quota, replicas, storage backend).

## Restore to a different bucket name

```bash
cbbackupmgr restore \
    ... \
    --include-buckets orders \
    --map-buckets orders=orders-restored
```

Restores the `orders` backup into a bucket named `orders-restored` on the destination. Useful for restore-to-staging without overwriting production.

## Restore a single scope or collection (7.1+)

```bash
cbbackupmgr restore \
    ... \
    --include-buckets orders \
    --include-scopes my-scope \
    --include-collections my-scope.active-orders
```

## Point-in-time restore

Restore to the state as of a specific timestamp:

```bash
cbbackupmgr restore \
    --archive /backup/couchbase-archive \
    --repo prod-cluster \
    --cluster couchbase://10.0.0.1 \
    --username Administrator \
    --password <pass> \
    --end 2026-05-20T14:30:00Z
```

This applies the last full backup before `--end` plus all increments up to `--end`. All mutations after that timestamp are not included.

## Restoring specific documents

cbbackupmgr doesn't have a single-document restore mode. Options:

**Option 1: `cbbackupmgr examine` + manual extract.**
Examine the backup to find the document key and its state. If you just need the field values, examine output (with `--json`) gives you the document content. Manually re-apply via `cb_upsert`.

**Option 2: Restore to isolated bucket + copy.**
Restore the relevant time window to a separate bucket (e.g. `orders-recovery`), then use SQL++ to find the specific document(s) and copy them back.

```sql
-- Copy a specific document from the recovery bucket to production
INSERT INTO `orders`.`my-scope`.`active-orders` (KEY, VALUE)
    SELECT META(r).id, r FROM `orders-recovery`.`my-scope`.`active-orders` AS r
    WHERE META(r).id = "order::abc123";
```

## Restore checklist

Before restoring to production:

1. **Notify stakeholders** — restore to a live cluster will overwrite data. Coordinate with application owners.
2. **Enable maintenance mode / drain traffic** if overwriting live data.
3. **`cbbackupmgr examine`** the target backup first — verify it contains the expected documents and time range.
4. **Test restore on staging first** if restoring a schema-changing backup.
5. **Use `--map-buckets`** to restore to a non-production bucket if you need to verify before going live.
6. **Document count check** after restore: `SELECT COUNT(*) FROM keyspace` vs expected count from backup examine output.
7. **GSI index build** — after restore with `--include-indexes`, indexes are in `created` state, not `online`. Trigger a build: `admin_index_build_indexes`.
