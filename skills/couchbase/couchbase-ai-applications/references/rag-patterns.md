# RAG patterns

## RAG pipeline overview

Retrieval-Augmented Generation (RAG) grounds LLM responses in your data by retrieving relevant documents at query time and including them in the prompt.

```
User Query
    ↓
[Embed query]
    ↓
[Vector search in Couchbase] ──→ Top-K chunks
    ↓
[Optional: rerank / filter]
    ↓
[Build prompt: system + chunks + user query]
    ↓
[LLM generates response]
    ↓
Response to user
```

## Retrieval strategies

### Pure vector search

Embed the query, search by vector similarity only. Good for semantic questions where keywords may not match the answer.

```python
query_embedding = embed("How do I configure XDCR filtering?")

results = cluster.search(
    "chunks-svi",
    VectorSearch.from_vector_query(
        VectorQuery("embedding", query_embedding, num_candidates=20)
    ),
    SearchOptions(limit=5, fields=["chunk_text", "parent_title", "section"])
)
```

### Hybrid search (vector + keyword)

Combines vector similarity with BM25 keyword scoring. Better recall than either alone — catches both semantic and lexical matches. Use when queries may include exact terms (product names, error codes, technical terms).

```python
# Via cb_fts_search — SVI + text in one index
cb_fts_search(
    index_name="chunks-hybrid-idx",
    query={
        "knn": [{"field": "embedding", "vector": query_vec, "k": 40, "boost": 1.0}],
        "query": {"match": user_query, "field": "chunk_text", "boost": 0.5}
    },
    size=10
)
```

### Filtered vector search (CVI)

Pre-filter by metadata before vector search. Dramatically improves results for multi-tenant or domain-partitioned corpora.

```sql
-- Only search within the user's tenant and the relevant product area
SELECT c.chunk_text, c.parent_title, c.section,
       ANN_DISTANCE(c.embedding, $query_vec, "COSINE") AS score
FROM `kb`.`default`.chunks AS c
WHERE c.tenant_id = $tenant_id
  AND c.product_area = $product_area
  AND c.status = "published"
ORDER BY ANN_DISTANCE(c.embedding, $query_vec, "COSINE")
LIMIT 10
```

## Chunking strategies

**Fixed-size chunking.** Split at every N tokens with M-token overlap. Simple, predictable. Works for unstructured prose. Typical: 512 tokens, 64-token overlap.

**Semantic chunking.** Split at sentence or paragraph boundaries rather than token count. Better coherence; variable chunk sizes. Use a sentence splitter library.

**Document-structure chunking.** Split at section headings for structured documents (documentation, reports). Each chunk = one section. Include the heading in the chunk text and as a metadata field.

**Hierarchical chunking.** Index both the full document and its chunks. At query time, retrieve chunks; at generation time, optionally fetch the parent document for broader context. Useful for long documents where the LLM needs surrounding context.

## Prompt construction

```python
def build_rag_prompt(user_query: str, retrieved_chunks: list[dict]) -> list[dict]:
    context_parts = []
    for i, chunk in enumerate(retrieved_chunks, 1):
        context_parts.append(
            f"[Source {i}: {chunk['parent_title']} — {chunk['section']}]\n"
            f"{chunk['chunk_text']}"
        )

    context = "\n\n".join(context_parts)

    return [
        {
            "role": "system",
            "content": (
                "You are a helpful assistant. Answer the user's question using only "
                "the provided sources. If the sources don't contain the answer, say so. "
                "Cite sources by their [Source N] label.\n\n"
                f"Sources:\n{context}"
            )
        },
        {
            "role": "user",
            "content": user_query
        }
    ]
```

Key prompt principles:
- Tell the LLM to use only the provided sources (reduces hallucination)
- Include the source title and section so the LLM can cite them
- Instruct the LLM to say "I don't know" when sources are insufficient
- Cap the number of chunks (5-10 is typical); too many chunks dilutes focus

## Reranking

After vector retrieval, reranking re-scores the results with a more expensive but more accurate cross-encoder model. Improves precision when initial retrieval quality is insufficient.

```python
# Retrieve more candidates than you need
initial_results = vector_search(query, k=40)

# Rerank with a cross-encoder (Cohere, Jina, Voyage, or local model)
reranked = cohere_client.rerank(
    model="rerank-english-v3.0",
    query=user_query,
    documents=[r["chunk_text"] for r in initial_results],
    top_n=5
)
```

Add reranking when:
- Initial retrieval precision is below ~70% on your evaluation set
- Queries are complex or ambiguous
- The cost of a wrong answer is high

## Retrieval evaluation

Before going to production, evaluate retrieval quality:

1. Build a test set of 50-100 question + expected-source pairs
2. Run your retrieval pipeline on each question
3. Measure recall@K: what fraction of questions have the correct source in the top K results?
4. Target recall@5 > 0.80 for production RAG

If recall is low:
- First check: is the correct chunk actually in the index? (examine the chunk text)
- Try hybrid search if you're using pure vector
- Try smaller chunk size if chunks are too broad
- Try a better embedding model
- Add reranking as a last resort (expensive)

## AI agent memory patterns

Couchbase works well as persistent memory for AI agents:

**Episodic memory (conversation history):**
```json
{
  "id": "memory::user-123::session-456::001",
  "type": "episode",
  "user_id": "user-123",
  "session_id": "session-456",
  "role": "user",
  "content": "How do I configure XDCR?",
  "timestamp": "2026-05-28T10:00:00Z",
  "embedding": [...]   // embed for semantic recall
}
```

**Semantic memory (long-term knowledge extracted from conversations):**
```json
{
  "id": "memory::user-123::fact::001",
  "type": "semantic_memory",
  "user_id": "user-123",
  "content": "User is a DBA managing a 3-node Couchbase 8.0 cluster on AWS.",
  "extracted_at": "2026-05-28T10:05:00Z",
  "embedding": [...]
}
```

At the start of each agent turn, retrieve relevant memories via vector search and include them in the system prompt alongside RAG context.
