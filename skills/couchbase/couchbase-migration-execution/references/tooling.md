# Migration tooling

The tools for moving data into Couchbase, organized from simplest to most complex. Picking the right tool saves weeks of engineering — picking the wrong one means writing infrastructure that already exists.

## The Couchbase-native tools

These ship with Couchbase Server (self-managed) and are available via Capella's tools UI:

### `cbimport`

Bulk import from JSON or CSV files into Couchbase. The simplest possible loader.

**Format 1 — JSON lines (one document per line):**

```bash
cbimport json \
    --cluster couchbases://cb.example.com \
    --username Administrator \
    --password ... \
    --bucket app_data \
    --format lines \
    --dataset file:///path/to/users.jsonl \
    --generate-key "user::%id%"
```

The `--generate-key` template extracts a field from the document to form the key. `%id%` means use the document's `id` field; multiple field substitutions are supported.

**Format 2 — JSON array (one big array of documents):**

```bash
cbimport json --format list --dataset file:///path/to/users.json ...
```

Same idea but a single JSON array instead of newline-delimited.

**Format 3 — CSV:**

```bash
cbimport csv \
    --cluster ... \
    --bucket app_data \
    --dataset file:///path/to/users.csv \
    --generate-key "user::%id%"
```

CSV is row-oriented; each row becomes a document. Field names come from the header row.

**Useful options:**
- `--threads N` — parallelism (default 1; bump to 4-16 for faster loads)
- `--scope-collection-exp` — target a specific scope/collection (e.g., `_default.users`)
- `--errors-log` — write rejected rows to a file for inspection
- `--ignore-fields` — skip specific fields during import
- `--scope-name` and `--collection-name` — required for non-default targets

**Performance:** `cbimport` is fast on small-to-medium files (single-digit GB). For TB-scale loads, parallelize by splitting the input file and running multiple cbimports against the same bucket.

**Limits:**
- Single source file (no streaming from databases — file-only)
- No transformation beyond key generation (the documents are loaded as-is)
- One bucket per invocation

### `cbexport`

Mirror of `cbimport`. Exports a bucket / collection / scope to JSON or CSV.

```bash
cbexport json \
    --cluster ... \
    --bucket app_data \
    --format lines \
    --output /path/to/export.jsonl
```

Useful for:
- Initial export from one Couchbase cluster as input to another's import
- Backing up specific collections for migration validation
- Pulling data out for ETL pipelines that can't read Couchbase directly

### `cbtransfer`

