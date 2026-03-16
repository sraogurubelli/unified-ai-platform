# AI MCP Gateway

## Overview

The **MCP (Model Context Protocol) Gateway** handles AI-native interactions, enabling agents to discover and invoke tools with proper authentication, context injection, and permission enforcement.

```
┌──────────────────────────────────────────────────────┐
│         Load Balancer / CDN                          │
│    (CloudFlare, AWS ALB, NGINX, HAProxy)             │
└────────────────────┬─────────────────────────────────┘
                     ▼
┌────────────────────────────────────────────────────────┐
│   MCP Gateway                                          │
│   (Model Context Protocol)                             │
│                                                        │
│  • Tool discovery                                      │
│  • Agent protocol                                      │
│  • Context management (tenant injection)               │
│  • Tool permissions (RBAC)                             │
│  • Streaming tool responses                            │
│  • Observability (metrics, logs, traces)               │
└────────────────────┬───────────────────────────────────┘
                     ▼
┌────────────────────────────────────────────────────────┐
│         MCP Server(s) (Tool Registry)                  │
└────────────────────┬───────────────────────────────────┘
                     ▼
┌────────────────────────────────────────────────────────┐
│         Backend Services (AI Platform)                 │
│  • Vector Search (Qdrant)                             │
│  • Knowledge Graph (Neo4j)                             │
│  • LLM Gateway                                         │
│  • Agent Orchestration                                 │
└────────────────────────────────────────────────────────┘
```

---

## Use Cases

**AI-Native Operations**:
- Agent discovers available tools
- Agent invokes tools (search, database, API calls)
- Multi-agent handoff and coordination
- Real-time streaming from tools
- Tool execution with tenant isolation
- Permission-based tool access

---

## Model Context Protocol (MCP)

MCP is an open protocol for AI agents to discover and invoke tools. It standardizes:
- **Tool discovery**: What tools are available?
- **Tool schemas**: What parameters does a tool accept?
- **Tool invocation**: How to call a tool?
- **Context management**: How to pass tenant/user context?

### Architecture Flow

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

---

## MCP Server Implementation

### Basic MCP Server

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

### AI-Specific Tools

```python
class AIPlatformTools:
    """AI-specific tools for MCP server."""

    @staticmethod
    def register_ai_tools(mcp_server: MCPServer):
        """Register AI platform tools."""

        # Semantic search tool
        @mcp_server.tool(
            name="semantic_search",
            description="Semantic search using vector embeddings",
            parameters={
                "query": {"type": "string"},
                "collection": {"type": "string", "default": "documents"},
                "top_k": {"type": "integer", "default": 5},
                "similarity_threshold": {"type": "number", "default": 0.7}
            }
        )
        async def semantic_search(
            query: str,
            collection: str,
            top_k: int,
            similarity_threshold: float,
            context: Context
        ):
            """Perform semantic search in Qdrant."""

            tenant_id = context.get("tenant_id")

            # Generate query embedding
            embedding = await embedding_service.embed(query)

            # Search Qdrant
            results = await qdrant_client.search(
                collection_name=f"{collection}_{tenant_id}",
                query_vector=embedding,
                limit=top_k,
                score_threshold=similarity_threshold
            )

            return {
                "results": [
                    {
                        "id": hit.id,
                        "content": hit.payload.get("text"),
                        "score": hit.score,
                        "metadata": hit.payload.get("metadata")
                    }
                    for hit in results
                ]
            }

        # Graph RAG tool
        @mcp_server.tool(
            name="graph_rag_search",
            description="Hybrid search using vector + knowledge graph",
            parameters={
                "query": {"type": "string"},
                "max_hops": {"type": "integer", "default": 2},
                "top_k": {"type": "integer", "default": 5}
            }
        )
        async def graph_rag_search(
            query: str,
            max_hops: int,
            top_k: int,
            context: Context
        ):
            """Perform Graph RAG retrieval."""

            tenant_id = context.get("tenant_id")

            # 1. Vector search
            vector_results = await qdrant_client.search(
                collection_name=f"embeddings_{tenant_id}",
                query_vector=await embedding_service.embed(query),
                limit=10
            )

            # 2. Extract entities
            entities = []
            for doc in vector_results:
                doc_entities = doc.payload.get("entities", [])
                entities.extend(doc_entities)

            # 3. Graph traversal
            graph_results = []
            for entity_name in set(entities):
                related = await neo4j_client.find_related_entities(
                    tenant_id=tenant_id,
                    entity_name=entity_name,
                    max_depth=max_hops
                )
                graph_results.extend(related)

            # 4. Combine using RRF
            combined = reciprocal_rank_fusion(
                vector_results,
                graph_results,
                k=60
            )

            return {
                "results": combined[:top_k],
                "vector_count": len(vector_results),
                "graph_count": len(graph_results)
            }

        # LLM completion tool
        @mcp_server.tool(
            name="llm_complete",
            description="Generate LLM completion with semantic caching",
            parameters={
                "prompt": {"type": "string"},
                "model": {"type": "string", "default": "gpt-4o"},
                "max_tokens": {"type": "integer", "default": 1000}
            }
        )
        async def llm_complete(
            prompt: str,
            model: str,
            max_tokens: int,
            context: Context
        ):
            """LLM completion with semantic caching."""

            tenant_id = context.get("tenant_id")

            # Check semantic cache
            cached_response = await semantic_cache.get(prompt, tenant_id)
            if cached_response:
                return {
                    "response": cached_response,
                    "cached": True,
                    "cost_saved": 0.015  # Example cost
                }

            # Call LLM
            response = await llm_gateway.complete(
                prompt=prompt,
                model=model,
                max_tokens=max_tokens
            )

            # Cache response
            await semantic_cache.set(prompt, response, tenant_id)

            return {
                "response": response,
                "cached": False,
                "tokens": response.usage.total_tokens
            }
```

