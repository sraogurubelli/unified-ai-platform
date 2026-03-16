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

## OLAP: StarRocks (Alternative)

### Overview

**StarRocks** is a modern OLAP database optimized for real-time analytics and high-concurrency queries. While ClickHouse excels at batch analytics, StarRocks provides:

- **Real-time data ingestion** with sub-second latency
- **MySQL compatibility** for easier migration
- **Better UPDATE support** (ClickHouse is append-only by design)
- **High-concurrency queries** (100+ concurrent users)
- **Federated queries** across multiple data sources

### ClickHouse vs StarRocks: Comparison

| Feature | ClickHouse | StarRocks | Recommendation |
|---------|-----------|-----------|----------------|
| **Query Speed** | Excellent (batch) | Excellent (real-time) | Tie |
| **Ingestion Latency** | Seconds to minutes | Sub-second | StarRocks |
| **UPDATE/DELETE** | Limited (MergeTree) | Full support | StarRocks |
| **Concurrency** | Good (10-50 queries) | Excellent (100+ queries) | StarRocks |
| **SQL Compatibility** | Custom dialect | MySQL-compatible | StarRocks |
| **Ecosystem** | Mature, large community | Growing | ClickHouse |
| **Operational Complexity** | Moderate | Moderate | Tie |
| **Cost** | Lower (resource usage) | Higher (more memory) | ClickHouse |

### Use Case Mapping

**Use ClickHouse when**:
- ✅ Batch analytics (hourly, daily aggregations)
- ✅ Append-only data (logs, events)
- ✅ Cost-sensitive workloads
- ✅ Complex analytical queries (window functions, aggregations)
- ✅ Time-series data with partitioning

**Use StarRocks when**:
- ✅ Real-time dashboards (sub-second refresh)
- ✅ High-concurrency workloads (100+ users)
- ✅ Data that requires UPDATEs (user profiles, aggregates)
- ✅ MySQL migration (existing MySQL analytics)
- ✅ Federated queries (query across PostgreSQL + ClickHouse + S3)

**Dual-Stack Strategy** (recommended for large deployments):
- **ClickHouse**: Batch analytics, historical aggregations, cost reports
- **StarRocks**: Real-time dashboards, live metrics, user-facing analytics

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    StarRocks Cluster                     │
├──────────────────┬──────────────────┬───────────────────┤
│ Frontend (FE)    │ Backend (BE)     │ Broker (optional) │
│ • Query planning │ • Data storage   │ • External data   │
│ • Metadata       │ • Query exec     │   source access   │
│ • Load balancing │ • Compaction     │   (S3, HDFS)      │
└──────────────────┴──────────────────┴───────────────────┘
         ↓                  ↓                   ↓
    MySQL Protocol    Distributed Storage    External Data
```

### Schema Design

```sql
-- Primary Key table (supports UPDATE/DELETE)
CREATE TABLE usage_realtime (
    event_time DATETIME,
    tenant_id VARCHAR(255),
    user_id VARCHAR(255),
    event_type VARCHAR(50),
    resource_type VARCHAR(50),
    quantity BIGINT,
    cost_usd DECIMAL(10, 4),
    metadata JSON
)
ENGINE = OLAP
PRIMARY KEY (tenant_id, user_id, event_time)
DISTRIBUTED BY HASH(tenant_id) BUCKETS 10
PROPERTIES (
    "replication_num" = "3",
    "storage_medium" = "SSD",
    "enable_persistent_index" = "true"
);

-- Aggregate Key table (pre-aggregation)
CREATE TABLE usage_aggregates (
    date DATE,
    tenant_id VARCHAR(255),
    event_type VARCHAR(50),
    total_quantity BIGINT SUM,
    total_cost DECIMAL(10, 4) SUM,
    event_count BIGINT SUM
)
ENGINE = OLAP
AGGREGATE KEY (date, tenant_id, event_type)
DISTRIBUTED BY HASH(tenant_id) BUCKETS 10
PROPERTIES (
    "replication_num" = "3"
);

-- Materialized View (automatic aggregation)
CREATE MATERIALIZED VIEW usage_hourly AS
SELECT
    date_trunc('hour', event_time) AS hour,
    tenant_id,
    event_type,
    SUM(quantity) AS total_quantity,
    SUM(cost_usd) AS total_cost,
    COUNT(*) AS event_count
FROM usage_realtime
GROUP BY hour, tenant_id, event_type;
```

### Real-Time Ingestion

**Stream Load** (HTTP-based):

```python
import requests
import json

