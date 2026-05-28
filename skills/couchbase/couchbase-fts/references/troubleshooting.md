# FTS troubleshooting

## Diagnostic workflow

When FTS isn't working as expected, work through this sequence before diving into specifics:

1. **`admin_fts_get_index_stats(index_name)`** — is the index healthy? Check `doc_count` (expected number of docs), `num_mutations_to_index` (should be near 0 when caught up), and `status` (should be `Ready`).
2. **`admin_fts_list_indexes()`** — does the index exist? Right bucket/scope/collection?
3. **`cb_fts_search` with a simple `{"match_all": {}}`** — does the index return any results at all?
4. **`cb_fts_search` with `"explain": true`** — for a specific known document ID, why did it or didn't it match?

## Index not returning expected documents

### "No results" for a document you know exists

**Check 1: Is the document in the right scope/collection?**
The index sourceParams must match the scope and collection the document is in. List the index definition and compare to where the document lives.

**Check 2: Is the field indexed?**
If using static mapping (`default_mapping.enabled: false`), the field must be explicitly listed in the type mapping. A typo in the field name causes it to be silently skipped.

**Check 3: Is `doc_config.type_field` matching?**
If your index uses `type_field` doc_config mode, the document must have that field with the value you mapped. If the field is missing or the value doesn't match any type mapping, the document is indexed under `default_mapping` — which may be disabled.

**Check 4: Analyzer mismatch.**
You're using a `term` query (no analysis applied) but the field was indexed with a lowercasing analyzer. `term: "Active"` misses documents indexed as `"active"`. Either lowercase your term query value, or use a `match` query instead (which applies the field's analyzer at query time).

**Check 5: Index not caught up.**
`num_mutations_to_index` > 0 means the index is behind. Wait for it to drain before declaring a document missing.

### "Fewer results than expected" for a text query

**Check 1: Operator is AND when you need OR.**
`match` defaults to OR. If you explicitly set `"operator": "and"`, all terms must be present. A query for "quick brown fox" with AND won't match a document that only has "quick fox."

**Check 2: Stopwords removed from query.**
The `en` analyzer removes common stop words. If the user's query is "is this a problem," it may be reduced to just "problem" after analysis. `match_phrase` is more fragile here.

**Check 3: Stemming mismatch.**
With `en` analyzer, "libraries" stems to "librari" and "library" also stems to "librari" — these match. But "library" and "lib" do not match (different stems). If you expect "lib" to match "library," you need fuzzy search or ngrams.

## Index not updating / stale results

**Check `num_mutations_to_index`.**
A non-zero and growing value means the FTS indexer is falling behind the mutation rate. Causes:

- Indexing too many fields (dynamic mapping, large documents) — reduce the index scope
- FTS service under-resourced (too little RAM or CPU) — check FTS node stats
- A backfill in progress (bulk load or rebalance adding data faster than FTS can consume) — normal; wait for it to drain

**Check for index paused.**
`admin_fts_list_indexes` includes an `isPaused` field. If true, use `admin_fts_resume_index_ingestion` to resume.

## Slow queries

**Use `explain: true` to see score breakdown and term matching.** Expensive at scale — use only in development.

**Check field cardinality.**
Wildcard and regexp queries (`*`, `?`, `.*`) expand to many terms at query time. On high-cardinality fields (millions of unique terms), this is slow. Rewrite as prefix queries if possible; prefix queries use the index's term dictionary efficiently.

**Reduce `k` for vector queries.**
kNN with a large `k` scans more index nodes. Start with `k = 3-5 × size` — don't set `k = 10000` to "be safe."

**Check FTS RAM pressure.**
`admin_stats_fts` returns the FTS service's memory usage. If the service is near its memory quota, it starts spilling to disk (or throttling), which degrades query latency. Increase the FTS service memory quota or add FTS nodes.

## Common error messages

| Error | Cause | Fix |
|---|---|---|
| `index not found` | Wrong index name or the index was deleted | `admin_fts_list_indexes` to get the real name |
| `no planPIndexes for indexName` | Index has no active pindexes — still building, or a node issue | Check `admin_fts_get_index_stats` status field |
| `timeout` | Query took too long | Simplify the query; check resource pressure; use `admin_fts_get_index_stats` to confirm ingestion isn't running a full rebuild |
| `bleve error: field not analyzed for geopoint` | Field mapped as text but used in a geo query | Fix the index definition — field type must be `geopoint` |
| `NumVecs is empty` | Vector query sent to an index with no vector mapping | Ensure the index has the vector field mapped and a vector index exists |

## Rebuilding an index

Sometimes the right answer is a clean rebuild:

1. `admin_fts_delete_index(index_name)` — removes the index (no document data lost, just the search index)
2. Fix the definition
3. `admin_fts_upsert_index(...)` — recreate with corrected definition
4. Monitor `admin_fts_get_index_stats` until `num_mutations_to_index` drops to ~0

Rebuilds can be disruptive. On large collections, FTS indexing can take minutes to hours. If the index is in production, create a new index with a different name, verify it, then swap the query target and delete the old one.
