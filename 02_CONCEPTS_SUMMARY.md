# Phase 6: Concepts Summary & Quick Reference

## Quick Navigation

| Concept | File | Key Points |
|---------|------|-----------|
| 1. Embeddings | CONCEPTS_DEEP_DIVE.md | 768D vectors, semantic meaning, cost $0.02/1M tokens |
| 2. Vector Search | CONCEPTS_DEEP_DIVE.md | Cosine similarity, ANN algorithms, <500ms latency |
| 3. Chunking | CONCEPTS_DEEP_DIVE.md | 4 strategies, recursive recommended, trade-offs |
| 4. Retrieval Pipeline | CONCEPTS_DEEP_DIVE.md | 6 steps, latency breakdown, precision/recall |
| 5. Freshness Monitoring | CONCEPTS_DEEP_DIVE.md | SLO <7 days, re-embedding strategy, cost |
| 6. Access Control | CONCEPTS_DEEP_DIVE.md | 4 levels, filter at retrieval, audit trail |

---

## Concept 1: Embeddings

**What:** Vectors (768 numbers) representing text meaning

**Key Components:**

- **Vector:** List of numbers arranged in order
  ```
  Vector: [0.12, -0.45, 0.89, ...]
  ```

- **Dimension:** One number in a vector (captures a trait)
  ```
  768 dimensions = 768 traits
  Each dimension captures a characteristic of the text
  ```

- **Trait:** A characteristic or quality
  ```
  Dimension 1: "animal-ness" (0.12 = somewhat animal-like)
  Dimension 2: "action-ness" (-0.45 = some action)
  Dimension 3: "location-ness" (0.89 = very location-focused)
  ```

- **Token:** Small piece of text (for pricing)
  ```
  1 token ≈ 4 characters ≈ 0.75 words
  Pricing: $0.02 per 1M tokens
  ```

**Key Formula:**
```
Text → Tokenize → Neural Network → Embedding [0.12, -0.45, 0.89, ...]
```

**Key Insight:**
```
Similar texts → Similar embeddings
"cat" embedding ≈ "dog" embedding
"cat" embedding ≠ "spaceship" embedding
```

**Why 768 Dimensions?**
- ✓ Captures enough detail (98.5% accuracy)
- ✓ Fast enough (100ms to embed)
- ✓ Cheap enough ($0.02 per 1M tokens)
- ✗ More dimensions = minimal improvement, 2-4x cost

**Cost:** $0.02 per 1M tokens

**Use:** Generate once, search many times

---

## Concept 2: Vector Search

**What:** Finding K nearest neighbors in 768D space

**Key Formula (Cosine Similarity):**
```
similarity(A, B) = (A · B) / (||A|| × ||B||)
Range: 0 (opposite) to 1 (identical)
```

**Performance:**
```
Brute force: O(N × D) = slow
ANN index: O(log N × D) = fast
```

**Target:** <500ms latency

---

## Concept 3: Chunking Strategies

**Problem:** Documents too long to embed

**4 Strategies:**

| Strategy | Speed | Quality | Cost | Use Case |
|----------|-------|---------|------|----------|
| Fixed Size | Fast | Low | Low | Simple docs |
| Recursive | Medium | Medium | Medium | **Recommended** |
| Semantic | Slow | High | High | Important |
| Sliding Window | Medium | Medium | High | Context-heavy |

**Recommended:** Start with Recursive

---

## Concept 4: Retrieval Pipeline

**6 Steps:**
```
1. Embed query (100ms)
2. Vector search (50ms)
3. Retrieve chunks (100ms)
4. Rank results (50ms)
5. Pass to LLM (0ms)
6. Generate answer (500ms)
Total: ~800ms
```

**Key Metrics:**
- **Latency:** <1000ms
- **Recall:** >80% (find relevant docs)
- **Precision:** >70% (docs are relevant)

---

## Concept 5: Freshness Monitoring

**Problem:** Embeddings become stale

**SLO Example:**
```
Embeddings must be <7 days old
95% of embeddings must meet SLO
```

**Re-embedding:**
```
When:
- Document updated
- Embedding age > SLO
- Scheduled refresh

Cost: $0.02 per 1M tokens
```

---

## Concept 6: Access Control

**4 Levels:**
```
PUBLIC → Anyone
INTERNAL → Employees
CONFIDENTIAL → Management
RESTRICTED → Specific people
```

**Implementation:**
```
1. Get user's access levels
2. Search all documents
3. Filter by permissions
4. Log access (audit trail)
```

**Best Practice:** Principle of Least Privilege

---

## Learning Path

### Week 1: Embeddings & Vector Search
- ✓ Generate embeddings
- ✓ Calculate cosine similarity
- ✓ Implement vector search
- ✓ Understand embedding space

### Week 2: Chunking & Pipeline
- ✓ Implement chunking strategies
- ✓ Design BigQuery schema
- ✓ Build ingestion pipeline
- ✓ Retrieve documents

### Week 3: Production Features
- ✓ Orchestrate with Airflow
- ✓ Monitor freshness
- ✓ Implement access control
- ✓ Log events

