# AI Control Plane

## Overview

The **AI Control Plane** manages policies and governance specific to AI operations including LLM quotas, cost tracking, model routing, and AI-specific compliance.

```
┌──────────────────────────────────────────────────────────────┐
│                AI Control Plane                               │
├──────────────┬───────────────┬──────────────────────────────┤
│ LLM Quotas   │ Cost Tracking │ AI Policies                   │
│              │               │                                │
│ • Token      │ • LLM costs   │ • Model routing                │
│   limits     │ • Cache       │ • Content filtering            │
│ • Model      │   savings     │ • Tool permissions             │
│   access     │ • Attribution │ • Agent quotas                 │
└──────────────┴───────────────┴────────────────────────────────┘
```

---

## LLM Quotas & Limits

### Quota Types

```python
class LLMQuotaType(str, Enum):
    TOKENS_MONTHLY = "tokens_monthly"
    COST_MONTHLY = "cost_monthly"
    REQUESTS_PER_MINUTE = "requests_per_minute"
    CONCURRENT_AGENTS = "concurrent_agents"
    EMBEDDINGS_MONTHLY = "embeddings_monthly"
    VECTOR_STORAGE_GB = "vector_storage_gb"
    GRAPH_NODES = "graph_nodes"

# LLM quota limits by tier
LLM_QUOTA_LIMITS = {
    SubscriptionTier.FREE: {
        LLMQuotaType.TOKENS_MONTHLY: 100_000,
        LLMQuotaType.COST_MONTHLY: 5.0,  # USD
        LLMQuotaType.REQUESTS_PER_MINUTE: 10,
        LLMQuotaType.CONCURRENT_AGENTS: 1,
        LLMQuotaType.EMBEDDINGS_MONTHLY: 10_000,
        LLMQuotaType.VECTOR_STORAGE_GB: 1,
        LLMQuotaType.GRAPH_NODES: 1_000,
    },
    SubscriptionTier.PRO: {
        LLMQuotaType.TOKENS_MONTHLY: 1_000_000,
        LLMQuotaType.COST_MONTHLY: 50.0,
        LLMQuotaType.REQUESTS_PER_MINUTE: 100,
        LLMQuotaType.CONCURRENT_AGENTS: 5,
        LLMQuotaType.EMBEDDINGS_MONTHLY: 100_000,
        LLMQuotaType.VECTOR_STORAGE_GB: 10,
        LLMQuotaType.GRAPH_NODES: 10_000,
    },
    SubscriptionTier.TEAM: {
        LLMQuotaType.TOKENS_MONTHLY: 10_000_000,
        LLMQuotaType.COST_MONTHLY: 500.0,
        LLMQuotaType.REQUESTS_PER_MINUTE: 500,
        LLMQuotaType.CONCURRENT_AGENTS: 20,
        LLMQuotaType.EMBEDDINGS_MONTHLY: 1_000_000,
        LLMQuotaType.VECTOR_STORAGE_GB: 100,
        LLMQuotaType.GRAPH_NODES: 100_000,
    },
    SubscriptionTier.ENTERPRISE: {
        LLMQuotaType.TOKENS_MONTHLY: -1,  # Unlimited
        LLMQuotaType.COST_MONTHLY: -1,
        LLMQuotaType.REQUESTS_PER_MINUTE: 2000,
        LLMQuotaType.CONCURRENT_AGENTS: 100,
        LLMQuotaType.EMBEDDINGS_MONTHLY: -1,
        LLMQuotaType.VECTOR_STORAGE_GB: 1000,
        LLMQuotaType.GRAPH_NODES: -1,
    },
}
```

### LLM Quota Enforcement

