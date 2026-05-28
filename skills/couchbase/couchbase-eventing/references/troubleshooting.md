# Eventing troubleshooting

## Diagnostic workflow

1. **`admin_eventing_get_function_stats(function_name)`** — start here. Key fields:
   - `dcp_backlog` — number of mutations waiting to be processed. Non-zero and growing means the function can't keep up.
   - `failed_count` — handler invocations that threw an uncaught exception.
   - `timeout_count` — handler invocations that exceeded `execution_timeout`.
   - `processed_count` — cumulative mutations handled successfully.
   - `timer_cancel_counter`, `timer_create_counter`, `timer_callback_missing_counter` — timer health.

2. **`admin_eventing_list_functions`** — confirm `deployment_status` is `deployed` and `processing_status` is `running`.

3. **Check Eventing logs** via the Couchbase UI (Eventing → function → Logs tab) or `admin_stats_logs` for service-level logs.

## Function not triggering

**Confirm the function is deployed and running.**
`deployment_status: false` means it's not deployed. `processing_status: false` means it's paused. Use `admin_eventing_deploy_function` or `admin_eventing_resume_function`.

**Confirm the source collection matches.**
The source bucket/scope/collection in `depcfg` must exactly match where documents are being written. A document written to `my-scope.orders` won't trigger a function sourced on `_default._default`.

**Confirm the mutation type matches.**
`OnUpdate` fires on creates and updates. `OnDelete` fires on deletes and TTL expirations. If documents are being deleted and only `OnUpdate` is defined, nothing triggers.

**Check `dcp_backlog`.**
A large `dcp_backlog` means mutations are queued. The function is processing — just slowly. Check `failed_count` and `timeout_count` for the reason.

## Function processing slowly / DCP backlog growing

**Option 1: Handler is too slow.**
Profile the handler logic. Common culprits: N1QL queries inside the handler (especially without covering indexes), curl() calls to slow external services, large document reads via bindings.

Fix: replace N1QL lookups with binding KV reads where possible; move slow external calls to a queue-and-process pattern.

**Option 2: Not enough workers.**
Increase `worker_count` via an `admin_eventing_upsert_function` + redeploy cycle (pause → update → resume to preserve checkpoint).

**Option 3: Source mutation rate exceeds processing capacity.**
This is a scaling limit. Eventing isn't designed for extremely high-throughput sources (> tens of thousands of mutations/sec per function). For those rates, consider SDK-based DCP consumers (`couchbase-app-integration`) which can be horizontally scaled outside the cluster.

## High failed_count

**Get the error details from the function logs** (Eventing UI → function → Logs). Errors are logged per-invocation.

Common causes:

| Error type | Cause | Fix |
|---|---|---|
| `TypeError: cannot read property X of undefined` | Handler doesn't guard against missing fields | Add `if (!doc.field) return;` guards at the top |
| `KVError: document not found` | Handler reads from a binding by key but the doc was deleted | Check before reading: `var d = binding[key]; if (d) { ... }` |
| `N1QL error: keyspace not found` | Bucket/scope/collection name in the N1QL is wrong | Verify the keyspace path; check for scope changes |
| `curl error: connection refused` | URL binding host is unreachable | Check network connectivity from Eventing nodes to the target; check URL binding config |
| `execution timeout` | Handler ran past `execution_timeout` | Identify the slow operation (usually N1QL or curl); optimize or raise the timeout |

## Handler runs but produces wrong results

**Check idempotency.** If a mutation is delivered twice (during rebalance, failover, or redeploy), does the handler produce the correct result? The second run should overwrite cleanly, not append or double-count.

**Check `from_now` vs historical deployment.** If you deployed with `from_now: false` (the default) on an existing collection, the function processes all existing documents. If your handler writes derived documents, you may see unexpected documents created from old data.

**Check the metadata keyspace for conflicts.** If two functions share the same metadata collection without distinct `user_prefix` values, their checkpoints can collide. Each function should have its own metadata collection or a unique prefix.

## Timer not firing

**Check `timer_callback_missing_counter`.**
A non-zero value means a timer fired but the callback function couldn't be found — usually because the function code was updated and the callback name changed. Timers created before the rename will fire and fail. Re-create the timers after renaming callbacks.

**Check that the function is deployed when the timer fires.**
If the function is undeployed between timer creation and fire time, the timer is lost.

**Check the metadata collection has sufficient space.**
Timer state is stored in the metadata keyspace. If the metadata bucket is full, timer creation fails silently. Monitor metadata bucket disk usage.

## Eventing service memory pressure

`admin_stats_eventing` reports the Eventing service's RAM usage. If the service runs out of memory, it throttles processing. Causes:

- Too many workers with large `timer_context_size`
- Large documents being held in-memory during handler execution
- High `dcp_backlog` causing the DCP buffer to grow

Fix: reduce `worker_count`, reduce `timer_context_size`, or add Eventing service nodes. The Eventing service memory quota is configured in the cluster's memory settings (separate from the Data service quota).
