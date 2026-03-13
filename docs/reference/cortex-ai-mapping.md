# Cortex-AI Reference Implementation Mapping

## Overview

[cortex-ai](../../../cortex-ai/) is the reference implementation of the unified AI platform architecture. This document maps cortex-ai components to the convergence architecture layers.

## Architecture Mapping

### Gateway Layer

| Architecture Layer | Cortex-AI Implementation | Location |
|-------------------|--------------------------|----------|
| **API Gateway** | FastAPI application | [cortex/api/main.py](../../../cortex-ai/cortex/api/main.py) |
| REST Endpoints | FastAPI routes | [cortex/api/routes/](../../../cortex-ai/cortex/api/routes/) |
| Request Validation | Pydantic models | Throughout route definitions |
| **MCP Gateway** | MCP Server integration (optional) | To be implemented |
| Tool Registry | ToolRegistry class | [cortex/orchestration/tools/registry.py](../../../cortex-ai/cortex/orchestration/tools/registry.py) |

### Control Plane

| Component | Cortex-AI Implementation | Location |
|-----------|--------------------------|----------|
| **IAM / RBAC** | | |
| Authentication | JWT token validation | [cortex/platform/auth/jwt.py](../../../cortex-ai/cortex/platform/auth/jwt.py) |
| Authorization | Permission checker | [cortex/platform/auth/dependencies.py](../../../cortex-ai/cortex/platform/auth/dependencies.py) |
| RBAC | Role enum, Membership model | [cortex/platform/database/models.py:55-267](../../../cortex-ai/cortex/platform/database/models.py) |
| Principal Management | Principal model | [cortex/platform/database/models.py:155-189](../../../cortex-ai/cortex/platform/database/models.py) |
| **Multi-tenancy** | | |
| Account Model | Account table | [cortex/platform/database/models.py:82-119](../../../cortex-ai/cortex/platform/database/models.py) |
| Organization Model | Organization table | [cortex/platform/database/models.py:121-153](../../../cortex-ai/cortex/platform/database/models.py) |
| Project Model | Project table | [cortex/platform/database/models.py:270-303](../../../cortex-ai/cortex/platform/database/models.py) |
| Hierarchy | Account → Org → Project | Database relationships |
| **Observability** | | |
| Metrics | OpenTelemetry | [cortex/platform/observability/telemetry.py](../../../cortex-ai/cortex/platform/observability/telemetry.py) |
| Usage Tracking | ModelUsageTracker | [cortex/orchestration/usage_tracking.py](../../../cortex-ai/cortex/orchestration/usage_tracking.py) |
| HTTP Logging | To be implemented | Planned |
| **Policy Enforcement** | | |
| Rate Limiting | To be implemented | Planned |
| Quota Management | Usage tracking foundation | Partial |

### Service Layer

#### Platform Services

| Service | Cortex-AI Implementation | Location |
|---------|--------------------------|----------|
| **Auth Service** | | |
| Signup | POST /api/v1/auth/signup | [cortex/api/routes/auth.py](../../../cortex-ai/cortex/api/routes/auth.py) |
| Login | POST /api/v1/auth/login | [cortex/api/routes/auth.py](../../../cortex-ai/cortex/api/routes/auth.py) |
| Token Management | Token model, JWT utils | [cortex/platform/auth/](../../../cortex-ai/cortex/platform/auth/) |
| **Organization Service** | | |
| CRUD Operations | OrganizationRepository | [cortex/platform/database/repositories.py](../../../cortex-ai/cortex/platform/database/repositories.py) |
| Member Management | Membership model | [cortex/platform/database/models.py:228-267](../../../cortex-ai/cortex/platform/database/models.py) |
| **Project Service** | | |
| CRUD Operations | ProjectRepository | [cortex/platform/database/repositories.py](../../../cortex-ai/cortex/platform/database/repositories.py) |
| Project-scoped Resources | Conversations, Messages | [cortex/platform/database/models.py:306-368](../../../cortex-ai/cortex/platform/database/models.py) |

#### AI Services