```python
class LLMQuotaService:
    """Enforce LLM quotas."""

    async def check_quota(
        self,
        tenant_id: str,
        quota_type: LLMQuotaType,
        requested_amount: int = 1
    ) -> bool:
        """Check if tenant has remaining quota."""

        # Get account tier
        account = await db.query(Account).filter_by(uid=tenant_id).first()
        tier = account.subscription_tier

        # Get quota limit
        limit = LLM_QUOTA_LIMITS[tier][quota_type]

        # Unlimited quota
        if limit == -1:
            return True

        # Get current usage
        usage = await self._get_usage(tenant_id, quota_type)

        return (usage + requested_amount) <= limit

    async def _get_usage(self, tenant_id: str, quota_type: LLMQuotaType) -> float:
        """Get current usage for LLM quota type."""

        if quota_type == LLMQuotaType.TOKENS_MONTHLY:
            # Query analytics database
            result = await clickhouse.query("""
                SELECT SUM(total_tokens) FROM llm_usage_events
                WHERE tenant_id = :tenant_id
                  AND timestamp >= date_trunc('month', NOW())
            """, {"tenant_id": tenant_id})
            return result[0][0] or 0

        elif quota_type == LLMQuotaType.COST_MONTHLY:
            # Query cost tracking
            result = await clickhouse.query("""
                SELECT SUM(cost_usd) FROM llm_usage_events
                WHERE tenant_id = :tenant_id
                  AND timestamp >= date_trunc('month', NOW())
            """, {"tenant_id": tenant_id})
            return float(result[0][0] or 0.0)

        elif quota_type == LLMQuotaType.REQUESTS_PER_MINUTE:
            # Query Redis for recent requests
            key = f"llm_requests:{tenant_id}:minute"
            count = await redis.get(key)
            return int(count) if count else 0

        elif quota_type == LLMQuotaType.CONCURRENT_AGENTS:
            # Query active agents
            return await redis.scard(f"active_agents:{tenant_id}")

        elif quota_type == LLMQuotaType.VECTOR_STORAGE_GB:
            # Query Qdrant collection size
            collection_info = await qdrant_client.get_collection(
                collection_name=f"embeddings_{tenant_id}"
            )
            # Estimate: 1536 dims * 4 bytes * point count
            size_bytes = collection_info.points_count * 1536 * 4
            return size_bytes / (1024**3)

        elif quota_type == LLMQuotaType.GRAPH_NODES:
            # Query Neo4j node count
            result = await neo4j.query(tenant_id, """
                MATCH (n {tenant_id: $tenant_id})
                RETURN COUNT(n) AS count
            """)
            return result[0]['count']

# Usage in LLM Gateway
async def llm_complete(prompt: str, tenant_id: str, model: str):
    """LLM completion with quota check."""

    # Estimate tokens (rough approximation)
    estimated_tokens = len(prompt.split()) * 1.3

    # Check token quota
    has_quota = await llm_quota_service.check_quota(
        tenant_id=tenant_id,
        quota_type=LLMQuotaType.TOKENS_MONTHLY,
        requested_amount=int(estimated_tokens)
    )

    if not has_quota:
        raise QuotaExceeded("Monthly token quota exceeded. Upgrade your plan.")

    # Check rate limit
    has_rate_quota = await llm_quota_service.check_quota(
        tenant_id=tenant_id,
        quota_type=LLMQuotaType.REQUESTS_PER_MINUTE
    )

    if not has_rate_quota:
        raise RateLimitExceeded("LLM rate limit exceeded. Try again in a minute.")

    # Proceed with LLM call
    response = await llm_provider.complete(prompt, model)

    # Track usage
    await llm_usage_tracker.record(
        tenant_id=tenant_id,
        model=model,
        prompt_tokens=response.usage.prompt_tokens,
        completion_tokens=response.usage.completion_tokens,
        cost_usd=calculate_cost(response.usage, model)
    )

    return response
```

---

## LLM Cost Tracking

### Cost Calculation

```python
class LLMCostCalculator:
    """Calculate LLM costs based on model and token usage."""

    # Pricing per 1M tokens (as of 2024)
    MODEL_PRICING = {
        "gpt-4o": {"input": 5.0, "output": 15.0},
        "gpt-4o-mini": {"input": 0.15, "output": 0.6},
        "claude-opus-4": {"input": 15.0, "output": 75.0},
        "claude-sonnet-4": {"input": 3.0, "output": 15.0},
        "claude-haiku-4": {"input": 0.25, "output": 1.25},
    }

    def calculate_cost(
        self,
        model: str,
        prompt_tokens: int,
        completion_tokens: int
    ) -> float:
        """Calculate cost in USD."""

        if model not in self.MODEL_PRICING:
            # Default pricing for unknown models
            pricing = {"input": 1.0, "output": 2.0}
        else:
            pricing = self.MODEL_PRICING[model]

        input_cost = (prompt_tokens / 1_000_000) * pricing["input"]
        output_cost = (completion_tokens / 1_000_000) * pricing["output"]

        return input_cost + output_cost

cost_calculator = LLMCostCalculator()

# Usage
cost = cost_calculator.calculate_cost(
    model="gpt-4o",
    prompt_tokens=1000,
    completion_tokens=500
)
# Result: (1000/1M * 5.0) + (500/1M * 15.0) = $0.0125
```

