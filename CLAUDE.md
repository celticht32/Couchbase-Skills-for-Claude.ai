# Couchbase Claude Skills — Project Memory

## What this repo is

A collection of Claude skill files for Couchbase. Skills are structured markdown files that
Claude loads before responding to requests in a specific domain. They live under `skills/` and
follow a strict layout: one `SKILL.md` per skill, with optional `references/*.md` for deep-dive
content that the SKILL.md routes to.

## Directory layout

```
skills/
  couchbase/               # Core Couchbase Server / Capella skills (22 skills)
  couchbase-analytics/     # Couchbase Analytics Service / cb-analytics-mcp skills (8 skills)
```

Each skill:
```
<skill-name>/
  SKILL.md          # YAML frontmatter + principles + routing table + decision trees
  references/       # Optional deep-dive docs linked from SKILL.md
    *.md
```

## SKILL.md format

Every SKILL.md starts with YAML frontmatter, then the skill body:

```markdown
---
name: skill-name-here
description: "One paragraph. Starts with what to do (verb phrase). Lists every trigger
  keyword a user might say. Ends with what it's distinct FROM (sibling skills)."
license: MIT
---

# Skill title

Brief orientation paragraph.

## When this skill applies
Bullet list of user questions that should trigger this skill.

## Pick the right reference
| Question | Read |
|---|---|
| "..." | `references/filename.md` |

## Core principles / rules
...

## Related skills
- `other-skill` — one-line description of the relationship
```

## Description field rules

- Hard limit: stay under 1024 characters
- Must start with an action verb ("Design...", "Operate...", "Execute...")
- Must include "Distinct from" or "Use proactively for" language to prevent false triggers
- Must list the key trigger words a user might say

## Reference file rules

- Self-contained: each reference must be readable without needing to read the others
- Decision tree or quick-reference table at the end of each file where appropriate
- No cross-references between reference files (route through SKILL.md instead)
- 80-400 lines is the target range; under 80 is too thin, over 500 starts to dilute focus

## Adding a new skill

1. Create `skills/couchbase/<skill-name>/SKILL.md` (or `couchbase-analytics/`)
2. Follow the SKILL.md format above exactly — frontmatter first, then body
3. Add reference files under `references/` if the skill has more than ~150 lines of content
4. Add a `## Related skills` section cross-referencing sibling skills
5. Add the skill to the table in `README.md` under the correct group
6. Update the directory tree in `README.md` if adding a new skill that was previously marked `(planned)`

## Editing an existing skill

- Make changes directly in the relevant `SKILL.md` or `references/*.md`
- If you add a new reference file, add a row to the "Pick the right reference" table in `SKILL.md`
- Keep the description under 1024 characters — check with `wc -c` if unsure
- Preserve the `## Related skills` section and update it if the change affects cross-references

## Version-sensitive content

Skills cover Couchbase 7.x and 8.x. Always gate version-specific features with a note:
- "Available in 7.1+"
- "8.0+ only"
- "EE only — not available in Community Edition"

Do not remove older-version guidance when adding newer-version content — both may be in use.

## What NOT to put in skills

- Exact API response shapes that may change between patch versions
- Specific version numbers for third-party tools (Filebeat, Prometheus, etc.) — these age badly
- Opinions about which cloud provider to use
- Pricing information — always changes

## Related projects

- `celticht32/MCP-Couchbase` — the Couchbase MCP server that `couchbase-mcp` skill documents
- `celticht32/Couchbase-Analytics-MCP-Server` — the Analytics MCP server that `cb-analytics-*` skills document

## License

MIT — Copyright (c) 2026 Chris Ahrendt
