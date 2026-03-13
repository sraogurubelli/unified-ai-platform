# Hybrid Deployment Model

## Overview

Hybrid deployment combines SaaS control plane with on-premise data plane. The platform provider manages authentication, billing, and orchestration, while customer data never leaves their premises.

**Ideal for**: Regulated industries (finance, healthcare), data residency requirements, compliance mandates (GDPR, HIPAA, SOC2).

## Architecture

```
┌─────────────────────────────────┐        ┌──────────────────────────────┐
│  SaaS Provider (Cloud)          │        │  Customer Environment        │
│                                 │        │  (On-Premise / VPC)          │
│  ┌───────────────────────────┐ │        │                              │
│  │   API Gateway (Proxy)     │ │◄──────►│  ┌────────────────────────┐  │
│  │   - Route to customer     │ │  HTTPS │  │  Customer API Gateway  │  │
│  │   - Auth enforcement      │ │        │  │  - Local routing       │  │
│  └───────────────────────────┘ │        │  └────────────────────────┘  │
│                                 │        │             ▼                │
│  ┌───────────────────────────┐ │        │  ┌────────────────────────┐  │
│  │   Control Plane           │ │        │  │  Service Layer         │  │
│  │   - IAM / RBAC            │ │◄──────►│  │  - Agent orchestration │  │
│  │   - Billing               │ │  gRPC  │  │  - RAG pipelines       │  │
│  │   - Observability         │ │        │  │  - Business logic      │  │
│  │   - Policy management     │ │        │  └────────────────────────┘  │
│  └───────────────────────────┘ │        │             ▼                │
│                                 │        │  ┌────────────────────────┐  │
│  ┌───────────────────────────┐ │        │  │  Data Plane (Private)  │  │
│  │   Agent Orchestrator      │ │        │  │                        │  │
│  │   (Stateless)             │ │───────►│  │  ┌──────────────────┐  │  │
│  │   - Workflow engine       │ │ Tasks  │  │  │  PostgreSQL      │  │  │
│  │   - No customer data      │ │        │  │  │  (Customer data) │  │  │
│  └───────────────────────────┘ │        │  │  └──────────────────┘  │  │
│                                 │        │  │                        │  │
│  ┌───────────────────────────┐ │        │  │  ┌──────────────────┐  │  │
│  │   Analytics Dashboard     │ │        │  │  │  Qdrant          │  │  │
│  │   - Aggregated metrics    │ │◄───────│  │  │  (Embeddings)    │  │  │
│  │   - No raw customer data  │ │Metrics │  │  └──────────────────┘  │  │
│  └───────────────────────────┘ │        │  │                        │  │
│                                 │        │  │  ┌──────────────────┐  │  │
└─────────────────────────────────┘        │  │  │  Neo4j           │  │  │
                                           │  │  │  (Knowledge)     │  │  │
                                           │  │  └──────────────────┘  │  │
                                           │  │                        │  │
                                           │  │  ┌──────────────────┐  │  │
                                           │  │  │  Redis           │  │  │
                                           │  │  │  (Cache/Queue)   │  │  │
                                           │  │  └──────────────────┘  │  │
                                           │  └────────────────────────┘  │
                                           │                              │
                                           │  Customer Data Never Leaves  │
                                           └──────────────────────────────┘
```

## Key Characteristics

### 1. Control Plane in Cloud (SaaS Provider)

**Hosted by Provider**:
- Identity & Access Management (IAM)
- Role-Based Access Control (RBAC)
- Billing and usage metering
- Audit logging and compliance
- Policy enforcement
- Observability dashboard (aggregated metrics only)

**Data Stored**:
- User accounts, emails, roles
- Subscription and billing information
- Aggregated usage metrics (no raw data)
- Audit logs (metadata, not content)

