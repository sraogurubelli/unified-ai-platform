# AI Platform Services: LLM Gateway, Chat & Memory

## Overview

This layer provides core AI platform services that enable intelligent routing, conversation management, and memory systems. These services bridge the gap between raw AI capabilities (LLMs, embeddings) and application-level features.

```
┌──────────────────────────────────────────────────────────────┐
│              AI Platform Services Layer                       │
├────────────────┬───────────────┬──────────────┬──────────────┤
│ LLM Gateway    │ Unified Chat  │ Memory       │ Agent        │
│                │               │ Management   │ Registry     │
│ - Routing      │ - Multi-turn  │              │              │
│ - Caching      │ - Session mgmt│ - Short-term │ - Discovery  │
│ - Rate limiting│ - Streaming   │ - Long-term  │ - Versioning │
│ - Fallback     │ - Threading   │ - Semantic   │ - Capability │
└────────────────┴───────────────┴──────────────┴──────────────┘
```

**Key Principle**: These services are designed to work across all deployment models (SaaS, Hybrid, On-Premise) with consistent interfaces.

## LLM Gateway

### Architecture

The LLM Gateway is a unified routing layer that abstracts multiple LLM providers and adds intelligent routing, caching, rate limiting, and cost optimization.

```
User Request
     ↓
┌──────────────────────────────────────────────────┐
│            LLM Gateway                            │
│                                                  │
│  ┌────────────────────────────────────────────┐ │
│  │  Request Router                             │ │
│  │  - Provider selection                      │ │
│  │  - Cost optimization                       │ │
│  │  - Latency-based routing                  │ │
│  └────────────────────────────────────────────┘ │
│                     ↓                            │
│  ┌────────────────────────────────────────────┐ │
│  │  Semantic Cache (Optional)                 │ │
│  │  - Vector similarity on prompts            │ │
│  │  - Qdrant-based caching                   │ │
│  └────────────────────────────────────────────┘ │
│                     ↓                            │
│  ┌────────────────────────────────────────────┐ │
│  │  Rate Limiter                              │ │
│  │  - Per-tenant quotas                       │ │
│  │  - Per-model limits                        │ │
│  │  - Redis-based                             │ │
│  └────────────────────────────────────────────┘ │
│                     ↓                            │
│  ┌────────────────────────────────────────────┐ │
│  │  Provider Pool                             │ │
│  │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐     │ │
│  │  │OpenAI│ │Anthro│ │Google│ │Vertex│     │ │
│  │  │      │ │pic   │ │      │ │  AI  │     │ │
│  │  └──────┘ └──────┘ └──────┘ └──────┘     │ │
│  │  - Health checks                          │ │
│  │  - Circuit breakers                       │ │
│  └────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────┘
     ↓
LLM Response
```

### Design Decisions

**Library vs Service**:
- **Phase 1**: Library component (current cortex-ai approach)
  - Pros: Lower latency, simpler deployment, easier debugging
  - Cons: Harder to scale independently
- **Phase 2**: Optional gateway service for scale
  - Pros: Independent scaling, centralized monitoring, cross-language support
  - Cons: Added network hop, operational complexity

**Recommended**: Start with library, evolve to service when needed (>1M requests/day)

### Implementation (Library Pattern)

```python
# cortex/orchestration/llm.py (cortex-ai reference)

from langchain_core.language_models import BaseChatModel
from cortex.orchestration.config import ModelConfig

class LLMClient:
    """
    LLM Client with provider abstraction and optional gateway routing.

    Supports:
    - Direct providers: OpenAI, Anthropic, Google, Vertex AI
    - Gateway routing: Routes through centralized HTTP endpoint
    - Provider auto-detection from model name
    """

    def __init__(self, config: ModelConfig):
        self.config = config

    def get_model(self) -> BaseChatModel:
        """Create chat model with routing logic."""
        if self.config.use_gateway:
            return self._create_gateway_model()
        return self._create_direct_model()

    def _create_gateway_model(self) -> BaseChatModel:
        """
        Route through LLM Gateway (OpenAI-compatible HTTP endpoint).

        Gateway metadata passed as headers:
        - X-Tenant-ID: For cost attribution
        - X-Source: Request source (web, api, internal)
        - X-Product: Product name
        """
        from langchain_openai import ChatOpenAI

        return ChatOpenAI(
            model=self.gateway_model_name,  # online/{provider}/{model}
            base_url=self.config.gateway_url,
            api_key="not-needed",  # Gateway handles auth
            temperature=self.config.temperature,
            max_tokens=self.config.max_tokens,
            default_headers={
                "X-Tenant-ID": self.config.tenant_id,
                "X-Source": self.config.source,
                "X-Product": self.config.product,
            }
        )

    def _create_direct_model(self) -> BaseChatModel:
        """Create model directly from provider."""
        provider = self.config.provider or self._detect_provider()

        if provider == "openai":
            from langchain_openai import ChatOpenAI
            return ChatOpenAI(
                model=self.config.model,
                temperature=self.config.temperature,
                max_tokens=self.config.max_tokens,
            )

        elif provider == "anthropic":
            from langchain_anthropic import ChatAnthropic
            return ChatAnthropic(
                model=self.config.model,
                temperature=self.config.temperature,
                max_tokens=self.config.max_tokens,
            )

        elif provider == "google":
            from langchain_google_genai import ChatGoogleGenerativeAI
            return ChatGoogleGenerativeAI(
                model=self.config.model,
                temperature=self.config.temperature,
                max_output_tokens=self.config.max_tokens,
            )

        elif provider == "anthropic_vertex":
            from langchain_google_vertexai.model_garden import ChatAnthropicVertex
            return ChatAnthropicVertex(
                model=self.config.model,
                project=self.config.project,
                location=self.config.location,
            )

        else:
            raise ValueError(f"Unknown provider: {provider}")

    def _detect_provider(self) -> str:
        """Auto-detect provider from model name."""
        model = self.config.model.lower()

        if model.startswith("gpt"):
            return "openai"
        elif model.startswith("claude"):
            return "anthropic"
        elif model.startswith("gemini"):
            return "google"
        elif "@" in model:  # Vertex AI format (model@version)
            return "anthropic_vertex"
        else:
            return "openai"  # Default

    @property
    def gateway_model_name(self) -> str:
        """Format model name for gateway: online/{provider}/{model}."""
        model = self.config.model
        if model.startswith("online/"):
            return model

        provider = self.config.provider or self._detect_provider()
        if provider == "anthropic_vertex":
            provider = "anthropic"  # Normalize for gateway

        return f"online/{provider}/{model}"
```

### Intelligent Routing

```python
# cortex/orchestration/llm_gateway/router.py (proposed)

from dataclasses import dataclass
from enum import Enum

class RoutingStrategy(str, Enum):
    """Routing strategy for LLM requests."""
    COST_OPTIMIZED = "cost"      # Cheapest available model
    LATENCY_OPTIMIZED = "latency"  # Fastest response time
    QUALITY_OPTIMIZED = "quality"   # Best quality (user-defined)
    ROUND_ROBIN = "round_robin"    # Distribute evenly
    FALLBACK = "fallback"          # Try primary, fall back on failure

@dataclass
class ProviderHealth:
    """Health status for an LLM provider."""
    provider: str
    is_healthy: bool
    latency_p50: float  # 50th percentile latency (ms)
    latency_p99: float  # 99th percentile latency (ms)
    error_rate: float   # % of requests that failed
    cost_per_1k_tokens: float

class LLMRouter:
    """Intelligent router for LLM requests."""

    def __init__(self, strategy: RoutingStrategy = RoutingStrategy.COST_OPTIMIZED):
        self.strategy = strategy
        self.provider_health: dict[str, ProviderHealth] = {}

    async def route(
        self,
        prompt: str,
        model_requirements: dict = None
    ) -> tuple[str, str]:
        """
        Route request to best provider.

        Args:
            prompt: User prompt
            model_requirements: Optional constraints (min_quality, max_cost, etc.)

        Returns:
            (provider, model) tuple
        """
        # Get available providers meeting requirements
        candidates = self._filter_providers(model_requirements)

        # Select based on strategy
        if self.strategy == RoutingStrategy.COST_OPTIMIZED:
            return min(candidates, key=lambda p: p.cost_per_1k_tokens)

        elif self.strategy == RoutingStrategy.LATENCY_OPTIMIZED:
            return min(candidates, key=lambda p: p.latency_p99)

        elif self.strategy == RoutingStrategy.QUALITY_OPTIMIZED:
            # User-defined quality ranking
            return self._select_by_quality(candidates, model_requirements)

        elif self.strategy == RoutingStrategy.FALLBACK:
            # Try primary, fallback to secondary
            primary = candidates[0]
            if primary.is_healthy and primary.error_rate < 0.05:
                return primary
            return candidates[1]  # Fallback

        else:  # ROUND_ROBIN
            return self._round_robin_select(candidates)

    def _filter_providers(self, requirements: dict) -> list[ProviderHealth]:
        """Filter providers based on requirements."""
        candidates = [
            p for p in self.provider_health.values()
            if p.is_healthy
        ]

        if requirements:
            if "max_cost" in requirements:
                candidates = [
                    p for p in candidates
                    if p.cost_per_1k_tokens <= requirements["max_cost"]
                ]

            if "max_latency" in requirements:
                candidates = [
                    p for p in candidates
                    if p.latency_p99 <= requirements["max_latency"]
                ]

        return candidates
```

### Semantic Caching

