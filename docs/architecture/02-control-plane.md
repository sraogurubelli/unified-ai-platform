# Control Plane: Unified Management Layer

## Overview

The control plane is the **unified management layer** that governs all services (both SaaS and AI). It provides centralized identity, access control, multi-tenancy, observability, and policy enforcement.

```
┌──────────────────────────────────────────────────────────────┐
│                      Control Plane                            │
├──────────────┬───────────────┬──────────────┬────────────────┤
│ IAM / RBAC   │ Multi-tenancy │ Observability│ Policy Engine  │
│              │               │              │                │
│ - Identity   │ - Accounts    │ - Metrics    │ - Rate limits  │
│ - Auth       │ - Isolation   │ - Logs       │ - Quotas       │
│ - Permissions│ - Hierarchy   │ - Traces     │ - Compliance   │
└──────────────┴───────────────┴──────────────┴────────────────┘
```

**Key Principle**: The control plane is **deployment-agnostic**. It works identically across SaaS, Hybrid, and On-Premise deployments.

## Identity & Access Management (IAM)

### Concepts

- **Principal**: An authenticated entity (user or service account)
- **Resource**: An object that can be accessed (project, document, agent)
- **Permission**: An action on a resource (create, read, update, delete)
- **Role**: A collection of permissions
- **Membership**: Assignment of a role to a principal on a resource

### Data Model

```python
# cortex/platform/database/models.py

class PrincipalType(str, Enum):
    USER = "user"
    SERVICE_ACCOUNT = "service_account"

class Principal(Base):
    """User or service account."""
    __tablename__ = "principals"

    id = Column(Integer, primary_key=True)
    uid = Column(String(255), unique=True, nullable=False)
    email = Column(String(255), unique=True, nullable=False)
    display_name = Column(String(255), nullable=False)
    principal_type = Column(Enum(PrincipalType), default=PrincipalType.USER)
    admin = Column(Boolean, default=False)  # System admin
    blocked = Column(Boolean, default=False)
    salt = Column(String(255), nullable=True)  # For password auth
    created_at = Column(DateTime(timezone=True), server_default=func.now())

class Role(str, Enum):
    """RBAC roles."""
    OWNER = "owner"        # Full access including delete
    ADMIN = "admin"        # Manage resources but cannot delete
    CONTRIBUTOR = "contributor"  # Create and edit
    READER = "reader"      # Read-only

class Membership(Base):
    """Role assignment on a resource."""
    __tablename__ = "memberships"

    id = Column(Integer, primary_key=True)
    principal_id = Column(Integer, ForeignKey("principals.id"), nullable=False)
    resource_type = Column(String(50), nullable=False)  # "project", "document"
    resource_id = Column(String(255), nullable=False)   # Resource UID
    role = Column(Enum(Role), nullable=False, default=Role.READER)

    __table_args__ = (
        UniqueConstraint(
            "principal_id",
            "resource_type",
            "resource_id",
            name="uq_membership_principal_resource"
        ),
    )
```

### Authentication Flow

#### JWT Token Structure

```json
{
  "sub": "user_abc123",
  "tenant_id": "acct_xyz789",
  "principal_id": "123",
  "email": "alice@acme.com",
  "roles": ["owner"],
  "iat": 1710345600,
  "exp": 1710349200
}
```

#### Login Flow

```python
# POST /api/v1/auth/login
{
    "email": "alice@acme.com",
    "password": "secure_password"
}

# Backend process:
async def login(email: str, password: str) -> TokenResponse:
    # 1. Lookup principal
    principal = await db.query(Principal).filter_by(email=email).first()
    if not principal:
        raise AuthenticationError("Invalid credentials")

    # 2. Verify password
    if not verify_password(password, principal.salt):
        raise AuthenticationError("Invalid credentials")

    # 3. Get tenant (account)
    # Find account where user is a member
    account = await get_principal_account(principal.id)

    # 4. Generate JWT
    token = jwt.encode(
        {
            "sub": principal.uid,
            "tenant_id": account.uid,
            "principal_id": str(principal.id),
            "email": principal.email,
            "iat": datetime.utcnow(),
            "exp": datetime.utcnow() + timedelta(hours=24)
        },
        settings.JWT_SECRET,
        algorithm="HS256"
    )

    return TokenResponse(
        access_token=token,
        token_type="bearer",
        expires_in=86400
    )
```

#### Service Account Tokens

