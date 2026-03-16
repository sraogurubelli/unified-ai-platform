# Deploy Full Platform

## Overview

Deploy the **complete platform** with both Traditional and AI stacks for maximum flexibility and capabilities. This is the recommended production configuration.

**Use Cases**:
- Production SaaS + AI platform
- All features enabled
- Maximum flexibility
- Enterprise deployments

---

## Architecture

```
┌────────────────────────────────────────────────────────────┐
│ Gateway: API Gateway + MCP Gateway                         │
└───────────────────────┬────────────────────────────────────┘
                        ▼
┌────────────────────────────────────────────────────────────┐
│ Control Plane: Traditional + AI + Shared                   │
│ • Full IAM, RBAC, Multi-tenancy                           │
│ • API quotas + LLM quotas                                  │
│ • Billing + Cost tracking                                  │
│ • Full observability                                       │
└───────────────────────┬────────────────────────────────────┘
                        ▼
┌────────────────────────────────────────────────────────────┐
│ Services: Traditional + AI                                 │
│ Traditional: Auth, Projects, Documents, Webhooks           │
│ AI: LLM Gateway, Agents, RAG, Memory                       │
└───────────────────────┬────────────────────────────────────┘
                        ▼
┌────────────────────────────────────────────────────────────┐
│ Data: Traditional + AI Data Platforms                      │
│ Traditional: PostgreSQL, ClickHouse, Redis                 │
│ AI: Qdrant, Neo4j, Semantic Cache                         │
└────────────────────────────────────────────────────────────┘
```

---

## Deployment Configuration

### Docker Compose (Development/Small Scale)

```yaml
# docker-compose.yml

version: '3.8'

services:
  # ============ Gateway Layer ============
  nginx:
    image: nginx:latest
    ports:
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - api-gateway
      - mcp-gateway

  api-gateway:
    image: cortex-ai/api-gateway:latest
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://cortex:${POSTGRES_PASSWORD}@postgres:5432/cortex
      - REDIS_URL=redis://redis:6379
      - JWT_SECRET=${JWT_SECRET}
    depends_on:
      - postgres
      - redis

  mcp-gateway:
    image: cortex-ai/mcp-gateway:latest
    ports:
      - "8001:8001"
    environment:
      - JWT_SECRET=${JWT_SECRET}
      - MCP_SERVER_URL=http://mcp-server:8002

  # ============ Traditional Services ============
  auth-service:
    image: cortex-ai/auth-service:latest
    environment:
      - DATABASE_URL=postgresql://cortex:${POSTGRES_PASSWORD}@postgres:5432/cortex
      - JWT_SECRET=${JWT_SECRET}

  project-service:
    image: cortex-ai/project-service:latest
    environment:
      - DATABASE_URL=postgresql://cortex:${POSTGRES_PASSWORD}@postgres:5432/cortex

  document-service:
    image: cortex-ai/document-service:latest
    environment:
      - DATABASE_URL=postgresql://cortex:${POSTGRES_PASSWORD}@postgres:5432/cortex
      - S3_BUCKET=${S3_BUCKET}
      - AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
      - AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}

  webhook-service:
    image: cortex-ai/webhook-service:latest
    environment:
      - DATABASE_URL=postgresql://cortex:${POSTGRES_PASSWORD}@postgres:5432/cortex

  analytics-service:
    image: cortex-ai/analytics-service:latest
    environment:
      - CLICKHOUSE_URL=clickhouse://clickhouse:8123

  # ============ AI Services ============
  mcp-server:
    image: cortex-ai/mcp-server:latest
    ports:
      - "8002:8002"
    environment:
      - QDRANT_URL=http://qdrant:6333
      - NEO4J_URL=bolt://neo4j:7687

  llm-gateway:
    image: cortex-ai/llm-gateway:latest
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
      - REDIS_URL=redis://redis:6379
      - QDRANT_URL=http://qdrant:6333

  rag-service:
    image: cortex-ai/rag-service:latest
    environment:
      - QDRANT_URL=http://qdrant:6333
      - NEO4J_URL=bolt://neo4j:7687

  agent-orchestrator:
    image: cortex-ai/agent-orchestrator:latest
    environment:
      - LLM_GATEWAY_URL=http://llm-gateway:8003
      - MCP_GATEWAY_URL=http://mcp-gateway:8001

  memory-service:
    image: cortex-ai/memory-service:latest
    environment:
      - REDIS_URL=redis://redis:6379
      - QDRANT_URL=http://qdrant:6333
      - DATABASE_URL=postgresql://cortex:${POSTGRES_PASSWORD}@postgres:5432/cortex

  # ============ Traditional Data Platform ============
  postgres:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=cortex
      - POSTGRES_USER=cortex
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  clickhouse:
    image: clickhouse/clickhouse-server:latest
    volumes:
      - clickhouse_data:/var/lib/clickhouse
    ports:
      - "8123:8123"
      - "9000:9000"

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"

  # ============ AI Data Platform ============
  qdrant:
    image: qdrant/qdrant:latest
    ports:
      - "6333:6333"
    volumes:
      - qdrant_storage:/qdrant/storage

  neo4j:
    image: neo4j:5-community
    ports:
      - "7474:7474"
      - "7687:7687"
    environment:
      - NEO4J_AUTH=neo4j/${NEO4J_PASSWORD}
      - NEO4J_PLUGINS=["graph-data-science"]
    volumes:
      - neo4j_data:/data

  # ============ Observability ============
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
    volumes:
      - grafana_data:/var/lib/grafana

  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"
      - "6831:6831/udp"

volumes:
  postgres_data:
  clickhouse_data:
  redis_data:
  qdrant_storage:
  neo4j_data:
  prometheus_data:
  grafana_data:
```

