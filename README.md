# Couchbase Claude Skills

Claude skill files for working with Couchbase — covering every major service and deployment pattern from application integration through AI applications, Kubernetes operations, mobile sync, security hardening, and analytics.

**29 skills · 124 files · ~18,000 lines**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

---

## What are Claude Skills?

Skills are structured context files (`SKILL.md` + optional `references/` docs) that Claude loads before responding to requests in a specific domain. They encode patterns, anti-patterns, tool inventory, and decision trees that Claude's general training doesn't reliably cover — especially for fast-moving products like Couchbase.

Skills work with Claude Code, Claude.ai Projects, OpenAI Codex CLI, Cursor, Gemini CLI, and any agent that supports the open Agent Skills standard.

---

## How to use these skills

### Option 1 — Skills CLI (one command, works with Claude Code and most agents)

```bash
npx skills add celticht32/Couchbase-Skills-for-Claude.ai
```

This installs all skills into your project's `.claude/skills/` directory. Claude Code picks them up automatically on the next session.

To install a single skill:

```bash
npx skills add celticht32/Couchbase-Skills-for-Claude.ai --skill couchbase-sqlpp-tuning
```

### Option 2 — Claude Code (manual import via CLAUDE.md)

Clone the repo and add references to your project's `CLAUDE.md`:

```markdown
@skills/couchbase/couchbase-mcp/SKILL.md
@skills/couchbase/couchbase-sqlpp-tuning/SKILL.md
```

Or load the full collection and let Claude route:

```markdown
See @skills/ for available Couchbase skills. Load the relevant one before responding to any Couchbase question.
```

### Option 3 — Claude.ai Projects

1. Open [claude.ai](https://claude.ai) and create or open a Project
2. Go to **Project Knowledge → Add content**
3. Upload the `SKILL.md` files for the skills you want active
4. Every conversation in that project will have them in context automatically

### Option 4 — Any Claude interface (manual paste)

Open any skill's `SKILL.md`, copy the contents, and paste it into your system prompt or at the start of a conversation. For deep questions, also paste the relevant `references/*.md` file.

---

## Best practices for Claude.ai Projects

A Claude.ai Project gives you a persistent Couchbase assistant — the right skills are always loaded, you never re-explain your environment, and every session starts with full context.

### Recommended Project Knowledge

Don't load all 29 skills — that consumes most of your context window before you type anything. Load the skills you reach for in nearly every session and pull the others in as needed.

**Always-on (upload these to Project Knowledge):**

| File | Why |
|---|---|
| `skills/couchbase/couchbase-mcp/SKILL.md` | You'll be calling MCP tools in almost every session |
| `skills/couchbase/couchbase-sqlpp-tuning/SKILL.md` | Query questions come up constantly |
| `skills/couchbase/couchbase-data-modeling/SKILL.md` | Design decisions arise early in any new work |

**Add reference files for your most common deep-dive topics.** For example if you do a lot of query work, also upload:
- `skills/couchbase/couchbase-sqlpp-tuning/references/query-patterns.md`
- `skills/couchbase/couchbase-sqlpp-tuning/references/explain-plan.md`

**For the other 26 skills:** paste the relevant `SKILL.md` at the start of the conversation when you need it, or use `@` imports in Claude Code.

### Recommended Project Instructions

Set this in your Project's instruction field (the persistent system prompt for every conversation in the project):

```
I work with Couchbase clusters and MCP servers. Use the loaded skills before
answering any Couchbase question. When a question falls outside the loaded
skills, say so and I'll provide the relevant skill. Prefer concise answers.
Treat me as technical — skip basics.
```

Adjust to match your environment (self-managed vs Capella, your Couchbase version, your primary languages).

### When to add more skills mid-conversation

If a session moves into territory not covered by your always-on skills — XDCR topology decisions, Kubernetes operator config, a migration plan — paste the relevant `SKILL.md` at that point:

> "Here's the XDCR skill for this question: [paste contents of couchbase-xdcr/SKILL.md]"

Claude will use it for the rest of that conversation without it consuming Project Knowledge space permanently.

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
