# Data Platform Integration

## Overview

The **Data Platform Integration** layer connects the **Traditional Data Platform** (PostgreSQL, ClickHouse, Redis) with the **AI Data Platform** (Qdrant, Neo4j, Semantic Cache), enabling seamless data flows and unified analytics across both stacks.

```
┌──────────────────────────────────────────────────────────────┐
│              Traditional Data Platform                        │
│   PostgreSQL (OLTP) · ClickHouse (OLAP) · Redis (Cache)      │
└────────────────────────┬─────────────────────────────────────┘
                         │
                         │ Integration Points:
                         │ • Event Bus (Kafka)
                         │ • Data Sync (Real-time & Batch)
                         │ • Federated Queries
                         │ • Unified Analytics
                         │
┌────────────────────────▼─────────────────────────────────────┐
│                 AI Data Platform                              │
│   Qdrant (Vector) · Neo4j (Graph) · Semantic Cache           │
└──────────────────────────────────────────────────────────────┘
```

---

## Data Flows Between Stacks

### Flow 1: Conversations → Embeddings

**Purpose**: Convert conversation messages to embeddings for semantic search.

```
PostgreSQL (messages)
    ↓ [Kafka event: message.created]
Embedding Service
    ↓ [Generate embeddings]
Qdrant (vector store)
```

**Implementation**:

```python
from kafka import KafkaConsumer
import asyncio

class ConversationToEmbeddingsPipeline:
    """Stream conversations to vector database."""

    async def consume_message_events(self):
        """Listen for new messages and generate embeddings."""

        consumer = KafkaConsumer(
            'message.created',
            bootstrap_servers='kafka:9092',
            group_id='embedding-pipeline'
        )

        for message in consumer:
            event = json.loads(message.value)

            # Extract message content
            message_id = event['message_id']
            content = event['content']
            tenant_id = event['tenant_id']
            conversation_id = event['conversation_id']

            # Generate embedding
            embedding = await self.embedding_service.embed(content)

            # Store in Qdrant
            await self.qdrant_client.upsert(
                collection_name=f"embeddings_{tenant_id}",
                points=[
                    PointStruct(
                        id=message_id,
                        vector=embedding,
                        payload={
                            "message_id": message_id,
                            "conversation_id": conversation_id,
                            "content": content,
                            "timestamp": event['created_at']
                        }
                    )
                ]
            )

            logger.info(f"Embedded message {message_id} for tenant {tenant_id}")
```

### Flow 2: Messages → Knowledge Graph

**Purpose**: Extract entities and relationships from conversations to build knowledge graph.

```
PostgreSQL (messages)
    ↓ [Kafka event: message.created]
Entity Extraction Service (LLM)
    ↓ [Extract entities, relationships]
Neo4j (knowledge graph)
```

**Implementation**:

```python
class MessageToKnowledgeGraphPipeline:
    """Extract entities from messages and build knowledge graph."""

    async def consume_message_events(self):
        """Listen for new messages and extract entities."""

        consumer = KafkaConsumer(
            'message.created',
            bootstrap_servers='kafka:9092',
            group_id='knowledge-graph-pipeline'
        )

        for message in consumer:
            event = json.loads(message.value)

            # Extract entities using LLM
            entities = await self.extract_entities(
                content=event['content'],
                tenant_id=event['tenant_id']
            )

            # Store in Neo4j
            for entity in entities['entities']:
                await self.neo4j_store.create_entity(
                    tenant_id=event['tenant_id'],
                    entity_type=entity['type'],
                    name=entity['name'],
                    attributes=entity.get('attributes', {})
                )

            for relationship in entities['relationships']:
                await self.neo4j_store.create_relationship(
                    tenant_id=event['tenant_id'],
                    from_entity=relationship['from'],
                    to_entity=relationship['to'],
                    relationship_type=relationship['type'],
                    attributes=relationship.get('attributes', {})
                )

    async def extract_entities(self, content: str, tenant_id: str) -> dict:
        """Use LLM to extract entities and relationships."""

        prompt = f"""
        Extract entities and relationships from the following text.
        Return JSON format with entities and relationships.

        Text: {content}
        """

        llm_response = await self.llm_gateway.complete(prompt)
        return json.loads(llm_response)
```

