# Traditional API Gateway

## Overview

The **Traditional API Gateway** handles SaaS-style interactions using standard HTTP/REST/GraphQL patterns. It provides:

- User authentication (JWT, OAuth)
- CRUD operations on resources
- RESTful or GraphQL APIs
- Request/response transformation
- Rate limiting per tenant
- CORS handling
- Input validation

```
┌──────────────────────────────────────────────────────┐
│         Load Balancer / CDN                          │
│    (CloudFlare, AWS ALB, NGINX, HAProxy)             │
└────────────────────┬─────────────────────────────────┘
                     ▼
┌────────────────────────────────────────────────────────┐
│   Traditional API Gateway                              │
│   (HTTP/REST/GraphQL)                                  │
│                                                        │
│  • JWT Authentication                                  │
│  • Rate limiting (per tenant)                          │
│  • Request routing                                     │
│  • Response transformation                             │
│  • CORS, Input validation                             │
│  • Observability (metrics, logs, traces)              │
└────────────────────┬───────────────────────────────────┘
                     ▼
┌────────────────────────────────────────────────────────┐
│           Control Plane (IAM, RBAC)                    │
└────────────────────┬───────────────────────────────────┘
                     ▼
┌────────────────────────────────────────────────────────┐
│         Service Layer (Business Logic)                 │
└────────────────────────────────────────────────────────┘
```

---

## Use Cases

**Traditional SaaS Operations**:
- User signup/login
- Create project, upload document
- List conversations, get analytics
- Manage billing and settings
- CRUD operations (Create, Read, Update, Delete)
- Resource management (projects, organizations, users)

---

## Technology Options

| Technology | Best For | Pros | Cons |
|------------|----------|------|------|
| **FastAPI** | Python-native apps | Fast, async, OpenAPI auto-gen | Limited enterprise features |
| **Kong** | Enterprise, K8s | Plugins, scalability, community | Complex configuration |
| **AWS API Gateway** | AWS deployments | Managed, integrated, scalable | Vendor lock-in, cost |
| **NGINX** | On-premise, custom | Flexible, performant, proven | Manual configuration |
| **Traefik** | Kubernetes | Auto-discovery, K8s-native | Learning curve |
| **Envoy** | Service mesh | Advanced routing, observability | Complexity |

**Recommendation**:
- **Development/MVP**: FastAPI (rapid iteration, OpenAPI docs)
- **Production (Cloud)**: Kong or AWS API Gateway (managed, scalable)
- **Production (On-Premise)**: Kong or NGINX (self-hosted, flexible)

---

## FastAPI Implementation (Reference)

### Application Setup

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

### Example Routes

```python
# cortex/api/routes/projects.py

from fastapi import APIRouter, Depends, HTTPException
from pydantic import BaseModel

router = APIRouter(prefix="/api/v1/projects", tags=["projects"])

class CreateProjectRequest(BaseModel):
    name: str
    description: str

class ProjectResponse(BaseModel):
    id: str
    name: str
    description: str
    created_at: str

@router.post("/", response_model=ProjectResponse)
@limiter.limit("100/minute")
async def create_project(
    request: Request,
    body: CreateProjectRequest
):
    """Create new project."""

    tenant_id = request.state.tenant_id
    principal_id = request.state.principal_id

    # Check permissions
    if not await rbac.has_permission(principal_id, "project.create"):
        raise HTTPException(status_code=403, detail="Permission denied")

    # Create project
    project = await db.create_project(
        tenant_id=tenant_id,
        name=body.name,
        description=body.description,
        created_by=principal_id
    )

    return ProjectResponse(
        id=project.uid,
        name=project.name,
        description=project.description,
        created_at=project.created_at.isoformat()
    )

@router.get("/{project_id}", response_model=ProjectResponse)
async def get_project(request: Request, project_id: str):
    """Get project by ID."""

    tenant_id = request.state.tenant_id

    # Fetch with tenant isolation
    project = await db.query(Project).filter_by(
        uid=project_id,
        tenant_id=tenant_id
    ).first()

    if not project:
        raise HTTPException(status_code=404, detail="Project not found")

    return ProjectResponse(
        id=project.uid,
        name=project.name,
        description=project.description,
        created_at=project.created_at.isoformat()
    )

@router.get("/", response_model=list[ProjectResponse])
async def list_projects(request: Request, skip: int = 0, limit: int = 100):
    """List projects for tenant."""

    tenant_id = request.state.tenant_id

    projects = await db.query(Project).filter_by(
        tenant_id=tenant_id
    ).offset(skip).limit(limit).all()

    return [
        ProjectResponse(
            id=p.uid,
            name=p.name,
            description=p.description,
            created_at=p.created_at.isoformat()
        )
        for p in projects
    ]
```

---

## Kong Implementation (Enterprise)

### Configuration

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

---

## Request Flow

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

---

## Rate Limiting Strategies

### Per-Tenant Rate Limiting

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

### Tiered Rate Limits

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

### Redis-Based Sliding Window

