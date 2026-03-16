# Team Ownership & Responsibilities

## Overview

The parallel stack architecture enables clear team ownership with minimal coordination overhead. Teams can work independently on their respective stacks while shared services are managed collaboratively.

---

## Team Structure

```
┌────────────────────────────────────────────────────────────┐
│                 Platform/Shared Team                        │
│   Auth · Multi-tenancy · Observability · Secrets           │
└───────────────────┬───────────────────┬────────────────────┘
                    │                   │
        ┌───────────┴────────┐  ┌───────┴────────────┐
        ▼                    ▼  ▼                    ▼
┌─────────────────┐  ┌─────────────────────────────────┐
│ Traditional     │  │ AI Stack Team                   │
│ Stack Team      │  │                                 │
│                 │  │                                 │
│ • API Gateway   │  │ • MCP Gateway                   │
│ • CRUD Services │  │ • LLM Gateway                   │
│ • PostgreSQL    │  │ • Agent Orchestration           │
│ • ClickHouse    │  │ • RAG Service                   │
│ • Analytics     │  │ • Qdrant, Neo4j                 │
└─────────────────┘  └─────────────────────────────────┘
```

---

## Team 1: Traditional Stack Team

### Responsibilities

| Area | Components | Tasks |
|------|-----------|-------|
| **Gateway** | API Gateway (REST/GraphQL) | • Implement API endpoints<br>• Request validation<br>• Rate limiting<br>• CORS configuration |
| **Services** | Auth, Project, Document, Webhook, Analytics | • CRUD operations<br>• Business logic<br>• Webhooks<br>• Traditional analytics |
| **Data** | PostgreSQL, ClickHouse, Redis (session) | • Schema design<br>• Query optimization<br>• Migrations<br>• Backup/restore |
| **Control Plane** | API quotas, Billing, Audit logs | • Quota enforcement<br>• Invoice generation<br>• Compliance logging |

### Documentation Ownership

| File | Responsibility |
|------|----------------|
| [01a-traditional-api-gateway.md](../01-gateway-layer/01a-traditional-api-gateway.md) | ✅ Maintain |
| [02a-traditional-control-plane.md](../02-control-plane/02a-traditional-control-plane.md) | ✅ Maintain |
| [03a-traditional-services.md](../03-service-layer/03a-traditional-services.md) | ✅ Maintain |
| [04a-traditional-data-platform.md](../04-data-plane/04a-traditional-data-platform.md) | ✅ Maintain |

### Technologies

- **Languages**: Python (FastAPI), TypeScript
- **Databases**: PostgreSQL, ClickHouse/StarRocks
- **Cache**: Redis
- **Infrastructure**: Docker, Kubernetes, AWS
- **Testing**: pytest, integration tests

### Team Size

- **Minimum**: 2-3 engineers
- **Recommended**: 4-6 engineers
- **Breakdown**:
  - 2x Backend Engineers (API/Services)
  - 1x Data Engineer (PostgreSQL, ClickHouse)
  - 1x DevOps Engineer (shared with platform team)
  - 1x Frontend Engineer (optional, if web UI)

---

## Team 2: AI Stack Team

### Responsibilities

| Area | Components | Tasks |
|------|-----------|-------|
| **Gateway** | MCP Gateway | • Tool discovery<br>• Tool permissions<br>• Context injection<br>• Streaming responses |
| **Services** | LLM Gateway, RAG, Agents, Memory | • LLM orchestration<br>• Semantic caching<br>• Agent coordination<br>• Graph RAG implementation |
| **Data** | Qdrant, Neo4j, Semantic Cache | • Vector indexing<br>• Knowledge graph<br>• Entity extraction<br>• Graph algorithms |
| **Control Plane** | LLM quotas, Cost tracking, AI policies | • Model routing<br>• Cost optimization<br>• Content filtering<br>• Tool permissions |

### Documentation Ownership

| File | Responsibility |
|------|----------------|
| [01b-ai-mcp-gateway.md](../01-gateway-layer/01b-ai-mcp-gateway.md) | ✅ Maintain |
| [02b-ai-control-plane.md](../02-control-plane/02b-ai-control-plane.md) | ✅ Maintain |
| [03b-ai-services.md](../03-service-layer/03b-ai-services.md) | ✅ Maintain |
| [04b-ai-data-platform.md](../04-data-plane/04b-ai-data-platform.md) | ✅ Maintain |

