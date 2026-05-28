# Framework integration

## LangChain

Couchbase has an official LangChain integration package: `langchain-couchbase`.

```bash
pip install langchain-couchbase langchain-openai
```

### Vector store

```python
from langchain_couchbase.vectorstores import CouchbaseVectorStore
from langchain_openai import OpenAIEmbeddings
from couchbase.cluster import Cluster
from couchbase.auth import PasswordAuthenticator
from couchbase.options import ClusterOptions

# Connect to Couchbase
cluster = Cluster(
    "couchbases://your-cluster.cloud.couchbase.com",
    ClusterOptions(PasswordAuthenticator("username", "password"))
)

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

vector_store = CouchbaseVectorStore(
    cluster=cluster,
    bucket_name="my-bucket",
    scope_name="my-scope",
    collection_name="chunks",
    embedding=embeddings,
    index_name="chunks-svi",        # your FTS/SVI index name
)

# Add documents
vector_store.add_texts(
    texts=["chunk text 1", "chunk text 2"],
    metadatas=[{"source": "doc1", "section": "intro"}, {"source": "doc1", "section": "body"}]
)

# Similarity search
results = vector_store.similarity_search("how to configure XDCR", k=5)

# As retriever (for RAG chains)
retriever = vector_store.as_retriever(search_kwargs={"k": 5})
```

### RAG chain with LangChain

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini")

prompt = ChatPromptTemplate.from_messages([
    ("system", "Answer using only the provided context. Context:\n\n{context}"),
    ("human", "{question}")
])

def format_docs(docs):
    return "\n\n".join(doc.page_content for doc in docs)

rag_chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | prompt
    | llm
    | StrOutputParser()
)

answer = rag_chain.invoke("How do I configure XDCR filtering?")
```

### Caching with LangChain + Couchbase

```python
from langchain_couchbase.cache import CouchbaseCache, CouchbaseSemanticCache

# Exact-match cache (fast, for repeated identical queries)
from langchain.globals import set_llm_cache
set_llm_cache(CouchbaseCache(cluster=cluster, bucket_name="cache", ...))

# Semantic cache (cache by similarity — catches paraphrased queries)
set_llm_cache(CouchbaseSemanticCache(
    cluster=cluster,
    embedding=embeddings,
    bucket_name="semantic-cache",
    score_threshold=0.95    # similarity threshold for cache hit
))
```

## LlamaIndex

```bash
pip install llama-index-vector-stores-couchbase
```

```python
from llama_index.vector_stores.couchbase import CouchbaseVectorStore
from llama_index.core import VectorStoreIndex, StorageContext
from llama_index.embeddings.openai import OpenAIEmbedding

vector_store = CouchbaseVectorStore(
    cluster=cluster,
    bucket_name="my-bucket",
    scope_name="my-scope",
    collection_name="chunks",
    index_name="chunks-svi",
)

storage_context = StorageContext.from_defaults(vector_store=vector_store)

# Build index from documents
index = VectorStoreIndex.from_documents(
    documents,
    storage_context=storage_context,
    embed_model=OpenAIEmbedding(model="text-embedding-3-small")
)

# Query
query_engine = index.as_query_engine(similarity_top_k=5)
response = query_engine.query("How do I configure XDCR filtering?")
```

## Direct SDK integration (no framework)

For maximum control or when frameworks add unnecessary overhead:

```python
from couchbase.cluster import Cluster
from couchbase.search import SearchRequest
from couchbase.vector_search import VectorSearch, VectorQuery
import openai

def rag_retrieve(query: str, cluster: Cluster, bucket: str,
                 scope: str, index: str, top_k: int = 5) -> list[dict]:
    # 1. Embed the query
    response = openai.embeddings.create(
        model="text-embedding-3-small",
        input=query
    )
    query_vector = response.data[0].embedding

    # 2. Search
    scope_obj = cluster.bucket(bucket).scope(scope)
    results = scope_obj.search(
        index,
        SearchRequest.create(
            VectorSearch.from_vector_query(
                VectorQuery("embedding", query_vector, num_candidates=top_k * 4)
            )
        ),
        SearchOptions(
            limit=top_k,
            fields=["chunk_text", "parent_title", "section", "parent_id"]
        )
    )

    return [
        {
            "text": r.fields.get("chunk_text", ""),
            "title": r.fields.get("parent_title", ""),
            "section": r.fields.get("section", ""),
            "score": r.score
        }
        for r in results.rows()
    ]
```

## Choosing between framework and direct SDK

| Situation | Recommendation |
|---|---|
| Prototyping / standard RAG pipeline | LangChain or LlamaIndex — faster to build |
| Complex filtering, multi-step retrieval, custom scoring | Direct SDK — full control |
| High-performance production (>1000 QPS) | Direct SDK — remove framework overhead |
| Team already using LangChain/LlamaIndex | Stick with the framework |
| Need LangChain semantic cache | LangChain — it's purpose-built for this |

## Environment-specific notes

**Capella:** Use `couchbases://` (TLS) connection string. Database credentials (separate from cluster admin credentials) for application access. Set allowed CIDRs before connecting.

**Self-managed with TLS:** Pass `ClusterOptions` with `TLSVerifyMode.NO_VERIFY` only for development. In production, provide the CA certificate.
