# Eventing deployment

## Function configuration

When creating a function via `admin_eventing_upsert_function`, the key configuration fields:

```json
{
  "appname": "my-function",
  "appcode": "function OnUpdate(doc, meta) { ... }",
  "settings": {
    "deployment_status": false,
    "processing_status": false,
    "description": "Syncs orders to processed collection",
    "worker_count": 3,
    "cpp_worker_thread_count": 2,
    "execution_timeout": 60,
    "cursor_checkpoint_timeout": 60,
    "timer_context_size": 1024,
    "log_level": "INFO",
    "language_compatibility": "6.6.2",
    "user_prefix": "eventing_meta",
    "n1ql_consistency": "request_plus",
    "curl_timeout": 2000
  },
  "depcfg": {
    "source_bucket": "my-bucket",
    "source_scope": "my-scope",
    "source_collection": "orders",
    "metadata_bucket": "eventing-meta",
    "metadata_scope": "_default",
    "metadata_collection": "_default",
    "buckets": [
      {
        "alias": "processed",
        "bucket_name": "my-bucket",
        "scope_name": "my-scope",
        "collection_name": "processed-orders",
        "access": "rw"
      }
    ],
    "curl": [
      {
        "hostname": "https://api.notification-service.internal",
        "value": "notificationService",
        "auth_type": "bearer",
        "bearer_key": "NOTIFICATION_API_KEY",
        "allow_cookies": false,
        "validate_ssl_certificate": true
      }
    ]
  }
}
```

## Key configuration decisions

### Worker count

`worker_count` controls how many parallel JavaScript V8 workers process mutations. Each worker handles one mutation at a time.

Guideline: start with `worker_count = 3`. Increase if:
- `admin_eventing_get_function_stats` shows a consistently high `dcp_backlog` (mutations waiting to be processed)
- Handler execution time is short (< 10ms) but throughput is bottlenecked

Don't over-provision: each worker holds memory and CPU. On a 3-node Eventing service, `worker_count = 3` gives 9 total workers across the cluster.

### from_now deployment

By default, deploying a function processes all existing documents in the source collection from the beginning. This can take a long time on large collections.

Use `from_now: true` when:
- You only want to process new mutations going forward
- The function is an additive pipeline (doesn't need to backfill historical data)

Set via the deploy request body: `{ "from_now": true }`.

### execution_timeout

Maximum execution time per handler invocation in seconds (default 60). If a handler exceeds this:
- The worker is killed and the mutation is considered failed
- The function's `timeout_count` stat increments

Lower this if your handlers are supposed to be fast and you want to catch runaway handlers early. Raise it only if you have legitimately long-running operations (rare — see principle 3 in SKILL.md).

### n1ql_consistency

Controls scan consistency for N1QL queries called inside the handler:
- `none` — no consistency guarantee (fastest)
- `request_plus` — consistent with the latest mutations at the time the query is issued

Use `request_plus` when the handler queries data it just wrote. Otherwise `none` is fine and reduces query latency.

### language_compatibility

Determines which JavaScript/Eventing API features are available. Set to the Couchbase version you're targeting. Use `6.6.2` unless you need features from a newer compatibility level.

## Metadata keyspace guidelines

The metadata keyspace is used internally by Eventing to:
- Store DCP stream checkpoints (where the function has processed to)
- Store timer state
- Store internal function state documents

Rules:
- Never store application data in the metadata collection
- Never use the metadata collection as a source collection for another function
- Dedicate one collection per Eventing function (or at minimum one bucket), or use a separate `user_prefix` to namespace multiple functions in the same collection
- The metadata collection needs ~1 vBucket worth of RAM per active function. A dedicated small bucket (256 MB) is fine for most deployments.

## Upgrading / redeploying functions

To update function code without losing DCP checkpoint position:

1. `admin_eventing_pause_function` — pauses processing; holds checkpoint
2. `admin_eventing_upsert_function` — update the function code/config
3. `admin_eventing_resume_function` — resumes from the paused checkpoint

If you undeploy instead of pause, the checkpoint is lost and the next deploy reprocesses from the beginning (unless `from_now: true`).

## RBAC requirements

The Eventing service runs under a configured service account. The account needs:

- `eventing_manage` — to deploy/undeploy functions
- Data read/write on source and destination buckets (via bucket bindings)
- `query_execute` on relevant keyspaces (if using N1QL inside handlers)

Grant the minimum needed. Overly broad permissions on the Eventing service account are a common security risk in Couchbase deployments.