### Cost Attribution

```python
class LLMUsageTracker:
    """Track LLM usage for cost attribution."""

    async def record(
        self,
        tenant_id: str,
        model: str,
        prompt_tokens: int,
        completion_tokens: int,
        cost_usd: float,
        cached: bool = False,
        metadata: dict = None
    ):
        """Record LLM usage event."""

        # Record to ClickHouse for analytics
        await clickhouse.insert("llm_usage_events", [{
            "timestamp": datetime.utcnow(),
            "tenant_id": tenant_id,
            "model": model,
            "prompt_tokens": prompt_tokens,
            "completion_tokens": completion_tokens,
            "total_tokens": prompt_tokens + completion_tokens,
            "cost_usd": cost_usd if not cached else 0.0,
            "cost_saved": cost_usd if cached else 0.0,
            "cached": cached,
            "metadata": json.dumps(metadata or {})
        }])

        # Update Redis counter for rate limiting
        key = f"llm_requests:{tenant_id}:minute"
        await redis.incr(key)
        await redis.expire(key, 60)

    async def get_monthly_cost(self, tenant_id: str) -> dict:
        """Get monthly cost breakdown."""

        result = await clickhouse.query("""
            SELECT
                model,
                SUM(total_tokens) AS total_tokens,
                SUM(cost_usd) AS total_cost,
                SUM(cost_saved) AS cache_savings,
                COUNT(*) AS request_count,
                COUNT(CASE WHEN cached = 1 THEN 1 END) AS cache_hits
            FROM llm_usage_events
            WHERE tenant_id = :tenant_id
              AND timestamp >= date_trunc('month', NOW())
            GROUP BY model
            ORDER BY total_cost DESC
        """, {"tenant_id": tenant_id})

        return {
            "by_model": [
                {
                    "model": row[0],
                    "tokens": row[1],
                    "cost": float(row[2]),
                    "cache_savings": float(row[3]),
                    "requests": row[4],
                    "cache_hits": row[5]
                }
                for row in result
            ],
            "total_cost": sum(float(row[2]) for row in result),
            "total_savings": sum(float(row[3]) for row in result),
            "cache_hit_rate": sum(row[5] for row in result) / sum(row[4] for row in result) if result else 0
        }
```

---

## Model Routing & Policies

### Smart Model Routing

```python
class ModelRouter:
    """Route requests to appropriate model based on policies."""

    async def route(
        self,
        prompt: str,
        tenant_id: str,
        context: dict = None
    ) -> str:
        """
        Select best model based on:
        - Tenant's budget constraints
        - Prompt complexity
        - Latency requirements
        - Cost optimization
        """

        # Get tenant policy
        policy = await self._get_routing_policy(tenant_id)

        if policy.strategy == "cost_optimized":
            # Use cheapest model that meets quality threshold
            return await self._select_cheapest_model(prompt, policy.min_quality)

        elif policy.strategy == "latency_optimized":
            # Use fastest model
            return "gpt-4o-mini"  # Fastest

        elif policy.strategy == "quality_optimized":
            # Use best model
            return "claude-opus-4"  # Best quality

        elif policy.strategy == "balanced":
            # Balance cost vs quality
            complexity = self._estimate_complexity(prompt)

            if complexity < 0.3:
                return "gpt-4o-mini"  # Simple prompt
            elif complexity < 0.7:
                return "gpt-4o"  # Medium complexity
            else:
                return "claude-sonnet-4"  # Complex prompt

    def _estimate_complexity(self, prompt: str) -> float:
        """Estimate prompt complexity (0-1)."""

        # Simple heuristic based on length and keywords
        length_score = min(len(prompt) / 5000, 1.0)

        complex_keywords = ["analyze", "explain", "compare", "evaluate", "synthesize"]
        keyword_score = sum(1 for kw in complex_keywords if kw in prompt.lower()) / len(complex_keywords)

        return (length_score * 0.5) + (keyword_score * 0.5)

class RoutingPolicy(Base):
    """Model routing policy per tenant."""
    __tablename__ = "routing_policies"

    id = Column(Integer, primary_key=True)
    tenant_id = Column(String(255), unique=True, nullable=False)
    strategy = Column(String(50), default="balanced")  # 'cost_optimized', 'latency_optimized', 'quality_optimized', 'balanced'
    min_quality = Column(Float, default=0.7)  # Minimum quality threshold (0-1)
    max_cost_per_request = Column(Float, default=0.1)  # Max cost in USD
    preferred_models = Column(JSON, nullable=True)  # List of allowed models
```