| Service | Cortex-AI Implementation | Location |
|---------|--------------------------|----------|
| **Agent Orchestration** | | |
| Agent Framework | Agent class | [cortex/orchestration/agent.py](../../../cortex-ai/cortex/orchestration/agent.py) |
| Multi-agent Coordination | HandoffTool, SwarmOrchestrator | [cortex/orchestration/tools/handoff.py](../../../cortex-ai/cortex/orchestration/tools/handoff.py) |
| Tool Registry | ToolRegistry | [cortex/orchestration/tools/registry.py](../../../cortex-ai/cortex/orchestration/tools/registry.py) |
| Session Persistence | LangGraph checkpointer | [cortex/orchestration/session/checkpointer.py](../../../cortex-ai/cortex/orchestration/session/checkpointer.py) |
| Streaming | StreamWriter, SSE | [cortex/core/streaming/stream_writer.py](../../../cortex-ai/cortex/core/streaming/stream_writer.py) |
| **RAG Service** | | |
| Vector Search | Qdrant integration | [cortex/search/vector/qdrant.py](../../../cortex-ai/cortex/search/vector/qdrant.py) |
| Document Processing | Document models | [cortex/search/vector/models.py](../../../cortex-ai/cortex/search/vector/models.py) |
| Embedding Generation | OpenAI/Anthropic providers | [cortex/orchestration/providers/](../../../cortex-ai/cortex/orchestration/providers/) |
| **Knowledge Graph** | | |
| Graph Storage | Neo4j integration | [cortex/search/graph/neo4j.py](../../../cortex-ai/cortex/search/graph/neo4j.py) |
| Entity Extraction | GraphRAG extractor | [cortex/search/graph/extractor.py](../../../cortex-ai/cortex/search/graph/extractor.py) |
| Graph Queries | Neo4j query builder | [cortex/search/graph/](../../../cortex-ai/cortex/search/graph/) |
| **Model Management** | | |
| Provider Abstraction | BaseProvider | [cortex/orchestration/providers/base.py](../../../cortex-ai/cortex/orchestration/providers/base.py) |
| Multi-provider Support | Anthropic, OpenAI, Google | [cortex/orchestration/providers/](../../../cortex-ai/cortex/orchestration/providers/) |
| Model Selection | ModelConfig | [cortex/orchestration/models.py](../../../cortex-ai/cortex/orchestration/models.py) |

#### Analytics Services

| Service | Cortex-AI Implementation | Location |
|---------|--------------------------|----------|
| Usage Analytics | ModelUsageTracker | [cortex/orchestration/usage_tracking.py](../../../cortex-ai/cortex/orchestration/usage_tracking.py) |
| Performance Metrics | OpenTelemetry instrumentation | [cortex/platform/observability/](../../../cortex-ai/cortex/platform/observability/) |

### Data Plane

| Storage Type | Technology | Cortex-AI Implementation | Location |
|--------------|-----------|--------------------------|----------|
| **OLTP** | PostgreSQL | SQLAlchemy models | [cortex/platform/database/models.py](../../../cortex-ai/cortex/platform/database/models.py) |
| Connection | AsyncPG | Database initialization | [cortex/platform/database/__init__.py](../../../cortex-ai/cortex/platform/database/__init__.py) |
| Repositories | Repository pattern | [cortex/platform/database/repositories.py](../../../cortex-ai/cortex/platform/database/repositories.py) |
| **OLAP** | ClickHouse (optional) | Not yet implemented | Planned |
| **Vector Store** | Qdrant | QdrantVectorStore | [cortex/search/vector/qdrant.py](../../../cortex-ai/cortex/search/vector/qdrant.py) |
| **Graph Database** | Neo4j | Neo4jGraphStore | [cortex/search/graph/neo4j.py](../../../cortex-ai/cortex/search/graph/neo4j.py) |
| **Cache/Queue** | Redis | Redis Streams, session storage | [cortex/storage/](../../../cortex-ai/cortex/storage/) |

## Code Examples

### 1. Request Flow (Chat Endpoint)

