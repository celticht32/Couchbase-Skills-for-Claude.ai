# Skill-File Format Reference

Authoritative conventions for authoring Claude skills in this repo. Reconciles Celtic Heart / celticht32 house style against Anthropic's official guidance, verified 2026-06-30 against:

- Claude Code skills doc — https://code.claude.com/docs/en/skills
- Skill authoring best practices — https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices

Everything below is grounded in one of those two sources or flagged explicitly as house style. Unverifiable claims from third-party guides (specific accuracy percentages, named research papers, "3.5x" figures) have been deliberately excluded. Where a rule is house style rather than Anthropic-official, it is labelled **[house]**.

## Pick the right reference

| You are doing this | Read this section |
|--------------------|-------------------|
| Writing frontmatter (name, description, fields) | Frontmatter |
| Deciding what goes in SKILL.md vs a reference file | Progressive disclosure |
| Structuring a multi-step task | Workflows and feedback loops |
| Writing paths inside a skill (Windows caveat) | Paths — always forward slashes |
| Bundling executable scripts | Scripts |
| Knowing when a skill is done | Evaluation |
| House conventions specific to this repo | House conventions |

## Frontmatter

Two fields matter. `name` is optional (defaults to directory name); `description` is the one that does the work.

```yaml
---
name: analyzing-spreadsheets
description: >
  Analyze Excel spreadsheets, build pivot tables, generate charts.
  Use when analyzing Excel files, tabular data, or .xlsx files.
license: MIT
---
```

Rules, verified:

- **`name`**: lowercase letters, numbers, hyphens only. Max 64 characters. No XML tags. Cannot contain the reserved words "anthropic" or "claude". Prefer gerund form (`processing-pdfs`, `analyzing-spreadsheets`, `managing-databases`) — Anthropic's recommended convention. Noun phrases (`pdf-processing`) and action forms (`process-pdfs`) are acceptable alternatives. Avoid vague names (`helper`, `utils`, `tools`).
- **`description`**: non-empty, max 1024 characters (platform/API hard limit), no XML tags. Write in **third person** — "Processes Excel files and generates reports", never "I can help you..." or "You can use this to...". The description is injected into the system prompt for skill selection, so first-person phrasing causes discovery problems. Include both *what it does* and *when to use it* (trigger terms, contexts, file extensions).
- **The Claude Code truncation caveat**: in Claude Code specifically, the combined `description` + `when_to_use` text is truncated at **1,536 characters** in the skill listing, and the whole listing lives under a budget (~1% of the context window, tunable via `skillListingBudgetFraction`). When many skills are loaded, low-priority descriptions are shortened or dropped. **Put the key use case in the first sentence** so it survives truncation. Run `/doctor` in Claude Code to see which skill descriptions are being shortened.

Optional Claude Code frontmatter fields worth knowing (all optional):

| Field | Use |
|-------|-----|
| `when_to_use` | Extra trigger phrases; appended to description, counts toward the 1,536-char cap |
| `disable-model-invocation: true` | Only the user can invoke via `/name`; keeps the skill out of auto-context. Use for side-effecting workflows (deploy, commit, send-message) |
| `user-invocable: false` | Only Claude can invoke; hides from the `/` menu. Use for background-knowledge skills that aren't a meaningful user action |
| `allowed-tools` | Pre-approve specific tools while the skill is active (does not restrict others) |
| `context: fork` | Run the skill in an isolated subagent; SKILL.md body becomes the subagent prompt |
| `paths` | Glob patterns that gate auto-activation to matching files |

## Progressive disclosure

The core structural principle. Three levels, verified against Anthropic's model:

| Level | What | When loaded |
|-------|------|-------------|
| 1 | `name` + `description` | Always, pre-loaded at startup |
| 2 | SKILL.md body | When the skill triggers |
| 3 | `references/` files | On demand, only when the body points Claude to them |

Rules, verified:

