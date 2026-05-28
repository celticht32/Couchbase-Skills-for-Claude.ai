# Capella managed backups

## How Capella handles backups

Capella automatically manages backups for every cluster. The defaults:

- **Full backup:** weekly
- **Incremental backup:** daily
- **Retention:** 30 days

Backups are stored in Couchbase-managed cloud storage (the same region as your cluster). You don't configure the storage location.

## Viewing and managing backups (Analytics MCP)

Via the `cb-analytics-capella` skill's tools:

```python
# List backups for a cluster
capella_list_backups(
    org_id="...",
    project_id="...",
    cluster_id="..."
)

# Trigger an immediate backup
capella_create_backup(
    org_id="...",
    project_id="...",
    cluster_id="..."
)

# Restore from a backup
capella_restore_backup(
    org_id="...",
    project_id="...",
    cluster_id="...",
    backup_id="...",
    target_cluster_id=None  # None = restore in place; set to restore to different cluster
)
```

## Customizing backup schedule

Backup schedule and retention are configured in the Capella UI (Clusters → Backup). The Capella v4 API exposes backup schedule management but the MCP tools don't currently wrap it — use the UI or direct API calls for schedule changes.

## Restore options

**In-place restore:** overwrites the cluster with data from the selected backup. All current data after the backup point is lost. Requires cluster to be healthy.

**Cross-cluster restore:** restores backup data into a different Capella cluster (same organization). Set `target_cluster_id` in `capella_restore_backup`. Useful for:
- Restore-to-staging for verification
- DR to a pre-provisioned standby cluster
- Cloning production to a new cluster

## Limitations vs self-managed cbbackupmgr

| Feature | Capella managed | cbbackupmgr |
|---|---|---|
| Backup to your own S3 bucket | No | Yes |
| Granular point-in-time restore | No (daily granularity) | Yes (to the minute) |
| Single-bucket restore | Via UI | Yes |
| Single-document restore | No | Workaround via examine + manual |
| Custom backup schedule | UI only | Full cron control |
| Backup access for audit/examination | Via API listing only | `cbbackupmgr examine` |
| Retention beyond 30 days | Configurable in UI (paid tiers) | Unlimited |

## Before a planned maintenance event

Always trigger a manual backup before:
- Cluster upgrades
- Significant data model changes
- Bulk deletes or schema migrations

`capella_create_backup` triggers an immediate backup. Wait for completion (poll `capella_list_backups` for a new backup entry with `status: complete`) before proceeding with the maintenance action.
