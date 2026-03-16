# Gateway Layer Integration

## Overview

The **Gateway Integration** layer connects the **Traditional API Gateway** (REST/GraphQL) with the **AI MCP Gateway** (Model Context Protocol), enabling seamless interoperability while maintaining clear separation of concerns.

```
┌──────────────────────────────────────────────────────────────┐
│         Load Balancer / CDN                                   │
│    (CloudFlare, AWS ALB, NGINX, HAProxy)                      │
└────────────────────┬─────────────────────────────────────────┘
                     │
        ┌────────────┴────────────┐
        ▼                         ▼
┌────────────────────┐   ┌────────────────────┐
│ Traditional API    │   │ AI MCP Gateway     │
│ Gateway            │   │                    │
│ (REST/GraphQL)     │   │ (Tool Protocol)    │
└────────────────────┘   └────────────────────┘
        │                         │
        └────────────┬────────────┘
                     │
        ┌────────────┴────────────┐
        │   SHARED SERVICES       │
        │   • Authentication      │
        │   • Rate Limiting       │
        │   • Observability       │
        │   • Tenant Context      │
        └─────────────────────────┘
```

---

## Shared Services

### 1. Unified Authentication

Both gateways use the **same JWT tokens** and authentication system.

```python
# Shared authentication module
class UnifiedAuthService:
    """Unified authentication for both API and MCP gateways."""

    def __init__(self, jwt_secret: str):
        self.jwt_secret = jwt_secret

    async def validate_token(self, token: str) -> dict:
        """
        Validate JWT token.

        Returns:
            {
                "tenant_id": "acct_123",
                "principal_id": "user_456",
                "roles": ["admin"],
                "permissions": ["project.create", "tool.execute"]
            }
        """
        try:
            payload = jwt.decode(token, self.jwt_secret, algorithms=["HS256"])

            # Verify expiration
            if datetime.fromtimestamp(payload["exp"]) < datetime.utcnow():
                raise AuthenticationError("Token expired")

            return payload

        except jwt.InvalidTokenError:
            raise AuthenticationError("Invalid token")

    async def create_token(
        self,
        tenant_id: str,
        principal_id: str,
        roles: list[str],
        ttl: int = 3600
    ) -> str:
        """Create JWT token."""

        payload = {
            "tenant_id": tenant_id,
            "principal_id": principal_id,
            "roles": roles,
            "exp": datetime.utcnow() + timedelta(seconds=ttl),
            "iat": datetime.utcnow()
        }

        return jwt.encode(payload, self.jwt_secret, algorithm="HS256")

# Usage in API Gateway
@app.middleware("http")
async def api_auth_middleware(request: Request, call_next):
    """API Gateway authentication."""

    token = request.headers.get("Authorization", "").replace("Bearer ", "")
    payload = await auth_service.validate_token(token)

    request.state.tenant_id = payload["tenant_id"]
    request.state.principal_id = payload["principal_id"]

    return await call_next(request)

# Usage in MCP Gateway
async def mcp_auth_middleware(request: MCPRequest) -> Context:
    """MCP Gateway authentication."""

    token = request.headers.get("Authorization", "").replace("Bearer ", "")
    payload = await auth_service.validate_token(token)

    return Context({
        "tenant_id": payload["tenant_id"],
        "principal_id": payload["principal_id"]
    })
```

### 2. Unified Rate Limiting

Both gateways share the same rate limiting infrastructure (Redis-based).

```python
class UnifiedRateLimiter:
    """Unified rate limiting for API and MCP gateways."""

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client

    async def check_limit(
        self,
        tenant_id: str,
        resource: str,
        limit: int,
        window: int = 60
    ) -> bool:
        """
        Check rate limit (sliding window).

        Args:
            tenant_id: Tenant identifier
            resource: Resource being accessed (e.g., "api:/chat", "tool:semantic_search")
            limit: Max requests per window
            window: Time window in seconds

        Returns:
            True if within limit, False otherwise
        """

        key = f"ratelimit:{tenant_id}:{resource}"
        now = time.time()
        window_start = now - window

        # Remove old entries
        await self.redis.zremrangebyscore(key, 0, window_start)

        # Count current requests
        count = await self.redis.zcard(key)

        if count >= limit:
            return False

        # Add current request
        await self.redis.zadd(key, {str(uuid.uuid4()): now})
        await self.redis.expire(key, window)

        return True

# Usage in API Gateway
@app.post("/api/v1/chat")
async def api_chat(request: Request, ...):
    """API endpoint with rate limiting."""

    allowed = await rate_limiter.check_limit(
        tenant_id=request.state.tenant_id,
        resource="api:/chat",
        limit=100,
        window=60
    )

    if not allowed:
        raise HTTPException(status_code=429, detail="Rate limit exceeded")

    ...

# Usage in MCP Gateway
async def mcp_invoke_tool(tool_name: str, context: Context):
    """MCP tool invocation with rate limiting."""

    allowed = await rate_limiter.check_limit(
        tenant_id=context.get("tenant_id"),
        resource=f"tool:{tool_name}",
        limit=50,
        window=60
    )

    if not allowed:
        raise RateLimitExceeded(f"Rate limit exceeded for {tool_name}")

    ...
```