def stream_load_to_starrocks(data: list[dict], table: str, database: str):
    """
    Stream load data into StarRocks via HTTP.

    Supports JSON format with sub-second ingestion latency.
    """
    url = f"http://fe-server:8030/api/{database}/{table}/_stream_load"

    headers = {
        "format": "json",
        "strip_outer_array": "true",
    }

    auth = ("root", "")  # StarRocks username/password

    # Convert to JSON Lines format
    payload = "\n".join(json.dumps(row) for row in data)

    response = requests.put(
        url,
        headers=headers,
        auth=auth,
        data=payload.encode("utf-8"),
    )

    result = response.json()
    if result["Status"] == "Success":
        print(f"Loaded {result['NumberLoadedRows']} rows")
    else:
        print(f"Load failed: {result['Message']}")

# Usage
events = [
    {
        "event_time": "2024-01-15 10:30:00",
        "tenant_id": "acct_123",
        "user_id": "user_456",
        "event_type": "llm_tokens",
        "resource_type": "conversation",
        "quantity": 1500,
        "cost_usd": 0.015,
        "metadata": {"model": "gpt-4o"}
    },
    # ... more events
]

stream_load_to_starrocks(events, "usage_realtime", "analytics")
```

**Routine Load** (Kafka):

```sql
-- Create routine load job from Kafka
CREATE ROUTINE LOAD analytics.usage_kafka_load ON usage_realtime
COLUMNS(event_time, tenant_id, user_id, event_type, resource_type, quantity, cost_usd, metadata)
PROPERTIES
(
    "desired_concurrent_number" = "3",
    "max_batch_interval" = "10",
    "max_batch_rows" = "10000",
    "format" = "json"
)
FROM KAFKA
(
    "kafka_broker_list" = "kafka:9092",
    "kafka_topic" = "usage_events",
    "property.group.id" = "starrocks_usage_load"
);
```

### Querying

**Python Client** (MySQL-compatible):

```python
import pymysql

# Connect via MySQL protocol
conn = pymysql.connect(
    host='fe-server',
    port=9030,
    user='root',
    password='',
    database='analytics'
)

cursor = conn.cursor()

# Real-time query (uses materialized views automatically)
cursor.execute("""
    SELECT
        hour,
        event_type,
        total_quantity,
        total_cost
    FROM usage_hourly
    WHERE tenant_id = %s
        AND hour >= NOW() - INTERVAL 24 HOUR
    ORDER BY hour DESC
""", ('acct_123',))

results = cursor.fetchall()
for row in results:
    print(f"{row[0]}: {row[1]} - {row[2]} units, ${row[3]:.2f}")

cursor.close()
conn.close()
```

**UPDATE Support** (unlike ClickHouse):

```sql
-- Update existing rows (PRIMARY KEY table)
UPDATE usage_realtime
SET cost_usd = cost_usd * 1.1  -- 10% price increase
WHERE tenant_id = 'acct_123'
    AND event_time >= '2024-01-01';

-- DELETE rows
DELETE FROM usage_realtime
WHERE tenant_id = 'acct_123'
    AND event_time < '2023-01-01';  -- Cleanup old data
```

### Federated Queries

**Query across multiple data sources**:

```sql
-- Query PostgreSQL catalog
CREATE EXTERNAL CATALOG postgres_catalog
PROPERTIES
(
    "type" = "jdbc",
    "user" = "cortex",
    "password" = "password",
    "jdbc_uri" = "jdbc:postgresql://postgres:5432/cortex",
    "driver_url" = "https://repo1.maven.org/maven2/org/postgresql/postgresql/42.3.1/postgresql-42.3.1.jar",
    "driver_class" = "org.postgresql.Driver"
);

-- Join StarRocks analytics with PostgreSQL users
SELECT
    u.name,
    u.email,
    SUM(a.total_cost) AS total_spend
FROM usage_aggregates a
JOIN postgres_catalog.cortex.users u
    ON a.tenant_id = u.account_id
WHERE a.date >= '2024-01-01'
GROUP BY u.name, u.email
ORDER BY total_spend DESC
LIMIT 10;
```

### Migration Strategies

#### 1. ClickHouse → StarRocks Migration

```python
async def migrate_clickhouse_to_starrocks():
    """
    Migrate data from ClickHouse to StarRocks.

    Strategy: Export ClickHouse → Transform → Load into StarRocks
    """
    from clickhouse_driver import Client as CHClient
    import pymysql

    # 1. Export from ClickHouse
    ch_client = CHClient(host='clickhouse')
    ch_data = ch_client.execute("""
        SELECT
            timestamp,
            tenant_id,
            event_type,
            resource_type,
            quantity,
            cost_usd
        FROM usage_events
        WHERE toDate(timestamp) >= '2024-01-01'
    """)

    # 2. Transform data (if schema differs)
    transformed = [
        {
            "event_time": row[0].strftime("%Y-%m-%d %H:%M:%S"),
            "tenant_id": row[1],
            "event_type": row[2],
            "resource_type": row[3],
            "quantity": row[4],
            "cost_usd": float(row[5]),
        }
        for row in ch_data
    ]

    # 3. Batch load into StarRocks
    batch_size = 10000
    for i in range(0, len(transformed), batch_size):
        batch = transformed[i:i + batch_size]
        stream_load_to_starrocks(batch, "usage_realtime", "analytics")
        print(f"Loaded batch {i // batch_size + 1}")