```python
# Long-lived tokens for service-to-service auth

class Token(Base):
    """PAT (Personal Access Token) or SAT (Service Account Token)."""
    __tablename__ = "tokens"

    id = Column(Integer, primary_key=True)
    uid = Column(String(255), unique=True)
    principal_id = Column(Integer, ForeignKey("principals.id"))
    token_type = Column(Enum(TokenType))  # PAT, SAT, SESSION
    token_hash = Column(String(255), nullable=False)
    scopes = Column(Text, nullable=True)  # JSON array
    expires_at = Column(DateTime(timezone=True), nullable=True)

# Create service account token
async def create_service_account_token(
    service_account_id: int,
    name: str,
    scopes: list[str]
) -> str:
    # Generate token
    raw_token = secrets.token_urlsafe(32)
    token_hash = hashlib.sha256(raw_token.encode()).hexdigest()

    # Store in database
    token = Token(
        uid=f"sat_{uuid.uuid4().hex[:12]}",
        principal_id=service_account_id,
        token_type=TokenType.SAT,
        token_hash=token_hash,
        scopes=json.dumps(scopes),
        expires_at=None  # Never expires (must be rotated manually)
    )
    await db.add(token)
    await db.commit()

    # Return raw token (only time it's visible)
    return f"cortex_sat_{raw_token}"
```

## Role-Based Access Control (RBAC)

### Permission Model

```python
class Permission(str, Enum):
    """Granular permissions."""

    # Project permissions
    PROJECT_VIEW = "project:view"
    PROJECT_EDIT = "project:edit"
    PROJECT_DELETE = "project:delete"

    # Conversation permissions
    CONVERSATION_VIEW = "conversation:view"
    CONVERSATION_CREATE = "conversation:create"
    CONVERSATION_DELETE = "conversation:delete"

    # Document permissions
    DOCUMENT_VIEW = "document:view"
    DOCUMENT_UPLOAD = "document:upload"
    DOCUMENT_DELETE = "document:delete"

    # Agent permissions
    AGENT_VIEW = "agent:view"
    AGENT_CREATE = "agent:create"
    AGENT_EXECUTE = "agent:execute"

    # Tool permissions
    TOOL_VIEW = "tool:view"
    TOOL_EXECUTE = "tool:execute"

# Role → Permission mapping
ROLE_PERMISSIONS = {
    Role.OWNER: [
        Permission.PROJECT_VIEW,
        Permission.PROJECT_EDIT,
        Permission.PROJECT_DELETE,
        Permission.CONVERSATION_VIEW,
        Permission.CONVERSATION_CREATE,
        Permission.CONVERSATION_DELETE,
        Permission.DOCUMENT_VIEW,
        Permission.DOCUMENT_UPLOAD,
        Permission.DOCUMENT_DELETE,
        Permission.AGENT_VIEW,
        Permission.AGENT_CREATE,
        Permission.AGENT_EXECUTE,
        Permission.TOOL_VIEW,
        Permission.TOOL_EXECUTE,
    ],
    Role.ADMIN: [
        Permission.PROJECT_VIEW,
        Permission.PROJECT_EDIT,
        # No PROJECT_DELETE
        Permission.CONVERSATION_VIEW,
        Permission.CONVERSATION_CREATE,
        Permission.DOCUMENT_VIEW,
        Permission.DOCUMENT_UPLOAD,
        Permission.AGENT_VIEW,
        Permission.AGENT_CREATE,
        Permission.AGENT_EXECUTE,
        Permission.TOOL_VIEW,
        Permission.TOOL_EXECUTE,
    ],
    Role.CONTRIBUTOR: [
        Permission.PROJECT_VIEW,
        Permission.CONVERSATION_VIEW,
        Permission.CONVERSATION_CREATE,
        Permission.DOCUMENT_VIEW,
        Permission.DOCUMENT_UPLOAD,
        Permission.AGENT_VIEW,
        Permission.AGENT_EXECUTE,
        Permission.TOOL_VIEW,
        Permission.TOOL_EXECUTE,
    ],
    Role.READER: [
        Permission.PROJECT_VIEW,
        Permission.CONVERSATION_VIEW,
        Permission.DOCUMENT_VIEW,
        Permission.AGENT_VIEW,
        Permission.TOOL_VIEW,
    ],
}
```

### Permission Checker

