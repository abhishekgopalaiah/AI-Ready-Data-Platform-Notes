# System Design: AI-Ready Data Platform

**Complete guide to designing RAG systems at different scales**

---

## Table of Contents
1. [Design Principles](#design-principles)
2. [Scale Scenarios](#scale-scenarios)
3. [Architecture Patterns](#architecture-patterns)
4. [Trade-off Analysis](#trade-off-analysis)
5. [Real-World Examples](#real-world-examples)

---

## Design Principles

### 1. **Start Simple, Scale Later**
```
Small (100K docs):
- Single embedding model
- FLAT index
- Simple chunking
- Basic monitoring

Medium (5M docs):
- Batch embedding
- HNSW index
- Semantic chunking
- Advanced monitoring

Large (100M docs):
- Distributed embedding
- IVF index
- Hybrid chunking
- Full observability
```

### 2. **Optimize for Your Constraints**
```
If latency is critical (<100ms):
- Use HNSW index
- Cache popular queries
- Smaller chunks
- Fewer dimensions (384)

If cost is critical (<$100/month):
- Use IVF index
- Batch queries
- Larger chunks
- Compress embeddings

If accuracy is critical (>99%):
- Use HNSW index
- Full dimensions (768)
- Semantic chunking
- Re-embed frequently
```

### 3. **Monitor Everything**
```
Must track:
- Embedding latency (target: 100ms)
- Search latency (target: 50-100ms)
- Index freshness (target: <7 days old)
- Query accuracy (target: >90%)
- Cost per query (target: <$0.001)
```

---

## Scale Scenarios

### Scenario 1: Small Scale (100K Documents)

**Use Case:** Customer support chatbot, internal knowledge base

```
Architecture:
┌─────────────────────────────────────┐
│ Documents (100K)                    │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│ Chunking (500 words)                │
│ Total chunks: 200K                  │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│ Embedding (Vertex AI Gecko)         │
│ Cost: $2.67 (one-time)              │
│ Time: 30 minutes                    │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│ Vector Database (BigQuery)          │
│ Index: FLAT (simple)                │
│ Memory: 600 MB                      │
│ Cost: $5/month                      │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│ Search                              │
│ Latency: 100ms                      │
│ Accuracy: 100%                      │
│ Cost: $0.01/month (1K queries/day)  │
└─────────────────────────────────────┘
```

**Specifications:**
```
Documents: 100K
Chunks: 200K
Dimensions: 768
Index: FLAT
Memory: 600 MB
Latency: 100ms
Cost/month: $5.01
Accuracy: 100%
```

**Implementation:**
```python
# Simple approach
documents = load_documents(100K)
chunks = chunk_documents(documents, size=500)
embeddings = embed_with_vertex_ai(chunks)
store_in_bigquery(embeddings)
search_results = vector_search(query, top_k=3)
```

---

### Scenario 2: Medium Scale (5M Documents)

**Use Case:** E-commerce product search, document retrieval system

```
Architecture:
┌─────────────────────────────────────┐
│ Documents (5M)                      │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│ Chunking (500 words)                │
│ Total chunks: 10M                   │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│ Batch Embedding (Vertex AI)         │
│ Batch size: 100                     │
│ Cost: $133 (one-time)               │
│ Time: 2 hours                       │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│ Vector Database (Pinecone/Weaviate) │
│ Index: HNSW                         │
│ Memory: 38.4 GB                     │
│ Cost: $50/month                     │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│ Search with Caching                 │
│ Latency: 50-100ms                   │
│ Accuracy: 99%+                      │
│ Cost: $3/month (10K queries/day)    │
└─────────────────────────────────────┘
```

**Specifications:**
```
Documents: 5M
Chunks: 10M
Dimensions: 768
Index: HNSW
Memory: 38.4 GB
Latency: 50-100ms
Cost/month: $53
Accuracy: 99%+
```

**Implementation:**
```python
# Batch approach with caching
documents = load_documents(5M)
chunks = chunk_documents(documents, size=500)

# Batch embedding
for batch in chunks.batch(100):
    embeddings = embed_with_vertex_ai(batch)
    store_in_vector_db(embeddings)

# Search with caching
@cache(ttl=3600)
def search(query):
    query_embedding = embed_with_vertex_ai(query)
    results = vector_db.search(query_embedding, top_k=3)
    return results
```

---

### Scenario 3: Large Scale (100M Documents)

**Use Case:** Web search, large knowledge base, enterprise search

```
Architecture:
┌─────────────────────────────────────┐
│ Documents (100M)                    │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│ Distributed Chunking                │
│ Total chunks: 200M                  │
│ Parallelism: 100                    │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│ Distributed Embedding (Spark)       │
│ Cost: $2,667 (one-time)             │
│ Time: 8 hours                       │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│ Distributed Vector DB (Elasticsearch)
│ Index: IVF                          │
│ Memory: 736 GB (distributed)        │
│ Cost: $500/month                    │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│ Search with Multi-level Caching     │
│ Latency: 100-200ms                  │
│ Accuracy: 95-98%                    │
│ Cost: $30/month (100K queries/day)  │
└─────────────────────────────────────┘
```

**Specifications:**
```
Documents: 100M
Chunks: 200M
Dimensions: 768
Index: IVF
Memory: 736 GB (distributed)
Latency: 100-200ms
Cost/month: $530
Accuracy: 95-98%
```

**Implementation:**
```python
# Distributed approach
documents = load_documents_distributed(100M)
chunks = chunk_documents_spark(documents, size=500)

# Distributed embedding
embeddings = embed_with_spark(chunks)

# Distributed vector DB
vector_db = ElasticsearchCluster(
    nodes=10,
    index_type="IVF",
    shards=50
)
vector_db.bulk_insert(embeddings)

# Search with multi-level caching
@cache_l1(ttl=60)  # Local cache
@cache_l2(ttl=3600)  # Redis cache
def search(query):
    query_embedding = embed_with_vertex_ai(query)
    results = vector_db.search(query_embedding, top_k=3)
    return results
```

---

### Scenario 4: Very Large Scale (1B Documents)

**Use Case:** Global search engine, massive knowledge base

```
Architecture:
┌─────────────────────────────────────┐
│ Documents (1B)                      │
│ Distributed across regions          │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│ Regional Chunking                   │
│ Total chunks: 2B                    │
│ Regions: 5                          │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│ Regional Embedding                  │
│ Cost: $26,667 (one-time)            │
│ Time: 24 hours                      │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│ Regional Vector DBs (Sharded)       │
│ Index: IVF + Quantization           │
│ Memory: 3.68 TB (distributed)       │
│ Cost: $2000/month                   │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│ Global Search with Routing          │
│ Latency: 200-500ms                  │
│ Accuracy: 90-95%                    │
│ Cost: $100/month (1M queries/day)   │
└─────────────────────────────────────┘
```

**Specifications:**
```
Documents: 1B
Chunks: 2B
Dimensions: 384 (quantized)
Index: IVF + Quantization
Memory: 3.68 TB (distributed)
Latency: 200-500ms
Cost/month: $2100
Accuracy: 90-95%
```

---

## Architecture Patterns

### Pattern 1: Simple (100K-1M docs)
```
Query → Embedding → Vector Search → Results
```

### Pattern 2: Cached (1M-10M docs)
```
Query → Cache Check → Embedding → Vector Search → Cache Store → Results
```

### Pattern 3: Distributed (10M-100M docs)
```
Query → Load Balancer → Regional Cache → Regional Search → Aggregate → Results
```

### Pattern 4: Advanced (100M-1B docs)
```
Query → Router → Regional Cache → Approximate Search → Re-rank → Results
```

---

## Trade-off Analysis

### Accuracy vs Speed
```
High Accuracy (99%+):
- HNSW index
- Full dimensions (768)
- Semantic chunking
- Latency: 50-100ms

Balanced (95-98%):
- IVF index
- Full dimensions (768)
- Fixed chunking
- Latency: 100-200ms

Fast (90-95%):
- IVF index
- Reduced dimensions (384)
- Large chunks
- Latency: 50-100ms
```

### Cost vs Quality
```
Cheap (<$100/month):
- IVF index
- Reduced dimensions (384)
- Large chunks (1000 words)
- Quarterly re-embedding

Balanced ($100-500/month):
- HNSW index
- Full dimensions (768)
- Medium chunks (500 words)
- Monthly re-embedding

Premium (>$500/month):
- HNSW index
- Full dimensions (768)
- Small chunks (250 words)
- Weekly re-embedding
```

### Memory vs Latency
```
Low Memory (<10GB):
- FLAT index
- Reduced dimensions (384)
- Latency: 100-200ms

Balanced (10-100GB):
- HNSW index
- Full dimensions (768)
- Latency: 50-100ms

High Memory (>100GB):
- HNSW index
- Full dimensions (768)
- Quantization
- Latency: 10-50ms
```

---

## Real-World Examples

### Example 1: Customer Support Chatbot
```
Requirements:
- 100K FAQs
- <200ms latency
- 99% accuracy
- $50/month budget

Solution:
- Chunk size: 500 words
- Index: FLAT
- Dimensions: 768
- Caching: Popular queries
- Cost: $5/month
- Latency: 100ms
- Accuracy: 100%
```

### Example 2: E-commerce Product Search
```
Requirements:
- 5M products
- <100ms latency
- 98% accuracy
- $200/month budget

Solution:
- Chunk size: 500 words
- Index: HNSW
- Dimensions: 768
- Caching: LRU cache
- Cost: $53/month
- Latency: 50-100ms
- Accuracy: 99%+
```

### Example 3: Enterprise Knowledge Base
```
Requirements:
- 100M documents
- <500ms latency
- 95% accuracy
- $1000/month budget

Solution:
- Chunk size: 500 words
- Index: IVF
- Dimensions: 768
- Caching: Multi-level
- Cost: $530/month
- Latency: 100-200ms
- Accuracy: 95-98%
```

---

## Interview Questions on System Design

**Q1: Design a RAG system for 5M documents**
```
Answer:
1. Chunk documents (500 words each)
2. Embed with Vertex AI (HNSW index)
3. Store in vector DB (Pinecone/Weaviate)
4. Add caching for popular queries
5. Monitor freshness (re-embed monthly)
6. Track latency (target: 50-100ms)
7. Optimize cost (target: $50/month)
```

**Q2: How would you scale to 100M documents?**
```
Answer:
1. Use distributed chunking (Spark)
2. Batch embedding (parallel jobs)
3. Use IVF index instead of HNSW
4. Shard vector DB across regions
5. Add multi-level caching
6. Implement approximate search
7. Monitor per-region latency
```

**Q3: What if latency becomes 500ms?**
```
Answer:
1. Measure bottleneck (embedding vs search)
2. If embedding: Use batch processing
3. If search: Add caching or reduce dimensions
4. If network: Use regional replicas
5. If index: Switch to approximate search
6. Monitor and iterate
```

---

## Key Takeaways

1. **Start small, scale later**
2. **Understand your constraints** (latency, cost, accuracy)
3. **Choose index based on scale** (FLAT → HNSW → IVF)
4. **Cache aggressively** (reduces latency & cost)
5. **Monitor everything** (freshness, latency, cost)
6. **Optimize for your use case** (not generic)
7. **Plan for growth** (design for 10x scale)