---

## MCP Gateway Middleware

### Authentication & Context Injection

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
        await self._check_rate_limit(tenant_id, request.tool_name)

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

    async def _check_rate_limit(self, tenant_id: str, tool_name: str):
        """Check tool-specific rate limits."""

        # Different tools may have different limits
        limits = {
            "semantic_search": "100/minute",
            "llm_complete": "50/minute",
            "query_database": "200/minute",
        }

        limit = limits.get(tool_name, "100/minute")
        allowed = await rate_limiter.check_limit(
            tenant_id=tenant_id,
            resource=f"tool:{tool_name}",
            limit=limit
        )

        if not allowed:
            raise RateLimitExceeded(f"Rate limit exceeded for {tool_name}")

    async def _log_tool_usage(
        self,
        tenant_id: str,
        tool_name: str,
        duration_ms: float
    ):
        """Log tool usage for analytics."""

        await analytics.log_event({
            "event_type": "tool_invocation",
            "tenant_id": tenant_id,
            "tool_name": tool_name,
            "duration_ms": duration_ms,
            "timestamp": datetime.utcnow().isoformat()
        })
```

---

## Agent Integration

### LangChain Agent with MCP Tools

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

### Multi-Agent Coordination

```python
class MultiAgentCoordinator:
    """Coordinate multiple agents via MCP Gateway."""

    async def coordinate(self, task: str):
        """
        Coordinate multiple agents to complete a task.

        Example: "Analyze customer sentiment and generate report"
        1. Data Agent: Fetch customer data (query_database tool)
        2. Analysis Agent: Analyze sentiment (llm_complete tool)
        3. Report Agent: Generate report (llm_complete + semantic_search)
        """

        # Agent 1: Data retrieval
        data_agent = CortexAgent(
            mcp_gateway_url=MCP_GATEWAY_URL,
            auth_token=self.auth_token
        )

        customer_data = await data_agent.run(
            "Fetch all customer feedback from last 30 days"
        )

        # Agent 2: Sentiment analysis
        analysis_agent = CortexAgent(
            mcp_gateway_url=MCP_GATEWAY_URL,
            auth_token=self.auth_token
        )

        sentiment_results = await analysis_agent.run(
            f"Analyze sentiment of this data: {customer_data}"
        )

        # Agent 3: Report generation
        report_agent = CortexAgent(
            mcp_gateway_url=MCP_GATEWAY_URL,
            auth_token=self.auth_token
        )

        report = await report_agent.run(
            f"Generate executive summary report based on: {sentiment_results}"
        )

        return report
