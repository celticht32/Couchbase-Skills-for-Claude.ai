# Couchbase Claude Skills

Claude skill files for working with Couchbase — covering every major service and deployment pattern from application integration through AI applications, Kubernetes operations, mobile sync, security hardening, and analytics.

**29 skills · 124 files · ~18,000 lines**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

---

## What are Claude Skills?

Skills are structured context files (`SKILL.md` + optional `references/` docs) that Claude loads before responding to requests in a specific domain. They encode patterns, anti-patterns, tool inventory, and decision trees that Claude's general training doesn't reliably cover — especially for fast-moving products like Couchbase.

---

## Installation

### Claude Code

Add the relevant skills to your project's `CLAUDE.md`:

```markdown
@skills/couchbase/couchbase-mcp/SKILL.md
@skills/couchbase/couchbase-sqlpp-tuning/SKILL.md
```

Or let Claude route across all skills:

```markdown
See @skills/ for available Couchbase skills. Load the relevant one before responding to any Couchbase question.
```

### Claude.ai Projects

Upload the skill files as Project Knowledge. Claude will reference them automatically in that project's conversations.

### Manual

Paste the relevant `SKILL.md` into your system prompt or at the start of a conversation.

---

## Skills

### Core Couchbase (`skills/couchbase/`)

| Skill | Coverage | Refs |
|---|---|---|
| [`couchbase-mcp`](skills/couchbase/couchbase-mcp/) | Operating clusters via [celticht32/MCP-Couchbase](https://github.com/celticht32/MCP-Couchbase) (167 tools) — KV, SQL++, FTS, indexes, XDCR, eventing, backup, KMIP, Capella v4 | 11 |
| [`couchbase-app-integration`](skills/couchbase/couchbase-app-integration/) | SDK integration — connection pooling, retry, durability, scan consistency, transactions, XDCR-aware patterns | 7 |
| [`couchbase-data-modeling`](skills/couchbase/couchbase-data-modeling/) | Document design, boundaries, key formats, embed vs reference, TTL, time-series, vector modeling | 7 |
| [`couchbase-sizing`](skills/couchbase/couchbase-sizing/) | Cluster sizing math — RAM, disk, node count, replicas, index memory budgets, Capella tier selection | 7 |
| [`couchbase-sqlpp-tuning`](skills/couchbase/couchbase-sqlpp-tuning/) | Query tuning — EXPLAIN plans, index design, 15 named anti-patterns, CBO, array indexes, joins, pagination | 7 |
| [`couchbase-migration-execution`](skills/couchbase/couchbase-migration-execution/) | Data migration — dual-write, CDC, cbimport, cutover runbooks, validation, rollback | 7 |
| [`couchbase-fts`](skills/couchbase/couchbase-fts/) | Full Text Search and vector search — index design, analyzers, query types, kNN, hybrid search, synonyms | 6 |
| [`couchbase-ai-applications`](skills/couchbase/couchbase-ai-applications/) | AI application design — HVI/CVI/SVI index selection, RAG pipelines, embedding strategies, LangChain/LlamaIndex | 4 |
| [`couchbase-transactions`](skills/couchbase/couchbase-transactions/) | Distributed ACID transactions — when to use, two-phase commit, ATRs, retry, Python/Java/Node patterns | 3 |
| [`couchbase-eventing`](skills/couchbase/couchbase-eventing/) | Eventing functions — OnUpdate/OnDelete, timers, curl(), N1QL in functions, deployment lifecycle | 3 |
| [`couchbase-xdcr`](skills/couchbase/couchbase-xdcr/) | Cross-datacenter replication — topology, active-active vs passive, conflict resolution, filtering | 4 |
| [`couchbase-backup-restore`](skills/couchbase/couchbase-backup-restore/) | Backup and restore — cbbackupmgr, strategy, full/incremental, S3, point-in-time, Capella managed backups | 4 |
| [`couchbase-mobile`](skills/couchbase/couchbase-mobile/) | Mobile / offline-first — Couchbase Lite, Sync Gateway, Capella App Services, sync functions | 3 |
| [`couchbase-capella`](skills/couchbase/couchbase-capella/) | Capella provisioning — getting started, networking (VPC peering, PrivateLink), database credentials | 3 |
| [`couchbase-kubernetes`](skills/couchbase/couchbase-kubernetes/) | Kubernetes via Couchbase Autonomous Operator — CRDs, AZ awareness, rolling upgrades, PVs, Prometheus | 4 |
| [`couchbase-security-hardening`](skills/couchbase/couchbase-security-hardening/) | Production security — TLS, RBAC design, LDAP/SAML, audit logging, DARE encryption, network hardening | 6 |
| [`couchbase-observability`](skills/couchbase/couchbase-observability/) | Monitoring — key metrics, Prometheus/Grafana, alert thresholds, log aggregation | 4 |
| [`couchbase-performance-tuning`](skills/couchbase/couchbase-performance-tuning/) | Cluster performance — KV latency, DCP backpressure, compaction, connection limits, OS tuning | 3 |
| [`couchbase-upgrade`](skills/couchbase/couchbase-upgrade/) | Version upgrades — upgrade paths, 8.0 breaking changes, Magma default, pre/post-upgrade checklist | — |
| [`couchbase-magma`](skills/couchbase/couchbase-magma/) | Magma storage engine — couchstore vs Magma, 128 vs 1024 vBuckets, compaction, memory requirements | — |
| [`couchbase-columnar`](skills/couchbase/couchbase-columnar/) | Capella Columnar — columnar analytics, links to operational data, BI tool integration | 2 |

### Analytics Service (`skills/couchbase-analytics/`)

These skills document the [celticht32/Couchbase-Analytics-MCP-Server](https://github.com/celticht32/Couchbase-Analytics-MCP-Server).

| Skill | Coverage |
|---|---|
| [`cb-analytics-query`](skills/couchbase-analytics/cb-analytics-query/) | SQL++ Analytics queries — tool selection, pagination, truncation, scan consistency |
| [`cb-analytics-schema`](skills/couchbase-analytics/cb-analytics-schema/) | Dataset discovery, schema inference, data dictionary workflows |
| [`cb-analytics-admin`](skills/couchbase-analytics/cb-analytics-admin/) | Runtime management — ingestion health, active requests, cancel, restart |
| [`cb-analytics-links`](skills/couchbase-analytics/cb-analytics-links/) | External data source links — S3, Azure Blob, GCS, remote Couchbase |
| [`cb-analytics-security`](skills/couchbase-analytics/cb-analytics-security/) | RBAC — users, groups, roles, service accounts, rate limits |
| [`cb-analytics-cluster`](skills/couchbase-analytics/cb-analytics-cluster/) | Cluster-level ops — nodes, memory quotas, rebalance, auto-failover, system events |
| [`cb-analytics-capella`](skills/couchbase-analytics/cb-analytics-capella/) | Capella v4 control-plane — list/create/delete clusters, backups, restore |
| [`cb-analytics-mcp-setup`](skills/couchbase-analytics/cb-analytics-mcp-setup/) | Installing and configuring the Analytics MCP server |

---

## Repository structure

```
skills/
  couchbase/           # 21 core Couchbase Server / Capella skills
    <skill-name>/
      SKILL.md
      references/
        <topic>.md
  couchbase-analytics/ # 8 Analytics MCP skills
    <skill-name>/
      SKILL.md

CLAUDE.md              # Claude Code project memory — contributor guidance
CONTRIBUTING.md        # Skill format spec and contribution workflow
README.md
LICENSE
```

---

## Related projects

| Project | What it is |
|---|---|
| [celticht32/MCP-Couchbase](https://github.com/celticht32/MCP-Couchbase) | 167-tool Couchbase MCP server documented by the `couchbase-mcp` skill |
| [celticht32/Couchbase-Analytics-MCP-Server](https://github.com/celticht32/Couchbase-Analytics-MCP-Server) | Analytics MCP server documented by the `cb-analytics-*` skills |

---

## Contributing

See [CLAUDE.md](CLAUDE.md) for the skill format spec and [CONTRIBUTING.md](CONTRIBUTING.md) for the contribution workflow.

---

## License

MIT — Copyright (c) 2026 Chris Ahrendt. See [LICENSE](LICENSE).
