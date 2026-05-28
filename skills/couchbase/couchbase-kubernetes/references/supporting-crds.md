# Supporting CRDs

## CouchbaseBucket

```yaml
apiVersion: couchbase.com/v2
kind: CouchbaseBucket
metadata:
  name: orders
  namespace: couchbase-prod
spec:
  memoryQuota: 4Gi
  replicas: 1
  ioPriority: high          # high or low
  evictionPolicy: valueOnly  # valueOnly (default) or fullEviction
  conflictResolution: lww    # lww (last-write-wins) or seqno
  enableFlush: false         # set true only for dev — disables flush in prod
  compressionMode: passive   # off, passive, or active
  maxTTL: 0                  # 0 = no TTL enforcement at bucket level
```

Reference from `CouchbaseCluster`:
```yaml
spec:
  buckets:
    managed: true    # operator creates/updates buckets from CouchbaseBucket CRDs
```

## CouchbaseUser

```yaml
apiVersion: couchbase.com/v2
kind: CouchbaseUser
metadata:
  name: orders-service
  namespace: couchbase-prod
spec:
  authSecret: orders-service-credentials   # k8s secret with "password" key
  authDomain: local
  roles:
    - name: data_reader
      bucket: orders
    - name: data_writer
      bucket: orders
    - name: query_select
      bucket: orders
    - name: query_insert
      bucket: orders
    - name: query_update
      bucket: orders
```

Create the credentials secret:
```bash
kubectl create secret generic orders-service-credentials \
  --namespace couchbase-prod \
  --from-literal=password="$(openssl rand -base64 32)"
```

## CouchbaseGroup

```yaml
apiVersion: couchbase.com/v2
kind: CouchbaseGroup
metadata:
  name: analytics-readers
  namespace: couchbase-prod
spec:
  roles:
    - name: analytics_reader
      bucket: "*"
    - name: data_reader
      bucket: "*"
```

Bind a user to a group via `CouchbaseRoleBinding`:
```yaml
apiVersion: couchbase.com/v2
kind: CouchbaseRoleBinding
metadata:
  name: analyst-to-analytics-readers
  namespace: couchbase-prod
spec:
  subjects:
    - kind: CouchbaseUser
      name: alice
  roleRef:
    kind: CouchbaseGroup
    name: analytics-readers
```

## CouchbaseBackup

Scheduled backup managed by the operator:

```yaml
apiVersion: couchbase.com/v2
kind: CouchbaseBackup
metadata:
  name: daily-backup
  namespace: couchbase-prod
spec:
  strategy: full_incremental   # full_incremental, full_only, incremental_only
  full:
    schedule: "0 2 * * 0"     # Weekly full — Sundays at 02:00
  incremental:
    schedule: "0 2 * * 1-6"   # Daily incremental — Mon-Sat at 02:00
  backupRetention: 14d         # Keep 14 days of backups
  logRetention: 3d
  storageClassName: couchbase-backup
  size: 200Gi
```

For S3 backup, use `spec.s3bucket` and the S3 credentials secret:
```yaml
spec:
  s3bucket: s3://my-couchbase-backups/prod
  storageSecret: s3-backup-credentials
```

## CouchbaseBackupRestore

Trigger a restore from a specific backup:

```yaml
apiVersion: couchbase.com/v2
kind: CouchbaseBackupRestore
metadata:
  name: restore-2026-05-28
  namespace: couchbase-prod
spec:
  backup: daily-backup          # references the CouchbaseBackup CRD name
  repo: daily-backup-repo
  start:
    int: 1                      # start from backup #1 (or use .str for timestamp)
  end:
    int: 5                      # end at backup #5
  # Optional: restore to specific buckets only
  buckets: {}                   # empty = all buckets
```

## CouchbaseReplicationRepresentation

XDCR between clusters managed as a CRD:

```yaml
# Remote cluster reference
apiVersion: couchbase.com/v2
kind: CouchbaseReplicationRepresentation
metadata:
  name: dr-cluster
  namespace: couchbase-prod
spec:
  uuid: ""                     # filled in automatically after connection
  hostname: 10.0.2.100         # destination cluster node IP
  adminSecret: dr-cluster-credentials
  tls:
    secret: dr-cluster-tls

---
# Replication
apiVersion: couchbase.com/v2
kind: CouchbaseReplicationRepresentation
metadata:
  name: orders-to-dr
  namespace: couchbase-prod
spec:
  bucket: orders
  remoteBucket: orders
  remoteCluster: dr-cluster
  filterExpression: ""         # empty = replicate all
  compressionType: Auto
```

## Scopes and collections

```yaml
apiVersion: couchbase.com/v2
kind: CouchbaseScope
metadata:
  name: my-scope
  namespace: couchbase-prod
spec:
  name: my-scope    # Couchbase scope name
  collections:
    managed: true   # operator manages CouchbaseCollection CRDs within this scope

---
apiVersion: couchbase.com/v2
kind: CouchbaseCollection
metadata:
  name: orders-collection
  namespace: couchbase-prod
spec:
  name: orders
  maxTTL: 0
```