```

---

## Tool Discovery & Schema

### Tool Discovery API

```python
class MCPDiscoveryService:
    """Service for tool discovery."""

    async def list_tools(self, context: Context) -> list[Tool]:
        """
        List all tools available to the current user.

        Filters tools based on RBAC permissions.
        """

        tenant_id = context.get("tenant_id")
        principal_id = context.get("principal_id")

        # Get all registered tools
        all_tools = await self.mcp_server.get_all_tools()

        # Filter by permissions
        accessible_tools = []
        for tool in all_tools:
            has_permission = await rbac.check_permission(
                principal_id=principal_id,
                resource_type="tool",
                resource_id=tool.name,
                permission=Permission.TOOL_EXECUTE
            )

            if has_permission:
                accessible_tools.append(tool)

        return accessible_tools

    async def get_tool_schema(self, tool_name: str) -> dict:
        """Get JSON schema for tool parameters."""

        tool = await self.mcp_server.get_tool(tool_name)

        return {
            "name": tool.name,
            "description": tool.description,
            "parameters": {
                "type": "object",
                "properties": tool.parameters,
                "required": [
                    param for param, schema in tool.parameters.items()
                    if schema.get("required", False)
                ]
            }
        }
```

---

## Security & Validation

### Input Validation

```python
import jsonschema

async def invoke_tool(tool_name: str, parameters: dict):
    """Invoke tool with schema validation."""

    # Get tool schema
    schema = await mcp_server.get_tool_schema(tool_name)

    # Validate parameters against schema
    try:
        jsonschema.validate(parameters, schema["parameters"])
    except jsonschema.ValidationError as e:
        raise ValueError(f"Invalid parameters: {e.message}")

    # Execute tool
    result = await mcp_server.invoke_tool(tool_name, parameters)
    return result
```

### SQL Injection Prevention (for database tools)

```python
def validate_sql_query(sql: str) -> bool:
    """Validate SQL query for safety."""

    # Block dangerous operations
    forbidden_keywords = [
        "DROP", "DELETE", "UPDATE", "INSERT",
        "ALTER", "CREATE", "TRUNCATE", "GRANT", "REVOKE"
    ]

    sql_upper = sql.upper()
    for keyword in forbidden_keywords:
        if keyword in sql_upper:
            raise ValueError(f"Forbidden SQL keyword: {keyword}")

    # Only allow SELECT
    if not sql_upper.strip().startswith("SELECT"):
        raise ValueError("Only SELECT queries are allowed")

    return True
```

### Cypher Injection Prevention (for graph tools)

```python
def validate_cypher_query(cypher: str) -> bool:
    """Validate Cypher query for safety."""

    # Block write operations
    forbidden_keywords = [
        "CREATE", "DELETE", "DETACH DELETE", "MERGE",
        "SET", "REMOVE", "DROP"
    ]

    cypher_upper = cypher.upper()
    for keyword in forbidden_keywords:
        if keyword in cypher_upper:
            raise ValueError(f"Forbidden Cypher keyword: {keyword}")

    # Only allow MATCH, RETURN
    if not cypher_upper.strip().startswith("MATCH"):
        raise ValueError("Only MATCH queries are allowed")

    return True
```

---

## Streaming Responses

### Server-Sent Events (SSE) for Streaming

```python
from fastapi import Response
from fastapi.responses import StreamingResponse

@app.post("/mcp/tools/{tool_name}/stream")
async def invoke_tool_stream(
    tool_name: str,
    parameters: dict,
    request: Request
):
    """Stream tool results using SSE."""

    async def event_stream():
        """Generate SSE events."""

        # Invoke tool with streaming
        async for chunk in mcp_server.invoke_tool_stream(
            name=tool_name,
            parameters=parameters,
            context=get_context(request)
        ):
            # Send chunk as SSE
            yield f"data: {json.dumps(chunk)}\n\n"

    return StreamingResponse(
        event_stream(),
        media_type="text/event-stream"
    )
```

### WebSocket for Real-Time Tool Execution

```python
from fastapi import WebSocket

