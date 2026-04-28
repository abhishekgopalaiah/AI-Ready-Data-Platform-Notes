# Phase 6: AI-Ready Data Platform

Complete implementation of RAG (Retrieval-Augmented Generation) system for building AI-ready data platforms.

## Project Structure

```
phase6_ai_platform/
├── setup.sh                    # GCP setup script
├── week1_embeddings.py         # Week 1: Embeddings & vector search
├── week2_pipeline.py           # Week 2: Ingestion pipeline
├── week3_airflow_dag.py        # Week 3: Airflow orchestration
├── week4_rag_system.py         # Week 4: Complete RAG system
├── requirements.txt            # Python dependencies
├── .env.example                # Environment variables template
└── README.md                   # This file
```

## Quick Start

### 1. Setup GCP Resources

```bash
# Make setup script executable
chmod +x setup.sh

# Run setup (replace with your GCP project ID)
./setup.sh your-gcp-project-id
```

This will:
- Enable required APIs (Vertex AI, BigQuery)
- Create service account with permissions
- Generate credentials file
- Create BigQuery dataset and tables

### 2. Install Dependencies

```bash
pip install -r requirements.txt
```

### 3. Set Environment Variables

```bash
# Set your GCP project ID
export GCP_PROJECT_ID="your-gcp-project-id"

# Set credentials path
export GOOGLE_APPLICATION_CREDENTIALS="$(pwd)/phase6-key.json"
```

### 4. Run Week 1 Code

```bash
python week1_embeddings.py
```

Expected output:
- ✓ Vertex AI initialized
- ✓ Model loaded
- ✓ 5 embeddings generated
- ✓ Vector search results for test queries
- ✓ Results saved to `week1_results.json`

## Week-by-Week Learning Path

### Week 1: Embeddings & Vector Search
**Goal:** Understand embeddings and how to search documents

**What you'll learn:**
- How embeddings work (768-dimensional vectors)
- Cosine similarity (measuring document closeness)
- Vector search (finding relevant documents)
- Embedding models and costs

**Code:** `week1_embeddings.py`
- Generate embeddings for 5 sample documents
- Test vector search with 4 queries
- Analyze embedding space
- Save results

**Time:** 2-3 hours

---

### Week 2: Building the Pipeline
**Goal:** Build end-to-end ingestion pipeline

**What you'll learn:**
- Chunking strategies (fixed, recursive, semantic)
- BigQuery schema design
- Document ingestion at scale
- Embedding storage and retrieval

**Code:** `week2_pipeline.py`
- Implement recursive chunking
- Create ingestion pipeline class
- Store documents and embeddings in BigQuery
- Query by similarity

**Time:** 3-4 hours

---

### Week 3: Airflow & Production
**Goal:** Orchestrate and monitor in production

**What you'll learn:**
- Airflow DAG design
- Freshness monitoring (SLOs)
- Access control enforcement
- Logging and alerting

**Code:** `week3_airflow_dag.py`
- Schedule embedding pipeline
- Monitor freshness SLOs
- Enforce access control
- Log events for audit

**Time:** 3-4 hours

---

### Week 4: Capstone RAG System
**Goal:** Build complete RAG system

**What you'll learn:**
- Retrieve context from vector index
- Generate answers with LLM
- Measure quality and latency
- Deploy to production

**Code:** `week4_rag_system.py`
- Retrieve relevant documents
- Generate answers with Gemini
- Measure performance
- Handle access control

**Time:** 2-3 hours

---

## Key Concepts

### Embeddings
Vectors that represent text meaning. Similar texts have similar embeddings.

```
"Return policy" → [0.12, -0.45, 0.89, ...]
"Refund process" → [0.11, -0.44, 0.88, ...]  (similar)
"Shipping info" → [0.50, 0.20, 0.30, ...]    (different)
```

### Vector Search
Finding K nearest neighbors in embedding space.

```
Query: "How do I return?"
↓
Find top-3 most similar documents
↓
Return to user
```

### Chunking
Splitting documents into manageable pieces.

```
Document (5000 words)
↓
Chunk 1 (500 words)
Chunk 2 (500 words)
...
Chunk 10 (500 words)
```

### RAG (Retrieval-Augmented Generation)
Combining retrieval with LLM generation.

```
User Question
↓
Retrieve relevant docs
↓
Pass to LLM with context
↓
LLM generates answer
```

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    User Query                            │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
         ┌───────────────────────┐
         │  Embed Query          │
         │  (Vertex AI)          │
         └───────────┬───────────┘
                     │
                     ▼
         ┌───────────────────────┐
         │  Vector Search        │
         │  (BigQuery)           │
         └───────────┬───────────┘
                     │
                     ▼
         ┌───────────────────────┐
         │  Retrieve Chunks      │
         │  (Apply ACL)          │
         └───────────┬───────────┘
                     │
                     ▼
         ┌───────────────────────┐
         │  Generate Answer      │
         │  (Gemini LLM)         │
         └───────────┬───────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│                    Answer to User                        │
└─────────────────────────────────────────────────────────┘
```

## GCP Resources Created

### BigQuery Dataset: `ai_platform`

**Tables:**
- `documents` - Source documents with metadata
- `chunks` - Document chunks with embeddings
- `access_audit` - Access log for compliance

### Service Account
- `phase6-sa` - Service account for API access
- Permissions: Vertex AI, BigQuery

## Costs

**Estimated monthly costs (1M documents, 10K queries/day):**

| Service | Cost |
|---------|------|
| Vertex AI Embeddings | $20 |
| BigQuery Storage | $5 |
| BigQuery Queries | $10 |
| **Total** | **~$35** |

## Troubleshooting

### Error: "Project not set"
```bash
export GCP_PROJECT_ID="your-project-id"
```

### Error: "Credentials not found"
```bash
export GOOGLE_APPLICATION_CREDENTIALS="$(pwd)/phase6-key.json"
```

### Error: "API not enabled"
```bash
gcloud services enable aiplatform.googleapis.com
gcloud services enable bigquery.googleapis.com
```

### Error: "Permission denied"
Check service account has required roles:
```bash
gcloud projects get-iam-policy YOUR_PROJECT \
  --flatten="bindings[].members" \
  --filter="bindings.members:phase6-sa*"
```

## Next Steps

1. ✓ Run `setup.sh` to create GCP resources
2. ✓ Run `week1_embeddings.py` to test embeddings
3. → Run `week2_pipeline.py` to build ingestion
4. → Run `week3_airflow_dag.py` to orchestrate
5. → Run `week4_rag_system.py` to build RAG

## Resources

- [Vertex AI Documentation](https://cloud.google.com/vertex-ai/docs)
- [BigQuery Documentation](https://cloud.google.com/bigquery/docs)
- [RAG Best Practices](https://cloud.google.com/vertex-ai/docs/generative-ai/rag-overview)
- [Embeddings Guide](https://cloud.google.com/vertex-ai/docs/generative-ai/embeddings/get-text-embeddings)

## Support

For issues or questions:
1. Check the learning guide: `/notes/self/AI-Ready/PHASE_6_AI_READY_DATA_PLATFORM.md`
2. Review code comments
3. Check GCP logs: `gcloud logging read`