### Flow 3: LLM Usage → Analytics

**Purpose**: Track LLM usage and costs for analytics.

```
LLM Gateway (usage metrics)
    ↓ [Kafka event: llm.request.completed]
ClickHouse (analytics)
    ↓ [Materialized views]
Analytics Dashboards
```

**Implementation**:

```python
class LLMUsageAnalyticsPipeline:
    """Stream LLM usage to analytics database."""

    async def consume_llm_events(self):
        """Listen for LLM requests and log to analytics."""

        consumer = KafkaConsumer(
            'llm.request.completed',
            bootstrap_servers='kafka:9092',
            group_id='analytics-pipeline'
        )

        batch = []
        for message in consumer:
            event = json.loads(message.value)

            batch.append({
                'timestamp': event['timestamp'],
                'tenant_id': event['tenant_id'],
                'model': event['model'],
                'prompt_tokens': event['prompt_tokens'],
                'completion_tokens': event['completion_tokens'],
                'total_tokens': event['total_tokens'],
                'cost_usd': event['cost_usd'],
                'latency_ms': event['latency_ms'],
                'cached': event.get('cached', False)
            })

            # Batch insert to ClickHouse
            if len(batch) >= 100:
                await self.clickhouse_client.insert(
                    'llm_usage_events',
                    batch
                )
                batch = []
```

### Flow 4: Semantic Cache → Cost Tracking

**Purpose**: Track semantic cache performance and cost savings.

```
Semantic Cache (Qdrant)
    ↓ [Cache hits/misses]
Redis (real-time metrics)
    ↓ [Aggregate metrics]
PostgreSQL (cost_savings table)
```

**Implementation**:

```python
class SemanticCacheMetricsPipeline:
    """Track semantic cache performance."""

    async def record_cache_event(
        self,
        tenant_id: str,
        query: str,
        cache_hit: bool,
        cost_saved: float = 0.0
    ):
        """Record cache hit/miss and cost savings."""

        # Real-time metrics (Redis)
        await self.redis.incr(f"cache_hits:{tenant_id}" if cache_hit else f"cache_misses:{tenant_id}")

        # Cost savings (PostgreSQL)
        if cache_hit:
            await self.db.execute("""
                INSERT INTO semantic_cache_savings (tenant_id, query, cost_saved, timestamp)
                VALUES (:tenant_id, :query, :cost_saved, NOW())
            """, {
                "tenant_id": tenant_id,
                "query": query,
                "cost_saved": cost_saved
            })

        # Analytics (ClickHouse)
        await self.clickhouse.execute("""
            INSERT INTO cache_events (timestamp, tenant_id, cache_hit, cost_saved)
            VALUES
        """, [(
            datetime.now(),
            tenant_id,
            1 if cache_hit else 0,
            cost_saved
        )])
```

---

## Event Bus Integration

### Kafka Topics

| Topic | Producer | Consumer | Purpose |
|-------|----------|----------|---------|
| `message.created` | Chat Service (Traditional) | Embedding Pipeline, Entity Extraction (AI) | Sync messages to AI systems |
| `llm.request.completed` | LLM Gateway (AI) | Analytics Pipeline (Traditional) | Track LLM usage |
| `entity.extracted` | Entity Extraction (AI) | Knowledge Graph Builder (AI) | Build knowledge graph |
| `cache.event` | Semantic Cache (AI) | Metrics Pipeline (Traditional) | Track cache performance |

### Event Schema

**message.created**:
```json
{
  "message_id": "msg_123",
  "conversation_id": "conv_456",
  "tenant_id": "acct_789",
  "content": "What is the weather today?",
  "role": "user",
  "created_at": "2024-01-15T10:30:00Z"
}
```

**llm.request.completed**:
```json
{
  "request_id": "req_123",
  "tenant_id": "acct_789",
  "model": "gpt-4o",
  "prompt_tokens": 100,
  "completion_tokens": 50,
  "total_tokens": 150,
  "cost_usd": 0.0015,
  "latency_ms": 1234,
  "cached": false,
  "timestamp": "2024-01-15T10:30:05Z"
}
```

