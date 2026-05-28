# Eventing function authoring

## Handler structure

Every Eventing function is a JavaScript file with one or both lifecycle handlers:

```javascript
function OnUpdate(doc, meta) {
    // Called when a document is created or updated in the source collection.
    // doc  = the full document as a JS object
    // meta = { id, cas, expiry, flags, type, datatype }
}

function OnDelete(meta, options) {
    // Called when a document is deleted (or expires with TTL).
    // meta    = { id, cas }
    // options = { is_expired: bool }
}
```

Both handlers are optional. If you only care about updates, omit `OnDelete`.

## Accessing other collections (bindings)

To read from or write to a collection other than the source, declare a **bucket binding** in the function configuration. The binding name becomes a global variable inside the handler:

```javascript
// Binding name: "dst" → bucket: products, scope: _default, collection: processed
function OnUpdate(doc, meta) {
    dst[meta.id] = { ...doc, processed_at: new Date().toISOString() };
}
```

Binding modes:
- **`read_write`** — read and write access to the bound collection
- **`read_only`** — read access only (safer for reference data lookups)

You can also delete from a bound collection: `delete dst[meta.id]`.

## Timers

Timers let you schedule a callback for a future time — useful for expiry reminders, delayed processing, or periodic sweeps.

```javascript
function OnUpdate(doc, meta) {
    if (doc.type === "session" && doc.expires_at) {
        var expiryDate = new Date(doc.expires_at);
        createTimer(checkExpiry, expiryDate, meta.id, { docId: meta.id });
    }
}

function checkExpiry(context) {
    var doc = src[context.docId];
    if (doc && new Date() >= new Date(doc.expires_at)) {
        // handle expiry
        delete src[context.docId];
    }
}
```

`createTimer(callback, fireAt, reference, context)`:
- `callback` — function name (string or function reference)
- `fireAt` — JavaScript `Date` or Unix timestamp
- `reference` — unique string key for the timer (used for deduplication and cancellation)
- `context` — arbitrary JS object passed to the callback

Cancel a timer: `cancelTimer(checkExpiry, meta.id)`.

Timers are stored in the metadata keyspace. If the function is undeployed, pending timers are lost.

## N1QL (SQL++) inside handlers

Eventing functions can execute SQL++ queries via the built-in `N1QL()` function:

```javascript
function OnUpdate(doc, meta) {
    if (doc.type !== "order") return;

    var result = N1QL(
        "SELECT SUM(o.total) AS total FROM `my-bucket`.orders.completed AS o WHERE o.customer_id = $customer_id",
        { customer_id: doc.customer_id }
    );

    for (var row of result) {
        var profile = profiles[doc.customer_id];
        if (profile) {
            profile.lifetime_value = row.total;
            profiles[doc.customer_id] = profile;
        }
    }
}
```

**Caution:** N1QL inside handlers runs on the Query service. Each mutation triggers a query round-trip. On high-write collections this can saturate the Query service. Prefer KV lookups via bindings for single-document reads.

## curl() for external HTTP calls

```javascript
function OnUpdate(doc, meta) {
    if (doc.status !== "shipped") return;

    var response = curl("POST", notificationService, {
        path: "/notify",
        params: { "Content-Type": "application/json" },
        body: JSON.stringify({ orderId: meta.id, email: doc.customer_email })
    });

    if (response.status !== 200) {
        log("Notification failed for", meta.id, "status:", response.status);
    }
}
```

`curl(method, urlBinding, options)`:
- `method` — `"GET"`, `"POST"`, `"PUT"`, `"DELETE"`, etc.
- `urlBinding` — a URL binding declared in the function configuration (not a raw string — you must declare the host as a binding)
- `options.path` — appended to the binding URL
- `options.params` — HTTP headers
- `options.body` — request body string

Declare URL bindings with a name (e.g. `notificationService`) in the function config, pointing to the base URL. This prevents hardcoded URLs and allows environment-specific configuration.

**Timeout:** curl() defaults to the function's `curl_timeout` setting (default 2000ms). Long-running HTTP calls will block mutation processing. Keep external calls fast; use asynchronous patterns (fire-and-forget via a queue collection) for slow APIs.

## Logging

```javascript
log("Processing order", meta.id, "status:", doc.status);
```

`log()` writes to the Eventing service log, visible via `admin_eventing_get_function_stats` or the Couchbase UI. Use it for debugging; remove or reduce in production (logging is synchronous and adds latency).

## Error handling patterns

**Transient errors (network, temporary unavailability):** the Eventing service will retry the handler on failure up to the configured `execution_timeout`. For expected transient failures (curl timeouts, temporary N1QL errors), catch the exception and log; the mutation will be retried.

```javascript
function OnUpdate(doc, meta) {
    try {
        // potentially-failing operation
    } catch (e) {
        log("Error processing", meta.id, ":", e.message);
        // If you want to retry, re-throw. If you want to skip, just log.
    }
}
```

**Permanent errors (bad data, expected missing fields):** guard defensively at the top of the handler and return early. Don't let bad documents block the queue.

```javascript
function OnUpdate(doc, meta) {
    if (!doc.type || doc.type !== "order") return;
    if (!doc.customer_id) { log("Missing customer_id on", meta.id); return; }
    // process
}
```

## Handler performance guidelines

- **Return early** for documents you don't care about. The type check on the first line is your biggest lever.
- **Prefer KV bindings over N1QL** for single-document lookups — a binding read is a direct KV get; N1QL involves query parsing, plan, and execution.
- **Batch-aware design:** Eventing processes mutations one at a time per worker. If you need bulk processing, accumulate work orders in a queue collection and process them with a separate function or external consumer.
- **Keep handlers under ~100ms.** Longer handlers reduce throughput proportionally. The Eventing service has a configurable `execution_timeout` (default 60s) but it's a ceiling, not a goal.
