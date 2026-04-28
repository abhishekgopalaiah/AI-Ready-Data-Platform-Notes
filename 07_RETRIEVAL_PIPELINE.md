# Retrieval Pipeline: Architecture & Optimization

**Complete guide to building and optimizing retrieval pipelines**

---

## Table of Contents
1. [Pipeline Architecture](#pipeline-architecture)
2. [6-Step Workflow](#6-step-workflow)
3. [Latency Optimization](#latency-optimization)
4. [Precision & Recall](#precision--recall)
5. [Production Patterns](#production-patterns)

---

## Pipeline Architecture

### High-Level Flow

```
User Query
    ↓
[1. Parse Query]
    ↓
[2. Generate Query Embedding]
    ↓
[3. Vector Search]
    ↓
[4. Rank Results]
    ↓
[5. Format Response]
    ↓
[6. Log & Monitor]
    ↓
Return Results
```

### Detailed Architecture

```
┌─────────────────────────────────────────────────────────┐
│ User Query: "How do I return an item?"                  │
└────────────────────┬────────────────────────────────────┘
                     ↓
        ┌────────────────────────┐
        │ Query Preprocessing    │
        ├────────────────────────┤
        │ - Normalize text       │
        │ - Remove special chars │
        │ - Lowercase            │
        └────────────┬───────────┘
                     ↓
        ┌────────────────────────┐
        │ Generate Embedding     │
        ├────────────────────────┤
        │ - Vertex AI API        │
        │ - 768 dimensions       │
        │ - Latency: 100ms       │
        └────────────┬───────────┘
                     ↓
        ┌────────────────────────┐
        │ Vector Search          │
        ├────────────────────────┤
        │ - Query vector DB      │
        │ - Top-K=10             │
        │ - Latency: 50-100ms    │
        └────────────┬───────────┘
                     ↓
        ┌────────────────────────┐
        │ Re-rank Results        │
        ├────────────────────────┤
        │ - BM25 scoring         │
        │ - Diversity filter     │
        │ - Top-3 final          │
        └────────────┬───────────┘
                     ↓
        ┌────────────────────────┐
        │ Format Response        │
        ├────────────────────────┤
        │ - Add metadata         │
        │ - Add scores           │
        │ - Add sources          │
        └────────────┬───────────┘
                     ↓
        ┌────────────────────────┐
        │ Log & Monitor          │
        ├────────────────────────┤
        │ - Log latency          │
        │ - Log accuracy         │
        │ - Log cost             │
        └────────────┬───────────┘
                     ↓
┌─────────────────────────────────────────────────────────┐
│ Results: [Doc1, Doc2, Doc3] with scores & metadata      │
└─────────────────────────────────────────────────────────┘
```

---

## 6-Step Workflow

### Step 1: Parse Query

```python
def parse_query(query: str) -> dict:
    """Parse and normalize query"""
    
    # Normalize
    query = query.lower().strip()
    
    # Remove special characters
    query = re.sub(r'[^\w\s]', '', query)
    
    # Extract intent (optional)
    intents = {
        'return': ['return', 'refund', 'exchange'],
        'shipping': ['shipping', 'delivery', 'track'],
        'payment': ['payment', 'charge', 'billing']
    }
    
    intent = None
    for intent_type, keywords in intents.items():
        if any(kw in query for kw in keywords):
            intent = intent_type
            break
    
    return {
        'original': query,
        'normalized': query,
        'intent': intent,
        'length': len(query.split())
    }

# Example
result = parse_query("How do I return an item?")
# {'original': 'how do i return an item?', 'normalized': 'how do i return an item', 'intent': 'return', 'length': 5}
```

### Step 2: Generate Query Embedding

```python
def generate_query_embedding(query: str) -> List[float]:
    """Generate embedding for query"""
    
    from google.cloud import aiplatform
    
    aiplatform.init(project="your-project-id", location="us-central1")
    model = aiplatform.TextEmbeddingModel.from_pretrained(
        "textembedding-gecko@001"
    )
    
    embeddings = model.get_embeddings([query])
    return embeddings[0].values

# Example
query = "How do I return an item?"
embedding = generate_query_embedding(query)
print(f"Embedding shape: {len(embedding)}")  # 768
```

### Step 3: Vector Search

```python
def vector_search(query_embedding: List[float], top_k: int = 10) -> List[dict]:
    """Search for similar documents"""
    
    from google.cloud import bigquery
    
    client = bigquery.Client()
    
    # Convert embedding to string
    embedding_str = ",".join(str(x) for x in query_embedding)
    
    query = f"""
    SELECT
        chunk_id,
        doc_id,
        content,
        1 - COSINE_DISTANCE(embedding, [{embedding_str}]) as similarity
    FROM `project.dataset.embeddings`
    ORDER BY similarity DESC
    LIMIT {top_k}
    """
    
    results = client.query(query).result()
    
    return [
        {
            'chunk_id': row.chunk_id,
            'doc_id': row.doc_id,
            'content': row.content,
            'similarity': row.similarity
        }
        for row in results
    ]

# Example
results = vector_search(embedding, top_k=10)
print(f"Found {len(results)} results")
```

### Step 4: Re-rank Results

```python
def rerank_results(query: str, results: List[dict], top_k: int = 3) -> List[dict]:
    """Re-rank results using BM25 and diversity"""
    
    from rank_bm25 import BM25Okapi
    
    # BM25 scoring
    corpus = [r['content'] for r in results]
    bm25 = BM25Okapi(corpus)
    
    query_tokens = query.lower().split()
    bm25_scores = bm25.get_scores(query_tokens)
    
    # Combine scores
    for i, result in enumerate(results):
        # Normalize scores to 0-1
        vector_score = result['similarity']  # Already 0-1
        bm25_score = bm25_scores[i] / max(bm25_scores) if max(bm25_scores) > 0 else 0
        
        # Weighted combination
        result['final_score'] = 0.7 * vector_score + 0.3 * bm25_score
    
    # Sort by final score
    results = sorted(results, key=lambda x: x['final_score'], reverse=True)
    
    # Diversity filter (remove similar results)
    final_results = []
    for result in results:
        # Check if too similar to existing results
        is_similar = False
        for existing in final_results:
            if result['doc_id'] == existing['doc_id']:
                is_similar = True
                break
        
        if not is_similar:
            final_results.append(result)
        
        if len(final_results) >= top_k:
            break
    
    return final_results

# Example
reranked = rerank_results(query, results, top_k=3)
print(f"Top {len(reranked)} results after re-ranking")
```

### Step 5: Format Response

```python
def format_response(results: List[dict]) -> dict:
    """Format results for user"""
    
    return {
        'query': query,
        'results': [
            {
                'rank': i + 1,
                'doc_id': r['doc_id'],
                'chunk_id': r['chunk_id'],
                'content': r['content'][:200] + '...',  # Preview
                'score': r['final_score'],
                'source': f"Document {r['doc_id']}"
            }
            for i, r in enumerate(results)
        ],
        'count': len(results),
        'timestamp': datetime.now().isoformat()
    }

# Example
response = format_response(reranked)
print(json.dumps(response, indent=2))
```

### Step 6: Log & Monitor

```python
def log_and_monitor(query: str, results: List[dict], latency_ms: float):
    """Log query and monitor metrics"""
    
    from google.cloud import bigquery
    
    client = bigquery.Client()
    
    # Log query
    log_entry = {
        'query': query,
        'num_results': len(results),
        'top_score': results[0]['final_score'] if results else 0,
        'latency_ms': latency_ms,
        'timestamp': datetime.now().isoformat()
    }
    
    # Insert into BigQuery
    table_id = "project.dataset.query_logs"
    errors = client.insert_rows_json(table_id, [log_entry])
    
    if errors:
        print(f"Error logging: {errors}")
    
    # Check SLO
    if latency_ms > 200:
        print(f"WARNING: Latency {latency_ms}ms exceeds SLO of 200ms")

# Example
log_and_monitor(query, reranked, latency_ms=150)
```

---

## Latency Optimization

### Latency Breakdown

```
Typical latency (5M documents):
- Query parsing: 10ms
- Embedding generation: 100ms ← Slowest
- Vector search: 50-100ms
- Re-ranking: 20ms
- Formatting: 5ms
- Logging: 10ms
- Network: 50ms
─────────────────────
Total: 245-295ms
```

### Optimization Strategies

**1. Cache Query Embeddings**
```python
from functools import lru_cache

@lru_cache(maxsize=1000)
def get_query_embedding(query: str) -> List[float]:
    """Cache query embeddings"""
    return generate_query_embedding(query)

# Benefit: 80% of queries are repeated
# Savings: 100ms per cached query
# Cost: Minimal (1000 queries = 3MB)
```

**2. Batch Queries**
```python
def batch_search(queries: List[str]) -> List[List[dict]]:
    """Process multiple queries in parallel"""
    
    # Generate embeddings in batch
    embeddings = batch_embed(queries)
    
    # Search in parallel
    results = []
    for embedding in embeddings:
        result = vector_search(embedding)
        results.append(result)
    
    return results

# Benefit: Amortize overhead
# Savings: 30-50% latency reduction
```

**3. Reduce Dimensions**
```
768 dimensions: 100ms search
384 dimensions: 50ms search
192 dimensions: 25ms search

Trade-off:
- Accuracy: 99% → 95% → 90%
- Latency: 100ms → 50ms → 25ms
```

**4. Use Approximate Search**
```
HNSW: 100ms (exact)
IVF: 50ms (approximate)
FLAT: 200ms (exact, slow)

Trade-off:
- Accuracy: 99% → 95%
- Latency: 100ms → 50ms
```

**5. Add Regional Replicas**
```
Single region: 100ms
Multiple regions: 50ms (parallel search)

Trade-off:
- Cost: 2x
- Latency: 50% reduction
```

---

## Precision & Recall

### Definitions

```
Precision: Of retrieved results, how many are relevant?
Recall: Of all relevant results, how many did we retrieve?

Example:
- 10 relevant documents exist
- We retrieve 5 documents
- 4 are relevant

Precision: 4/5 = 80%
Recall: 4/10 = 40%
```

### Measuring Accuracy

```python
def measure_accuracy(queries: List[str], ground_truth: List[List[str]]) -> dict:
    """Measure precision and recall"""
    
    precisions = []
    recalls = []
    
    for query, relevant_docs in zip(queries, ground_truth):
        # Get results
        embedding = generate_query_embedding(query)
        results = vector_search(embedding, top_k=3)
        retrieved_docs = [r['doc_id'] for r in results]
        
        # Calculate metrics
        relevant_set = set(relevant_docs)
        retrieved_set = set(retrieved_docs)
        
        intersection = relevant_set & retrieved_set
        
        precision = len(intersection) / len(retrieved_set) if retrieved_set else 0
        recall = len(intersection) / len(relevant_set) if relevant_set else 0
        
        precisions.append(precision)
        recalls.append(recall)
    
    return {
        'avg_precision': sum(precisions) / len(precisions),
        'avg_recall': sum(recalls) / len(recalls),
        'f1_score': 2 * (sum(precisions)/len(precisions)) * (sum(recalls)/len(recalls)) / 
                   ((sum(precisions)/len(precisions)) + (sum(recalls)/len(recalls)))
    }

# Example
metrics = measure_accuracy(test_queries, ground_truth)
print(f"Precision: {metrics['avg_precision']:.1%}")
print(f"Recall: {metrics['avg_recall']:.1%}")
print(f"F1 Score: {metrics['f1_score']:.1%}")
```

### Improving Accuracy

```
1. Better chunking
   - Use recursive chunking
   - Smaller chunks (250 words)
   - Add overlap

2. Better embeddings
   - Use larger model (1536 dims)
   - Fine-tune on domain data
   - Use multi-modal embeddings

3. Better search
   - Use HNSW instead of IVF
   - Increase top-K (search 10, return 3)
   - Add re-ranking (BM25)

4. Better monitoring
   - Track accuracy metrics
   - Alert on accuracy drop
   - A/B test improvements
```

---

## Production Patterns

### Pattern 1: Simple Pipeline

```python
def simple_pipeline(query: str) -> List[dict]:
    """Simple retrieval pipeline"""
    
    # Generate embedding
    embedding = generate_query_embedding(query)
    
    # Search
    results = vector_search(embedding, top_k=3)
    
    return results

# Use for: Small datasets, simple queries
# Latency: 200-250ms
# Accuracy: 90%+
```

### Pattern 2: Cached Pipeline

```python
@cache_decorator(ttl=3600)
def cached_pipeline(query: str) -> List[dict]:
    """Pipeline with caching"""
    
    # Check cache
    cached = cache.get(query)
    if cached:
        return cached
    
    # Generate embedding
    embedding = generate_query_embedding(query)
    
    # Search
    results = vector_search(embedding, top_k=3)
    
    # Cache results
    cache.set(query, results, ttl=3600)
    
    return results

# Use for: Repeated queries
# Latency: 10ms (cached), 200ms (uncached)
# Accuracy: 90%+
```

### Pattern 3: Hybrid Pipeline

```python
def hybrid_pipeline(query: str) -> List[dict]:
    """Hybrid vector + keyword search"""
    
    # Vector search
    embedding = generate_query_embedding(query)
    vector_results = vector_search(embedding, top_k=10)
    
    # Keyword search
    keyword_results = keyword_search(query, top_k=10)
    
    # Combine and re-rank
    combined = vector_results + keyword_results
    reranked = rerank_results(query, combined, top_k=3)
    
    return reranked

# Use for: Complex queries, mixed content
# Latency: 300-400ms
# Accuracy: 95%+
```

### Pattern 4: Multi-Stage Pipeline

```python
def multi_stage_pipeline(query: str) -> List[dict]:
    """Multi-stage retrieval pipeline"""
    
    # Stage 1: Fast approximate search
    embedding = generate_query_embedding(query)
    candidates = approximate_search(embedding, top_k=100)
    
    # Stage 2: Slow exact search on candidates
    reranked = exact_rerank(query, candidates, top_k=10)
    
    # Stage 3: Final ranking
    final = final_rank(query, reranked, top_k=3)
    
    return final

# Use for: Large datasets, high accuracy needed
# Latency: 400-500ms
# Accuracy: 99%+
```

---

## Key Takeaways

1. **6-step workflow** - Parse → Embed → Search → Rank → Format → Log
2. **Latency optimization** - Cache, batch, reduce dimensions, approximate search
3. **Precision & recall** - Measure and improve continuously
4. **Production patterns** - Choose based on requirements

