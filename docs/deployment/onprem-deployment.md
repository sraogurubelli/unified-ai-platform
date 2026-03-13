# On-Premise Deployment Model

## Overview

Pure on-premise deployment is a fully self-contained installation within the customer's environment. All components (control plane, services, data plane) run on customer infrastructure with no external dependencies.

**Ideal for**: Government, defense, high-security enterprises, air-gapped environments, complete data sovereignty.

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│         Customer Environment (Air-gapped / On-Premise)        │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │              Load Balancer (Internal)                    │ │
│  │          (HAProxy, NGINX, F5, NetScaler)                 │ │
│  └──────────────────────┬───────────────────────────────────┘ │
│                         ▼                                     │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │         Gateway Layer (Dual Gateway)                      │ │
│  ├────────────────────────┬─────────────────────────────────┤ │
│  │   API Gateway          │    MCP Gateway                   │ │
│  │   (Customer-managed)   │    (Customer-managed)            │ │
│  └────────────────────────┴─────────────────────────────────┘ │
│                         ▼                                     │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │      Control Plane (Customer-managed)                     │ │
│  │  - IAM / RBAC (Local LDAP/AD integration)                │ │
│  │  - Observability (Prometheus, Grafana)                   │ │
│  │  - Policy Enforcement                                    │ │
│  │  - Optional: Local billing/usage tracking               │ │
│  └──────────────────────────────────────────────────────────┘ │
│                         ▼                                     │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │         Service Layer (All services on-prem)             │ │
│  │  - Platform Services (Auth, Orgs, Projects)              │ │
│  │  - AI Services (Agents, RAG, Knowledge Graphs)           │ │
│  │  - Analytics Services (Usage, Performance)               │ │
│  └──────────────────────────────────────────────────────────┘ │
│                         ▼                                     │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │        Data Plane (Customer infrastructure)              │ │
│  │                                                           │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌────────────────┐ │ │
│  │  │ PostgreSQL   │  │ ClickHouse   │  │  Redis         │ │ │
│  │  │ (High Avail) │  │ (Analytics)  │  │  (Cache/Queue) │ │ │
│  │  └──────────────┘  └──────────────┘  └────────────────┘ │ │
│  │                                                           │ │
│  │  ┌──────────────┐  ┌──────────────┐                      │ │
│  │  │ Qdrant       │  │  Neo4j       │                      │ │
│  │  │ (Vectors)    │  │  (Graphs)    │                      │ │
│  │  └──────────────┘  └──────────────┘                      │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                               │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │   LLM Providers (Customer choice)                         │ │
│  │   - Self-hosted models (Ollama, vLLM)                    │ │
│  │   - Enterprise API (OpenAI Azure, Anthropic AWS)         │ │
│  │   - Hybrid (local + cloud with egress control)           │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                               │
│  All data and processing stays within customer premises      │
└──────────────────────────────────────────────────────────────┘
```

## Key Characteristics

### 1. Complete Self-Sufficiency

- **No external dependencies**: Platform runs fully disconnected
- **Customer-managed**: All updates, backups, scaling
- **Data sovereignty**: 100% data control, no data egress
- **Compliance**: Meets strictest regulatory requirements

### 2. Enterprise Integration

- **LDAP/Active Directory**: Integrate with existing identity systems
- **SAML/SSO**: Enterprise single sign-on
- **Network policies**: Fit within customer security architecture
- **Monitoring**: Integrate with existing Prometheus, Grafana, Splunk

### 3. Flexible LLM Options

- **Self-hosted models**: Llama, Mistral via Ollama, vLLM, TGI
- **Enterprise cloud**: OpenAI Azure, Anthropic on AWS (with egress control)
- **Hybrid**: Local models for sensitive data, cloud for general tasks
- **No vendor lock-in**: Swap providers without data migration

## Infrastructure Setup

### Hardware Requirements

#### Minimum Configuration

```
Control Plane:
- 3 nodes (HA control plane)
- 8 CPU, 16 GB RAM each
- 100 GB SSD each

Application Servers:
- 5 nodes (API, agents, workers)
- 16 CPU, 32 GB RAM each
- 200 GB SSD each

Database Cluster (PostgreSQL):
- 3 nodes (primary + 2 replicas)
- 8 CPU, 32 GB RAM each
- 500 GB SSD (expandable)