```python
# cortex/orchestration/llm_gateway/semantic_cache.py (proposed)

from cortex.rag.vector_store import VectorStore
from cortex.rag.embeddings import EmbeddingService

class SemanticCache:
    """
    Semantic caching for LLM prompts using vector similarity.

    Caches LLM responses and retrieves them when a similar prompt
    is detected (cosine similarity > threshold).
    """

    def __init__(
        self,
        vector_store: VectorStore,
        embedding_service: EmbeddingService,
        similarity_threshold: float = 0.95,
        ttl: int = 3600,  # 1 hour
    ):
        self.vector_store = vector_store
        self.embedding_service = embedding_service
        self.similarity_threshold = similarity_threshold
        self.ttl = ttl

    async def get(
        self,
        prompt: str,
        model: str,
        tenant_id: str
    ) -> str | None:
        """
        Retrieve cached response for similar prompt.

        Args:
            prompt: User prompt
            model: Model name (cache is model-specific)
            tenant_id: Tenant ID for isolation

        Returns:
            Cached response if found, None otherwise
        """
        # Generate embedding for prompt
        prompt_embedding = await self.embedding_service.embed(prompt)

        # Search for similar prompts in cache
        results = await self.vector_store.search(
            collection_name=f"llm_cache_{tenant_id}",
            query_vector=prompt_embedding,
            filters={"model": model},
            limit=1,
            score_threshold=self.similarity_threshold
        )

        if results:
            cached_result = results[0]

            # Check TTL
            if self._is_expired(cached_result.metadata["cached_at"]):
                await self._delete_cache_entry(cached_result.id)
                return None

            return cached_result.payload["response"]

        return None

    async def set(
        self,
        prompt: str,
        response: str,
        model: str,
        tenant_id: str,
        metadata: dict = None
    ):
        """
        Cache LLM response.

        Args:
            prompt: User prompt
            response: LLM response
            model: Model name
            tenant_id: Tenant ID
            metadata: Optional metadata (tokens, cost, etc.)
        """
        # Generate embedding
        prompt_embedding = await self.embedding_service.embed(prompt)

        # Store in vector database
        await self.vector_store.upsert(
            collection_name=f"llm_cache_{tenant_id}",
            points=[
                {
                    "id": f"{hash(prompt)}_{model}",
                    "vector": prompt_embedding,
                    "payload": {
                        "prompt": prompt,
                        "response": response,
                        "model": model,
                        "cached_at": datetime.utcnow().isoformat(),
                        "metadata": metadata or {}
                    }
                }
            ]
        )

    def _is_expired(self, cached_at: str) -> bool:
        """Check if cache entry is expired."""
        cached_time = datetime.fromisoformat(cached_at)
        return (datetime.utcnow() - cached_time).total_seconds() > self.ttl
```

### Rate Limiting

```python
# cortex/orchestration/llm_gateway/rate_limiter.py (proposed)

import redis.asyncio as redis
import time
from dataclasses import dataclass

@dataclass
class RateLimit:
    """Rate limit configuration."""
    requests_per_minute: int
    tokens_per_minute: int
    requests_per_day: int

# Tier-based rate limits
TIER_LIMITS = {
    "free": RateLimit(10, 10000, 1000),
    "pro": RateLimit(100, 100000, 50000),
    "team": RateLimit(500, 500000, 250000),
    "enterprise": RateLimit(5000, 5000000, 10000000),
}

class LLMRateLimiter:
    """
    Rate limiter for LLM requests using Redis sliding window.

    Enforces limits at multiple levels:
    - Per tenant
    - Per model
    - Global
    """

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client

    async def check_limit(
        self,
        tenant_id: str,
        model: str,
        tier: str,
        estimated_tokens: int = 0
    ) -> tuple[bool, str]:
        """
        Check if request is within rate limits.

        Args:
            tenant_id: Tenant ID
            model: Model name
            tier: Subscription tier
            estimated_tokens: Estimated token count for request

        Returns:
            (allowed: bool, reason: str) tuple
        """
        limits = TIER_LIMITS.get(tier, TIER_LIMITS["free"])

        # Check request rate (per minute)
        request_key = f"ratelimit:{tenant_id}:requests:minute"
        request_count = await self._sliding_window_count(request_key, 60)

        if request_count >= limits.requests_per_minute:
            return False, f"Rate limit exceeded: {limits.requests_per_minute} requests/minute"

        # Check token rate (per minute)
        if estimated_tokens > 0:
            token_key = f"ratelimit:{tenant_id}:tokens:minute"
            token_count = await self._sliding_window_sum(token_key, 60)

            if token_count + estimated_tokens > limits.tokens_per_minute:
                return False, f"Token rate limit exceeded: {limits.tokens_per_minute} tokens/minute"

        # Check daily limit
        daily_key = f"ratelimit:{tenant_id}:requests:day"
        daily_count = await self._sliding_window_count(daily_key, 86400)

        if daily_count >= limits.requests_per_day:
            return False, f"Daily limit exceeded: {limits.requests_per_day} requests/day"

        return True, ""

    async def record_usage(
        self,
        tenant_id: str,
        model: str,
        tokens_used: int
    ):
        """Record usage after successful request."""
        now = time.time()

        # Increment request count
        request_key = f"ratelimit:{tenant_id}:requests:minute"
        await self.redis.zadd(request_key, {str(now): now})
        await self.redis.expire(request_key, 60)

        daily_key = f"ratelimit:{tenant_id}:requests:day"
        await self.redis.zadd(daily_key, {str(now): now})
        await self.redis.expire(daily_key, 86400)

        # Increment token count
        token_key = f"ratelimit:{tenant_id}:tokens:minute"
        await self.redis.zadd(token_key, {f"{now}": tokens_used})
        await self.redis.expire(token_key, 60)

    async def _sliding_window_count(self, key: str, window: int) -> int:
        """Count requests in sliding window."""
        now = time.time()
        window_start = now - window

        # Remove old entries
        await self.redis.zremrangebyscore(key, 0, window_start)

        # Count current window
        return await self.redis.zcard(key)

    async def _sliding_window_sum(self, key: str, window: int) -> int:
        """Sum values in sliding window."""
        now = time.time()
        window_start = now - window

        # Remove old entries
        await self.redis.zremrangebyscore(key, 0, window_start)

        # Sum scores in current window
        entries = await self.redis.zrange(key, 0, -1, withscores=True)
        return sum(score for _, score in entries)
```

### Health Checks & Circuit Breakers

```python
# cortex/orchestration/llm_gateway/health.py (proposed)

from dataclasses import dataclass
from enum import Enum
import time

class CircuitState(str, Enum):
    CLOSED = "closed"      # Normal operation
    OPEN = "open"          # Too many failures, reject requests
    HALF_OPEN = "half_open"  # Testing if service recovered

@dataclass
class CircuitBreaker:
    """Circuit breaker for LLM provider."""
    provider: str
    state: CircuitState = CircuitState.CLOSED
    failure_count: int = 0
    failure_threshold: int = 5  # Open after 5 failures
    reset_timeout: int = 60     # Try again after 60 seconds
    last_failure_time: float = 0

    def should_allow_request(self) -> bool:
        """Check if request should be allowed."""
        if self.state == CircuitState.CLOSED:
            return True

        if self.state == CircuitState.OPEN:
            # Check if enough time has passed to try again
            if time.time() - self.last_failure_time > self.reset_timeout:
                self.state = CircuitState.HALF_OPEN
                return True
            return False

        if self.state == CircuitState.HALF_OPEN:
            # Allow limited requests to test recovery
            return True

        return False

    def record_success(self):
        """Record successful request."""
        if self.state == CircuitState.HALF_OPEN:
            # Recovery confirmed, close circuit
            self.state = CircuitState.CLOSED
            self.failure_count = 0

    def record_failure(self):
        """Record failed request."""
        self.failure_count += 1
        self.last_failure_time = time.time()

        if self.failure_count >= self.failure_threshold:
            self.state = CircuitState.OPEN

class ProviderHealthMonitor:
    """Monitor health of LLM providers."""

    def __init__(self):
        self.circuit_breakers: dict[str, CircuitBreaker] = {}
        self.latency_samples: dict[str, list[float]] = {}

    async def check_health(self, provider: str) -> bool:
        """Check if provider is healthy."""
        breaker = self.circuit_breakers.get(provider)
        if not breaker:
            self.circuit_breakers[provider] = CircuitBreaker(provider=provider)
            return True

        return breaker.should_allow_request()

    async def record_request(
        self,
        provider: str,
        latency_ms: float,
        success: bool
    ):
        """Record request result."""
        breaker = self.circuit_breakers.get(provider)
        if not breaker:
            self.circuit_breakers[provider] = CircuitBreaker(provider=provider)
            breaker = self.circuit_breakers[provider]

        if success:
            breaker.record_success()
        else:
            breaker.record_failure()

        # Track latency
        if provider not in self.latency_samples:
            self.latency_samples[provider] = []

        self.latency_samples[provider].append(latency_ms)

        # Keep only recent samples (last 100)
        if len(self.latency_samples[provider]) > 100:
            self.latency_samples[provider] = self.latency_samples[provider][-100:]

    def get_avg_latency(self, provider: str) -> float:
        """Get average latency for provider."""
        samples = self.latency_samples.get(provider, [])
        if not samples:
            return 0.0
        return sum(samples) / len(samples)
```

## Unified Chat Service

### Architecture

The Unified Chat service orchestrates multi-turn conversations across agents, managing session state, message history, and real-time streaming.

