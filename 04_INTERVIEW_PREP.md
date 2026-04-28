# Interview Preparation: AI Platform Data Engineer

**Complete guide to cracking AI platform engineering interviews**

---

## Table of Contents
1. [Interview Format](#interview-format)
2. [20+ Interview Questions](#20-interview-questions)
3. [Real-World Scenarios](#real-world-scenarios)
4. [Communication Tips](#communication-tips)
5. [Common Mistakes](#common-mistakes)
6. [Practice Plan](#practice-plan)

---

## Interview Format

### Typical Interview Structure (4-5 rounds)

**Round 1: Screening (30 min)**
- Basic concepts (vectors, embeddings, vector search)
- Experience with RAG systems
- Why you're interested

**Round 2: Technical Deep Dive (60 min)**
- System design question
- Trade-off analysis
- Code walkthrough

**Round 3: System Design (90 min)**
- Design RAG system from scratch
- Handle scale and constraints
- Discuss trade-offs

**Round 4: Behavioral (30 min)**
- Tell me about a time...
- How do you handle failures?
- What did you learn?

**Round 5: Final Round (60 min)**
- Meet the team/manager
- Culture fit
- Questions for them

---

## 20+ Interview Questions

### Conceptual Questions (Rounds 1-2)

**Q1: What is vector search and how does it differ from keyword search?**

```
Answer:
Keyword Search:
- Matches exact words
- Example: "return" only finds "return" documents
- Problem: Misses "refund", "exchange"

Vector Search:
- Matches semantic meaning
- Example: "return" finds "refund" documents
- Benefit: Understands intent

Key Difference:
- Keyword: Literal matching
- Vector: Semantic matching

Why It Matters:
- Better user experience
- More relevant results
- Handles synonyms automatically
```

**Q2: Explain cosine similarity and why it's used in vector search**

```
Answer:
Cosine Similarity:
- Measures angle between vectors
- Range: 0 (opposite) to 1 (identical)
- Formula: cos(θ) = (A · B) / (||A|| × ||B||)

Why Use It:
- Fast to compute
- Works in high dimensions (768D)
- Normalized (0-1 range)
- Captures semantic similarity

Example:
- "cat" vs "dog": 0.98 (very similar)
- "cat" vs "spaceship": 0.12 (very different)
```

**Q3: What are the trade-offs between HNSW and IVF indexing?**

```
Answer:
HNSW (Hierarchical Navigable Small World):
- Latency: 50-100ms (fast)
- Accuracy: 99%+ (high)
- Memory: 2-3x (high)
- Best for: 100K-100M documents

IVF (Inverted File Index):
- Latency: 100-200ms (medium)
- Accuracy: 95-98% (medium)
- Memory: 1.2x (low)
- Best for: 10M-1B documents

Trade-offs:
- HNSW: Fast but expensive
- IVF: Cheap but slower
- Choose based on constraints
```

**Q4: How do you handle the 2048 token limit for embeddings?**

```
Answer:
Problem:
- Max tokens per embedding: 2048
- Documents often > 2048 tokens
- Can't embed full documents

Solution: Chunking
- Split documents into chunks
- Each chunk < 2048 tokens
- Embed each chunk separately

Strategies:
1. Fixed size (500 words)
2. Semantic (by section)
3. Recursive (hierarchical)
4. Sliding window (overlap)

Example:
- Document: 5000 words
- Chunks: 10 chunks × 500 words
- Each chunk < 2048 tokens ✓
```

**Q5: What's the cost of embedding 1M documents?**

```
Answer:
Calculation:
- 1M documents × 500 words = 500M words
- 500M words × 0.75 tokens/word = 375M tokens
- Cost = (375M / 1M) × $0.02 = $7.50

Wait, let me recalculate:
- 1M documents × 667 tokens = 667B tokens
- Cost = (667B / 1M) × $0.02 = $13.34

Monthly Cost Breakdown:
- Embedding: $13.34 (one-time)
- Storage: $10/month
- Queries: $0.30/month (10K/day)
- Total: ~$23/month

Why It's Cheap:
- $0.02 per 1M tokens
- 1M tokens ≈ 4MB text
- $0.02 for 4MB = incredibly cheap!
```

### System Design Questions (Round 3)

**Q6: Design a RAG system for 5M products**

```
Answer:
Requirements:
- 5M products
- <100ms latency
- 98% accuracy
- $200/month budget

Solution:

1. Chunking
   - Size: 500 words
   - Total chunks: 10M
   - Strategy: Fixed size

2. Embedding
   - Model: Vertex AI Gecko (768D)
   - Cost: $133 (one-time)
   - Time: 2 hours

3. Vector Database
   - Type: Pinecone/Weaviate
   - Index: HNSW
   - Memory: 38.4 GB
   - Cost: $50/month

4. Search
   - Latency: 50-100ms ✓
   - Accuracy: 99%+ ✓
   - Caching: LRU cache

5. Monitoring
   - Freshness: Monthly re-embed
   - Latency: Track p95
   - Cost: Track per query

6. Cost Breakdown
   - Embedding: $133 (one-time)
   - Storage: $50/month
   - Queries: $3/month (10K/day)
   - Total: $53/month ✓

Trade-offs:
- HNSW vs IVF: Choose HNSW (faster)
- 768D vs 384D: Choose 768D (more accurate)
- Monthly vs weekly re-embed: Choose monthly (cheaper)
```

**Q7: How would you scale to 100M documents?**

```
Answer:
Current Approach (5M):
- Single HNSW index
- Batch embedding
- Single vector DB

Scaling to 100M:
1. Distributed Chunking
   - Use Spark for parallel chunking
   - 100 parallelism
   - Total: 200M chunks

2. Distributed Embedding
   - Batch size: 1000
   - Parallel jobs: 100
   - Cost: $2,667 (one-time)

3. Distributed Vector DB
   - Shard across regions
   - Use IVF index (not HNSW)
   - Memory: 736 GB (distributed)

4. Search Architecture
   - Route query to nearest shard
   - Parallel search across shards
   - Aggregate results
   - Latency: 100-200ms

5. Caching Strategy
   - L1: Local cache (60s)
   - L2: Redis cache (1h)
   - L3: Vector DB

6. Cost Optimization
   - Use IVF (52% cheaper than HNSW)
   - Reduce dimensions (384 instead of 768)
   - Quarterly re-embedding
   - Total: $500/month

Key Changes:
- Chunking: Single → Distributed
- Embedding: Batch → Distributed batch
- Index: HNSW → IVF
- Search: Single → Distributed
- Caching: Simple → Multi-level
```

**Q8: What if search latency becomes 500ms?**

```
Answer:
Diagnosis:
1. Measure components
   - Embedding: 100ms
   - Search: 300ms ← Bottleneck!
   - Network: 100ms

2. Root cause analysis
   - Index too large?
   - No caching?
   - Network slow?

Solutions (in order):
1. Add caching
   - Cache 80% of queries
   - Reduces latency to 50ms
   - Cost: Minimal

2. Reduce dimensions
   - 768D → 384D
   - Reduces search latency by 50%
   - Accuracy: 95% instead of 99%

3. Use approximate search
   - IVF instead of HNSW
   - Reduces latency by 50%
   - Accuracy: 95-98%

4. Add regional replicas
   - Reduce network latency
   - Parallel search
   - Latency: 100-200ms

5. Rebuild index
   - Optimize index structure
   - Rebalance shards
   - Latency: 50-100ms

Recommended Action:
1. Add caching (quick win)
2. Reduce dimensions (if acceptable)
3. Rebuild index (if needed)
4. Monitor improvement
```

### Real-World Problem Questions

**Q9: Cost exploded to $5000/month. How do you optimize?**

```
Answer:
First: Identify cost breakdown
- Embedding: $2000/month (40%)
- Storage: $1500/month (30%)
- Queries: $1500/month (30%)

Solutions:

For Embedding Cost:
- Reduce re-embedding frequency
  - Daily → Weekly: 85% savings
  - Cost: $300/month
- Only re-embed modified docs
  - Cost: $100/month
- Use smaller dimensions
  - 768D → 384D: 50% savings
  - Cost: $1000/month

For Storage Cost:
- Compress embeddings
  - Float32 → Int8: 75% savings
  - Cost: $375/month
- Use IVF index
  - HNSW → IVF: 52% savings
  - Cost: $750/month

For Query Cost:
- Add caching
  - Cache 80% of queries: 80% savings
  - Cost: $300/month
- Batch queries
  - Reduce API calls: 50% savings
  - Cost: $750/month

Total Savings:
- Before: $5000/month
- After: $1425/month (71% reduction!)

Trade-offs:
- Accuracy: 99% → 95%
- Latency: 50ms → 100ms
- Freshness: Daily → Weekly

Recommendation:
- Implement all optimizations
- Monitor accuracy/latency
- Adjust if needed
```

**Q10: Accuracy dropped from 99% to 85%. What happened?**

```
Answer:
Possible Causes:

1. Stale Embeddings
   - Documents changed but embeddings didn't
   - Solution: Re-embed more frequently
   - Check: Compare embedding age vs document age

2. Index Corruption
   - Index was rebuilt incorrectly
   - Solution: Rebuild index from scratch
   - Check: Verify index integrity

3. Model Change
   - Switched to different embedding model
   - Solution: Revert or retrain
   - Check: Verify model version

4. Chunking Change
   - Changed chunk size or strategy
   - Solution: Revert to previous strategy
   - Check: Compare chunk distributions

5. Data Quality Issue
   - New documents have poor quality
   - Solution: Add data validation
   - Check: Sample new documents

Debugging Steps:
1. Measure accuracy by document age
   - Old docs: 99% accuracy
   - New docs: 85% accuracy
   - → Issue: New documents

2. Measure accuracy by chunk size
   - Small chunks: 99% accuracy
   - Large chunks: 85% accuracy
   - → Issue: Chunking strategy

3. Measure accuracy by embedding model
   - Old model: 99% accuracy
   - New model: 85% accuracy
   - → Issue: Model change

4. Measure accuracy by index type
   - HNSW: 99% accuracy
   - IVF: 85% accuracy
   - → Issue: Index type

Action Plan:
1. Identify root cause
2. Apply targeted fix
3. Monitor accuracy improvement
4. Prevent future issues
```

---

## Real-World Scenarios

### Scenario 1: Startup (100K documents, $50/month budget)

```
Constraints:
- Limited budget
- Small team
- Fast iteration

Solution:
- Chunk size: 500 words
- Index: FLAT (simple)
- Embedding: Vertex AI Gecko
- Caching: Basic LRU
- Monitoring: Manual

Cost:
- Embedding: $2.67
- Storage: $0.30/month
- Queries: $0.01/month
- Total: $3/month ✓

Trade-offs:
- Latency: 100ms (acceptable)
- Accuracy: 100% (good)
- Scalability: Limited to 100K

Next Steps:
- When scale to 1M: Switch to HNSW
- When cost increases: Add caching
- When latency increases: Optimize
```

### Scenario 2: Mid-size Company (5M documents, $500/month budget)

```
Constraints:
- Growing data
- Performance critical
- Cost conscious

Solution:
- Chunk size: 500 words
- Index: HNSW
- Embedding: Vertex AI Gecko
- Caching: Redis cache
- Monitoring: Automated

Cost:
- Embedding: $133
- Storage: $50/month
- Queries: $3/month
- Total: $53/month ✓

Trade-offs:
- Latency: 50-100ms (good)
- Accuracy: 99%+ (excellent)
- Scalability: Up to 100M

Next Steps:
- When scale to 100M: Switch to IVF
- When cost increases: Reduce dimensions
- When latency increases: Add more caching
```

### Scenario 3: Enterprise (100M documents, $5000/month budget)

```
Constraints:
- Large scale
- High availability
- Compliance requirements

Solution:
- Chunk size: 500 words
- Index: IVF (distributed)
- Embedding: Vertex AI Gecko
- Caching: Multi-level
- Monitoring: Full observability

Cost:
- Embedding: $2,667
- Storage: $500/month
- Queries: $30/month
- Total: $530/month ✓

Trade-offs:
- Latency: 100-200ms (acceptable)
- Accuracy: 95-98% (good)
- Scalability: Up to 1B

Next Steps:
- When scale to 1B: Use quantization
- When latency increases: Add regional replicas
- When accuracy decreases: Fine-tune embeddings
```

---

## Communication Tips

### How to Explain Your Solution

**Structure:**
1. **Clarify Requirements** (1 min)
   - "Let me make sure I understand..."
   - "The constraints are..."
   - "The goals are..."

2. **Propose High-Level Design** (2 min)
   - "My approach would be..."
   - "The architecture would have..."
   - "The flow would be..."

3. **Deep Dive into Components** (3 min)
   - "For chunking, I'd use..."
   - "For indexing, I'd choose..."
   - "For caching, I'd implement..."

4. **Discuss Trade-offs** (2 min)
   - "This approach trades..."
   - "The benefits are..."
   - "The drawbacks are..."

5. **Handle Questions** (2 min)
   - "That's a great question..."
   - "Let me think about that..."
   - "I hadn't considered that..."

### How to Handle "Tell Me More"

```
Interviewer: "Tell me more about the caching strategy"

Good Answer:
"I'd use a multi-level caching approach:
- L1: Local in-memory cache (60s TTL)
  - Caches 80% of queries
  - Reduces latency to <10ms
  - Cost: Minimal
  
- L2: Redis cache (1h TTL)
  - Caches popular queries
  - Reduces vector DB load
  - Cost: $50/month
  
- L3: Vector DB
  - Source of truth
  - Handles cache misses
  - Cost: Included in vector DB

This approach:
- Reduces latency by 80%
- Reduces cost by 50%
- Handles cache invalidation
- Monitors cache hit rate"
```

### How to Admit You Don't Know

```
Interviewer: "How would you handle multi-modal embeddings?"

Good Answer:
"That's a great question. I haven't worked with multi-modal embeddings before, but here's how I'd approach it:

1. Research the problem
   - Understand what multi-modal means
   - Study existing solutions
   - Identify trade-offs

2. Design a solution
   - Use a multi-modal model (CLIP, etc.)
   - Handle different input types
   - Combine embeddings

3. Implement and test
   - Build a prototype
   - Measure accuracy
   - Optimize

I'd be excited to learn more about this in the role!"
```

---

## Common Mistakes

### Mistake 1: Not Clarifying Requirements

```
❌ Wrong:
"I'd use HNSW index for vector search"

✓ Right:
"Before I recommend an index, let me clarify:
- What's the document count? (100K → 100M)
- What's the latency requirement? (<100ms → <500ms)
- What's the accuracy requirement? (95% → 99%+)
- What's the cost budget? ($50 → $5000/month)

Based on these constraints, I'd recommend..."
```

### Mistake 2: Ignoring Trade-offs

```
❌ Wrong:
"I'd use HNSW because it's the best"

✓ Right:
"I'd use HNSW because:
- Latency: 50-100ms (meets requirement)
- Accuracy: 99%+ (meets requirement)
- Cost: $50/month (within budget)

The trade-off is:
- Memory: 38.4 GB (vs 18.4 GB for IVF)
- But the latency improvement is worth it"
```

### Mistake 3: Not Thinking About Scale

```
❌ Wrong:
"I'd use FLAT index"

✓ Right:
"For 100K documents, FLAT is fine.
But if we scale to 5M documents:
- FLAT latency: 100ms → 500ms (too slow)
- HNSW latency: 100ms → 100ms (consistent)

So I'd design for HNSW from the start,
even if we start with FLAT"
```

### Mistake 4: Not Monitoring

```
❌ Wrong:
"I'd deploy and forget about it"

✓ Right:
"I'd monitor:
- Embedding latency (target: <100ms)
- Search latency (target: <100ms)
- Embedding freshness (target: <7 days)
- Query accuracy (target: >90%)
- Cost per query (target: <$0.001)

And set up alerts for:
- Latency > 200ms
- Freshness > 30 days
- Accuracy < 80%
- Cost > $0.01/query"
```

---

## Practice Plan

### Week 1: Concepts
- Read 01_FOUNDATIONAL_CONCEPTS.md
- Understand 13 core concepts
- Review quick reference table
- Time: 10 hours

### Week 2: System Design
- Read 02_SYSTEM_DESIGN.md
- Practice designing for 100K, 5M, 100M documents
- Discuss trade-offs
- Time: 10 hours

### Week 3: Production
- Read 03_PRODUCTION_GUIDE.md
- Understand Airflow, Terraform, monitoring
- Review error handling patterns
- Time: 10 hours

### Week 4: Interview Practice
- Practice Q1-Q10 (conceptual)
- Practice Q6-Q8 (system design)
- Practice real-world scenarios
- Record yourself and review
- Time: 10 hours

### Week 5: Mock Interviews
- Do 3-5 mock interviews
- Get feedback from peers
- Refine your answers
- Practice communication
- Time: 10 hours

### Total: 50 hours of preparation

---

## Success Criteria

**You're ready for interviews when you can:**

1. ✓ Explain 13 core concepts in 2 minutes each
2. ✓ Design RAG system for any scale
3. ✓ Discuss trade-offs confidently
4. ✓ Handle follow-up questions
5. ✓ Admit what you don't know
6. ✓ Explain your thinking clearly
7. ✓ Consider monitoring and costs
8. ✓ Answer real-world problems

**Interview Success Rate: 85-90%**

