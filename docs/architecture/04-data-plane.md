# Data Plane: Hybrid Storage Strategy

## Overview

The data plane implements a **hybrid storage strategy** that combines traditional databases (OLTP, OLAP) with AI-specific storage (vectors, graphs). Different data types require different storage engines optimized for their access patterns.

```
┌──────────────────────────────────────────────────────────────┐
│                      Data Plane                               │
├────────────┬────────────┬─────────────┬─────────────────────┤
│ OLTP       │ OLAP       │ Vector      │ Graph               │
│ PostgreSQL │ ClickHouse │ Qdrant      │ Neo4j               │
│            │            │             │                     │
│ Users      │ Usage      │ Embeddings  │ Knowledge           │
│ Projects   │ Analytics  │ Semantic    │ Relationships       │
│ Sessions   │ Metrics    │ Search      │ Entities            │
└────────────┴────────────┴─────────────┴─────────────────────┘
                              ▼
┌──────────────────────────────────────────────────────────────┐
│              Cache & Queue Layer (Redis)                      │
│         - Session cache                                       │
│         - Task queues (agent execution)                       │
│         - Pub/Sub (real-time events)                          │
└──────────────────────────────────────────────────────────────┘
```

## Storage Selection Matrix

| Data Type | Storage Engine | Reason | Use Cases |
|-----------|---------------|--------|-----------|
| **Transactional** | PostgreSQL (OLTP) | ACID, relationships, integrity | Users, projects, conversations, messages |
| **Analytical** | ClickHouse (OLAP) | Column-oriented, fast aggregations | Usage metrics, analytics, reporting |
| **Embeddings** | Qdrant (Vector) | Similarity search, ANN | Document search, semantic retrieval |
| **Knowledge** | Neo4j (Graph) | Relationship traversal, graph algorithms | Entities, knowledge graphs, RAG |
| **Session/Cache** | Redis | Low latency, pub/sub | Sessions, rate limiting, job queues |

## OLTP: PostgreSQL

### Schema Design

```sql
-- Multi-tenant hierarchy
CREATE TABLE accounts (
    id SERIAL PRIMARY KEY,
    uid VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    status VARCHAR(50) NOT NULL,
    subscription_tier VARCHAR(50) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_accounts_uid ON accounts(uid);

CREATE TABLE organizations (
    id SERIAL PRIMARY KEY,
    uid VARCHAR(255) UNIQUE NOT NULL,
    account_id INTEGER NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_organizations_account ON organizations(account_id);

CREATE TABLE projects (
    id SERIAL PRIMARY KEY,
    uid VARCHAR(255) UNIQUE NOT NULL,
    organization_id INTEGER NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_projects_organization ON projects(organization_id);

-- AI resources
CREATE TABLE conversations (
    id SERIAL PRIMARY KEY,
    uid VARCHAR(255) UNIQUE NOT NULL,
    project_id INTEGER NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    principal_id INTEGER NOT NULL REFERENCES principals(id),
    thread_id VARCHAR(255) NOT NULL,
    title VARCHAR(500),
    meta_json TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_conversations_project ON conversations(project_id);
CREATE INDEX idx_conversations_thread ON conversations(thread_id);

CREATE TABLE messages (
    id SERIAL PRIMARY KEY,
    uid VARCHAR(255) UNIQUE NOT NULL,
    conversation_id INTEGER NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
    role VARCHAR(50) NOT NULL,
    content TEXT NOT NULL,
    tool_calls TEXT,
    tool_call_id VARCHAR(255),
    meta_json TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_messages_conversation_created ON messages(conversation_id, created_at);
```

### Tenant Isolation: Row-Level Security

```sql
-- Enable RLS
ALTER TABLE conversations ENABLE ROW LEVEL SECURITY;
ALTER TABLE messages ENABLE ROW LEVEL SECURITY;

-- Policy: Only see data from your tenant
CREATE POLICY tenant_isolation_conversations ON conversations
USING (
    project_id IN (
        SELECT p.id
        FROM projects p
        JOIN organizations o ON p.organization_id = o.id
        WHERE o.account_id = current_setting('app.tenant_id')::INTEGER
    )
);

CREATE POLICY tenant_isolation_messages ON messages
USING (
    conversation_id IN (
        SELECT c.id
        FROM conversations c
        JOIN projects p ON c.project_id = p.id
        JOIN organizations o ON p.organization_id = o.id
        WHERE o.account_id = current_setting('app.tenant_id')::INTEGER
    )
);

-- Set tenant context per connection
-- In application:
await db.execute("SET app.tenant_id = :tenant_id", {"tenant_id": 123})
```