### 3. Unified Observability

Both gateways emit metrics to the same Prometheus instance.

```python
from prometheus_client import Counter, Histogram

# Shared metrics
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

# API Gateway usage
@app.middleware("http")
async def api_metrics_middleware(request: Request, call_next):
    """API Gateway metrics collection."""

    with gateway_latency.labels(
        gateway_type='api',
        tenant_id=request.state.tenant_id,
        endpoint=request.url.path
    ).time():
        response = await call_next(request)

    gateway_requests_total.labels(
        gateway_type='api',
        tenant_id=request.state.tenant_id,
        endpoint=request.url.path,
        status=str(response.status_code)
    ).inc()

    return response

# MCP Gateway usage
async def mcp_metrics_middleware(tool_name: str, context: Context):
    """MCP Gateway metrics collection."""

    with gateway_latency.labels(
        gateway_type='mcp',
        tenant_id=context.get("tenant_id"),
        endpoint=tool_name
    ).time():
        result = await mcp_server.invoke_tool(tool_name, context)

    gateway_requests_total.labels(
        gateway_type='mcp',
        tenant_id=context.get("tenant_id"),
        endpoint=tool_name,
        status='success'
    ).inc()

    return result
```

### 4. Unified Logging

Both gateways use structured logging with the same format.

```python
import structlog

logger = structlog.get_logger()

# API Gateway logging
@app.post("/api/v1/chat")
async def api_chat(request: Request, ...):
    logger.info(
        "api_request",
        gateway_type="api",
        tenant_id=request.state.tenant_id,
        endpoint="/api/v1/chat",
        method="POST"
    )

    ...

    logger.info(
        "api_response",
        gateway_type="api",
        tenant_id=request.state.tenant_id,
        status_code=200,
        duration_ms=123
    )

# MCP Gateway logging
async def mcp_invoke_tool(tool_name: str, context: Context):
    logger.info(
        "mcp_request",
        gateway_type="mcp",
        tenant_id=context.get("tenant_id"),
        tool_name=tool_name
    )

    ...

    logger.info(
        "mcp_response",
        gateway_type="mcp",
        tenant_id=context.get("tenant_id"),
        tool_name=tool_name,
        duration_ms=456
    )
```

---

## Cross-Gateway Communication

### API Gateway → MCP Gateway

**Use Case**: API endpoint triggers MCP tool execution.

```python
# API endpoint that uses MCP tools internally
@app.post("/api/v1/search")
async def api_search(request: Request, body: SearchRequest):
    """API endpoint that uses MCP semantic search tool."""

    tenant_id = request.state.tenant_id

    # Create MCP context from API request
    mcp_context = Context({
        "tenant_id": tenant_id,
        "principal_id": request.state.principal_id,
        "request_id": str(uuid.uuid4())
    })

    # Invoke MCP tool
    mcp_client = MCPClient(
        gateway_url=MCP_GATEWAY_URL,
        auth_token=request.headers.get("Authorization")
    )

    results = await mcp_client.invoke_tool(
        name="semantic_search",
        parameters={
            "query": body.query,
            "top_k": body.limit
        },
        context=mcp_context
    )

    # Transform MCP results to API response
    return {
        "results": results["results"],
        "count": len(results["results"])
    }
```

### MCP Gateway → API Gateway

**Use Case**: MCP tool needs to call traditional API.

```python
# MCP tool that calls traditional API
@mcp_server.tool(
    name="create_project",
    description="Create new project via API",
    parameters={
        "name": {"type": "string"},
        "description": {"type": "string"}
    }
)
async def create_project_tool(
    name: str,
    description: str,
    context: Context
):
    """MCP tool that creates project via API Gateway."""

    tenant_id = context.get("tenant_id")

    # Get auth token from context
    auth_token = context.get("auth_token")

    # Call API Gateway
    async with httpx.AsyncClient() as client:
        response = await client.post(
            f"{API_GATEWAY_URL}/api/v1/projects",
            json={
                "name": name,
                "description": description
            },
            headers={
                "Authorization": f"Bearer {auth_token}",
                "Content-Type": "application/json"
            }
        )

        response.raise_for_status()

        return response.json()
```

