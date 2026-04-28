# Code Examples: Working Implementation

**Production-ready code examples for RAG systems**

---

## Table of Contents
1. [Vertex AI Embeddings](#vertex-ai-embeddings)
2. [Vector Search](#vector-search)
3. [BigQuery Integration](#bigquery-integration)
4. [Monitoring](#monitoring)
5. [Error Handling](#error-handling)

---

## Vertex AI Embeddings

### Basic Embedding

```python
from google.cloud import aiplatform
import os

def embed_text(text: str) -> list:
    """Generate embedding for text using Vertex AI"""
    
    # Initialize client
    aiplatform.init(project="your-project-id", location="us-central1")
    
    # Create embedding request
    model = aiplatform.TextEmbeddingModel.from_pretrained(
        "textembedding-gecko@001"
    )
    
    embeddings = model.get_embeddings([text])
    
    return embeddings[0].values

# Usage
text = "How do I return an item?"
embedding = embed_text(text)
print(f"Embedding shape: {len(embedding)}")  # 768
print(f"First 5 values: {embedding[:5]}")
```

### Batch Embedding

```python
from google.cloud import aiplatform
from typing import List

def embed_batch(texts: List[str], batch_size: int = 100) -> List[List[float]]:
    """Generate embeddings for multiple texts"""
    
    aiplatform.init(project="your-project-id", location="us-central1")
    model = aiplatform.TextEmbeddingModel.from_pretrained(
        "textembedding-gecko@001"
    )
    
    all_embeddings = []
    
    # Process in batches
    for i in range(0, len(texts), batch_size):
        batch = texts[i:i+batch_size]
        embeddings = model.get_embeddings(batch)
        all_embeddings.extend([e.values for e in embeddings])
    
    return all_embeddings

# Usage
texts = [
    "How do I return an item?",
    "What's your return policy?",
    "Can I get a refund?",
]
embeddings = embed_batch(texts)
print(f"Generated {len(embeddings)} embeddings")
```

### Async Embedding

```python
import asyncio
from google.cloud import aiplatform
from typing import List

async def embed_async(texts: List[str]) -> List[List[float]]:
    """Generate embeddings asynchronously"""
    
    aiplatform.init(project="your-project-id", location="us-central1")
    model = aiplatform.TextEmbeddingModel.from_pretrained(
        "textembedding-gecko@001"
    )
    
    # Process in parallel
    tasks = []
    for text in texts:
        task = asyncio.create_task(
            asyncio.to_thread(model.get_embeddings, [text])
        )
        tasks.append(task)
    
    results = await asyncio.gather(*tasks)
    embeddings = [r[0].values for r in results]
    
    return embeddings

# Usage
async def main():
    texts = ["text1", "text2", "text3"]
    embeddings = await embed_async(texts)
    print(f"Generated {len(embeddings)} embeddings")

asyncio.run(main())
```

---

## Vector Search

### BigQuery Vector Search

```python
from google.cloud import bigquery
import numpy as np

def search_bigquery(query_embedding: List[float], top_k: int = 3) -> List[dict]:
    """Search for similar documents in BigQuery"""
    
    client = bigquery.Client()
    
    # Convert embedding to string for SQL
    embedding_str = ",".join(str(x) for x in query_embedding)
    
    query = f"""
    SELECT
        doc_id,
        content,
        COSINE_DISTANCE(embedding, [{embedding_str}]) as distance,
        1 - COSINE_DISTANCE(embedding, [{embedding_str}]) as similarity
    FROM `project.dataset.embeddings`
    ORDER BY distance ASC
    LIMIT {top_k}
    """
    
    results = client.query(query).result()
    
    return [
        {
            'doc_id': row.doc_id,
            'content': row.content,
            'similarity': row.similarity
        }
        for row in results
    ]

# Usage
query_embedding = embed_text("How do I return?")
results = search_bigquery(query_embedding, top_k=3)

for result in results:
    print(f"Doc: {result['doc_id']}")
    print(f"Similarity: {result['similarity']:.2f}")
    print(f"Content: {result['content'][:100]}...")
```

### Pinecone Vector Search

```python
import pinecone
from typing import List

def init_pinecone():
    """Initialize Pinecone client"""
    pinecone.init(
        api_key="your-api-key",
        environment="us-west1-gcp"
    )

def search_pinecone(query_embedding: List[float], top_k: int = 3) -> List[dict]:
    """Search for similar documents in Pinecone"""
    
    index = pinecone.Index("rag-index")
    
    # Search
    results = index.query(
        vector=query_embedding,
        top_k=top_k,
        include_metadata=True
    )
    
    return [
        {
            'doc_id': match.metadata['doc_id'],
            'content': match.metadata['content'],
            'similarity': match.score
        }
        for match in results.matches
    ]

# Usage
init_pinecone()
query_embedding = embed_text("How do I return?")
results = search_pinecone(query_embedding, top_k=3)

for result in results:
    print(f"Doc: {result['doc_id']}")
    print(f"Similarity: {result['similarity']:.2f}")
```

### Weaviate Vector Search

```python
import weaviate
from typing import List

def search_weaviate(query_embedding: List[float], top_k: int = 3) -> List[dict]:
    """Search for similar documents in Weaviate"""
    
    client = weaviate.Client("http://localhost:8080")
    
    # Search
    results = client.query.get(
        "Document"
    ).with_near_vector(
        {"vector": query_embedding}
    ).with_limit(
        top_k
    ).with_additional(
        ["distance"]
    ).do()
    
    return [
        {
            'doc_id': doc['doc_id'],
            'content': doc['content'],
            'similarity': 1 - doc['_additional']['distance']
        }
        for doc in results['data']['Get']['Document']
    ]

# Usage
query_embedding = embed_text("How do I return?")
results = search_weaviate(query_embedding, top_k=3)

for result in results:
    print(f"Doc: {result['doc_id']}")
    print(f"Similarity: {result['similarity']:.2f}")
```

---

## BigQuery Integration

### Load Documents

```python
from google.cloud import bigquery
from typing import List

def load_documents(file_path: str) -> List[dict]:
    """Load documents from CSV"""
    import csv
    
    documents = []
    with open(file_path, 'r') as f:
        reader = csv.DictReader(f)
        for row in reader:
            documents.append({
                'doc_id': row['id'],
                'content': row['content'],
                'created_at': row['created_at']
            })
    
    return documents

def store_documents(documents: List[dict]):
    """Store documents in BigQuery"""
    
    client = bigquery.Client()
    table_id = "project.dataset.documents"
    
    errors = client.insert_rows_json(table_id, documents)
    
    if errors:
        print(f"Errors: {errors}")
    else:
        print(f"Stored {len(documents)} documents")

# Usage
documents = load_documents("documents.csv")
store_documents(documents)
```

### Store Embeddings

```python
from google.cloud import bigquery
from typing import List

def store_embeddings(embeddings: List[dict]):
    """Store embeddings in BigQuery"""
    
    client = bigquery.Client()
    table_id = "project.dataset.embeddings"
    
    errors = client.insert_rows_json(table_id, embeddings)
    
    if errors:
        print(f"Errors: {errors}")
    else:
        print(f"Stored {len(embeddings)} embeddings")

# Usage
embeddings = [
    {
        'chunk_id': 'doc1_0',
        'doc_id': 'doc1',
        'embedding': [0.12, -0.45, 0.89, ...],
        'created_at': '2024-01-01T00:00:00Z'
    },
    ...
]
store_embeddings(embeddings)
```

### Query Embeddings

```python
from google.cloud import bigquery

def get_embedding_stats():
    """Get statistics about embeddings"""
    
    client = bigquery.Client()
    
    query = """
    SELECT
        COUNT(*) as total_embeddings,
        COUNT(DISTINCT doc_id) as unique_documents,
        MIN(created_at) as oldest_embedding,
        MAX(created_at) as newest_embedding,
        TIMESTAMP_DIFF(CURRENT_TIMESTAMP(), MAX(created_at), DAY) as days_since_latest
    FROM `project.dataset.embeddings`
    """
    
    results = client.query(query).result()
    
    for row in results:
        print(f"Total embeddings: {row.total_embeddings}")
        print(f"Unique documents: {row.unique_documents}")
        print(f"Days since latest: {row.days_since_latest}")

# Usage
get_embedding_stats()
```

---

## Monitoring

### Track Latency

```python
import time
from google.cloud import monitoring_v3

class LatencyTracker:
    def __init__(self, project_id: str):
        self.client = monitoring_v3.MetricServiceClient()
        self.project_name = f"projects/{project_id}"
    
    def record_embedding_latency(self, latency_ms: float):
        """Record embedding latency"""
        series = monitoring_v3.TimeSeries()
        series.metric.type = 'custom.googleapis.com/rag/embedding_latency_ms'
        series.resource.type = 'global'
        
        now = time.time()
        seconds = int(now)
        nanos = int((now - seconds) * 10 ** 9)
        interval = monitoring_v3.TimeInterval(
            {"seconds": seconds, "nanos": nanos}
        )
        point = monitoring_v3.Point({
            "interval": interval,
            "value": {"double_value": latency_ms}
        })
        series.points = [point]
        
        self.client.create_time_series(
            name=self.project_name,
            time_series=[series]
        )
    
    def record_search_latency(self, latency_ms: float):
        """Record search latency"""
        # Similar to embedding_latency
        pass

# Usage
tracker = LatencyTracker('your-project-id')

start = time.time()
embedding = embed_text("How do I return?")
latency = (time.time() - start) * 1000
tracker.record_embedding_latency(latency)

start = time.time()
results = search_bigquery(embedding)
latency = (time.time() - start) * 1000
tracker.record_search_latency(latency)
```

### Track Cost

```python
from google.cloud import bigquery

def calculate_cost():
    """Calculate monthly cost"""
    
    client = bigquery.Client()
    
    # Get embedding cost
    query = """
    SELECT
        COUNT(*) as total_embeddings,
        COUNT(*) * 667 / 1_000_000 * 0.02 as embedding_cost
    FROM `project.dataset.embeddings`
    WHERE created_at > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
    """
    
    results = client.query(query).result()
    for row in results:
        print(f"Embedding cost: ${row.embedding_cost:.2f}")
    
    # Get storage cost
    query = """
    SELECT
        COUNT(*) as total_embeddings,
        COUNT(*) * 768 * 4 / (1024 ** 3) * 0.02 as storage_cost_monthly
    FROM `project.dataset.embeddings`
    """
    
    results = client.query(query).result()
    for row in results:
        print(f"Storage cost: ${row.storage_cost_monthly:.2f}/month")

# Usage
calculate_cost()
```

---

## Error Handling

### Retry with Exponential Backoff

```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=2, max=10)
)
def embed_with_retry(text: str) -> List[float]:
    """Embed with retry"""
    try:
        return embed_text(text)
    except Exception as e:
        print(f"Embedding failed: {e}")
        raise

# Usage
try:
    embedding = embed_with_retry("How do I return?")
except Exception as e:
    print(f"Failed after retries: {e}")
```

### Circuit Breaker

```python
from pybreaker import CircuitBreaker

embedding_breaker = CircuitBreaker(
    fail_max=5,
    reset_timeout=60,
)

@embedding_breaker
def embed_with_circuit_breaker(text: str) -> List[float]:
    """Embed with circuit breaker"""
    return embed_text(text)

# Usage
try:
    embedding = embed_with_circuit_breaker("How do I return?")
except Exception as e:
    print(f"Circuit breaker open: {e}")
    # Fall back to cached embedding
    embedding = get_cached_embedding("How do I return?")
```

### Graceful Degradation

```python
def search_with_fallback(query: str, top_k: int = 3) -> List[dict]:
    """Search with fallback to cached results"""
    
    try:
        # Try vector search
        query_embedding = embed_text(query)
        results = search_bigquery(query_embedding, top_k)
        return results
    
    except Exception as e:
        print(f"Vector search failed: {e}")
        
        try:
            # Fall back to keyword search
            results = keyword_search(query, top_k)
            print("Using keyword search fallback")
            return results
        
        except Exception as e2:
            print(f"Keyword search also failed: {e2}")
            
            # Fall back to cached results
            results = get_cached_results(query)
            print("Using cached results fallback")
            return results

# Usage
results = search_with_fallback("How do I return?")
```

---

## Complete Example: RAG Pipeline

```python
from typing import List
import time

class RAGPipeline:
    def __init__(self, project_id: str):
        self.project_id = project_id
        self.tracker = LatencyTracker(project_id)
    
    def ingest_documents(self, documents: List[dict]):
        """Ingest documents"""
        print(f"Ingesting {len(documents)} documents...")
        store_documents(documents)
    
    def chunk_documents(self, documents: List[dict], chunk_size: int = 500) -> List[dict]:
        """Chunk documents"""
        chunks = []
        for doc in documents:
            words = doc['content'].split()
            for i in range(0, len(words), chunk_size):
                chunk = ' '.join(words[i:i+chunk_size])
                chunks.append({
                    'chunk_id': f"{doc['doc_id']}_{i//chunk_size}",
                    'doc_id': doc['doc_id'],
                    'content': chunk
                })
        return chunks
    
    def embed_documents(self, chunks: List[dict]) -> List[dict]:
        """Embed documents"""
        print(f"Embedding {len(chunks)} chunks...")
        
        embeddings = []
        for i, chunk in enumerate(chunks):
            start = time.time()
            embedding = embed_text(chunk['content'])
            latency = (time.time() - start) * 1000
            
            self.tracker.record_embedding_latency(latency)
            
            embeddings.append({
                'chunk_id': chunk['chunk_id'],
                'doc_id': chunk['doc_id'],
                'embedding': embedding,
                'created_at': time.strftime('%Y-%m-%dT%H:%M:%SZ', time.gmtime())
            })
            
            if (i + 1) % 100 == 0:
                print(f"Embedded {i + 1}/{len(chunks)} chunks")
        
        return embeddings
    
    def store_embeddings(self, embeddings: List[dict]):
        """Store embeddings"""
        print(f"Storing {len(embeddings)} embeddings...")
        store_embeddings(embeddings)
    
    def search(self, query: str, top_k: int = 3) -> List[dict]:
        """Search for similar documents"""
        start = time.time()
        
        query_embedding = embed_text(query)
        results = search_bigquery(query_embedding, top_k)
        
        latency = (time.time() - start) * 1000
        self.tracker.record_search_latency(latency)
        
        return results

# Usage
pipeline = RAGPipeline('your-project-id')

# Load documents
documents = load_documents("documents.csv")

# Ingest
pipeline.ingest_documents(documents)

# Chunk
chunks = pipeline.chunk_documents(documents)

# Embed
embeddings = pipeline.embed_documents(chunks)

# Store
pipeline.store_embeddings(embeddings)

# Search
results = pipeline.search("How do I return?", top_k=3)
for result in results:
    print(f"Doc: {result['doc_id']}, Similarity: {result['similarity']:.2f}")
```

---

## Key Takeaways

1. **Use batch embedding** for efficiency
2. **Handle errors gracefully** with retry and fallback
3. **Monitor latency and cost** continuously
4. **Test with small data first** before scaling
5. **Use async for parallel processing** when possible

