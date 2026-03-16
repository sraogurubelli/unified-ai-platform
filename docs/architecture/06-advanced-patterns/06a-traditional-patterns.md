# Advanced Traditional Patterns

## Overview

Advanced patterns for Traditional Stack including API design, caching strategies, event-driven architectures, and SaaS best practices.

---

## Multi-Tier Caching Strategy

### L1-L4 Caching Layers

```
Request → L1 (CDN) → L2 (API Gateway) → L3 (Redis) → L4 (Database)
```

| Layer | Technology | TTL | Use Case | Hit Rate Target |
|-------|-----------|-----|----------|----------------|
| **L1** | CloudFlare CDN | 1 hour | Static assets, public pages | >98% |
| **L2** | API Gateway (in-memory) | 1 minute | Frequently accessed API responses | >70% |
| **L3** | Redis | 5-60 minutes | Session data, query results | >85% |
| **L4** | PostgreSQL (query cache) | N/A | Source of truth | N/A |

### Implementation

```python
class MultiTierCache:
    """Multi-tier caching implementation."""

    async def get(self, key: str):
        """Get from cache with fallthrough."""

        # L2: API Gateway cache (in-memory)
        if key in self.local_cache:
            return self.local_cache[key]

        # L3: Redis cache
        redis_value = await redis.get(key)
        if redis_value:
            self.local_cache[key] = redis_value  # Populate L2
            return redis_value

        # L4: Database (cache miss)
        db_value = await db.query(...)
        await redis.setex(key, ttl=300, value=db_value)  # Populate L3
        self.local_cache[key] = db_value  # Populate L2

        return db_value
```

---

## API Versioning

### URL-Based Versioning

```python
# v1/routes.py
@app.get("/api/v1/projects")
async def list_projects_v1():
    return {"projects": [...]}

# v2/routes.py
@app.get("/api/v2/projects")
async def list_projects_v2():
    # Breaking change: different response format
    return {
        "data": {"projects": [...]},
        "meta": {"total": 10}
    }
```

### Header-Based Versioning

```python
@app.get("/api/projects")
async def list_projects(request: Request):
    """API version from header."""

    version = request.headers.get("X-API-Version", "1")

    if version == "1":
        return v1_response()
    elif version == "2":
        return v2_response()
```

---

## Pagination Patterns

### Cursor-Based Pagination (Recommended)

```python
@app.get("/api/v1/projects")
async def list_projects(
    cursor: str = None,
    limit: int = 100
):
    """Cursor-based pagination (efficient for large datasets)."""

    if cursor:
        # Decode cursor (base64 encoded ID)
        last_id = decode_cursor(cursor)
        query = db.query(Project).filter(Project.id > last_id)
    else:
        query = db.query(Project)

    projects = await query.order_by(Project.id).limit(limit + 1).all()

    # Check if more results exist
    has_next = len(projects) > limit
    if has_next:
        projects = projects[:limit]

    next_cursor = encode_cursor(projects[-1].id) if has_next else None

    return {
        "data": projects,
        "pagination": {
            "next_cursor": next_cursor,
            "has_next": has_next
        }
    }
```

---

## Webhook Patterns

### Retry with Exponential Backoff

```python
class WebhookDelivery:
    """Webhook delivery with retry logic."""

    async def deliver(
        self,
        webhook: Webhook,
        event: dict,
        max_retries: int = 5
    ):
        """Deliver webhook with exponential backoff."""

        for attempt in range(max_retries):
            try:
                await self._send(webhook, event)
                return  # Success

            except httpx.HTTPError as e:
                if attempt == max_retries - 1:
                    # Final attempt failed
                    await self._mark_failed(webhook, event)
                    return

                # Exponential backoff: 2^attempt * 60 seconds
                wait_time = (2 ** attempt) * 60

                logger.warning(
                    "webhook_retry",
                    webhook_id=webhook.uid,
                    attempt=attempt + 1,
                    wait_time=wait_time
                )

                await asyncio.sleep(wait_time)
```

---

## Event-Driven Architecture

### Event Sourcing Pattern

```python
class EventStore:
    """Store all events for audit and replay."""

    async def append(
        self,
        aggregate_id: str,
        event_type: str,
        data: dict
    ):
        """Append event to store."""

        event = Event(
            aggregate_id=aggregate_id,
            event_type=event_type,
            data=json.dumps(data),
            version=await self._get_next_version(aggregate_id),
            timestamp=datetime.utcnow()
        )

        await db.add(event)
        await db.commit()

        # Publish to event bus
        await event_bus.publish(event_type, data)

    async def replay(self, aggregate_id: str):
        """Replay all events for aggregate."""

        events = await db.query(Event).filter_by(
            aggregate_id=aggregate_id
        ).order_by(Event.version).all()

        state = {}
        for event in events:
            state = self._apply_event(state, event)

        return state
```

---

## Multi-Tenancy Patterns

### Shared Database, Shared Schema (with RLS)

**Advantages**:
- Cost-effective
- Easy management
- Efficient resource usage

**Implementation**: See [02a-traditional-control-plane.md](../02-control-plane/02a-traditional-control-plane.md#multi-tenancy)

### Shared Database, Separate Schemas

```sql
-- Create schema per tenant
CREATE SCHEMA tenant_acct_123;
CREATE SCHEMA tenant_acct_456;

-- Create tables in tenant schema
CREATE TABLE tenant_acct_123.projects (...);
CREATE TABLE tenant_acct_456.projects (...);
```

```python
# Set schema per request
async def set_tenant_schema(tenant_id: str):
    """Set PostgreSQL schema for tenant."""
    await db.execute(f"SET search_path TO tenant_{tenant_id}")
```

---

## Rate Limiting Patterns

### Token Bucket Algorithm

```python
class TokenBucket:
    """Token bucket rate limiter."""

    def __init__(self, capacity: int, refill_rate: float):
        self.capacity = capacity
        self.refill_rate = refill_rate  # tokens/second
        self.tokens = capacity
        self.last_refill = time.time()

    async def consume(self, tokens: int = 1) -> bool:
        """Try to consume tokens."""

        # Refill bucket
        now = time.time()
        elapsed = now - self.last_refill
        self.tokens = min(
            self.capacity,
            self.tokens + (elapsed * self.refill_rate)
        )
        self.last_refill = now

        # Try to consume
        if self.tokens >= tokens:
            self.tokens -= tokens
            return True

        return False
```

---

**Next**: [AI Patterns](./06b-ai-patterns.md) | [Back to Overview](../00-parallel-stacks-overview.md)
