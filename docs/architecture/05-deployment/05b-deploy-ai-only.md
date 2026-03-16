# Deploy AI-Only Stack

## Overview

Deploy the platform **with only AI features** for pure AI agent use cases like Replit Agent, Cursor AI, or agent-first applications. Minimal traditional SaaS overhead.

**Use Cases**:
- Pure AI agent platforms
- Research/prototyping AI capabilities
- Agent-first applications (no CRUD)
- AI-native products

---

## Architecture

```
┌────────────────────────────────────────────────────────────┐
│ Gateway: MCP Gateway Only                                  │
│ (Minimal API Gateway for auth)                             │
└───────────────────────┬────────────────────────────────────┘
                        ▼
┌────────────────────────────────────────────────────────────┐
│ Control Plane: AI + Shared (Minimal Traditional)           │
│ • LLM quotas, cost tracking                                │
│ • Tool permissions                                          │
│ • Minimal auth (JWT only)                                  │
└───────────────────────┬────────────────────────────────────┘
                        ▼
┌────────────────────────────────────────────────────────────┐
│ Services: AI Only                                          │
│ • LLM Gateway, Agent Orchestration                         │
│ • RAG, Memory Management                                   │
│ • Tool Registry                                            │
└───────────────────────┬────────────────────────────────────┘
                        ▼
┌────────────────────────────────────────────────────────────┐
│ Data: AI Data Platform                                     │
│ • PostgreSQL (minimal: auth/users only)                    │
│ • Qdrant (Vector DB)                                       │
│ • Neo4j (Knowledge Graph)                                  │
│ • Redis (Cache + Memory)                                   │
└────────────────────────────────────────────────────────────┘
```

**Included**:
- ✅ MCP Gateway
- ✅ LLM Gateway
- ✅ Agent Orchestration
- ✅ RAG Service
- ✅ Qdrant (Vector DB)
- ✅ Neo4j (Knowledge Graph)
- ✅ Semantic Cache

**Excluded**:
- ❌ Full API Gateway (only minimal auth endpoint)
- ❌ Project Management
- ❌ Document Service (or minimal version)
- ❌ Webhooks
- ❌ Traditional Analytics
- ❌ ClickHouse (no traditional analytics)

---

## Deployment Configuration

### Docker Compose

```yaml
# docker-compose-ai-only.yml

version: '3.8'

services:
  # Minimal API Gateway (auth only)
  api-gateway-minimal:
    image: cortex-ai/api-gateway-minimal:latest
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://postgres:5432/cortex
      - REDIS_URL=redis://redis:6379
      - JWT_SECRET=${JWT_SECRET}
      - MODE=ai_only  # Disable traditional features
    depends_on:
      - postgres
      - redis

  # MCP Gateway (AI)
  mcp-gateway:
    image: cortex-ai/mcp-gateway:latest
    ports:
      - "8001:8001"
    environment:
      - JWT_SECRET=${JWT_SECRET}
      - MCP_SERVER_URL=http://mcp-server:8002
    depends_on:
      - mcp-server

  # MCP Server (Tool Registry)
  mcp-server:
    image: cortex-ai/mcp-server:latest
    ports:
      - "8002:8002"
    environment:
      - QDRANT_URL=http://qdrant:6333
      - NEO4J_URL=bolt://neo4j:7687
      - DATABASE_URL=postgresql://postgres:5432/cortex

  # AI Services
  llm-gateway:
    image: cortex-ai/llm-gateway:latest
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
      - REDIS_URL=redis://redis:6379  # Semantic cache
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

  # AI Data Platform
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

  # Minimal PostgreSQL (auth/users only)
  postgres:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=cortex
      - POSTGRES_USER=cortex
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data

  # Redis (cache + memory)
  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data

volumes:
  qdrant_storage:
  neo4j_data:
  postgres_data:
  redis_data:
```

### Kubernetes (AI-Only)