```
┌──────────────────────────────────────────────────┐
│           Unified Chat Service                    │
│                                                  │
│  ┌────────────────────────────────────────────┐ │
│  │  Conversation Orchestrator                 │ │
│  │  - Multi-turn management                   │ │
│  │  - Agent routing                           │ │
│  │  - Context injection                       │ │
│  └────────────────────────────────────────────┘ │
│                     ↓                            │
│  ┌────────────────────────────────────────────┐ │
│  │  Message Manager                           │ │
│  │  - History tracking                        │ │
│  │  - Token estimation                        │ │
│  │  - Compression (LLM-based)                 │ │
│  └────────────────────────────────────────────┘ │
│                     ↓                            │
│  ┌────────────────────────────────────────────┐ │
│  │  Session Store                             │ │
│  │  - Checkpointer (PostgreSQL)               │ │
│  │  - Thread state persistence                │ │
│  │  - LangGraph integration                   │ │
│  └────────────────────────────────────────────┘ │
│                     ↓                            │
│  ┌────────────────────────────────────────────┐ │
│  │  Streaming Handler                         │ │
│  │  - SSE streaming                           │ │
│  │  - WebSocket support                       │ │
│  │  - Real-time updates                       │ │
│  └────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────┘
```

### Message Management

```python
# cortex/core/agents/conversation_manager.py (cortex-ai reference)

from dataclasses import dataclass
from enum import Enum
from typing import Any, Dict, List, Union

class MessageRole(str, Enum):
    """Message roles in a conversation."""
    SYSTEM = "system"
    USER = "user"
    ASSISTANT = "assistant"
    TOOL = "tool"

@dataclass
class Message:
    """Base message structure."""
    role: MessageRole
    content: Union[str, List[Dict[str, Any]]]
    source: Optional[str] = None

    def to_dict(self) -> Dict[str, Any]:
        return {
            "role": self.role.value,
            "content": self.content,
            "source": self.source,
        }

class ConversationManager:
    """
    Manages conversation history with token limits and compression.

    Features:
    - Token estimation and tracking
    - Automatic compression when threshold reached
    - LLM-based summarization
    - Tool output size limits
    """

    def __init__(
        self,
        llm_client=None,
        agent_name: str = "cortex",
        token_threshold: int = 100000,
        token_limit: int = 200000,
        summarization_percentage: int = 50,
        max_tool_output_tokens: int = 50000,
    ):
        self.conversation_history: List[Message] = []
        self.llm_client = llm_client
        self.agent_name = agent_name
        self.token_threshold = token_threshold
        self.token_limit = token_limit
        self.summarization_percentage = summarization_percentage
        self.max_tool_output_tokens = max_tool_output_tokens
        self.current_token_count = 0

    def add_message(
        self,
        role: MessageRole,
        content: Union[str, List, Dict],
        source: Optional[str] = None
    ):
        """Add message to history with token tracking."""
        message = Message(role=role, content=content, source=source)

        # Estimate tokens
        token_count = self.estimate_token_count(content)
        self.current_token_count += token_count

        # Add to history
        self.conversation_history.append(message)

        # Check if compression needed
        if self.current_token_count > self.token_threshold:
            self.compress_history()

    def estimate_token_count(self, content: Union[str, List, Dict]) -> int:
        """
        Estimate token count from content.
        Uses ~3.5 characters per token heuristic.
        """
        if isinstance(content, str):
            return len(content) // 3.5
        elif isinstance(content, (list, dict)):
            content_str = str(content)
            return len(content_str) // 3.5
        return 0

    async def compress_history(self):
        """
        Compress conversation history using LLM summarization.

        Summarizes oldest N% of conversation and replaces with summary.
        Preserves recent messages for context continuity.
        """
        if not self.llm_client:
            # Can't compress without LLM, just truncate
            self._truncate_history()
            return

        # Determine split point
        total_messages = len(self.conversation_history)
        split_index = int(total_messages * (self.summarization_percentage / 100))

        if split_index < 2:
            return  # Not enough to summarize

        # Messages to summarize vs keep
        to_summarize = self.conversation_history[:split_index]
        to_keep = self.conversation_history[split_index:]

        # Generate summary
        summary_prompt = self._build_summary_prompt(to_summarize)
        summary = await self.llm_client.ainvoke(summary_prompt)

        # Replace old messages with summary
        summary_message = Message(
            role=MessageRole.SYSTEM,
            content=f"[Conversation Summary]\n{summary.content}",
            source="compression"
        )

        # Rebuild history
        self.conversation_history = [summary_message] + to_keep

        # Recalculate token count
        self.current_token_count = sum(
            self.estimate_token_count(msg.content)
            for msg in self.conversation_history
        )

    def _build_summary_prompt(self, messages: List[Message]) -> str:
        """Build prompt for summarization."""
        history_text = "\n\n".join([
            f"{msg.role.value}: {msg.content}"
            for msg in messages
        ])

        return f"""
Summarize this conversation history, preserving:
1. Key facts and decisions
2. Tool calls and their results
3. Important context for continuing the conversation

History:
{history_text}

Summary:
"""

    def _truncate_history(self):
        """Truncate history if compression not available."""
        # Keep only recent messages
        keep_count = len(self.conversation_history) // 2
        self.conversation_history = self.conversation_history[-keep_count:]

        # Recalculate tokens
        self.current_token_count = sum(
            self.estimate_token_count(msg.content)
            for msg in self.conversation_history
        )

    def get_history(self) -> List[Dict[str, Any]]:
        """Get conversation history as list of dicts."""
        return [msg.to_dict() for msg in self.conversation_history]

    def clear(self, keep_system: bool = True):
        """Clear conversation history."""
        if keep_system:
            system_messages = [
                msg for msg in self.conversation_history
                if msg.role == MessageRole.SYSTEM
            ]
            self.conversation_history = system_messages
        else:
            self.conversation_history = []

        self.current_token_count = sum(
            self.estimate_token_count(msg.content)
            for msg in self.conversation_history
        )
```

### Session Persistence

```python
# cortex/orchestration/session/checkpointer.py (cortex-ai reference)

from langgraph.checkpoint.base import BaseCheckpointSaver
from langgraph.checkpoint.memory import MemorySaver
import os
import logging

logger = logging.getLogger(__name__)

# Module-level singleton
_pool = None
_checkpointer: BaseCheckpointSaver | None = None

async def open_checkpointer_pool(
    database_url: str | None = None,
    use_memory: bool = False,
) -> None:
    """
    Open checkpoint connection pool.

    Modes:
    - PostgreSQL: Production persistence (AsyncPostgresSaver)
    - In-memory: Development/testing (MemorySaver)

    Args:
        database_url: PostgreSQL connection string
        use_memory: Use in-memory saver instead of PostgreSQL

    Example:
        # Production
        await open_checkpointer_pool(
            database_url="postgresql://user:pass@localhost/cortex"
        )

        # Development
        await open_checkpointer_pool(use_memory=True)
    """
    global _pool, _checkpointer

    if _checkpointer is not None:
        logger.debug("Checkpointer already initialized")
        return

    if use_memory:
        logger.info("Using in-memory checkpoint saver")
        _checkpointer = MemorySaver()
        return

    # PostgreSQL mode
    db_url = database_url or os.getenv("CORTEX_DATABASE_URL")
    if not db_url:
        logger.error("Database URL required for PostgreSQL checkpointer")
        return

    logger.info("Opening PostgreSQL checkpoint pool")

    try:
        from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver
        from psycopg_pool import AsyncConnectionPool

        # Create connection pool
        _pool = AsyncConnectionPool(
            conninfo=db_url,
            max_size=10,
            timeout=30,
        )

        # Create checkpointer
        _checkpointer = AsyncPostgresSaver(_pool)

        # Setup tables (idempotent)
        await _checkpointer.setup()

        logger.info("PostgreSQL checkpointer ready")

    except ImportError as e:
        logger.error(f"Failed to import PostgreSQL dependencies: {e}")
        logger.info("Install with: pip install langgraph-checkpoint-postgres psycopg[pool]")
    except Exception as e:
        logger.error(f"Failed to initialize checkpointer: {e}")

def get_checkpointer() -> BaseCheckpointSaver | None:
    """Get the checkpointer instance."""
    return _checkpointer

async def close_checkpointer_pool():
    """Close checkpoint connection pool."""
    global _pool, _checkpointer

    if _pool:
        await _pool.close()
        _pool = None

    _checkpointer = None
    logger.info("Checkpointer pool closed")

def build_thread_id(agent_name: str, conversation_id: str) -> str:
    """
    Build thread ID for conversation.

    Format: {agent_name}-{conversation_id}
    Provides namespace isolation per agent.
    """
    return f"{agent_name}-{conversation_id}"

async def has_existing_checkpoint(thread_id: str) -> bool:
    """Check if checkpoint exists for thread."""
    if not _checkpointer:
        return False

    try:
        checkpoint = await _checkpointer.aget({"configurable": {"thread_id": thread_id}})
        return checkpoint is not None
    except:
        return False
```

### Streaming Handler