---

## Data Synchronization Patterns

### Real-Time Sync (Streaming)

**Use Case**: Immediate propagation of changes (e.g., new messages → embeddings).

```python
class RealTimeSyncPipeline:
    """Real-time data sync using Kafka."""

    async def sync_postgres_to_qdrant(self):
        """Stream PostgreSQL changes to Qdrant in real-time."""

        # PostgreSQL CDC (Change Data Capture) via Debezium
        consumer = KafkaConsumer(
            'postgres.cortex.messages',  # Debezium topic
            bootstrap_servers='kafka:9092'
        )

        for message in consumer:
            event = json.loads(message.value)

            if event['op'] == 'c':  # CREATE operation
                # Generate embedding
                embedding = await self.embedding_service.embed(event['after']['content'])

                # Store in Qdrant
                await self.qdrant.upsert(
                    collection_name=f"embeddings_{event['after']['tenant_id']}",
                    points=[PointStruct(
                        id=event['after']['id'],
                        vector=embedding,
                        payload=event['after']
                    )]
                )

            elif event['op'] == 'd':  # DELETE operation
                # Remove from Qdrant
                await self.qdrant.delete(
                    collection_name=f"embeddings_{event['before']['tenant_id']}",
                    points_selector=[event['before']['id']]
                )
```

### Batch Sync (Scheduled)

**Use Case**: Periodic synchronization for non-critical data (e.g., analytics aggregates).

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import timedelta

# Airflow DAG for batch sync
default_args = {
    'owner': 'data-platform',
    'retries': 3,
    'retry_delay': timedelta(minutes=5)
}

dag = DAG(
    'sync_postgres_to_clickhouse',
    default_args=default_args,
    schedule_interval='@hourly',
    catchup=False
)

def sync_conversations_to_analytics():
    """Batch sync conversations from PostgreSQL to ClickHouse."""

    # Extract from PostgreSQL
    pg_data = pg_client.execute("""
        SELECT
            id,
            tenant_id,
            created_at,
            updated_at,
            message_count
        FROM conversations
        WHERE updated_at >= NOW() - INTERVAL '1 hour'
    """)

    # Transform
    transformed = [
        {
            'conversation_id': row[0],
            'tenant_id': row[1],
            'created_at': row[2],
            'message_count': row[4]
        }
        for row in pg_data
    ]

    # Load to ClickHouse
    clickhouse_client.insert('conversations_analytics', transformed)

sync_task = PythonOperator(
    task_id='sync_conversations',
    python_callable=sync_conversations_to_analytics,
    dag=dag
)
```

### Event-Driven Sync (Triggers)

**Use Case**: Sync triggered by specific events (e.g., conversation completed → generate summary).

```python
class EventDrivenSyncPipeline:
    """Trigger-based data synchronization."""

    async def on_conversation_completed(self, event: dict):
        """
        When conversation is marked complete:
        1. Generate embeddings for all messages
        2. Extract entities and build knowledge graph
        3. Update analytics
        """

        tenant_id = event['tenant_id']
        conversation_id = event['conversation_id']

        # 1. Fetch all messages
        messages = await self.db.query("""
            SELECT id, content FROM messages
            WHERE conversation_id = :conversation_id
        """, {"conversation_id": conversation_id})

        # 2. Generate embeddings (batch)
        contents = [msg['content'] for msg in messages]
        embeddings = await self.embedding_service.embed_batch(contents)

        # 3. Store in Qdrant
        points = [
            PointStruct(
                id=msg['id'],
                vector=embedding,
                payload={"content": msg['content']}
            )
            for msg, embedding in zip(messages, embeddings)
        ]

        await self.qdrant.upsert(
            collection_name=f"embeddings_{tenant_id}",
            points=points
        )

        # 4. Extract entities
        entities = await self.entity_extractor.extract_batch(contents)

        # 5. Store in Neo4j
        for entity in entities:
            await self.neo4j.create_entity(tenant_id, **entity)