### Kubernetes (Production)

Due to length, see [k8s-full-platform.yaml](./k8s-manifests/k8s-full-platform.yaml) for complete manifest.

**Key components**:
- 3 replicas of API Gateway (horizontal scaling)
- 3 replicas of MCP Gateway
- StatefulSets for databases (PostgreSQL, ClickHouse, Qdrant, Neo4j)
- HorizontalPodAutoscaler for dynamic scaling
- Ingress for SSL termination
- Persistent Volume Claims for data
- Secrets for sensitive config

---

## Cost Estimation

### Infrastructure Costs (AWS - Production)

| Resource | Size | Monthly Cost |
|----------|------|--------------|
| **Gateway Layer** | |
| API Gateway (3x t3.large) | $150 |
| MCP Gateway (3x t3.large) | $150 |
| Load Balancer (ALB) | $25 |
| **Traditional Services** | |
| Service pods (6x t3.medium) | $150 |
| **AI Services** | |
| AI pods (6x t3.large) | $300 |
| **Traditional Data** | |
| PostgreSQL (db.r5.2xlarge) | $500 |
| ClickHouse (3x r5.2xlarge) | $1,200 |
| Redis (cache.r5.xlarge) | $200 |
| **AI Data** | |
| Qdrant (3x r5.2xlarge) | $1,200 |
| Neo4j (r5.xlarge) | $300 |
| **Storage** | |
| EBS volumes (2TB total) | $200 |
| S3 (documents, backups) | $50 |
| **Observability** | |
| Prometheus/Grafana/Jaeger | $100 |
| **LLM Costs** | |
| OpenAI/Anthropic usage | $1,000-10,000+ |
| **Subtotal (Infrastructure)** | | **~$4,525/month** |
| **Total (with LLM)** | | **~$5,525-14,525/month** |

### Cost Optimization

With semantic caching (90% hit rate):
- **LLM costs reduced**: $1,000-10,000 → $100-1,000
- **New total**: **~$4,625-5,525/month**
- **Savings**: **~$900-9,000/month**

---

## Scaling Configuration

### Horizontal Pod Autoscaler (HPA)

```yaml
# hpa-api-gateway.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-gateway-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-gateway
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"

---
# hpa-llm-gateway.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: llm-gateway-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: llm-gateway
  minReplicas: 3
  maxReplicas: 30
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 75
  - type: Pods
    pods:
      metric:
        name: llm_requests_per_second
      target:
        type: AverageValue
        averageValue: "50"
```

### Database Scaling

**PostgreSQL**:
- Read replicas (2-3) for read-heavy workloads
- Connection pooling (PgBouncer)
- Partitioning for large tables

