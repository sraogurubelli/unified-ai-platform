# SaaS Deployment Model

## Overview

Pure SaaS deployment is a fully managed, multi-tenant architecture where all infrastructure is hosted and operated by the platform provider. Tenants share resources with logical isolation.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│              SaaS Provider Infrastructure (Cloud)            │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Load Balancer / CDN                      │  │
│  │          (CloudFlare, AWS CloudFront)                 │  │
│  └────────────────────┬─────────────────────────────────┘  │
│                       ▼                                     │
│  ┌────────────────────────────────────────────────────┐    │
│  │          Gateway Layer (Dual Gateway)              │    │
│  ├──────────────────────┬─────────────────────────────┤    │
│  │   API Gateway        │    MCP Gateway              │    │
│  │   (FastAPI/Kong)     │    (MCP Servers)            │    │
│  └──────────────────────┴─────────────────────────────┘    │
│                       ▼                                     │
│  ┌─────────────────────────────────────────────────────┐   │
│  │         Control Plane (Shared - Multi-tenant)       │   │
│  │  - IAM / RBAC                                       │   │
│  │  - Billing & Usage Tracking                         │   │
│  │  - Observability                                    │   │
│  │  - Policy Enforcement                               │   │
│  └─────────────────────────────────────────────────────┘   │
│                       ▼                                     │
│  ┌─────────────────────────────────────────────────────┐   │
│  │         Service Layer (Shared instances)            │   │
│  │  - Platform Services (Auth, Orgs, Projects)         │   │
│  │  - AI Services (Agents, RAG, Knowledge Graphs)      │   │
│  └─────────────────────────────────────────────────────┘   │
│                       ▼                                     │
│  ┌─────────────────────────────────────────────────────┐   │
│  │      Data Plane (Logical Multi-tenancy)             │   │
│  │                                                      │   │
│  │  ┌──────────┐  ┌───────────┐  ┌─────────────────┐  │   │
│  │  │PostgreSQL│  │ClickHouse │  │  Redis          │  │   │
│  │  │(RDS)     │  │(Analytics)│  │  (ElastiCache)  │  │   │
│  │  │tenant_id │  │tenant_id  │  │  tenant:* keys  │  │   │
│  │  └──────────┘  └───────────┘  └─────────────────┘  │   │
│  │                                                      │   │
│  │  ┌──────────┐  ┌───────────┐                        │   │
│  │  │ Qdrant   │  │  Neo4j    │                        │   │
│  │  │(Managed) │  │  (Aura)   │                        │   │
│  │  │Collections│  │ Namespace │                        │   │
│  │  │per tenant │  │ per tenant│                        │   │
│  │  └──────────┘  └───────────┘                        │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

## Key Characteristics

### 1. Multi-tenant Resource Sharing

All tenants share:
- Compute instances (API servers, workers)
- Database clusters (logical separation)
- Storage systems (namespace/collection isolation)
- Network infrastructure

### 2. Logical Tenant Isolation

**Database Level** (PostgreSQL):
```sql
-- All tables have tenant_id
CREATE TABLE conversations (
    id SERIAL PRIMARY KEY,
    tenant_id VARCHAR(255) NOT NULL,
    project_id INTEGER NOT NULL,
    thread_id VARCHAR(255) NOT NULL,
    -- other fields
    FOREIGN KEY (tenant_id) REFERENCES tenants(id)
);

-- Row-level security
CREATE POLICY tenant_isolation ON conversations
    USING (tenant_id = current_setting('app.current_tenant')::VARCHAR);

-- All queries filtered by tenant
SELECT * FROM conversations WHERE tenant_id = 'acct_abc123';
```

**Vector Store** (Qdrant):
```python
# Collection per tenant or metadata filtering
collection_name = f"embeddings_{tenant_id}"

# Or single collection with tenant metadata
qdrant.upsert(
    collection_name="embeddings",
    points=[
        PointStruct(
            id=uuid.uuid4(),
            vector=embedding,
            payload={
                "tenant_id": tenant_id,
                "document_id": doc_id,
                "content": text
            }
        )
    ]
)

# Query with tenant filter
results = qdrant.search(
    collection_name="embeddings",
    query_vector=query_embedding,
    query_filter=Filter(
        must=[
            FieldCondition(
                key="tenant_id",
                match=MatchValue(value=tenant_id)
            )
        ]
    )
)
```

**Cache/Queue** (Redis):
```python
# Prefix all keys with tenant ID
session_key = f"tenant:{tenant_id}:session:{session_id}"
queue_key = f"tenant:{tenant_id}:queue:agent_tasks"

# Namespace isolation
redis.set(session_key, session_data)
redis.lpush(queue_key, task_data)
```

### 3. Cost Efficiency

**Economies of Scale**:
- Shared infrastructure reduces per-tenant costs
- Centralized operations and maintenance
- Bulk purchasing of cloud resources

**Resource Optimization**:
- Dynamic scaling based on aggregate load
- Load balancing across all tenants
- Shared caching layers

## Infrastructure Setup

