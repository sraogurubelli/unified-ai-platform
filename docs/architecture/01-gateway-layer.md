# Gateway Layer: Dual Gateway Pattern

## Overview

The gateway layer is the entry point for all external traffic. In the convergence architecture, we implement a **dual gateway pattern** that handles both traditional API requests and AI-native tool/agent communication.

```
External Traffic
       ▼
┌──────────────────────────────────────────────────┐
│         Load Balancer / CDN                       │
│    (CloudFlare, AWS ALB, NGINX, HAProxy)          │
└────────────┬────────────────────┬─────────────────┘
             ▼                    ▼
┌────────────────────────┐  ┌────────────────────────┐
│   API Gateway          │  │   MCP Gateway          │
│   (HTTP/REST/GraphQL)  │  │   (Model Context       │
│                        │  │    Protocol)           │
│  - Authentication      │  │  - Tool discovery      │
│  - Rate limiting       │  │  - Agent protocol      │
│  - Request routing     │  │  - Context management  │
│  - Response transform  │  │  - Tool permissions    │
└────────────────────────┘  └────────────────────────┘
             │                    │
             └────────┬───────────┘
                      ▼
           ┌─────────────────────┐
           │   Control Plane     │
           │   (IAM, RBAC)       │
           └─────────────────────┘
```

## Why Dual Gateways?

### Traditional API Gateway

Handles **SaaS-style** interactions:
- User authentication (JWT, OAuth)
- CRUD operations on resources
- RESTful or GraphQL APIs
- Request/response transformation
- Rate limiting per tenant

**Use cases**:
- User signup/login
- Create project, upload document
- List conversations, get analytics
- Manage billing and settings

### MCP Gateway

Handles **AI-native** interactions:
- Tool discovery and registration
- Agent-to-tool communication
- Context injection and management
- Streaming tool responses
- Tool permission enforcement

**Use cases**:
- Agent discovers available tools
- Agent invokes tools (search, database, API calls)
- Multi-agent handoff and coordination
- Real-time streaming from tools

### Convergence Point

Both gateways share:
- **Same authentication system** (JWT tokens)
- **Same RBAC enforcement** (unified permissions)
- **Same rate limiting** (tenant-scoped quotas)
- **Same observability** (unified metrics/logs)

## API Gateway Deep Dive

### Technology Options

| Technology | Best For | Pros | Cons |
|------------|----------|------|------|
| **FastAPI** | Python-native apps | Fast, async, OpenAPI auto-gen | Limited enterprise features |
| **Kong** | Enterprise, K8s | Plugins, scalability, community | Complex configuration |
| **AWS API Gateway** | AWS deployments | Managed, integrated, scalable | Vendor lock-in, cost |
| **NGINX** | On-premise, custom | Flexible, performant, proven | Manual configuration |
| **Traefik** | Kubernetes | Auto-discovery, K8s-native | Learning curve |
| **Envoy** | Service mesh | Advanced routing, observability | Complexity |

### FastAPI Implementation (Reference)

```python
# cortex/api/main.py

from fastapi import FastAPI, Request, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from fastapi.middleware.trustedhost import TrustedHostMiddleware
from slowapi import Limiter
from slowapi.util import get_remote_address

app = FastAPI(
    title="Cortex AI Platform API",
    version="0.1.0",
    docs_url="/docs",
    redoc_url="/redoc",
)

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://app.cortex-ai.com"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Rate limiting
limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter

# Authentication middleware
@app.middleware("http")
async def auth_middleware(request: Request, call_next):
    """Extract and validate JWT from Authorization header."""

    # Skip auth for public endpoints
    if request.url.path in ["/health", "/docs", "/openapi.json"]:
        return await call_next(request)

    # Extract JWT
    auth_header = request.headers.get("Authorization", "")
    if not auth_header.startswith("Bearer "):
        raise HTTPException(status_code=401, detail="Missing or invalid token")

    token = auth_header.replace("Bearer ", "")

    # Validate JWT
    try:
        payload = jwt.decode(token, settings.JWT_SECRET, algorithms=["HS256"])
        request.state.tenant_id = payload["tenant_id"]
        request.state.principal_id = payload["principal_id"]
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="Invalid token")

    response = await call_next(request)
    return response

# Request logging middleware
@app.middleware("http")
async def logging_middleware(request: Request, call_next):
    """Log all requests with tenant context."""

    logger.info(
        "api_request",
        method=request.method,
        path=request.url.path,
        tenant_id=getattr(request.state, "tenant_id", None),
        principal_id=getattr(request.state, "principal_id", None),
    )

    response = await call_next(request)

    logger.info(
        "api_response",
        status_code=response.status_code,
        tenant_id=getattr(request.state, "tenant_id", None),
    )

    return response

# Health check
@app.get("/health")
async def health():
    return {"status": "healthy"}

# Include routers
from cortex.api.routes import auth, chat, projects, documents

app.include_router(auth.router)
app.include_router(chat.router)
app.include_router(projects.router)
app.include_router(documents.router)
```