```python
# cortex/core/streaming/stream_writer.py (cortex-ai reference)

import asyncio
import json
from typing import AsyncIterator

class StreamWriter:
    """
    SSE (Server-Sent Events) stream writer.

    Handles real-time streaming of:
    - Text chunks (token-by-token)
    - Tool calls and results
    - Metadata and events
    """

    def __init__(self):
        self._queue: asyncio.Queue = asyncio.Queue()
        self._closed = False

    async def write_text(self, text: str):
        """Write text chunk to stream."""
        if not self._closed:
            await self._queue.put({"type": "text", "content": text})

    async def write_tool_call(self, tool_name: str, args: dict):
        """Write tool call to stream."""
        if not self._closed:
            await self._queue.put({
                "type": "tool_call",
                "tool": tool_name,
                "args": args
            })

    async def write_tool_result(self, tool_name: str, result: str):
        """Write tool result to stream."""
        if not self._closed:
            await self._queue.put({
                "type": "tool_result",
                "tool": tool_name,
                "result": result
            })

    async def write_event(self, event_type: str, data: dict):
        """Write custom event to stream."""
        if not self._closed:
            await self._queue.put({
                "type": "event",
                "event": event_type,
                "data": data
            })

    async def close(self):
        """Close the stream."""
        self._closed = True
        await self._queue.put(None)  # Sentinel

    async def __aiter__(self) -> AsyncIterator[str]:
        """Iterate over stream events."""
        while True:
            item = await self._queue.get()

            if item is None:
                break

            # Format as SSE
            event_type = item.get("type", "message")
            data = json.dumps(item)
            yield f"event: {event_type}\ndata: {data}\n\n"

async def create_streaming_response(stream_writer: StreamWriter):
    """
    Create FastAPI StreamingResponse for SSE.

    Example:
        @app.post("/chat/stream")
        async def chat_stream():
            stream_writer = StreamWriter()

            # Start streaming in background
            asyncio.create_task(stream_agent(stream_writer))

            # Return SSE response
            return await create_streaming_response(stream_writer)
    """
    from fastapi.responses import StreamingResponse

    return StreamingResponse(
        stream_writer.__aiter__(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
            "X-Accel-Buffering": "no",  # Disable nginx buffering
        }
    )
```

## Memory Management

### Architecture

Memory management is critical for AI agents to maintain context across conversations, remember user preferences, and accumulate knowledge over time. The system uses a **3-layer architecture**:

```
┌──────────────────────────────────────────────────────────────┐
│              Memory Management System                         │
├──────────────────┬────────────────────┬──────────────────────┤
│  Short-Term      │  Long-Term         │  Semantic            │
│  Memory          │  Memory            │  Memory              │
│                  │                    │                      │
│  Redis           │  PostgreSQL        │  Qdrant              │
│  (Ephemeral)     │  (Structured)      │  (Vector-based)      │
│                  │                    │                      │
│  - Conversation  │  - User prefs      │  - Past conversations│
│    window        │  - Facts/entities  │  - Similar queries   │
│  - Session state │  - Relationships   │  - Contextual        │
│  - Temp context  │  - Explicit saves  │    retrieval         │
│  TTL: Minutes    │  TTL: Indefinite   │  TTL: Days/Weeks     │
└──────────────────┴────────────────────┴──────────────────────┘
```

### Layer 1: Short-Term Memory (Conversation Window)

Short-term memory holds the immediate conversation context - recent messages that inform the current turn. This is ephemeral and managed in Redis for fast access.

**Characteristics**:
- **Storage**: Redis (in-memory)
- **TTL**: Minutes to hours
- **Scope**: Current conversation session
- **Size**: Limited by token window (typically last 10-20 exchanges)

**Implementation**:

```python
# cortex/platform/memory/short_term.py (proposed)

import redis.asyncio as redis
import json
from typing import List, Dict

class ShortTermMemory:
    """
    Short-term memory for conversation context.

    Stores recent messages in Redis with TTL.
    Automatically manages token limits via ConversationManager.
    """

    def __init__(
        self,
        redis_client: redis.Redis,
        ttl: int = 3600,  # 1 hour
        max_messages: int = 20,
    ):
        self.redis = redis_client
        self.ttl = ttl
        self.max_messages = max_messages

    async def add_message(
        self,
        conversation_id: str,
        role: str,
        content: str,
        metadata: dict = None
    ):
        """Add message to short-term memory."""
        key = f"memory:short:{conversation_id}"

        message = {
            "role": role,
            "content": content,
            "metadata": metadata or {},
            "timestamp": datetime.utcnow().isoformat()
        }

        # Add to list
        await self.redis.rpush(key, json.dumps(message))

        # Trim to max size
        await self.redis.ltrim(key, -self.max_messages, -1)

        # Set TTL
        await self.redis.expire(key, self.ttl)

    async def get_messages(
        self,
        conversation_id: str,
        limit: int = None
    ) -> List[Dict]:
        """Get recent messages from short-term memory."""
        key = f"memory:short:{conversation_id}"

        # Get messages
        messages_json = await self.redis.lrange(key, 0, -1)

        # Parse JSON
        messages = [json.loads(msg) for msg in messages_json]

        # Apply limit
        if limit:
            messages = messages[-limit:]

        return messages

    async def clear(self, conversation_id: str):
        """Clear short-term memory for conversation."""
        key = f"memory:short:{conversation_id}"
        await self.redis.delete(key)
```

### Layer 2: Long-Term Memory (Structured Facts)

Long-term memory stores structured information about users, preferences, facts, and relationships. This is persisted in PostgreSQL with explicit schema.

**Characteristics**:
- **Storage**: PostgreSQL (relational)
- **TTL**: Indefinite (until explicitly deleted)
- **Scope**: Per user across all conversations
- **Size**: Unlimited (grows with usage)

**Schema**:

```python
# cortex/platform/database/models.py (additions)

from sqlalchemy import Column, Integer, String, Text, DateTime, ForeignKey, JSON
from sqlalchemy.orm import relationship

class UserMemory(Base):
    """Long-term memory for users."""
    __tablename__ = "user_memories"

    id = Column(Integer, primary_key=True)
    tenant_id = Column(String(255), nullable=False, index=True)
    principal_id = Column(Integer, ForeignKey("principals.id"), nullable=False)
    memory_type = Column(String(50), nullable=False)  # 'preference', 'fact', 'skill'
    key = Column(String(255), nullable=False)  # e.g., 'language_preference'
    value = Column(Text, nullable=False)  # e.g., 'python'
    context = Column(JSON, nullable=True)  # Additional metadata
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    updated_at = Column(DateTime(timezone=True), onupdate=func.now())

    # Relationships
    principal = relationship("Principal")

    __table_args__ = (
        UniqueConstraint("tenant_id", "principal_id", "memory_type", "key"),
        Index("idx_user_memory_tenant_principal", "tenant_id", "principal_id"),
    )

class EntityMemory(Base):
    """Memory about entities mentioned in conversations."""
    __tablename__ = "entity_memories"

    id = Column(Integer, primary_key=True)
    tenant_id = Column(String(255), nullable=False, index=True)
    principal_id = Column(Integer, ForeignKey("principals.id"), nullable=False)
    entity_type = Column(String(50), nullable=False)  # 'person', 'project', 'company'
    entity_name = Column(String(255), nullable=False)
    attributes = Column(JSON, nullable=False)  # Flexible attributes
    first_mentioned = Column(DateTime(timezone=True), server_default=func.now())
    last_updated = Column(DateTime(timezone=True), onupdate=func.now())

    # Relationships
    principal = relationship("Principal")

    __table_args__ = (
        Index("idx_entity_memory_tenant_principal", "tenant_id", "principal_id"),
        Index("idx_entity_memory_name", "entity_name"),
    )
```

**Service Layer**:

```python
# cortex/platform/memory/long_term.py (proposed)

class LongTermMemory:
    """
    Long-term structured memory storage.

    Stores user preferences, facts, and entity information.
    """

    def __init__(self, db_session: AsyncSession):
        self.db = db_session

    async def save_preference(
        self,
        tenant_id: str,
        principal_id: int,
        key: str,
        value: str,
        context: dict = None
    ):
        """Save user preference."""
        memory = await self.db.query(UserMemory).filter_by(
            tenant_id=tenant_id,
            principal_id=principal_id,
            memory_type="preference",
            key=key
        ).first()

        if memory:
            # Update existing
            memory.value = value
            memory.context = context
        else:
            # Create new
            memory = UserMemory(
                tenant_id=tenant_id,
                principal_id=principal_id,
                memory_type="preference",
                key=key,
                value=value,
                context=context
            )
            await self.db.add(memory)

        await self.db.commit()

    async def get_preference(
        self,
        tenant_id: str,
        principal_id: int,
        key: str
    ) -> str | None:
        """Get user preference."""
        memory = await self.db.query(UserMemory).filter_by(
            tenant_id=tenant_id,
            principal_id=principal_id,
            memory_type="preference",
            key=key
        ).first()

        return memory.value if memory else None

    async def save_fact(
        self,
        tenant_id: str,
        principal_id: int,
        key: str,
        value: str
    ):
        """Save fact about user."""
        memory = UserMemory(
            tenant_id=tenant_id,
            principal_id=principal_id,
            memory_type="fact",
            key=key,
            value=value
        )
        await self.db.add(memory)
        await self.db.commit()

    async def get_facts(
        self,
        tenant_id: str,
        principal_id: int
    ) -> List[Dict]:
        """Get all facts about user."""
        memories = await self.db.query(UserMemory).filter_by(
            tenant_id=tenant_id,
            principal_id=principal_id,
            memory_type="fact"
        ).all()

        return [
            {"key": m.key, "value": m.value, "context": m.context}
            for m in memories
        ]

    async def save_entity(
        self,
        tenant_id: str,
        principal_id: int,
        entity_type: str,
        entity_name: str,
        attributes: dict
    ):
        """Save entity information."""
        entity = await self.db.query(EntityMemory).filter_by(
            tenant_id=tenant_id,
            principal_id=principal_id,
            entity_type=entity_type,
            entity_name=entity_name
        ).first()

        if entity:
            # Merge attributes
            entity.attributes.update(attributes)
        else:
            entity = EntityMemory(
                tenant_id=tenant_id,
                principal_id=principal_id,
                entity_type=entity_type,
                entity_name=entity_name,
                attributes=attributes
            )
            await self.db.add(entity)

        await self.db.commit()

    async def get_entity(
        self,
        tenant_id: str,
        principal_id: int,
        entity_name: str
    ) -> Dict | None:
        """Get entity information."""
        entity = await self.db.query(EntityMemory).filter_by(
            tenant_id=tenant_id,
            principal_id=principal_id,
            entity_name=entity_name
        ).first()

        if entity:
            return {
                "type": entity.entity_type,
                "name": entity.entity_name,
                "attributes": entity.attributes
            }

        return None
```