Vector Store (Qdrant):
- 3 nodes (HA cluster)
- 8 CPU, 32 GB RAM each
- 1 TB SSD

Graph Database (Neo4j):
- 3 nodes (causal cluster)
- 8 CPU, 16 GB RAM each
- 500 GB SSD

Analytics (ClickHouse):
- 3 nodes (cluster)
- 16 CPU, 64 GB RAM each
- 2 TB SSD

Cache/Queue (Redis):
- 3 nodes (cluster)
- 4 CPU, 16 GB RAM each
- 50 GB SSD

Storage:
- NFS or object storage (MinIO) for file uploads
- 5 TB minimum (expandable)

Total: ~27 nodes, ~200 CPU cores, ~600 GB RAM, ~10 TB storage
```

#### Recommended Production Configuration

```
Scale above by 2-3x for:
- High availability (multiple availability zones)
- Performance headroom
- Future growth
```

### Kubernetes Deployment

#### 1. Cluster Setup

```bash
# Using kubeadm, Rancher, or OpenShift
# Example: 3 control plane nodes, 24 worker nodes

kubeadm init --control-plane-endpoint="k8s-cp-lb:6443" \
  --upload-certs \
  --pod-network-cidr=10.244.0.0/16

# Install CNI (Calico, Flannel, etc.)
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

# Join additional control plane and worker nodes
```

#### 2. Storage Configuration

```yaml
# storage-class.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: bulk-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

#### 3. Deploy Core Services

```bash
# Create namespaces
kubectl create namespace cortex-control-plane
kubectl create namespace cortex-services
kubectl create namespace cortex-data-plane

# Deploy PostgreSQL (StatefulSet with PVCs)
helm install postgresql bitnami/postgresql \
  --namespace cortex-data-plane \
  --set primary.persistence.storageClass=fast-ssd \
  --set primary.persistence.size=500Gi \
  --set readReplicas.replicaCount=2 \
  --set readReplicas.persistence.size=500Gi \
  --set metrics.enabled=true

# Deploy Redis Cluster
helm install redis bitnami/redis-cluster \
  --namespace cortex-data-plane \
  --set cluster.nodes=6 \
  --set persistence.storageClass=fast-ssd \
  --set persistence.size=50Gi

# Deploy Qdrant
helm install qdrant qdrant/qdrant \
  --namespace cortex-data-plane \
  --set replicaCount=3 \
  --set persistence.size=1Ti \
  --set persistence.storageClass=fast-ssd

# Deploy Neo4j
helm install neo4j neo4j/neo4j \
  --namespace cortex-data-plane \
  --set neo4j.edition=enterprise \
  --set neo4j.acceptLicenseAgreement=yes \
  --set core.numberOfServers=3 \
  --set core.persistentVolume.size=500Gi

# Deploy ClickHouse (optional)
helm install clickhouse altinity/clickhouse-operator \
  --namespace cortex-data-plane
```

#### 4. Deploy Application

```yaml
# cortex-api-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cortex-api
  namespace: cortex-services
spec:
  replicas: 5
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
        image: cortex-ai:v0.1.0  # Customer's private registry
        imagePullPolicy: Always
        resources:
          requests:
            cpu: "2000m"
            memory: "4Gi"
          limits:
            cpu: "4000m"
            memory: "8Gi"
        env:
        - name: DEPLOYMENT_MODEL
          value: "onprem"
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: url
        - name: REDIS_URL
          value: "redis://redis-cluster:6379"
        - name: QDRANT_URL
          value: "http://qdrant:6333"
        - name: NEO4J_URL
          value: "bolt://neo4j:7687"
        - name: LDAP_URL
          value: "ldaps://ldap.company.internal:636"  # Customer LDAP
        - name: LOG_LEVEL
          value: "INFO"
        ports:
        - containerPort: 8000
          name: http
        livenessProbe:
          httpGet:
            path: /health
            port: http
          initialDelaySeconds: 30
        readinessProbe:
          httpGet:
            path: /ready
            port: http
          initialDelaySeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: cortex-api
  namespace: cortex-services
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 8000
  selector:
    app: cortex-api
```

### Authentication Integration

#### LDAP/Active Directory