---

## Unified RBAC

Both gateways use the same permission system.

```python
class UnifiedRBACService:
    """Unified RBAC for API and MCP gateways."""

    async def check_permission(
        self,
        principal_id: str,
        resource_type: str,
        resource_id: str,
        permission: str
    ) -> bool:
        """
        Check if principal has permission.

        Args:
            principal_id: User/service identifier
            resource_type: Type of resource ("project", "tool", "conversation")
            resource_id: Resource identifier
            permission: Permission to check ("create", "read", "execute")

        Returns:
            True if permission granted
        """

        # Get principal's roles
        roles = await self._get_principal_roles(principal_id)

        # Check each role's permissions
        for role in roles:
            role_permissions = await self._get_role_permissions(role)

            # Check if permission exists
            permission_key = f"{resource_type}.{permission}"
            if permission_key in role_permissions:
                return True

        return False

# API Gateway permission check
@app.post("/api/v1/projects")
async def create_project(request: Request, ...):
    """API endpoint with permission check."""

    has_permission = await rbac.check_permission(
        principal_id=request.state.principal_id,
        resource_type="project",
        resource_id="*",  # Creating new project
        permission="create"
    )

    if not has_permission:
        raise HTTPException(status_code=403, detail="Permission denied")

    ...

# MCP Gateway permission check
async def mcp_invoke_tool(tool_name: str, context: Context):
    """MCP tool invocation with permission check."""

    has_permission = await rbac.check_permission(
        principal_id=context.get("principal_id"),
        resource_type="tool",
        resource_id=tool_name,
        permission="execute"
    )

    if not has_permission:
        raise PermissionDenied(f"No permission to execute {tool_name}")

    ...
```

---

## Load Balancer Configuration

### NGINX Configuration

```nginx
# nginx.conf

upstream api_gateway {
    server api-gateway-1:8000;
    server api-gateway-2:8000;
    server api-gateway-3:8000;
}

upstream mcp_gateway {
    server mcp-gateway-1:8001;
    server mcp-gateway-2:8001;
    server mcp-gateway-3:8001;
}

server {
    listen 443 ssl http2;
    server_name api.cortex-ai.com;

    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;

    # API Gateway routes
    location /api/ {
        proxy_pass http://api_gateway;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # MCP Gateway routes
    location /mcp/ {
        proxy_pass http://mcp_gateway;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket support for MCP streaming
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }

    # Health checks
    location /health {
        access_log off;
        return 200 "healthy\n";
    }
}
```

---

## Gateway Comparison

| Feature | API Gateway | MCP Gateway | Shared |
|---------|-------------|-------------|--------|
| **Protocol** | HTTP/REST/GraphQL | MCP (JSON-RPC-like) | - |
| **Use Case** | CRUD operations | Tool invocation | - |
| **Clients** | Web apps, mobile apps | AI agents | - |
| **Authentication** | JWT, OAuth | JWT | ✅ Same system |
| **Rate Limiting** | Per tenant | Per tenant | ✅ Same Redis |
| **RBAC** | Permission checks | Permission checks | ✅ Same service |
| **Observability** | Prometheus, logs | Prometheus, logs | ✅ Same metrics |
| **Context** | Request headers | MCP Context object | ✅ Same tenant_id |
| **Response** | JSON, HTML, SSE | Structured tool output | - |
| **Caching** | HTTP caching | Tool result caching | ✅ Same Redis |

---

## Unified Deployment

### Docker Compose

```yaml
# docker-compose.yml

version: '3.8'

services:
  # Load Balancer
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

  # API Gateway
  api-gateway:
    image: cortex-ai/api-gateway:latest
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - JWT_SECRET=${JWT_SECRET}
      - REDIS_URL=${REDIS_URL}
    depends_on:
      - redis
      - postgres

  # MCP Gateway
  mcp-gateway:
    image: cortex-ai/mcp-gateway:latest
    environment:
      - MCP_SERVER_URL=${MCP_SERVER_URL}
      - JWT_SECRET=${JWT_SECRET}  # Same as API Gateway
      - REDIS_URL=${REDIS_URL}    # Same as API Gateway
    depends_on:
      - redis
      - mcp-server

  # Shared services
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  postgres:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=cortex
      - POSTGRES_USER=cortex
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
```