Moves data between Couchbase clusters (or between buckets within a cluster). Older tool, less commonly recommended now — XDCR (via the `couchbase-mcp` skill's `admin_xdcr_*` tools) is usually a better choice for cluster-to-cluster.

When `cbtransfer` is still useful: one-shot transfers where you don't want to set up ongoing replication.

### `cbbackup` / `cbbackupmgr`

Couchbase's backup tools, accessible via the `couchbase-mcp` skill's `admin_backup_*` tools. While primarily for backup/restore, they can be used for migration:

- Backup the source cluster
- Restore the backup into the target (different cluster, version, or layout)

Useful for: cluster version upgrades, hardware migrations, cluster-to-cluster moves where structure is preserved.

## Source-specific export tools

The source database often has better export tooling than what you'd write yourself:

| Source | Recommended export tool |
|---|---|
| MongoDB | `mongoexport` (per-collection JSON), `mongodump` (BSON dumps) |
| PostgreSQL | `pg_dump` (full schema + data), `COPY ... TO ...` (CSV/text per-table) |
| MySQL | `mysqldump`, or `SELECT ... INTO OUTFILE` |
| Oracle | `expdp` (Data Pump), or `SQL*Plus` for CSV exports |
| SQL Server | `bcp` (bulk copy), or SSIS for complex exports |
| DynamoDB | AWS Data Pipeline, or DynamoDB Streams + custom Lambda |
| Cassandra | `cqlsh` `COPY TO`, or DSBulk |

Use these to export from source; transform with code or tooling; load with `cbimport`. This split (export-transform-import) is cleaner than trying to do all three in one pipeline.

## ETL frameworks

For migrations that need transformation (relational → document), an ETL framework saves time vs writing custom code.

### Open-source ETL tools

**Apache Nifi** — visual flow-based ETL. Has processors for many sources and a Couchbase sink. Good for one-shot or recurring migrations.

**Apache Airflow** — workflow orchestrator. Not strictly an ETL tool but commonly used to orchestrate migration tasks: export → transform → import → validate.

**Talend Open Studio** — visual ETL designer; commercial paid version available.

**Pentaho Data Integration (Kettle)** — similar visual ETL. Free community edition.

**Custom Python with pandas / polars** — for one-shot migrations where the transformation is well-defined, often the right answer. Read source, transform in DataFrame, write to Couchbase via SDK.

### Cloud ETL services

**AWS Glue** — managed ETL with Spark. Has a Couchbase connector (verify current support). Good if you're already AWS-native.

**Azure Data Factory** — managed ETL with Couchbase Connector (verify current support).

**Google Cloud Dataflow** — Apache Beam-based; build a custom pipeline.

These are good for ongoing pipelines but heavy for one-shot migrations.

## CDC tools

For ongoing source-to-target sync during dual-write migrations:

### Debezium

The most popular open-source CDC tool. Reads source DB transaction logs and emits changes to Kafka. A separate sink connector then writes from Kafka to Couchbase.

**Architecture:**

```
[Source DB] → [Debezium] → [Kafka] → [Couchbase Kafka Connector] → [Couchbase]
```

**Sources supported:**
- PostgreSQL
- MySQL
- MongoDB
- SQL Server
- Oracle (with extra setup)
- DB2

**Pros:**
- Battle-tested in production at scale
- Captures all changes (inserts, updates, deletes)
- Resumable — survives connector restarts
- Open-source

**Cons:**
- Real ops surface: Kafka cluster, Kafka Connect, Debezium connector, Couchbase connector — multiple things that can break
- Setup complexity is meaningful — plan a week minimum to get it stable
- Transformation between source and Couchbase format is via Single Message Transforms (SMTs) or Kafka stream processing — can get complex

### AWS DMS (Database Migration Service)

Managed CDC for AWS users. Sources: most RDBMS + MongoDB. Targets include Couchbase via custom configuration.

**Pros:**
- Managed — no Kafka cluster to run
- Pay-per-task-hour pricing
- Built-in monitoring

**Cons:**
- AWS-only
- Transformation capabilities are limited compared to Debezium + custom processors
- Couchbase target may need custom JS transformation rules

### Custom CDC

For sources without good off-the-shelf CDC (or for simpler use cases), build your own:
- Poll the source's `updated_at` timestamps
- Subscribe to source's pub/sub (MongoDB changestreams, Postgres logical replication)
- Stream from a queue (Kafka, SQS, Pub/Sub)

Workable for small projects; doesn't scale to enterprise migration. Use Debezium for anything serious.

## Couchbase-specific extras

### Kafka Connector

If your migration involves Kafka anywhere (CDC, event streams, etc.), the Couchbase Kafka Connector is the standard sink. It runs on Kafka Connect alongside Debezium or other source connectors.

Source mode: read from a Couchbase bucket and emit to Kafka.
Sink mode: read from Kafka and write to Couchbase.

For migrations: sink mode is what you usually want.

### Spark Connector

For migrations involving Spark (often Hadoop / data lake migrations to Couchbase), the Couchbase Spark Connector exposes Couchbase as a Spark DataFrame source/sink. Read a Spark DataFrame from anywhere; write it to Couchbase via the connector.

### Custom code via SDK

When the standard tools don't fit:

```python
from couchbase.cluster import Cluster
from couchbase.auth import PasswordAuthenticator
from couchbase.options import ClusterOptions, UpsertMultiOptions
from datetime import timedelta

cluster = Cluster("couchbases://...",
                  ClusterOptions(authenticator=PasswordAuthenticator("user", "pass")))
collection = cluster.bucket("app").scope("_default").collection("users")

batch = {}
for row in read_source():
    doc_key = f"user::{row['user_id']}"
    doc = transform(row)
    batch[doc_key] = doc
    if len(batch) >= 1000:
        collection.upsert_multi(batch)
        batch.clear()
if batch:
    collection.upsert_multi(batch)
```

This pattern works for:
- Custom source formats
- Complex transformations
- Migrations where standard tools choke (oddly-shaped data)

See `couchbase-app-integration` skill for the SDK details.

## Picking the right tool

Decision flow:

```
Source is a JSON/CSV file?
├── No transformation needed → cbimport
└── Needs transformation → write a script: read file, transform, SDK upsert_multi

Source is MongoDB?
├── Use mongoexport + transform script + cbimport (one-shot)
└── Or Debezium + Kafka + Couchbase Kafka Connector (ongoing)

Source is relational?
├── pg_dump/mysqldump + transform + cbimport (one-shot)
└── Debezium / AWS DMS (ongoing CDC)

Source is in AWS?
└── AWS DMS or AWS Glue, depending on transformation complexity

Source is something else (DynamoDB, Cassandra)?
└── Source's export tool + transform + cbimport, OR custom CDC
```

## Estimating tool effort

| Tool | Setup time | Maintenance burden |
|---|---|---|
| `cbimport` | Minutes | None — one-shot |
| Custom Python script | Hours-days | Low — runs once |
| ETL framework (Nifi, Airflow) | Days | Medium — operate the framework |
| Debezium + Kafka | 1-2 weeks | High — ongoing ops |
| AWS DMS | Days | Low-medium — managed |
| Apache Spark migration | Days-weeks | High if Spark cluster isn't already running |

For one-shot migrations, prefer simpler tools. For ongoing sync (dual-write CDC), invest in real CDC tooling.

## Quick decision tree

- **One-shot, file-based source, no transformation?** → `cbimport`
- **One-shot, file source, needs transformation?** → Custom script + SDK
- **One-shot, complex source schemas?** → ETL framework (Nifi, Airflow, Spark)
- **Ongoing sync (dual-write or zero-downtime CDC)?** → Debezium + Kafka + Couchbase Kafka Connector
- **AWS-native, ongoing sync?** → AWS DMS
- **Couchbase to Couchbase?** → XDCR (via `couchbase-mcp` skill), not these tools
- **Anything involving Kafka?** → Couchbase Kafka Connector as sink