```python
# cortex/platform/auth/ldap.py

from ldap3 import Server, Connection, ALL, NTLM

class LDAPAuthProvider:
    """Integrate with customer LDAP/AD."""

    def __init__(self, url: str, base_dn: str):
        self.server = Server(url, get_info=ALL, use_ssl=True)
        self.base_dn = base_dn

    async def authenticate(self, username: str, password: str) -> dict:
        """Authenticate user against LDAP."""

        try:
            # Bind with user credentials
            user_dn = f"uid={username},{self.base_dn}"
            conn = Connection(
                self.server,
                user=user_dn,
                password=password,
                auto_bind=True
            )

            # Fetch user attributes
            conn.search(
                user_dn,
                '(objectClass=person)',
                attributes=['mail', 'displayName', 'memberOf']
            )

            if conn.entries:
                entry = conn.entries[0]
                return {
                    "username": username,
                    "email": str(entry.mail),
                    "display_name": str(entry.displayName),
                    "groups": [str(g) for g in entry.memberOf]
                }

            raise ValueError("User not found")

        except Exception as e:
            raise AuthenticationError(f"LDAP auth failed: {e}")

    async def sync_user(self, ldap_user: dict) -> Principal:
        """Create or update user in local database."""

        principal = await db.query(Principal).filter_by(
            email=ldap_user["email"]
        ).first()

        if not principal:
            # Create new principal
            principal = Principal(
                uid=f"user_{uuid.uuid4().hex[:12]}",
                email=ldap_user["email"],
                display_name=ldap_user["display_name"],
                principal_type=PrincipalType.USER,
                # No password stored (LDAP auth only)
            )
            await db.add(principal)
            await db.commit()

        return principal
```

#### SAML SSO

```python
# cortex/platform/auth/saml.py

from onelogin.saml2.auth import OneLogin_Saml2_Auth

class SAMLAuthProvider:
    """SAML 2.0 SSO integration."""

    def __init__(self, settings: dict):
        self.settings = settings

    async def initiate_login(self, request: Request) -> str:
        """Initiate SAML login flow."""

        auth = OneLogin_Saml2_Auth(request, self.settings)
        return auth.login()  # Redirect to IdP

    async def handle_callback(self, request: Request) -> dict:
        """Handle SAML assertion from IdP."""

        auth = OneLogin_Saml2_Auth(request, self.settings)
        auth.process_response()

        if not auth.is_authenticated():
            raise AuthenticationError("SAML authentication failed")

        # Extract user attributes
        attributes = auth.get_attributes()

        return {
            "email": attributes['email'][0],
            "display_name": attributes['displayName'][0],
            "groups": attributes.get('groups', [])
        }
```

### Monitoring & Observability

#### Prometheus & Grafana

```yaml
# monitoring/prometheus.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: cortex-control-plane
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      evaluation_interval: 15s

    scrape_configs:
    - job_name: 'cortex-api'
      kubernetes_sd_configs:
      - role: pod
        namespaces:
          names:
          - cortex-services
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        regex: cortex-api
        action: keep

    - job_name: 'postgres'
      static_configs:
      - targets: ['postgresql-metrics:9187']

    - job_name: 'redis'
      static_configs:
      - targets: ['redis-metrics:9121']

    - job_name: 'qdrant'
      static_configs:
      - targets: ['qdrant:6333']

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: cortex-control-plane
spec:
  replicas: 2
  selector:
    matchLabels:
      app: prometheus
  template:
    spec:
      containers:
      - name: prometheus
        image: prom/prometheus:latest
        args:
        - '--config.file=/etc/prometheus/prometheus.yml'
        - '--storage.tsdb.path=/prometheus'
        - '--storage.tsdb.retention.time=30d'
        volumeMounts:
        - name: config
          mountPath: /etc/prometheus
        - name: data
          mountPath: /prometheus
      volumes:
      - name: config
        configMap:
          name: prometheus-config
      - name: data
        persistentVolumeClaim:
          claimName: prometheus-data

---
# Grafana deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: cortex-control-plane
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: grafana
        image: grafana/grafana:latest
        env:
        - name: GF_SECURITY_ADMIN_PASSWORD
          valueFrom:
            secretKeyRef:
              name: grafana-admin
              key: password
        - name: GF_AUTH_LDAP_ENABLED
          value: "true"
        volumeMounts:
        - name: grafana-storage
          mountPath: /var/lib/grafana
```

### LLM Provider Options

#### Option 1: Self-Hosted Models (Ollama)