```

#### 2. Parallel Writes (Transition Period)

```python
class DualWriteAnalytics:
    """
    Write to both ClickHouse and StarRocks during migration.

    Gradual migration strategy:
    1. Dual-write for 30 days
    2. Verify data consistency
    3. Switch reads to StarRocks
    4. Stop ClickHouse writes
    """

    def __init__(self, ch_client, sr_conn):
        self.ch = ch_client
        self.sr = sr_conn

    async def record_event(self, event: dict):
        """Record event to both systems."""

        # Write to ClickHouse
        try:
            self.ch.execute(
                "INSERT INTO usage_events VALUES",
                [(
                    event["timestamp"],
                    event["tenant_id"],
                    event["event_type"],
                    event["resource_type"],
                    event["quantity"],
                    event["cost_usd"],
                    json.dumps(event.get("metadata", {}))
                )]
            )
        except Exception as e:
            logger.error(f"ClickHouse write failed: {e}")

        # Write to StarRocks
        try:
            stream_load_to_starrocks([event], "usage_realtime", "analytics")
        except Exception as e:
            logger.error(f"StarRocks write failed: {e}")
            # Fallback: ClickHouse is source of truth during migration
```

#### 3. Schema Compatibility

```sql
-- ClickHouse schema
CREATE TABLE usage_events (
    timestamp DateTime,
    tenant_id String,
    event_type String,
    resource_type String,
    quantity UInt64,
    cost_usd Float64,
    metadata String
) ENGINE = MergeTree()
ORDER BY (tenant_id, event_type, timestamp);

-- Equivalent StarRocks schema
CREATE TABLE usage_realtime (
    event_time DATETIME,      -- Renamed for clarity
    tenant_id VARCHAR(255),
    event_type VARCHAR(50),
    resource_type VARCHAR(50),
    quantity BIGINT,          -- UInt64 → BIGINT
    cost_usd DECIMAL(10, 4),  -- More precise than Float64
    metadata JSON             -- Native JSON type
)
ENGINE = OLAP
PRIMARY KEY (tenant_id, event_time)
DISTRIBUTED BY HASH(tenant_id);
```

### Performance Optimization

**Partition Pruning**:

```sql
-- Add date partition for efficient queries
CREATE TABLE usage_realtime (
    event_time DATETIME,
    tenant_id VARCHAR(255),
    -- ... other columns
)
ENGINE = OLAP
PRIMARY KEY (tenant_id, event_time)
PARTITION BY RANGE(event_time) (
    PARTITION p20240101 VALUES LESS THAN ("2024-02-01"),
    PARTITION p20240201 VALUES LESS THAN ("2024-03-01"),
    PARTITION p20240301 VALUES LESS THAN ("2024-04-01")
)
DISTRIBUTED BY HASH(tenant_id);

-- Query only scans relevant partition
SELECT * FROM usage_realtime
WHERE event_time >= '2024-01-15'
    AND event_time < '2024-01-16';  -- Only scans p20240101
```

**Colocate Join**:

```sql
-- Distribute both tables by tenant_id for local joins
CREATE TABLE usage_realtime (...)
DISTRIBUTED BY HASH(tenant_id) BUCKETS 10
PROPERTIES ("colocate_with" = "tenant_group");

CREATE TABLE tenant_configs (...)
DISTRIBUTED BY HASH(tenant_id) BUCKETS 10
PROPERTIES ("colocate_with" = "tenant_group");

-- Join is local (no data shuffle)
SELECT u.*, c.config_value
FROM usage_realtime u
JOIN tenant_configs c
    ON u.tenant_id = c.tenant_id;
```

### Deployment Considerations

**Resource Requirements**:

```yaml
# StarRocks cluster (minimum)
frontends:
  replicas: 3  # High availability
  cpu: 4 cores
  memory: 16 GB

backends:
  replicas: 3  # Replication factor = 3
  cpu: 16 cores
  memory: 64 GB
  storage: 1 TB SSD
```

**Monitoring**:

```python
import pymysql