- **Keep the SKILL.md body under 500 lines.** Stated twice in Anthropic's docs. Split into reference files when approaching the limit.
- **Concise is non-negotiable.** Once loaded, the body stays in context and every line is a recurring token cost. Default assumption: Claude is already smart. Cut any explanation of things Claude already knows (what a PDF is, how a library works). Challenge each line: "does this justify its token cost?"
- **Keep references one level deep from SKILL.md.** Do not chain `SKILL.md → advanced.md → details.md`. Claude may only partially read (`head -100`) files reached through nested references, producing incomplete information. Every reference file links directly from SKILL.md.
- **Reference files over ~100 lines get a table of contents at the top**, so a partial read still shows the full scope.
- **No context penalty for large bundled files** until they're actually read — so bundling complete API docs, large examples, or datasets in `references/` is fine and encouraged, as long as they're only pulled in when needed.

Three organizing patterns from Anthropic, use whichever fits:

1. **High-level guide with references** — quick-start in the body, "for X see [X.md]" links for the rest.
2. **Domain-specific organization** — one reference file per domain (`reference/finance.md`, `reference/sales.md`), so a query about one domain never loads the others.
3. **Conditional details** — show basic content inline, link advanced/edge-case content out.

## Workflows and feedback loops

For multi-step tasks:

- **Break complex operations into explicit sequential steps.** For particularly complex workflows, provide a checklist Claude can copy into its response and tick off. This prevents skipped validation steps.
- **Implement validate → fix → repeat loops** for quality-critical work. The "validator" can be a script *or* a reference doc (e.g. a STYLE_GUIDE.md that Claude reads and checks against). Explicitly gate progress: "only proceed when validation passes."
- **Match specificity to task fragility** (Anthropic's "degrees of freedom"):
  - High freedom (prose instructions) when multiple approaches are valid and context decides — e.g. code review.
  - Medium freedom (parameterized scripts/templates) when a preferred pattern exists with acceptable variation.
  - Low freedom (exact scripts, "run exactly this, do not modify") when operations are fragile and consistency is critical — e.g. migrations.
- **For high-stakes batch operations, use plan → validate → execute.** Have Claude write a structured plan file (e.g. `changes.json`), validate it with a script, then execute. Catches errors before they're applied.
- **Use consistent terminology throughout.** Pick one term ("field", "extract", "API endpoint") and never drift to synonyms — inconsistency degrades instruction-following.
- **Avoid time-sensitive information.** Don't write "before August 2025, use the old API". Put deprecated guidance in a collapsed "Old patterns" section instead.
- **Don't offer many options.** Give one default with an escape hatch ("use pdfplumber; for scanned PDFs use pdf2image instead"), not a menu of five libraries.

## Paths — always forward slashes

**Inside a SKILL.md, all file paths use forward slashes — `scripts/helper.py`, `reference/guide.md` — even though the author works on Windows.** This is an explicit Anthropic anti-pattern: backslash paths break on Unix runtimes where skills execute.

This is the one place the standing Windows-cmd/PowerShell house preference does **not** apply. Shell commands *given to the user for their own machine* still use Windows syntax; paths *written into a skill file* are always Unix-style.

## Scripts

For skills that bundle executable code:

- **Solve, don't punt.** Handle error conditions inside the script (FileNotFoundError, PermissionError) rather than letting it fail and expecting Claude to recover.
- **No voodoo constants.** Every config value gets an inline justification comment. "If you don't know the right value, how will Claude?" (Ousterhout's law.)
- **Prefer pre-made scripts over generated code** for deterministic operations — more reliable, saves tokens, ensures consistency.
- **Make execution intent explicit**: "Run `analyze_form.py` to extract fields" (execute) vs "See `analyze_form.py` for the algorithm" (read as reference). Execution is preferred for utility scripts — the script runs via bash and only its output consumes tokens.
- **List required packages explicitly.** Don't assume anything is installed. Note the environment limits: claude.ai code execution can install from npm/PyPI; the Claude API code execution has no network access and no runtime install.
- **Use fully-qualified MCP tool names** in skill instructions — `ServerName:tool_name` (e.g. `BigQuery:bigquery_schema`), never the bare tool name, or Claude may fail to locate it when multiple servers are connected.

## Evaluation

Seeing a skill trigger only tells you Claude found it, not that it worked. Verified guidance:

- **Build evaluations before writing extensive documentation.** Anthropic's strongest process claim. Run Claude on representative tasks *without* the skill, document the specific failures, write minimal instructions to fix exactly those, then iterate. This solves real gaps rather than imagined ones.
- **Minimum three test scenarios**, each with a query, input files, and expected-behavior assertions. Measure baseline without the skill, compare with it. Use a fresh session each time — leftover authoring context masks gaps in the written instructions.
- **The `skill-creator` plugin automates this loop in Claude Code.** Install: `/plugin install skill-creator@claude-plugins-official`. It stores test cases in `evals/evals.json`, runs each in an isolated subagent, grades assertions with evidence, benchmarks with-skill vs without (pass rate, tokens, time), does blind A/B between two skill versions, and tunes the description by measuring should-trigger vs should-not-trigger hit rates. This is the concrete tool for maintaining a multi-skill suite.
- **The Claude A / Claude B authoring loop**: use one Claude instance to author and refine the skill, a fresh instance to test it on real tasks, and bring observed gaps back to the author instance. Claude natively understands the skill format — no special "skill-writing" prompt is needed.
- **Test across every model you'll run the skill on.** What's sufficient for Opus may under-specify for Haiku.

## House conventions

These are Celtic Heart / celticht32 repo conventions, **[house]** — compatible with Anthropic's guidance but not part of it. Keep them; just don't present them as the official standard when sharing skills externally.

- **`license: MIT`** in frontmatter, Copyright (c) 2026 Chris Ahrendt, unless otherwise specified. (Note: Anthropic's own bundled skills use their own license; this applies to skills authored in this repo.)
- **`## Pick the right reference` routing table** near the top of any multi-file skill — the table this document opens with is the template.
- **`## Related skills` section** linking sibling skills in the suite.
- **Reference files under `references/`**, self-contained, 80–400 lines each. (Anthropic says "one level deep, TOC over 100 lines, under 500-line body" — the 80–400 band is the tighter house target within that.)
- **Don't create empty `references/` directories.** A single-file skill with no routing table is valid when the topic doesn't need splitting.
- **Verify version-sensitive symbols against pinned project versions**, never inferred from training data — the standing repo rule applies to skill scripts too.

## Checklist before shipping a skill

Core:

- [ ] Description is third-person, specific, includes trigger terms, key use case first
- [ ] Description under 1024 chars (and reads well if truncated at 1,536 in Claude Code)
- [ ] SKILL.md body under 500 lines
- [ ] Reference files one level deep; TOC on any over 100 lines
- [ ] No time-sensitive info (or quarantined in an "Old patterns" section)
- [ ] Consistent terminology throughout
- [ ] All paths use forward slashes
- [ ] Progressive disclosure applied — heavy content in references/, not the body

Scripts (if any):

- [ ] Scripts handle errors, don't punt to Claude
- [ ] Every constant justified inline
- [ ] Required packages listed; environment install limits noted
- [ ] MCP tools use fully-qualified `Server:tool` names
- [ ] Execution vs read-as-reference intent is explicit

Testing:

- [ ] At least three evaluations written (ideally in `evals/evals.json` via skill-creator)
- [ ] Baseline measured with skill disabled
- [ ] Tested on every model the skill will run on

House:

- [ ] `license: MIT` frontmatter
- [ ] `## Pick the right reference` routing table (multi-file skills)
- [ ] `## Related skills` section
- [ ] No empty `references/` directory