```python
import redis.asyncio as redis

class RateLimiter:
    """Sliding window rate limiter using Redis."""

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client

    async def check_limit(
        self,
        tenant_id: str,
        resource: str,
        limit: int,
        window: int = 60
    ) -> bool:
        """Check if request is within rate limit."""

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

# Usage
rate_limiter = RateLimiter(redis_client)

@app.post("/api/v1/chat")
async def chat(request: Request, ...):
    allowed = await rate_limiter.check_limit(
        tenant_id=request.state.tenant_id,
        resource="chat",
        limit=100,
        window=60
    )

    if not allowed:
        raise HTTPException(status_code=429, detail="Rate limit exceeded")

    ...
```

---

## Security Best Practices

### 1. Input Validation

```python
from pydantic import BaseModel, validator

# Pydantic automatically validates input
class ChatRequest(BaseModel):
    message: str
    max_tokens: int = 1000

    @validator('message')
    def validate_message(cls, v):
        if len(v) > 10000:
            raise ValueError("Message too long")
        return v

    @validator('max_tokens')
    def validate_max_tokens(cls, v):
        if v < 1 or v > 4000:
            raise ValueError("max_tokens must be between 1 and 4000")
        return v

@app.post("/api/v1/chat")
async def chat(request: ChatRequest):  # Pydantic validates
    # request.message already validated by Pydantic
    ...
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

### 5. HTTPS Enforcement

```python
from fastapi.middleware.httpsredirect import HTTPSRedirectMiddleware

# Redirect HTTP to HTTPS
app.add_middleware(HTTPSRedirectMiddleware)
```

---

## Performance Optimization

### 1. Connection Pooling

```python
# Database connection pool
from sqlalchemy.ext.asyncio import create_async_engine

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

# Initialize cache
FastAPICache.init(RedisBackend(redis), prefix="api-cache:")

@app.get("/api/v1/projects/{project_id}")
@cache(expire=300)  # Cache for 5 minutes
async def get_project(project_id: str):
    ...
```

### 3. Gzip Compression

```python
from fastapi.middleware.gzip import GZipMiddleware

app.add_middleware(GZipMiddleware, minimum_size=1000)
```

### 4. Async Database Queries

```python
# Async database operations
from sqlalchemy.ext.asyncio import AsyncSession

async def get_projects(db: AsyncSession, tenant_id: str):
    result = await db.execute(
        select(Project).filter_by(tenant_id=tenant_id)
    )
    return result.scalars().all()
```

---

## Observability

### Metrics (Prometheus)

```python
from prometheus_client import Counter, Histogram

# Request metrics
api_requests_total = Counter(
    'api_requests_total',
    'Total API requests',
    ['tenant_id', 'endpoint', 'status']
)

api_latency = Histogram(
    'api_latency_seconds',
    'API request latency',
    ['tenant_id', 'endpoint']
)

@app.middleware("http")
async def metrics_middleware(request: Request, call_next):
    """Collect request metrics."""

    with api_latency.labels(
        tenant_id=getattr(request.state, "tenant_id", "anonymous"),
        endpoint=request.url.path
    ).time():
        response = await call_next(request)

    api_requests_total.labels(
        tenant_id=getattr(request.state, "tenant_id", "anonymous"),
        endpoint=request.url.path,
        status=str(response.status_code)
    ).inc()

    return response
```

### Logging (Structured)

```python
import structlog

logger = structlog.get_logger()

@app.post("/api/v1/projects")
async def create_project(request: Request, ...):
    logger.info(
        "project_created",
        tenant_id=request.state.tenant_id,
        principal_id=request.state.principal_id,
        project_name=body.name
    )
```

### Distributed Tracing (OpenTelemetry)

```python
from opentelemetry import trace
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor

# Initialize tracing
FastAPIInstrumentor.instrument_app(app)

@app.post("/api/v1/chat")
async def chat(request: Request, ...):
    tracer = trace.get_tracer(__name__)

    with tracer.start_as_current_span("chat_request") as span:
        span.set_attribute("tenant_id", request.state.tenant_id)
        span.set_attribute("message_length", len(body.message))

        # Perform chat
        response = await chat_service.complete(body.message)

        span.set_attribute("response_length", len(response))

    return {"response": response}
```

---

## GraphQL Support (Optional)

```python
from strawberry.fastapi import GraphQLRouter
import strawberry

@strawberry.type
class Project:
    id: str
    name: str
    description: str

@strawberry.type
class Query:
    @strawberry.field
    async def projects(self, info) -> list[Project]:
        tenant_id = info.context["request"].state.tenant_id

        projects = await db.query(Project).filter_by(
            tenant_id=tenant_id
        ).all()

        return [
            Project(id=p.uid, name=p.name, description=p.description)
            for p in projects
        ]

schema = strawberry.Schema(query=Query)
graphql_app = GraphQLRouter(schema)

app.include_router(graphql_app, prefix="/graphql")
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

CMD ["uvicorn", "cortex.api.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Kubernetes

```yaml
# deployment.yaml
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
        env:
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
```

---

**Next**: [AI MCP Gateway](./01b-ai-mcp-gateway.md) | [Integration](./01c-gateway-integration.md) | [Back to Overview](../00-parallel-stacks-overview.md)