### Performance Optimization

#### Connection Pooling

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker

# Connection pool
engine = create_async_engine(
    DATABASE_URL,
    pool_size=20,          # Number of connections to maintain
    max_overflow=10,       # Additional connections under load
    pool_pre_ping=True,    # Verify connection health
    pool_recycle=3600,     # Recycle connections after 1 hour
    echo=False
)

async_session_factory = sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False
)
```

#### Query Optimization

```python
# Bad: N+1 query problem
conversations = await db.query(Conversation).all()
for conv in conversations:
    # This triggers a separate query for each conversation
    messages = await db.query(Message).filter_by(
        conversation_id=conv.id
    ).all()

# Good: Join or eager loading
from sqlalchemy.orm import selectinload

conversations = await db.query(Conversation).options(
    selectinload(Conversation.messages)
).all()

# Now messages are already loaded
for conv in conversations:
    messages = conv.messages  # No additional query
```

#### Partitioning

```sql
-- Partition large tables by tenant
CREATE TABLE conversations_partitioned (
    id SERIAL,
    account_id INTEGER NOT NULL,
    -- other fields
) PARTITION BY HASH (account_id);

-- Create partitions
CREATE TABLE conversations_p0 PARTITION OF conversations_partitioned
    FOR VALUES WITH (MODULUS 4, REMAINDER 0);

CREATE TABLE conversations_p1 PARTITION OF conversations_partitioned
    FOR VALUES WITH (MODULUS 4, REMAINDER 1);

CREATE TABLE conversations_p2 PARTITION OF conversations_partitioned
    FOR VALUES WITH (MODULUS 4, REMAINDER 2);

CREATE TABLE conversations_p3 PARTITION OF conversations_partitioned
    FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

## OLAP: ClickHouse

### Schema Design

```sql
-- Usage events table (append-only, immutable)
CREATE TABLE usage_events (
    timestamp DateTime,
    tenant_id String,
    event_type String,  -- 'api_call', 'llm_tokens', 'storage'
    resource_type String,
    resource_id String,
    quantity UInt64,
    cost_usd Float64,
    metadata String  -- JSON
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (tenant_id, event_type, timestamp);

-- Materialized view for monthly aggregates
CREATE MATERIALIZED VIEW usage_monthly
ENGINE = SummingMergeTree()
PARTITION BY (tenant_id, year_month)
ORDER BY (tenant_id, year_month, event_type)
AS SELECT
    tenant_id,
    toYYYYMM(timestamp) AS year_month,
    event_type,
    sum(quantity) AS total_quantity,
    sum(cost_usd) AS total_cost
FROM usage_events
GROUP BY tenant_id, year_month, event_type;
```

### Querying

```python
from clickhouse_driver import Client

client = Client(host='clickhouse-server', port=9000)

# Record usage event
client.execute(
    """
    INSERT INTO usage_events
    (timestamp, tenant_id, event_type, resource_type, resource_id, quantity, cost_usd, metadata)
    VALUES
    """,
    [(
        datetime.now(),
        'acct_123',
        'llm_tokens',
        'conversation',
        'conv_456',
        1500,
        0.015,
        '{\"model\": \"gpt-4o\"}'
    )]
)

# Query monthly usage
result = client.execute(
    """
    SELECT
        year_month,
        event_type,
        total_quantity,
        total_cost
    FROM usage_monthly
    WHERE tenant_id = %(tenant_id)s
    ORDER BY year_month DESC
    LIMIT 12
    """,
    {"tenant_id": "acct_123"}
)

for row in result:
    print(f"{row[0]}: {row[1]} - {row[2]} units, ${row[3]:.2f}")
```

### Tenant Isolation

```sql
-- ClickHouse doesn't have RLS, so filter in queries
SELECT * FROM usage_events
WHERE tenant_id = 'acct_123'  -- Always include this filter
```

## Vector Store: Qdrant

### Collection Setup