### Technologies

- **Languages**: Python (LangChain, LangGraph)
- **AI/ML**: OpenAI, Anthropic, Cohere
- **Databases**: Qdrant, Neo4j
- **Cache**: Redis (semantic cache)
- **Infrastructure**: Docker, Kubernetes, AWS
- **Testing**: pytest, LLM evaluation frameworks

### Team Size

- **Minimum**: 3-4 engineers
- **Recommended**: 6-8 engineers
- **Breakdown**:
  - 2x AI Engineers (LLM, RAG)
  - 2x Backend Engineers (Services, MCP)
  - 1x ML Engineer (Embeddings, Evaluation)
  - 1x Data Engineer (Qdrant, Neo4j)
  - 1x DevOps Engineer (shared with platform team)

---

## Team 3: Platform/Shared Team

### Responsibilities

| Area | Components | Tasks |
|------|-----------|-------|
| **Authentication** | JWT, SSO/SAML | • Token management<br>• SSO integration<br>• Key rotation |
| **Multi-tenancy** | Account → Org → Project | • Hierarchy management<br>• Tenant isolation<br>• Context injection |
| **Observability** | Prometheus, Grafana, Jaeger | • Metrics collection<br>• Dashboards<br>• Distributed tracing<br>• Alerting |
| **RBAC** | Permissions, Roles, Memberships | • Permission model<br>• Role assignments<br>• Access checks |
| **Event Bus** | Kafka, Event routing | • Topic management<br>• Schema registry<br>• Event routing |
| **Secrets** | Vault, Key rotation | • Secret storage<br>• Key management<br>• Rotation policies |

### Documentation Ownership

| File | Responsibility |
|------|----------------|
| [01c-gateway-integration.md](../01-gateway-layer/01c-gateway-integration.md) | ✅ Maintain |
| [02c-control-plane-shared.md](../02-control-plane/02c-control-plane-shared.md) | ✅ Maintain |
| [03c-service-integration.md](../03-service-layer/03c-service-integration.md) | ✅ Maintain |
| [04c-data-integration.md](../04-data-plane/04c-data-integration.md) | ✅ Maintain |
| [00-parallel-stacks-overview.md](../00-parallel-stacks-overview.md) | ✅ Maintain |

### Technologies

- **Languages**: Python, Go
- **Infrastructure**: Kubernetes, Terraform, Helm
- **Observability**: Prometheus, Grafana, Jaeger, ELK
- **Security**: Vault, AWS Secrets Manager
- **Event Bus**: Kafka, Redis Streams
- **Testing**: Integration tests, load tests

### Team Size

- **Minimum**: 2-3 engineers
- **Recommended**: 4-5 engineers
- **Breakdown**:
  - 2x DevOps/SRE Engineers
  - 1x Security Engineer
  - 1x Platform Engineer (shared services)

---

## Communication Patterns

### Async Communication (Preferred)

**Event Bus** for cross-team integration:

| Event | Producer Team | Consumer Team |
|-------|--------------|---------------|
| `document.uploaded` | Traditional | AI (for embedding) |
| `message.created` | AI | AI (for entity extraction) |
| `user.created` | Traditional/Shared | Both (for notifications) |
| `quota.exceeded` | Traditional/AI | Shared (for notifications) |

### Sync Communication (When Needed)

**API Calls** for real-time needs:

| Call | From Team | To Team | Purpose |
|------|-----------|---------|---------|
| Create Project | AI (Agent) | Traditional | Agent creates project via API |
| RAG Retrieval | Traditional (Chat UI) | AI | Get semantic search results |
| Cost Dashboard | Traditional (Analytics) | AI | Get LLM cost metrics |

---

## Development Workflow

### Independent Development

```
Traditional Stack Team              AI Stack Team
        │                                  │
        ├─ Feature Branch                  ├─ Feature Branch
        │  (traditional/feature-x)         │  (ai/feature-y)
        │                                  │
        ├─ Tests (Traditional only)        ├─ Tests (AI only)
        │                                  │
        ├─ Deploy to Dev                   ├─ Deploy to Dev
        │  (Traditional services)          │  (AI services)
        │                                  │
        └─ PR Review                        └─ PR Review
           (Traditional team)                  (AI team)
```

