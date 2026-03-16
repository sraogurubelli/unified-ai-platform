# Shared Control Plane Services

## Overview

The **Shared Control Plane** provides foundational services used by both Traditional and AI stacks including authentication, multi-tenancy, observability, and secrets management.

```
┌──────────────────────────────────────────────────────────────┐
│              Shared Control Plane Services                    │
├──────────────┬───────────────┬──────────────┬────────────────┤
│ IAM & Auth   │ Multi-tenancy │ Observability│ Secrets Mgmt   │
│              │               │              │                │
│ • JWT tokens │ • Account→Org │ • Metrics    │ • Vault        │
│ • RBAC       │   →Project    │ • Logs       │ • Key rotation │
│ • SSO/SAML   │ • Isolation   │ • Traces     │ • Encryption   │
└──────────────┴───────────────┴──────────────┴────────────────┘
```

---

## Identity & Access Management (IAM)

### Shared Identity Model

```python
class PrincipalType(str, Enum):
    USER = "user"
    SERVICE_ACCOUNT = "service_account"
    API_KEY = "api_key"

class Principal(Base):
    """Unified identity for users and service accounts."""
    __tablename__ = "principals"

    id = Column(Integer, primary_key=True)
    uid = Column(String(255), unique=True, nullable=False)
    email = Column(String(255), unique=True, nullable=True)
    display_name = Column(String(255), nullable=False)
    principal_type = Column(Enum(PrincipalType), default=PrincipalType.USER)
    admin = Column(Boolean, default=False)
    blocked = Column(Boolean, default=False)
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    last_login_at = Column(DateTime(timezone=True), nullable=True)
```

### Authentication Service (Shared by Both Gateways)

```python
class AuthenticationService:
    """Unified authentication for API and MCP gateways."""

    def __init__(self, jwt_secret: str, jwt_algorithm: str = "HS256"):
        self.jwt_secret = jwt_secret
        self.jwt_algorithm = jwt_algorithm

    async def login(
        self,
        email: str,
        password: str
    ) -> dict:
        """Authenticate user and return JWT token."""

        # Validate credentials
        principal = await db.query(Principal).filter_by(email=email).first()

        if not principal:
            raise AuthenticationError("Invalid credentials")

        if principal.blocked:
            raise AuthenticationError("Account blocked")

        # Verify password
        if not self._verify_password(password, principal.salt):
            raise AuthenticationError("Invalid credentials")

        # Get account (tenant)
        membership = await db.query(Membership).filter_by(
            principal_id=principal.id
        ).first()

        if not membership:
            raise AuthenticationError("No account access")

        # Get tenant_id from account
        account = await self._get_account_for_principal(principal.id)

        # Create JWT token
        token = self._create_token(
            principal_id=principal.id,
            tenant_id=account.uid,
            email=principal.email
        )

        # Update last login
        principal.last_login_at = datetime.utcnow()
        await db.commit()

        return {
            "token": token,
            "principal": {
                "id": principal.uid,
                "email": principal.email,
                "display_name": principal.display_name
            },
            "tenant_id": account.uid
        }

    def _create_token(
        self,
        principal_id: int,
        tenant_id: str,
        email: str,
        ttl: int = 3600
    ) -> str:
        """Create JWT token."""

        payload = {
            "sub": str(principal_id),
            "tenant_id": tenant_id,
            "principal_id": principal_id,
            "email": email,
            "iat": datetime.utcnow(),
            "exp": datetime.utcnow() + timedelta(seconds=ttl)
        }

        return jwt.encode(payload, self.jwt_secret, algorithm=self.jwt_algorithm)

    async def validate_token(self, token: str) -> dict:
        """Validate JWT token and return payload."""

        try:
            payload = jwt.decode(
                token,
                self.jwt_secret,
                algorithms=[self.jwt_algorithm]
            )

            # Check expiration
            if datetime.fromtimestamp(payload["exp"]) < datetime.utcnow():
                raise AuthenticationError("Token expired")

            # Check if principal is blocked
            principal = await db.query(Principal).filter_by(
                id=payload["principal_id"]
            ).first()

            if principal and principal.blocked:
                raise AuthenticationError("Account blocked")

            return payload

        except jwt.InvalidTokenError as e:
            raise AuthenticationError(f"Invalid token: {str(e)}")

# Usage in both gateways
@app.middleware("http")
async def auth_middleware(request: Request, call_next):
    """Shared authentication middleware."""

    token = request.headers.get("Authorization", "").replace("Bearer ", "")
    payload = await auth_service.validate_token(token)

    request.state.tenant_id = payload["tenant_id"]
    request.state.principal_id = payload["principal_id"]

    return await call_next(request)
```