### Kong Implementation (Enterprise)

```yaml
# kong.yaml
_format_version: "3.0"

services:
  - name: cortex-api
    url: http://cortex-api:8000
    routes:
      - name: api-routes
        paths:
          - /api/v1
        methods:
          - GET
          - POST
          - PUT
          - DELETE
        strip_path: false

    plugins:
      # JWT authentication
      - name: jwt
        config:
          secret_is_base64: false
          key_claim_name: tenant_id
          claims_to_verify:
            - exp

      # Rate limiting (per tenant)
      - name: rate-limiting
        config:
          minute: 1000
          hour: 10000
          policy: local
          limit_by: header
          header_name: X-Tenant-ID

      # CORS
      - name: cors
        config:
          origins:
            - https://app.cortex-ai.com
          credentials: true
          max_age: 3600

      # Request transformation
      - name: request-transformer
        config:
          add:
            headers:
              - X-Gateway: kong

      # Response transformer
      - name: response-transformer
        config:
          add:
            headers:
              - X-Request-ID: $(uuid)

      # Logging
      - name: file-log
        config:
          path: /var/log/kong/api.log

# Global plugins
plugins:
  - name: prometheus
    config:
      per_consumer: true
```

### Request Flow

```
1. Client Request
   ↓
   POST /api/v1/projects/proj_123/chat
   Authorization: Bearer eyJhbGc...
   Content-Type: application/json
   {
     "message": "Analyze customer sentiment"
   }

2. Load Balancer
   ↓
   - SSL termination
   - Route to API Gateway

3. API Gateway
   ↓
   - Extract JWT from Authorization header
   - Validate signature and expiry
   - Extract tenant_id from JWT claims
   - Check rate limit for tenant_id
   - Route to backend service

4. Backend Service (FastAPI)
   ↓
   - Middleware extracts tenant context
   - Route handler requires permission check
   - Permission checker validates RBAC
   - Execute business logic
   - Return response

5. Response
   ↓
   - API Gateway adds headers (X-Request-ID, etc.)
   - Log response metrics
   - Return to client
```

### Rate Limiting Strategies

#### Per-Tenant Rate Limiting

```python
from slowapi import Limiter
from fastapi import Request

# Extract tenant ID for rate limiting
def get_tenant_id(request: Request) -> str:
    """Extract tenant ID from JWT for rate limiting."""
    return getattr(request.state, "tenant_id", "anonymous")

limiter = Limiter(key_func=get_tenant_id)

@app.post("/api/v1/projects/{project_uid}/chat")
@limiter.limit("100/minute")  # Per tenant
async def chat(request: Request, ...):
    ...
```

#### Tiered Rate Limits

```python
# Different limits per subscription tier
RATE_LIMITS = {
    "free": "10/minute",
    "pro": "100/minute",
    "team": "500/minute",
    "enterprise": "5000/minute",
}

async def get_tenant_tier(tenant_id: str) -> str:
    account = await db.query(Account).filter_by(uid=tenant_id).first()
    return account.subscription_tier

@app.post("/api/v1/chat")
async def chat(request: Request, ...):
    tier = await get_tenant_tier(request.state.tenant_id)
    limit = RATE_LIMITS[tier]

    # Apply dynamic rate limit
    await limiter.check(request, limit)
    ...
```

## MCP Gateway Deep Dive

### Model Context Protocol (MCP)

MCP is an open protocol for AI agents to discover and invoke tools. It standardizes:
- **Tool discovery**: What tools are available?
- **Tool schemas**: What parameters does a tool accept?
- **Tool invocation**: How to call a tool?
- **Context management**: How to pass tenant/user context?

### Architecture