```python
# File: cortex/api/routes/chat.py

@router.post("/projects/{project_uid}/chat/stream")
async def chat_stream(
    project_uid: str,
    request: ChatRequest,
    principal: Principal = Depends(
        require_permission(Permission.CONVERSATION_CREATE, "project", "project_uid")
    ),
    session: AsyncSession = Depends(get_db),
):
    # Gateway Layer: FastAPI receives request
    # Control Plane: Permission check via require_permission()

    # Load project (multi-tenancy enforcement)
    project_repo = ProjectRepository(session)
    project = await project_repo.find_by_uid(project_uid)

    # Load organization and account (tenant hierarchy)
    org_repo = OrganizationRepository(session)
    organization = await org_repo.find_by_id(project.organization_id)
    account_repo = AccountRepository(session)
    account = await account_repo.find_by_id(organization.account_id)

    # Build agent with context (Service Layer: AI Services)
    tool_registry = ToolRegistry()
    tool_registry.set_context(
        tenant_id=account.uid,
        project_id=project.uid,
        # ... context injection for multi-tenancy
    )

    agent = Agent(
        name="assistant",
        model=ModelConfig(model=request.model or "gpt-4o"),
        tool_registry=tool_registry,
        checkpointer=get_checkpointer(),  # Session persistence
    )

    # Execute agent (AI orchestration)
    # Data Plane: Queries PostgreSQL, Qdrant, Neo4j as needed
    result = await agent.stream_to_writer(...)

    # Return SSE stream (Streaming)
    return await create_streaming_response(stream_writer)
```

**Mapping**:
- **Gateway Layer**: FastAPI route, Pydantic validation
- **Control Plane**: `require_permission()` RBAC check
- **Multi-tenancy**: Account → Organization → Project hierarchy
- **Service Layer**: Agent orchestration, tool execution
- **Data Plane**: PostgreSQL (conversations), Qdrant (context), LLM calls

### 2. Multi-tenancy Pattern

```python
# File: cortex/platform/database/models.py

class Account(Base):
    """Top-level tenant (billing entity)."""
    __tablename__ = "accounts"
    id = Column(Integer, primary_key=True)
    uid = Column(String(255), unique=True, nullable=False)
    subscription_tier = Column(Enum(SubscriptionTier), ...)
    organizations = relationship("Organization", cascade="all, delete-orphan")

class Organization(Base):
    """Business unit within account."""
    __tablename__ = "organizations"
    account_id = Column(Integer, ForeignKey("accounts.id"))
    projects = relationship("Project", cascade="all, delete-orphan")

class Project(Base):
    """Workspace within organization."""
    __tablename__ = "projects"
    organization_id = Column(Integer, ForeignKey("organizations.id"))
    conversations = relationship("Conversation", cascade="all, delete-orphan")

class Conversation(Base):
    """AI conversation (tenant-scoped)."""
    __tablename__ = "conversations"
    project_id = Column(Integer, ForeignKey("projects.id"))
    # Implicit tenant via project → org → account
```

**Mapping**:
- **Hierarchy**: Account (tenant) → Organization → Project → Resources
- **Cascade Delete**: Deleting account deletes all child resources
- **Data Isolation**: Every conversation belongs to exactly one tenant

### 3. RBAC Permission Check

```python
# File: cortex/platform/auth/dependencies.py

def require_permission(
    permission: Permission,
    resource_type: str,
    resource_param: str = "resource_id",
) -> Callable:
    """Dependency factory for permission checks."""

    async def dependency(
        request: Request,
        session: AsyncSession = Depends(get_db),
        principal: Principal = Depends(get_current_principal),
    ) -> Principal:
        # Extract resource ID from path params
        resource_id = request.path_params.get(resource_param)

        # Check permission
        checker = await get_permission_checker(session)
        has_access = await checker.check(
            principal=principal,
            resource_type=resource_type,
            resource_id=resource_id,
            permission=permission,
        )

        if not has_access:
            raise HTTPException(status_code=403, detail="Access denied")

        return principal

    return dependency
```

**Mapping**:
- **Control Plane**: Centralized RBAC enforcement
- **Applies to**: Both traditional APIs and AI services
- **Resource Types**: "project", "document", "agent", etc.

### 4. Agent Orchestration with Tools

```python
# File: cortex/orchestration/agent.py

class Agent:
    """LangGraph-based agent with tool support."""

    def __init__(
        self,
        name: str,
        model: ModelConfig,
        tool_registry: ToolRegistry,
        checkpointer: BaseCheckpointSaver,
    ):
        self.name = name
        self.model = model
        self.tools = tool_registry.get_tools()
        self.checkpointer = checkpointer

    async def run(self, message: str, thread_id: str) -> AgentResult:
        """Execute agent with context from tool registry."""

        # Build LangGraph workflow
        graph = self._build_graph()

        # Execute with session persistence
        config = {"configurable": {"thread_id": thread_id}}
        result = await graph.ainvoke(
            {"messages": [HumanMessage(content=message)]},
            config=config
        )

        return AgentResult(
            response=result["messages"][-1].content,
            messages=result["messages"],
            token_usage=result.get("usage", {})
        )
```