### SSO/SAML Integration

```python
from onelogin.saml2.auth import OneLogin_Saml2_Auth

class SSOService:
    """SSO authentication via SAML."""

    async def initiate_sso_login(self, request: Request):
        """Initiate SSO login flow."""

        saml_auth = OneLogin_Saml2_Auth(request, self.saml_settings)
        return saml_auth.login()

    async def process_sso_response(self, request: Request):
        """Process SAML response after SSO login."""

        saml_auth = OneLogin_Saml2_Auth(request, self.saml_settings)
        saml_auth.process_response()

        if not saml_auth.is_authenticated():
            raise AuthenticationError("SSO authentication failed")

        # Get user attributes
        attributes = saml_auth.get_attributes()
        email = attributes["email"][0]
        display_name = attributes["displayName"][0]

        # Find or create principal
        principal = await db.query(Principal).filter_by(email=email).first()

        if not principal:
            principal = Principal(
                uid=f"sso_{uuid.uuid4().hex[:12]}",
                email=email,
                display_name=display_name,
                principal_type=PrincipalType.USER
            )
            await db.add(principal)
            await db.commit()

        # Create JWT token
        account = await self._get_account_for_principal(principal.id)

        token = auth_service._create_token(
            principal_id=principal.id,
            tenant_id=account.uid,
            email=principal.email
        )

        return {
            "token": token,
            "principal": {
                "id": principal.uid,
                "email": principal.email
            }
        }
```

---

## Role-Based Access Control (RBAC)

### Unified Permission Model