@app.websocket("/mcp/ws")
async def mcp_websocket(websocket: WebSocket):
    """WebSocket endpoint for real-time tool execution."""

    await websocket.accept()

    try:
        while True:
            # Receive tool invocation request
            data = await websocket.receive_json()

            tool_name = data["tool_name"]
            parameters = data["parameters"]

            # Execute tool
            result = await mcp_server.invoke_tool(
                name=tool_name,
                parameters=parameters,
                context=get_context_from_websocket(websocket)
            )

            # Send result
            await websocket.send_json({
                "tool_name": tool_name,
                "result": result
            })

    except WebSocketDisconnect:
        logger.info("WebSocket disconnected")
```

---

## Observability

### Metrics

```python
from prometheus_client import Counter, Histogram

# Tool invocation metrics
mcp_tool_invocations_total = Counter(
    'mcp_tool_invocations_total',
    'Total MCP tool invocations',
    ['tenant_id', 'tool_name', 'status']
)

mcp_tool_latency = Histogram(
    'mcp_tool_latency_seconds',
    'MCP tool latency',
    ['tenant_id', 'tool_name']
)

# Usage
async def invoke_tool_with_metrics(tool_name: str, context: Context):
    with mcp_tool_latency.labels(
        tenant_id=context.get("tenant_id"),
        tool_name=tool_name
    ).time():
        try:
            result = await mcp_server.invoke_tool(tool_name, context)

            mcp_tool_invocations_total.labels(
                tenant_id=context.get("tenant_id"),
                tool_name=tool_name,
                status="success"
            ).inc()

            return result

        except Exception as e:
            mcp_tool_invocations_total.labels(
                tenant_id=context.get("tenant_id"),
                tool_name=tool_name,
                status="error"
            ).inc()
            raise
```

### Distributed Tracing

```python
from opentelemetry import trace

tracer = trace.get_tracer(__name__)

async def invoke_tool_with_tracing(
    tool_name: str,
    parameters: dict,
    context: Context
):
    """Invoke tool with distributed tracing."""

    with tracer.start_as_current_span("mcp_tool_invocation") as span:
        span.set_attribute("tool_name", tool_name)
        span.set_attribute("tenant_id", context.get("tenant_id"))
        span.set_attribute("parameter_count", len(parameters))

        result = await mcp_server.invoke_tool(
            name=tool_name,
            parameters=parameters,
            context=context
        )

        span.set_attribute("result_size", len(str(result)))

        return result
```

---

## Deployment

### Docker

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY . .

CMD ["uvicorn", "cortex.mcp.gateway:app", "--host", "0.0.0.0", "--port", "8001"]
```

### Kubernetes

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mcp-gateway
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
        - name: MCP_SERVER_URL
          value: "http://mcp-server:8002"
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: auth-secrets
              key: jwt-secret
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
```

---

## Performance Optimization

### Tool Result Caching

```python
from functools import lru_cache

class ToolResultCache:
    """Cache tool results for identical parameters."""

    async def get_cached_result(
        self,
        tool_name: str,
        parameters: dict,
        tenant_id: str
    ) -> Optional[dict]:
        """Get cached tool result."""

        cache_key = f"tool:{tool_name}:{tenant_id}:{hash(json.dumps(parameters))}"

        cached = await redis.get(cache_key)
        if cached:
            return json.loads(cached)

        return None

    async def cache_result(
        self,
        tool_name: str,
        parameters: dict,
        tenant_id: str,
        result: dict,
        ttl: int = 300
    ):
        """Cache tool result."""

        cache_key = f"tool:{tool_name}:{tenant_id}:{hash(json.dumps(parameters))}"

        await redis.setex(
            cache_key,
            ttl,
            json.dumps(result)
        )
```

### Connection Pooling

```python
# Connection pool for backend services
class MCPBackendPool:
    """Connection pool for MCP backend services."""

    def __init__(self):
        self.qdrant_pool = QdrantClientPool(size=20)
        self.neo4j_pool = Neo4jDriverPool(size=20)
        self.postgres_pool = PostgresPool(size=50)

    async def get_qdrant_client(self):
        """Get Qdrant client from pool."""
        return await self.qdrant_pool.acquire()

    async def get_neo4j_driver(self):
        """Get Neo4j driver from pool."""
        return await self.neo4j_pool.acquire()
```

---

**Next**: [Traditional API Gateway](./01a-traditional-api-gateway.md) | [Integration](./01c-gateway-integration.md) | [Back to Overview](../00-parallel-stacks-overview.md)