### Layer 3: Semantic Memory (Vector-based Retrieval)

Semantic memory stores embeddings of past conversations and allows retrieval based on semantic similarity. This enables agents to recall relevant past interactions.

**Characteristics**:
- **Storage**: Qdrant (vector database)
- **TTL**: Days to weeks (with LRU eviction)
- **Scope**: Per tenant, searchable by semantic similarity
- **Size**: Limited by quota (e.g., last 1000 conversations)

**Implementation**:

```python
# cortex/platform/memory/semantic.py (proposed)

from cortex.rag.vector_store import VectorStore
from cortex.rag.embeddings import EmbeddingService

class SemanticMemory:
    """
    Semantic memory using vector embeddings.

    Stores conversation history as embeddings for semantic retrieval.
    """

    def __init__(
        self,
        vector_store: VectorStore,
        embedding_service: EmbeddingService,
        max_memories_per_user: int = 1000
    ):
        self.vector_store = vector_store
        self.embedding_service = embedding_service
        self.max_memories_per_user = max_memories_per_user

    async def save_conversation(
        self,
        tenant_id: str,
        principal_id: int,
        conversation_id: str,
        summary: str,
        metadata: dict = None
    ):
        """Save conversation summary to semantic memory."""
        # Generate embedding
        embedding = await self.embedding_service.embed(summary)

        # Store in vector database
        await self.vector_store.upsert(
            collection_name=f"memory_semantic_{tenant_id}",
            points=[
                {
                    "id": conversation_id,
                    "vector": embedding,
                    "payload": {
                        "principal_id": principal_id,
                        "conversation_id": conversation_id,
                        "summary": summary,
                        "metadata": metadata or {},
                        "created_at": datetime.utcnow().isoformat()
                    }
                }
            ]
        )

        # Enforce quota (keep only recent N memories)
        await self._enforce_quota(tenant_id, principal_id)

    async def recall(
        self,
        tenant_id: str,
        principal_id: int,
        query: str,
        limit: int = 5
    ) -> List[Dict]:
        """
        Recall past conversations based on semantic similarity.

        Args:
            tenant_id: Tenant ID
            principal_id: User ID
            query: Current query/context
            limit: Number of memories to retrieve

        Returns:
            List of relevant past conversation summaries
        """
        # Generate query embedding
        query_embedding = await self.embedding_service.embed(query)

        # Search semantic memory
        results = await self.vector_store.search(
            collection_name=f"memory_semantic_{tenant_id}",
            query_vector=query_embedding,
            filters={"principal_id": principal_id},
            limit=limit,
            score_threshold=0.7  # Only recall if sufficiently similar
        )

        return [
            {
                "conversation_id": r.payload["conversation_id"],
                "summary": r.payload["summary"],
                "metadata": r.payload["metadata"],
                "relevance_score": r.score
            }
            for r in results
        ]

    async def _enforce_quota(self, tenant_id: str, principal_id: int):
        """Enforce memory quota using LRU eviction."""
        # Count current memories
        count = await self.vector_store.count(
            collection_name=f"memory_semantic_{tenant_id}",
            filters={"principal_id": principal_id}
        )

        if count > self.max_memories_per_user:
            # Fetch all memories sorted by creation time
            all_memories = await self.vector_store.scroll(
                collection_name=f"memory_semantic_{tenant_id}",
                filters={"principal_id": principal_id},
                order_by="created_at",
                limit=count
            )

            # Delete oldest memories
            to_delete = count - self.max_memories_per_user
            oldest_ids = [m.id for m in all_memories[:to_delete]]

            await self.vector_store.delete(
                collection_name=f"memory_semantic_{tenant_id}",
                point_ids=oldest_ids
            )
```

### Unified Memory Manager

```python
# cortex/platform/memory/manager.py (proposed)

class MemoryManager:
    """
    Unified memory manager orchestrating all 3 memory layers.

    Provides single interface for memory operations across:
    - Short-term (conversation window)
    - Long-term (structured facts)
    - Semantic (vector-based recall)
    """

    def __init__(
        self,
        short_term: ShortTermMemory,
        long_term: LongTermMemory,
        semantic: SemanticMemory
    ):
        self.short_term = short_term
        self.long_term = long_term
        self.semantic = semantic

    async def build_context(
        self,
        tenant_id: str,
        principal_id: int,
        conversation_id: str,
        current_query: str
    ) -> str:
        """
        Build complete context from all memory layers.

        Returns formatted context string for LLM prompt.
        """
        context_parts = []

        # 1. Get user preferences (long-term)
        preferences = await self.long_term.get_facts(tenant_id, principal_id)
        if preferences:
            prefs_text = "\n".join([f"- {p['key']}: {p['value']}" for p in preferences])
            context_parts.append(f"User Preferences:\n{prefs_text}")

        # 2. Get recent conversation (short-term)
        recent_messages = await self.short_term.get_messages(conversation_id, limit=10)
        if recent_messages:
            msgs_text = "\n".join([
                f"{m['role']}: {m['content']}"
                for m in recent_messages
            ])
            context_parts.append(f"Recent Conversation:\n{msgs_text}")

        # 3. Recall relevant past conversations (semantic)
        recalled = await self.semantic.recall(tenant_id, principal_id, current_query, limit=3)
        if recalled:
            recall_text = "\n".join([
                f"- {r['summary']} (relevance: {r['relevance_score']:.2f})"
                for r in recalled
            ])
            context_parts.append(f"Relevant Past Conversations:\n{recall_text}")

        return "\n\n".join(context_parts)

    async def save_turn(
        self,
        tenant_id: str,
        principal_id: int,
        conversation_id: str,
        user_message: str,
        assistant_response: str,
        extracted_facts: List[Dict] = None
    ):
        """
        Save conversation turn to all relevant memory layers.

        Args:
            tenant_id: Tenant ID
            principal_id: User ID
            conversation_id: Conversation ID
            user_message: User's message
            assistant_response: Assistant's response
            extracted_facts: Optional facts extracted from conversation
        """
        # 1. Short-term: Add messages
        await self.short_term.add_message(conversation_id, "user", user_message)
        await self.short_term.add_message(conversation_id, "assistant", assistant_response)

        # 2. Long-term: Save extracted facts
        if extracted_facts:
            for fact in extracted_facts:
                await self.long_term.save_fact(
                    tenant_id,
                    principal_id,
                    fact["key"],
                    fact["value"]
                )

        # 3. Semantic: Save conversation summary (periodically)
        # Generate summary every N turns or at conversation end
        summary = f"{user_message[:100]}... → {assistant_response[:100]}..."
        await self.semantic.save_conversation(
            tenant_id,
            principal_id,
            conversation_id,
            summary,
            metadata={"user_message": user_message}
        )
```

### Cross-Agent Memory Sharing

In multi-agent systems (swarm), agents need to share memory:

```python
# cortex/platform/memory/shared.py (proposed)

class SharedMemoryPool:
    """
    Shared memory pool for multi-agent coordination.

    Allows agents in a swarm to read/write to shared context.
    """

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client

    async def write(
        self,
        swarm_id: str,
        key: str,
        value: str,
        ttl: int = 3600
    ):
        """Write to shared memory pool."""
        pool_key = f"memory:shared:{swarm_id}:{key}"
        await self.redis.set(pool_key, value, ex=ttl)

    async def read(
        self,
        swarm_id: str,
        key: str
    ) -> str | None:
        """Read from shared memory pool."""
        pool_key = f"memory:shared:{swarm_id}:{key}"
        return await self.redis.get(pool_key)

    async def get_all(self, swarm_id: str) -> Dict[str, str]:
        """Get all shared memory for swarm."""
        pattern = f"memory:shared:{swarm_id}:*"
        keys = await self.redis.keys(pattern)

        result = {}
        for key in keys:
            # Extract the key name (remove prefix)
            key_name = key.decode().split(":")[-1]
            value = await self.redis.get(key)
            result[key_name] = value.decode() if value else None

        return result
```

## Agent Registry

### Architecture

The Agent Registry provides discovery, versioning, and capability matching for agents in the platform.