```yaml
# k8s-ai-only.yaml

apiVersion: v1
kind: Namespace
metadata:
  name: cortex-ai

---
# MCP Gateway
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mcp-gateway
  namespace: cortex-ai
spec:
  replicas: 3
  selector:
    matchLabels:
      app: mcp-gateway
  template:
    metadata:
      labels:
        app: mcp-gateway
    spec:
      containers:
      - name: mcp-gateway
        image: cortex-ai/mcp-gateway:latest
        ports:
        - containerPort: 8001
        env:
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: auth-secrets
              key: jwt-secret
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"

---
# LLM Gateway
apiVersion: apps/v1
kind: Deployment
metadata:
  name: llm-gateway
  namespace: cortex-ai
spec:
  replicas: 3
  selector:
    matchLabels:
      app: llm-gateway
  template:
    metadata:
      labels:
        app: llm-gateway
    spec:
      containers:
      - name: llm-gateway
        image: cortex-ai/llm-gateway:latest
        env:
        - name: OPENAI_API_KEY
          valueFrom:
            secretKeyRef:
              name: llm-secrets
              key: openai-key
        resources:
          requests:
            memory: "1Gi"
            cpu: "1000m"

---
# Qdrant (Vector DB)
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: qdrant
  namespace: cortex-ai
spec:
  serviceName: qdrant
  replicas: 3  # Sharding for scalability
  selector:
    matchLabels:
      app: qdrant
  template:
    metadata:
      labels:
        app: qdrant
    spec:
      containers:
      - name: qdrant
        image: qdrant/qdrant:latest
        ports:
        - containerPort: 6333
        volumeMounts:
        - name: qdrant-storage
          mountPath: /qdrant/storage
        resources:
          requests:
            memory: "8Gi"
            cpu: "4000m"
          limits:
            memory: "16Gi"
            cpu: "8000m"
  volumeClaimTemplates:
  - metadata:
      name: qdrant-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 500Gi

---
# Neo4j (Knowledge Graph)
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: neo4j
  namespace: cortex-ai
spec:
  serviceName: neo4j
  replicas: 1
  selector:
    matchLabels:
      app: neo4j
  template:
    metadata:
      labels:
        app: neo4j
    spec:
      containers:
      - name: neo4j
        image: neo4j:5-community
        ports:
        - containerPort: 7474
        - containerPort: 7687
        env:
        - name: NEO4J_AUTH
          valueFrom:
            secretKeyRef:
              name: neo4j-secrets
              key: auth
        volumeMounts:
        - name: neo4j-data
          mountPath: /data
        resources:
          requests:
            memory: "4Gi"
            cpu: "2000m"
  volumeClaimTemplates:
  - metadata:
      name: neo4j-data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 200Gi
```

---

## Cost Estimation

### Infrastructure Costs (AWS)

| Resource | Size | Monthly Cost |
|----------|------|--------------|
| **Compute** | |
| MCP Gateway (3x t3.medium) | $75 |
| LLM Gateway (3x t3.large) | $150 |
| Agent Orchestrator (3x t3.medium) | $75 |
| **AI Data Platform** | |
| Qdrant (3x r5.2xlarge) | $1,200 |
| Neo4j (r5.xlarge) | $300 |
| Redis (cache.r5.large) | $150 |
| **Storage** | |
| Qdrant storage (1TB EBS) | $100 |
| Neo4j storage (200GB EBS) | $20 |
| **LLM Costs** | |
| OpenAI/Anthropic usage | $500-5000+ |
| **Total** | | **~$2,570-7,070/month** |

### Savings vs Full Platform

| Component | Savings |
|-----------|---------|
| No ClickHouse | -$400/month |
| No full API services | -$200/month |
| Minimal PostgreSQL | -$100/month |
| **Total Savings** | **~$700/month** |

---

## Feature Availability

| Feature | Available |
|---------|-----------|
| **AI Features** | |
| Chat/LLM completion | ✅ |
| Semantic search | ✅ |
| Knowledge graph | ✅ |
| Agent orchestration | ✅ |
| RAG (Graph RAG) | ✅ |
| Tool execution (MCP) | ✅ |
| Memory management | ✅ |
| Semantic caching | ✅ |
| **Traditional Features** | |
| User authentication | ✅ (minimal) |
| Project management | ❌ |
| Document management | ⚠️ (minimal, for RAG) |
| Webhooks | ❌ |
| Analytics dashboards | ❌ |

---

## Minimal API Endpoints

Only essential endpoints for auth:

```python
# Minimal API Gateway for AI-only deployment
@app.post("/api/v1/auth/login")
async def login(email: str, password: str):
    """Minimal login endpoint."""
    ...

@app.post("/api/v1/auth/register")
async def register(email: str, password: str):
    """Minimal registration."""
    ...

@app.get("/api/v1/auth/me")
async def get_current_user():
    """Get current user (for token validation)."""
    ...

# No project/document/webhook endpoints in AI-only mode
```

---

## Migration Path to Full Platform

When ready to add traditional features:

```yaml
# 1. Add ClickHouse for analytics
services:
  clickhouse:
    image: clickhouse/clickhouse-server:latest

# 2. Enable full API Gateway
  api-gateway:
    image: cortex-ai/api-gateway:latest  # Full version
    environment:
      - MODE=full  # Enable all features

# 3. Add traditional services
  project-service:
    image: cortex-ai/project-service:latest

  document-service:
    image: cortex-ai/document-service:latest

  analytics-service:
    image: cortex-ai/analytics-service:latest

# 4. Upgrade PostgreSQL
  postgres:
    # Increase resources for full schema
    resources:
      requests:
        memory: "4Gi"
```

---

**Next**: [Deploy Traditional-Only](./05a-deploy-traditional-only.md) | [Deploy Full Platform](./05c-deploy-full-platform.md) | [Back to Overview](../00-parallel-stacks-overview.md)