def check_starrocks_health():
    """Check StarRocks cluster health."""

    conn = pymysql.connect(host='fe-server', port=9030, user='root')
    cursor = conn.cursor()

    # Check backend health
    cursor.execute("SHOW BACKENDS")
    backends = cursor.fetchall()

    alive_count = sum(1 for b in backends if b[9] == 'true')  # Alive column
    print(f"Backends alive: {alive_count}/{len(backends)}")

    # Check load jobs
    cursor.execute("SHOW ROUTINE LOAD")
    jobs = cursor.fetchall()

    running = sum(1 for j in jobs if j[7] == 'RUNNING')
    print(f"Routine load jobs running: {running}/{len(jobs)}")

    cursor.close()
    conn.close()
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

## Performance SLAs

### Latency Targets

| Operation | Target (p95) | Target (p99) | Measured At |
|-----------|-------------|-------------|-------------|
| **OLTP Operations** |
| Transaction write | <10ms | <20ms | PostgreSQL |
| Simple read query | <5ms | <10ms | PostgreSQL |
| Complex join query | <50ms | <100ms | PostgreSQL |
| **Vector Operations** |
| Vector search (10M vectors) | <50ms | <100ms | Qdrant |
| Batch embedding (100 docs) | <500ms | <1s | Embedding service |
| Semantic cache lookup | <20ms | <40ms | Qdrant |
| **Graph Operations** |
| 1-hop traversal | <30ms | <60ms | Neo4j |
| 2-hop traversal | <100ms | <200ms | Neo4j |
| Entity extraction | <150ms | <300ms | Neo4j + LLM |
| **OLAP Operations** |
| Analytics query (1M rows) | <3s | <5s | StarRocks |
| Aggregate query (10M rows) | <10s | <20s | StarRocks |
| Dashboard refresh | <2s | <4s | StarRocks materialized views |
| **Cache Operations** |
| Redis GET/SET | <2ms | <5ms | Redis |
| Session lookup | <5ms | <10ms | Redis |
| Rate limit check | <3ms | <7ms | Redis sorted sets |
| **End-to-End SLAs** |
| Chat response (first token) | <200ms | <400ms | API Gateway → LLM |
| Chat response (complete) | <2s | <5s | Full conversation context |
| RAG retrieval + generation | <3s | <6s | Vector search + LLM |
| Knowledge graph query + LLM | <4s | <8s | Graph traversal + LLM |

### Throughput Targets

| Resource | Target | Scaling Strategy |
|----------|--------|------------------|
| **API Gateway** | 10,000 req/s | Horizontal scaling (K8s HPA) |
| **PostgreSQL** | 5,000 writes/s, 20,000 reads/s | Read replicas + connection pooling |
| **StarRocks** | 100,000 rows/s ingest, 1M rows/s scan | Distributed query execution |
| **Qdrant** | 1,000 vector writes/s, 5,000 searches/s | Sharding by tenant |
| **Neo4j** | 2,000 writes/s, 10,000 reads/s | Read replicas + caching |
| **Redis** | 100,000 ops/s | Cluster mode with sharding |
| **LLM Gateway** | 500 concurrent requests | Queue-based backpressure |

### Availability Targets

| Tier | Uptime SLA | Downtime/Year | Deployment Model |
|------|-----------|---------------|------------------|
| **Production (SaaS)** | 99.9% | 8.76 hours | Multi-AZ, auto-failover |
| **Enterprise (Self-hosted)** | 99.95% | 4.38 hours | Multi-region with manual failover |
| **Critical (Regulated)** | 99.99% | 52.6 minutes | Active-active multi-region |

**Recovery Targets**:
- **RTO (Recovery Time Objective)**: <30 minutes (SaaS), <1 hour (self-hosted)
- **RPO (Recovery Point Objective)**: <5 minutes (streaming replication)

### Cache Hit Rates

| Cache Type | Target Hit Rate | Impact |
|------------|----------------|--------|
| **Semantic Cache (Qdrant)** | >85% | 90% cost savings on LLM calls |
| **Session Cache (Redis)** | >95% | <5ms auth checks |
| **Query Cache (StarRocks)** | >70% | 10x faster dashboard loads |
| **CDN Cache (Static Assets)** | >98% | Reduced origin server load |

### Monitoring & Alerts

```python
# Prometheus metrics for SLA tracking
from prometheus_client import Histogram, Counter, Gauge

# Latency histograms
db_query_duration = Histogram(
    'db_query_duration_seconds',
    'Database query duration',
    ['database', 'operation'],
    buckets=[0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0]
)

vector_search_duration = Histogram(
    'vector_search_duration_seconds',
    'Vector search duration',
    ['collection'],
    buckets=[0.01, 0.02, 0.05, 0.1, 0.2, 0.5, 1.0]
)

graph_query_duration = Histogram(
    'graph_query_duration_seconds',
    'Graph query duration',
    ['hops'],
    buckets=[0.03, 0.05, 0.1, 0.2, 0.5, 1.0, 2.0]
)

# Cache hit rates
cache_hits = Counter('cache_hits_total', 'Cache hits', ['cache_type'])
cache_misses = Counter('cache_misses_total', 'Cache misses', ['cache_type'])

# Availability
service_up = Gauge('service_up', 'Service availability', ['service'])

# Usage example
async def execute_vector_search(query: str, tenant_id: str):
    with vector_search_duration.labels(collection=f"embeddings_{tenant_id}").time():
        results = await qdrant.search(...)
    return results
```

