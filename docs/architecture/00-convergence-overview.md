# Convergence Architecture Overview

## The Marriage of SaaS and AI Architectures

Modern AI platforms exist at the intersection of two architectural paradigms:

1. **Traditional SaaS Architecture**: Multi-tenancy, RBAC, billing, observability
2. **AI-Native Patterns**: Agent orchestration, RAG pipelines, vector search, knowledge graphs

The convergence architecture recognizes that AI platforms must excel in **both worlds simultaneously**.

## Core Principle: Parallel Infrastructure Patterns

### Gateway Layer - Dual Gateways

| SaaS World | AI World | Unified Function |
|------------|----------|------------------|
| API Gateway | MCP Gateway | Request routing, auth, rate limiting |
| REST/GraphQL APIs | Tools/Agents | Service interface |
| OpenAPI specs | MCP server definitions | Interface contracts |

### Data Plane - Hybrid Storage Strategy

| SaaS World | AI World | Purpose |
|------------|----------|---------|
| OLTP (PostgreSQL) | Vector Store (Qdrant) | Transactional data vs Embeddings |
| OLAP (ClickHouse) | Graph Database (Neo4j) | Analytics vs Knowledge relationships |
| Redis Cache | Redis Queue | Fast access vs Agent task coordination |

### Control Plane - Unified Management

Both SaaS and AI services share:
- **IAM/RBAC**: Users, roles, permissions apply to both APIs and agents
- **Multi-tenancy**: Tenant isolation for both traditional data and AI artifacts
- **Observability**: Unified metrics, logs, traces across all services
- **Billing**: Unified usage tracking for API calls, LLM tokens, storage

## Architecture Layers

```
┌─────────────────────────────────────────────────────────────┐
│                    Gateway Layer (Dual)                      │
├──────────────────────────┬──────────────────────────────────┤
│    API Gateway           │       MCP Gateway                │
│  (HTTP/REST/GraphQL)     │    (Tools/Agent Protocol)        │
└──────────────────────────┴──────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   Control Plane (Unified)                    │
├─────────────┬─────────────┬─────────────┬──────────────────┤
│ IAM / RBAC  │ Multi-tenant│ Observability│ Billing/Usage   │
└─────────────┴─────────────┴─────────────┴──────────────────┘
                              ▼
┌──────────────────────────────────────────────────────────────┐
│              Service Layer (Platform + AI Convergence)        │
├──────────────────────────┬───────────────────────────────────┤
│   Platform Services      │      AI Services                  │
│   - Auth                 │      - Agent Orchestration        │
│   - Organizations        │      - RAG Pipelines              │
│   - Projects             │      - Knowledge Graphs           │
│   - Webhooks             │      - Model Management           │
│   - Analytics            │      - Fine-tuning                │
└──────────────────────────┴───────────────────────────────────┘
                              ▼
┌──────────────────────────────────────────────────────────────┐
│                Data Plane (Hybrid Storage)                    │
├────────────┬────────────┬─────────────┬─────────────────────┤
│ OLTP       │ OLAP       │ Vector      │ Graph               │
│ PostgreSQL │ ClickHouse │ Qdrant      │ Neo4j               │
│ (Users,    │ (Usage,    │ (Embeddings,│ (Knowledge,         │
│  Projects) │  Analytics)│  Semantic)  │  Relationships)     │
└────────────┴────────────┴─────────────┴─────────────────────┘
                              ▼
┌──────────────────────────────────────────────────────────────┐
│              Cache & Queue Layer (Redis/Kafka)                │
│         - Session cache                                       │
│         - Task queues (agent execution)                       │
│         - Pub/Sub (real-time events)                          │
└──────────────────────────────────────────────────────────────┘
```

## Key Architectural Decisions

### 1. Gateway Layer: Dual Gateway Pattern

**Why two gateways?**
- **API Gateway**: Handles traditional HTTP/REST/GraphQL traffic
  - User authentication
  - Rate limiting by tenant
  - Request/response transformation
  - Traditional CRUD operations

- **MCP Gateway**: Handles Model Context Protocol for agents/tools
  - Tool discovery and registration
  - Agent-to-tool communication
  - Context management
  - Tool permission enforcement

**Convergence Point**: Both gateways enforce the same IAM/RBAC policies and route to the same control plane for auth decisions.

### 2. Control Plane: Unified Management

All services (SaaS and AI) share a single control plane:

```python
# Example: Same RBAC check for API and Agent
async def check_permission(
    principal: Principal,
    resource_type: str,  # "project", "document", "agent"
    resource_id: str,
    permission: Permission  # CREATE, READ, UPDATE, DELETE
) -> bool:
    # Works for both:
    # - API: "Can user create a project?"
    # - Agent: "Can agent access this document?"
    return await rbac_engine.check(...)
```