**Mapping**:
- **Service Layer**: AI orchestration
- **Tool Registry**: Context injection (tenant_id, project_id)
- **Session Persistence**: LangGraph checkpointer (Redis)
- **Multi-provider**: Model selected via ModelConfig

### 5. Vector Search (RAG)

```python
# File: cortex/search/vector/qdrant.py

class QdrantVectorStore:
    """Qdrant vector store for semantic search."""

    async def search(
        self,
        query_embedding: list[float],
        collection_name: str,
        tenant_id: str,  # Multi-tenancy
        limit: int = 10,
    ) -> list[Document]:
        """Search with tenant filtering."""

        results = await self.client.search(
            collection_name=collection_name,
            query_vector=query_embedding,
            query_filter=Filter(
                must=[
                    FieldCondition(
                        key="tenant_id",
                        match=MatchValue(value=tenant_id)
                    )
                ]
            ),
            limit=limit
        )

        return [Document.from_qdrant(hit) for hit in results]
```

**Mapping**:
- **Data Plane**: Qdrant vector storage
- **Multi-tenancy**: Tenant filtering in queries
- **AI Service**: RAG pipeline uses this for context retrieval

## Deployment Model Support

### Current: SaaS (Multi-tenant)

Cortex-AI is currently configured for SaaS deployment:

- **Multi-tenancy**: Logical isolation via `tenant_id` (Account)
- **Shared infrastructure**: All tenants share databases
- **Scalability**: Kubernetes-ready, stateless API servers
- **Cloud-native**: Docker Compose (dev), Kubernetes (prod)

Configuration:
```env
# .env
DEPLOYMENT_MODEL=saas
DATABASE_URL=postgresql://cortex-db.cloud:5432/cortex
QDRANT_URL=https://qdrant-cloud.cortex.ai
NEO4J_URL=bolt://neo4j-aura.cortex.ai:7687
```

### Planned: Hybrid & On-Premise

To support hybrid and on-premise deployments:

1. **Configuration Management**:
   ```python
   # cortex/platform/config.py
   class DeploymentConfig:
       model: Literal["saas", "hybrid", "onprem"]
       control_plane_url: Optional[str]  # For hybrid
       ldap_url: Optional[str]  # For onprem
   ```

2. **LDAP Integration** (on-prem):
   - Add LDAPAuthProvider
   - Integrate with customer Active Directory

3. **Air-gapped LLMs** (on-prem):
   - Add OllamaProvider
   - Support vLLM, TGI

4. **Hybrid Routing**:
   - Proxy requests to customer data plane
   - Control plane stays in SaaS

## Testing Architecture Alignment

### Unit Tests

Test individual components in isolation:

```python
# tests/unit/test_rbac.py
async def test_permission_check():
    """Test RBAC permission enforcement."""
    checker = PermissionChecker(session)
    has_access = await checker.check(
        principal=principal,
        resource_type="project",
        resource_id="proj_123",
        permission=Permission.CONVERSATION_CREATE,
    )
    assert has_access is True
```

### Integration Tests

Test inter-layer communication:

```python
# tests/integration/test_chat_flow.py
async def test_chat_request_flow():
    """Test full chat request flow through all layers."""

    # Gateway: API request
    response = await client.post(
        "/api/v1/projects/proj_123/chat",
        headers={"Authorization": f"Bearer {jwt_token}"},
        json={"message": "Hello"}
    )

    # Control Plane: Auth verified
    # Service Layer: Agent executed
    # Data Plane: Conversation stored
    assert response.status_code == 200
    conversation = response.json()
    assert conversation["response"] is not None
```

## Gaps & Roadmap

### Implemented ✅

- Gateway Layer: FastAPI API Gateway
- Control Plane: IAM/RBAC, multi-tenancy, basic observability
- Service Layer: Auth, projects, agent orchestration, RAG
- Data Plane: PostgreSQL, Redis, Qdrant, Neo4j

### In Progress 🚧

- MCP Gateway
- HTTP logging
- Quota enforcement
- Hybrid deployment support

### Planned 📋

- ClickHouse for OLAP
- LDAP/SAML integration
- Self-hosted LLM support (Ollama)
- Advanced analytics dashboard
- On-premise deployment guides

---

**Cortex-AI** demonstrates how the unified architecture works in practice, serving as a production-ready reference for others building AI platforms.
