# Advanced Topics: Beyond Basics

**Advanced techniques for production RAG systems**

---

## Table of Contents
1. [Hybrid Search](#hybrid-search)
2. [Multi-Modal Embeddings](#multi-modal-embeddings)
3. [Fine-Tuning Embeddings](#fine-tuning-embeddings)
4. [Distributed Vector Search](#distributed-vector-search)
5. [Real-Time Indexing](#real-time-indexing)

---

## Hybrid Search

### Definition
Combine vector search (semantic) with keyword search (lexical)

### Why Hybrid?

```
Vector Search Alone:
- Pros: Understands meaning
- Cons: Misses exact matches
- Example: "return" finds "refund" but misses "return policy"

Keyword Search Alone:
- Pros: Finds exact matches
- Cons: Misses synonyms
- Example: "return" finds "return policy" but misses "refund"

Hybrid:
- Pros: Best of both worlds
- Cons: Slower, more complex
- Example: Finds both "return policy" AND "refund"
```

### Implementation

```python
def hybrid_search(query: str, top_k: int = 3) -> List[dict]:
    """Hybrid vector + keyword search"""
    
    # Vector search
    embedding = generate_query_embedding(query)
    vector_results = vector_search(embedding, top_k=10)
    vector_scores = {r['chunk_id']: r['similarity'] for r in vector_results}
    
    # Keyword search (BM25)
    keyword_results = keyword_search(query, top_k=10)
    keyword_scores = {r['chunk_id']: r['score'] for r in keyword_results}
    
    # Combine scores
    combined_scores = {}
    for chunk_id in set(list(vector_scores.keys()) + list(keyword_scores.keys())):
        vector_score = vector_scores.get(chunk_id, 0)
        keyword_score = keyword_scores.get(chunk_id, 0)
        
        # Weighted combination
        combined_scores[chunk_id] = 0.6 * vector_score + 0.4 * keyword_score
    
    # Sort and return top-k
    sorted_results = sorted(combined_scores.items(), key=lambda x: x[1], reverse=True)
    
    return [
        {
            'chunk_id': chunk_id,
            'score': score
        }
        for chunk_id, score in sorted_results[:top_k]
    ]

# Example
results = hybrid_search("return policy", top_k=3)
```

### When to Use

```
Use Hybrid When:
- Exact matches matter (product names, codes)
- Synonyms are important (return/refund)
- Mixed content (structured + unstructured)
- High accuracy is critical

Don't Use When:
- Pure semantic search is enough
- Latency is critical (slower)
- Cost is tight (2x embeddings)
```

---

## Multi-Modal Embeddings

### Definition
Embeddings that handle multiple data types (text, images, audio)

### Why Multi-Modal?

```
Text-Only:
- Can only search text documents
- Misses images, tables, diagrams
- Example: Can't find product images

Multi-Modal:
- Can search across all content types
- Better user experience
- Example: "red shoes" finds images AND descriptions

Use Cases:
- E-commerce (products + images)
- Documentation (text + diagrams)
- News (articles + photos)
- Social media (posts + images)
```

### Implementation

```python
from google.cloud import aiplatform

def embed_multimodal(content_type: str, content: str) -> List[float]:
    """Generate multi-modal embedding"""
    
    aiplatform.init(project="your-project-id", location="us-central1")
    
    if content_type == "text":
        model = aiplatform.TextEmbeddingModel.from_pretrained(
            "textembedding-gecko@001"
        )
        embeddings = model.get_embeddings([content])
    
    elif content_type == "image":
        # Use CLIP or similar model
        from PIL import Image
        import requests
        
        # Download image
        img = Image.open(requests.get(content, stream=True).raw)
        
        # Generate embedding
        # (Implementation depends on model)
        embeddings = generate_image_embedding(img)
    
    elif content_type == "audio":
        # Use audio embedding model
        embeddings = generate_audio_embedding(content)
    
    return embeddings[0].values if hasattr(embeddings[0], 'values') else embeddings

# Example
text_embedding = embed_multimodal("text", "Red shoes")
image_embedding = embed_multimodal("image", "https://example.com/shoe.jpg")
```

### Challenges

```
1. Different modalities have different dimensions
   - Text: 768D
   - Image: 512D
   - Audio: 256D
   Solution: Project to common space

2. Cross-modal search is harder
   - Text query vs image document
   Solution: Use aligned embeddings (CLIP)

3. Higher computational cost
   - Multiple models to run
   Solution: Batch processing, caching

4. Training data is limited
   - Fewer multi-modal datasets
   Solution: Use pre-trained models
```

---

## Fine-Tuning Embeddings

### Definition
Train embedding model on your domain data

### Why Fine-Tune?

```
Pre-trained Model:
- General knowledge
- Works for most cases
- Accuracy: 90%

Fine-Tuned Model:
- Domain-specific knowledge
- Better for your data
- Accuracy: 95%+

Example:
- Pre-trained: "return" = "refund" (general)
- Fine-tuned: "return" = "refund" + "exchange" (e-commerce)
```

### Implementation

```python
from sentence_transformers import SentenceTransformer, InputExample, losses
from torch.utils.data import DataLoader

def finetune_embeddings(training_data: List[tuple]):
    """Fine-tune embedding model"""
    
    # Load pre-trained model
    model = SentenceTransformer('all-MiniLM-L6-v2')
    
    # Prepare training data
    # Format: (text1, text2, similarity_score)
    train_examples = [
        InputExample(texts=[text1, text2], label=score)
        for text1, text2, score in training_data
    ]
    
    # Create data loader
    train_dataloader = DataLoader(train_examples, shuffle=True, batch_size=16)
    
    # Define loss function
    train_loss = losses.CosineSimilarityLoss(model)
    
    # Fine-tune
    model.fit(
        train_objectives=[(train_dataloader, train_loss)],
        epochs=1,
        warmup_steps=100
    )
    
    return model

# Example training data
training_data = [
    ("return policy", "refund policy", 0.95),
    ("how to return", "how to refund", 0.90),
    ("product quality", "shipping speed", 0.10),
]

# Fine-tune
finetuned_model = finetune_embeddings(training_data)

# Use for embedding
embedding = finetuned_model.encode("return policy")
```

### When to Fine-Tune

```
Fine-Tune When:
- Domain-specific vocabulary
- Unique similarity relationships
- Accuracy is critical
- You have labeled data (100+ pairs)

Don't Fine-Tune When:
- Pre-trained works well
- Limited labeled data
- Cost is tight
- Time is limited
```

---

## Distributed Vector Search

### Definition
Vector search across multiple shards/regions

### Why Distributed?

```
Single Node:
- Max documents: 100M
- Max latency: 100ms
- Max throughput: 1000 QPS

Distributed:
- Max documents: 1B+
- Max latency: 200ms (parallel)
- Max throughput: 10K+ QPS
```

### Architecture

```
Query
  ↓
┌─────────────────────────────────────┐
│ Router (Load Balancer)              │
└────┬────────────────┬────────────────┘
     ↓                ↓
┌──────────────┐  ┌──────────────┐
│ Shard 1      │  │ Shard 2      │
│ (Region US)  │  │ (Region EU)  │
└────┬─────────┘  └────┬─────────┘
     ↓                ↓
┌─────────────────────────────────────┐
│ Aggregator (Merge Results)          │
└────┬────────────────────────────────┘
     ↓
Results
```

### Implementation

```python
from concurrent.futures import ThreadPoolExecutor

def distributed_search(query: str, shards: List[str], top_k: int = 3) -> List[dict]:
    """Search across multiple shards"""
    
    # Generate embedding once
    embedding = generate_query_embedding(query)
    
    # Search in parallel
    def search_shard(shard_url):
        return vector_search_remote(shard_url, embedding, top_k=10)
    
    with ThreadPoolExecutor(max_workers=len(shards)) as executor:
        shard_results = list(executor.map(search_shard, shards))
    
    # Merge results
    merged = {}
    for results in shard_results:
        for result in results:
            chunk_id = result['chunk_id']
            if chunk_id not in merged:
                merged[chunk_id] = result
            else:
                # Keep highest score
                if result['score'] > merged[chunk_id]['score']:
                    merged[chunk_id] = result
    
    # Sort and return top-k
    sorted_results = sorted(merged.values(), key=lambda x: x['score'], reverse=True)
    return sorted_results[:top_k]

# Example
shards = [
    "https://shard1.example.com",
    "https://shard2.example.com",
    "https://shard3.example.com"
]

results = distributed_search("return policy", shards, top_k=3)
```

### Challenges

```
1. Consistency
   - Different shards may have different data
   Solution: Replication, versioning

2. Latency
   - Slowest shard determines latency
   Solution: Timeout, fallback

3. Complexity
   - Harder to debug and maintain
   Solution: Monitoring, logging

4. Cost
   - Multiple shards = higher cost
   Solution: Efficient sharding strategy
```

---

## Real-Time Indexing

### Definition
Update index immediately when documents change

### Why Real-Time?

```
Batch Indexing:
- Update once per day
- Freshness: 24 hours old
- Cost: Low
- Complexity: Low

Real-Time Indexing:
- Update immediately
- Freshness: < 1 minute old
- Cost: High
- Complexity: High

Use Cases:
- News/breaking updates
- Stock prices
- Social media
- Real-time customer support
```

### Implementation

```python
from kafka import KafkaConsumer
import json

def realtime_indexing():
    """Real-time indexing pipeline"""
    
    # Listen to document changes
    consumer = KafkaConsumer(
        'document-updates',
        bootstrap_servers=['localhost:9092'],
        value_deserializer=lambda m: json.loads(m.decode('utf-8'))
    )
    
    for message in consumer:
        doc = message.value
        
        # Chunk document
        chunks = chunk_recursive(doc['content'])
        
        # Generate embeddings
        embeddings = []
        for chunk in chunks:
            embedding = generate_query_embedding(chunk)
            embeddings.append({
                'chunk_id': f"{doc['id']}_{len(embeddings)}",
                'doc_id': doc['id'],
                'embedding': embedding,
                'created_at': datetime.now().isoformat()
            })
        
        # Update vector index
        update_vector_index(embeddings)
        
        # Update metadata
        update_metadata(doc)
        
        print(f"Indexed {len(embeddings)} chunks for document {doc['id']}")

# Run in background
import threading
indexing_thread = threading.Thread(target=realtime_indexing, daemon=True)
indexing_thread.start()
```

### Challenges

```
1. Consistency
   - Index may be temporarily out of sync
   Solution: Versioning, eventual consistency

2. Performance
   - Real-time updates are expensive
   Solution: Batching, async processing

3. Complexity
   - Harder to manage
   Solution: Message queue, monitoring

4. Cost
   - Continuous processing
   Solution: Efficient algorithms, caching
```

---

## Comparison Table

```
| Feature | Hybrid | Multi-Modal | Fine-Tuned | Distributed | Real-Time |
|---------|--------|-------------|-----------|-------------|-----------|
| Accuracy | 95% | 90% | 98% | 95% | 95% |
| Latency | 300ms | 400ms | 100ms | 200ms | 50ms |
| Cost | 2x | 3x | 1x | 2x | 3x |
| Complexity | Medium | High | High | High | High |
| Best For | Mixed | Images | Domain | Scale | Updates |
```

---

## Key Takeaways

1. **Hybrid Search** - Combine vector + keyword for best results
2. **Multi-Modal** - Handle text, images, audio
3. **Fine-Tuning** - Improve accuracy for your domain
4. **Distributed** - Scale to billions of documents
5. **Real-Time** - Keep index fresh with streaming updates