**Alerting Rules** (Prometheus):
```yaml
groups:
  - name: sla_violations
    interval: 30s
    rules:
      # Latency violations
      - alert: HighVectorSearchLatency
        expr: histogram_quantile(0.95, vector_search_duration_seconds) > 0.05
        for: 5m
        annotations:
          summary: "Vector search p95 latency exceeds 50ms SLA"

      - alert: HighDatabaseLatency
        expr: histogram_quantile(0.95, db_query_duration_seconds) > 0.01
        for: 5m
        annotations:
          summary: "Database p95 latency exceeds 10ms SLA"

      # Cache hit rate violations
      - alert: LowSemanticCacheHitRate
        expr: rate(cache_hits_total{cache_type="semantic"}[5m]) / (rate(cache_hits_total{cache_type="semantic"}[5m]) + rate(cache_misses_total{cache_type="semantic"}[5m])) < 0.85
        for: 10m
        annotations:
          summary: "Semantic cache hit rate below 85% target"

      # Availability violations
      - alert: ServiceDown
        expr: service_up == 0
        for: 1m
        annotations:
          summary: "Service unavailable"
```

### Performance Optimization Strategies

| Strategy | Implementation | Impact |
|----------|----------------|--------|
| **Database Connection Pooling** | PgBouncer (transaction mode), pool size 100 | 5x more concurrent connections |
| **Read Replicas** | PostgreSQL streaming replication (2 replicas) | 3x read throughput |
| **Materialized Views** | StarRocks auto-refresh (1-minute interval) | 100x faster analytics queries |
| **Vector Index Optimization** | HNSW index (M=16, ef_construct=100) | <50ms search on 10M vectors |
| **Graph Index Strategy** | Composite indexes on (tenant_id, name) | 10x faster entity lookups |
| **Query Result Caching** | Redis TTL-based cache (5-minute default) | 95% reduction in repeated queries |
| **Batch Processing** | Kafka + Spark Streaming (micro-batches) | 10x ingest throughput |
| **Auto-scaling** | KEDA metrics-based (CPU >70%, queue depth >100) | Dynamic capacity management |

## 5-Layer Data Architecture

Our data platform follows a layered architecture pattern, separating concerns from data ingestion through to consumption.

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        CONSUMER LAYER                                │
│  Dashboards (Grafana) · BI Tools · APIs · LLM Context Retrieval     │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────────┐
│                         QUERY LAYER                                  │
│  Unified Query API · SQL (StarRocks) · Cypher (Neo4j) ·             │
│  Vector Similarity (Qdrant) · Logical Federation                    │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────────┐
│                       PROCESSING LAYER                               │
│  Materialized Views (StarRocks) · Vector Indexing (Qdrant) ·        │
│  Graph Algorithms (Neo4j) · Event Stream Processing (Kafka)         │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────────┐
│                        STORAGE LAYER                                 │
│  PostgreSQL (OLTP) · StarRocks (OLAP) · Qdrant (Vector) ·           │
│  Neo4j (Graph) · Redis (Cache/Session) · S3 (Archive)               │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────────┐
│                       INGESTION LAYER                                │
│  Kafka Streaming · Spark Streaming · Schema Validation ·            │
│  Data Contracts · Event Bus · Batch Import                          │
└─────────────────────────────────────────────────────────────────────┘
```

### Layer 1: Ingestion

**Responsibility**: Collect, validate, and route data from multiple sources.

**Components**:
- **Kafka Topics**: Event streaming platform
  - `llm-requests`: LLM Gateway usage events
  - `conversations`: Chat message events
  - `entity-extractions`: Knowledge Graph updates
  - `audit-logs`: Security and compliance events
- **Spark Streaming**: Micro-batch processing (5-second windows)
- **Schema Registry**: Enforce data contracts (Avro/Protobuf schemas)
- **Event Router**: Route events to appropriate storage systems

**Data Contracts** (Enforced at Ingestion):
```python
from pydantic import BaseModel, validator
from datetime import datetime