```
Agent → MCP Gateway → MCP Server(s) → Backend Services

Example:
Agent needs to search documents
   ↓
MCP Gateway: "What tools are available?"
   ↓
MCP Server: "I provide: search_documents, get_document, create_document"
   ↓
Agent: "Call search_documents with query='AI platform'"
   ↓
MCP Gateway: Validate permissions, inject tenant context
   ↓
MCP Server: Execute search with tenant_id filter
   ↓
Backend: Query Qdrant with tenant isolation
   ↓
Return results to agent
```

### MCP Server Implementation

```python
# cortex/mcp/server.py

from mcp import MCPServer, Tool, Context

class CortexMCPServer:
    """MCP server for Cortex AI platform tools."""

    def __init__(self):
        self.server = MCPServer(name="cortex-ai")
        self._register_tools()

    def _register_tools(self):
        """Register all available tools."""

        # Document search tool
        @self.server.tool(
            name="search_documents",
            description="Search documents using semantic search",
            parameters={
                "query": {"type": "string", "description": "Search query"},
                "limit": {"type": "integer", "default": 10},
            }
        )
        async def search_documents(query: str, limit: int, context: Context):
            # Extract tenant from MCP context
            tenant_id = context.get("tenant_id")

            # Perform search with tenant isolation
            results = await vector_store.search(
                query_text=query,
                tenant_id=tenant_id,
                limit=limit
            )

            return {
                "documents": [
                    {
                        "id": doc.id,
                        "title": doc.title,
                        "snippet": doc.content[:200],
                        "score": doc.score
                    }
                    for doc in results
                ]
            }

        # Database query tool
        @self.server.tool(
            name="query_database",
            description="Query PostgreSQL database",
            parameters={
                "sql": {"type": "string", "description": "SQL query"},
            }
        )
        async def query_database(sql: str, context: Context):
            tenant_id = context.get("tenant_id")

            # Validate SQL (prevent injection)
            if not self._is_safe_sql(sql):
                raise ValueError("Unsafe SQL query")

            # Execute with tenant filter
            results = await db.execute(
                f"SELECT * FROM ({sql}) WHERE tenant_id = :tenant_id",
                {"tenant_id": tenant_id}
            )

            return {"rows": results.fetchall()}

        # Knowledge graph query tool
        @self.server.tool(
            name="query_knowledge_graph",
            description="Query Neo4j knowledge graph",
            parameters={
                "cypher": {"type": "string", "description": "Cypher query"},
            }
        )
        async def query_knowledge_graph(cypher: str, context: Context):
            tenant_id = context.get("tenant_id")

            # Inject tenant filter into Cypher
            cypher_with_tenant = f"""
            MATCH (n)
            WHERE n.tenant_id = $tenant_id
            {cypher}
            """

            results = await neo4j.execute_query(
                cypher_with_tenant,
                tenant_id=tenant_id
            )

            return {"nodes": results}

    def _is_safe_sql(self, sql: str) -> bool:
        """Basic SQL safety check."""
        forbidden = ["DROP", "DELETE", "UPDATE", "INSERT", "ALTER", "CREATE"]
        return not any(keyword in sql.upper() for keyword in forbidden)
```

### MCP Gateway Middleware

```python
# cortex/mcp/gateway.py

class MCPGateway:
    """Gateway for MCP requests with auth and context injection."""

    async def handle_request(self, request: MCPRequest) -> MCPResponse:
        """Handle MCP tool invocation request."""

        # 1. Authentication
        token = request.headers.get("Authorization", "").replace("Bearer ", "")
        try:
            payload = jwt.decode(token, settings.JWT_SECRET)
            tenant_id = payload["tenant_id"]
            principal_id = payload["principal_id"]
        except jwt.InvalidTokenError:
            return MCPResponse(error="Invalid authentication")

        # 2. Permission check
        has_permission = await self._check_tool_permission(
            principal_id=principal_id,
            tool_name=request.tool_name
        )

        if not has_permission:
            return MCPResponse(error="Permission denied")

        # 3. Inject context
        context = Context({
            "tenant_id": tenant_id,
            "principal_id": principal_id,
            "request_id": str(uuid.uuid4()),
        })

        # 4. Rate limiting
        await self._check_rate_limit(tenant_id)

        # 5. Forward to MCP server
        response = await self.mcp_server.invoke_tool(
            name=request.tool_name,
            parameters=request.parameters,
            context=context
        )

        # 6. Log usage
        await self._log_tool_usage(
            tenant_id=tenant_id,
            tool_name=request.tool_name,
            duration_ms=response.duration_ms
        )

        return response

    async def _check_tool_permission(
        self, principal_id: str, tool_name: str
    ) -> bool:
        """Check if principal has permission to use tool."""

        # Tools are resources in RBAC system
        return await rbac.check_permission(
            principal_id=principal_id,
            resource_type="tool",
            resource_id=tool_name,
            permission=Permission.TOOL_EXECUTE
        )
```