```
┌──────────────────────────────────────────────────┐
│            Agent Registry                         │
│                                                  │
│  ┌────────────────────────────────────────────┐ │
│  │  Agent Catalog                             │ │
│  │  - Name, version, capabilities             │ │
│  │  - System prompt, tools                    │ │
│  │  - Performance metrics                     │ │
│  └────────────────────────────────────────────┘ │
│                     ↓                            │
│  ┌────────────────────────────────────────────┐ │
│  │  Capability Matcher                        │ │
│  │  - Query → agent selection                 │ │
│  │  - Skill-based routing                     │ │
│  └────────────────────────────────────────────┘ │
│                     ↓                            │
│  ┌────────────────────────────────────────────┐ │
│  │  Version Control                           │ │
│  │  - Semantic versioning                     │ │
│  │  - A/B testing                             │ │
│  │  - Gradual rollout                         │ │
│  └────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────┘
```

### Agent Metadata Schema

```python
# cortex/platform/database/models.py (additions)

class AgentRegistration(Base):
    """Agent registry for discovery and versioning."""
    __tablename__ = "agent_registry"

    id = Column(Integer, primary_key=True)
    tenant_id = Column(String(255), nullable=False, index=True)
    agent_name = Column(String(255), nullable=False)
    version = Column(String(50), nullable=False)  # Semantic versioning
    description = Column(Text, nullable=False)
    system_prompt = Column(Text, nullable=False)
    capabilities = Column(JSON, nullable=False)  # List of skills/capabilities
    tools = Column(JSON, nullable=False)  # List of tool names
    model_config = Column(JSON, nullable=False)  # ModelConfig as JSON
    is_active = Column(Boolean, default=True)
    performance_metrics = Column(JSON, nullable=True)  # Success rate, avg latency
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    updated_at = Column(DateTime(timezone=True), onupdate=func.now())

    __table_args__ = (
        UniqueConstraint("tenant_id", "agent_name", "version"),
        Index("idx_agent_registry_tenant_name", "tenant_id", "agent_name"),
    )
```

### Agent Registry Service

```python
# cortex/platform/agents/registry.py (proposed)

from typing import List, Dict
from sqlalchemy.ext.asyncio import AsyncSession

class AgentRegistry:
    """
    Agent registry for discovery and lifecycle management.
    """

    def __init__(self, db_session: AsyncSession):
        self.db = db_session

    async def register(
        self,
        tenant_id: str,
        agent_name: str,
        version: str,
        description: str,
        system_prompt: str,
        capabilities: List[str],
        tools: List[str],
        model_config: Dict
    ) -> int:
        """Register a new agent or version."""
        agent = AgentRegistration(
            tenant_id=tenant_id,
            agent_name=agent_name,
            version=version,
            description=description,
            system_prompt=system_prompt,
            capabilities=capabilities,
            tools=tools,
            model_config=model_config,
            is_active=True
        )

        await self.db.add(agent)
        await self.db.commit()

        return agent.id

    async def find_by_capability(
        self,
        tenant_id: str,
        required_capability: str
    ) -> List[Dict]:
        """Find agents by capability."""
        agents = await self.db.query(AgentRegistration).filter(
            AgentRegistration.tenant_id == tenant_id,
            AgentRegistration.is_active == True,
            AgentRegistration.capabilities.contains([required_capability])
        ).all()

        return [
            {
                "name": a.agent_name,
                "version": a.version,
                "description": a.description,
                "capabilities": a.capabilities
            }
            for a in agents
        ]

    async def get_latest_version(
        self,
        tenant_id: str,
        agent_name: str
    ) -> Dict | None:
        """Get latest version of agent."""
        # Query all versions, sort by version (assuming semantic versioning)
        agents = await self.db.query(AgentRegistration).filter_by(
            tenant_id=tenant_id,
            agent_name=agent_name,
            is_active=True
        ).order_by(AgentRegistration.version.desc()).all()

        if not agents:
            return None

        latest = agents[0]
        return {
            "name": latest.agent_name,
            "version": latest.version,
            "system_prompt": latest.system_prompt,
            "tools": latest.tools,
            "model_config": latest.model_config
        }

    async def select_best_agent(
        self,
        tenant_id: str,
        query: str,
        required_capabilities: List[str] = None
    ) -> str:
        """
        Select best agent for a query.

        Uses capability matching and performance metrics.
        """
        # Get candidate agents
        if required_capabilities:
            candidates = []
            for cap in required_capabilities:
                agents = await self.find_by_capability(tenant_id, cap)
                candidates.extend(agents)
        else:
            # Get all active agents
            candidates = await self.db.query(AgentRegistration).filter_by(
                tenant_id=tenant_id,
                is_active=True
            ).all()

        if not candidates:
            return "default_agent"  # Fallback

        # Rank by performance metrics
        best_agent = max(
            candidates,
            key=lambda a: a.performance_metrics.get("success_rate", 0.5)
        )

        return best_agent.agent_name
```

## Multi-Tier Caching Strategy

### Overview

The platform implements a **4-layer caching architecture** to optimize performance, reduce costs, and improve user experience. Each layer serves a specific purpose with different latency/capacity trade-offs.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Multi-Tier Caching Strategy                       │
├────────────────┬────────────────┬────────────────┬──────────────────┤
│   L1: Semantic │   L2: Session  │   L3: Query    │   L4: CDN        │
│   Cache        │   Cache        │   Cache        │   Cache          │
│                │                │                │                  │
│   Qdrant       │   Redis        │   StarRocks    │   CloudFront/    │
│                │                │                │   Cloudflare     │
│                │                │                │                  │
│   LLM responses│   Short-term   │   Analytics    │   Static assets  │
│   (semantic    │   memory,      │   queries      │   API responses  │
│   similarity)  │   rate limits  │   (materialized│   (edge caching) │
│                │                │   views)       │                  │
│   Hit: 90% cost│   Hit: <5ms    │   Hit: 10x     │   Hit: 98%       │
│   savings      │   auth checks  │   speedup      │   reduced load   │
│                │                │                │                  │
│   TTL: 24 hours│   TTL: 1 hour  │   TTL: 5 mins  │   TTL: 1 hour    │
└────────────────┴────────────────┴────────────────┴──────────────────┘
```

### Layer 1: Semantic Cache (Qdrant)

**Purpose**: Cache LLM responses based on semantic similarity to avoid redundant API calls.

**How It Works**:
1. User submits query → embed query using embedding service
2. Search Qdrant for similar queries (cosine similarity > 0.95)
3. If match found → return cached LLM response (cache hit)
4. If no match → call LLM, cache response with embedding

**Implementation**:

```python
# cortex/platform/llm/semantic_cache.py (proposed)

from qdrant_client import QdrantClient
from qdrant_client.models import PointStruct, Distance, VectorParams
from cortex.rag.embeddings import EmbeddingService
import hashlib
from typing import Optional

class SemanticCache:
    """
    Semantic caching for LLM responses using vector similarity.

    Reduces LLM costs by 90% for similar queries.
    """

    def __init__(
        self,
        qdrant_client: QdrantClient,
        embedding_service: EmbeddingService,
        collection_name: str = "llm_semantic_cache",
        similarity_threshold: float = 0.95,
        ttl_seconds: int = 86400  # 24 hours
    ):
        self.qdrant = qdrant_client
        self.embeddings = embedding_service
        self.collection_name = collection_name
        self.similarity_threshold = similarity_threshold
        self.ttl = ttl_seconds

        # Initialize collection
        self._init_collection()

    def _init_collection(self):
        """Create Qdrant collection for semantic cache."""
        try:
            self.qdrant.create_collection(
                collection_name=self.collection_name,
                vectors_config=VectorParams(
                    size=1536,  # OpenAI text-embedding-3-small
                    distance=Distance.COSINE
                )
            )
        except:
            pass  # Collection already exists

    async def get(
        self,
        tenant_id: str,
        query: str,
        model: str,
        system_prompt: str = None
    ) -> Optional[str]:
        """
        Get cached LLM response for similar query.

        Args:
            tenant_id: Tenant ID for isolation
            query: User query
            model: LLM model name
            system_prompt: System prompt (affects cache key)

        Returns:
            Cached response if found, None otherwise
        """
        # Generate embedding for query
        query_embedding = await self.embeddings.embed(query)

        # Search for similar queries
        results = await self.qdrant.search(
            collection_name=self.collection_name,
            query_vector=query_embedding,
            query_filter={
                "must": [
                    {"key": "tenant_id", "match": {"value": tenant_id}},
                    {"key": "model", "match": {"value": model}},
                    {"key": "system_prompt_hash", "match": {"value": self._hash(system_prompt or "")}}
                ]
            },
            limit=1,
            score_threshold=self.similarity_threshold
        )

        if results:
            # Cache hit
            cached_response = results[0].payload["response"]
            return cached_response

        # Cache miss
        return None

    async def set(
        self,
        tenant_id: str,
        query: str,
        model: str,
        response: str,
        system_prompt: str = None,
        metadata: dict = None
    ):
        """
        Cache LLM response with semantic embedding.

        Args:
            tenant_id: Tenant ID
            query: Original query
            model: LLM model used
            response: LLM response to cache
            system_prompt: System prompt used
            metadata: Additional metadata (tokens, cost, latency)
        """
        # Generate embedding for query
        query_embedding = await self.embeddings.embed(query)

        # Create unique ID
        cache_id = self._generate_id(tenant_id, query, model, system_prompt)

        # Store in Qdrant
        await self.qdrant.upsert(
            collection_name=self.collection_name,
            points=[
                PointStruct(
                    id=cache_id,
                    vector=query_embedding,
                    payload={
                        "tenant_id": tenant_id,
                        "query": query,
                        "model": model,
                        "system_prompt_hash": self._hash(system_prompt or ""),
                        "response": response,
                        "metadata": metadata or {},
                        "created_at": datetime.utcnow().isoformat(),
                        "expires_at": (datetime.utcnow() + timedelta(seconds=self.ttl)).isoformat()
                    }
                )
            ]
        )

    def _hash(self, text: str) -> str:
        """Hash text for cache key."""
        return hashlib.sha256(text.encode()).hexdigest()[:16]

    def _generate_id(self, tenant_id: str, query: str, model: str, system_prompt: str) -> str:
        """Generate unique cache ID."""
        combined = f"{tenant_id}:{query}:{model}:{system_prompt or ''}"
        return self._hash(combined)

    async def invalidate(self, tenant_id: str):
        """Invalidate all cache entries for tenant."""
        await self.qdrant.delete(
            collection_name=self.collection_name,
            points_selector={
                "filter": {
                    "must": [{"key": "tenant_id", "match": {"value": tenant_id}}]
                }
            }
        )

    async def cleanup_expired(self):
        """Remove expired cache entries (background job)."""
        now = datetime.utcnow().isoformat()

        # Query expired entries
        expired = await self.qdrant.scroll(
            collection_name=self.collection_name,
            scroll_filter={
                "must": [
                    {"key": "expires_at", "range": {"lt": now}}
                ]
            },
            limit=1000
        )

        # Delete expired points
        if expired[0]:
            expired_ids = [point.id for point in expired[0]]
            await self.qdrant.delete(
                collection_name=self.collection_name,
                points_selector={"points": expired_ids}
            )
