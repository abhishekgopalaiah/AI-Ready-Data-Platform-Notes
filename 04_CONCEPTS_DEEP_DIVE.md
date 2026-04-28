# Phase 6: Deep Dive into Each Concept
## Complete In-Depth Learning Guide

---

## Table of Contents
1. [Concept 1: Embeddings (In-Depth)](#concept-1-embeddings-in-depth)
2. [Concept 2: Vector Search (In-Depth)](#concept-2-vector-search-in-depth)
3. [Concept 3: Chunking Strategies (In-Depth)](#concept-3-chunking-strategies-in-depth)
4. [Concept 4: Retrieval Pipeline (In-Depth)](#concept-4-retrieval-pipeline-in-depth)
5. [Concept 5: Freshness Monitoring (In-Depth)](#concept-5-freshness-monitoring-in-depth)
6. [Concept 6: Access Control (In-Depth)](#concept-6-access-control-in-depth)

---

## Concept 1: Embeddings (In-Depth)

### What Are Embeddings?

**Definition:** An embedding is a numerical representation of text in a high-dimensional space where semantic meaning is preserved.

**Simple Analogy:**
Imagine a library where books are organized not by alphabetical order, but by meaning:
- Books about "cats" are close to books about "dogs" (both animals)
- Books about "cats" are far from books about "mathematics" (different topics)

Embeddings do the same thing with text, but in 768-dimensional space instead of 3D.

### How Embeddings Work

**Step 1: Training (Done by Google)**
```
Input: Billions of text pairs with similarity labels
("cat", "dog") → similarity: 0.9
("cat", "spaceship") → similarity: 0.1

Neural Network learns:
- If texts are similar, their embeddings should be close
- If texts are different, their embeddings should be far
```

**Step 2: Inference (What you do)**
```
Input: "The cat sat on the mat"
↓
Neural Network processes text
↓
Output: [0.12, -0.45, 0.89, 0.23, ..., -0.67] (768 dimensions)
```

### Mathematical Foundation: Vectors, Dimensions, and Traits

**What is a Vector?**

A vector is a list of numbers arranged in a specific order.

```
Vector: [0.12, -0.45, 0.89]

This is a vector with 3 numbers (3 dimensions)
```

**What are Dimensions?**

A dimension is one number in a vector. Each dimension captures a different trait (characteristic) of the text.

```
Embedding: [0.12, -0.45, 0.89, 0.23, -0.67, ..., 0.11]
           ↑     ↑      ↑     ↑     ↑        ↑
        dim1   dim2   dim3  dim4  dim5    dim768

This embedding has 768 dimensions
```

**What are Traits?**

A trait is a characteristic or quality that something has. In embeddings, each dimension represents a trait.

```
Dimension 1 (0.12): "animal-ness" trait
  → How much is this text about animals?
  → 0.12 = somewhat animal-like

Dimension 2 (-0.45): "action-ness" trait
  → How much action is in this text?
  → -0.45 = some action

Dimension 3 (0.89): "location-ness" trait
  → How much is this text about location?
  → 0.89 = very location-focused

...and 765 more traits
```

**Vector Space:**
- Each dimension represents a learned trait/feature
- Dimensions don't have human-interpretable meaning (we can't know for sure what they capture)
- The model learned these traits from billions of text examples
- Similar texts have similar trait values across dimensions

**Example in 2D (simplified):**
```
        ↑ "Action"
        |
        | "The dog ran"
        |    ●
        |
"Sad"   |  "Happy"
←───────┼───────→
        |
"The cat sat" ●
        |
        | "Calm"
        ↓
```

Real embeddings are 768D, so we can't visualize them, but the principle is the same:
- Texts with similar traits (similar values across dimensions) are close together
- Texts with different traits are far apart

**Why 768 Dimensions?**

768 is the sweet spot because:
- ✓ Captures enough detail (98.5% accuracy on semantic similarity)
- ✓ Fast enough for production (100ms to embed, 600ms to search 1M docs)
- ✓ Cheap enough to scale ($0.02 per 1M tokens)
- ✗ More dimensions (1536, 3072) give minimal improvement but 2-4x cost
- ✗ Fewer dimensions (256, 512) lose important nuances

### Embedding Models Explained

**Vertex AI Text Embedding (textembedding-gecko@001)**
- Dimensions: 768
- Training data: Billions of text examples
- Cost: $0.02 per 1M tokens
- Speed: Fast (< 1 second for 100 texts)
- Best for: Production systems

**How it works:**
```
Input text: "How do I return a product?"
↓
Tokenizer: ["How", "do", "I", "return", "a", "product", "?"]
↓
Embedding layer 1: [0.1, 0.2, 0.3, ...]
↓
Embedding layer 2: [0.15, 0.25, 0.35, ...]
↓
... (multiple layers)
↓
Output: [0.12, -0.45, 0.89, ...] (768 dimensions)
```

**Why 768 dimensions?**
- More dimensions = more expressive power
- But more dimensions = more storage, slower search
- 768 is a sweet spot (good quality, reasonable speed)

### Practical Example: Understanding Embeddings

**Code: Analyze what embeddings capture**

```python
from google.cloud import aiplatform
import numpy as np

aiplatform.init(project="your-project", location="us-central1")
model = aiplatform.TextEmbeddingModel.from_pretrained("textembedding-gecko@001")

# Get embeddings for similar and different texts
texts = [
    "The cat sat on the mat",           # Animals
    "The dog ran in the park",          # Animals
    "The bird flew in the sky",         # Animals
    "The car drove on the road",        # Vehicles
    "The truck moved down the street",  # Vehicles
    "Mathematics is the study of numbers",  # Abstract
]

embeddings_obj = model.get_embeddings(texts)
embeddings = [np.array(e.values) for e in embeddings_obj]

# Calculate pairwise similarities
print("Similarity Matrix:")
print("(How similar are texts to each other?)\n")

for i, text1 in enumerate(texts):
    print(f"{text1[:30]:30}", end="")
    for j, text2 in enumerate(texts):
        # Cosine similarity
        sim = np.dot(embeddings[i], embeddings[j]) / (
            np.linalg.norm(embeddings[i]) * np.linalg.norm(embeddings[j])
        )
        print(f"{sim:6.3f}", end=" ")
    print()

# Output:
# The cat sat on the mat         1.000  0.876  0.834  0.234  0.198  0.045
# The dog ran in the park        0.876  1.000  0.892  0.267  0.223  0.052
# The bird flew in the sky       0.834  0.892  1.000  0.198  0.156  0.038
# The car drove on the road      0.234  0.267  0.198  1.000  0.923  0.067
# The truck moved down the street 0.198  0.223  0.156  0.923  1.000  0.071
# Mathematics is the study of... 0.045  0.052  0.038  0.067  0.071  1.000

# Notice:
# - Animal texts are similar to each other (0.8+)
# - Vehicle texts are similar to each other (0.9+)
# - Animal and vehicle texts are different (0.2-0.3)
# - Math text is different from all (0.04-0.07)
```

**Key Insight:** Embeddings automatically capture semantic meaning without explicit programming!

### Tokens: How Pricing Works

**What is a Token?**

A token is a small piece of text that an AI model processes. It's how models break down text into manageable chunks.

```
Sentence: "The cat sat on the mat"

Tokens:
1. "The"
2. "cat"
3. "sat"
4. "on"
5. "the"
6. "mat"

Total: 6 tokens
```

**Important: Tokens ≠ Words**

```
Word: "don't"
Tokens: ["do", "n't"]  (2 tokens, not 1 word)

Word: "running"
Tokens: ["run", "ning"]  (2 tokens, not 1 word)

Number: "12345"
Tokens: ["123", "45"]  (2 tokens, not 1 number)
```

**Token Counting (Rough Estimate)**

```
1 token ≈ 4 characters
1 token ≈ 0.75 words

Example:
"The cat sat on the mat" (23 characters, 6 words)
Estimated tokens: 23 / 4 = 5.75 ≈ 6 tokens ✓
```

**Why Tokens Matter for Embeddings**

Embeddings are charged by tokens, not words:

```
Vertex AI pricing: $0.02 per 1M tokens

Example costs:
- Document: "Return policy: 30 days" (8 tokens)
  Cost: 8 × ($0.02 / 1M) = $0.00000016

- 1M documents × 500 words each ≈ 650M tokens
  Cost: 650M × ($0.02 / 1M) = $13

- 10K queries/day × 50 tokens/query = 500K tokens/day
  Cost: 500K × ($0.02 / 1M) = $0.01/day = $3/month
```

**Token Limits**

Embedding models have maximum token limits:

```
Vertex AI Text Embedding: 2048 tokens max
OpenAI text-embedding-3: 8191 tokens max

If document > limit:
- Split into chunks
- Embed each chunk separately

Example:
Document: 5000 tokens (too long!)

Split into chunks:
Chunk 1: 2000 tokens ✓
Chunk 2: 2000 tokens ✓
Chunk 3: 1000 tokens ✓

Embed each chunk separately
```

### Embedding Quality Metrics

**How to know if embeddings are good?**

1. **Semantic Similarity**
   - Similar texts should have high similarity (>0.8)
   - Different texts should have low similarity (<0.3)

2. **Stability**
   - Same text should always produce same embedding
   - Slight variations should produce similar embeddings

3. **Dimensionality**
   - 768 dimensions is standard
   - More dimensions = more expressive but slower
   - Fewer dimensions = faster but less expressive

### Cost Analysis

**Vertex AI Text Embedding Pricing:**
```
$0.02 per 1M tokens

Example costs:
- 1,000 documents (500 words each) = 500K tokens = $0.01
- 1M documents (500 words each) = 500B tokens = $10,000
- 10K queries/day = 50K tokens/day = $0.001/day = $0.30/month
```

**Cost Optimization:**
- Embed once, search many times
- Cache embeddings in BigQuery
- Batch embedding requests
- Use smaller models for less critical data

---

## Concept 2: Vector Search (In-Depth)

### What is Vector Search?

**Definition:** Finding K nearest neighbors in high-dimensional space based on vector similarity.

**Real-world analogy:**
You're in a library with 1 million books arranged in a 768-dimensional space. You want to find books similar to "How to return products?"

Instead of reading all 1M books, you:
1. Convert your query to a vector
2. Find the K closest books in space
3. Return those K books

### Similarity Metrics

**Cosine Similarity (Most Common)**

Formula:
```
similarity(A, B) = (A · B) / (||A|| × ||B||)

Where:
- A · B = dot product (sum of element-wise multiplication)
- ||A|| = magnitude (length) of vector A
- ||B|| = magnitude (length) of vector B
```

**Example:**
```
Vector A: [1, 0, 0]
Vector B: [1, 1, 0]

A · B = (1×1) + (0×1) + (0×0) = 1
||A|| = √(1² + 0² + 0²) = 1
||B|| = √(1² + 1² + 0²) = √2 ≈ 1.414

similarity = 1 / (1 × 1.414) = 0.707
```

**Why Cosine Similarity?**
- Ignores magnitude (only direction matters)
- Fast to compute
- Works in high dimensions
- Range: 0 (opposite) to 1 (identical)

**Other Metrics:**
```
1. Euclidean Distance
   distance = √((A₁-B₁)² + (A₂-B₂)² + ... + (Aₙ-Bₙ)²)
   Problem: Affected by magnitude
   
2. Manhattan Distance
   distance = |A₁-B₁| + |A₂-B₂| + ... + |Aₙ-Bₙ|
   Problem: Slower than cosine
   
3. Dot Product
   similarity = A · B
   Problem: Affected by magnitude
```

### Vector Search Algorithms

**Brute Force (Baseline)**
```
For each query:
  For each document:
    Calculate similarity
  Sort by similarity
  Return top-K
  
Time: O(N × D) where N=documents, D=dimensions
For 1M documents: 1M × 768 = 768M operations (slow!)
```

**Approximate Nearest Neighbor (ANN)**
```
Build index (once):
  Organize vectors in tree structure
  
For each query:
  Traverse tree to find approximate neighbors
  
Time: O(log N × D)
For 1M documents: log(1M) × 768 ≈ 7,680 operations (fast!)
```

**Algorithms:**
- HNSW (Hierarchical Navigable Small World)
- IVF (Inverted File Index)
- LSH (Locality Sensitive Hashing)
- Annoy (Approximate Nearest Neighbors Oh Yeah)

### Vector Search in BigQuery

**How BigQuery does vector search:**

```sql
-- Create vector index
CREATE VECTOR INDEX idx_embeddings
ON `project.dataset.chunks`(embedding)
OPTIONS(index_type = 'TREE_AH');

-- Search
SELECT 
    chunk_id,
    chunk_text,
    ML.DISTANCE(embedding, @query_embedding) as distance
FROM `project.dataset.chunks`
ORDER BY distance
LIMIT 10;
```

**Behind the scenes:**
1. Query embedding is converted to vector
2. Index is used to find approximate neighbors
3. Exact distances calculated for top candidates
4. Results returned sorted by distance

### Practical Example: Vector Search

**Code: Implement vector search**

```python
import numpy as np
from google.cloud import aiplatform, bigquery

def vector_search(project_id, dataset_id, query, top_k=5):
    """Search documents by vector similarity."""
    
    # Initialize clients
    aiplatform.init(project=project_id, location="us-central1")
    bq = bigquery.Client(project=project_id)
    model = aiplatform.TextEmbeddingModel.from_pretrained("textembedding-gecko@001")
    
    # Step 1: Embed query
    query_embedding_obj = model.get_embeddings([query])[0]
    query_embedding = np.array(query_embedding_obj.values)
    
    print(f"Query: {query}")
    print(f"Query embedding shape: {query_embedding.shape}")
    print(f"Query embedding (first 5): {query_embedding[:5]}")
    print()
    
    # Step 2: Fetch all chunks (in production, use vector index)
    sql = f"""
    SELECT 
        chunk_id,
        chunk_text,
        embedding,
        title
    FROM `{project_id}.{dataset_id}.chunks`
    LIMIT 1000
    """
    
    results = bq.query(sql).result()
    
    # Step 3: Calculate similarities
    similarities = []
    for row in results:
        chunk_embedding = np.array(row.embedding)
        
        # Cosine similarity
        dot_product = np.dot(query_embedding, chunk_embedding)
        magnitude_query = np.linalg.norm(query_embedding)
        magnitude_chunk = np.linalg.norm(chunk_embedding)
        
        similarity = dot_product / (magnitude_query * magnitude_chunk)
        
        similarities.append({
            "chunk_id": row.chunk_id,
            "text": row.chunk_text,
            "title": row.title,
            "similarity": similarity
        })
    
    # Step 4: Sort and return top-K
    similarities.sort(key=lambda x: x["similarity"], reverse=True)
    
    print(f"Top {top_k} results:")
    for rank, result in enumerate(similarities[:top_k], 1):
        print(f"{rank}. {result['title']}")
        print(f"   Similarity: {result['similarity']:.4f}")
        print(f"   Text: {result['text'][:100]}...")
        print()
    
    return similarities[:top_k]

# Usage
results = vector_search("your-project", "ai_platform", "How do I return?", top_k=5)
```

### Performance Tuning

**Latency Optimization:**
```
Query latency = embedding_time + search_time + retrieval_time

Typical breakdown:
- Embedding query: 100ms
- Search (with index): 50ms
- Retrieve from BigQuery: 200ms
- Total: 350ms (target: <500ms)

Optimization:
1. Cache query embeddings
2. Use vector index (not brute force)
3. Limit results fetched (LIMIT 100)
4. Batch queries
```

**Accuracy vs Speed:**
```
Brute force search: 100% accuracy, slow
ANN search: 95-99% accuracy, fast

In practice: ANN is good enough and much faster
```

---

## Concept 3: Chunking Strategies (In-Depth)

### Why Chunking Matters

**Problem:**
```
Document: "The return policy allows returns within 30 days..."
(5000 words, 10,000 tokens)

Embedding model limit: 2048 tokens

Solution: Split into chunks
Chunk 1: "The return policy allows returns within 30 days..." (500 tokens)
Chunk 2: "Refunds are processed within 5-7 business days..." (500 tokens)
...
```

**Why not just embed the whole document?**
1. Token limits (embedding models have max input)
2. Loss of detail (long documents lose nuance)
3. Cost (longer = more tokens = more cost)
4. Retrieval precision (fine-grained chunks better for search)

### Chunking Strategy 1: Fixed Size

**How it works:**
```
Document: "Paragraph 1. Paragraph 2. Paragraph 3. Paragraph 4."

Chunk size: 2 sentences
Overlap: 1 sentence

Chunk 1: "Paragraph 1. Paragraph 2."
Chunk 2: "Paragraph 2. Paragraph 3."
Chunk 3: "Paragraph 3. Paragraph 4."
```

**Code:**
```python
def fixed_size_chunk(text, chunk_size=500, overlap=50):
    """Split text into fixed-size chunks."""
    words = text.split()
    chunks = []
    
    for i in range(0, len(words), chunk_size - overlap):
        chunk = " ".join(words[i:i + chunk_size])
        if chunk.strip():
            chunks.append(chunk)
    
    return chunks

# Example
text = "word1 word2 word3 ... word1000"
chunks = fixed_size_chunk(text, chunk_size=100, overlap=10)
# Result: 10 chunks of ~100 words each
```

**Pros:**
- Simple to implement
- Fast
- Predictable

**Cons:**
- Breaks sentences mid-way
- Loses context
- May split related content

**Use when:**
- You have simple, uniform documents
- Speed is critical
- You don't care about perfect boundaries

---

### Chunking Strategy 2: Recursive (Recommended)

**How it works:**
```
Document (5000 words)
↓
Try splitting by paragraph (separator: "\n\n")
↓
If paragraph > 1000 words:
  Split by sentence (separator: ". ")
↓
If sentence > 1000 words:
  Split by word (separator: " ")
↓
Result: Logical chunks that preserve meaning
```

**Code:**
```python
def recursive_chunk(text, max_tokens=500, separators=None):
    """Recursively split text by separators."""
    if separators is None:
        separators = ["\n\n", "\n", ". ", " ", ""]
    
    def _split(text, separator):
        if separator == "":
            return list(text)
        splits = text.split(separator)
        return [s for s in splits if s]
    
    def _merge_splits(splits, separator, max_tokens):
        """Merge splits to reach max_tokens."""
        separator_len = len(separator.split())
        chunks = []
        current_chunk = []
        current_length = 0
        
        for split in splits:
            split_length = len(split.split())
            
            if current_length + split_length + separator_len > max_tokens:
                if current_chunk:
                    chunk = separator.join(current_chunk)
                    chunks.append(chunk)
                    current_chunk = [split]
                    current_length = split_length
                else:
                    chunks.append(split)
            else:
                current_chunk.append(split)
                current_length += split_length + separator_len
        
        if current_chunk:
            chunk = separator.join(current_chunk)
            chunks.append(chunk)
        
        return chunks
    
    for separator in separators:
        splits = _split(text, separator)
        
        if len(splits) > 1:
            good_splits = []
            for split in splits:
                if len(split.split()) < max_tokens:
                    good_splits.append(split)
                else:
                    sub_splits = recursive_chunk(
                        split,
                        max_tokens=max_tokens,
                        separators=separators[separators.index(separator)+1:]
                    )
                    good_splits.extend(sub_splits)
            
            return _merge_splits(good_splits, separator, max_tokens)
    
    return [text]

# Example
text = """
Paragraph 1: Introduction to returns.

Paragraph 2: Return policy details.

Paragraph 3: Refund process.
"""

chunks = recursive_chunk(text, max_tokens=50)
# Result: 3 chunks, one per paragraph
```

**Pros:**
- Preserves sentence structure
- Logical chunk boundaries
- Better context preservation

**Cons:**
- Slightly more complex
- Slower than fixed size
- May create uneven chunk sizes

**Use when:**
- You want quality over speed
- Documents have clear structure
- You need good search results

---

### Chunking Strategy 3: Semantic (Best Quality)

**How it works:**
```
Sentence 1: "The cat sat on the mat."
Sentence 2: "The dog ran away."
Sentence 3: "The bird flew high."

Calculate similarity between consecutive sentences:
Similarity(1→2): 0.45 (low, break here)
Similarity(2→3): 0.42 (low, break here)

Result:
Chunk 1: "The cat sat on the mat."
Chunk 2: "The dog ran away."
Chunk 3: "The bird flew high."
```

**Code:**
```python
def semantic_chunk(text, threshold=0.5):
    """Split text based on semantic similarity."""
    from google.cloud import aiplatform
    import numpy as np
    
    aiplatform.init(project="your-project", location="us-central1")
    model = aiplatform.TextEmbeddingModel.from_pretrained("textembedding-gecko@001")
    
    # Split into sentences
    sentences = text.split(". ")
    sentences = [s + "." for s in sentences[:-1]] + [sentences[-1]]
    
    # Get embeddings
    embeddings_obj = model.get_embeddings(sentences)
    embeddings = [np.array(e.values) for e in embeddings_obj]
    
    # Find break points
    chunks = []
    current_chunk = [sentences[0]]
    
    for i in range(1, len(sentences)):
        # Calculate similarity
        similarity = np.dot(embeddings[i-1], embeddings[i]) / (
            np.linalg.norm(embeddings[i-1]) * np.linalg.norm(embeddings[i])
        )
        
        if similarity < threshold:
            # Start new chunk
            chunks.append(" ".join(current_chunk))
            current_chunk = [sentences[i]]
        else:
            # Add to current chunk
            current_chunk.append(sentences[i])
    
    if current_chunk:
        chunks.append(" ".join(current_chunk))
    
    return chunks

# Example
text = "The cat sat. The dog ran. The bird flew."
chunks = semantic_chunk(text, threshold=0.6)
# Result: 3 chunks based on semantic breaks
```

**Pros:**
- Best quality chunks
- Preserves semantic meaning
- Optimal for retrieval

**Cons:**
- Requires embeddings (cost!)
- Slower (need to embed all sentences)
- More complex

**Use when:**
- Quality is critical
- You have budget for embeddings
- You're doing important retrieval

---

### Chunking Strategy 4: Sliding Window

**How it works:**
```
Chunk size: 500 words
Overlap: 100 words

Chunk 1: [words 0-500]
Chunk 2: [words 400-900]    ← overlaps with Chunk 1
Chunk 3: [words 800-1300]   ← overlaps with Chunk 2
```

**Code:**
```python
def sliding_window_chunk(text, chunk_size=500, overlap=50):
    """Create overlapping chunks."""
    words = text.split()
    chunks = []
    
    for i in range(0, len(words), chunk_size - overlap):
        chunk = " ".join(words[i:i + chunk_size])
        if chunk.strip():
            chunks.append(chunk)
    
    return chunks

# Example
text = "word1 word2 ... word1000"
chunks = sliding_window_chunk(text, chunk_size=100, overlap=20)
# Result: Overlapping chunks
```

**Pros:**
- Preserves context across chunks
- Good for retrieval
- Handles boundary cases

**Cons:**
- More chunks (higher cost)
- More storage needed
- Duplicate content

**Use when:**
- Context across chunks is important
- You have budget for extra chunks
- You want to minimize missed content

---

### Chunking Strategy Comparison

| Strategy | Speed | Quality | Cost | Use Case |
|----------|-------|---------|------|----------|
| Fixed Size | Fast | Low | Low | Simple docs |
| Recursive | Medium | Medium | Medium | Structured docs |
| Semantic | Slow | High | High | Important retrieval |
| Sliding Window | Medium | Medium | High | Context-heavy |

**Recommendation:**
Start with **Recursive**, upgrade to **Semantic** if needed.

---

## Concept 4: Retrieval Pipeline (In-Depth)

### Pipeline Architecture

```
User Query
    ↓
[1] Embed Query (Vertex AI)
    ↓
[2] Vector Search (BigQuery)
    ↓
[3] Retrieve Chunks (BigQuery)
    ↓
[4] Rank Results (Optional)
    ↓
[5] Pass to LLM (Vertex AI)
    ↓
[6] Generate Answer (LLM)
    ↓
User Answer
```

### Step 1: Embed Query

**What happens:**
```
Input: "How do I return a product?"
↓
Tokenize: ["How", "do", "I", "return", "a", "product", "?"]
↓
Embed: [0.12, -0.45, 0.89, ...] (768 dimensions)
↓
Output: Query embedding
```

**Code:**
```python
from google.cloud import aiplatform

aiplatform.init(project="your-project", location="us-central1")
model = aiplatform.TextEmbeddingModel.from_pretrained("textembedding-gecko@001")

query = "How do I return a product?"
query_embedding_obj = model.get_embeddings([query])[0]
query_embedding = query_embedding_obj.values

print(f"Query: {query}")
print(f"Embedding dimensions: {len(query_embedding)}")
print(f"Embedding (first 5): {query_embedding[:5]}")
```

**Cost:** ~$0.00002 per query (negligible)

---

### Step 2: Vector Search

**What happens:**
```
Query embedding: [0.12, -0.45, 0.89, ...]
↓
Search index for K nearest neighbors
↓
Return top-K chunk IDs with similarities
```

**Code:**
```python
import numpy as np
from google.cloud import bigquery

bq = bigquery.Client(project="your-project")

# Fetch chunks
sql = """
SELECT chunk_id, embedding, chunk_text, title
FROM `your-project.ai_platform.chunks`
LIMIT 10000
"""

results = bq.query(sql).result()

# Calculate similarities
similarities = []
for row in results:
    chunk_embedding = np.array(row.embedding)
    similarity = np.dot(query_embedding, chunk_embedding) / (
        np.linalg.norm(query_embedding) * np.linalg.norm(chunk_embedding)
    )
    similarities.append({
        "chunk_id": row.chunk_id,
        "similarity": similarity
    })

# Get top-K
top_k = sorted(similarities, key=lambda x: x["similarity"], reverse=True)[:5]
```

**Performance:**
- Brute force: O(N × D) = 10K × 768 = 7.68M operations ≈ 100ms
- With index: O(log N × D) = 14 × 768 ≈ 10ms

---

### Step 3: Retrieve Chunks

**What happens:**
```
Top-K chunk IDs: [chunk_1, chunk_5, chunk_12]
↓
Fetch full chunk text from BigQuery
↓
Apply access control (filter by user permissions)
↓
Return chunks to LLM
```

**Code:**
```python
# Get top chunk IDs
top_chunk_ids = [r["chunk_id"] for r in top_k]

# Fetch full chunks
sql = f"""
SELECT chunk_id, chunk_text, title, access_level
FROM `your-project.ai_platform.chunks`
WHERE chunk_id IN ({", ".join([f"'{id}'" for id in top_chunk_ids])})
"""

results = bq.query(sql).result()

# Filter by access level
user_access_levels = ["PUBLIC", "INTERNAL"]  # For this user
filtered_chunks = [
    row for row in results
    if row.access_level in user_access_levels
]
```

---

### Step 4: Rank Results (Optional)

**Why rank?**
Vector search gives approximate results. Ranking refines them.

**Methods:**
1. **Reranker Model** (most accurate)
   ```python
   # Use a specialized reranker model
   # Scores pairs of (query, document)
   # More accurate than embedding similarity
   ```

2. **BM25** (keyword-based)
   ```python
   # Traditional full-text search
   # Good for keyword matching
   # Complement to vector search
   ```

3. **Fusion** (combine methods)
   ```python
   # Combine vector search + BM25
   # Better coverage
   ```

**Code: Simple reranking**
```python
def rerank_results(query, chunks, top_k=3):
    """Rerank chunks by relevance."""
    from google.cloud import aiplatform
    
    # Get query and chunk embeddings
    texts = [query] + [c["chunk_text"] for c in chunks]
    embeddings_obj = model.get_embeddings(texts)
    embeddings = [np.array(e.values) for e in embeddings_obj]
    
    query_emb = embeddings[0]
    chunk_embs = embeddings[1:]
    
    # Recalculate similarities (more accurate)
    reranked = []
    for chunk, chunk_emb in zip(chunks, chunk_embs):
        similarity = np.dot(query_emb, chunk_emb) / (
            np.linalg.norm(query_emb) * np.linalg.norm(chunk_emb)
        )
        reranked.append({**chunk, "similarity": similarity})
    
    # Sort and return top-K
    reranked.sort(key=lambda x: x["similarity"], reverse=True)
    return reranked[:top_k]
```

---

### Step 5: Pass to LLM

**What happens:**
```
Top-3 chunks:
1. "Return policy: 30 days"
2. "Refund process: 5-7 days"
3. "Contact support: support@company.com"

↓
Build prompt:
"Based on these documents:
[chunk 1]
[chunk 2]
[chunk 3]

Answer: How do I return a product?"

↓
Send to LLM
```

**Code:**
```python
import vertexai
from vertexai.generative_models import GenerativeModel

vertexai.init(project="your-project", location="us-central1")
model = GenerativeModel("gemini-pro")

# Build context
context = "\n\n".join([
    f"Document: {c['title']}\n{c['chunk_text']}"
    for c in top_chunks
])

# Build prompt
prompt = f"""You are a helpful assistant. Answer the user's question based on the provided documents.
If the answer is not in the documents, say "I don't have information about that."

Documents:
{context}

User Question: {query}

Answer:"""

# Generate response
response = model.generate_content(prompt)
answer = response.text
```

---

### Step 6: Generate Answer

**LLM does:**
1. Read context (retrieved chunks)
2. Read question
3. Generate answer based on context
4. Cite sources

**Why this works:**
- LLM has knowledge of language
- Context prevents hallucination
- LLM can synthesize information

---

### Key Metrics

**Latency:**
```
Embedding query: 100ms
Vector search: 50ms
Retrieve chunks: 100ms
Rank results: 50ms
Generate answer: 500ms
Total: ~800ms (target: <1000ms)
```

**Recall:**
```
Definition: Did we find relevant documents?

Example:
Query: "How do I return?"
Relevant documents: 5
Retrieved documents: 3
Recall: 3/5 = 60%

Target: >80%
```

**Precision:**
```
Definition: Were retrieved documents actually relevant?

Example:
Retrieved documents: 5
Actually relevant: 4
Precision: 4/5 = 80%

Target: >70%
```

---

## Concept 5: Freshness Monitoring (In-Depth)

### Why Freshness Matters

**Scenario:**
```
Document: "Return policy: 30 days"
Embedded: 2025-01-01
Current: 2025-04-24
Age: 113 days

User asks: "What's the return policy?"
LLM returns: "30 days" (outdated!)

Actual policy: "60 days" (changed in March)
```

**Impact:**
- Wrong answers to users
- Loss of trust
- Compliance issues
- Legal liability

---

### Freshness SLO (Service Level Objective)

**Definition:** Maximum acceptable age of embeddings.

**Example SLO:**
```
SLO: Embeddings must be <7 days old
Threshold: 95% of embeddings must meet SLO

Monitoring:
- Age of oldest embedding: 8 days → ALERT
- Percentage meeting SLO: 92% → ALERT
```

**Code: Monitor freshness**

```python
from google.cloud import bigquery
from datetime import datetime, timedelta

bq = bigquery.Client(project="your-project")

# Check freshness
sql = """
SELECT 
    COUNT(*) as total_chunks,
    COUNTIF(TIMESTAMP_DIFF(CURRENT_TIMESTAMP(), last_embedded_at, DAY) <= 7) as fresh_chunks,
    MIN(last_embedded_at) as oldest_embedding,
    MAX(last_embedded_at) as newest_embedding,
    AVG(TIMESTAMP_DIFF(CURRENT_TIMESTAMP(), last_embedded_at, DAY)) as avg_age_days
FROM `your-project.ai_platform.chunks`
"""

result = bq.query(sql).result()
row = list(result)[0]

total = row.total_chunks
fresh = row.fresh_chunks
freshness_pct = (fresh / total * 100) if total > 0 else 0

print(f"Freshness: {freshness_pct:.1f}%")
print(f"Fresh chunks: {fresh}/{total}")
print(f"Oldest embedding: {row.oldest_embedding}")
print(f"Average age: {row.avg_age_days:.1f} days")

# Alert if below SLO
SLO_THRESHOLD = 95
if freshness_pct < SLO_THRESHOLD:
    print(f"⚠️  ALERT: Freshness {freshness_pct:.1f}% < SLO {SLO_THRESHOLD}%")
```

---

### Re-embedding Strategy

**When to re-embed:**
```
1. Document updated
2. Embedding age > SLO
3. Embedding model changed
4. Scheduled refresh (weekly/monthly)
```

**Code: Re-embed stale documents**

```python
from google.cloud import bigquery, aiplatform
from datetime import datetime, timedelta

bq = bigquery.Client(project="your-project")
aiplatform.init(project="your-project", location="us-central1")
model = aiplatform.TextEmbeddingModel.from_pretrained("textembedding-gecko@001")

# Find stale documents
sql = """
SELECT DISTINCT d.doc_id, d.content
FROM `your-project.ai_platform.documents` d
LEFT JOIN `your-project.ai_platform.chunks` c ON d.doc_id = c.doc_id
WHERE d.updated_at > COALESCE(MAX(c.last_embedded_at), TIMESTAMP('1970-01-01'))
OR MAX(c.last_embedded_at) < TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
GROUP BY d.doc_id, d.content
"""

results = bq.query(sql).result()
stale_docs = list(results)

print(f"Found {len(stale_docs)} stale documents")

for doc in stale_docs:
    doc_id = doc.doc_id
    content = doc.content
    
    # Chunk
    words = content.split()
    chunks = []
    for i in range(0, len(words), 500 - 50):
        chunk = " ".join(words[i:i + 500])
        if chunk.strip():
            chunks.append(chunk)
    
    # Embed
    embeddings_obj = model.get_embeddings(chunks)
    embeddings = [e.values for e in embeddings_obj]
    
    # Delete old chunks
    delete_sql = f"DELETE FROM `your-project.ai_platform.chunks` WHERE doc_id = '{doc_id}'"
    bq.query(delete_sql).result()
    
    # Insert new chunks
    now = datetime.utcnow().isoformat() + "Z"
    chunk_rows = []
    for i, (chunk, emb) in enumerate(zip(chunks, embeddings)):
        chunk_rows.append({
            "chunk_id": f"{doc_id}_{i}",
            "doc_id": doc_id,
            "chunk_text": chunk,
            "chunk_index": i,
            "embedding": emb,
            "embedding_model": "textembedding-gecko@001",
            "created_at": now,
            "last_embedded_at": now,
        })
    
    bq.insert_rows_json(
        f"your-project.ai_platform.chunks",
        chunk_rows
    )
    
    print(f"Re-embedded {doc_id}: {len(chunks)} chunks")
```

---

### Cost of Freshness

**Trade-off:**
```
More frequent re-embedding:
+ Fresher data
+ Better accuracy
- Higher cost
- More API calls

Less frequent re-embedding:
+ Lower cost
+ Fewer API calls
- Stale data
- Potential wrong answers
```

**Cost calculation:**
```
1M documents, 500 words each = 500M tokens
Re-embed weekly: 500M tokens × 4 weeks = 2B tokens/month
Cost: 2B × $0.02/1M = $40/month

Re-embed monthly: 500M tokens × 1 month = 500M tokens/month
Cost: 500M × $0.02/1M = $10/month

Difference: $30/month for weekly vs monthly
```

---

## Concept 6: Access Control (In-Depth)

### Why Access Control Matters

**Scenario:**
```
Documents:
- PUBLIC: "Return policy"
- INTERNAL: "Employee handbook"
- CONFIDENTIAL: "Salary information"

User: alice@company.com (employee)
Query: "salary budget"

Without access control:
- Vector search returns all documents
- LLM sees salary information
- Alice learns salaries (security breach!)

With access control:
- Vector search returns all documents
- Filter by alice's permissions
- LLM only sees PUBLIC + INTERNAL
- Alice doesn't see CONFIDENTIAL
```

---

### Access Control Levels

**Typical hierarchy:**
```
PUBLIC
├─ Anyone can see
├─ Customers, employees, public
└─ Example: "Return policy"

INTERNAL
├─ Employees only
├─ Company information
└─ Example: "Employee handbook"

CONFIDENTIAL
├─ Management only
├─ Sensitive information
└─ Example: "Salary information"

RESTRICTED
├─ Specific people only
├─ Legal, executive
└─ Example: "Board minutes"
```

---

### Implementation Strategy

**Option 1: Filter at Retrieval Time**

```python
def search_with_acl(query, user_email, top_k=3):
    """Search respecting access control."""
    
    # Get user's access levels
    access_levels = get_user_access_levels(user_email)
    
    # Embed query
    query_embedding = model.get_embeddings([query])[0].values
    
    # Fetch chunks (all)
    sql = """
    SELECT chunk_id, embedding, chunk_text, title, access_level
    FROM `your-project.ai_platform.chunks` c
    JOIN `your-project.ai_platform.documents` d ON c.doc_id = d.doc_id
    """
    
    results = bq.query(sql).result()
    
    # Calculate similarities
    similarities = []
    for row in results:
        # Check access
        if row.access_level not in access_levels:
            continue  # Skip if user can't see
        
        # Calculate similarity
        chunk_embedding = np.array(row.embedding)
        similarity = np.dot(query_embedding, chunk_embedding) / (
            np.linalg.norm(query_embedding) * np.linalg.norm(chunk_embedding)
        )
        
        similarities.append({
            "chunk_id": row.chunk_id,
            "text": row.chunk_text,
            "similarity": similarity
        })
    
    # Sort and return
    similarities.sort(key=lambda x: x["similarity"], reverse=True)
    return similarities[:top_k]

def get_user_access_levels(user_email):
    """Get access levels for user."""
    # In production, fetch from IAM or identity provider
    mapping = {
        "admin@company.com": ["PUBLIC", "INTERNAL", "CONFIDENTIAL", "RESTRICTED"],
        "manager@company.com": ["PUBLIC", "INTERNAL", "CONFIDENTIAL"],
        "employee@company.com": ["PUBLIC", "INTERNAL"],
        "customer@company.com": ["PUBLIC"],
    }
    return mapping.get(user_email, ["PUBLIC"])
```

**Pros:**
- Simple to implement
- Flexible (can change permissions dynamically)
- Works with any vector database

**Cons:**
- Slower (filter after search)
- May return fewer results (if many are filtered)

---

**Option 2: Filter at Index Time**

```python
# Create separate indexes for each access level
# Index 1: PUBLIC documents
# Index 2: PUBLIC + INTERNAL documents
# Index 3: ALL documents

def search_with_acl_v2(query, user_email, top_k=3):
    """Search using pre-filtered index."""
    
    # Get user's access levels
    access_levels = get_user_access_levels(user_email)
    
    # Determine which index to use
    if "RESTRICTED" in access_levels:
        index_name = "index_all"
    elif "CONFIDENTIAL" in access_levels:
        index_name = "index_confidential"
    elif "INTERNAL" in access_levels:
        index_name = "index_internal"
    else:
        index_name = "index_public"
    
    # Search in appropriate index
    # (faster because index is pre-filtered)
    results = search_in_index(index_name, query, top_k)
    
    return results
```

**Pros:**
- Faster (index is pre-filtered)
- Better performance at scale

**Cons:**
- More complex
- Need to maintain multiple indexes
- Permissions must be static

---

### Audit Trail

**Why audit?**
```
Compliance requirements:
- Who accessed what?
- When did they access it?
- What did they see?

Example:
User: alice@company.com
Query: "salary budget"
Time: 2025-04-24 12:00:00
Access level: INTERNAL
Result: "No results" (filtered out CONFIDENTIAL)
```

**Code: Log access**

```python
def log_access(user_email, query, results, access_levels):
    """Log access for audit trail."""
    
    from datetime import datetime
    
    audit_row = {
        "user_email": user_email,
        "query": query,
        "accessed_docs": len(results),
        "access_levels": ", ".join(access_levels),
        "timestamp": datetime.utcnow().isoformat() + "Z",
    }
    
    # Insert into audit table
    bq.insert_rows_json(
        "your-project.ai_platform.access_audit",
        [audit_row]
    )
```

---

### Best Practices

1. **Principle of Least Privilege**
   - Give users minimum necessary access
   - Default to restrictive
   - Require explicit grants

2. **Regular Audits**
   - Review access logs monthly
   - Check for unusual patterns
   - Remove unnecessary permissions

3. **Encryption**
   - Encrypt sensitive data at rest
   - Encrypt in transit (HTTPS)
   - Use service accounts for API access

4. **Monitoring**
   - Alert on access to CONFIDENTIAL
   - Alert on failed access attempts
   - Track permission changes

---

## Summary: All Concepts Together

```
User Query
    ↓
[Concept 1] Embed query (Embeddings)
    ↓
[Concept 2] Vector search (Vector Search)
    ↓
[Concept 3] Retrieve chunks (Chunking)
    ↓
[Concept 6] Filter by access (Access Control)
    ↓
[Concept 4] Rank results (Retrieval Pipeline)
    ↓
[Concept 5] Check freshness (Freshness Monitoring)
    ↓
Pass to LLM
    ↓
User Answer
```

All 6 concepts work together to create a production-grade RAG system.

---

## Next Steps

1. **Understand each concept** (you just did!)
2. **Implement in code** (Week 1-4)
3. **Test and measure** (latency, accuracy, cost)
4. **Deploy to production** (Airflow, monitoring)
5. **Iterate and improve** (based on metrics)

You're ready to build! 🚀
