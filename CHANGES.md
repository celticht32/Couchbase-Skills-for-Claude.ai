# Couchbase Skills — update set (full deep-scan + new coding-standards skill + governance fix)

Unzip into the root of your local `Couchbase-Skills-for-Claude.ai` checkout; mirrors repo
layout, overwrites modified files, adds the new skill. Then `git add -A && commit && push`.

24 files: 18 modified + 6 new. Accelerator excluded throughout.

## Deep scan coverage
Read/swept all 30 skills (124 -> 130 files). codespell across the whole repo: zero real
misspellings (only false positives: "myocardial infarction", "AKS", deliberate "usre" example).
Web-verified every version-sensitive or numeric claim. All 30 descriptions <1024. All internal
reference links resolve (incl. cross-skill pointers).

## NEW SKILL: couchbase-coding-standards (6 files)
Coding standards + style guide blending general clean-code discipline with Couchbase
conventions. Registered in README (table + counts 30/130) and CLAUDE.md (22 core).
SKILL.md + 5 refs: general-coding-standards, couchbase-sdk-idioms,
key-and-document-conventions, error-handling-and-resilience, sqlpp-in-code. SDK symbols
verified vs current 3.x/4.x, each Couchbase-specific ref carries a "verify vs pinned SDK
version" caveat.

## GOVERNANCE DOC FIX
- `CONTRIBUTING.md` — brought into line with the authoritative SKILL_FILE_FORMAT_REFERENCE.md
  on two points where CONTRIBUTING was stricter/wrong than the standard:
  - Description rule: was "Must start with a verb phrase"; standard's actual rule is third
    person (never first/second person). Reworded to that, noting imperative openers are the
    house pattern that satisfies it. Added the "key use case first" (truncation) point.
  - Reference cross-links: was a blanket "No cross-references to other reference files";
    standard only prohibits nested chains (a ref reachable ONLY through another). Reworded to
    "keep references one level deep" — sibling pointers are fine when both link from SKILL.md.
    (This is what the coding-standards references already do; they comply with the standard.)

## FIXED THIS PASS (skill content)
- `couchbase-magma/SKILL.md` — FIXED couchstore min RAM contradiction (table 256 MB vs body
  100 MB). Correct min is 100 MB (256 was "recommended"). Added verified memory-to-data ratios:
  couchstore ~10%, Magma ~1%. Confirmed Magma EE-only / couchstore is the only CE engine.
- `couchbase-transactions/SKILL.md` + `couchbase-mcp/references/data-plane.md` — removed the
  unverifiable "3-5x slower" transaction multiplier (format-ref bars unverifiable figures);
  replaced with qualitative "meaningfully slower — staging + two-phase commit".

## CARRIED FROM PRIOR PASSES (still not on GitHub HEAD)
- README/CLAUDE counts; couchbase-mcp two-server distinction; columnar->Capella Analytics
  rename; ai-applications AI Data Plane / Agent Memory / Agent Catalog note + HVI benchmark
  attribution; eventing vectorization batching + DCP dedup; fts HNSW->SVI/CVI/HVI algorithm
  fix + 2048->4096 dimension limit; mobile edge/mobile GA wave + Lite vector 2.0->3.2.

## Verified clean, left alone
- analytics family (cb-analytics-*): no stale naming/version. (Enterprise Analytics 2.2
  Iceberg/Trino is a separate product surface — NEW scope, not a staleness fix.)
- kubernetes CAO 2.9; sizing rules-of-thumb ranges (legit estimates); capella/backup/
  observability/performance/security-hardening.

## Flagged, NOT asserted
- SDK symbols: "verify vs pinned version" caveats in the new skill.
- Columnar JDBC scheme string; Agent Memory API signatures; Lite lazy-index throughput.
- Transaction slowdown: qualitative, not a multiplier.

## NOT touched
- Accelerator content — excluded everywhere. couchbase-migration-execution/* — unchanged.