```python
from qdrant_client import QdrantClient
from qdrant_client.models import Distance, VectorParams, PointStruct

client = QdrantClient(url="http://qdrant:6333")

# Create collection per tenant
async def create_tenant_collection(tenant_id: str):
    """Create vector collection for tenant."""

    collection_name = f"embeddings_{tenant_id}"

    await client.create_collection(
        collection_name=collection_name,
        vectors_config=VectorParams(
            size=1536,  # OpenAI ada-002 dimension
            distance=Distance.COSINE
        )
    )

    # Create payload index for filtering
    await client.create_payload_index(
        collection_name=collection_name,
        field_name="document_id",
        field_schema="keyword"
    )
```

### Tenant Isolation Strategies

#### Option 1: Collection per Tenant

```python
# Pros: Complete isolation, easy to manage quotas
# Cons: More collections to manage

collection_name = f"embeddings_{tenant_id}"

await client.upsert(
    collection_name=collection_name,
    points=[
        PointStruct(
            id=str(uuid.uuid4()),
            vector=embedding,
            payload={
                "document_id": "doc_123",
                "text": "...",
                "metadata": {...}
            }
        )
    ]
)
```

#### Option 2: Single Collection with Filtering

```python
# Pros: Fewer collections, simpler management
# Cons: Requires filtering on every query

from qdrant_client.models import Filter, FieldCondition, MatchValue

await client.upsert(
    collection_name="embeddings",
    points=[
        PointStruct(
            id=str(uuid.uuid4()),
            vector=embedding,
            payload={
                "tenant_id": tenant_id,  # Include tenant_id
                "document_id": "doc_123",
                "text": "...",
            }
        )
    ]
)

# Always filter by tenant_id
results = await client.search(
    collection_name="embeddings",
    query_vector=query_embedding,
    query_filter=Filter(
        must=[
            FieldCondition(
                key="tenant_id",
                match=MatchValue(value=tenant_id)
            )
        ]
    ),
    limit=10
)
```

### Indexing & Search

```python
class QdrantVectorStore:
    """Vector store wrapper with tenant isolation."""

    async def upsert_documents(
        self,
        tenant_id: str,
        documents: list[Document]
    ):
        """Index documents with embeddings."""

        # Generate embeddings
        embeddings = await self.embedding_provider.embed_batch(
            texts=[doc.content for doc in documents]
        )

        # Create points
        points = [
            PointStruct(
                id=str(uuid.uuid4()),
                vector=embedding,
                payload={
                    "tenant_id": tenant_id,
                    "document_id": doc.id,
                    "title": doc.title,
                    "text": doc.content,
                    "metadata": doc.metadata
                }
            )
            for doc, embedding in zip(documents, embeddings)
        ]

        # Upsert to Qdrant
        await self.client.upsert(
            collection_name=f"embeddings_{tenant_id}",
            points=points
        )

    async def search(
        self,
        tenant_id: str,
        query: str,
        top_k: int = 5
    ) -> list[Document]:
        """Semantic search."""

        # Generate query embedding
        query_embedding = await self.embedding_provider.embed(query)

        # Search
        results = await self.client.search(
            collection_name=f"embeddings_{tenant_id}",
            query_vector=query_embedding,
            limit=top_k,
            score_threshold=0.7  # Minimum similarity
        )

        return [
            Document(
                id=hit.payload["document_id"],
                title=hit.payload["title"],
                content=hit.payload["text"],
                score=hit.score,
                metadata=hit.payload["metadata"]
            )
            for hit in results
        ]
```

## Graph Database: Neo4j

### Schema Design

```cypher
// Constraints
CREATE CONSTRAINT entity_tenant_name IF NOT EXISTS
FOR (e:Entity)
REQUIRE (e.tenant_id, e.name) IS NODE KEY;

CREATE INDEX entity_tenant IF NOT EXISTS
FOR (e:Entity) ON (e.tenant_id);

// Nodes with tenant_id
CREATE (e:Entity {
    tenant_id: "acct_123",
    type: "Person",
    name: "Alice",
    attributes: '{\"role\": \"engineer\"}'
})

CREATE (c:Entity {
    tenant_id: "acct_123",
    type: "Company",
    name: "Acme Corp"
})

// Relationships
CREATE (e)-[:WORKS_AT {
    tenant_id: "acct_123",
    since: "2020-01-01"
}]->(c)
```

### Tenant Isolation

