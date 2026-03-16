# Deploy Traditional-Only Stack

## Overview

Deploy the platform **without AI features** for traditional SaaS use cases. This configuration provides REST APIs, CRUD operations, user management, and analytics without LLM, vector search, or knowledge graph capabilities.

**Use Cases**:
- MVP/prototyping phase before adding AI
- Cost-sensitive deployments
- Compliance requirements (no AI/LLM)
- Traditional SaaS applications

---

## Architecture

```
┌────────────────────────────────────────────────────────────┐
│ Gateway: API Gateway Only                                  │
│ (No MCP Gateway)                                           │
└───────────────────────┬────────────────────────────────────┘
                        ▼
┌────────────────────────────────────────────────────────────┐
│ Control Plane: Traditional + Shared                        │
│ • IAM, RBAC                                               │
│ • API quotas, billing                                      │
│ • Audit logging                                            │
└───────────────────────┬────────────────────────────────────┘
                        ▼
┌────────────────────────────────────────────────────────────┐
│ Services: Traditional Only                                 │
│ • Auth, Projects, Documents                                │
│ • Webhooks, Analytics, Notifications                       │
└───────────────────────┬────────────────────────────────────┘
                        ▼
┌────────────────────────────────────────────────────────────┐
│ Data: Traditional Data Platform                            │
│ • PostgreSQL (OLTP)                                        │
│ • ClickHouse/StarRocks (OLAP)                             │
│ • Redis (Cache)                                            │
└────────────────────────────────────────────────────────────┘
```

**Excluded**:
- ❌ MCP Gateway
- ❌ LLM Gateway
- ❌ Agent Orchestration
- ❌ RAG Service
- ❌ Qdrant (Vector DB)
- ❌ Neo4j (Knowledge Graph)
- ❌ Semantic Cache

---

## Deployment Configuration

### Docker Compose

```yaml
# docker-compose-traditional.yml

version: '3.8'

services:
  # Gateway
  api-gateway:
    image: cortex-ai/api-gateway:latest
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://postgres:5432/cortex
      - REDIS_URL=redis://redis:6379
      - JWT_SECRET=${JWT_SECRET}
      - ENABLE_AI_FEATURES=false  # Disable AI
    depends_on:
      - postgres
      - redis

  # Traditional Services
  auth-service:
    image: cortex-ai/auth-service:latest
    environment:
      - DATABASE_URL=postgresql://postgres:5432/cortex
      - JWT_SECRET=${JWT_SECRET}

  project-service:
    image: cortex-ai/project-service:latest
    environment:
      - DATABASE_URL=postgresql://postgres:5432/cortex

  document-service:
    image: cortex-ai/document-service:latest
    environment:
      - DATABASE_URL=postgresql://postgres:5432/cortex
      - S3_BUCKET=${S3_BUCKET}
      - AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
      - AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}

  analytics-service:
    image: cortex-ai/analytics-service:latest
    environment:
      - CLICKHOUSE_URL=clickhouse://clickhouse:8123

  # Data Platform (Traditional)
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

volumes:
  postgres_data:
  clickhouse_data:
  redis_data:
```

### Kubernetes

```yaml
# k8s-traditional-only.yaml

apiVersion: v1
kind: Namespace
metadata:
  name: cortex-traditional

---
# API Gateway
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
  namespace: cortex-traditional
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-gateway
  template:
    metadata:
      labels:
        app: api-gateway
    spec:
      containers:
      - name: api-gateway
        image: cortex-ai/api-gateway:latest
        ports:
        - containerPort: 8000
        env:
        - name: ENABLE_AI_FEATURES
          value: "false"
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secrets
              key: url
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"

---
# PostgreSQL
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: cortex-traditional
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15-alpine
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_DB
          value: cortex
        - name: POSTGRES_USER
          value: cortex
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secrets
              key: password
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 100Gi

---
# ClickHouse
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: clickhouse
  namespace: cortex-traditional
spec:
  serviceName: clickhouse
  replicas: 1
  selector:
    matchLabels:
      app: clickhouse
  template:
    metadata:
      labels:
        app: clickhouse
    spec:
      containers:
      - name: clickhouse
        image: clickhouse/clickhouse-server:latest
        ports:
        - containerPort: 8123
        - containerPort: 9000
        volumeMounts:
        - name: clickhouse-storage
          mountPath: /var/lib/clickhouse
  volumeClaimTemplates:
  - metadata:
      name: clickhouse-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 500Gi

---
# Redis
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: cortex-traditional
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
```

---

## Cost Estimation

### Infrastructure Costs (AWS)

| Resource | Size | Monthly Cost |
|----------|------|--------------|
| **Compute** | 3x t3.medium (API Gateway) | $75 |
| **Database** | db.r5.large (PostgreSQL) | $200 |
| **Analytics** | r5.2xlarge (ClickHouse) | $400 |
| **Cache** | cache.r5.large (Redis) | $150 |
| **Storage** | 500GB EBS | $50 |
| **Load Balancer** | ALB | $25 |
| **Total** | | **~$900/month** |

### Without AI Components Saved

| Component | Savings |
|-----------|---------|
| Qdrant cluster | -$300/month |
| Neo4j cluster | -$400/month |
| LLM costs (avoided) | -$0-1000+/month |
| **Total Savings** | **~$700-1700/month** |

---

## Feature Availability

| Feature | Available |
|---------|-----------|
| **Traditional Features** | |
| User authentication | ✅ |
| Project management | ✅ |
| Document upload/storage | ✅ |
| REST API | ✅ |
| Webhooks | ✅ |
| Analytics dashboards | ✅ |
| Audit logging | ✅ |
| Multi-tenancy | ✅ |
| RBAC | ✅ |
| **AI Features** | |
| Chat/LLM completion | ❌ |
| Semantic search | ❌ |
| Knowledge graph | ❌ |
| Agent orchestration | ❌ |
| RAG | ❌ |

---

## Migration Path to Full Platform

When ready to add AI features:

```yaml
# 1. Add AI data platform
services:
  qdrant:
    image: qdrant/qdrant:latest
    ports:
      - "6333:6333"

  neo4j:
    image: neo4j:5-community
    ports:
      - "7474:7474"
      - "7687:7687"

# 2. Add MCP Gateway
  mcp-gateway:
    image: cortex-ai/mcp-gateway:latest
    environment:
      - ENABLE_AI_FEATURES=true

# 3. Add AI services
  llm-gateway:
    image: cortex-ai/llm-gateway:latest

  rag-service:
    image: cortex-ai/rag-service:latest

# 4. Enable AI in API Gateway
  api-gateway:
    environment:
      - ENABLE_AI_FEATURES=true  # Change to true
```

---

**Next**: [Deploy AI-Only](./05b-deploy-ai-only.md) | [Deploy Full Platform](./05c-deploy-full-platform.md) | [Back to Overview](../00-parallel-stacks-overview.md)