**Benefits**:
- Consistent security model
- Single source of truth for tenancy
- Unified audit logs
- Simplified compliance

### 3. Service Layer: Platform + AI Convergence

Services are organized by domain, not by "traditional" vs "AI":

**Organization Service** (Platform):
- CRUD for organizations
- Member management
- Settings

**Project Service** (Platform + AI):
- Project metadata (SaaS)
- Project-scoped agents (AI)
- Project-scoped knowledge bases (AI)

**Agent Orchestration Service** (AI):
- Agent lifecycle management
- Multi-agent coordination
- Tool registry
- Session persistence

**Convergence Pattern**: Services can be hybrid. A "Project" contains both traditional resources (members, settings) and AI resources (agents, knowledge bases).

### 4. Data Plane: Hybrid Storage Strategy

Different data types require different storage engines:

| Data Type | Storage Engine | Reason |
|-----------|---------------|--------|
| Users, Accounts, Organizations | PostgreSQL (OLTP) | ACID transactions, relational integrity |
| Conversation history | PostgreSQL (OLTP) | Transactional consistency, FK relationships |
| Usage metrics, analytics | ClickHouse (OLAP) | Column-oriented, fast aggregations |
| Document embeddings | Qdrant (Vector) | Similarity search, ANN algorithms |
| Knowledge graphs | Neo4j (Graph) | Relationship traversal, graph algorithms |
| Session state, cache | Redis | Low latency, pub/sub |
| Agent task queues | Redis Streams | Ordered messages, consumer groups |

**Tenant Isolation Strategy**:
- **OLTP**: Row-level security with `tenant_id` columns
- **OLAP**: Partition by `tenant_id`, query-time filtering
- **Vector**: Collections per tenant or metadata filtering
- **Graph**: Namespace nodes by tenant, enforce at query time
- **Cache**: Prefix keys with `tenant:{id}:`

## Deployment Models

The convergence architecture supports three deployment models:

### 1. Pure SaaS (Multi-tenant Cloud)

```
┌─────────────────────────────────────────────┐
│         SaaS Provider Infrastructure         │
│                                             │
│  ┌─────────────┐  ┌──────────────────────┐ │
│  │ All tenants │  │ Shared Control Plane │ │
│  │ share same  │  │ (IAM, RBAC, Billing) │ │
│  │ resources   │  └──────────────────────┘ │
│  └─────────────┘                           │
│                                             │
│  Data Plane (Logical isolation):            │
│  ┌──────────┬───────────┬────────────────┐ │
│  │PostgreSQL│ Qdrant    │ Neo4j          │ │
│  │(multi-   │(multi-    │(multi-tenant)  │ │
│  │ tenant)  │ tenant)   │                │ │
│  └──────────┴───────────┴────────────────┘ │
└─────────────────────────────────────────────┘
```

**Characteristics**:
- Shared infrastructure, logical isolation via `tenant_id`
- Cost-efficient for provider
- Best for: SMBs, startups, developers

### 2. Hybrid (Control Plane SaaS, Data On-Premise)

```
┌─────────────────────────┐      ┌──────────────────────────┐
│   SaaS Provider (Cloud) │      │ Customer Environment     │
│                         │      │ (On-Premise/VPC)         │
│  Control Plane:         │      │                          │
│  - IAM/RBAC             │◄────►│  Data Plane:             │
│  - Billing              │ API  │  - PostgreSQL            │
│  - Observability        │      │  - Qdrant                │
│  - Policy enforcement   │      │  - Neo4j                 │
│                         │      │  - Redis                 │
│  Agent Orchestration    │      │                          │
│  (stateless)            │      │  Customer Data           │
└─────────────────────────┘      │  (never leaves premises) │
                                 └──────────────────────────┘
```

**Characteristics**:
- Control plane managed by provider
- Data plane runs in customer environment
- Best for: Regulated industries, data residency requirements (finance, healthcare)

### 3. Pure On-Premise (Air-gapped)

```
┌──────────────────────────────────────────────┐
│     Customer Environment (Air-gapped)        │
│                                              │
│  Full Stack:                                 │
│  ┌────────────────────────────────────────┐ │
│  │ Control Plane (customer-managed)       │ │
│  │ - IAM/RBAC                             │ │
│  │ - Observability                        │ │
│  │ - Billing (optional)                   │ │
│  └────────────────────────────────────────┘ │
│                                              │
│  ┌────────────────────────────────────────┐ │
│  │ Service Layer (all services)           │ │
│  └────────────────────────────────────────┘ │
│                                              │
│  ┌────────────────────────────────────────┐ │
│  │ Data Plane (customer infrastructure)   │ │
│  └────────────────────────────────────────┘ │
│                                              │
│  No external connectivity required          │
└──────────────────────────────────────────────┘
```