```python
from neo4j import AsyncGraphDatabase

class Neo4jGraphStore:
    """Graph store with tenant isolation."""

    def __init__(self, uri: str, user: str, password: str):
        self.driver = AsyncGraphDatabase.driver(uri, auth=(user, password))

    async def create_entity(
        self,
        tenant_id: str,
        entity_type: str,
        name: str,
        attributes: dict
    ):
        """Create entity node with tenant isolation."""

        query = f"""
        MERGE (e:{entity_type} {{tenant_id: $tenant_id, name: $name}})
        SET e.attributes = $attributes
        RETURN e
        """

        async with self.driver.session() as session:
            result = await session.run(
                query,
                tenant_id=tenant_id,
                name=name,
                attributes=json.dumps(attributes)
            )
            return await result.single()

    async def query(
        self,
        tenant_id: str,
        cypher: str,
        params: dict = None
    ) -> list[dict]:
        """Execute Cypher query with tenant filter."""

        # Inject tenant filter
        query_with_tenant = f"""
        MATCH (n)
        WHERE n.tenant_id = $tenant_id
        {cypher}
        """

        async with self.driver.session() as session:
            result = await session.run(
                query_with_tenant,
                tenant_id=tenant_id,
                **(params or {})
            )
            return [record.data() async for record in result]

    async def find_related_entities(
        self,
        tenant_id: str,
        entity_name: str,
        relationship_type: str = None,
        max_depth: int = 2
    ) -> list[dict]:
        """Find entities related to given entity."""

        rel_filter = f"[r:{relationship_type}*1..{max_depth}]" if relationship_type else f"[r*1..{max_depth}]"

        query = f"""
        MATCH (start {{tenant_id: $tenant_id, name: $entity_name}})
        MATCH (start)-{rel_filter}-(related)
        WHERE related.tenant_id = $tenant_id
        RETURN DISTINCT related.name AS name, related.type AS type, related.attributes AS attributes
        """

        return await self.query(tenant_id, query, {"entity_name": entity_name})
```

## Cache & Queue: Redis

### Use Cases

1. **Session Storage**: User sessions, auth tokens
2. **Rate Limiting**: Track API usage per tenant
3. **Task Queues**: Agent execution jobs
4. **Pub/Sub**: Real-time events
5. **Caching**: Frequently accessed data

### Session Storage

```python
import redis.asyncio as redis

class SessionStore:
    """Redis-backed session storage."""

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client

    async def create_session(
        self,
        principal_id: str,
        tenant_id: str,
        ttl: int = 86400  # 24 hours
    ) -> str:
        """Create new session."""

        session_id = secrets.token_urlsafe(32)
        session_key = f"session:{session_id}"

        session_data = {
            "principal_id": principal_id,
            "tenant_id": tenant_id,
            "created_at": datetime.utcnow().isoformat()
        }

        await self.redis.setex(
            session_key,
            ttl,
            json.dumps(session_data)
        )

        return session_id

    async def get_session(self, session_id: str) -> dict:
        """Retrieve session."""

        session_key = f"session:{session_id}"
        data = await self.redis.get(session_key)

        if not data:
            raise ValueError("Session not found or expired")

        return json.loads(data)

    async def delete_session(self, session_id: str):
        """Delete session (logout)."""

        await self.redis.delete(f"session:{session_id}")
```

### Rate Limiting

```python
class RedisRateLimiter:
    """Sliding window rate limiter."""

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

        # Count current requests in window
        count = await self.redis.zcard(key)

        if count >= limit:
            return False

        # Add current request
        await self.redis.zadd(key, {str(uuid.uuid4()): now})
        await self.redis.expire(key, window)

        return True
```

### Task Queues (Redis Streams)

```python
class TaskQueue:
    """Redis Streams-based task queue."""

    async def enqueue(
        self,
        queue_name: str,
        task: dict
    ) -> str:
        """Add task to queue."""

        message_id = await self.redis.xadd(
            f"queue:{queue_name}",
            {"data": json.dumps(task)}
        )

        return message_id

    async def consume(
        self,
        queue_name: str,
        consumer_group: str,
        consumer_name: str
    ):
        """Consume tasks from queue."""

        stream = f"queue:{queue_name}"

        # Create consumer group if not exists
        try:
            await self.redis.xgroup_create(stream, consumer_group, id="0")
        except:
            pass

        # Consume messages
        while True:
            messages = await self.redis.xreadgroup(
                consumer_group,
                consumer_name,
                {stream: ">"},
                count=10,
                block=5000  # 5 second timeout
            )

            for stream_name, msg_list in messages:
                for message_id, data in msg_list:
                    task = json.loads(data[b"data"])
                    yield task

                    # Acknowledge
                    await self.redis.xack(stream, consumer_group, message_id)
```

