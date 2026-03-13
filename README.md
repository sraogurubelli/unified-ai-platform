# Unified AI Platform Architecture

**Open-source architecture standard for AI platforms that unify SaaS and AI service patterns**

## Vision

Modern AI platforms must operate in both worlds: traditional SaaS architecture and AI-native patterns. This project defines a convergence architecture that:

- **Unifies Patterns**: API Gateway + MCP Gateway, traditional services + AI agents
- **Hybrid Data Plane**: OLTP + OLAP + AI-specific storage (vectors, graphs)
- **Unified Control Plane**: IAM, RBAC, multi-tenancy, observability, billing
- **Deployment Flexibility**: Pure SaaS, Hybrid (control plane SaaS, data on-prem), or Pure On-Premise

## Architecture Layers

### 1. Gateway Layer (Dual Gateway Pattern)
- **API Gateway**: Traditional HTTP/REST APIs (FastAPI, Kong, AWS API Gateway)
- **MCP Gateway**: Model Context Protocol for tools/agents (MCP servers, tool registries)

### 2. Control Plane (Unified)
- Identity & Access Management (IAM)
- Role-Based Access Control (RBAC)
- Multi-tenancy & Tenant Isolation
- Observability & Monitoring
- Usage Tracking & Billing
- Policy Enforcement

### 3. Service Layer (Platform + AI Convergence)
- **Platform Services**: Auth, Projects, Organizations, Webhooks
- **AI Services**: Agents, RAG, Knowledge Graphs, Model Management
- **Analytics Services**: Usage Analytics, Model Performance, Business Intelligence

### 4. Data Plane (Hybrid Storage)
- **OLTP**: PostgreSQL, MySQL (transactional data)
- **OLAP**: ClickHouse, StarRocks, BigQuery (analytics)
- **Vector Store**: Qdrant, Pinecone, Weaviate (embeddings)
- **Graph Database**: Neo4j, NebulaGraph (knowledge graphs)
- **Cache/Queue**: Redis, Kafka (real-time data)

## Deployment Models

### 1. Pure SaaS (Multi-tenant)
- Full platform hosted and managed
- Shared infrastructure with logical isolation
- Best for: Startups, SMBs, rapid deployment

### 2. Hybrid (Control Plane SaaS, Data On-Premise)
- Control plane (IAM, billing) in cloud
- Data plane (vectors, graphs, databases) on-premise
- Best for: Regulated industries, data residency requirements

### 3. Pure On-Premise (Air-gapped)
- Complete platform deployed in customer environment
- Full data sovereignty and control
- Best for: Government, defense, high-security enterprises

## Project Structure

```
unified-ai-platform/
├── docs/
│   ├── architecture/
│   │   ├── 00-convergence-overview.md
│   │   ├── 01-gateway-layer.md
│   │   ├── 02-control-plane.md
│   │   ├── 03-service-layer.md
│   │   └── 04-data-plane.md
│   ├── deployment/
│   │   ├── saas-deployment.md
│   │   ├── hybrid-deployment.md
│   │   └── onprem-deployment.md
│   └── reference/
│       ├── cortex-ai-mapping.md
│       └── comparison-matrix.md
├── specs/
│   ├── control-plane/
│   │   ├── iam-rbac.yaml
│   │   ├── multi-tenancy.yaml
│   │   └── observability.yaml
│   ├── platform-services/
│   │   ├── auth-service.yaml
│   │   └── project-service.yaml
│   ├── ai-services/
│   │   ├── agent-orchestration.yaml
│   │   ├── rag-service.yaml
│   │   └── knowledge-graph.yaml
│   └── data-plane/
│       ├── storage-strategy.yaml
│       └── data-isolation.yaml
└── examples/
    ├── cortex-ai/           # Reference implementation
    └── minimal-starter/     # Minimal starting template
```

## Key Differentiators

1. **Architecture Convergence**: First to formally merge SaaS and AI patterns
2. **Deployment Flexibility**: Support for SaaS, Hybrid, and On-Premise from day one
3. **Open Standards**: OpenAPI, MCP, OpenTelemetry, OpenID Connect
4. **Reference Implementation**: cortex-ai as production-ready example
5. **Multi-tenancy Native**: Tenant isolation strategies for all deployment models

## Reference Implementation

[cortex-ai](../cortex-ai/) serves as the reference implementation demonstrating:
- FastAPI + SQLAlchemy for OLTP
- Qdrant (vectors) + Neo4j (graphs) for AI storage
- LangGraph for agent orchestration
- JWT + RBAC for access control
- Multi-provider LLM support (Anthropic, OpenAI, Google)
- SSE streaming for real-time responses

## Contributing

This is an open-source architecture standard. Contributions welcome for:
- Architecture patterns and best practices
- Deployment guides and runbooks
- Reference implementations in different tech stacks
- Case studies and real-world applications

## License

MIT License - See LICENSE file

## Maintainers

- Srinivasa Rao Gurubelli (@sraogurubelli)

---

**Status**: Alpha - Architecture specification in progress
**Version**: 0.1.0
**Last Updated**: March 2026
