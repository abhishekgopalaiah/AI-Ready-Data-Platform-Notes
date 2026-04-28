# Quick Reference: Formulas, Checklists & Decision Trees

**Fast lookup guide for interviews and implementation**

---

## Formulas

### Embedding Cost
```
Cost = (Total Tokens / 1,000,000) × $0.02

Example:
- 1M documents × 667 tokens = 667B tokens
- Cost = (667B / 1M) × $0.02 = $13.34
```

### Token Count
```
Tokens ≈ Words × 0.75
Tokens ≈ Characters / 4

Example:
- 500 words ≈ 375 tokens
- 2000 characters ≈ 500 tokens
```

### Memory for Vector Index
```
Memory = Documents × Dimensions × Bytes × Index Overhead

FLAT: Documents × 768 × 4 × 1.0
HNSW: Documents × 768 × 4 × 2.5
IVF: Documents × 768 × 4 × 1.2

Example (5M documents):
- FLAT: 15.36 GB
- HNSW: 38.4 GB
- IVF: 18.43 GB
```

### Cosine Similarity
```
similarity = (A · B) / (||A|| × ||B||)

Range: 0 (opposite) to 1 (identical)
Threshold: > 0.7 for relevance
```

### Latency Breakdown
```
Total Latency = Embedding + Search + Network

Typical:
- Embedding: 100ms
- Search: 50-100ms
- Network: 50ms
- Total: 200-250ms
```

---

## Decision Trees

### Choose Index Type
```
Document Count?
├─ < 100K
│  └─ Use FLAT
│     - Latency: 100ms
│     - Accuracy: 100%
│     - Memory: 15GB
│
├─ 100K - 100M
│  └─ Use HNSW
│     - Latency: 50-100ms
│     - Accuracy: 99%+
│     - Memory: 38GB
│
└─ > 100M
   └─ Use IVF
      - Latency: 100-200ms
      - Accuracy: 95-98%
      - Memory: 18GB
```

### Choose Chunk Size
```
Document Type?
├─ FAQ (short)
│  └─ 500 words
│     - Tokens: 375
│     - Chunks: Few
│
├─ Article (medium)
│  └─ 500 words
│     - Tokens: 375
│     - Chunks: Many
│
└─ Book (long)
   └─ 250 words
      - Tokens: 188
      - Chunks: Very many
```

### Choose Embedding Model
```
Requirement?
├─ Speed (< 100ms)
│  └─ Vertex AI Gecko
│     - Dimensions: 768
│     - Speed: 100ms
│
├─ Quality (> 99%)
│  └─ OpenAI text-embedding-3
│     - Dimensions: 1536
│     - Speed: 150ms
│
└─ Cost (< $100/month)
   └─ Vertex AI Gecko
      - Dimensions: 768
      - Cost: $0.02 per 1M tokens
```

---

## Checklists

### Pre-Interview Checklist
- [ ] Understand 13 core concepts
- [ ] Can explain vector search in 2 minutes
- [ ] Know trade-offs (HNSW vs IVF)
- [ ] Can design for 100K, 5M, 100M documents
- [ ] Know cost calculations
- [ ] Can discuss monitoring
- [ ] Know common mistakes
- [ ] Practiced 5+ mock interviews

### System Design Checklist
- [ ] Clarified requirements
- [ ] Proposed high-level design
- [ ] Discussed chunking strategy
- [ ] Chose index type with justification
- [ ] Discussed caching strategy
- [ ] Addressed monitoring
- [ ] Discussed cost
- [ ] Handled follow-up questions

### Implementation Checklist
- [ ] Load documents
- [ ] Chunk documents
- [ ] Generate embeddings
- [ ] Store in vector DB
- [ ] Implement search
- [ ] Add caching
- [ ] Add monitoring
- [ ] Handle errors
- [ ] Test with small data
- [ ] Scale to production

### Production Checklist
- [ ] Airflow DAG for daily refresh
- [ ] Terraform for infrastructure
- [ ] Monitoring and alerting
- [ ] Error handling and retry
- [ ] Cost tracking
- [ ] Backup and recovery
- [ ] Documentation
- [ ] Runbooks for common issues

---

## Common Values

### Costs
```
Embedding: $0.02 per 1M tokens
Storage: $0.02 per GB per month (BigQuery)
Vector DB: $50-500 per month (Pinecone/Weaviate)
```

### Latencies
```
Embedding: 100ms
Search (HNSW): 50-100ms
Search (IVF): 100-200ms
Network: 50ms
Total: 200-250ms
```

### Accuracies
```
FLAT: 100%
HNSW: 99%+
IVF: 95-98%
Approximate: 90-95%
```

### Dimensions
```
Vertex AI Gecko: 768
OpenAI text-embedding-3: 1536
sentence-transformers: 384
```

### Token Ratios
```
1 token ≈ 4 characters
1 token ≈ 0.75 words
1 token ≈ 1.3 subwords
```

---

## Common Mistakes to Avoid