```python
# Example: Control plane stores metadata only
control_plane_db = {
    "accounts": [
        {
            "id": "acct_abc",
            "name": "Acme Corp",
            "status": "active",
            "tier": "enterprise",
            "data_plane_endpoint": "https://acme-data.internal"
        }
    ],
    "principals": [
        {
            "id": "user_123",
            "email": "alice@acme.com",
            "account_id": "acct_abc",
            "roles": ["admin"]
        }
    ],
    "usage_summary": [
        {
            "account_id": "acct_abc",
            "month": "2026-03",
            "api_calls": 125000,
            "llm_tokens": 5000000,
            "storage_gb": 50
            # No actual conversation data, just aggregates
        }
    ]
}
```

### 2. Data Plane On-Premise (Customer)

**Hosted by Customer**:
- All databases (PostgreSQL, ClickHouse)
- Vector stores (Qdrant)
- Graph databases (Neo4j)
- Caches and queues (Redis)
- Business logic services
- Customer data and documents

**Data Stored**:
- Conversations, messages, chat history
- Document content and embeddings
- Knowledge graphs
- User-generated content
- Proprietary data

```python
# Example: Customer data plane stores all content
customer_data_plane = {
    "conversations": [
        {
            "id": "conv_456",
            "project_id": "proj_789",
            "messages": [
                {
                    "role": "user",
                    "content": "SENSITIVE: Customer financial data here..."
                },
                {
                    "role": "assistant",
                    "content": "Analysis of financial data..."
                }
            ]
        }
    ],
    "documents": [
        {
            "id": "doc_001",
            "content": "PROPRIETARY: Trade secrets, internal documents...",
            "embedding": [0.1, 0.2, ...],  # Vector never leaves premises
        }
    ]
}
```

### 3. Secure Communication Channel

**VPN / Private Link**:
- AWS PrivateLink, Azure Private Link, or site-to-site VPN
- Encrypted tunnel between SaaS control plane and customer data plane
- Customer controls firewall rules

**API Contract**:
- Control plane calls customer data plane via well-defined APIs
- mTLS for authentication
- All data stays within customer network

## Communication Flow

### 1. User Authentication (Control Plane)

```
User → SaaS API Gateway → Control Plane IAM
                           ↓
                     Validate credentials
                           ↓
                     Issue JWT with:
                     - tenant_id
                     - principal_id
                     - data_plane_endpoint
                           ↓
                     Return JWT to user
```

### 2. Data Access (Data Plane)

```
User request with JWT
       ↓
SaaS API Gateway (validates JWT)
       ↓
Route to customer data plane endpoint (from JWT)
       ↓
Customer API Gateway (local)
       ↓
Validate JWT signature (shared secret)
       ↓
Execute query on local PostgreSQL/Qdrant/Neo4j
       ↓
Return data to user (via SaaS proxy)
```

### 3. Agent Execution (Orchestration)

```
User: "Analyze customer churn"
       ↓
SaaS Control Plane: Create agent task
       ↓
Send task to customer data plane (gRPC)
       ↓
Customer: Execute agent locally
          - Load data from local PostgreSQL
          - Query local Qdrant for context
          - Call LLM (OpenAI/Anthropic)
          - Generate response
       ↓
Return response to user (aggregated metrics to SaaS)
```

## Setup Guide

### Customer Infrastructure Setup

#### 1. Deploy Data Plane Components

```bash
# Assuming Kubernetes in customer environment
kubectl create namespace cortex-data-plane

# Deploy PostgreSQL
helm install postgres bitnami/postgresql \
  --namespace cortex-data-plane \
  --set auth.database=cortex \
  --set primary.persistence.size=100Gi

# Deploy Redis
helm install redis bitnami/redis \
  --namespace cortex-data-plane \
  --set master.persistence.size=10Gi

# Deploy Qdrant
helm install qdrant qdrant/qdrant \
  --namespace cortex-data-plane \
  --set persistence.size=50Gi

# Deploy Neo4j
helm install neo4j neo4j/neo4j \
  --namespace cortex-data-plane \
  --set neo4j.resources.requests.memory=8Gi
```