### Cloud Provider: AWS (Example)

```yaml
# infrastructure.yaml (Terraform/Pulumi)

# Compute (ECS/EKS)
api_cluster:
  type: aws_ecs_cluster
  capacity_providers: [FARGATE, FARGATE_SPOT]
  services:
    - api_gateway (FastAPI)
    - agent_orchestrator
    - rag_service

# Databases
rds_postgres:
  engine: postgres15
  instance_class: db.r6g.xlarge
  multi_az: true
  storage_encrypted: true
  backup_retention: 7

# Analytics
clickhouse:
  type: aws_ec2  # Or managed ClickHouse Cloud
  instance_type: r6g.2xlarge
  ebs_optimized: true

# Vector Search
qdrant:
  type: qdrant_cloud  # Or self-hosted on EC2
  cluster_size: 3
  memory: 32GB

# Graph Database
neo4j:
  type: neo4j_aura  # Managed Neo4j
  tier: professional
  memory: 16GB

# Cache/Queue
elasticache_redis:
  engine: redis7
  node_type: cache.r6g.large
  num_cache_nodes: 3
  automatic_failover: true

# Object Storage
s3:
  buckets:
    - user_uploads
    - document_storage
    - backup_snapshots
  encryption: AES256
  versioning: enabled
```

### Kubernetes Deployment (Example)

```yaml
# k8s/api-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
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
      - name: api
        image: cortex-ai:latest
        resources:
          requests:
            cpu: "1000m"
            memory: "2Gi"
          limits:
            cpu: "2000m"
            memory: "4Gi"
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: url
        - name: REDIS_URL
          valueFrom:
            secretKeyRef:
              name: redis-credentials
              key: url
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5

---
apiVersion: v1
kind: Service
metadata:
  name: api-gateway
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8000
  selector:
    app: api-gateway

---
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
```

## Tenant Onboarding Flow

### 1. Sign-Up

```python
# POST /api/v1/auth/signup
{
    "email": "admin@acme.com",
    "organization_name": "Acme Corp",
    "subscription_tier": "pro"
}

# Backend creates:
# 1. Account (tenant) with unique tenant_id
# 2. Organization under account
# 3. Principal (user) as owner
# 4. Default project
# 5. Initial RBAC memberships
```

### 2. Provision Resources

```python
async def provision_tenant(account: Account):
    """Provision tenant resources on signup."""

    # 1. Database: Already exists (logical isolation)
    # No action needed - all tables have tenant_id

    # 2. Vector Store: Create collection or namespace
    await qdrant_client.create_collection(
        collection_name=f"embeddings_{account.uid}",
        vectors_config=VectorParams(size=1536, distance="Cosine")
    )

    # 3. Graph Database: Create namespace/label
    await neo4j_driver.execute_query(
        "CREATE CONSTRAINT tenant_isolation IF NOT EXISTS "
        "FOR (n:Entity) REQUIRE n.tenant_id IS NOT NULL"
    )

    # 4. Cache: No pre-provisioning (keys created on-demand)

    # 5. Usage tracking: Initialize quotas
    await usage_tracker.initialize_tenant(
        tenant_id=account.uid,
        tier=account.subscription_tier,
        quotas={
            "api_calls_per_month": 50000,
            "llm_tokens_per_month": 1000000,
            "storage_gb": 10
        }
    )
```

### 3. Access Control

```python
# Every request authenticated and tenant-scoped
@app.middleware("http")
async def tenant_context_middleware(request: Request, call_next):
    """Extract tenant context from JWT."""
    token = request.headers.get("Authorization", "").replace("Bearer ", "")
    payload = jwt.decode(token, settings.JWT_SECRET)

    # Set tenant context for the request
    request.state.tenant_id = payload["tenant_id"]
    request.state.principal_id = payload["principal_id"]

    response = await call_next(request)
    return response

# All database queries filtered by tenant
async def get_conversations(request: Request):
    tenant_id = request.state.tenant_id

    conversations = await db.query(Conversation).filter_by(
        tenant_id=tenant_id
    ).all()

    return conversations
```

## Scaling Strategy

### Horizontal Scaling

**API Layer**:
- Stateless API servers
- Auto-scale based on CPU/memory/request rate
- Load balanced across availability zones

**Worker Layer**:
- Agent execution workers
- Task queue consumers
- Scale based on queue depth

### Database Scaling

**PostgreSQL**:
- Read replicas for read-heavy workloads
- Connection pooling (PgBouncer)
- Partitioning by tenant_id for large tables

**ClickHouse**:
- Distributed tables for analytics
- Replication for high availability

**Qdrant**:
- Horizontal scaling with sharding
- Replication for durability

**Neo4j**:
- Clustering for read scaling
- Causal cluster for write availability

## Monitoring & Observability

### Metrics (Prometheus)

```python
# Tenant-scoped metrics
api_requests_total = Counter(
    'api_requests_total',
    'Total API requests',
    ['tenant_id', 'endpoint', 'status']
)

llm_tokens_used = Counter(
    'llm_tokens_used',
    'LLM tokens consumed',
    ['tenant_id', 'model', 'provider']
)

vector_search_latency = Histogram(
    'vector_search_latency_seconds',
    'Vector search latency',
    ['tenant_id', 'collection']
)
```