```

---

## Federated Queries

### StarRocks Querying PostgreSQL

**Use Case**: Join analytics data (StarRocks) with transactional data (PostgreSQL).

```sql
-- Create external catalog for PostgreSQL
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

-- Federated query: Join StarRocks analytics with PostgreSQL users
SELECT
    u.name,
    u.email,
    SUM(a.total_cost) AS total_spend,
    AVG(a.avg_latency) AS avg_latency
FROM llm_usage_analytics a
JOIN postgres_catalog.cortex.users u
    ON a.tenant_id = u.account_id
WHERE a.date >= '2024-01-01'
GROUP BY u.name, u.email
ORDER BY total_spend DESC
LIMIT 10;
```

### Unified Query API

**Purpose**: Provide single API that queries both Traditional and AI data platforms.

```python
class UnifiedDataPlatformAPI:
    """Unified query interface across all data stores."""

    async def get_conversation_with_context(
        self,
        conversation_id: str,
        tenant_id: str
    ) -> dict:
        """
        Fetch conversation from PostgreSQL + related context from AI platform.

        Data sources:
        - PostgreSQL: Conversation metadata, messages
        - Qdrant: Semantically similar conversations
        - Neo4j: Related entities
        - ClickHouse: Usage analytics
        """

        # 1. Fetch conversation (PostgreSQL)
        conversation = await self.postgres.query("""
            SELECT id, title, created_at
            FROM conversations
            WHERE id = :conversation_id AND tenant_id = :tenant_id
        """, {"conversation_id": conversation_id, "tenant_id": tenant_id})

        # 2. Fetch messages (PostgreSQL)
        messages = await self.postgres.query("""
            SELECT id, content, role, created_at
            FROM messages
            WHERE conversation_id = :conversation_id
            ORDER BY created_at
        """, {"conversation_id": conversation_id})

        # 3. Find similar conversations (Qdrant)
        last_message = messages[-1]['content']
        similar = await self.qdrant.search(
            tenant_id=tenant_id,
            query=last_message,
            top_k=3
        )

        # 4. Extract entities from messages (Neo4j)
        entities = []
        for message in messages:
            message_entities = await self.neo4j.find_entities_in_text(
                tenant_id=tenant_id,
                text=message['content']
            )
            entities.extend(message_entities)

        # 5. Fetch usage analytics (ClickHouse)
        analytics = await self.clickhouse.query("""
            SELECT
                COUNT(*) as message_count,
                SUM(total_tokens) as total_tokens,
                SUM(cost_usd) as total_cost
            FROM llm_usage_events
            WHERE conversation_id = :conversation_id
        """, {"conversation_id": conversation_id})

        # 6. Combine results
        return {
            "conversation": conversation,
            "messages": messages,
            "similar_conversations": similar,
            "entities": entities,
            "analytics": analytics
        }