class LLMRequestEvent(BaseModel):
    """Schema for LLM request events."""

    tenant_id: str
    request_id: str
    model: str
    prompt_tokens: int
    completion_tokens: int
    latency_ms: float
    cost_usd: float
    timestamp: datetime

    @validator('prompt_tokens', 'completion_tokens')
    def validate_positive(cls, v):
        if v < 0:
            raise ValueError("Token count must be positive")
        return v

    @validator('cost_usd')
    def validate_cost(cls, v):
        if v < 0 or v > 100:  # Sanity check
            raise ValueError("Cost out of expected range")
        return v

# Schema enforcement at ingestion
async def ingest_llm_event(event_data: dict):
    try:
        # Validate against schema
        validated_event = LLMRequestEvent(**event_data)

        # Publish to Kafka
        await kafka_producer.send(
            topic="llm-requests",
            value=validated_event.model_dump_json()
        )
    except ValidationError as e:
        # Log schema violation
        logger.error(f"Schema validation failed: {e}")
        await publish_to_dead_letter_queue(event_data, error=str(e))
```

**Performance**:
- Throughput: 100,000 events/second (Kafka + Spark)
- Latency: <100ms (producer → Kafka), <2 minutes (Kafka → storage)
- Schema validation overhead: <5ms per event

### Layer 2: Storage

**Responsibility**: Persist data optimized for different access patterns.

**Storage Systems**:

| System | Use Case | Data Model | Access Pattern |
|--------|----------|------------|----------------|
| **PostgreSQL** | Transactional data (conversations, messages, users) | Relational (normalized) | ACID transactions, strong consistency |
| **StarRocks** | Analytics, dashboards, aggregations | Columnar OLAP | Fast scans, aggregations on large datasets |
| **Qdrant** | Semantic search, embeddings, semantic cache | Vector database | Similarity search (cosine/dot product) |
| **Neo4j** | Knowledge graph, entity relationships | Property graph | Graph traversal, relationship queries |
| **Redis** | Session state, rate limits, caching | Key-value, sorted sets, streams | Sub-millisecond reads/writes |
| **S3** | Long-term archival, cold storage | Object storage | Infrequent access, compliance retention |

**Data Lifecycle**:
```
Hot Data (Active Use)
  → PostgreSQL (7 days) + StarRocks (30 days) + Redis (24 hours)

Warm Data (Recent History)
  → StarRocks (90 days) + S3 Glacier Instant Retrieval

Cold Data (Archive)
  → S3 Glacier Deep Archive (7 years, compliance)
```

### Layer 3: Processing

**Responsibility**: Transform, index, and optimize data for query performance.

**Processing Patterns**:

1. **Materialized Views (StarRocks)**
   - **Purpose**: Pre-aggregate analytics data for fast dashboard queries
   - **Refresh**: Auto-refresh every 1 minute (incremental updates)
   - **Example**:
     ```sql
     -- Materialized view for LLM usage dashboard
     CREATE MATERIALIZED VIEW mv_llm_usage_by_tenant
     REFRESH ASYNC START('2024-01-01 00:00:00') EVERY(INTERVAL 1 MINUTE)
     AS
     SELECT
       tenant_id,
       date_trunc('hour', timestamp) AS hour,
       COUNT(*) AS request_count,
       SUM(prompt_tokens) AS total_prompt_tokens,
       SUM(completion_tokens) AS total_completion_tokens,
       SUM(cost_usd) AS total_cost,
       AVG(latency_ms) AS avg_latency,
       PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY latency_ms) AS p95_latency
     FROM llm_requests
     GROUP BY tenant_id, hour;
     ```

2. **Vector Indexing (Qdrant)**
   - **Purpose**: Build HNSW indexes for fast similarity search
   - **Index Parameters**: M=16 (links per node), ef_construct=100 (index quality)
   - **Performance**: <50ms search on 10M vectors

3. **Graph Algorithms (Neo4j)**
   - **Purpose**: Compute entity importance, community detection
   - **Algorithms**:
     - PageRank: Identify influential entities
     - Community Detection (Louvain): Group related entities
     - Shortest Path: Find relationship chains
   - **Execution**: Background jobs (scheduled via Airflow)

4. **Event Stream Processing (Kafka + Spark)**
   - **Purpose**: Real-time aggregations, anomaly detection
   - **Window Types**:
     - Tumbling windows (5-minute non-overlapping)
     - Sliding windows (1-hour with 15-minute slide)
     - Session windows (user activity bursts)
   - **Example**:
     ```python
     from pyspark.sql.functions import window, count, avg

     # Real-time rate limiting enforcement
     llm_requests_stream = (
         spark.readStream
         .format("kafka")
         .option("kafka.bootstrap.servers", "kafka:9092")
         .option("subscribe", "llm-requests")
         .load()
     )

     # 1-minute tumbling window for rate limiting
     rate_limit_violations = (
         llm_requests_stream
         .groupBy(
             window(col("timestamp"), "1 minute"),
             col("tenant_id")
         )
         .agg(count("*").alias("requests_per_minute"))
         .filter(col("requests_per_minute") > 1000)  # Threshold
     )

     # Alert on violations
     rate_limit_violations.writeStream \
         .foreachBatch(send_rate_limit_alert) \
         .start()
     ```

### Layer 4: Query

**Responsibility**: Provide unified, optimized access to data across all storage systems.

**Unified Query API**:
```python
from enum import Enum