```python
class PermissionChecker:
    """Check if principal has permission on resource."""

    def __init__(self, session: AsyncSession):
        self.session = session

    async def check(
        self,
        principal: Principal,
        resource_type: str,
        resource_id: str,
        permission: Permission,
    ) -> bool:
        """Check if principal has permission on resource."""

        # System admins have all permissions
        if principal.admin:
            return True

        # Find membership
        membership = await self.session.query(Membership).filter_by(
            principal_id=principal.id,
            resource_type=resource_type,
            resource_id=resource_id
        ).first()

        if not membership:
            return False

        # Check if role has permission
        role_permissions = ROLE_PERMISSIONS.get(membership.role, [])
        return permission in role_permissions

    async def check_any(
        self,
        principal: Principal,
        resource_type: str,
        resource_id: str,
        permissions: list[Permission],
    ) -> bool:
        """Check if principal has ANY of the permissions."""
        for permission in permissions:
            if await self.check(principal, resource_type, resource_id, permission):
                return True
        return False
```

### Permission Enforcement (FastAPI Dependency)

```python
# cortex/platform/auth/dependencies.py

def require_permission(
    permission: Permission,
    resource_type: str,
    resource_param: str = "resource_id",
) -> Callable:
    """
    Dependency factory for permission checks.

    Usage:
        @app.get("/projects/{project_uid}")
        async def get_project(
            project_uid: str,
            principal: Principal = Depends(
                require_permission(Permission.PROJECT_VIEW, "project", "project_uid")
            )
        ):
            ...
    """

    async def dependency(
        request: Request,
        session: AsyncSession = Depends(get_db),
        principal: Principal = Depends(get_current_principal),
    ) -> Principal:
        # Extract resource ID from path params
        resource_id = request.path_params.get(resource_param)

        if not resource_id:
            raise HTTPException(
                status_code=400,
                detail=f"Missing resource parameter: {resource_param}"
            )

        # Check permission
        checker = PermissionChecker(session)
        has_access = await checker.check(
            principal=principal,
            resource_type=resource_type,
            resource_id=resource_id,
            permission=permission,
        )

        if not has_access:
            raise HTTPException(
                status_code=403,
                detail="Access denied"
            )

        return principal

    return dependency
```

## Multi-tenancy

### Hierarchy Model

```
Account (Tenant)
  ├── Organization 1
  │   ├── Project A
  │   │   ├── Conversation 1
  │   │   ├── Conversation 2
  │   │   └── Documents
  │   └── Project B
  └── Organization 2
      └── Project C
```

### Data Model

```python
class Account(Base):
    """Top-level tenant (billing entity)."""
    __tablename__ = "accounts"

    id = Column(Integer, primary_key=True)
    uid = Column(String(255), unique=True, nullable=False)
    name = Column(String(255), nullable=False)
    status = Column(Enum(AccountStatus), default=AccountStatus.TRIAL)
    subscription_tier = Column(Enum(SubscriptionTier), default=SubscriptionTier.FREE)
    owner_id = Column(Integer, ForeignKey("principals.id"))

    # Relationships
    organizations = relationship("Organization", back_populates="account")

class Organization(Base):
    """Business unit within account."""
    __tablename__ = "organizations"

    id = Column(Integer, primary_key=True)
    uid = Column(String(255), unique=True, nullable=False)
    account_id = Column(Integer, ForeignKey("accounts.id"), nullable=False)
    name = Column(String(255), nullable=False)

    # Relationships
    account = relationship("Account", back_populates="organizations")
    projects = relationship("Project", back_populates="organization")

class Project(Base):
    """Workspace within organization."""
    __tablename__ = "projects"

    id = Column(Integer, primary_key=True)
    uid = Column(String(255), unique=True, nullable=False)
    organization_id = Column(Integer, ForeignKey("organizations.id"))
    name = Column(String(255), nullable=False)

    # Relationships
    organization = relationship("Organization", back_populates="projects")
    conversations = relationship("Conversation", back_populates="project")
```

### Tenant Isolation Strategies

#### 1. Row-Level Security (PostgreSQL)

```sql
-- Enable RLS on table
ALTER TABLE conversations ENABLE ROW LEVEL SECURITY;

-- Policy: Users can only see conversations in their tenant
CREATE POLICY tenant_isolation ON conversations
    USING (
        project_id IN (
            SELECT p.id FROM projects p
            JOIN organizations o ON p.organization_id = o.id
            WHERE o.account_id = current_setting('app.current_tenant')::INTEGER
        )
    );

-- Set tenant context per session
SET app.current_tenant = 123;  -- Account ID
```