### Week 4: Complete RAG
- ✓ Retrieve context
- ✓ Generate answers
- ✓ Measure quality
- ✓ Deploy to production

---

## Code Examples by Concept

### Concept 1: Generate Embeddings
```python
from google.cloud import aiplatform

aiplatform.init(project="your-project", location="us-central1")
model = aiplatform.TextEmbeddingModel.from_pretrained("textembedding-gecko@001")

texts = ["The cat sat on the mat", "The dog ran in the park"]
embeddings = model.get_embeddings(texts)

for text, embedding in zip(texts, embeddings):
    print(f"{text}: {len(embedding.values)} dimensions")
```

### Concept 2: Vector Search
```python
import numpy as np

def cosine_similarity(vec1, vec2):
    return np.dot(vec1, vec2) / (np.linalg.norm(vec1) * np.linalg.norm(vec2))

query_emb = np.array([0.12, -0.45, 0.89, ...])
doc_emb = np.array([0.11, -0.44, 0.88, ...])

similarity = cosine_similarity(query_emb, doc_emb)
print(f"Similarity: {similarity:.3f}")  # 0.987
```

### Concept 3: Recursive Chunking
```python
def recursive_chunk(text, max_tokens=500, separators=None):
    if separators is None:
        separators = ["\n\n", "\n", ". ", " ", ""]
    
    for sep in separators:
        splits = text.split(sep)
        if len(splits) > 1:
            return splits  # Simplified
    
    return [text]

chunks = recursive_chunk("Para 1. Para 2. Para 3.", max_tokens=10)
```

### Concept 4: Retrieval Pipeline
```python
# 1. Embed query
query_emb = model.get_embeddings([query])[0].values

# 2. Vector search
similarities = [cosine_similarity(query_emb, doc_emb) for doc_emb in doc_embs]

# 3. Retrieve chunks
top_chunks = sorted(zip(chunks, similarities), key=lambda x: x[1], reverse=True)[:3]

# 4. Rank (optional)
# 5. Pass to LLM
# 6. Generate answer
```

### Concept 5: Freshness Monitoring
```python
sql = """
SELECT 
    COUNT(*) as total,
    COUNTIF(TIMESTAMP_DIFF(CURRENT_TIMESTAMP(), last_embedded_at, DAY) <= 7) as fresh
FROM chunks
"""

result = bq.query(sql).result()
freshness_pct = (result.fresh / result.total) * 100
print(f"Freshness: {freshness_pct:.1f}% (SLO: >95%)")
```

### Concept 6: Access Control
```python
def search_with_acl(query, user_email):
    # Get user's access levels
    access_levels = get_user_access_levels(user_email)
    
    # Search all documents
    results = vector_search(query)
    
    # Filter by access
    filtered = [r for r in results if r.access_level in access_levels]
    
    # Log access
    log_access(user_email, query, filtered)
    
    return filtered
```

---

## Key Takeaways

### Embeddings
- 768-dimensional vectors
- Capture semantic meaning
- Cost: $0.02 per 1M tokens
- Embed once, search many times

### Vector Search
- Cosine similarity (0-1 range)
- ANN for fast search
- Target: <500ms latency
- Works across languages

### Chunking
- 4 strategies with trade-offs
- Recursive recommended
- Balance quality vs cost
- Preserve context

### Retrieval Pipeline
- 6 steps from query to answer
- ~800ms total latency
- Precision >70%, Recall >80%
- LLM generates final answer

### Freshness
- SLO: <7 days old
- Re-embed on schedule
- Monitor continuously
- Cost: $0.02 per 1M tokens

### Access Control
- 4 levels: PUBLIC, INTERNAL, CONFIDENTIAL, RESTRICTED
- Filter at retrieval time
- Audit all access
- Principle of least privilege

---

## Common Mistakes to Avoid

1. **Not chunking documents**
   - Embedding entire documents loses detail
   - Chunks enable fine-grained retrieval

2. **Ignoring freshness**
   - Stale embeddings = wrong answers
   - Set SLO and monitor

3. **No access control**
   - Sensitive data exposed
   - Compliance violations
   - Security breach

4. **Brute force search**
   - O(N × D) is too slow
   - Use ANN index
   - Target <500ms

5. **Not measuring quality**
   - Don't know if system works
   - Measure precision, recall, latency
   - Iterate based on metrics

---

## Next Steps

1. **Read CONCEPTS_DEEP_DIVE.md** for detailed explanations
2. **Run week1_embeddings.py** to see embeddings in action
3. **Implement chunking** (Week 2)
4. **Build retrieval pipeline** (Week 2)
5. **Add freshness monitoring** (Week 3)
6. **Implement access control** (Week 3)
7. **Build complete RAG** (Week 4)

---

## Resources

- **PHASE_6_AI_READY_DATA_PLATFORM.md** - Overview
- **CONCEPTS_DEEP_DIVE.md** - In-depth explanations
- **week1_embeddings.py** - Working code
- **README.md** - Setup guide
- **QUICKSTART.md** - 15-minute setup

---

**You're ready to master Phase 6! 🚀**