### Agent Integration

```python
# How agents use MCP gateway

from langchain.agents import Agent
from langchain.tools import MCPTool

class CortexAgent:
    """Agent with MCP tool support."""

    def __init__(self, mcp_gateway_url: str, auth_token: str):
        self.mcp_client = MCPClient(
            gateway_url=mcp_gateway_url,
            auth_token=auth_token
        )

        # Discover available tools
        self.tools = self.mcp_client.list_tools()

    async def run(self, query: str):
        """Run agent with MCP tools."""

        # Create LangChain tools from MCP tools
        langchain_tools = [
            MCPTool(
                name=tool.name,
                description=tool.description,
                mcp_client=self.mcp_client
            )
            for tool in self.tools
        ]

        # Create agent with tools
        agent = Agent(
            tools=langchain_tools,
            llm=self.llm
        )

        # Execute
        result = await agent.arun(query)
        return result
```

## Gateway Comparison

| Feature | API Gateway | MCP Gateway |
|---------|-------------|-------------|
| **Protocol** | HTTP/REST/GraphQL | MCP (JSON-RPC-like) |
| **Use Case** | CRUD operations | Tool invocation |
| **Clients** | Web apps, mobile apps | AI agents |
| **Authentication** | JWT, OAuth | JWT (same) |
| **Rate Limiting** | Per tenant | Per tenant (same) |
| **Context** | Request headers | MCP Context object |
| **Response** | JSON, HTML, SSE | Structured tool output |
| **Caching** | HTTP caching | Tool result caching |

## Unified Observability

Both gateways emit the same metrics format:

```python
# Prometheus metrics (unified)

gateway_requests_total = Counter(
    'gateway_requests_total',
    'Total gateway requests',
    ['gateway_type', 'tenant_id', 'endpoint', 'status']
)

gateway_latency = Histogram(
    'gateway_latency_seconds',
    'Gateway request latency',
    ['gateway_type', 'tenant_id', 'endpoint']
)

# Usage
# API Gateway
gateway_requests_total.labels(
    gateway_type='api',
    tenant_id='acct_123',
    endpoint='/api/v1/chat',
    status='200'
).inc()

# MCP Gateway
gateway_requests_total.labels(
    gateway_type='mcp',
    tenant_id='acct_123',
    endpoint='search_documents',
    status='200'
).inc()
```

## Security Best Practices

### 1. Input Validation

```python
# API Gateway
@app.post("/api/v1/chat")
async def chat(request: ChatRequest):  # Pydantic validates
    # request.message already validated by Pydantic
    ...

# MCP Gateway
async def invoke_tool(tool_name: str, parameters: dict):
    # Validate against tool schema
    schema = mcp_server.get_tool_schema(tool_name)
    jsonschema.validate(parameters, schema)
```

### 2. SQL Injection Prevention

```python
# Bad: String concatenation
query = f"SELECT * FROM docs WHERE id = {doc_id}"

# Good: Parameterized queries
query = "SELECT * FROM docs WHERE id = :doc_id AND tenant_id = :tenant_id"
await db.execute(query, {"doc_id": doc_id, "tenant_id": tenant_id})
```

### 3. XSS Prevention

```python
from markupsafe import escape

# Escape user input before rendering
safe_content = escape(user_input)
```

### 4. CSRF Protection

```python
from fastapi_csrf_protect import CsrfProtect

@app.post("/api/v1/action")
async def action(csrf_protect: CsrfProtect = Depends()):
    await csrf_protect.validate_csrf_in_cookies(request)
```

## Performance Optimization

### 1. Connection Pooling

```python
# Database connection pool
engine = create_async_engine(
    DATABASE_URL,
    pool_size=20,
    max_overflow=10,
    pool_pre_ping=True
)
```

### 2. Response Caching

```python
from fastapi_cache import FastAPICache
from fastapi_cache.backends.redis import RedisBackend

@app.get("/api/v1/projects/{project_id}")
@cache(expire=300)  # Cache for 5 minutes
async def get_project(project_id: str):
    ...
```