#### 2. Application-Level Filtering

```python
# Always filter by tenant in queries

async def get_conversations(tenant_id: str, project_id: str):
    """Get conversations with tenant isolation."""

    # Verify project belongs to tenant
    project = await db.query(Project).join(Organization).join(Account).filter(
        Project.uid == project_id,
        Account.uid == tenant_id
    ).first()

    if not project:
        raise HTTPException(status_code=404, detail="Project not found")

    # Query conversations with tenant filter
    conversations = await db.query(Conversation).filter(
        Conversation.project_id == project.id
    ).all()

    return conversations
```

#### 3. Collection-Based Isolation (Qdrant)

```python
# Separate collection per tenant
collection_name = f"embeddings_{tenant_id}"

# Or single collection with metadata filtering
qdrant.search(
    collection_name="embeddings",
    query_vector=embedding,
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

#### 4. Namespace-Based Isolation (Neo4j)

```cypher
// All nodes have tenant_id property
CREATE (e:Entity {
    tenant_id: "acct_123",
    name: "Customer X"
})

// Always filter by tenant
MATCH (e:Entity)
WHERE e.tenant_id = $tenant_id
RETURN e
```

### Tenant Onboarding

```python
async def create_tenant(
    name: str,
    owner_email: str,
    subscription_tier: SubscriptionTier = SubscriptionTier.FREE
) -> Account:
    """Create new tenant (account)."""

    # 1. Create principal (owner)
    owner = Principal(
        uid=f"user_{uuid.uuid4().hex[:12]}",
        email=owner_email,
        display_name=owner_email.split("@")[0],
        principal_type=PrincipalType.USER
    )
    await db.add(owner)
    await db.flush()

    # 2. Create account
    account = Account(
        uid=f"acct_{uuid.uuid4().hex[:12]}",
        name=name,
        status=AccountStatus.TRIAL,
        subscription_tier=subscription_tier,
        owner_id=owner.id
    )
    await db.add(account)
    await db.flush()

    # 3. Create default organization
    org = Organization(
        uid=f"org_{uuid.uuid4().hex[:12]}",
        account_id=account.id,
        name=f"{name} (Default)",
        owner_id=owner.id
    )
    await db.add(org)
    await db.flush()

    # 4. Create default project
    project = Project(
        uid=f"proj_{uuid.uuid4().hex[:12]}",
        organization_id=org.id,
        name="Default Project",
        owner_id=owner.id
    )
    await db.add(project)

    # 5. Create RBAC memberships (owner role on all)
    for resource in [
        ("account", account.uid),
        ("organization", org.uid),
        ("project", project.uid)
    ]:
        membership = Membership(
            principal_id=owner.id,
            resource_type=resource[0],
            resource_id=resource[1],
            role=Role.OWNER
        )
        await db.add(membership)

    # 6. Provision tenant resources
    await provision_tenant_resources(account)

    await db.commit()
    return account

async def provision_tenant_resources(account: Account):
    """Provision tenant-specific resources."""

    # Create Qdrant collection
    await qdrant.create_collection(
        collection_name=f"embeddings_{account.uid}",
        vectors_config=VectorParams(size=1536, distance="Cosine")
    )

    # Initialize usage tracking
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

## Observability

### Metrics (Prometheus)

```python
from prometheus_client import Counter, Histogram, Gauge

# API metrics
api_requests_total = Counter(
    'api_requests_total',
    'Total API requests',
    ['tenant_id', 'method', 'endpoint', 'status']
)

api_latency = Histogram(
    'api_latency_seconds',
    'API request latency',
    ['tenant_id', 'endpoint']
)

# LLM metrics
llm_tokens_used = Counter(
    'llm_tokens_used',
    'LLM tokens consumed',
    ['tenant_id', 'model', 'provider']
)

llm_api_latency = Histogram(
    'llm_api_latency_seconds',
    'LLM API call latency',
    ['tenant_id', 'model', 'provider']
)

# Resource metrics
active_conversations = Gauge(
    'active_conversations',
    'Number of active conversations',
    ['tenant_id']
)

storage_used_bytes = Gauge(
    'storage_used_bytes',
    'Storage used in bytes',
    ['tenant_id']
)

# Agent metrics
agent_executions_total = Counter(
    'agent_executions_total',
    'Total agent executions',
    ['tenant_id', 'agent_name', 'status']
)

tool_invocations_total = Counter(
    'tool_invocations_total',
    'Total tool invocations',
    ['tenant_id', 'tool_name', 'status']
)
```