**Qdrant**:
- Sharding by tenant_id (3-10 shards)
- HNSW index optimization (M=16, ef_construct=100)

**ClickHouse**:
- Distributed tables across 3+ nodes
- Materialized views for common queries

**Neo4j**:
- Read replicas for graph queries
- Graph caching in Redis

---

## High Availability (HA) Configuration

### Multi-AZ Deployment

```yaml
# Spread pods across availability zones
apiVersion: v1
kind: Pod
metadata:
  name: api-gateway
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - api-gateway
        topologyKey: failure-domain.beta.kubernetes.io/zone
```

### Database Replication

**PostgreSQL**:
```yaml
# Primary-Standby replication
primary:
  host: postgres-primary.cortex.svc.cluster.local

standby:
  - host: postgres-standby-1.cortex.svc.cluster.local
  - host: postgres-standby-2.cortex.svc.cluster.local

replication:
  mode: streaming
  sync: true
```

---

## Monitoring & Alerts

### Prometheus Alerts

```yaml
# prometheus-alerts.yaml
groups:
  - name: platform_health
    interval: 30s
    rules:
      # Gateway health
      - alert: HighGatewayLatency
        expr: histogram_quantile(0.95, gateway_latency_seconds) > 2
        for: 5m
        annotations:
          summary: "Gateway p95 latency exceeds 2s"

      # Database health
      - alert: HighDatabaseConnections
        expr: pg_stat_database_numbackends / pg_settings_max_connections > 0.8
        for: 5m
        annotations:
          summary: "PostgreSQL connection pool >80%"

      # AI platform health
      - alert: LowSemanticCacheHitRate
        expr: semantic_cache_hit_rate < 0.85
        for: 10m
        annotations:
          summary: "Semantic cache hit rate below 85%"

      # LLM costs
      - alert: HighLLMCosts
        expr: rate(llm_cost_usd_total[1h]) > 100
        for: 10m
        annotations:
          summary: "LLM costs exceeding $100/hour"
```

---

## Backup & Disaster Recovery

### Automated Backups

```yaml
# backup-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: database-backup
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: cortex-ai/backup-tool:latest
            env:
            - name: BACKUP_TARGETS
              value: "postgres,qdrant,neo4j"
            - name: S3_BUCKET
              value: "cortex-backups"
            command:
            - /bin/sh
            - -c
            - |
              # PostgreSQL
              pg_dump -h postgres -U cortex cortex | gzip > /tmp/postgres-$(date +%Y%m%d).sql.gz
              aws s3 cp /tmp/postgres-$(date +%Y%m%d).sql.gz s3://cortex-backups/postgres/

              # Qdrant snapshots
              curl -X POST http://qdrant:6333/collections/embeddings/snapshots

              # Neo4j backup
              neo4j-admin dump --database=neo4j --to=/tmp/neo4j-$(date +%Y%m%d).dump
              aws s3 cp /tmp/neo4j-$(date +%Y%m%d).dump s3://cortex-backups/neo4j/
          restartPolicy: OnFailure
```

### Recovery Time Objectives

| Component | RTO | RPO |
|-----------|-----|-----|
| API Gateway | <5 min | 0 (stateless) |
| PostgreSQL | <30 min | <5 min |
| ClickHouse | <1 hour | <15 min |
| Qdrant | <1 hour | <30 min |
| Neo4j | <1 hour | <30 min |

---

## Security Configuration

### Network Policies

```yaml
# network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

---
# Allow API Gateway → Services
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-gateway
spec:
  podSelector:
    matchLabels:
      tier: services
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api-gateway
```

### Secrets Management

```yaml
# Using AWS Secrets Manager
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: cortex-secrets
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: cortex-app-secrets
  data:
  - secretKey: jwt-secret
    remoteRef:
      key: cortex/jwt-secret
  - secretKey: openai-api-key
    remoteRef:
      key: cortex/openai-api-key
```

---

**Next**: [Team Ownership](./05d-team-ownership.md) | [Traditional-Only](./05a-deploy-traditional-only.md) | [AI-Only](./05b-deploy-ai-only.md) | [Back to Overview](../00-parallel-stacks-overview.md)
