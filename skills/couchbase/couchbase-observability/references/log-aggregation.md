# Log aggregation

## Couchbase log locations

On each node, logs live in `/var/lib/couchbase/logs/` (Linux) or the equivalent install path on Windows. Key log files:

| File | Contains |
|---|---|
| `couchdb.log` | Data service (KV, DCP, persistence) |
| `query.log` | Query service (N1QL execution, errors) |
| `fts.log` | Full Text Search service |
| `eventing.log` | Eventing function execution and errors |
| `xdcr.log` | XDCR replication events |
| `analytics.log` | Analytics service |
| `memcached.log` | KV engine detailed log |
| `ns_server.log` | Cluster management, REST API |
| `babysitter.log` | Process supervision (startup, crashes) |
| `audit.log` | Audit events (see `audit-logging.md`) |
| `error.log` | Critical errors across all services |

**Note:** each log file rotates automatically. The active file is `<name>.log`; rotated files are `<name>.log.1`, `.2`, etc. Configure your log shipper to tail the active file and pick up rotated files.

## Fetching logs via MCP

For quick triage without log infrastructure:

```python
admin_cluster_get_logs(
    cluster="prod",
    logName="ns_server",     # optional: filter to a specific log
    since="2026-05-28T03:00:00Z"  # optional: from timestamp
)
```

Returns recent log entries in JSON. Useful for incident investigation; not a substitute for a log pipeline.

## Filebeat configuration

```yaml
# /etc/filebeat/filebeat.yml
filebeat.inputs:
  - type: log
    id: couchbase-data-service
    paths:
      - /var/lib/couchbase/logs/couchdb.log*
    fields:
      couchbase_service: kv
      couchbase_cluster: prod
      couchbase_node: "${NODE_HOSTNAME}"
    fields_under_root: true
    multiline:
      type: pattern
      pattern: '^\d{4}-\d{2}-\d{2}'
      negate: true
      match: after

  - type: log
    id: couchbase-query
    paths:
      - /var/lib/couchbase/logs/query.log*
    fields:
      couchbase_service: query
      couchbase_cluster: prod
      couchbase_node: "${NODE_HOSTNAME}"
    fields_under_root: true

  - type: log
    id: couchbase-audit
    paths:
      - /var/lib/couchbase/logs/audit.log*
    fields:
      couchbase_service: audit
      couchbase_cluster: prod
    fields_under_root: true
    json.keys_under_root: true   # audit.log is structured JSON

output.elasticsearch:
  hosts: ["https://elasticsearch.example.com:9200"]
  username: "couchbase-shipper"
  password: "${ES_PASSWORD}"
  index: "couchbase-logs-%{+yyyy.MM.dd}"
```

Run Filebeat on every Couchbase node (or use a log sidecar in containerized deployments).

## Fluent Bit configuration

```ini
[INPUT]
    Name              tail
    Path              /var/lib/couchbase/logs/couchdb.log*
    Tag               couchbase.kv
    Refresh_Interval  5
    Skip_Long_Lines   On

[INPUT]
    Name              tail
    Path              /var/lib/couchbase/logs/audit.log*
    Tag               couchbase.audit
    Parser            json
    Refresh_Interval  5

[FILTER]
    Name    record_modifier
    Match   couchbase.*
    Record  cluster prod
    Record  node    ${HOSTNAME}

[OUTPUT]
    Name            es
    Match           couchbase.*
    Host            elasticsearch.example.com
    Port            9200
    TLS             On
    Index           couchbase-logs
    Time_Key        @timestamp
```

## Splunk

Use the Couchbase Technology Add-On for Splunk (available on Splunkbase). It provides pre-built field extractions for Couchbase log formats and source types. Configure the Universal Forwarder to monitor the Couchbase log directory.

If using the TA: `source=/var/lib/couchbase/logs/*, sourcetype=couchbase:*`

## Datadog

Use the Datadog Agent with the Couchbase integration. The integration provides:
- Log collection (configure in `couchbase.d/conf.yaml`)
- Metric collection from the REST API
- Pre-built dashboards and monitors

```yaml
# /etc/datadog-agent/conf.d/couchbase.d/conf.yaml
instances:
  - server: https://localhost:18091
    user: datadog-reader
    password: ENC[<encrypted>]
    timeout: 10

logs:
  - type: file
    path: /var/lib/couchbase/logs/error.log
    service: couchbase
    source: couchbase
    tags:
      - cluster:prod

  - type: file
    path: /var/lib/couchbase/logs/audit.log
    service: couchbase
    source: couchbase
    tags:
      - cluster:prod
      - log_type:audit
```

## Key log patterns to alert on

Set up log-based alerts in your SIEM for these patterns:

| Pattern | Log file | Severity | What it means |
|---|---|---|---|
| `OOM\|out of memory` | couchdb.log | P1 | Data service memory exhausted |
| `CRITICAL\|ERROR` | error.log | P2 | Cross-service critical errors |
| `Failed to connect\|connection refused` | xdcr.log | P2 | XDCR connectivity issue |
| `FATAL` | any | P1 | Process about to crash |
| `Rebalance started\|Rebalance completed\|Rebalance failed` | ns_server.log | Info / P2 if failed | Track rebalance lifecycle |
| `Failover\|failover` | ns_server.log | P1 | Node has failed over |
| `authentication failed\|401` | audit.log | P2 | Repeated auth failures = brute force |
| `n1ql_errors` spike in query.log | query.log | P2 | Query failure rate increase |