class QueryType(Enum):
    SQL = "sql"           # StarRocks OLAP queries
    VECTOR = "vector"     # Qdrant similarity search
    GRAPH = "graph"       # Neo4j Cypher queries
    HYBRID = "hybrid"     # Combined vector + graph (Graph RAG)

class UnifiedQueryEngine:
    """Unified query interface across data stores."""

    async def execute(
        self,
        query_type: QueryType,
        query: str,
        tenant_id: str,
        params: dict = None
    ) -> list[dict]:
        """Execute query with automatic tenant isolation."""

        if query_type == QueryType.SQL:
            return await self._execute_sql(query, tenant_id, params)
        elif query_type == QueryType.VECTOR:
            return await self._execute_vector_search(query, tenant_id, params)
        elif query_type == QueryType.GRAPH:
            return await self._execute_graph_query(query, tenant_id, params)
        elif query_type == QueryType.HYBRID:
            return await self._execute_hybrid_query(query, tenant_id, params)

    async def _execute_hybrid_query(
        self,
        query: str,
        tenant_id: str,
        params: dict
    ) -> list[dict]:
        """Graph RAG: Combine vector search + graph traversal."""

        # 1. Vector search for relevant documents
        vector_results = await self.qdrant.search(
            collection_name=f"embeddings_{tenant_id}",
            query_vector=await self.embed(query),
            limit=10
        )

        # 2. Extract entities from results
        entities = [doc.payload.get("entities", []) for doc in vector_results]

        # 3. Graph traversal to find related entities
        graph_results = await self.neo4j.query(
            tenant_id,
            """
            MATCH (e:Entity)
            WHERE e.name IN $entity_names AND e.tenant_id = $tenant_id
            MATCH (e)-[r*1..2]-(related)
            WHERE related.tenant_id = $tenant_id
            RETURN DISTINCT related.name, related.type, related.attributes
            """,
            {"entity_names": entities, "tenant_id": tenant_id}
        )

        # 4. Combine results using Reciprocal Rank Fusion (RRF)
        combined_results = self._reciprocal_rank_fusion(
            vector_results,
            graph_results,
            k=60
        )

        return combined_results

    def _reciprocal_rank_fusion(
        self,
        vector_results: list,
        graph_results: list,
        k: int = 60
    ) -> list[dict]:
        """RRF algorithm for combining ranked lists."""

        scores = {}

        # Score vector results
        for rank, result in enumerate(vector_results, start=1):
            doc_id = result.id
            scores[doc_id] = scores.get(doc_id, 0) + 1 / (k + rank)

        # Score graph results
        for rank, result in enumerate(graph_results, start=1):
            entity_name = result["name"]
            scores[entity_name] = scores.get(entity_name, 0) + 1 / (k + rank)

        # Sort by combined score
        ranked = sorted(scores.items(), key=lambda x: x[1], reverse=True)

        return [{"id": doc_id, "rrf_score": score} for doc_id, score in ranked]