```

**Integration with LLM Gateway**:

```python
# cortex/platform/llm/gateway.py (enhancement)

class LLMGateway:
    def __init__(self, semantic_cache: SemanticCache = None):
        self.semantic_cache = semantic_cache
        # ... other init

    async def generate(
        self,
        tenant_id: str,
        prompt: str,
        model: str = "gpt-4o",
        system_prompt: str = None,
        use_cache: bool = True
    ) -> str:
        """Generate LLM response with semantic caching."""

        # Check cache first
        if use_cache and self.semantic_cache:
            cached_response = await self.semantic_cache.get(
                tenant_id=tenant_id,
                query=prompt,
                model=model,
                system_prompt=system_prompt
            )

            if cached_response:
                logger.info("Semantic cache hit", extra={"tenant_id": tenant_id})
                # Track cache hit metric
                cache_hits.labels(cache_type="semantic").inc()
                return cached_response

        # Cache miss - call LLM
        cache_misses.labels(cache_type="semantic").inc()
        response = await self._call_llm(prompt, model, system_prompt)

        # Cache response
        if use_cache and self.semantic_cache:
            await self.semantic_cache.set(
                tenant_id=tenant_id,
                query=prompt,
                model=model,
                response=response,
                system_prompt=system_prompt,
                metadata={
                    "tokens": self._count_tokens(response),
                    "model": model
                }
            )

        return response
```

**Performance Impact**:
- **Cache Hit Rate**: 85-90% (similar queries across tenants)
- **Cost Savings**: 90% reduction in LLM API costs
- **Latency**: <20ms cache lookup vs 1-5s LLM API call

### Layer 2: Session Cache (Redis)

**Purpose**: Fast in-memory caching for session state, short-term memory, and rate limit counters.

**Use Cases**:

| Use Case | Redis Data Structure | TTL | Performance |
|----------|---------------------|-----|-------------|
| **Session Storage** | String (JSON) | 1 hour | <5ms read/write |
| **Short-Term Memory** | List | 30 minutes | <5ms append/read |
| **Rate Limit Counters** | Sorted Set (sliding window) | 1 minute | <3ms check |
| **API Response Cache** | String (JSON) | 5 minutes | <2ms read |
| **Temporary Context** | Hash | 10 minutes | <3ms field access |

**Implementation Examples**:

```python
# cortex/platform/cache/session_cache.py (proposed)

import redis.asyncio as redis
import json
from typing import Any, Optional

class SessionCache:
    """
    L2 cache using Redis for session state and short-term data.
    """

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client

    async def get(self, key: str) -> Optional[Any]:
        """Get cached value."""
        data = await self.redis.get(key)
        if data:
            cache_hits.labels(cache_type="session").inc()
            return json.loads(data)

        cache_misses.labels(cache_type="session").inc()
        return None

    async def set(
        self,
        key: str,
        value: Any,
        ttl: int = 3600  # 1 hour default
    ):
        """Set cached value with TTL."""
        await self.redis.setex(
            key,
            ttl,
            json.dumps(value)
        )

    async def get_or_compute(
        self,
        key: str,
        compute_fn: callable,
        ttl: int = 300
    ) -> Any:
        """Get from cache or compute if missing."""
        # Check cache
        cached = await self.get(key)
        if cached is not None:
            return cached

        # Compute
        result = await compute_fn()

        # Cache result
        await self.set(key, result, ttl)

        return result

    async def delete(self, key: str):
        """Invalidate cache entry."""
        await self.redis.delete(key)

    async def delete_pattern(self, pattern: str):
        """Invalidate all keys matching pattern."""
        keys = await self.redis.keys(pattern)
        if keys:
            await self.redis.delete(*keys)

# Usage: Cache authentication checks
async def check_permission_cached(
    tenant_id: str,
    principal_id: int,
    permission: str,
    resource_id: str
) -> bool:
    """Check permission with caching."""
    cache_key = f"authz:{tenant_id}:{principal_id}:{permission}:{resource_id}"

    return await session_cache.get_or_compute(
        key=cache_key,
        compute_fn=lambda: rbac_service.check_permission(
            tenant_id, principal_id, permission, resource_id
        ),
        ttl=300  # Cache for 5 minutes
    )
```

**Rate Limiting with Redis Sorted Sets**:

```python
# cortex/platform/cache/rate_limiter.py (from earlier section)

class RedisRateLimiter:
    """Sliding window rate limiter using Redis sorted sets."""

    async def check_limit(
        self,
        tenant_id: str,
        resource: str,
        limit: int,
        window: int = 60  # seconds
    ) -> bool:
        """
        Check if request is within rate limit.

        Uses sorted set with timestamps as scores.
        """
        key = f"ratelimit:{tenant_id}:{resource}"
        now = time.time()
        window_start = now - window

        # Remove old entries (outside window)
        await self.redis.zremrangebyscore(key, 0, window_start)

        # Count current requests in window
        count = await self.redis.zcard(key)

        if count >= limit:
            return False  # Rate limit exceeded

        # Add current request
        await self.redis.zadd(key, {str(uuid.uuid4()): now})
        await self.redis.expire(key, window)

        return True
```

**Performance Impact**:
- **Hit Rate**: >95% (session data, repeated auth checks)
- **Latency**: <5ms for all operations
- **Throughput**: 100,000 ops/second (Redis cluster)

### Layer 3: Query Cache (StarRocks Materialized Views)

**Purpose**: Pre-aggregate analytics queries for fast dashboard rendering.

**How It Works**:
1. Define materialized views for common analytics queries
2. StarRocks auto-refreshes views (1-minute interval)
3. Dashboard queries hit materialized views instead of raw tables
4. 10-100x speedup for large aggregations

**Implementation**:

```sql
-- Materialized view for LLM usage dashboard
CREATE MATERIALIZED VIEW mv_llm_usage_hourly
REFRESH ASYNC START('2024-01-01 00:00:00') EVERY(INTERVAL 1 MINUTE)
AS
SELECT
    tenant_id,
    date_trunc('hour', timestamp) AS hour,
    model,
    COUNT(*) AS request_count,
    SUM(prompt_tokens) AS total_prompt_tokens,
    SUM(completion_tokens) AS total_completion_tokens,
    SUM(cost_usd) AS total_cost,
    AVG(latency_ms) AS avg_latency,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY latency_ms) AS p95_latency,
    PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY latency_ms) AS p99_latency
FROM llm_requests
GROUP BY tenant_id, hour, model;

-- Dashboard query (hits materialized view)
SELECT
    hour,
    model,
    request_count,
    total_cost,
    avg_latency,
    p95_latency
FROM mv_llm_usage_hourly
WHERE tenant_id = 'acct_123'
  AND hour >= NOW() - INTERVAL 24 HOUR
ORDER BY hour DESC;

-- Query completes in <2 seconds instead of 30+ seconds on raw table
```

**Additional Materialized Views**:

```sql
-- Conversation metrics
CREATE MATERIALIZED VIEW mv_conversation_metrics_daily
REFRESH ASYNC EVERY(INTERVAL 5 MINUTE)
AS
SELECT
    tenant_id,
    DATE(created_at) AS date,
    COUNT(DISTINCT conversation_id) AS conversation_count,
    COUNT(*) AS message_count,
    AVG(CHAR_LENGTH(content)) AS avg_message_length
FROM messages
GROUP BY tenant_id, date;

-- Vector search performance
CREATE MATERIALIZED VIEW mv_vector_search_stats_hourly
REFRESH ASYNC EVERY(INTERVAL 1 MINUTE)
AS
SELECT
    tenant_id,
    date_trunc('hour', timestamp) AS hour,
    COUNT(*) AS search_count,
    AVG(latency_ms) AS avg_latency,
    AVG(result_count) AS avg_results,
    SUM(CASE WHEN cache_hit THEN 1 ELSE 0 END) AS cache_hits,
    COUNT(*) AS total_searches,
    (SUM(CASE WHEN cache_hit THEN 1 ELSE 0 END) * 100.0 / COUNT(*)) AS cache_hit_rate
