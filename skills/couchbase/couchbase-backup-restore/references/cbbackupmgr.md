# cbbackupmgr

The primary CLI for self-managed backup and restore. Runs on any node with access to the cluster and to the backup storage location.

## Core commands

### config — create a backup repository

```bash
cbbackupmgr config \
    --archive /backup/couchbase-archive \
    --repo prod-cluster
```

A **repository** is a named logical backup set within an **archive** (the storage root). One cluster → one repository. Multiple repositories in an archive is fine (for different clusters or environments).

### backup — run a backup

```bash
# Full backup
cbbackupmgr backup \
    --archive /backup/couchbase-archive \
    --repo prod-cluster \
    --cluster couchbase://10.0.0.1 \
    --username Administrator \
    --password <pass> \
    --full-backup

# Incremental backup (omit --full-backup)
cbbackupmgr backup \
    --archive /backup/couchbase-archive \
    --repo prod-cluster \
    --cluster couchbase://10.0.0.1 \
    --username Administrator \
    --password <pass>
```

If the repository has no prior backup, the first backup is always a full regardless of `--full-backup`.

Key options:
- `--include-buckets bucket1,bucket2` — back up only specific buckets
- `--exclude-buckets bucket3` — exclude specific buckets
- `--include-indexes` — include GSI and FTS index definitions
- `--include-users` — include RBAC user definitions
- `--threads N` — parallel backup threads (default 1; increase for speed at the cost of cluster I/O)
- `--no-progress-bar` — cleaner for automated scripts

### list — inspect backups in a repository

```bash
cbbackupmgr list \
    --archive /backup/couchbase-archive \
    --repo prod-cluster
```

Shows all backup runs with timestamps, sizes, and document counts. Use to select a target for restore.

### examine — inspect contents without restoring

```bash
cbbackupmgr examine \
    --archive /backup/couchbase-archive \
    --repo prod-cluster \
    --backup 2026-05-20T02:00:00Z \
    --bucket orders
```

Lists document keys, metadata, and document counts. Verifies backup integrity without requiring a running cluster.

### merge — consolidate backups

```bash
cbbackupmgr merge \
    --archive /backup/couchbase-archive \
    --repo prod-cluster \
    --start 2026-01-01T00:00:00Z \
    --end 2026-05-01T00:00:00Z
```

Produces a new full backup covering the date range. After confirming the merge succeeded, delete the original full + increments to reclaim storage.

### info — repository metadata

```bash
cbbackupmgr info \
    --archive /backup/couchbase-archive \
    --repo prod-cluster \
    --json
```

Returns detailed repository metadata. `--json` for machine-parseable output.

## Backup service (7.0+ EE)

The Couchbase Backup Service (separate from cbbackupmgr CLI) is a managed service embedded in the cluster. It exposes the `admin_backup_*` REST/MCP tools and provides:

- Scheduled backups via plan configuration
- Backup history via the REST API and UI
- Trigger on-demand backups without a dedicated backup node

Use the Backup Service via MCP tools (`admin_backup_trigger_backup`, etc.) when running backups as part of automated workflows. Use cbbackupmgr CLI for restore (the Backup Service's restore capabilities are more limited than the CLI's granular options).

## Authentication

cbbackupmgr requires a Couchbase user with `backup_admin` or `cluster_admin` role. For restore, `restore_admin` in addition to read/write on the target bucket.

Never use `Administrator` / root credentials for automated scripts. Create a dedicated `cbbackupmgr-agent` user with only the required roles.
