# Backup strategy

## Backup types

**Full backup:** a complete snapshot of all data in the scope of the backup. Slowest and largest, but self-contained — restore from a full backup requires nothing else.

**Incremental (differential):** captures only mutations since the last backup (full or incremental). Fast and small, but restore requires the full backup plus all increments in order. Couchbase calls these "incremental" but they are technically differential (each increment is against the prior backup, not against the full).

**Merge:** combines a full backup and increments into a new full backup. Reduces the number of files needed for restore while reclaiming storage from older increments.

## Recommended schedule

For most production workloads:

| Frequency | Type | Retention |
|---|---|---|
| Weekly (Sunday 02:00) | Full | Keep 4 weeks |
| Every 6 hours | Incremental | Keep 2 weeks |

This gives a maximum RPO of 6 hours and keeps restore complexity low (1 full + ≤ 28 increments).

For higher-value data:
- Hourly incremental, daily full, 4-week retention
- Pre-upgrade full backup (keep indefinitely until the upgrade is confirmed stable)

## What to back up

Couchbase backup scopes:

- **Cluster-level:** includes all buckets plus cluster configuration (RBAC, settings, indexes)
- **Bucket-level:** one bucket's data (documents) — does not include GSI/FTS index definitions unless you include `--include-indexes`
- **Scope/collection level:** (cbbackupmgr 7.1+) subset of a bucket

Always include index definitions in your backup (`--include-indexes` in cbbackupmgr) unless you have your index DDL in version control and can recreate them reliably.

## Backup to S3 (and S3-compatible storage)

cbbackupmgr supports writing directly to S3:

```bash
cbbackupmgr config \
    --archive s3://my-backup-bucket/couchbase-backups \
    --repo prod-cluster \
    --s3-region us-east-1 \
    --s3-key-id AKIAIOSFODNN7EXAMPLE \
    --s3-key-secret wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

For MinIO or other S3-compatible endpoints, add `--s3-endpoint https://minio.example.com`.

**Lifecycle policies on the S3 bucket:** configure S3 object lifecycle to automatically expire old backups. Set this on the S3 side — cbbackupmgr doesn't manage S3 object expiry.

**Important:** Backup to S3 transfers data over the network. Ensure the backup node has adequate egress bandwidth. For large clusters, run backups from a dedicated backup node or a node with low data traffic.

## Retention management

Keeping too many backups wastes storage. Keeping too few risks being unable to restore to a needed point in time.

`cbbackupmgr merge` consolidates a full + increments into a new full, allowing old files to be deleted:

```bash
# Merge everything older than 7 days into a single full
cbbackupmgr merge --archive /backups --repo prod-cluster --start 2026-01-01 --end 2026-05-20
```

Automate this in your backup script: after each new full backup, merge-and-delete the prior full + its increments once they exceed retention.

## Backup verification

Never assume backups are valid without testing them. At minimum:

1. **`cbbackupmgr examine`** — lists backup contents, verifies metadata integrity, reports document counts. Does not require a running Couchbase cluster. Run this after every full backup.
2. **Restore to staging** — quarterly (or after any major schema/data change), restore the most recent full backup to a staging cluster and run your application's test suite against it.
3. **Document count parity check** — after restore, compare `SELECT COUNT(*) FROM keyspace` on production vs staging for key collections.