#### 2. Deploy Application Services

```yaml
# k8s/data-plane-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cortex-api
  namespace: cortex-data-plane
spec:
  replicas: 3
  selector:
    matchLabels:
      app: cortex-api
  template:
    metadata:
      labels:
        app: cortex-api
    spec:
      containers:
      - name: api
        image: cortex-ai:latest
        env:
        # Local database connections
        - name: DATABASE_URL
          value: "postgresql://postgres:5432/cortex"
        - name: REDIS_URL
          value: "redis://redis:6379"
        - name: QDRANT_URL
          value: "http://qdrant:6333"
        - name: NEO4J_URL
          value: "bolt://neo4j:7687"

        # Control plane connection
        - name: CONTROL_PLANE_URL
          value: "https://control-plane.cortex-ai.com"
        - name: TENANT_ID
          valueFrom:
            secretKeyRef:
              name: cortex-secrets
              key: tenant_id
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: cortex-secrets
              key: api_key

        ports:
        - containerPort: 8000
```

#### 3. Configure Network Connectivity

**Option A: AWS PrivateLink**

```terraform
# Customer VPC
resource "aws_vpc_endpoint_service" "cortex_data_plane" {
  acceptance_required        = true
  network_load_balancer_arns = [aws_lb.cortex_nlb.arn]

  allowed_principals = [
    "arn:aws:iam::PROVIDER_ACCOUNT_ID:root"  # SaaS provider
  ]
}

# SaaS provider creates endpoint in their VPC
resource "aws_vpc_endpoint" "cortex_connection" {
  vpc_id             = aws_vpc.saas_vpc.id
  service_name       = "com.amazonaws.vpce.us-east-1.vpce-svc-CUSTOMER_SERVICE_ID"
  vpc_endpoint_type  = "Interface"

  security_group_ids = [aws_security_group.cortex_endpoint.id]
  subnet_ids         = aws_subnet.private[*].id
}
```

**Option B: Site-to-Site VPN**

```bash
# Customer configures VPN gateway
# SaaS provider establishes VPN tunnel
# Firewall rules allow specific IPs/ports
```

### Provider Control Plane Setup

#### 1. Register Customer Tenant

```python
# POST /admin/tenants/register
{
    "tenant_id": "acct_abc",
    "name": "Acme Corp",
    "deployment_model": "hybrid",
    "data_plane_endpoint": "https://acme-data.internal",
    "control_plane_access": {
        "api_key": "encrypted_key_here",
        "certificate": "mTLS_cert"
    }
}

# Backend creates:
# 1. Tenant account in control plane DB
# 2. Shared JWT signing secret
# 3. Network routing config
# 4. Billing account
```

#### 2. Configure Routing

```python
# api_gateway/routing.py

async def route_request(request: Request):
    """Route requests to appropriate data plane."""

    # Extract tenant from JWT
    token = extract_jwt(request)
    tenant_id = token["tenant_id"]

    # Lookup data plane endpoint
    tenant = await control_plane_db.get_tenant(tenant_id)
    data_plane_url = tenant.data_plane_endpoint

    if tenant.deployment_model == "hybrid":
        # Proxy to customer data plane
        async with httpx.AsyncClient() as client:
            response = await client.request(
                method=request.method,
                url=f"{data_plane_url}{request.url.path}",
                headers=request.headers,
                content=await request.body()
            )
        return response
    else:
        # Handle locally (SaaS tenant)
        return await local_handler(request)
```

## Data Flow Examples

### Example 1: Chat Request