### 3. Request Compression

```python
from fastapi.middleware.gzip import GZipMiddleware

app.add_middleware(GZipMiddleware, minimum_size=1000)
```

## Frontend Architecture: Micro Frontend Pattern

Modern AI platforms require modular, independently deployable frontend applications. The **Micro Frontend Architecture** enables parallel development, independent deployment, and technology flexibility.

### Why Micro Frontends?

**Traditional Monolithic Frontend**:
- ❌ Single large bundle (5-10MB+)
- ❌ Tight coupling between modules
- ❌ All-or-nothing deployments
- ❌ Coordination overhead across teams

**Micro Frontend Approach**:
- ✅ Small, focused bundles (500KB-1MB each)
- ✅ Independent deployment per module
- ✅ Technology flexibility (React, Vue, Svelte)
- ✅ Team autonomy

### Architecture

```
┌────────────────────────────────────────────────────────────┐
│               Parent Application (Shell)                    │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Shared Services                                      │  │
│  │  - Authentication state                               │  │
│  │  - Navigation router                                  │  │
│  │  - Design system (UI components)                     │  │
│  │  - Theme provider                                     │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │ Chat Module  │  │ Admin Portal │  │ Analytics        │  │
│  │ (lazy loaded)│  │ (lazy loaded)│  │ Dashboard        │  │
│  │              │  │              │  │ (lazy loaded)    │  │
│  │ - Independ   │  │ - Independ   │  │ - Independ       │  │
│  │   deploy     │  │   deploy     │  │   deploy         │  │
│  │ - Own state  │  │ - Own state  │  │ - Own state      │  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
└────────────────────────────────────────────────────────────┘
```

### Key Benefits

| Benefit | Description | Impact |
|---------|-------------|--------|
| **Lazy Loading** | Load modules on-demand, not upfront | 80% faster initial load |
| **Independent Deployment** | Deploy Chat UI without redeploying Admin | 5x more frequent releases |
| **Small Parent Bundle** | Parent shell is <500KB | Sub-second load time |
| **Resource Sharing** | Shared auth, navigation, design system | Consistent UX |
| **Technology Flexibility** | Use React for Chat, Vue for Analytics | Best tool for the job |

### Implementation: Module Federation

Using **Webpack Module Federation** for runtime code sharing:

**Parent Application (Shell)**:
```javascript
// webpack.config.js (parent)
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'shell',
      remotes: {
        chat: 'chat@https://cdn.cortex.ai/chat/remoteEntry.js',
        admin: 'admin@https://cdn.cortex.ai/admin/remoteEntry.js',
        analytics: 'analytics@https://cdn.cortex.ai/analytics/remoteEntry.js',
      },
      shared: {
        react: { singleton: true, eager: true },
        'react-dom': { singleton: true, eager: true },
        '@cortex/design-system': { singleton: true },
      },
    }),
  ],
};
```

**Child Module (Chat UI)**:
```javascript
// webpack.config.js (chat module)
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'chat',
      filename: 'remoteEntry.js',
      exposes: {
        './ChatApp': './src/ChatApp',
      },
      shared: {
        react: { singleton: true },
        'react-dom': { singleton: true },
        '@cortex/design-system': { singleton: true },
      },
    }),
  ],
};
```

**Parent Loads Module Dynamically**:
```typescript
// Parent shell
import React, { lazy, Suspense } from 'react';

const ChatApp = lazy(() => import('chat/ChatApp'));

function App() {
  return (
    <Suspense fallback={<Loading />}>
      <ChatApp />
    </Suspense>
  );
}
```

### Module Responsibilities

#### Parent Application (Shell)
- **Authentication**: JWT token management, refresh logic
- **Navigation**: React Router / Next.js routing
- **Design System**: Shared UI components (buttons, forms, modals)
- **Theme Provider**: Dark mode, branding, colors
- **Error Boundary**: Global error handling
- **Analytics**: Page view tracking, user events

**Bundle Size**: ~300KB (gzipped)

#### Chat Module (Child)
- **Conversation UI**: Message list, input box, streaming responses
- **Agent Selection**: Choose AI assistant
- **Context Management**: File uploads, web searches
- **Markdown Rendering**: Code blocks, math, tables

**Bundle Size**: ~800KB (gzipped)
**Loaded**: On-demand when user navigates to `/chat`

