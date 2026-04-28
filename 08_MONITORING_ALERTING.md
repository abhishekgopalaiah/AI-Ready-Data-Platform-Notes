# Monitoring & Alerting: Production Observability

**Complete guide to monitoring RAG systems in production**

---

## Table of Contents
1. [Key Metrics](#key-metrics)
2. [SLO/SLI/SLA](#sloslisl)
3. [Alerting Strategy](#alerting-strategy)
4. [Dashboards](#dashboards)
5. [Incident Response](#incident-response)

---

## Key Metrics

### Latency Metrics

```
Embedding Latency:
- Definition: Time to generate query embedding
- Target: < 100ms
- Warning: > 150ms
- Critical: > 300ms
- Tool: Cloud Monitoring

Search Latency:
- Definition: Time to search vector DB
- Target: < 100ms
- Warning: > 150ms
- Critical: > 500ms
- Tool: Cloud Monitoring

Total Latency:
- Definition: End-to-end query time
- Target: < 200ms
- Warning: > 300ms
- Critical: > 500ms
- Tool: Application Insights
```

### Freshness Metrics

```
Embedding Age:
- Definition: How old are the embeddings?
- Target: < 7 days
- Warning: > 14 days
- Critical: > 30 days
- Tool: BigQuery

Last Update:
- Definition: When was index last updated?
- Target: Daily
- Warning: > 2 days
- Critical: > 7 days
- Tool: Airflow monitoring
```

### Accuracy Metrics

```
Search Accuracy:
- Definition: % of results that are relevant
- Target: > 90%
- Warning: < 85%
- Critical: < 80%
- Tool: Custom evaluation

Precision@K:
- Definition: % of top-K results that are relevant
- Target: > 90%
- Warning: < 85%
- Critical: < 80%
- Tool: Custom evaluation

Recall@K:
- Definition: % of relevant results in top-K
- Target: > 80%
- Warning: < 70%
- Critical: < 60%
- Tool: Custom evaluation
```

### Cost Metrics

```
Cost per Query:
- Definition: Embedding + search cost per query
- Target: < $0.001
- Warning: > $0.005
- Critical: > $0.01
- Tool: BigQuery cost analysis

Monthly Cost:
- Definition: Total monthly spend
- Target: < $500
- Warning: > $1000
- Critical: > $2000
- Tool: GCP billing

Cost per Document:
- Definition: Cost to embed and store one document
- Target: < $0.0001
- Warning: > $0.001
- Critical: > $0.01
- Tool: Custom calculation
```

### Reliability Metrics

```
Error Rate:
- Definition: % of queries that fail
- Target: < 0.1%
- Warning: > 1%
- Critical: > 5%
- Tool: Cloud Logging

Availability:
- Definition: % of time service is up
- Target: > 99.9%
- Warning: < 99%
- Critical: < 95%
- Tool: Uptime monitoring

Cache Hit Rate:
- Definition: % of queries served from cache
- Target: > 80%
- Warning: < 60%
- Critical: < 40%
- Tool: Cache metrics
```

---

## SLO/SLI/SLA

### Definitions

```
SLA (Service Level Agreement):
- Legal contract with users
- Example: "99.9% uptime guaranteed"
- Breach = compensation

SLO (Service Level Objective):
- Internal target
- Example: "Target 99.9% uptime"
- Breach = incident response

SLI (Service Level Indicator):
- Measurable metric
- Example: "Actual uptime: 99.95%"
- Used to track SLO
```

### Example SLOs

```
Latency SLO:
- p50: < 100ms
- p95: < 200ms
- p99: < 500ms

Accuracy SLO:
- Precision: > 90%
- Recall: > 80%
- F1 Score: > 85%

Availability SLO:
- Uptime: > 99.9%
- Error rate: < 0.1%
- Cache hit rate: > 80%

Freshness SLO:
- Embedding age: < 7 days
- Index update: Daily
- Metadata sync: Hourly
```

### Error Budget

```
SLO: 99.9% uptime
Error budget: 0.1% = 43 minutes per month

If you exceed error budget:
- Stop deploying new features
- Focus on reliability
- Investigate root cause
- Implement fixes

Example:
- Month 1: 99.95% uptime (within budget)
- Month 2: 99.85% uptime (exceeded budget)
- Action: Freeze deployments, fix issues
```

---

## Alerting Strategy

### Alert Rules

```
Latency Alert:
- Metric: p95 latency
- Threshold: > 200ms
- Duration: 5 minutes
- Action: Page on-call engineer

Freshness Alert:
- Metric: Embedding age
- Threshold: > 14 days
- Duration: 1 hour
- Action: Send email, create ticket

Accuracy Alert:
- Metric: Precision
- Threshold: < 85%
- Duration: 1 hour
- Action: Send email, create ticket

Cost Alert:
- Metric: Daily cost
- Threshold: > $50
- Duration: 1 hour
- Action: Send email, investigate
```

### Alert Severity Levels

```
P1 (Critical):
- Service down
- Accuracy < 80%
- Error rate > 5%
- Action: Page on-call immediately

P2 (High):
- Latency > 500ms
- Accuracy < 85%
- Error rate > 1%
- Action: Page on-call within 15 min

P3 (Medium):
- Latency > 200ms
- Accuracy < 90%
- Error rate > 0.1%
- Action: Send email, create ticket

P4 (Low):
- Latency > 100ms
- Accuracy < 95%
- Cost > budget
- Action: Send email
```

### Implementation

```python
from google.cloud import monitoring_v3

def create_alert_policy(project_id: str, metric_name: str, threshold: float):
    """Create alert policy"""
    
    client = monitoring_v3.AlertPolicyServiceClient()
    project_name = f"projects/{project_id}"
    
    # Define condition
    condition = monitoring_v3.AlertPolicy.Condition(
        display_name=f"Alert: {metric_name} > {threshold}",
        condition_threshold=monitoring_v3.AlertPolicy.Condition.MetricThreshold(
            filter=f'metric.type="custom.googleapis.com/rag/{metric_name}"',
            comparison=monitoring_v3.ComparisonType.COMPARISON_GT,
            threshold_value=threshold,
            duration={"seconds": 300},  # 5 minutes
        ),
    )
    
    # Define notification channel
    notification_channel = monitoring_v3.NotificationChannel(
        type="email",
        labels={"email_address": "alerts@company.com"},
    )
    
    # Create policy
    policy = monitoring_v3.AlertPolicy(
        display_name=f"Alert: {metric_name}",
        conditions=[condition],
        notification_channels=[notification_channel],
    )
    
    created_policy = client.create_alert_policy(
        name=project_name,
        alert_policy=policy
    )
    
    return created_policy

# Example
create_alert_policy("your-project-id", "query_latency_ms", 200)
```

---

## Dashboards

### Latency Dashboard

```
┌─────────────────────────────────────────────────────┐
│ RAG System - Latency Dashboard                      │
├─────────────────────────────────────────────────────┤
│                                                     │
│ ┌──────────────┐  ┌──────────────┐  ┌────────────┐ │
│ │ p50 Latency  │  │ p95 Latency  │  │ p99 Latency│ │
│ │   95ms       │  │   180ms      │  │   450ms    │ │
│ │ (Target:100) │  │ (Target:200) │  │(Target:500)│ │
│ └──────────────┘  └──────────────┘  └────────────┘ │
│                                                     │
│ Latency Breakdown:                                  │
│ ┌──────────────────────────────────────────────────┐│
│ │ Embedding: 100ms (50%)                           ││
│ │ Search: 75ms (37%)                               ││
│ │ Network: 25ms (13%)                              ││
│ └──────────────────────────────────────────────────┘│
│                                                     │
│ Latency Trend (Last 24h):                           │
│ ┌──────────────────────────────────────────────────┐│
│ │     ╱╲    ╱╲    ╱╲                                ││
│ │    ╱  ╲  ╱  ╲  ╱  ╲                               ││
│ │   ╱    ╲╱    ╲╱    ╲                              ││
│ └──────────────────────────────────────────────────┘│
│                                                     │
└─────────────────────────────────────────────────────┘
```

### Accuracy Dashboard

```
┌─────────────────────────────────────────────────────┐
│ RAG System - Accuracy Dashboard                     │
├─────────────────────────────────────────────────────┤
│                                                     │
│ ┌──────────────┐  ┌──────────────┐  ┌────────────┐ │
│ │ Precision    │  │ Recall       │  │ F1 Score   │ │
│ │   92%        │  │   88%        │  │   90%      │ │
│ │ (Target:90%) │  │ (Target:80%) │  │(Target:85%)│ │
│ └──────────────┘  └──────────────┘  └────────────┘ │
│                                                     │
│ Accuracy by Query Type:                             │
│ ┌──────────────────────────────────────────────────┐│
│ │ Return queries: 95%                              ││
│ │ Shipping queries: 88%                            ││
│ │ Payment queries: 85%                             ││
│ └──────────────────────────────────────────────────┘│
│                                                     │
│ Accuracy Trend (Last 7d):                           │
│ ┌──────────────────────────────────────────────────┐│
│ │ ▁▂▃▄▅▆▇█▇▆▅▄▃▂▁                                   ││
│ │ Avg: 90%                                         ││
│ └──────────────────────────────────────────────────┘│
│                                                     │
└─────────────────────────────────────────────────────┘
```

### Cost Dashboard

```
┌─────────────────────────────────────────────────────┐
│ RAG System - Cost Dashboard                         │
├─────────────────────────────────────────────────────┤
│                                                     │
│ ┌──────────────┐  ┌──────────────┐  ┌────────────┐ │
│ │ Daily Cost   │  │ Monthly Cost  │  │ Cost/Query │ │
│ │   $15        │  │   $450        │  │  $0.0005   │ │
│ │ (Budget:$20) │  │ (Budget:$500) │  │(Budget:$0.001)
│ └──────────────┘  └──────────────┘  └────────────┘ │
│                                                     │
│ Cost Breakdown:                                     │
│ ┌──────────────────────────────────────────────────┐│
│ │ Embedding: $200 (44%)                            ││
│ │ Storage: $150 (33%)                              ││
│ │ Search: $100 (23%)                               ││
│ └──────────────────────────────────────────────────┘│
│                                                     │
│ Cost Trend (Last 30d):                              │
│ ┌──────────────────────────────────────────────────┐│
│ │ ▂▃▄▅▆▇█▇▆▅▄▃▂▁▂▃▄▅▆▇█▇▆▅▄▃▂▁                     ││
│ │ Avg: $450/month                                  ││
│ └──────────────────────────────────────────────────┘│
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## Incident Response

### Incident Severity

```
P1 (Critical):
- Service completely down
- Accuracy < 50%
- Error rate > 10%
- Response time: Immediate
- Escalation: VP Engineering

P2 (High):
- Service degraded
- Accuracy < 80%
- Error rate > 1%
- Response time: 15 minutes
- Escalation: Engineering Manager

P3 (Medium):
- Service slow
- Accuracy < 90%
- Error rate > 0.1%
- Response time: 1 hour
- Escalation: Team Lead

P4 (Low):
- Minor issue
- Accuracy < 95%
- No errors
- Response time: Next business day
- Escalation: None
```

### Incident Runbook

```
Incident: High Latency (p95 > 200ms)

1. Verify Alert
   - Check dashboard
   - Confirm p95 latency > 200ms
   - Check error rate (should be low)

2. Identify Bottleneck
   - Check embedding latency
   - Check search latency
   - Check network latency
   - Check cache hit rate

3. Mitigate
   If embedding latency high:
   - Check Vertex AI quota
   - Check batch size
   - Reduce dimension size
   
   If search latency high:
   - Check vector DB load
   - Check index size
   - Enable caching
   
   If network latency high:
   - Check regional replicas
   - Check network connectivity

4. Root Cause Analysis
   - Review logs
   - Check metrics
   - Identify pattern
   - Document findings

5. Implement Fix
   - Deploy fix
   - Monitor metrics
   - Verify improvement

6. Post-Incident
   - Write incident report
   - Schedule retrospective
   - Implement preventive measures
```

---

## Key Takeaways

1. **Monitor latency, accuracy, cost, freshness**
2. **Set clear SLOs and error budgets**
3. **Alert on P1/P2 issues immediately**
4. **Create dashboards for visibility**
5. **Have incident runbooks ready**