```python
# 1. User sends chat request
POST https://api.cortex-ai.com/api/v1/projects/proj_123/chat
Authorization: Bearer JWT_TOKEN
{
    "message": "Analyze customer sentiment"
}

# 2. SaaS API Gateway validates JWT
# - Check signature (control plane)
# - Extract tenant_id = "acct_abc"
# - Lookup data_plane_endpoint = "https://acme-data.internal"

# 3. SaaS Gateway proxies to customer data plane
POST https://acme-data.internal/api/v1/projects/proj_123/chat
Authorization: Bearer JWT_TOKEN
{
    "message": "Analyze customer sentiment"
}

# 4. Customer data plane executes locally
# - Validate JWT (shared secret with SaaS)
# - Load conversation from local PostgreSQL
# - Query local Qdrant for relevant documents
# - Execute agent locally
# - Call LLM (OpenAI/Anthropic from customer network)
# - Store results in local PostgreSQL

# 5. Response returns to user
{
    "conversation_id": "conv_456",
    "response": "Based on recent feedback...",
    "token_usage": {...}
}

# 6. Customer data plane sends metrics to SaaS (aggregated only)
POST https://control-plane.cortex-ai.com/api/v1/metrics
{
    "tenant_id": "acct_abc",
    "event": "chat_completion",
    "tokens": 1500,
    "duration_ms": 3200
    # No message content sent!
}
```

### Example 2: Document Upload

```python
# 1. User uploads document
POST https://api.cortex-ai.com/api/v1/projects/proj_123/documents
Authorization: Bearer JWT_TOKEN
Content-Type: multipart/form-data

# 2. SaaS Gateway routes to customer data plane
# Document is never stored on SaaS side

# 3. Customer data plane receives document
# - Store in local PostgreSQL
# - Generate embedding (local OpenAI call or local model)
# - Store embedding in local Qdrant
# - Extract entities, store in local Neo4j

# 4. SaaS receives success confirmation
{
    "document_id": "doc_789",
    "status": "processed"
}

# 5. SaaS billing records storage usage (size only, not content)
{
    "tenant_id": "acct_abc",
    "usage_type": "storage_gb",
    "quantity": 0.5
}
```

## Security Considerations

### 1. JWT Sharing

```python
# Shared JWT secret between SaaS and customer
# SaaS signs JWTs, customer validates

# SaaS control plane
jwt_token = jwt.encode(
    {
        "tenant_id": "acct_abc",
        "principal_id": "user_123",
        "data_plane_endpoint": "https://acme-data.internal",
        "exp": datetime.utcnow() + timedelta(hours=1)
    },
    shared_secret,  # Shared with customer
    algorithm="HS256"
)

# Customer data plane
try:
    payload = jwt.decode(jwt_token, shared_secret, algorithms=["HS256"])
    tenant_id = payload["tenant_id"]
    # Proceed with request
except jwt.InvalidTokenError:
    raise Unauthorized()
```

### 2. mTLS for Service-to-Service

```python
# SaaS → Customer communication uses client certificates
import httpx

async def call_customer_data_plane(tenant: Tenant, endpoint: str, data: dict):
    """Call customer data plane with mTLS."""

    cert_path = f"/certs/tenant_{tenant.id}_client.pem"
    key_path = f"/certs/tenant_{tenant.id}_client.key"

    async with httpx.AsyncClient(
        cert=(cert_path, key_path),
        verify=tenant.server_ca_cert  # Verify customer's server cert
    ) as client:
        response = await client.post(
            f"{tenant.data_plane_endpoint}{endpoint}",
            json=data
        )

    return response.json()
```

### 3. Network Isolation

```bash
# Customer firewall rules: Only allow SaaS provider IPs
# iptables example
iptables -A INPUT -p tcp --dport 443 -s SAAS_PROVIDER_IP -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j DROP

# Kubernetes NetworkPolicy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-saas-only
spec:
  podSelector:
    matchLabels:
      app: cortex-api
  ingress:
  - from:
    - ipBlock:
        cidr: SAAS_PROVIDER_IP/32
```

### 4. Data Residency Compliance

