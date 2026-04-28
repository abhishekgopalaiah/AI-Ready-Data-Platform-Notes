# Production Guide: Building AI-Ready Systems

**Complete guide to deploying and maintaining RAG systems in production**

---

## Table of Contents
1. [Airflow DAG Patterns](#airflow-dag-patterns)
2. [Terraform Infrastructure](#terraform-infrastructure)
3. [Monitoring & Alerting](#monitoring--alerting)
4. [Error Handling](#error-handling)
5. [Cost Optimization](#cost-optimization)
6. [Troubleshooting](#troubleshooting)

---

## Airflow DAG Patterns

### Pattern 1: Daily Embedding Refresh

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.google.cloud.operators.bigquery import BigQueryCreateEmptyTableOperator
from datetime import datetime, timedelta

default_args = {
    'owner': 'data-platform',
    'retries': 2,
    'retry_delay': timedelta(minutes=5),
    'email_on_failure': True,
    'email': ['alerts@company.com'],
}

dag = DAG(
    'daily_embedding_refresh',
    default_args=default_args,
    description='Daily embedding refresh for RAG system',
    schedule_interval='0 2 * * *',  # 2 AM daily
    start_date=datetime(2024, 1, 1),
    catchup=False,
)

def load_new_documents():
    """Load documents modified in last 24 hours"""
    from google.cloud import bigquery
    client = bigquery.Client()
    
    query = """
    SELECT doc_id, content, modified_at
    FROM documents
    WHERE modified_at > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 24 HOUR)
    """
    
    results = client.query(query).result()
    return [dict(row) for row in results]

def chunk_documents(docs):
    """Chunk documents into 500-word chunks"""
    chunks = []
    for doc in docs:
        words = doc['content'].split()
        for i in range(0, len(words), 500):
            chunk = ' '.join(words[i:i+500])
            chunks.append({
                'doc_id': doc['doc_id'],
                'chunk_id': f"{doc['doc_id']}_{i//500}",
                'content': chunk,
                'modified_at': doc['modified_at']
            })
    return chunks

def embed_documents(chunks):
    """Generate embeddings using Vertex AI"""
    from google.cloud import aiplatform
    
    client = aiplatform.gapic.PredictionServiceClient(
        client_options={'api_endpoint': 'us-central1-aiplatform.googleapis.com'}
    )
    
    embeddings = []
    for chunk in chunks:
        response = client.predict(
            endpoint='projects/YOUR_PROJECT/locations/us-central1/endpoints/YOUR_ENDPOINT',
            instances=[{'content': chunk['content']}]
        )
        embeddings.append({
            'chunk_id': chunk['chunk_id'],
            'doc_id': chunk['doc_id'],
            'embedding': response.predictions[0]['embeddings']['values'],
            'created_at': datetime.now().isoformat()
        })
    
    return embeddings

def store_embeddings(embeddings):
    """Store embeddings in BigQuery"""
    from google.cloud import bigquery
    
    client = bigquery.Client()
    table_id = 'project.dataset.embeddings'
    
    errors = client.insert_rows_json(table_id, embeddings)
    if errors:
        raise Exception(f"Failed to insert embeddings: {errors}")
    
    return len(embeddings)

def update_vector_index(embeddings):
    """Update vector search index"""
    from google.cloud import aiplatform
    
    # Update Pinecone/Weaviate/BigQuery Vector Search
    # Implementation depends on your vector DB
    
    return f"Updated {len(embeddings)} embeddings"

# Define tasks
load_task = PythonOperator(
    task_id='load_documents',
    python_callable=load_new_documents,
)

chunk_task = PythonOperator(
    task_id='chunk_documents',
    python_callable=chunk_documents,
    op_args=[load_task.output],
)

embed_task = PythonOperator(
    task_id='embed_documents',
    python_callable=embed_documents,
    op_args=[chunk_task.output],
)

store_task = PythonOperator(
    task_id='store_embeddings',
    python_callable=store_embeddings,
    op_args=[embed_task.output],
)

index_task = PythonOperator(
    task_id='update_vector_index',
    python_callable=update_vector_index,
    op_args=[embed_task.output],
)

# Define dependencies
load_task >> chunk_task >> embed_task >> [store_task, index_task]
```

### Pattern 2: Hourly Query Logging

```python
dag = DAG(
    'hourly_query_logging',
    default_args=default_args,
    description='Log queries for monitoring',
    schedule_interval='0 * * * *',  # Every hour
    start_date=datetime(2024, 1, 1),
)

def aggregate_queries():
    """Aggregate queries from last hour"""
    from google.cloud import bigquery
    
    client = bigquery.Client()
    query = """
    SELECT 
        query,
        COUNT(*) as count,
        AVG(latency_ms) as avg_latency,
        MAX(latency_ms) as max_latency,
        CURRENT_TIMESTAMP() as logged_at
    FROM query_logs
    WHERE timestamp > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 1 HOUR)
    GROUP BY query
    """
    
    results = client.query(query).result()
    return [dict(row) for row in results]

def check_quality_metrics(queries):
    """Check if metrics are within SLO"""
    alerts = []
    
    for query_stat in queries:
        if query_stat['avg_latency'] > 200:  # SLO: <200ms
            alerts.append({
                'query': query_stat['query'],
                'issue': 'High latency',
                'value': query_stat['avg_latency'],
                'threshold': 200
            })
    
    if alerts:
        send_alert(alerts)
    
    return alerts

# Tasks
aggregate_task = PythonOperator(
    task_id='aggregate_queries',
    python_callable=aggregate_queries,
)

check_task = PythonOperator(
    task_id='check_quality',
    python_callable=check_quality_metrics,
    op_args=[aggregate_task.output],
)

aggregate_task >> check_task
```

---

## Terraform Infrastructure

### Complete Infrastructure Setup

```hcl
# variables.tf
variable "project_id" {
  type = string
}

variable "region" {
  type    = string
  default = "us-central1"
}

variable "environment" {
  type    = string
  default = "prod"
}

# main.tf
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}

# BigQuery Dataset
resource "google_bigquery_dataset" "rag_dataset" {
  dataset_id    = "rag_${var.environment}"
  friendly_name = "RAG System Dataset"
  description   = "Dataset for RAG embeddings and documents"
  location      = var.region

  access {
    role          = "OWNER"
    user_by_email = google_service_account.rag_sa.email
  }
}

# BigQuery Tables
resource "google_bigquery_table" "documents" {
  dataset_id = google_bigquery_dataset.rag_dataset.dataset_id
  table_id   = "documents"

  schema = jsonencode([
    {
      name        = "doc_id"
      type        = "STRING"
      mode        = "REQUIRED"
      description = "Document ID"
    },
    {
      name        = "content"
      type        = "STRING"
      mode        = "REQUIRED"
      description = "Document content"
    },
    {
      name        = "created_at"
      type        = "TIMESTAMP"
      mode        = "REQUIRED"
      description = "Creation timestamp"
    },
    {
      name        = "modified_at"
      type        = "TIMESTAMP"
      mode        = "REQUIRED"
      description = "Last modified timestamp"
    }
  ])
}

resource "google_bigquery_table" "embeddings" {
  dataset_id = google_bigquery_dataset.rag_dataset.dataset_id
  table_id   = "embeddings"

  schema = jsonencode([
    {
      name        = "chunk_id"
      type        = "STRING"
      mode        = "REQUIRED"
      description = "Chunk ID"
    },
    {
      name        = "doc_id"
      type        = "STRING"
      mode        = "REQUIRED"
      description = "Document ID"
    },
    {
      name        = "embedding"
      type        = "FLOAT64"
      mode        = "REPEATED"
      description = "Embedding vector (768 dimensions)"
    },
    {
      name        = "created_at"
      type        = "TIMESTAMP"
      mode        = "REQUIRED"
      description = "Creation timestamp"
    }
  ])

  clustering = ["doc_id"]
}

# Cloud Storage Bucket
resource "google_storage_bucket" "rag_bucket" {
  name          = "${var.project_id}-rag-${var.environment}"
  location      = var.region
  force_destroy = false

  versioning {
    enabled = true
  }

  lifecycle_rule {
    action {
      type = "Delete"
    }
    condition {
      num_newer_versions = 10
    }
  }
}

# Service Account
resource "google_service_account" "rag_sa" {
  account_id   = "rag-system-${var.environment}"
  display_name = "RAG System Service Account"
}

# IAM Roles
resource "google_project_iam_member" "rag_bigquery_editor" {
  project = var.project_id
  role    = "roles/bigquery.dataEditor"
  member  = "serviceAccount:${google_service_account.rag_sa.email}"
}

resource "google_project_iam_member" "rag_storage_admin" {
  project = var.project_id
  role    = "roles/storage.objectAdmin"
  member  = "serviceAccount:${google_service_account.rag_sa.email}"
}

resource "google_project_iam_member" "rag_aiplatform_user" {
  project = var.project_id
  role    = "roles/aiplatform.user"
  member  = "serviceAccount:${google_service_account.rag_sa.email}"
}

# Cloud Monitoring Alert Policy
resource "google_monitoring_alert_policy" "high_latency" {
  display_name = "RAG High Latency Alert"
  combiner     = "OR"

  conditions {
    display_name = "Query latency > 200ms"

    condition_threshold {
      filter          = "metric.type=\"custom.googleapis.com/rag/query_latency_ms\""
      duration        = "300s"
      comparison      = "COMPARISON_GT"
      threshold_value = 200

      aggregations {
        alignment_period  = "60s"
        per_series_aligner = "ALIGN_MEAN"
      }
    }
  }

  notification_channels = [google_monitoring_notification_channel.email.name]
}

resource "google_monitoring_notification_channel" "email" {
  display_name = "RAG Alerts Email"
  type         = "email"

  labels = {
    email_address = "alerts@company.com"
  }
}

# Outputs
output "dataset_id" {
  value = google_bigquery_dataset.rag_dataset.dataset_id
}

output "bucket_name" {
  value = google_storage_bucket.rag_bucket.name
}

output "service_account_email" {
  value = google_service_account.rag_sa.email
}
```

---

## Monitoring & Alerting

### Key Metrics to Monitor

```python
from google.cloud import monitoring_v3
import time

class RAGMonitoring:
    def __init__(self, project_id):
        self.client = monitoring_v3.MetricServiceClient()
        self.project_name = f"projects/{project_id}"
    
    def record_embedding_latency(self, latency_ms):
        """Record embedding generation latency"""
        series = monitoring_v3.TimeSeries()
        series.metric.type = 'custom.googleapis.com/rag/embedding_latency_ms'
        series.resource.type = 'global'
        
        now = time.time()
        seconds = int(now)
        nanos = int((now - seconds) * 10 ** 9)
        interval = monitoring_v3.TimeInterval(
            {"seconds": seconds, "nanos": nanos}
        )
        point = monitoring_v3.Point({"interval": interval, "value": {"double_value": latency_ms}})
        series.points = [point]
        
        self.client.create_time_series(name=self.project_name, time_series=[series])
    
    def record_search_latency(self, latency_ms):
        """Record vector search latency"""
        # Similar to embedding_latency
        pass
    
    def record_embedding_freshness(self, days_old):
        """Record how old embeddings are"""
        # Similar to embedding_latency
        pass
    
    def record_query_cost(self, cost_usd):
        """Record cost per query"""
        # Similar to embedding_latency
        pass

# Usage
monitoring = RAGMonitoring('your-project-id')

# In your embedding code
start = time.time()
embeddings = embed_documents(chunks)
latency = (time.time() - start) * 1000
monitoring.record_embedding_latency(latency)

# In your search code
start = time.time()
results = vector_search(query)
latency = (time.time() - start) * 1000
monitoring.record_search_latency(latency)
```

### Alert Thresholds

```
Embedding Latency:
- Warning: > 150ms
- Critical: > 300ms

Search Latency:
- Warning: > 100ms
- Critical: > 500ms

Embedding Freshness:
- Warning: > 7 days old
- Critical: > 30 days old

Query Cost:
- Warning: > $0.01 per query
- Critical: > $0.05 per query

Error Rate:
- Warning: > 1%
- Critical: > 5%
```

---

## Error Handling

### Retry Strategy

```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=2, max=10)
)
def embed_with_retry(text):
    """Embed with exponential backoff retry"""
    try:
        return embed_document(text)
    except Exception as e:
        print(f"Embedding failed: {e}")
        raise

@retry(
    stop=stop_after_attempt(5),
    wait=wait_exponential(multiplier=1, min=1, max=5)
)
def search_with_retry(query):
    """Search with retry"""
    try:
        return vector_search(query)
    except Exception as e:
        print(f"Search failed: {e}")
        raise
```

### Circuit Breaker Pattern

```python
from pybreaker import CircuitBreaker

# Create circuit breaker for embedding service
embedding_breaker = CircuitBreaker(
    fail_max=5,
    reset_timeout=60,
    listeners=[],
)

@embedding_breaker
def embed_with_circuit_breaker(text):
    """Embed with circuit breaker"""
    return embed_document(text)

# Usage
try:
    embedding = embed_with_circuit_breaker(text)
except Exception as e:
    print(f"Embedding service unavailable: {e}")
    # Fall back to cached embedding
    embedding = get_cached_embedding(text)
```

---

## Cost Optimization

### Cost Tracking

```python
def calculate_embedding_cost(num_tokens):
    """Calculate embedding cost"""
    cost_per_million = 0.02
    return (num_tokens / 1_000_000) * cost_per_million

def calculate_search_cost(num_queries):
    """Calculate search cost (BigQuery)"""
    # BigQuery charges per GB scanned
    bytes_per_query = 768 * 4  # 768 dimensions × 4 bytes
    gb_scanned = (num_queries * bytes_per_query) / (1024 ** 3)
    cost_per_gb = 6.25  # $6.25 per TB = $0.00625 per GB
    return gb_scanned * cost_per_gb

def calculate_storage_cost(num_embeddings):
    """Calculate storage cost"""
    bytes_per_embedding = 768 * 4  # 768 dimensions × 4 bytes
    gb_stored = (num_embeddings * bytes_per_embedding) / (1024 ** 3)
    cost_per_gb_month = 0.02  # $0.02 per GB per month
    return gb_stored * cost_per_gb_month

# Monthly cost calculation
num_documents = 5_000_000
num_chunks = 10_000_000
num_queries_per_day = 10_000

embedding_cost = calculate_embedding_cost(num_chunks * 667)  # 667 tokens per chunk
search_cost = calculate_search_cost(num_queries_per_day * 30)
storage_cost = calculate_storage_cost(num_chunks)

total_monthly_cost = embedding_cost + search_cost + storage_cost
print(f"Monthly cost: ${total_monthly_cost:.2f}")
```

### Cost Optimization Strategies

```
1. Batch Embedding
   - Embed 100 documents at once
   - Reduces API calls by 99%
   - Saves: 50% of embedding cost

2. Caching
   - Cache popular queries
   - Reduces search queries by 80%
   - Saves: 80% of search cost

3. Dimension Reduction
   - Use 384 dimensions instead of 768
   - Reduces storage by 50%
   - Saves: 50% of storage cost

4. Selective Re-embedding
   - Only re-embed modified documents
   - Re-embed monthly instead of daily
   - Saves: 95% of re-embedding cost

5. Compression
   - Quantize embeddings (int8 instead of float32)
   - Reduces storage by 75%
   - Saves: 75% of storage cost
```

---

## Troubleshooting

### Problem: High Latency (>500ms)

```
Diagnosis:
1. Check embedding latency
   - If > 100ms: Embedding service is slow
   - Solution: Use batch processing

2. Check search latency
   - If > 200ms: Vector search is slow
   - Solution: Add caching or reduce dimensions

3. Check network latency
   - If > 100ms: Network is slow
   - Solution: Use regional replicas

4. Check index size
   - If > 100GB: Index is too large
   - Solution: Use IVF instead of HNSW

Action Plan:
1. Measure each component
2. Identify bottleneck
3. Apply targeted optimization
4. Monitor improvement
```

### Problem: High Cost (>$500/month)

```
Diagnosis:
1. Check embedding cost
   - If > $200/month: Too many embeddings
   - Solution: Reduce re-embedding frequency

2. Check search cost
   - If > $200/month: Too many searches
   - Solution: Add caching

3. Check storage cost
   - If > $100/month: Storage is expensive
   - Solution: Use compression or reduce dimensions

Action Plan:
1. Calculate cost breakdown
2. Identify expensive component
3. Apply cost optimization
4. Monitor savings
```

### Problem: Low Accuracy (<90%)

```
Diagnosis:
1. Check embedding quality
   - If accuracy < 90%: Embeddings are poor
   - Solution: Use better model or larger dimensions

2. Check chunking strategy
   - If chunks are too large: Context is lost
   - Solution: Use smaller chunks

3. Check index quality
   - If index is stale: Results are outdated
   - Solution: Re-embed more frequently

Action Plan:
1. Measure accuracy
2. Identify root cause
3. Apply targeted improvement
4. Monitor accuracy
```

---

## Key Takeaways

1. **Automate with Airflow** - Daily refresh, hourly monitoring
2. **Infrastructure as Code** - Use Terraform for reproducibility
3. **Monitor Everything** - Latency, cost, freshness, accuracy
4. **Handle Errors Gracefully** - Retry, circuit breaker, fallback
5. **Optimize Costs** - Batch, cache, compress, selective re-embedding
6. **Troubleshoot Systematically** - Measure, diagnose, optimize, monitor