#### Admin Portal (Child)
- **User Management**: CRUD for users, roles, permissions
- **Organization Setup**: Org hierarchy, project creation
- **Billing Dashboard**: Usage reports, invoices
- **System Configuration**: Feature flags, quotas

**Bundle Size**: ~500KB (gzipped)
**Loaded**: On-demand when user navigates to `/admin`

#### Analytics Dashboard (Child)
- **Usage Metrics**: API calls, LLM tokens, costs
- **Performance Charts**: Latency, throughput, errors
- **Cost Attribution**: Per-tenant, per-project breakdowns
- **Query Builder**: Custom analytics queries

**Bundle Size**: ~1.2MB (gzipped, includes charting libraries)
**Loaded**: On-demand when user navigates to `/analytics`

### Resource Sharing Control

**Parent Controls**:
- Which versions of shared libraries to use
- Theme and branding enforcement
- Authentication state propagation
- Navigation and routing rules

**Example**: Parent enforces React 18.2.0 across all modules to prevent version conflicts.

### Independent Development

**Team Autonomy**:
```bash
# Chat team deploys independently
cd apps/chat
npm run build
aws s3 sync dist/ s3://cdn.cortex.ai/chat/

# Admin team deploys independently (no coordination needed)
cd apps/admin
npm run build
aws s3 sync dist/ s3://cdn.cortex.ai/admin/
```

**Zero downtime**: Parent shell continues serving while child modules update.

### Performance Optimizations

**Code Splitting**:
```typescript
// Split large dependencies per route
const ChatApp = lazy(() => import(/* webpackChunkName: "chat" */ 'chat/ChatApp'));
const AdminApp = lazy(() => import(/* webpackChunkName: "admin" */ 'admin/AdminApp'));
```

**CDN Caching**:
```nginx
# Cache static assets aggressively
location ~* \.(js|css|png|jpg|jpeg|gif|svg|woff2)$ {
  expires 1y;
  add_header Cache-Control "public, immutable";
}
```

**Bundle Analysis**:
```bash
# Identify large dependencies
npx webpack-bundle-analyzer stats.json
```

### API Integration

**All modules use the same API Gateway**:

```typescript
// Shared API client (provided by parent)
import { apiClient } from '@cortex/api-client';

// Chat module uses API client
const messages = await apiClient.get('/api/v1/conversations/123/messages');

// Admin module uses same client (shared auth token)
const users = await apiClient.get('/api/v1/users');
```

**Benefits**:
- Single authentication flow
- Centralized error handling
- Consistent retry logic
- Unified request/response transformations

### Deployment Strategy

**Progressive Rollout**:
1. Deploy new version to CDN (e.g., chat v2.0.0)
2. Update parent config to point to new version
3. Monitor error rates, performance metrics
4. Rollback by reverting parent config (30 seconds)

**Blue-Green Deployment**:
```javascript
// Parent can A/B test module versions
const chatVersion = userIsInExperiment('chat-v2') ? 'v2.0.0' : 'v1.9.0';
const ChatApp = lazy(() => import(`chat@${chatVersion}/ChatApp`));
```

### Comparison: Monolith vs Micro Frontends

| Metric | Monolithic SPA | Micro Frontends | Improvement |
|--------|---------------|-----------------|-------------|
| **Initial Bundle** | 5-10MB | 300KB (shell only) | **95% smaller** |
| **Time to Interactive** | 8-12 seconds | 1-2 seconds | **85% faster** |
| **Deployment Frequency** | 1-2x/week | 10-20x/week | **10x more frequent** |
| **Team Coordination** | High (merge conflicts) | Low (independent) | **80% less overhead** |
| **Rollback Time** | 15-30 minutes | 30 seconds | **30x faster** |

---

## Deployment Patterns

### Kubernetes Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: cortex-gateway
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/rate-limit: "100"
spec:
  tls:
  - hosts:
    - api.cortex-ai.com
    secretName: cortex-tls
  rules:
  - host: api.cortex-ai.com
    http:
      paths:
      # API Gateway
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: cortex-api
            port:
              number: 80

      # MCP Gateway
      - path: /mcp
        pathType: Prefix
        backend:
          service:
            name: cortex-mcp
            port:
              number: 80
```

---

**Next**: [Control Plane](./02-control-plane.md) | [Back to Overview](./00-convergence-overview.md)
