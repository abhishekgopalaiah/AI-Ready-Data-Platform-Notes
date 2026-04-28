# Phase 6 Quick Start Guide

Get up and running in 15 minutes.

## Prerequisites

- GCP account with billing enabled
- `gcloud` CLI installed
- Python 3.9+
- Terminal access

## Step 1: Clone/Setup (2 min)

```bash
cd /Users/4906031/Projects/phase6_ai_platform
```

## Step 2: Run GCP Setup (5 min)

```bash
# Make script executable
chmod +x setup.sh

# Run setup (replace YOUR_PROJECT_ID)
./setup.sh YOUR_PROJECT_ID

# Example:
# ./setup.sh my-ai-platform-project
```

This creates:
- ✓ Service account
- ✓ BigQuery dataset
- ✓ 3 tables (documents, chunks, audit)
- ✓ Credentials file

## Step 3: Install Dependencies (2 min)

```bash
pip install -r requirements.txt
```

## Step 4: Set Environment (1 min)

```bash
# Copy example config
cp .env.example .env

# Edit .env with your project ID
nano .env

# Export variables
export GCP_PROJECT_ID="your-project-id"
export GOOGLE_APPLICATION_CREDENTIALS="$(pwd)/phase6-key.json"
```

## Step 5: Run Week 1 Code (5 min)

```bash
python week1_embeddings.py
```

Expected output:
```
============================================================
Phase 6 Week 1: Embeddings & Vector Search
============================================================
Project: your-project-id

1. Initializing Vertex AI...
✓ Vertex AI initialized

2. Loading embedding model...
✓ Model loaded: textembedding-gecko@001 (768 dimensions)

3. Generating embeddings for documents...
✓ Generated 5 embeddings
✓ Embedding dimensions: 768

4. Testing vector search with sample queries...
Query: 'How do I return a product?'
  Top 3 results:
    1. Return Policy: 0.8765
    2. Customer Support: 0.5432
    3. Store Hours: 0.4123

...

6. Saving results...
✓ Results saved to: week1_results.json

============================================================
Key Insights from Week 1
============================================================

✓ Embeddings are 768-dimensional vectors
✓ Similar documents have similar embeddings
✓ Cosine similarity measures how close vectors are (0-1)
✓ Vector search finds relevant docs without keyword matching
✓ Works across languages and paraphrases

Next: Week 2 - Build the ingestion pipeline
============================================================
```

## Verify Success

Check if these files were created:

```bash
# Check credentials
ls -la phase6-key.json

# Check BigQuery dataset
bq ls ai_platform

# Check results
cat week1_results.json
```

## Troubleshooting

### "Project not found"
```bash
# Set your project ID
export GCP_PROJECT_ID="your-actual-project-id"

# Verify
echo $GCP_PROJECT_ID
```

### "Permission denied"
```bash
# Check service account has permissions
gcloud projects get-iam-policy $GCP_PROJECT_ID \
  --flatten="bindings[].members" \
  --filter="bindings.members:phase6-sa*"
```

### "API not enabled"
```bash
# Re-run setup
./setup.sh $GCP_PROJECT_ID
```

## What's Next?

After Week 1 works:

1. **Week 2:** Build ingestion pipeline
   ```bash
   python week2_pipeline.py
   ```

2. **Week 3:** Deploy Airflow DAG
   ```bash
   python week3_airflow_dag.py
   ```

3. **Week 4:** Build RAG system
   ```bash
   python week4_rag_system.py
   ```

## Key Files

| File | Purpose |
|------|---------|
| `setup.sh` | Create GCP resources |
| `week1_embeddings.py` | Learn embeddings |
| `week2_pipeline.py` | Build pipeline |
| `week3_airflow_dag.py` | Orchestrate |
| `week4_rag_system.py` | Build RAG |
| `requirements.txt` | Dependencies |
| `.env` | Configuration |

## Costs

**Week 1 alone:** ~$0.01 (5 documents, 4 queries)

**Full pipeline (monthly):** ~$35 for 1M documents

## Support

1. Check README.md for detailed docs
2. Review code comments
3. Check learning guide: `/notes/self/AI-Ready/PHASE_6_AI_READY_DATA_PLATFORM.md`

---

**You're ready to start Phase 6! 🚀**
