# Vector index types (8.0+)

Couchbase 8.0 ships three vector index types. Pick one based on scale, filtering needs, and whether you need hybrid text+vector.

## Decision matrix

| | Search Vector Index (SVI) | Composite Vector Index (CVI) | Hyperscale Vector Index (HVI) |
|---|---|---|---|
| **Service** | FTS / Search | Index Service (GSI) | Index Service (GSI) |
| **Scale** | Up to ~50M vectors | Up to ~500M vectors | Billions of vectors |
| **Memory model** | FTS memory quota | Index Service RAM | Low memory footprint (disk-optimized) |
| **Scalar filters** | Post-filter only | Pre-filter (in index) | Post-filter only |
| **Hybrid text+vector** | Yes — same index | No | No |
| **SQL++ support** | No | Yes — `ANN_DISTANCE()` | No |
| **Query interface** | `cb_fts_search` knn | `cb_query` with `ANN_DISTANCE()` | `cb_fts_search` knn |
| **When to use** | Hybrid search; smaller corpora | Filtered vector search in SQL++ | Billion-scale; low-memory requirement |

## Search Vector Index (SVI) — pre-8.0 approach, still valid

Defined inside an FTS index. Best for hybrid search (vector + text in one query).

```python
# Defined as part of an FTS index definition
# Field type: "vector", dimension: 1536, similarity: "dot_product"
# Then query via cb_fts_search with knn + query combined
```

Use when:
- You need hybrid text+vector search in a single query
- Your corpus is under ~50M vectors
- You're already using FTS for text search on the same collection

## Composite Vector Index (CVI) — filtered vector search in SQL++

Stored in the Index Service alongside GSI indexes. The key differentiator: scalar filters are applied *before* the vector search, not after. This means filtering out 90% of documents before ANN search is cheap and produces accurate results.

```python
admin_vector_index_create_composite(
    index_name="products-cvi",
    bucket="my-bucket",
    scope_name="my-scope",
    collection_name="products",
    vector_field="embedding",
    dimension=1536,
    similarity="COSINE",
    # Include scalar fields for pre-filtering
    include_fields=["category", "region", "status"],
    cluster="prod"
)
```

Query via SQL++:

```sql
SELECT p.title, p.description,
       ANN_DISTANCE(p.embedding, $query_vector, "COSINE") AS distance
FROM `my-bucket`.`my-scope`.products AS p
WHERE p.category = "electronics"
  AND p.status = "active"
ORDER BY ANN_DISTANCE(p.embedding, $query_vector, "COSINE")
LIMIT 10
```

Use when:
- You need to filter by category, region, date range, status, or other fields before vector search
- You want to run vector search in SQL++ alongside other query logic
- Corpus is up to ~500M vectors
- You're already using the Index Service for GSI indexes

**Critical:** the scalar fields used in WHERE clauses must be included in the index definition. A filter on a field not in the index falls back to post-filtering, losing the performance benefit.

## Hyperscale Vector Index (HVI) — billion-scale

Stored in the Index Service with a disk-optimized storage format. Trades some query speed for massive scale and low memory footprint. Independent benchmarks show 19,000 QPS at 28ms latency at 66% recall (tunable).

```python
admin_vector_index_create_hyperscale(
    index_name="corpus-hvi",
    bucket="my-bucket",
    scope_name="my-scope",
    collection_name="documents",
    vector_field="embedding",
    dimension=1536,
    similarity="COSINE",
    # Recall vs latency tradeoff — tune for your workload
    recall_target=0.90,     # 0.0-1.0; higher = slower build + slower query
    cluster="prod"
)
```

Query via `cb_fts_search` (same interface as SVI):

```python
cb_fts_search(
    index_name="corpus-hvi",
    query={"knn": [{"field": "embedding", "vector": [...], "k": 20}]},
    size=10,
    cluster="prod"
)
```

Use when:
- Corpus exceeds ~500M vectors
- Memory is constrained (HVI is disk-optimized, not RAM-resident)
- You can tolerate slightly higher query latency vs CVI
- You don't need pre-filtered vector search (use post-filtering or accept the recall tradeoff)

## Recall tuning (HVI)

`recall_target` controls the speed/accuracy tradeoff during index construction:

| recall_target | QPS (approx) | Latency | Use case |
|---|---|---|---|
| 0.66 | ~19,000 | 28ms | High-throughput, precision less critical |
| 0.90 | ~700 | < 1s | Balanced — most production workloads |
| 0.99 | < 100 | Seconds | Maximum accuracy requirement |

Start at 0.90 for production. Lower if latency is critical; raise if retrieval quality is critical.

## Migration path from pre-8.0

If you have SVI indexes from pre-8.0 and want to move to CVI or HVI:

1. Create the new index alongside the existing one
2. Wait for the new index to build fully (monitor via `admin_vector_index_list`)
3. Update your application to query the new index
4. Drop the old SVI once traffic is migrated

There is no in-place migration — index type is set at creation time.