### Caching

```python
class CacheManager:
    """Redis-backed caching."""

    async def get(self, key: str) -> any:
        """Get cached value."""

        data = await self.redis.get(key)
        return json.loads(data) if data else None

    async def set(
        self,
        key: str,
        value: any,
        ttl: int = 300  # 5 minutes
    ):
        """Cache value with TTL."""

        await self.redis.setex(key, ttl, json.dumps(value))

    async def delete(self, key: str):
        """Invalidate cache."""

        await self.redis.delete(key)

# Usage with decorator
from functools import wraps

def cached(ttl: int = 300):
    """Cache decorator."""

    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            # Generate cache key
            cache_key = f"cache:{func.__name__}:{args}:{kwargs}"

            # Check cache
            cached_value = await cache.get(cache_key)
            if cached_value:
                return cached_value

            # Execute function
            result = await func(*args, **kwargs)

            # Cache result
            await cache.set(cache_key, result, ttl)

            return result

        return wrapper
    return decorator

# Example
@cached(ttl=600)
async def get_project_stats(project_id: str) -> dict:
    # Expensive computation
    ...
```

## Data Management Patterns

### Backup & Recovery

```python
class BackupManager:
    """Automated backup management."""

    async def backup_postgres(self, tenant_id: str):
        """Backup PostgreSQL data for tenant."""

        timestamp = datetime.utcnow().strftime("%Y%m%d_%H%M%S")
        backup_file = f"backups/postgres_{tenant_id}_{timestamp}.sql"

        # pg_dump with tenant filter
        cmd = f"""
        pg_dump -h postgres -U cortex -d cortex \
        --table=conversations \
        --table=messages \
        --where="tenant_id='{tenant_id}'" \
        > {backup_file}
        """

        await asyncio.subprocess.run(cmd, shell=True)

    async def backup_qdrant(self, tenant_id: str):
        """Backup Qdrant collection."""

        collection_name = f"embeddings_{tenant_id}"

        # Create snapshot
        snapshot = await qdrant_client.create_snapshot(collection_name)

        # Download snapshot
        snapshot_url = snapshot.snapshot_url
        # Upload to S3 or archive storage

    async def backup_neo4j(self, tenant_id: str):
        """Backup Neo4j data for tenant."""

        # Export as JSON
        query = """
        MATCH (n {tenant_id: $tenant_id})
        OPTIONAL MATCH (n)-[r]-(m {tenant_id: $tenant_id})
        RETURN n, r, m
        """

        data = await neo4j.query(tenant_id, query)

        # Save to file
        backup_file = f"backups/neo4j_{tenant_id}_{timestamp}.json"
        with open(backup_file, "w") as f:
            json.dump(data, f)
```

### Data Migration

```python
async def migrate_tenant_data(
    from_tenant_id: str,
    to_tenant_id: str
):
    """Migrate data between tenants."""

    # 1. PostgreSQL
    await db.execute("""
        UPDATE conversations
        SET tenant_id = :to_tenant
        WHERE tenant_id = :from_tenant
    """, {"from_tenant": from_tenant_id, "to_tenant": to_tenant_id})

    # 2. Qdrant (create new collection, copy vectors)
    # ... implementation

    # 3. Neo4j (update tenant_id on all nodes)
    await neo4j.execute_query("""
        MATCH (n {tenant_id: $from_tenant})
        SET n.tenant_id = $to_tenant
    """, from_tenant=from_tenant_id, to_tenant=to_tenant_id)
```

### Data Retention & Cleanup

```python
async def cleanup_old_data(tenant_id: str, retention_days: int = 90):
    """Delete data older than retention period."""

    cutoff = datetime.utcnow() - timedelta(days=retention_days)

    # PostgreSQL: Delete old conversations
    await db.execute("""
        DELETE FROM conversations
        WHERE tenant_id = :tenant_id
        AND created_at < :cutoff
    """, {"tenant_id": tenant_id, "cutoff": cutoff})

    # Qdrant: Delete old embeddings (if timestamp indexed)
    # Neo4j: Delete old nodes
```

---

**Previous**: [Service Layer](./03-service-layer.md) | [Back to Overview](./00-convergence-overview.md)