### Kubernetes

```yaml
# gateway-deployment.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: gateway-config
data:
  JWT_SECRET: ${JWT_SECRET}
  REDIS_URL: redis://redis:6379

---
# API Gateway Deployment
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
      - name: api-gateway
        image: cortex-ai/api-gateway:latest
        ports:
        - containerPort: 8000
        envFrom:
        - configMapRef:
            name: gateway-config

---
# MCP Gateway Deployment
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
        envFrom:
        - configMapRef:
            name: gateway-config

---
# Ingress (unified)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gateway-ingress
spec:
  rules:
  - host: api.cortex-ai.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-gateway
            port:
              number: 8000
      - path: /mcp
        pathType: Prefix
        backend:
          service:
            name: mcp-gateway
            port:
              number: 8001
```

---

## Monitoring & Alerting

### Prometheus Alerts (Unified)

```yaml
# alerts.yaml

groups:
  - name: gateway_alerts
    interval: 30s
    rules:
      # High latency (both gateways)
      - alert: HighGatewayLatency
        expr: histogram_quantile(0.95, gateway_latency_seconds) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Gateway p95 latency exceeds 1s"
          description: "{{ $labels.gateway_type }} gateway latency is {{ $value }}s"

      # High error rate (both gateways)
      - alert: HighGatewayErrorRate
        expr: |
          rate(gateway_requests_total{status=~"5.."}[5m]) /
          rate(gateway_requests_total[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Gateway error rate exceeds 5%"
          description: "{{ $labels.gateway_type }} gateway error rate is {{ $value }}"

      # Rate limit exceeded frequently
      - alert: FrequentRateLimitExceeded
        expr: rate(rate_limit_exceeded_total[5m]) > 10
        for: 5m
        labels:
          severity: info
        annotations:
          summary: "Frequent rate limit violations"
          description: "Tenant {{ $labels.tenant_id }} hitting rate limits"
```

### Grafana Dashboard (Unified)

```json
{
  "dashboard": {
    "title": "Gateway Overview",
    "panels": [
      {
        "title": "Request Rate (API vs MCP)",
        "targets": [
          {
            "expr": "sum(rate(gateway_requests_total{gateway_type='api'}[5m])) by (tenant_id)"
          },
          {
            "expr": "sum(rate(gateway_requests_total{gateway_type='mcp'}[5m])) by (tenant_id)"
          }
        ]
      },
      {
        "title": "Latency Comparison",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, gateway_latency_seconds{gateway_type='api'})"
          },
          {
            "expr": "histogram_quantile(0.95, gateway_latency_seconds{gateway_type='mcp'})"
          }
        ]
      },
      {
        "title": "Authentication Failures",
        "targets": [
          {
            "expr": "sum(rate(auth_failures_total[5m])) by (gateway_type)"
          }
        ]
      }
    ]
  }
}
```

---

## Security Considerations

### Shared Security Headers

```python
# Security headers middleware (used by both gateways)
@app.middleware("http")
async def security_headers_middleware(request: Request, call_next):
    """Add security headers."""

    response = await call_next(request)

    # HSTS
    response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains"

    # XSS Protection
    response.headers["X-Content-Type-Options"] = "nosniff"
    response.headers["X-Frame-Options"] = "DENY"
    response.headers["X-XSS-Protection"] = "1; mode=block"

    # CSP
    response.headers["Content-Security-Policy"] = "default-src 'self'"

    return response
```

### Mutual TLS (mTLS) Between Gateways

```python
# mTLS configuration for gateway-to-gateway communication
import ssl

ssl_context = ssl.create_default_context(ssl.Purpose.CLIENT_AUTH)
ssl_context.load_cert_chain(
    certfile="/etc/certs/gateway.crt",
    keyfile="/etc/certs/gateway.key"
)
ssl_context.load_verify_locations(cafile="/etc/certs/ca.crt")

# API Gateway → MCP Gateway call
async with httpx.AsyncClient(verify=ssl_context) as client:
    response = await client.post(
        f"{MCP_GATEWAY_URL}/invoke",
        json={"tool": "semantic_search", "params": {...}},
        headers={"Authorization": f"Bearer {token}"}
    )
```

---

**Previous**: [Traditional API Gateway](./01a-traditional-api-gateway.md) | [AI MCP Gateway](./01b-ai-mcp-gateway.md) | [Back to Overview](../00-parallel-stacks-overview.md)