```
❌ Not clarifying requirements
✓ Always ask: scale, latency, accuracy, cost

❌ Ignoring trade-offs
✓ Always discuss: speed vs accuracy, cost vs quality

❌ Not thinking about scale
✓ Always design for 10x growth

❌ Choosing wrong index
✓ FLAT < 100K, HNSW 100K-100M, IVF > 100M

❌ Not monitoring
✓ Always track: latency, cost, freshness, accuracy

❌ Not handling errors
✓ Always implement: retry, circuit breaker, fallback

❌ Not optimizing cost
✓ Always use: batch, cache, compress, selective re-embed

❌ Not testing
✓ Always test: small data first, then scale
```

---

## Interview Answers (30 seconds)

### Q: What is vector search?
```
Vector search finds documents whose embeddings are "close" to the query embedding using cosine similarity. It differs from keyword search because it understands semantic meaning, not just keywords.
```

### Q: Why use HNSW?
```
HNSW provides fast search (50-100ms) with high accuracy (99%+) for medium-scale datasets (100K-100M documents). The trade-off is higher memory usage (2-3x).
```

### Q: How much does it cost?
```
Embedding costs $0.02 per 1M tokens. For 1M documents with 667 tokens each, that's $13.34 one-time. Monthly cost includes storage ($10) and queries ($0.30 for 10K/day).
```

### Q: How do you handle 2048 token limit?
```
Chunk documents into pieces < 2048 tokens. For a 5000-word document, split into 10 chunks of 500 words each. Embed each chunk separately.
```

### Q: What if latency is 500ms?
```
Measure components: embedding (100ms), search (300ms), network (100ms). Bottleneck is search. Solutions: add caching (quick), reduce dimensions, use IVF index, or add regional replicas.
```

---

## Calculation Examples

### Example 1: 100K Documents
```
Documents: 100K
Chunk size: 500 words
Total chunks: 200K
Tokens per chunk: 375
Total tokens: 75M

Cost:
- Embedding: (75M / 1M) × $0.02 = $1.50
- Storage: 200K × 768 × 4 / (1024^3) × $0.02 = $0.30/month
- Total: $1.50 + $0.30 = $1.80

Latency:
- Embedding: 100ms
- Search (FLAT): 100ms
- Network: 50ms
- Total: 250ms

Memory:
- FLAT: 200K × 768 × 4 = 600 MB
```

### Example 2: 5M Documents
```
Documents: 5M
Chunk size: 500 words
Total chunks: 10M
Tokens per chunk: 375
Total tokens: 3.75B

Cost:
- Embedding: (3.75B / 1M) × $0.02 = $75
- Storage: 10M × 768 × 4 / (1024^3) × $0.02 = $5/month
- Total: $75 + $5 = $80

Latency:
- Embedding: 100ms
- Search (HNSW): 50-100ms
- Network: 50ms
- Total: 200-250ms

Memory:
- HNSW: 10M × 768 × 4 × 2.5 = 76.8 GB
```

### Example 3: 100M Documents
```
Documents: 100M
Chunk size: 500 words
Total chunks: 200M
Tokens per chunk: 375
Total tokens: 75B

Cost:
- Embedding: (75B / 1M) × $0.02 = $1500
- Storage: 200M × 768 × 4 / (1024^3) × $0.02 = $100/month
- Total: $1500 + $100 = $1600

Latency:
- Embedding: 100ms
- Search (IVF): 100-200ms
- Network: 50ms
- Total: 250-350ms

Memory:
- IVF: 200M × 768 × 4 × 1.2 = 737 GB (distributed)
```

---

## Key Metrics

### SLOs (Service Level Objectives)
```
Embedding Latency: < 100ms
Search Latency: < 100ms
Total Latency: < 200ms
Embedding Freshness: < 7 days
Query Accuracy: > 90%
Error Rate: < 1%
```

### SLIs (Service Level Indicators)
```
p50 Latency: 100ms
p95 Latency: 200ms
p99 Latency: 300ms
Embedding Age: 3 days (average)
Accuracy: 95%
Error Rate: 0.1%
```

### Thresholds for Alerts
```
Embedding Latency > 150ms: Warning
Embedding Latency > 300ms: Critical
Search Latency > 100ms: Warning
Search Latency > 500ms: Critical
Embedding Age > 7 days: Warning
Embedding Age > 30 days: Critical
Error Rate > 1%: Warning
Error Rate > 5%: Critical
```

---

## Resources

### Tools
- **Embedding**: Vertex AI, OpenAI, Hugging Face
- **Vector DB**: Pinecone, Weaviate, Milvus, BigQuery
- **Orchestration**: Airflow, Prefect, Dagster
- **Infrastructure**: Terraform, Kubernetes
- **Monitoring**: Cloud Monitoring, Datadog, New Relic

### Documentation
- Vertex AI Embeddings: https://cloud.google.com/vertex-ai/docs/generative-ai/embeddings/get-text-embeddings
- BigQuery Vector Search: https://cloud.google.com/bigquery/docs/vector-search-intro
- Pinecone: https://docs.pinecone.io/
- Weaviate: https://weaviate.io/developers/weaviate/