---

## Content Filtering & Safety

### Content Moderation

```python
class ContentModerator:
    """Filter unsafe content in prompts and responses."""

    UNSAFE_CATEGORIES = [
        "hate", "harassment", "violence", "self-harm",
        "sexual", "illegal", "pii"  # Personally Identifiable Information
    ]

    async def check_content(
        self,
        text: str,
        tenant_id: str
    ) -> dict:
        """
        Check content for policy violations.

        Returns:
            {
                "safe": bool,
                "categories": ["hate", "violence"],
                "confidence": 0.95
            }
        """

        # Use OpenAI moderation API
        response = await openai.moderations.create(input=text)

        flagged_categories = [
            category
            for category, flagged in response.results[0].categories.dict().items()
            if flagged
        ]

        # Log moderation event
        if flagged_categories:
            await audit_service.log_action(
                tenant_id=tenant_id,
                principal_id=None,
                action="content.flagged",
                resource_type="moderation",
                resource_id=str(uuid.uuid4()),
                metadata={
                    "categories": flagged_categories,
                    "text_length": len(text)
                }
            )

        return {
            "safe": len(flagged_categories) == 0,
            "categories": flagged_categories,
            "confidence": max(response.results[0].category_scores.dict().values())
        }

# Usage in LLM Gateway
async def llm_complete_with_moderation(
    prompt: str,
    tenant_id: str,
    model: str
):
    """LLM completion with content moderation."""

    # Check prompt
    prompt_check = await content_moderator.check_content(prompt, tenant_id)

    if not prompt_check["safe"]:
        raise ContentViolation(
            f"Prompt violates content policy: {', '.join(prompt_check['categories'])}"
        )

    # Complete
    response = await llm_provider.complete(prompt, model)

    # Check response
    response_check = await content_moderator.check_content(response.text, tenant_id)

    if not response_check["safe"]:
        # Log violation and return sanitized response
        logger.warning(
            "llm_response_flagged",
            tenant_id=tenant_id,
            categories=response_check["categories"]
        )

        return "I cannot provide that response due to content policy."

    return response.text
```

### PII Detection

```python
class PIIDetector:
    """Detect and redact personally identifiable information."""

    PII_PATTERNS = {
        "email": r"\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b",
        "phone": r"\b\d{3}[-.]?\d{3}[-.]?\d{4}\b",
        "ssn": r"\b\d{3}-\d{2}-\d{4}\b",
        "credit_card": r"\b\d{4}[-\s]?\d{4}[-\s]?\d{4}[-\s]?\d{4}\b",
    }

    async def detect_pii(self, text: str) -> list[dict]:
        """Detect PII in text."""

        detected = []

        for pii_type, pattern in self.PII_PATTERNS.items():
            matches = re.finditer(pattern, text)

            for match in matches:
                detected.append({
                    "type": pii_type,
                    "value": match.group(),
                    "start": match.start(),
                    "end": match.end()
                })

        return detected

    async def redact_pii(self, text: str) -> str:
        """Redact PII from text."""

        redacted = text

        for pii_type, pattern in self.PII_PATTERNS.items():
            redacted = re.sub(pattern, f"[REDACTED_{pii_type.upper()}]", redacted)

        return redacted

# Usage
async def store_conversation_with_pii_redaction(message: str, tenant_id: str):
    """Store conversation message with PII redaction."""

    # Detect PII
    pii_detected = await pii_detector.detect_pii(message)

    if pii_detected:
        # Log PII detection
        logger.warning(
            "pii_detected",
            tenant_id=tenant_id,
            pii_types=[p["type"] for p in pii_detected]
        )

        # Redact PII
        redacted_message = await pii_detector.redact_pii(message)

        # Store redacted version
        await db.insert(Message, {
            "content": redacted_message,
            "tenant_id": tenant_id,
            "pii_redacted": True
        })
    else:
        # Store original
        await db.insert(Message, {
            "content": message,
            "tenant_id": tenant_id,
            "pii_redacted": False
        })
```

---

## Tool Permissions & Agent Quotas

### Tool Permission Matrix