```bash
# Deploy Ollama on GPU nodes
kubectl label nodes gpu-node-1 gpu=true

# ollama-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ollama
  namespace: cortex-services
spec:
  replicas: 2
  template:
    spec:
      nodeSelector:
        gpu: "true"
      containers:
      - name: ollama
        image: ollama/ollama:latest
        resources:
          limits:
            nvidia.com/gpu: 1
        volumeMounts:
        - name: models
          mountPath: /root/.ollama
      volumes:
      - name: models
        persistentVolumeClaim:
          claimName: ollama-models

# Pull models
kubectl exec -it ollama-xxx -- ollama pull llama3
kubectl exec -it ollama-xxx -- ollama pull mistral
```

```python
# cortex/orchestration/providers/ollama.py

class OllamaProvider:
    """Self-hosted Ollama provider."""

    def __init__(self, base_url: str = "http://ollama:11434"):
        self.base_url = base_url

    async def chat_completion(self, messages: list, model: str = "llama3"):
        async with httpx.AsyncClient() as client:
            response = await client.post(
                f"{self.base_url}/api/chat",
                json={
                    "model": model,
                    "messages": messages,
                    "stream": False
                }
            )
        return response.json()
```

#### Option 2: Enterprise Cloud (Air-gapped)

```python
# Use OpenAI Azure or Anthropic AWS within customer network
# With egress firewall rules to allow only specific endpoints

class EnterpriseCloudProvider:
    """Enterprise LLM with egress control."""

    def __init__(self):
        # OpenAI Azure endpoint (customer's Azure tenant)
        self.azure_endpoint = os.getenv("AZURE_OPENAI_ENDPOINT")
        self.azure_key = os.getenv("AZURE_OPENAI_KEY")

        # Or Anthropic on AWS Bedrock (customer's AWS account)
        self.bedrock_client = boto3.client(
            'bedrock-runtime',
            region_name='us-east-1'
        )

    async def chat_completion(self, messages: list):
        # Route to customer's enterprise LLM account
        # Data never leaves customer's cloud tenant
        ...
```

#### Option 3: Hybrid (Local + Cloud)

```python
# Sensitive queries → local model
# General queries → cloud model (with approval)

class HybridLLMRouter:
    """Route queries based on sensitivity."""

    def __init__(self):
        self.local_provider = OllamaProvider()
        self.cloud_provider = EnterpriseCloudProvider()

    async def route_query(self, messages: list, sensitivity: str):
        # Check for sensitive data
        if self._is_sensitive(messages) or sensitivity == "high":
            return await self.local_provider.chat_completion(messages)
        else:
            return await self.cloud_provider.chat_completion(messages)

    def _is_sensitive(self, messages: list) -> bool:
        """Detect sensitive data (PII, secrets, etc.)."""
        # Implement sensitivity detection
        ...
```

## Network Architecture

### Ingress Configuration

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: cortex-ingress
  namespace: cortex-services
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - cortex.company.internal
    secretName: cortex-tls-cert  # Customer's internal CA
  rules:
  - host: cortex.company.internal
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: cortex-api
            port:
              number: 80
      - path: /mcp
        pathType: Prefix
        backend:
          service:
            name: cortex-mcp-gateway
            port:
              number: 80
```

### Network Policies

```yaml
# network-policies.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: cortex-api-policy
  namespace: cortex-services
spec:
  podSelector:
    matchLabels:
      app: cortex-api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: cortex-services
    - podSelector:
        matchLabels:
          app: nginx-ingress
  egress:
  # Allow database access
  - to:
    - namespaceSelector:
        matchLabels:
          name: cortex-data-plane
    ports:
    - protocol: TCP
      port: 5432  # PostgreSQL
    - protocol: TCP
      port: 6379  # Redis
    - protocol: TCP
      port: 6333  # Qdrant
    - protocol: TCP
      port: 7687  # Neo4j

  # Allow LDAP access
  - to:
    - podSelector: {}
    ports:
    - protocol: TCP
      port: 636  # LDAPS

  # Allow LLM access (if external)
  - to:
    - podSelector: {}
    ports:
    - protocol: TCP
      port: 443
```

## Backup & Disaster Recovery

### Backup Strategy

```bash
# PostgreSQL backups (pg_dump + WAL archiving)
# Daily full backups, continuous WAL archiving

# Velero for Kubernetes backup
velero install \
  --provider aws \
  --bucket cortex-backups \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1

# Schedule daily backups
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --include-namespaces cortex-services,cortex-data-plane