**Characteristics**:
- Complete platform deployed in customer environment
- Full data sovereignty
- Best for: Government, defense, high-security enterprises

## Design Principles

### 1. Multi-tenancy from Day One

Every resource must have a tenant context:

```python
# Bad: No tenant awareness
def get_conversations(user_id: str):
    return db.query(Conversation).filter_by(user_id=user_id).all()

# Good: Tenant-scoped
def get_conversations(tenant_id: str, user_id: str):
    return db.query(Conversation).filter_by(
        tenant_id=tenant_id,
        user_id=user_id
    ).all()
```

### 2. Unified Observability

Single observability stack for all services:

- **Metrics**: Prometheus/OpenTelemetry
  - API latency, throughput
  - LLM token usage, costs
  - Vector search performance
  - Graph query latency

- **Logs**: Structured logging (JSON)
  - Request/response logs
  - Agent execution traces
  - Tool call logs
  - Error tracking

- **Traces**: Distributed tracing
  - End-to-end request flow
  - Cross-service calls
  - LLM provider latency
  - Database query breakdown

### 3. Cost Attribution

Track costs at tenant and resource level:

```python
# Unified usage tracking
usage_tracker.record(
    tenant_id="acct_123",
    resource_type="llm_token",
    resource_id="conv_456",
    quantity=1500,  # tokens
    cost_usd=0.015,
    metadata={"model": "gpt-4o", "provider": "openai"}
)

usage_tracker.record(
    tenant_id="acct_123",
    resource_type="vector_search",
    resource_id="kb_789",
    quantity=1,  # query
    cost_usd=0.0001,
    metadata={"collection": "docs", "results": 10}
)
```

### 4. Graceful Degradation

Services should degrade gracefully:

- If vector search is down → fall back to keyword search
- If primary LLM fails → route to backup provider
- If graph database is slow → use cached results
- If billing service is unavailable → allow usage, bill later

### 5. Security by Default

- **Zero Trust**: Every request authenticated and authorized
- **Encryption**: TLS in transit, encryption at rest
- **Least Privilege**: Default deny, explicit grants
- **Audit Logging**: All access logged with tenant context
- **Secrets Management**: Vault/KMS for API keys, tokens

## Technology Stack (Reference)

This is the stack used in cortex-ai reference implementation:

| Layer | Technology | Purpose |
|-------|-----------|---------|
| Gateway | FastAPI | API Gateway (REST) |
| Gateway | MCP Servers | MCP Gateway (Tools) |
| Control Plane | PostgreSQL + Redis | IAM, RBAC, Sessions |
| Service Layer | Python + FastAPI | Microservices |
| Orchestration | LangGraph + LangChain | Agent coordination |
| OLTP | PostgreSQL | Transactional data |
| OLAP | ClickHouse (optional) | Analytics |
| Vector Store | Qdrant | Document embeddings |
| Graph Database | Neo4j | Knowledge graphs |
| Cache/Queue | Redis | Caching, task queues |
| Observability | OpenTelemetry + Prometheus | Metrics, traces |
| LLM Providers | Anthropic, OpenAI, Google | Multi-provider support |

## Next Steps

1. **Read Deployment Guides**:
   - [SaaS Deployment](../deployment/saas-deployment.md)
   - [Hybrid Deployment](../deployment/hybrid-deployment.md)
   - [On-Premise Deployment](../deployment/onprem-deployment.md)

2. **Explore Architecture Deep Dives**:
   - [Gateway Layer](./01-gateway-layer.md)
   - [Control Plane](./02-control-plane.md)
   - [Service Layer](./03-service-layer.md)
   - [Data Plane](./04-data-plane.md)

3. **Review Reference Implementation**:
   - [cortex-ai Mapping](../reference/cortex-ai-mapping.md)

4. **Study Specifications**:
   - [IAM/RBAC Spec](../../specs/control-plane/iam-rbac.yaml)
   - [Multi-tenancy Spec](../../specs/control-plane/multi-tenancy.yaml)
   - [Agent Orchestration Spec](../../specs/ai-services/agent-orchestration.yaml)

---

**Key Takeaway**: AI platforms must excel in both traditional SaaS patterns (multi-tenancy, RBAC, billing) and AI-native patterns (agents, RAG, knowledge graphs). The convergence architecture provides a unified framework for building platforms that work seamlessly across both worlds and support flexible deployment models (SaaS, Hybrid, On-Premise).