```

**Query Federation**:
- Join data across PostgreSQL (transactional) + StarRocks (analytics)
- Enrich vector search results with graph context
- Cache frequent queries in Redis (5-minute TTL)

**Performance Optimizations**:
- **Query Planning**: Cost-based optimizer chooses optimal storage system
- **Parallel Execution**: Fan-out queries to multiple databases concurrently
- **Result Streaming**: Stream large result sets instead of buffering
- **Pushdown Filters**: Apply tenant_id filters at storage layer

### Layer 5: Consumer

**Responsibility**: Deliver data to end users and applications.

**Consumer Types**:

1. **Dashboards (Grafana)**
   - **Data Source**: StarRocks (analytics), Prometheus (metrics)
   - **Refresh Rate**: 30 seconds (real-time), 5 minutes (batch aggregations)
   - **Performance**: <2s dashboard load time (materialized views)
   - **Example Dashboards**:
     - LLM Usage by Tenant (cost, tokens, request volume)
     - System Performance (latency p95/p99, error rates, cache hit rates)
     - Knowledge Graph Metrics (entity count, relationship density)

2. **BI Tools (Tableau, Metabase)**
   - **Data Source**: StarRocks via JDBC/ODBC
   - **Access Pattern**: Ad-hoc queries, exploratory analysis
   - **Security**: Row-level security (tenant_id filter enforced)

3. **REST APIs**
   - **Data Source**: PostgreSQL (transactional), StarRocks (analytics), Redis (cache)
   - **Response Format**: JSON, Protocol Buffers
   - **Pagination**: Cursor-based (efficient for large result sets)
   - **Rate Limiting**: 1,000 requests/minute per tenant (Redis sliding window)

4. **LLM Context Retrieval**
   - **Data Source**: Qdrant (semantic search), Neo4j (graph context)
   - **Pattern**: Graph RAG (vector + graph fusion)
   - **Latency**: <3s end-to-end (retrieval + LLM generation)
   - **Context Window**: Top 5 vector results + 2-hop graph neighbors

**Access Patterns**:
```python
# Dashboard consumer (pull model)
async def get_llm_usage_dashboard(tenant_id: str, time_range: str):
    """Fetch dashboard data from materialized view."""

    query = """
    SELECT hour, request_count, total_cost, avg_latency, p95_latency
    FROM mv_llm_usage_by_tenant
    WHERE tenant_id = :tenant_id
      AND hour >= NOW() - INTERVAL :time_range
    ORDER BY hour DESC
    """

    return await starrocks.query(query, {
        "tenant_id": tenant_id,
        "time_range": time_range
    })

# LLM context consumer (on-demand)
async def retrieve_context_for_llm(query: str, tenant_id: str):
    """Retrieve context using Graph RAG."""

    # Unified query engine handles vector + graph fusion
    context_docs = await query_engine.execute(
        query_type=QueryType.HYBRID,
        query=query,
        tenant_id=tenant_id,
        params={"top_k": 5, "max_hops": 2}
    )

    # Format for LLM consumption
    context_text = "\n\n".join([
        f"Document {i+1}:\n{doc['content']}"
        for i, doc in enumerate(context_docs)
    ])

    return context_text
```

**Consumer Performance**:
- **Dashboard Load Time**: <2s (via materialized views)
- **API Response Time**: <100ms (cached), <500ms (database)
- **LLM Context Retrieval**: <3s (Graph RAG)
- **BI Query Execution**: <5s (1M rows), <20s (10M rows)

### Data Flow Example: LLM Request Analytics

```
1. INGESTION
   User makes LLM request → LLM Gateway logs event → Kafka topic "llm-requests"
   Schema validation: ✓ tenant_id, model, tokens, cost, latency

2. STORAGE
   Kafka → Spark Streaming → Parallel writes:
     - StarRocks: Columnar storage for analytics
     - PostgreSQL: Transactional record (request_id, conversation_id)
     - Redis: Increment rate limit counter (sliding window)

3. PROCESSING
   StarRocks: Auto-refresh materialized view (1-minute interval)
   Aggregates: COUNT(*), SUM(cost), AVG(latency), PERCENTILE(95, latency)
   Grouping: tenant_id, hour

4. QUERY
   Dashboard query: SELECT * FROM mv_llm_usage_by_tenant WHERE tenant_id = 'acct_123'
   Query hits materialized view (pre-aggregated) → <2s response

5. CONSUMER
   Grafana dashboard displays:
     - Total requests (last 24 hours): 15,432
     - Total cost: $124.56
     - Avg latency: 1,234ms
     - p95 latency: 2,456ms
   Auto-refresh: Every 30 seconds
```

### Cross-Layer Concerns

**Tenant Isolation** (Applied at All Layers):
- **Ingestion**: Validate tenant_id on every event
- **Storage**: tenant_id indexed on all tables/collections
- **Processing**: Tenant-specific materialized views
- **Query**: Automatic tenant_id filter injection
- **Consumer**: API-level tenant context enforcement

**Schema Evolution**:
- **Backward Compatibility**: New fields optional, old consumers unaffected
- **Schema Registry**: Centralized schema versioning (Confluent Schema Registry)
- **Migration Strategy**: Dual-write during transition, gradual cutover

**Monitoring** (Per Layer):
```yaml
# Ingestion Layer
- kafka_consumer_lag_seconds{topic="llm-requests"} < 60
- schema_validation_failures_total < 10/minute

# Storage Layer
- db_connection_pool_utilization < 80%
- disk_usage_percent < 85%

# Processing Layer
- materialized_view_refresh_lag_seconds < 120
- spark_job_failure_rate < 1%

# Query Layer
- query_duration_p95_seconds < 3
- cache_hit_rate > 70%

# Consumer Layer
- dashboard_load_time_seconds < 2
- api_error_rate < 0.1%
```

---

**Previous**: [Service Layer](./03-service-layer.md) | [Back to Overview](./00-convergence-overview.md)
