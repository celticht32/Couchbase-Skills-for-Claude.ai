# Contributing

## Skill format

Every skill follows this structure:

```
<skill-name>/
  SKILL.md
  references/       # optional
    <topic>.md
```

### SKILL.md

```markdown
---
name: skill-name-kebab-case
description: "Single paragraph under 1024 characters. Starts with a verb.
  Lists trigger keywords. States what it's distinct from."
license: MIT
---

# Title

One-paragraph orientation.

## When this skill applies
- "User question that triggers this skill"
- "Another trigger phrase"

## Pick the right reference
| Question | Read |
|---|---|
| "What the user asks" | `references/filename.md` |

## Core principles / rules / tool map
...

## Related skills
- `other-skill` — one line on the relationship
```

**Description field rules:**
- Write in third person; never first or second person ("I can help you...", "You can use this to..." break skill discovery — see `SKILL_FILE_FORMAT_REFERENCE.md`). An imperative opener ("Design...", "Operate...", "Execute...") is the house pattern and satisfies this.
- Put the key use case in the first sentence (it survives truncation).
- Must include explicit "Distinct from" or "Use proactively for" to prevent false triggers
- Hard limit: 1024 characters. Check with `wc -m <<< "your description"`

### Reference files

- Self-contained — readable without the SKILL.md or other references
- 80–400 lines is the target range
- Keep references one level deep: every reference file must link directly from SKILL.md, never only through another reference (no `SKILL.md → a.md → b.md` chains — Claude may only partially read a transitively-reached file). A reference *pointing to* a sibling ("see `error-handling.md`") is fine as long as both are also linked from SKILL.md.
- End with a decision tree or quick-reference table where it helps

## Adding a new skill

1. Pick the right directory: `skills/couchbase/` for core Couchbase, `skills/couchbase-analytics/` for Analytics MCP
2. Create `skills/<group>/<skill-name>/SKILL.md` using the format above
3. Add `references/*.md` if the skill body would exceed ~150 lines
4. Add a `## Related skills` section
5. Add the skill to the table in `README.md`
6. Commit with a clear message: `Add couchbase-<name> skill: <one-line summary>`

## Editing an existing skill

- Edit the file directly — no amendment documents
- If you add a reference file, add a row to the routing table in `SKILL.md`
- Keep the description under 1024 chars after your edit
- Update `## Related skills` in any sibling skills affected by the change

## Version-sensitive content

Gate version-specific features explicitly:

```markdown
Available in Couchbase 7.1+.
8.0+ only (EE only — not available in Community Edition).
```

Do not remove guidance for older versions — both 7.x and 8.x are in active use.

## What to avoid

- Specific version pins for third-party tools (Filebeat 8.x, Prometheus 2.x) — they age badly
- Cloud provider opinions or recommendations
- Pricing information — always changes
- Exact API response payloads that may change between patch versions

## Commit style

```
Add couchbase-fts skill: FTS index design, query types, vector search, analyzers
Fix couchbase-mcp: update tool counts to 167/151/17 (from 164/148/16)
Update couchbase-sqlpp-tuning: add 8.x AUS notes to CBO reference
```