# Backup Qdrant collections
curl -X POST http://qdrant:6333/collections/embeddings/snapshots

# Backup Neo4j
kubectl exec -it neo4j-0 -- neo4j-admin dump --database=neo4j --to=/backups/neo4j.dump
```

### Disaster Recovery Plan

```
RTO: 4 hours
RPO: 1 hour

1. Restore Kubernetes cluster (from IaC)
2. Restore databases (PostgreSQL, Redis, etc.)
3. Restore application state (Velero)
4. Verify data integrity
5. Resume operations
```

## Security Hardening

### 1. Image Scanning

```bash
# Scan container images before deployment
trivy image cortex-ai:v0.1.0

# Policy: No HIGH/CRITICAL vulnerabilities allowed
```

### 2. Secrets Management

```bash
# Use Sealed Secrets or Vault
kubectl create secret generic db-credentials \
  --from-literal=url='postgresql://...' \
  --dry-run=client -o yaml | \
  kubeseal -o yaml > sealed-secret.yaml

kubectl apply -f sealed-secret.yaml
```

### 3. RBAC (Kubernetes)

```yaml
# Least privilege for service accounts
apiVersion: rbac.authorization.k8s.io/v1
kind:Role
metadata:
  name: cortex-api-role
  namespace: cortex-services
rules:
- apiGroups: [""]
  resources: ["pods", "configmaps"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cortex-api-binding
  namespace: cortex-services
subjects:
- kind: ServiceAccount
  name: cortex-api
roleRef:
  kind: Role
  name: cortex-api-role
  apiGroup: rbac.authorization.k8s.io
```

### 4. Pod Security Standards

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: cortex-services
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

## Upgrade Strategy

### Rolling Updates

```bash
# Update application image
kubectl set image deployment/cortex-api \
  api=cortex-ai:v0.2.0 \
  -n cortex-services

# Monitor rollout
kubectl rollout status deployment/cortex-api -n cortex-services

# Rollback if issues
kubectl rollout undo deployment/cortex-api -n cortex-services
```

### Database Migrations

```python
# Run migrations before application upgrade
# Use Alembic for schema changes

# Pre-upgrade: Backup database
# Run migrations: alembic upgrade head
# Deploy new code: kubectl apply -f ...
# Verify: Health checks pass
```

## Cost Optimization

### Resource Rightsizing

```bash
# Monitor actual resource usage
kubectl top pods -n cortex-services

# Adjust resource requests/limits based on metrics
# Use Vertical Pod Autoscaler for recommendations
```

### Storage Tiering

```
Hot data (recent conversations): Fast SSD
Warm data (30-90 days): Standard SSD
Cold data (>90 days): Archive storage (S3-compatible)
```

## Compliance & Audit

### Audit Logging

```python
# All API calls logged with:
# - Timestamp
# - Principal (user/service account)
# - Action (create, read, update, delete)
# - Resource type and ID
# - Outcome (success/failure)

audit_logger.log(
    timestamp=datetime.utcnow(),
    principal_id=principal.id,
    action="DELETE",
    resource_type="conversation",
    resource_id=conversation_id,
    outcome="SUCCESS",
    ip_address=request.client.host
)
```

### Compliance Reports

```bash
# Generate compliance reports
# - Access logs
# - Data residency verification
# - Encryption status
# - Backup verification
```

## Best Practices

1. **High Availability**: Deploy across multiple availability zones
2. **Regular Backups**: Automated daily backups with off-site storage
3. **Monitoring**: Comprehensive metrics and alerting
4. **Security Patches**: Regular updates and vulnerability scanning
5. **Capacity Planning**: Monitor growth, scale proactively
6. **Documentation**: Runbooks for all operational procedures
7. **Training**: Ensure ops team is trained on the platform
8. **Disaster Recovery Testing**: Regular DR drills
9. **Change Management**: Controlled rollout process
10. **Audit Trail**: Comprehensive logging for compliance

## Support Model

### Customer Responsibilities

- Infrastructure provisioning and management
- Backups and disaster recovery
- Security patching and updates
- Monitoring and incident response
- User support and training

### Vendor Support (Optional)

- Installation assistance
- Configuration guidance
- Troubleshooting support
- Version upgrades
- Feature requests and bug fixes

---

**Complete Deployment Guide**: [SaaS](./saas-deployment.md) | [Hybrid](./hybrid-deployment.md) | On-Premise (this doc)