### Integration Points (Shared Team Coordinates)

```
        Traditional Team         AI Team
                 │                 │
                 └────┬────────────┘
                      │
                Platform Team
                      │
            ┌─────────┴──────────┐
            │                    │
      Event Schema         API Contracts
      (Kafka topics)       (OpenAPI)
```

---

## Deployment Independence

### Separate Deployment Pipelines

**Traditional Stack**:
```yaml
# .github/workflows/deploy-traditional.yml
name: Deploy Traditional Stack
on:
  push:
    paths:
      - 'services/traditional/**'
      - 'gateway/api-gateway/**'
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy API Gateway
        run: kubectl apply -f k8s/api-gateway.yaml
      - name: Deploy Traditional Services
        run: kubectl apply -f k8s/traditional-services.yaml
```

**AI Stack**:
```yaml
# .github/workflows/deploy-ai.yml
name: Deploy AI Stack
on:
  push:
    paths:
      - 'services/ai/**'
      - 'gateway/mcp-gateway/**'
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy MCP Gateway
        run: kubectl apply -f k8s/mcp-gateway.yaml
      - name: Deploy AI Services
        run: kubectl apply -f k8s/ai-services.yaml
```

---

## On-Call Rotation

### Tier 1: Platform Team (24/7)

- **Scope**: Infrastructure, observability, shared services
- **Escalates to**: Traditional or AI team based on issue

### Tier 2: Traditional Stack Team (Business Hours)

- **Scope**: API Gateway, CRUD services, PostgreSQL, ClickHouse
- **Alerts**:
  - API Gateway p95 latency >1s
  - PostgreSQL connection pool >80%
  - ClickHouse query failures

### Tier 3: AI Stack Team (Business Hours)

- **Scope**: MCP Gateway, LLM Gateway, RAG, Qdrant, Neo4j
- **Alerts**:
  - LLM costs >$100/hour
  - Semantic cache hit rate <85%
  - Qdrant query latency >100ms

---

## Metrics & KPIs

### Traditional Stack Team

| Metric | Target | Measurement |
|--------|--------|-------------|
| API p95 latency | <200ms | Prometheus |
| API error rate | <0.1% | Prometheus |
| Database query p95 | <50ms | PostgreSQL stats |
| Webhook delivery rate | >99% | Custom metric |

### AI Stack Team

| Metric | Target | Measurement |
|--------|--------|-------------|
| LLM p95 latency | <2s | Prometheus |
| Semantic cache hit rate | >85% | Custom metric |
| RAG retrieval accuracy | >80% | LLM-as-Judge |
| Agent task completion | >90% | Custom metric |

### Platform Team

| Metric | Target | Measurement |
|--------|--------|-------------|
| Platform uptime | 99.9% | Prometheus |
| Auth latency p95 | <100ms | Prometheus |
| Event bus lag | <1 minute | Kafka metrics |
| Secret rotation | Monthly | Manual checklist |

---

## Hiring Profiles

### Traditional Stack Engineer

**Required**:
- Python/TypeScript proficiency
- REST API design
- PostgreSQL/SQL expertise
- Docker/Kubernetes basics

**Nice to have**:
- ClickHouse/StarRocks experience
- FastAPI framework
- AWS/GCP experience

### AI Stack Engineer

**Required**:
- Python proficiency
- LLM experience (OpenAI/Anthropic)
- Vector databases (Qdrant/Pinecone)
- Understanding of RAG patterns

**Nice to have**:
- LangChain/LangGraph
- Graph databases (Neo4j)
- Agent orchestration experience
- ML engineering background

### Platform Engineer

**Required**:
- Kubernetes expertise
- Infrastructure as Code (Terraform)
- Observability tools (Prometheus, Grafana)
- Security best practices

**Nice to have**:
- Service mesh (Istio)
- Vault/Secrets management
- Multi-cloud experience

---

**Previous**: [Deploy Full Platform](./05c-deploy-full-platform.md) | [Traditional-Only](./05a-deploy-traditional-only.md) | [AI-Only](./05b-deploy-ai-only.md) | [Back to Overview](../00-parallel-stacks-overview.md)