### Logging (Structured JSON)

```python
import structlog

logger = structlog.get_logger()

# Request logging
logger.info(
    "api_request",
    tenant_id=tenant_id,
    principal_id=principal_id,
    method=request.method,
    path=request.url.path,
    ip=request.client.host
)

# Agent execution logging
logger.info(
    "agent_execution",
    tenant_id=tenant_id,
    agent_name="assistant",
    thread_id=thread_id,
    message_count=len(messages),
    duration_ms=duration
)

# Error logging
logger.error(
    "database_error",
    tenant_id=tenant_id,
    error=str(e),
    query=query,
    exc_info=True
)
```

### Tracing (OpenTelemetry)

```python
from opentelemetry import trace
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor

# Instrument FastAPI
FastAPIInstrumentor.instrument_app(app)

tracer = trace.get_tracer(__name__)

async def chat_endpoint(request: ChatRequest):
    with tracer.start_as_current_span("chat_endpoint") as span:
        span.set_attribute("tenant_id", tenant_id)
        span.set_attribute("conversation_id", conversation_id)

        # Nested span: Load conversation
        with tracer.start_as_current_span("load_conversation"):
            conversation = await load_conversation(...)

        # Nested span: Agent execution
        with tracer.start_as_current_span("agent_execution"):
            result = await agent.run(...)

        # Nested span: Persist messages
        with tracer.start_as_current_span("persist_messages"):
            await persist_messages(...)

        return result
```

## Policy Engine

### Rate Limiting

```python
class RateLimiter:
    """Tenant-scoped rate limiting."""

    def __init__(self, redis: Redis):
        self.redis = redis

    async def check_limit(
        self,
        tenant_id: str,
        resource: str,
        limit: int,
        window: int = 60  # seconds
    ) -> bool:
        """Check if tenant has exceeded rate limit."""

        key = f"ratelimit:{tenant_id}:{resource}:{int(time.time() / window)}"
        count = await self.redis.incr(key)

        if count == 1:
            await self.redis.expire(key, window)

        return count <= limit
```

### Quota Management

```python
class QuotaManager:
    """Enforce usage quotas per tenant."""

    async def check_quota(
        self,
        tenant_id: str,
        quota_type: str,
        amount: int = 1
    ) -> bool:
        """Check if tenant has quota remaining."""

        # Get tenant's quota
        account = await db.query(Account).filter_by(uid=tenant_id).first()
        quotas = TIER_QUOTAS[account.subscription_tier]
        limit = quotas.get(quota_type)

        # Get current usage this month
        usage = await self._get_monthly_usage(tenant_id, quota_type)

        return (usage + amount) <= limit

    async def record_usage(
        self,
        tenant_id: str,
        quota_type: str,
        amount: int
    ):
        """Record usage against quota."""

        usage = Usage(
            tenant_id=tenant_id,
            quota_type=quota_type,
            amount=amount,
            period=datetime.utcnow().strftime("%Y-%m")
        )
        await db.add(usage)
        await db.commit()

# Tier quotas
TIER_QUOTAS = {
    SubscriptionTier.FREE: {
        "api_calls_per_month": 1000,
        "llm_tokens_per_month": 100000,
        "storage_gb": 1
    },
    SubscriptionTier.PRO: {
        "api_calls_per_month": 50000,
        "llm_tokens_per_month": 1000000,
        "storage_gb": 10
    },
    SubscriptionTier.ENTERPRISE: {
        "api_calls_per_month": 1000000,
        "llm_tokens_per_month": 10000000,
        "storage_gb": 100
    }
}
```

### Compliance Policies

```python
class CompliancePolicy:
    """Enforce compliance policies."""

    async def check_data_retention(self, tenant_id: str):
        """Delete old data per retention policy."""

        account = await db.query(Account).filter_by(uid=tenant_id).first()
        retention_days = account.settings.get("data_retention_days", 90)

        # Delete old conversations
        cutoff = datetime.utcnow() - timedelta(days=retention_days)
        await db.query(Conversation).filter(
            Conversation.tenant_id == tenant_id,
            Conversation.created_at < cutoff
        ).delete()

    async def check_pii_compliance(self, content: str) -> bool:
        """Detect and flag PII in content."""

        # Detect email, SSN, credit card, etc.
        pii_patterns = [
            r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b',  # Email
            r'\b\d{3}-\d{2}-\d{4}\b',  # SSN
            r'\b\d{4}[- ]?\d{4}[- ]?\d{4}[- ]?\d{4}\b',  # Credit card
        ]

        for pattern in pii_patterns:
            if re.search(pattern, content):
                return False  # PII detected

        return True  # No PII
```