```python
class Permission(str, Enum):
    # Project permissions
    PROJECT_CREATE = "project.create"
    PROJECT_READ = "project.read"
    PROJECT_UPDATE = "project.update"
    PROJECT_DELETE = "project.delete"

    # Document permissions
    DOCUMENT_CREATE = "document.create"
    DOCUMENT_READ = "document.read"
    DOCUMENT_UPDATE = "document.update"
    DOCUMENT_DELETE = "document.delete"

    # Conversation permissions
    CONVERSATION_CREATE = "conversation.create"
    CONVERSATION_READ = "conversation.read"

    # Tool permissions (AI-specific)
    TOOL_EXECUTE = "tool.execute"
    TOOL_MANAGE = "tool.manage"

    # Agent permissions (AI-specific)
    AGENT_CREATE = "agent.create"
    AGENT_EXECUTE = "agent.execute"

class Role(str, Enum):
    OWNER = "owner"          # Full access
    ADMIN = "admin"          # Manage resources
    CONTRIBUTOR = "contributor"  # Create and edit
    READER = "reader"        # Read-only

# Role → Permission mapping
ROLE_PERMISSIONS = {
    Role.OWNER: [
        # All permissions
        Permission.PROJECT_CREATE,
        Permission.PROJECT_READ,
        Permission.PROJECT_UPDATE,
        Permission.PROJECT_DELETE,
        Permission.DOCUMENT_CREATE,
        Permission.DOCUMENT_READ,
        Permission.DOCUMENT_UPDATE,
        Permission.DOCUMENT_DELETE,
        Permission.CONVERSATION_CREATE,
        Permission.CONVERSATION_READ,
        Permission.TOOL_EXECUTE,
        Permission.TOOL_MANAGE,
        Permission.AGENT_CREATE,
        Permission.AGENT_EXECUTE,
    ],
    Role.ADMIN: [
        # Cannot delete projects
        Permission.PROJECT_CREATE,
        Permission.PROJECT_READ,
        Permission.PROJECT_UPDATE,
        Permission.DOCUMENT_CREATE,
        Permission.DOCUMENT_READ,
        Permission.DOCUMENT_UPDATE,
        Permission.DOCUMENT_DELETE,
        Permission.CONVERSATION_CREATE,
        Permission.CONVERSATION_READ,
        Permission.TOOL_EXECUTE,
        Permission.AGENT_CREATE,
        Permission.AGENT_EXECUTE,
    ],
    Role.CONTRIBUTOR: [
        Permission.PROJECT_READ,
        Permission.DOCUMENT_CREATE,
        Permission.DOCUMENT_READ,
        Permission.DOCUMENT_UPDATE,
        Permission.CONVERSATION_CREATE,
        Permission.CONVERSATION_READ,
        Permission.TOOL_EXECUTE,
        Permission.AGENT_EXECUTE,
    ],
    Role.READER: [
        Permission.PROJECT_READ,
        Permission.DOCUMENT_READ,
        Permission.CONVERSATION_READ,
    ],
}

class RBACService:
    """Unified RBAC for all resources."""

    async def check_permission(
        self,
        principal_id: int,
        permission: Permission,
        resource_type: str = None,
        resource_id: str = None
    ) -> bool:
        """Check if principal has permission on resource."""

        # Get principal's roles
        memberships = await db.query(Membership).filter_by(
            principal_id=principal_id
        ).all()

        if resource_type and resource_id:
            # Check specific resource membership
            memberships = [
                m for m in memberships
                if m.resource_type == resource_type and m.resource_id == resource_id
            ]

        # Check if any role grants permission
        for membership in memberships:
            role_permissions = ROLE_PERMISSIONS.get(membership.role, [])
            if permission in role_permissions:
                return True

        return False

# Usage in API Gateway
@app.post("/api/v1/projects")
async def create_project(request: Request, ...):
    """API endpoint with RBAC check."""

    has_permission = await rbac_service.check_permission(
        principal_id=request.state.principal_id,
        permission=Permission.PROJECT_CREATE
    )

    if not has_permission:
        raise HTTPException(status_code=403, detail="Permission denied")

    ...

# Usage in MCP Gateway
async def mcp_invoke_tool(tool_name: str, context: Context):
    """MCP tool with RBAC check."""

    has_permission = await rbac_service.check_permission(
        principal_id=context.get("principal_id"),
        permission=Permission.TOOL_EXECUTE,
        resource_type="tool",
        resource_id=tool_name
    )

    if not has_permission:
        raise PermissionDenied(f"No permission to execute {tool_name}")

    ...
```

---

## Multi-Tenancy (Account → Organization → Project)

### Hierarchy Model

```python
class Account(Base):
    """Top-level tenant (billing entity)."""
    __tablename__ = "accounts"

    id = Column(Integer, primary_key=True)
    uid = Column(String(255), unique=True, nullable=False)
    name = Column(String(255), nullable=False)
    status = Column(String(50), default="active")
    subscription_tier = Column(Enum(SubscriptionTier), default=SubscriptionTier.FREE)
    created_at = Column(DateTime(timezone=True), server_default=func.now())

class Organization(Base):
    """Organizational unit within account."""
    __tablename__ = "organizations"

    id = Column(Integer, primary_key=True)
    uid = Column(String(255), unique=True, nullable=False)
    account_id = Column(Integer, ForeignKey("accounts.id", ondelete="CASCADE"))
    name = Column(String(255), nullable=False)
    created_at = Column(DateTime(timezone=True), server_default=func.now())

class Project(Base):
    """Project within organization."""
    __tablename__ = "projects"

    id = Column(Integer, primary_key=True)
    uid = Column(String(255), unique=True, nullable=False)
    organization_id = Column(Integer, ForeignKey("organizations.id", ondelete="CASCADE"))
    name = Column(String(255), nullable=False)
    created_at = Column(DateTime(timezone=True), server_default=func.now())
```

