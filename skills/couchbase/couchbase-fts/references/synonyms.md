# FTS synonyms (8.x)

## What they are

FTS synonym sets let you define equivalences between terms so queries automatically expand to include synonyms. Available in Couchbase 8.0+ via `cb_fts_synonym_*` tools.

Two types:

**Equivalent synonyms:** terms that are fully interchangeable. Querying any of them matches documents containing any of them.
```
laptop, notebook, portable computer
```

**Explicit mapping (one-directional):** queries for the left-side term expand to include right-side terms. Right-side queries do NOT expand to the left.
```
iphone => apple smartphone, mobile phone, ios device
```

## Tools

| Tool | Purpose |
|---|---|
| `cb_fts_synonym_upsert(set_name, synonyms, type, cluster)` | Create or replace a synonym set |
| `cb_fts_synonym_list(cluster)` | List all synonym sets |
| `cb_fts_synonym_delete(set_name, cluster)` | Delete a synonym set |

Synonym sets are cluster-wide, not index-scoped. An FTS index must opt in to a synonym set via its index definition (set `"synonyms": ["my-synonym-set"]` in the index params).

## Creating a synonym set

```python
cb_fts_synonym_upsert(
    set_name="product-synonyms",
    synonyms=[
        # Equivalent synonyms
        {"terms": ["laptop", "notebook", "portable computer"]},
        {"terms": ["smartphone", "mobile phone", "cell phone"]},
        # Explicit mapping
        {"terms": ["iphone"], "mapped_terms": ["apple smartphone", "ios device"]},
        {"terms": ["buy", "purchase", "order"]}
    ],
    type="equivalent",  # or "explicit" — applies to whole set
    cluster="prod"
)
```

For mixed equivalent and explicit in one set, create two separate sets.

## Wiring a synonym set to an FTS index

In the index definition's `params`:

```json
"params": {
  "mapping": { ... },
  "analysis": {
    "synonym_sources": {
      "product-synonyms": {}
    }
  }
}
```

Then reference it in the field mapping:

```json
"title": {
  "fields": [{
    "name": "title",
    "type": "text",
    "analyzer": "en",
    "synonyms": "product-synonyms"
  }]
}
```

The synonym expansion happens at query time, not index time. This means you don't need to rebuild the index when you update the synonym set — just update the set and queries immediately use the new synonyms.

## When to use synonyms

**Good use cases:**
- Product search: "TV" should match "television", "flat screen"
- Domain-specific vocabulary: "myocardial infarction" ↔ "heart attack"
- Abbreviations: "US" ↔ "United States", "FAQ" ↔ "frequently asked questions"
- Spelling variants: "colour" ↔ "color" (if not handled by a language analyzer)
- Brand/generic equivalences: product names that users might search by either brand or category

**Not the right tool for:**
- Typo correction → use `fuzziness` in match queries instead
- Stemming ("run" ↔ "running") → handled by language analyzers
- Stop-word removal → handled by analyzers

## Maintenance

Synonym sets are updated in place — an upsert replaces the entire set. To add terms:

1. `cb_fts_synonym_list` — read the current set
2. Add the new terms to the list
3. `cb_fts_synonym_upsert` — write it back

There's no partial-update API. Script management in your deployment tooling if the synonym list is large.

## Limitations

- Synonym expansion happens at query time only. Documents indexed before a synonym was added are still findable — the query expands, not the index.
- Circular expansions are handled correctly (A → B, B → A won't loop infinitely).
- Very large synonym sets (thousands of entries) can add measurable latency to query planning. Benchmark at scale before deploying to hot-path queries.
- Synonym sets are not bucket/scope/collection scoped — they're cluster-global. Name them clearly to avoid collisions across teams.
