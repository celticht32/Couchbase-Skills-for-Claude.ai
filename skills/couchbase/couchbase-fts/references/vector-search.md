# Vector search

## What it is

Vector search (kNN — k-nearest neighbours) finds documents whose stored embedding vectors are closest to a query vector in high-dimensional space. The typical use case is semantic search: convert a query to an embedding using a language model, then find documents whose embeddings are nearby.

Available in Couchbase 8.0+ via dedicated vector indexes (`admin_vector_index_*` tools). Late 7.x releases had experimental vector support inside FTS indexes — the 8.x approach is separate and more capable.

## Document structure for vector search

Store the embedding as a JSON array of floats in your document:

```json
{
  "id": "doc::001",
  "title": "Introduction to Couchbase",
  "description": "A primer on the Couchbase NoSQL platform...",
  "embedding": [0.023, -0.147, 0.891, ...]  // 1536 floats for text-embedding-ada-002
}
```

Guidelines:
- The array must be a flat list of numbers — no nested arrays, no nulls.
- All vectors in the collection must have the same dimension. Mixed dimensions break the index.
- Don't store embeddings in deeply nested sub-documents — the vector index path must be addressable from the document root.

## Creating a vector index (8.x)

Use `admin_vector_index_create`:

```python
admin_vector_index_create(
    index_name="products-vector-idx",
    bucket="my-bucket",
    scope="my-scope",
    collection="products",
    field_path="embedding",
    dimension=1536,          # must match your embedding model's output dimension
    similarity="dot_product", # or "euclidean" or "cosine"
    optimized_for="recall",  # or "latency" — see below
    cluster="prod"
)
```

**similarity metric — pick one and use it everywhere:**

| Metric | Use when | Notes |
|---|---|---|
| `dot_product` | Embeddings are unit-normalized (OpenAI, Cohere) | Fastest; equivalent to cosine on normalized vectors |
| `cosine` | Embeddings aren't guaranteed normalized | Slightly slower than dot_product |
| `euclidean` (l2_norm) | Models that use L2 distance (some image models) | Different geometry than dot/cosine |

Match the metric to your embedding model's recommended metric. Using the wrong one produces wrong rankings.

**optimized_for:**
- `recall` — higher accuracy, slightly larger index, slightly slower build. Use for production search quality.
- `latency` — faster queries, slightly lower recall. Use for real-time autocomplete or latency-critical paths.

## Running a kNN query

```python
cb_fts_search(
    index_name="products-vector-idx",
    query={
        "knn": [
            {
                "field": "embedding",
                "vector": [0.023, -0.147, 0.891, ...],  # query embedding, same dimension
                "k": 10  # return top-10 nearest neighbours
            }
        ]
    },
    size=10,
    cluster="prod"
)
```

`k` is the number of candidates to return. The result is ranked by vector similarity score, descending.

## Hybrid search (text + vector)

Combine a traditional FTS query with a kNN query to get results that are both textually relevant and semantically similar:

```python
cb_fts_search(
    index_name="products-hybrid-idx",  # must have both text and vector mappings
    query={
        "knn": [
            {
                "field": "embedding",
                "vector": [...],
                "k": 50,
                "boost": 1.0
            }
        ],
        "query": {
            "match": "fast delivery",
            "field": "description",
            "boost": 0.5
        }
    },
    size=10,
    cluster="prod"
)
```

The scores are combined: `total_score = vector_score * knn_boost + text_score * query_boost`. Tune the boost values to balance semantic vs lexical relevance for your use case.

For hybrid search, the index needs:
- A vector index (`admin_vector_index_*`) for the kNN component
- An FTS index with the text fields mapped for the text component

Or, if using the Couchbase 8.x composite vector index feature, both can live in a single index definition.

## Generating embeddings

The MCP server doesn't generate embeddings — that's your application's job. Common approaches:

- **OpenAI `text-embedding-ada-002` / `text-embedding-3-small`:** 1536 dimensions, normalized, use `dot_product`
- **Cohere `embed-english-v3.0`:** 1024 dimensions, normalized, use `dot_product`
- **Sentence Transformers (local):** Dimensions vary by model; check docs for the right metric

Your indexing pipeline:
1. Generate embedding for each document at write time (in your application or an ETL job)
2. Store the embedding array in the document alongside the text fields
3. At query time, embed the query string with the same model, then call `cb_fts_search` with `knn`

**Important:** always use the same embedding model for indexing and querying. Changing models requires re-embedding every document and rebuilding the vector index.

## Sizing vector indexes

Vector indexes are memory-intensive. A rough formula:

```
Index RAM (GB) ≈ (num_documents × dimension × 4 bytes) × 1.5 overhead factor / 1_073_741_824
```

Example: 1M documents, 1536-dim embeddings:
```
1,000,000 × 1536 × 4 × 1.5 / 1e9 ≈ 9.2 GB
```

Plan dedicated FTS/Search nodes with enough RAM. Vector index data isn't served from Data service RAM — it lives in the FTS/Search service's memory quota.

## Limitations and gotchas

- **Dimension limit:** 2048 dimensions max in current releases. Models producing larger vectors (some multimodal models) can't be used directly — consider dimensionality reduction.
- **No null vectors:** documents missing the embedding field are excluded from vector search results. Make sure every document in the collection has the field, or use a partial FTS index to scope to documents that have it.
- **Index rebuild on dimension change:** there is no in-place resize. If you switch embedding models, drop and recreate the index, re-embed all documents.
- **k is not the result count:** `k` is the number of candidates the index considers. The final `size` parameter trims the result. Set `k >= size` (and typically `k = 3-5x size`) for reasonable recall.