## Security Integration Matrix

The control plane enforces security controls across **all layers** of the platform. This matrix shows where each security domain is implemented and which integration points it protects.

### Security Domain Mapping

| Security Domain | Implementation | Integration Points | Enforcement Layer |
|----------------|----------------|-------------------|-------------------|
| **Authentication** | JWT validation, SAML/OIDC, MFA | All API endpoints, UI components, service-to-service calls | Gateway Layer |
| **Authorization** | RBAC permission checks, resource-level ACLs | Every API route, MCP tool calls, database queries | Control Plane |
| **Data Encryption** | AES-256 at rest, TLS 1.3 in transit | Database storage, Redis cache, API communications, LLM requests | All Layers |
| **Secrets Management** | Centralized vault (Hashicorp Vault/AWS KMS), rotation policies | LLM API keys, database credentials, connector authentication, webhook signing | Control Plane |
| **Audit Logging** | Structured logging, immutable audit trail | All user actions, system events, data access, permission changes | Control Plane |
| **Rate Limiting** | Token bucket, sliding window (Redis) | API endpoints, LLM gateway, vector search queries | Gateway + Control Plane |
| **Network Security** | VPC isolation, security groups, firewall rules | All service communication, database access, external APIs | Infrastructure |
| **Input Validation** | Schema validation (Pydantic), SQL injection prevention | All API inputs, user uploads, tool parameters | Gateway + Service Layer |

### Authentication Integration

**Where Applied**:

```python
# 1. API Gateway (FastAPI)
@app.get("/api/v1/projects/{project_id}")
async def get_project(
    project_id: str,
    principal: Principal = Depends(get_current_principal)  # ← JWT validation
):
    ...

# 2. MCP Gateway (Tool execution)
@mcp_server.tool("search_documents")
async def search_documents(
    query: str,
    context: MCPContext  # ← Contains authenticated principal
):
    ...

# 3. Service-to-Service (Internal APIs)
async def call_internal_service():
    response = await httpx.post(
        "http://analytics-service/internal/usage",
        headers={"Authorization": f"Bearer {service_account_token}"}  # ← Service auth
    )
```

**Technologies**:
- **JWT**: Stateless authentication for API calls
- **SAML/OIDC**: Enterprise SSO integration
- **MFA**: Time-based OTP (TOTP) via authenticator apps
- **Service Accounts**: Machine-to-machine authentication

### Authorization Integration

**Where Applied**:

```python
# 1. API endpoints (permission checks)
@app.post("/api/v1/projects/{project_id}/documents")
async def create_document(
    project_id: str,
    document: DocumentCreate,
    principal: Principal = Depends(
        require_permission(Permission.DOCUMENT_CREATE, "project", "project_id")
    )  # ← RBAC check
):
    ...

# 2. Database queries (tenant isolation)
async def get_conversations(tenant_id: str, user_id: str):
    return await db.query(Conversation).filter(
        Conversation.tenant_id == tenant_id,  # ← Tenant filter
        Conversation.user_id == user_id
    ).all()

# 3. LLM Gateway (quota enforcement)
async def generate_response(tenant_id: str, prompt: str):
    # Check tenant quota before LLM call
    if not await quota_service.check(tenant_id, "llm_tokens"):
        raise QuotaExceededError("Monthly token limit reached")
    ...
```

**Permission Granularity**:
- **Resource-level**: `project.create`, `document.read`, `conversation.delete`
- **Organization-level**: `org.admin`, `org.billing`
- **System-level**: `system.admin`, `system.audit`

### Data Encryption Integration

**Where Applied**:

| Data Type | At Rest | In Transit | Key Management |
|-----------|---------|------------|----------------|
| **PostgreSQL** | AES-256 (database encryption) | TLS 1.3 (pg_hba.conf) | AWS RDS managed keys |
| **Redis** | AES-256 (disk encryption) | TLS 1.3 (stunnel) | KMS-managed keys |
| **Qdrant** | AES-256 (volume encryption) | TLS 1.3 (HTTPS) | Customer-managed keys |
| **Neo4j** | AES-256 (enterprise edition) | TLS 1.3 (bolt+tls) | Customer-managed keys |
| **S3 (file uploads)** | AES-256 (SSE-KMS) | TLS 1.3 (HTTPS) | AWS KMS |
| **LLM API calls** | N/A (ephemeral) | TLS 1.3 (HTTPS) | N/A |

**Implementation**:

```python
# Database connection with TLS
DATABASE_URL = "postgresql://user:pass@host:5432/db?sslmode=require"

# Redis connection with TLS
REDIS_URL = "rediss://user:pass@host:6379/0"  # rediss = Redis with TLS

# Qdrant with HTTPS
qdrant_client = QdrantClient(
    url="https://qdrant.example.com",
    api_key=get_secret("qdrant_api_key"),  # From vault
    timeout=10,
)
```

### Secrets Management Integration

**Where Applied**:

```python
from cortex.platform.secrets import SecretManager

secret_manager = SecretManager()  # Backed by Hashicorp Vault or AWS Secrets Manager

# 1. LLM API keys
openai_key = await secret_manager.get("llm/openai/api_key")
anthropic_key = await secret_manager.get("llm/anthropic/api_key")

# 2. Database credentials
db_password = await secret_manager.get("database/postgres/password")

# 3. Connector authentication (OAuth tokens)
github_token = await secret_manager.get(f"connectors/github/{tenant_id}/token")

# 4. Webhook signing keys
webhook_secret = await secret_manager.get(f"webhooks/{tenant_id}/signing_key")

# Auto-rotation every 90 days
await secret_manager.rotate("llm/openai/api_key")
```

**Rotation Policies**:
- **LLM API keys**: Manual rotation (when compromised)
- **Database credentials**: 90-day auto-rotation
- **OAuth tokens**: Refresh token flow (automatic)
- **Webhook secrets**: 180-day rotation with grace period

### Audit Logging Integration

**Where Applied**:

```python
# 1. API calls (all mutations)
@app.post("/api/v1/projects")
async def create_project(project: ProjectCreate, principal: Principal):
    audit_log.record(
        event_type="project.create",
        actor=principal.uid,
        tenant_id=principal.tenant_id,
        resource_type="project",
        resource_id=new_project.uid,
        metadata={"name": project.name}
    )

# 2. Permission changes
async def add_membership(principal_id: str, resource_id: str, role: Role):
    membership = Membership(...)
    await db.add(membership)

    audit_log.record(
        event_type="membership.add",
        actor=current_principal.uid,
        resource_type="membership",
        resource_id=membership.id,
        metadata={"principal_id": principal_id, "role": role}
    )

# 3. Data access (sensitive queries)
async def get_user_profile(user_id: str):
    audit_log.record(
        event_type="user.profile.access",
        actor=current_principal.uid,
        resource_type="principal",
        resource_id=user_id
    )

# 4. LLM calls (token usage, prompts)
async def llm_generate(prompt: str):
    audit_log.record(
        event_type="llm.generate",
        actor=principal.uid,
        tenant_id=principal.tenant_id,
        metadata={
            "model": "gpt-4o",
            "prompt_length": len(prompt),
            "tokens": 1500,
            "cost_usd": 0.015
        }
    )
```

**Audit Log Storage**:
- **Hot storage**: PostgreSQL (last 30 days, queryable)
- **Cold storage**: S3 (90+ days, archival, immutable)
- **Retention**: 7 years (compliance requirement for SOC 2)

### Network Security Integration

**Deployment Pattern**:

```
┌─────────────────────────────────────────────────────────┐
│                   Public Internet                        │
└───────────────────────┬─────────────────────────────────┘
                        ↓
┌───────────────────────────────────────────────────────────┐
│  WAF (Web Application Firewall)                           │
│  - DDoS protection                                        │
│  - SQL injection blocking                                 │
│  - Rate limiting (per IP)                                 │
└───────────────────────┬───────────────────────────────────┘
                        ↓
┌───────────────────────────────────────────────────────────┐
│  Load Balancer (AWS ALB / CloudFlare)                     │
│  - TLS termination                                        │
│  - Health checks                                          │
└───────────────────────┬───────────────────────────────────┘
                        ↓
┌───────────────────────────────────────────────────────────┐
│  VPC (Virtual Private Cloud)                              │
│  ┌─────────────────────────────────────────────────────┐  │
│  │  Public Subnet (Gateway Layer)                      │  │
│  │  - API Gateway (0.0.0.0/0 → allow HTTPS)            │  │
│  │  - MCP Gateway (0.0.0.0/0 → allow HTTPS)            │  │
│  └─────────────────────────────────────────────────────┘  │
│  ┌─────────────────────────────────────────────────────┐  │
│  │  Private Subnet (Service Layer)                     │  │
│  │  - No direct internet access                        │  │
│  │  - Security group: Allow from gateway only          │  │
│  └─────────────────────────────────────────────────────┘  │
│  ┌─────────────────────────────────────────────────────┐  │
│  │  Private Subnet (Data Layer)                        │  │
│  │  - PostgreSQL (allow from service subnet only)      │  │
│  │  - Redis (allow from service subnet only)           │  │
│  │  - Security group: Deny all external                │  │
│  └─────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────┘
```

**Security Group Rules**:

```yaml
# API Gateway (public subnet)
ingress:
  - port: 443
    cidr: 0.0.0.0/0  # HTTPS from internet
egress:
  - port: 5432
    cidr: 10.0.2.0/24  # PostgreSQL in private subnet

# PostgreSQL (private subnet)
ingress:
  - port: 5432
    cidr: 10.0.1.0/24  # Service layer only
egress: []  # No outbound allowed
```

### Input Validation Integration

**Where Applied**:

```python
# 1. API input validation (Pydantic)
from pydantic import BaseModel, Field, validator

class DocumentCreate(BaseModel):
    title: str = Field(..., min_length=1, max_length=500)
    content: str = Field(..., min_length=1, max_length=100000)
    tags: list[str] = Field(default_factory=list, max_items=10)

    @validator("tags")
    def validate_tags(cls, tags):
        # Prevent XSS in tags
        for tag in tags:
            if re.search(r'[<>"\']', tag):
                raise ValueError("Tags cannot contain HTML/script characters")
        return tags

# 2. SQL injection prevention (parameterized queries)
# Bad (vulnerable to SQL injection)
query = f"SELECT * FROM users WHERE email = '{user_input}'"

# Good (parameterized)
query = db.query(User).filter(User.email == user_input)

# 3. Path traversal prevention
async def get_file(file_path: str):
    # Prevent ../../../etc/passwd
    safe_path = os.path.normpath(file_path)
    if safe_path.startswith(".."):
        raise ValueError("Invalid file path")
```

### Compliance & Certifications

**Current Status**:

| Standard | Status | Scope | Audit Frequency |
|----------|--------|-------|-----------------|
| **SOC 2 Type II** | ✅ Certified | Control Plane + Data Plane | Annual |
| **HIPAA** | ✅ Ready | On-premise deployments | Customer-managed |
| **GDPR** | ✅ Compliant | All EU tenants | Continuous |
| **ISO 27001** | 🚧 In Progress | Platform-wide | Target: Q3 2024 |
| **PCI DSS** | ❌ Not applicable | We don't process payments | N/A |

**SOC 2 Controls**:
- ✅ Access Control (IAM/RBAC)
- ✅ Encryption (at rest + in transit)
- ✅ Monitoring (OpenTelemetry + alerts)
- ✅ Incident Response (runbooks, escalation)
- ✅ Change Management (PR reviews, audit logs)
- ✅ Vendor Management (subprocessors documented)

### Security Best Practices

**Principle of Least Privilege**:
```python
# Bad: Grant broad permissions
role = Role.ADMIN  # Can do everything

# Good: Grant minimum required permissions
role = Role.CONTRIBUTOR  # Can only create/edit, not delete
```

**Defense in Depth**:
1. **Layer 1**: WAF blocks malicious requests
2. **Layer 2**: Gateway validates JWT
3. **Layer 3**: Control Plane checks RBAC
4. **Layer 4**: Service validates input
5. **Layer 5**: Database enforces constraints

**Zero Trust Architecture**:
- Every request authenticated (no implicit trust)
- Every resource access authorized (explicit grants)
- Every action logged (immutable audit trail)
- Every secret rotated (time-based expiration)

---

**Next**: [Service Layer](./03-service-layer.md) | [Previous](./01-gateway-layer.md) | [Back to Overview](./00-convergence-overview.md)