```python
class ToolPermission(Base):
    """Tool execution permissions."""
    __tablename__ = "tool_permissions"

    id = Column(Integer, primary_key=True)
    tenant_id = Column(String(255), nullable=False)
    tool_name = Column(String(100), nullable=False)
    allowed = Column(Boolean, default=False)
    max_executions_per_day = Column(Integer, default=100)

    __table_args__ = (
        UniqueConstraint("tenant_id", "tool_name"),
    )

class ToolPermissionService:
    """Manage tool permissions."""

    async def check_permission(
        self,
        tenant_id: str,
        tool_name: str
    ) -> bool:
        """Check if tenant can execute tool."""

        permission = await db.query(ToolPermission).filter_by(
            tenant_id=tenant_id,
            tool_name=tool_name
        ).first()

        if not permission:
            # Default: deny unless explicitly allowed
            return False

        if not permission.allowed:
            return False

        # Check daily quota
        today = datetime.utcnow().date()
        executions_today = await redis.get(
            f"tool_executions:{tenant_id}:{tool_name}:{today}"
        )

        if executions_today and int(executions_today) >= permission.max_executions_per_day:
            return False

        return True

    async def record_execution(
        self,
        tenant_id: str,
        tool_name: str
    ):
        """Record tool execution."""

        today = datetime.utcnow().date()
        key = f"tool_executions:{tenant_id}:{tool_name}:{today}"

        await redis.incr(key)
        await redis.expire(key, 86400)  # 24 hours

# Usage in MCP Gateway
async def mcp_invoke_tool(tool_name: str, context: Context):
    """Invoke tool with permission check."""

    tenant_id = context.get("tenant_id")

    # Check permission
    has_permission = await tool_permission_service.check_permission(
        tenant_id=tenant_id,
        tool_name=tool_name
    )

    if not has_permission:
        raise PermissionDenied(f"Tool '{tool_name}' not allowed or quota exceeded")

    # Execute tool
    result = await mcp_server.invoke_tool(tool_name, context)

    # Record execution
    await tool_permission_service.record_execution(tenant_id, tool_name)

    return result
```

---

## AI Compliance & Governance

### Model Version Tracking

```python
class ModelVersion(Base):
    """Track model versions for compliance."""
    __tablename__ = "model_versions"

    id = Column(Integer, primary_key=True)
    model_name = Column(String(100), nullable=False)
    version = Column(String(50), nullable=False)
    deployed_at = Column(DateTime(timezone=True), server_default=func.now())
    deprecated_at = Column(DateTime(timezone=True), nullable=True)
    compliance_certified = Column(Boolean, default=False)
    notes = Column(Text, nullable=True)

    __table_args__ = (
        UniqueConstraint("model_name", "version"),
    )
```

### AI Decision Audit Trail

```python
class AIDecisionLog(Base):
    """Audit trail for AI decisions (for regulated industries)."""
    __tablename__ = "ai_decision_logs"

    id = Column(Integer, primary_key=True)
    tenant_id = Column(String(255), nullable=False)
    decision_id = Column(String(255), unique=True, nullable=False)
    model_name = Column(String(100), nullable=False)
    model_version = Column(String(50), nullable=False)
    input_data = Column(Text, nullable=False)  # Prompt
    output_data = Column(Text, nullable=False)  # Response
    confidence = Column(Float, nullable=True)
    explanation = Column(Text, nullable=True)  # Model reasoning
    human_reviewed = Column(Boolean, default=False)
    timestamp = Column(DateTime(timezone=True), server_default=func.now())

    __table_args__ = (
        Index("idx_ai_decision_tenant_timestamp", "tenant_id", "timestamp"),
    )

# Usage in critical AI decisions
async def ai_credit_decision(application_data: dict, tenant_id: str):
    """AI-based credit decision with full audit trail."""

    # Generate decision
    prompt = f"Analyze credit application: {json.dumps(application_data)}"
    response = await llm_provider.complete(prompt, model="gpt-4o")

    decision_id = str(uuid.uuid4())

    # Log decision
    await db.insert(AIDecisionLog, {
        "tenant_id": tenant_id,
        "decision_id": decision_id,
        "model_name": "gpt-4o",
        "model_version": "2024-08-06",
        "input_data": prompt,
        "output_data": response.text,
        "confidence": 0.85,
        "explanation": "Based on credit history, income, and debt-to-income ratio",
        "human_reviewed": False
    })

    return {
        "decision_id": decision_id,
        "decision": response.text
    }
```

---

**Next**: [Traditional Control Plane](./02a-traditional-control-plane.md) | [Shared Control Plane](./02c-control-plane-shared.md) | [Back to Overview](../00-parallel-stacks-overview.md)
