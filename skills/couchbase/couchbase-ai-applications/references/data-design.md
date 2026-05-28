# Data design for AI applications

## Document structure patterns

### Pattern 1: Embedded embedding (most common)

Store the embedding vector directly in the source document:

```json
{
  "id": "product::p-001",
  "type": "product",
  "title": "Wireless Noise-Cancelling Headphones",
  "description": "Premium over-ear headphones with 30-hour battery life...",
  "category": "electronics",
  "brand": "SoundPro",
  "price": 249.99,
  "status": "active",
  "embedding": [0.023, -0.147, 0.891, ...],   // 1536 floats
  "embedding_model": "text-embedding-3-small",  // track the model
  "embedding_updated_at": "2026-01-15T10:30:00Z"
}
```

Good for: product search, document retrieval, entity search where documents are self-contained.

### Pattern 2: Chunked documents (RAG)

For long documents (articles, PDFs, knowledge base entries), split into chunks. Store the parent document and child chunks separately:

```json
// Parent document
{
  "id": "doc::kb-001",
  "type": "knowledge_article",
  "title": "Configuring Couchbase XDCR",
  "source_url": "https://docs.example.com/xdcr",
  "last_updated": "2026-03-10"
}

// Child chunk documents
{
  "id": "chunk::kb-001::001",
  "type": "chunk",
  "parent_id": "doc::kb-001",
  "parent_title": "Configuring Couchbase XDCR",
  "chunk_index": 1,
  "chunk_text": "XDCR (Cross-Datacenter Replication) enables...",
  "embedding": [0.012, 0.847, -0.233, ...],
  "embedding_model": "text-embedding-3-small",
  "section": "Introduction"
}
```

**Chunk design decisions:**
- **Chunk size:** 512-1024 tokens is typical. Smaller chunks = more precise retrieval; larger chunks = more context per result. Benchmark with your content.
- **Overlap:** include ~10-15% overlap between chunks to avoid splitting context at chunk boundaries.
- **Parent reference:** always store `parent_id` and key parent metadata (title, URL, date) in the chunk. At query time you'll display these to the user without a separate lookup.
- **Section/heading:** store the document section heading in each chunk. It helps the LLM understand context and improves retrieval quality for structured content.

### Pattern 3: Multi-vector documents

Store multiple embeddings per document for different fields or modalities:

```json
{
  "id": "product::p-001",
  "title": "Wireless Headphones",
  "description": "...",
  "title_embedding": [0.023, ...],       // embedding of title only
  "desc_embedding": [0.891, ...],        // embedding of description only
  "combined_embedding": [0.456, ...],    // embedding of title + description
  "image_embedding": [0.334, ...]        // visual embedding (if multimodal)
}
```

Use multi-vector when different query types need different context (title search vs description search). Requires a separate vector index per embedding field.

## Embedding field naming conventions

Be consistent. Recommended convention:

- Single embedding: `embedding`
- Multiple embeddings: `<field>_embedding` (e.g. `title_embedding`, `body_embedding`)
- Always store `embedding_model` (string, e.g. `"text-embedding-3-small"`) alongside the vector
- Store `embedding_updated_at` (ISO timestamp) to track staleness

When you update your embedding model, `embedding_model` lets you identify which documents need re-embedding without scanning all documents.

## Metadata fields for filtered vector search (CVI)

If using a Composite Vector Index, the fields you'll filter on must be top-level document fields and included in the index definition. Design these deliberately:

```json
{
  "embedding": [...],
  "category": "electronics",          // filter: WHERE category = "electronics"
  "region": "us-east",               // filter: WHERE region = $region
  "status": "active",                // filter: WHERE status = "active"
  "created_at": "2026-01-15",        // filter: WHERE created_at > "2026-01-01"
  "tenant_id": "acme-corp"           // filter: WHERE tenant_id = $tenant_id
}
```

Low-cardinality fields (status, category, region) make the most effective pre-filters. High-cardinality fields (user_id, document_id) are less useful for pre-filtering — use them for post-filtering.

## Embedding pipeline architecture

```
Documents → [Chunking] → [Embedding Model] → [Write to Couchbase]
                                ↑
                         (same model used at query time)
```

**At write time:**
1. Create or update the document in Couchbase (without embedding first)
2. Generate the embedding from the relevant text fields
3. Update the document with the embedding via subdoc `mutate_in` (avoids rewriting the full document)

```python
# Efficient: update only the embedding field
coll.mutate_in(doc_id, [
    SD.upsert("embedding", embedding_vector),
    SD.upsert("embedding_model", "text-embedding-3-small"),
    SD.upsert("embedding_updated_at", datetime.utcnow().isoformat())
])
```

**Batch embedding pipeline:**
For bulk indexing, generate embeddings in batches matching your embedding provider's batch API limits (OpenAI: 2048 inputs/batch, Cohere: 96/batch). Write batches to Couchbase in parallel. Track progress with a `embedding_status` field (`pending`, `complete`, `failed`).

**At query time:**
1. Embed the user's query with the same model
2. Run the vector search
3. Pass retrieved chunks + user query to the LLM

## Stale embedding detection

Embeddings become stale when the source text changes. Track this with a content hash or update timestamp:

```json
{
  "description": "...",
  "description_hash": "sha256:abc123",   // hash of the text that was embedded
  "embedding": [...],
  "embedding_model": "text-embedding-3-small"
}
```

When `description` changes, compare the new hash to `description_hash`. If different, re-embed. Use an Eventing function or CDC pipeline to trigger re-embedding automatically.