```

---

## 5-Layer Data Architecture

The 5-layer architecture applies to **both Traditional and AI platforms**, showing how data flows from ingestion to consumption across both stacks.

```
┌─────────────────────────────────────────────────────────────────────┐
│                        CONSUMER LAYER                                │
│  Dashboards (Grafana) · BI Tools · APIs · LLM Context Retrieval     │
│  [Traditional: API responses, dashboards]                            │
│  [AI: RAG context, agent responses]                                  │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────────┐
│                         QUERY LAYER                                  │
│  Unified Query API · SQL (StarRocks) · Cypher (Neo4j) ·             │
│  Vector Similarity (Qdrant) · Logical Federation                    │
│  [Integration: Federated queries across both platforms]             │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────────┐
│                       PROCESSING LAYER                               │
│  [Traditional: Materialized Views (StarRocks), Aggregations]        │
│  [AI: Vector Indexing (Qdrant), Graph Algorithms (Neo4j)]          │
│  [Shared: Event Stream Processing (Kafka + Spark)]                  │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────────┐
│                        STORAGE LAYER                                 │
│  [Traditional: PostgreSQL, StarRocks, Redis]                        │
│  [AI: Qdrant, Neo4j, Semantic Cache]                                │
│  [Shared: S3 Archive]                                                │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────────┐
│                       INGESTION LAYER                                │
│  Kafka Streaming · Spark Streaming · Schema Validation ·            │
│  Data Contracts · Event Bus · Batch Import                          │
│  [Shared: Both platforms use same ingestion infrastructure]         │
└─────────────────────────────────────────────────────────────────────┘
```

### Layer 1: Ingestion (Shared)

**Both stacks** use the same ingestion infrastructure (Kafka, Spark Streaming).

**Kafka Topics**:
- `llm-requests`: LLM Gateway usage events → ClickHouse (Traditional), Semantic Cache (AI)
- `conversations`: Chat message events → PostgreSQL (Traditional), Qdrant/Neo4j (AI)
- `entity-extractions`: Knowledge Graph updates → Neo4j (AI)
- `audit-logs`: Security events → PostgreSQL (Traditional), ClickHouse (Traditional)

### Layer 2: Storage (Separated by Stack)

**Traditional Storage**:
- PostgreSQL: Transactional data (ACID)
- StarRocks: Analytics (OLAP)
- Redis: Session cache, rate limits

**AI Storage**:
- Qdrant: Embeddings, semantic search
- Neo4j: Knowledge graph
- Semantic Cache: LLM response caching

**Shared**:
- S3: Long-term archival for both stacks

### Layer 3: Processing (Stack-Specific + Shared)

**Traditional Processing**:
- StarRocks materialized views: Pre-aggregate analytics
- ClickHouse aggregations: Time-series analytics

**AI Processing**:
- Qdrant vector indexing: HNSW index for fast similarity search
- Neo4j graph algorithms: PageRank, community detection

**Shared Processing**:
- Kafka + Spark Streaming: Real-time event processing for both stacks

### Layer 4: Query (Unified)

**Unified Query Engine** supports:
- SQL queries → PostgreSQL, StarRocks
- Cypher queries → Neo4j
- Vector similarity → Qdrant
- **Federated queries** → Join across stacks (StarRocks + PostgreSQL)
- **Hybrid queries** → Graph RAG (Qdrant + Neo4j)

### Layer 5: Consumer (Stack-Agnostic)

**Consumers** can query either stack transparently:
- **Dashboards**: Pull from StarRocks (analytics) + PostgreSQL (transactional)
- **LLM Context**: Pull from Qdrant (semantic) + Neo4j (graph)
- **APIs**: Pull from PostgreSQL (data) + Redis (cache)

---

## Data Lifecycle Management

### Hot → Warm → Cold Transition

```
┌────────────────────────────────────────────────────────────┐
│ Hot Data (Active Use, 0-7 days)                            │
│ • PostgreSQL: Conversations, messages (ACID transactions)  │
│ • Redis: Sessions, rate limits (sub-ms access)             │
│ • Qdrant: Recent embeddings (fast semantic search)         │
│ • Neo4j: Active knowledge graph (graph traversal)          │
└────────────────────────────────┬───────────────────────────┘
                                 │ (7 days)
┌────────────────────────────────▼───────────────────────────┐
│ Warm Data (Recent History, 7-90 days)                      │
│ • StarRocks: Analytics, aggregations (OLAP queries)        │
│ • PostgreSQL: Archived conversations (occasional access)   │
│ • Qdrant: Older embeddings (slower search acceptable)      │
└────────────────────────────────┬───────────────────────────┘
                                 │ (90 days)