```python
# Customer controls all data
# Audit: Verify no data leaves premises

# Customer can inspect outgoing traffic
# Only metrics/aggregates should go to SaaS

class OutboundDataMonitor:
    """Monitor data leaving customer premises."""

    async def inspect_outgoing(self, data: dict):
        """Ensure no sensitive data in outgoing calls."""

        # Allowed: Aggregated metrics
        allowed_fields = {"tenant_id", "event", "tokens", "duration_ms"}

        # Block: Any message content, PII
        blocked_fields = {"content", "message", "document", "email"}

        for field in blocked_fields:
            if field in data:
                raise SecurityError(f"Blocked sensitive field: {field}")

        return data
```

## Monitoring & Observability

### SaaS Control Plane Metrics

```python
# Control plane tracks:
# - Authentication events
# - Authorization decisions
# - Billing events
# - Aggregated usage

control_plane_metrics = {
    "auth_requests_total": Counter(["tenant_id", "success"]),
    "billing_events_total": Counter(["tenant_id", "usage_type"]),
    "data_plane_latency": Histogram(["tenant_id"]),
}
```

### Customer Data Plane Metrics

```python
# Customer tracks all detailed metrics locally
# Optionally sends aggregated metrics to SaaS

customer_metrics = {
    "conversations_total": Counter(["project_id"]),
    "messages_total": Counter(["role"]),
    "llm_tokens_used": Counter(["model"]),
    "vector_searches_total": Counter(["collection"]),
}

# Send aggregates to SaaS for billing
async def report_usage_to_saas():
    """Send aggregated usage to SaaS billing."""

    usage_summary = {
        "tenant_id": tenant_id,
        "period": "2026-03-13",
        "metrics": {
            "api_calls": 5000,
            "llm_tokens": 250000,
            "storage_gb": 75
        }
    }

    await httpx.post(
        "https://control-plane.cortex-ai.com/api/v1/usage",
        json=usage_summary,
        headers={"Authorization": f"Bearer {api_key}"}
    )
```

## Disaster Recovery

### Customer Responsibility

- **Data backups**: Customer owns and manages
- **Infrastructure redundancy**: Customer configures HA
- **Disaster recovery**: Customer's DR plan

### Provider Responsibility

- **Control plane backups**: Provider manages
- **Auth service availability**: SLA-backed uptime
- **Billing data integrity**: Provider ensures accuracy

### Failure Scenarios

**Scenario 1: SaaS control plane down**
- Customer data plane can continue operating
- Cached auth tokens allow continued access
- New authentication blocked until control plane recovers

**Scenario 2: Customer data plane down**
- Only affects that tenant
- SaaS control plane and other tenants unaffected
- Customer responsible for recovery

**Scenario 3: Network connectivity lost**
- Customer data plane isolated (can't auth new users)
- Cached sessions can continue
- No new billing events recorded (reconciled later)

## Best Practices

1. **Minimize data sent to SaaS**: Only send aggregated metrics, never raw content
2. **Encrypt all traffic**: TLS for API, mTLS for service-to-service
3. **Audit outgoing traffic**: Customer should monitor all data leaving premises
4. **Cache aggressively**: Reduce dependency on control plane
5. **Plan for network failure**: Graceful degradation when SaaS unreachable
6. **Document data flows**: Clear mapping of what data lives where
7. **Regular security audits**: Verify no data leakage
8. **Compliance validation**: Ensure deployment meets regulatory requirements

## Migration Path

### From Pure SaaS to Hybrid

1. **Deploy customer data plane** infrastructure
2. **Export tenant data** from SaaS databases
3. **Import to customer data plane** with validation
4. **Update routing** in SaaS gateway
5. **Cutover** (point tenant to new data plane)
6. **Verify** and monitor

### From On-Premise to Hybrid

1. **Keep existing on-premise** data plane
2. **Integrate with SaaS control plane** (IAM, billing)
3. **Configure network** connectivity (PrivateLink/VPN)
4. **Update auth** to use SaaS JWT
5. **Send aggregated metrics** to SaaS billing

---

**Next**: [On-Premise Deployment](./onprem-deployment.md) | [Back to SaaS](./saas-deployment.md)