### Logging (Structured JSON)

```python
logger.info(
    "conversation_created",
    tenant_id=tenant_id,
    conversation_id=conversation.uid,
    project_id=project.uid,
    user_id=principal.uid
)
```

### Tracing (OpenTelemetry)

```python
from opentelemetry import trace

tracer = trace.get_tracer(__name__)

async def chat_endpoint(request: ChatRequest):
    with tracer.start_as_current_span("chat_endpoint") as span:
        span.set_attribute("tenant_id", tenant_id)
        span.set_attribute("conversation_id", conversation_id)

        # Nested spans for sub-operations
        with tracer.start_as_current_span("load_conversation"):
            conversation = await load_conversation(...)

        with tracer.start_as_current_span("agent_execution"):
            result = await agent.run(...)

        return result
```

## Security Considerations

### 1. Tenant Isolation Enforcement

```python
# Database queries MUST filter by tenant_id
# BAD: Missing tenant filter
conversations = db.query(Conversation).filter_by(project_id=project_id).all()

# GOOD: Always include tenant_id
conversations = db.query(Conversation).filter_by(
    tenant_id=tenant_id,
    project_id=project_id
).all()

# Use database-level row security where possible
ALTER TABLE conversations ENABLE ROW LEVEL SECURITY;
```

### 2. Rate Limiting

```python
# Per-tenant rate limiting
@app.middleware("http")
async def rate_limit_middleware(request: Request, call_next):
    tenant_id = request.state.tenant_id

    # Check rate limit
    is_allowed = await rate_limiter.check_limit(
        key=f"tenant:{tenant_id}:api_calls",
        limit=1000,  # calls per minute
        window=60
    )

    if not is_allowed:
        raise HTTPException(status_code=429, detail="Rate limit exceeded")

    return await call_next(request)
```

### 3. Data Encryption

- **In Transit**: TLS 1.3 for all connections
- **At Rest**: AES-256 encryption for databases and storage
- **Secrets**: AWS Secrets Manager / HashiCorp Vault

### 4. Audit Logging

```python
# Log all sensitive operations
audit_logger.log(
    event="conversation_deleted",
    tenant_id=tenant_id,
    principal_id=principal_id,
    resource_type="conversation",
    resource_id=conversation_id,
    timestamp=datetime.utcnow()
)
```

## Cost Management

### Usage Tracking

```python
# Track all billable events
await billing.record_usage(
    tenant_id=tenant_id,
    usage_type="llm_tokens",
    quantity=response.usage.total_tokens,
    unit_cost=0.00001,  # per token
    metadata={
        "model": "gpt-4o",
        "conversation_id": conversation_id
    }
)
```

### Quota Enforcement

```python
# Check quota before expensive operations
async def create_embedding(tenant_id: str, text: str):
    quota = await billing.get_quota(tenant_id, "llm_tokens")
    usage = await billing.get_usage(tenant_id, "llm_tokens", period="month")

    if usage >= quota:
        raise HTTPException(
            status_code=402,  # Payment Required
            detail="Monthly token quota exceeded"
        )

    # Proceed with operation
    embedding = await openai.embeddings.create(...)

    # Record usage
    await billing.record_usage(...)
```

## Disaster Recovery

### Backup Strategy

- **PostgreSQL**: Daily snapshots, 30-day retention, point-in-time recovery
- **ClickHouse**: Weekly backups to S3
- **Qdrant**: Snapshot API, backup to object storage
- **Neo4j**: Automated backups via Neo4j Aura
- **Redis**: AOF + RDB persistence, replicas in multiple AZs

### Recovery Time Objectives (RTO)

- **Critical Path (API, Auth)**: RTO < 15 minutes
- **Data Plane (Databases)**: RTO < 1 hour
- **Analytics**: RTO < 4 hours

## Best Practices

1. **Always filter by tenant_id** in database queries
2. **Use connection pooling** for database efficiency
3. **Implement circuit breakers** for external LLM APIs
4. **Cache aggressively** (with tenant-scoped keys)
5. **Monitor per-tenant costs** for pricing optimization
6. **Set quotas** for all tenants to prevent runaway costs
7. **Use feature flags** for gradual rollouts
8. **Encrypt sensitive data** (API keys, PII)
9. **Log everything** with structured logging
10. **Test tenant isolation** with automated tests

## Migration from On-Premise

If migrating a tenant from on-premise to SaaS:

1. **Data Export**: Export tenant data (databases, vectors, graphs)
2. **Import to SaaS**: Create tenant, import data with tenant_id tagging
3. **Validation**: Verify data integrity and access controls
4. **Cutover**: Update DNS/routing to SaaS endpoints
5. **Monitoring**: Monitor performance and errors post-migration

---

**Next**: [Hybrid Deployment](./hybrid-deployment.md) | [On-Premise Deployment](./onprem-deployment.md)
