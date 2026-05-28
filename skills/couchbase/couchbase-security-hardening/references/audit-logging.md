# Audit logging

## What Couchbase audit logging captures

Couchbase's audit subsystem logs security-relevant events to a structured JSON log file on each node. Events fall into categories:

| Category | Example events |
|---|---|
| Authentication | Login success/failure, logout |
| Authorization | Permission denied |
| Data access | Document reads, writes, deletes (selective) |
| Admin operations | User create/delete, role change, bucket create/delete |
| Query | SQL++ queries executed (optional, high volume) |
| Cluster | Node add/remove, rebalance, failover |
| Security | Certificate change, password policy change, TLS setting change |
| XDCR | Reference create/delete, replication create/delete |

## Enabling and configuring audit logging

```python
admin_cluster_update_audit_config(
    auditdEnabled=True,
    rotateInterval=86400,        # seconds between log rotations (86400 = daily)
    rotateSize=524288000,        # rotate when file exceeds this size (bytes, 500MB here)
    logPath="/var/lib/couchbase/logs",  # usually the default; don't change unless necessary
    disabledUsers=[],            # users whose actions are not audited (for high-frequency service accounts)
    cluster="prod"
)
```

**Important:** `disabledUsers` is a performance escape hatch, not a security control. Don't use it to hide activity. Use it only for very high-frequency service accounts (e.g. a monitoring agent doing thousands of reads/sec) where the audit volume would overwhelm the log pipeline.

## Recommended event set for compliance

For SOC2 / HIPAA / PCI-DSS, at minimum enable:

- `admin_loggedIn` / `admin_loggedOut` — all authentication events
- `createUser` / `deleteUser` / `setUserRoles` — RBAC changes
- `createBucket` / `deleteBucket` — bucket lifecycle
- `updateDocument` / `deleteDocument` on sensitive collections — data mutation audit
- `querySelectStatement` — optional; enable only if required by your compliance framework (high volume)
- `xdcrCreateReplication` / `xdcrDeleteReplication` — replication changes

Enable events via the Couchbase UI (Security → Audit → Event Filters) or via the audit config API.

## Audit log format

Each audit event is a JSON object on a single line:

```json
{
  "timestamp": "2026-05-28T03:14:15.926Z",
  "id": 8192,
  "name": "admin login",
  "description": "Successful login to admin console",
  "real_userid": {"source": "local", "user": "alice"},
  "sessionid": "abc123",
  "remote": {"ip": "10.0.0.5", "port": 52341},
  "local": {"ip": "10.0.0.1", "port": 8091}
}
```

For data mutation events, the document key is included but not the document value (to avoid capturing PII in audit logs).

## Log rotation and retention

Configure log rotation to prevent unbounded disk growth:

```
rotateInterval: 86400    # daily rotation
rotateSize: 524288000    # or when file exceeds 500MB (whichever comes first)
```

Retention policy: Couchbase doesn't manage retention — ship logs to an external system and apply retention there. Most compliance frameworks require 12 months minimum retention.

**On-disk risk:** audit logs on cluster nodes are a security risk if nodes are compromised — attackers can delete evidence. Ship to a SIEM as soon as possible and consider audit log file permissions (read-only for non-root once written).

## SIEM integration

Couchbase audit logs are written as JSON to the filesystem. Common shipping approaches:

**Filebeat + Elasticsearch/Splunk:**
```yaml
# filebeat.yml
filebeat.inputs:
  - type: log
    paths:
      - /var/lib/couchbase/logs/audit.log*
    json.keys_under_root: true
    json.overwrite_keys: true
    fields:
      source: couchbase-audit
      cluster: prod
```

**Fluentd / Fluent Bit:** similar — tail the file, parse JSON, forward to your pipeline.

**CloudWatch / Datadog:** use their respective log agents with the audit log path.

Ship from every Couchbase node (each node writes its own audit events). Use your SIEM's deduplication or a central aggregator if needed.

## Audit for Capella

Capella clusters have audit logging managed by Couchbase. The audit log content is the same; log access and shipping are handled via the Capella UI and Capella's log forwarding integrations (available in Enterprise tier). You do not have direct filesystem access to audit logs on Capella.