┌────────────────────────────────▼───────────────────────────┐
│ Cold Data (Archive, 90+ days)                              │
│ • S3 Glacier: Long-term archival (compliance, 7 years)     │
│ • Parquet files: Compressed analytics data                 │
│ • JSON exports: Knowledge graph snapshots                  │
└────────────────────────────────────────────────────────────┘
```

**Implementation**:

```python
class DataLifecycleManager:
    """Manage data lifecycle across hot/warm/cold tiers."""

    async def transition_to_warm(self, cutoff_days: int = 7):
        """Move data from hot to warm storage."""

        cutoff = datetime.utcnow() - timedelta(days=cutoff_days)

        # PostgreSQL → StarRocks
        old_conversations = await self.postgres.query("""
            SELECT * FROM conversations
            WHERE created_at < :cutoff
        """, {"cutoff": cutoff})

        await self.starrocks.batch_insert(
            'conversations_archive',
            old_conversations
        )

        # Qdrant: Mark old embeddings (no deletion, just metadata)
        await self.qdrant.set_payload(
            collection_name="embeddings",
            payload={"tier": "warm"},
            points_selector=Filter(
                must=[
                    FieldCondition(
                        key="timestamp",
                        range=Range(lt=cutoff.isoformat())
                    )
                ]
            )
        )

    async def transition_to_cold(self, cutoff_days: int = 90):
        """Archive data to S3."""

        cutoff = datetime.utcnow() - timedelta(days=cutoff_days)

        # Export to Parquet
        old_data = await self.starrocks.query("""
            SELECT * FROM conversations_archive
            WHERE created_at < :cutoff
        """, {"cutoff": cutoff})

        # Write to S3
        parquet_file = f"archive/conversations_{cutoff.strftime('%Y%m%d')}.parquet"
        await self.s3.upload_parquet(parquet_file, old_data)

        # Delete from StarRocks (data now in S3)
        await self.starrocks.execute("""
            DELETE FROM conversations_archive
            WHERE created_at < :cutoff
        """, {"cutoff": cutoff})
```

---

## Cross-Platform Analytics

### Example: LLM Cost Analysis with Context

**Query**: "Show LLM costs by tenant, including semantic cache savings and knowledge graph usage."

```python
async def get_llm_cost_analysis(tenant_id: str, date_range: str):
    """
    Cross-platform analytics query.

    Data sources:
    - ClickHouse: LLM usage events
    - PostgreSQL: Semantic cache savings
    - Neo4j: Knowledge graph query count
    """

    # 1. LLM usage costs (ClickHouse)
    llm_costs = await clickhouse.query("""
        SELECT
            tenant_id,
            SUM(cost_usd) AS total_cost,
            SUM(total_tokens) AS total_tokens,
            COUNT(*) AS request_count
        FROM llm_usage_events
        WHERE tenant_id = :tenant_id
          AND timestamp >= :date_range
        GROUP BY tenant_id
    """, {"tenant_id": tenant_id, "date_range": date_range})

    # 2. Cache savings (PostgreSQL)
    cache_savings = await postgres.query("""
        SELECT
            tenant_id,
            SUM(cost_saved) AS total_savings,
            COUNT(*) AS cache_hits
        FROM semantic_cache_savings
        WHERE tenant_id = :tenant_id
          AND timestamp >= :date_range
        GROUP BY tenant_id
    """, {"tenant_id": tenant_id, "date_range": date_range})

    # 3. Knowledge graph usage (Neo4j)
    kg_usage = await neo4j.query(tenant_id, """
        MATCH (n:Entity)
        WHERE n.created_at >= $date_range
        RETURN COUNT(n) AS entity_count
    """, {"date_range": date_range})

    # 4. Combine results
    return {
        "tenant_id": tenant_id,
        "llm_cost": llm_costs[0]['total_cost'],
        "cache_savings": cache_savings[0]['total_savings'],
        "net_cost": llm_costs[0]['total_cost'] - cache_savings[0]['total_savings'],
        "cache_hit_rate": cache_savings[0]['cache_hits'] / llm_costs[0]['request_count'],
        "knowledge_graph_entities": kg_usage[0]['entity_count']
    }
```

---

## Monitoring Cross-Platform Data Flows

```yaml
# Prometheus metrics for integration health
- kafka_consumer_lag_seconds{topic="message.created", consumer_group="embedding-pipeline"} < 60
- data_sync_delay_seconds{source="postgres", target="qdrant"} < 30
- federated_query_duration_seconds{source="starrocks", target="postgres"} < 2
- semantic_cache_sync_lag_seconds < 10
```

---

**Previous**: [Traditional Data Platform](./04a-traditional-data-platform.md) | [AI Data Platform](./04b-ai-data-platform.md) | [Back to Overview](../00-parallel-stacks-overview.md)