### Tenant Context Injection

```python
class TenantContext:
    """Tenant context for request handling."""

    def __init__(self, tenant_id: str, principal_id: int):
        self.tenant_id = tenant_id
        self.principal_id = principal_id
        self.account = None
        self.organization = None
        self.project = None

    async def load(self):
        """Load tenant hierarchy."""

        self.account = await db.query(Account).filter_by(
            uid=self.tenant_id
        ).first()

# Middleware to inject tenant context
@app.middleware("http")
async def tenant_context_middleware(request: Request, call_next):
    """Inject tenant context."""

    tenant_context = TenantContext(
        tenant_id=request.state.tenant_id,
        principal_id=request.state.principal_id
    )

    await tenant_context.load()

    request.state.tenant_context = tenant_context

    return await call_next(request)
```

---

## Unified Observability

### Metrics (Prometheus)

```python
from prometheus_client import Counter, Histogram, Gauge, Info

# Request metrics (unified for both gateways)
request_count = Counter(
    'platform_requests_total',
    'Total requests',
    ['gateway_type', 'tenant_id', 'endpoint', 'status']
)

request_latency = Histogram(
    'platform_request_latency_seconds',
    'Request latency',
    ['gateway_type', 'tenant_id', 'endpoint'],
    buckets=[0.01, 0.05, 0.1, 0.5, 1.0, 2.0, 5.0, 10.0]
)

# Resource metrics
active_users = Gauge(
    'platform_active_users',
    'Active users',
    ['tenant_id']
)

storage_bytes = Gauge(
    'platform_storage_bytes',
    'Storage usage',
    ['tenant_id', 'storage_type']
)

# System info
platform_info = Info('platform_info', 'Platform information')
platform_info.info({
    'version': '1.0.0',
    'deployment': 'saas'
})
```

### Distributed Tracing (OpenTelemetry)

```python
from opentelemetry import trace
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.exporter.jaeger import JaegerExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

# Initialize tracing
trace.set_tracer_provider(TracerProvider())
jaeger_exporter = JaegerExporter(
    agent_host_name="jaeger",
    agent_port=6831,
)
trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(jaeger_exporter)
)

# Instrument FastAPI
FastAPIInstrumentor.instrument_app(app)

# Create custom spans
tracer = trace.get_tracer(__name__)

async def execute_with_tracing(operation_name: str, func, **kwargs):
    """Execute function with distributed tracing."""

    with tracer.start_as_current_span(operation_name) as span:
        span.set_attribute("tenant_id", kwargs.get("tenant_id"))
        span.set_attribute("operation", operation_name)

        result = await func(**kwargs)

        span.set_attribute("result_size", len(str(result)))

        return result
```

### Structured Logging

```python
import structlog

# Configure structured logging
structlog.configure(
    processors=[
        structlog.stdlib.filter_by_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.stdlib.add_logger_name,
        structlog.stdlib.add_log_level,
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
        structlog.processors.UnicodeDecoder(),
        structlog.processors.JSONRenderer()
    ],
    logger_factory=structlog.stdlib.LoggerFactory(),
    cache_logger_on_first_use=True,
)

logger = structlog.get_logger()

# Usage
logger.info(
    "request_processed",
    gateway_type="api",
    tenant_id="acct_123",
    endpoint="/api/v1/chat",
    status_code=200,
    latency_ms=123
)

logger.error(
    "database_error",
    tenant_id="acct_123",
    error_type="ConnectionTimeout",
    database="postgres"
)
```

---

## Secrets Management

### Vault Integration