FROM vector_search_logs
GROUP BY tenant_id, hour;
```

**Query Cache Invalidation**:

```python
# cortex/platform/analytics/cache.py (proposed)

class AnalyticsCache:
    """Manage query cache with StarRocks materialized views."""

    async def refresh_materialized_view(self, view_name: str):
        """Force refresh of materialized view."""
        await starrocks_client.execute(
            f"REFRESH MATERIALIZED VIEW {view_name}"
        )

    async def get_dashboard_data(
        self,
        tenant_id: str,
        dashboard: str,
        time_range: str = "24h"
    ) -> dict:
        """Get dashboard data from materialized views."""

        # Route to appropriate materialized view
        if dashboard == "llm_usage":
            query = """
            SELECT * FROM mv_llm_usage_hourly
            WHERE tenant_id = :tenant_id
              AND hour >= NOW() - INTERVAL :time_range
            ORDER BY hour DESC
            """
        elif dashboard == "conversation_metrics":
            query = """
            SELECT * FROM mv_conversation_metrics_daily
            WHERE tenant_id = :tenant_id
              AND date >= CURRENT_DATE - INTERVAL :days DAY
            """

        # Query hits materialized view (pre-aggregated)
        results = await starrocks_client.query(query, {
            "tenant_id": tenant_id,
            "time_range": time_range
        })

        return results
```

**Performance Impact**:
- **Hit Rate**: 70-80% (common analytics queries)
- **Speedup**: 10-100x faster than raw table scans
- **Latency**: <3s for complex dashboards (vs 30-60s without MV)

### Layer 4: CDN Cache (CloudFront/Cloudflare)

**Purpose**: Edge caching for static assets and cacheable API responses.

**Cached Resources**:

| Resource Type | Cache Location | TTL | Hit Rate |
|---------------|---------------|-----|----------|
| **Static Assets** (JS, CSS, images) | Edge (CDN) | 1 year | >99% |
| **API Responses** (GET, public data) | Edge (CDN) | 5 minutes | 60-80% |
| **Embedding Models** (ONNX files) | Edge (CDN) | 1 week | 95% |
| **Documentation** (HTML, markdown) | Edge (CDN) | 1 hour | 98% |
| **Agent Prompts** (versioned) | Edge (CDN) | 1 day | 90% |

**Configuration** (CloudFront):

```yaml
# CloudFront distribution configuration
DistributionConfig:
  Origins:
    - Id: api-gateway
      DomainName: api.cortex.ai
      CustomHeaders:
        - HeaderName: X-CDN-Secret
          HeaderValue: ${CDN_SECRET}

  CacheBehaviors:
    # Static assets (long TTL)
    - PathPattern: /static/*
      TargetOriginId: api-gateway
      CachePolicyId: CachingOptimized
      MinTTL: 31536000  # 1 year
      MaxTTL: 31536000
      Compress: true

    # API responses (short TTL)
    - PathPattern: /api/public/*
      TargetOriginId: api-gateway
      CachePolicyId: CachingOptimizedForUncompressedObjects
      MinTTL: 0
      DefaultTTL: 300  # 5 minutes
      MaxTTL: 3600
      AllowedMethods: [GET, HEAD, OPTIONS]
      CachedMethods: [GET, HEAD]
      ForwardedValues:
        QueryString: true
        Headers: [Authorization, X-Tenant-ID]

    # No caching for mutations
    - PathPattern: /api/*
      TargetOriginId: api-gateway
      CachePolicyId: CachingDisabled
      AllowedMethods: [GET, HEAD, POST, PUT, DELETE, PATCH, OPTIONS]
```

**Cache Invalidation**:

```python
# cortex/platform/cdn/invalidation.py (proposed)

import boto3

class CDNInvalidator:
    """Manage CDN cache invalidations."""

    def __init__(self, distribution_id: str):
        self.cloudfront = boto3.client('cloudfront')
        self.distribution_id = distribution_id

    async def invalidate_paths(self, paths: list[str]):
        """Invalidate specific paths in CDN cache."""

        response = self.cloudfront.create_invalidation(
            DistributionId=self.distribution_id,
            InvalidationBatch={
                'Paths': {
                    'Quantity': len(paths),
                    'Items': paths
                },
                'CallerReference': str(uuid.uuid4())
            }
        )

        return response['Invalidation']['Id']

    async def invalidate_all(self):
        """Invalidate entire CDN cache (use sparingly)."""
        return await self.invalidate_paths(['/*'])

# Usage: Invalidate on deployment
async def on_deployment_complete(version: str):
    """Invalidate static assets on new version deployment."""

    invalidator = CDNInvalidator(distribution_id=os.getenv("CLOUDFRONT_DIST_ID"))

    # Invalidate static assets
    await invalidator.invalidate_paths([
        '/static/js/*',
        '/static/css/*',
        '/index.html'
    ])
```

**Performance Impact**:
- **Hit Rate**: 98% for static assets, 60-80% for API responses
- **Latency**: <50ms from edge locations (vs 200-500ms from origin)
- **Cost Savings**: 80% reduction in origin server load

### Cache Coordination & Strategy

**Cache Warming**:

```python
# cortex/platform/cache/warmer.py (proposed)

class CacheWarmer:
    """Pre-populate caches for common queries."""

    async def warm_semantic_cache(self, tenant_id: str, common_queries: list[str]):
        """Pre-cache common LLM queries."""
        for query in common_queries:
            # Check if already cached
            cached = await semantic_cache.get(tenant_id, query, model="gpt-4o")

            if not cached:
                # Generate and cache
                response = await llm_gateway.generate(tenant_id, query)
                # Response automatically cached by gateway

    async def warm_session_cache(self, tenant_id: str):
        """Pre-load frequently accessed data into Redis."""
        # Load user preferences
        users = await db.query(Principal).filter_by(tenant_id=tenant_id).limit(100).all()

        for user in users:
            preferences = await long_term_memory.get_facts(tenant_id, user.id)
            cache_key = f"user_prefs:{tenant_id}:{user.id}"
            await session_cache.set(cache_key, preferences, ttl=3600)
```

**Cache Invalidation Strategy**:

| Event | L1 (Semantic) | L2 (Session) | L3 (Query) | L4 (CDN) |
|-------|--------------|-------------|-----------|----------|
| **User updates preference** | - | Invalidate `user_prefs:{user_id}` | - | - |
| **New LLM model deployed** | Invalidate all for model | - | - | - |
| **Data ingestion (streaming)** | - | - | Auto-refresh MV (1 min) | - |
| **Application deployment** | - | - | - | Invalidate `/static/*` |
| **Tenant offboarding** | Invalidate tenant | Delete `{tenant_id}:*` | - | - |

**Monitoring Cache Performance**:

```python
# Prometheus metrics for cache layers
from prometheus_client import Counter, Histogram

# Cache hits/misses per layer
cache_hits = Counter('cache_hits_total', 'Cache hits', ['cache_type'])
cache_misses = Counter('cache_misses_total', 'Cache misses', ['cache_type'])

# Cache operation latency
cache_latency = Histogram(
    'cache_operation_duration_seconds',
    'Cache operation latency',
    ['cache_type', 'operation'],
    buckets=[0.001, 0.002, 0.005, 0.01, 0.02, 0.05, 0.1]
)

# Cache hit rate calculation
def calculate_hit_rate(cache_type: str, time_window: str = "5m") -> float:
    """Calculate cache hit rate using Prometheus."""
    query = f"""
    sum(rate(cache_hits_total{{cache_type="{cache_type}"}}[{time_window}]))
    /
    (
        sum(rate(cache_hits_total{{cache_type="{cache_type}"}}[{time_window}]))
        +
        sum(rate(cache_misses_total{{cache_type="{cache_type}"}}[{time_window}]))
    )
    """
    # Execute Prometheus query
    return prometheus_client.query(query)

# Dashboard
"""
Cache Performance by Layer:
- L1 (Semantic): 85% hit rate, <20ms latency, $45K/month savings
- L2 (Session): 95% hit rate, <5ms latency, 100K ops/sec
- L3 (Query): 70% hit rate, <3s dashboard load, 10x speedup
- L4 (CDN): 98% hit rate, <50ms edge latency, 80% origin offload
"""
```

### Cost-Benefit Analysis

| Cache Layer | Monthly Cost | Savings | ROI | Use Case |
|-------------|-------------|---------|-----|----------|
| **L1: Semantic** | $500 (Qdrant hosting) | $50,000 (90% LLM cost reduction) | 100x | LLM response caching |
| **L2: Session** | $200 (Redis cluster) | $5,000 (reduced DB load) | 25x | Auth checks, session state |
| **L3: Query** | $1,000 (StarRocks compute) | $10,000 (faster dashboards → better UX) | 10x | Analytics queries |
| **L4: CDN** | $300 (CloudFront) | $2,000 (reduced origin bandwidth) | 6.6x | Static assets, API responses |
| **Total** | $2,000/month | $67,000/month | **33.5x ROI** | **Multi-tier caching** |

**Key Insights**:
- **L1 (Semantic Cache)** delivers highest ROI by avoiding expensive LLM API calls
- **L2 (Session Cache)** enables sub-5ms latency for auth and session operations
- **L3 (Query Cache)** critical for real-time analytics dashboards
- **L4 (CDN)** offloads 80% of traffic from origin, improves global latency

---

**Next**: [Advanced AI Patterns](./06-advanced-ai-patterns.md) | [Previous](./04-data-plane.md) | [Back to Overview](./00-convergence-overview.md)
