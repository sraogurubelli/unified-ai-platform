# Traditional Data Platform

## Overview

The **Traditional Data Platform** provides the foundation for storing and querying transactional, analytical, and cached data using proven database technologies. This stack handles:

- **OLTP (Online Transaction Processing)**: PostgreSQL for ACID transactions
- **OLAP (Online Analytical Processing)**: ClickHouse or StarRocks for analytics
- **Cache & Session Management**: Redis for low-latency data access
- **Multi-tenancy**: Row-level security and tenant isolation across all systems

```
┌──────────────────────────────────────────────────────────────┐
│              Traditional Data Platform                        │
├────────────┬────────────┬─────────────────────────────────────┤
│ OLTP       │ OLAP       │ Cache & Queue                       │
│ PostgreSQL │ ClickHouse │ Redis                               │
│            │ StarRocks  │                                     │
│            │            │                                     │
│ Users      │ Usage      │ Sessions                            │
│ Projects   │ Analytics  │ Rate Limits                         │
│ Sessions   │ Metrics    │ Task Queues                         │
└────────────┴────────────┴─────────────────────────────────────┘
```

---

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

-- AI resources (stored in OLTP for ACID guarantees)
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

---

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
        '{"model": "gpt-4o"}'
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

---

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

---

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

---

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
```

### Data Migration

```python
async def migrate_tenant_data(
    from_tenant_id: str,
    to_tenant_id: str
):
    """Migrate data between tenants."""

    # PostgreSQL
    await db.execute("""
        UPDATE conversations
        SET tenant_id = :to_tenant
        WHERE tenant_id = :from_tenant
    """, {"from_tenant": from_tenant_id, "to_tenant": to_tenant_id})
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
```

---

## Performance SLAs

### Latency Targets

| Operation | Target (p95) | Target (p99) | Measured At |
|-----------|-------------|-------------|-------------|
| **OLTP Operations** |
| Transaction write | <10ms | <20ms | PostgreSQL |
| Simple read query | <5ms | <10ms | PostgreSQL |
| Complex join query | <50ms | <100ms | PostgreSQL |
| **OLAP Operations** |
| Analytics query (1M rows) | <3s | <5s | StarRocks |
| Aggregate query (10M rows) | <10s | <20s | StarRocks |
| Dashboard refresh | <2s | <4s | StarRocks materialized views |
| **Cache Operations** |
| Redis GET/SET | <2ms | <5ms | Redis |
| Session lookup | <5ms | <10ms | Redis |
| Rate limit check | <3ms | <7ms | Redis sorted sets |

### Throughput Targets

| Resource | Target | Scaling Strategy |
|----------|--------|------------------|
| **PostgreSQL** | 5,000 writes/s, 20,000 reads/s | Read replicas + connection pooling |
| **StarRocks** | 100,000 rows/s ingest, 1M rows/s scan | Distributed query execution |
| **Redis** | 100,000 ops/s | Cluster mode with sharding |

### Availability Targets

| Tier | Uptime SLA | Downtime/Year | Deployment Model |
|------|-----------|---------------|------------------|
| **Production (SaaS)** | 99.9% | 8.76 hours | Multi-AZ, auto-failover |
| **Enterprise (Self-hosted)** | 99.95% | 4.38 hours | Multi-region with manual failover |
| **Critical (Regulated)** | 99.99% | 52.6 minutes | Active-active multi-region |

**Recovery Targets**:
- **RTO (Recovery Time Objective)**: <30 minutes (SaaS), <1 hour (self-hosted)
- **RPO (Recovery Point Objective)**: <5 minutes (streaming replication)

---

**Next**: [AI Data Platform](./04b-ai-data-platform.md) | [Integration](./04c-data-integration.md) | [Back to Overview](../00-parallel-stacks-overview.md)