```python
import hvac

class SecretsManager:
    """Manage secrets using HashiCorp Vault."""

    def __init__(self, vault_url: str, vault_token: str):
        self.client = hvac.Client(url=vault_url, token=vault_token)

    async def get_secret(self, path: str) -> dict:
        """Retrieve secret from Vault."""

        try:
            response = self.client.secrets.kv.v2.read_secret_version(path=path)
            return response["data"]["data"]
        except Exception as e:
            logger.error("secret_retrieval_failed", path=path, error=str(e))
            raise

    async def set_secret(self, path: str, data: dict):
        """Store secret in Vault."""

        self.client.secrets.kv.v2.create_or_update_secret(
            path=path,
            secret=data
        )

# Usage
secrets_manager = SecretsManager(
    vault_url="http://vault:8200",
    vault_token=os.getenv("VAULT_TOKEN")
)

# Get database credentials
db_creds = await secrets_manager.get_secret("database/postgres")
DATABASE_URL = f"postgresql://{db_creds['username']}:{db_creds['password']}@postgres:5432/cortex"

# Get API keys
llm_api_key = await secrets_manager.get_secret("api_keys/openai")
```

### Key Rotation

```python
class KeyRotationService:
    """Automatic key rotation for secrets."""

    async def rotate_jwt_secret(self):
        """Rotate JWT secret key."""

        # Generate new secret
        new_secret = secrets.token_urlsafe(64)

        # Store in Vault
        await secrets_manager.set_secret("auth/jwt_secret", {
            "secret": new_secret,
            "rotated_at": datetime.utcnow().isoformat()
        })

        # Update application config (requires restart)
        logger.info("jwt_secret_rotated")

    async def rotate_database_password(self):
        """Rotate database password."""

        # Generate new password
        new_password = secrets.token_urlsafe(32)

        # Update database user
        await db.execute(f"ALTER USER cortex WITH PASSWORD '{new_password}'")

        # Store in Vault
        await secrets_manager.set_secret("database/postgres", {
            "username": "cortex",
            "password": new_password,
            "rotated_at": datetime.utcnow().isoformat()
        })

        logger.info("database_password_rotated")

# Schedule key rotation (monthly)
from apscheduler.schedulers.asyncio import AsyncIOScheduler

scheduler = AsyncIOScheduler()
scheduler.add_job(
    key_rotation_service.rotate_jwt_secret,
    'cron',
    day=1,  # First day of month
    hour=0
)
scheduler.start()
```

---

## Event Bus (Kafka)

### Shared Event Topics

```python
class EventBus:
    """Unified event bus for cross-stack communication."""

    def __init__(self, kafka_bootstrap_servers: str):
        self.producer = KafkaProducer(
            bootstrap_servers=kafka_bootstrap_servers,
            value_serializer=lambda v: json.dumps(v).encode('utf-8')
        )

    async def publish(self, topic: str, event: dict):
        """Publish event to Kafka topic."""

        # Add metadata
        event["event_id"] = str(uuid.uuid4())
        event["timestamp"] = datetime.utcnow().isoformat()

        self.producer.send(topic, value=event)

        logger.info(
            "event_published",
            topic=topic,
            event_id=event["event_id"]
        )

# Shared topics
SHARED_TOPICS = {
    "user.created": "Events when new users are created",
    "user.deleted": "Events when users are deleted",
    "subscription.upgraded": "Subscription tier changes",
    "quota.exceeded": "Quota violations",
    "security.alert": "Security incidents",
}

# Usage
event_bus = EventBus(kafka_bootstrap_servers="kafka:9092")

# Publish event
await event_bus.publish("user.created", {
    "tenant_id": "acct_123",
    "principal_id": 456,
    "email": "alice@example.com"
})
```

---

**Previous**: [Traditional Control Plane](./02a-traditional-control-plane.md) | [AI Control Plane](./02b-ai-control-plane.md) | [Back to Overview](../00-parallel-stacks-overview.md)
