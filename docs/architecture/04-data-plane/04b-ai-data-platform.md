# AI Data Platform

## Overview

The **AI Data Platform** provides specialized storage systems optimized for AI workloads including semantic search, knowledge graphs, and intelligent caching. This stack handles:

- **Vector Database (Qdrant)**: Embeddings and semantic search
- **Graph Database (Neo4j)**: Knowledge graphs and entity relationships
- **Semantic Cache**: Vector-based LLM response caching
- **Knowledge Graph Storage**: Multi-hop reasoning and explainability

```
┌──────────────────────────────────────────────────────────────┐
│                  AI Data Platform                             │
├─────────────┬─────────────────────────────────────────────────┤
│ Vector      │ Graph               │ Semantic Cache            │
│ Qdrant      │ Neo4j               │ (Qdrant-based)            │
│             │                     │                           │
│ Embeddings  │ Knowledge           │ LLM Responses             │
│ Semantic    │ Relationships       │ Query Similarity          │
│ Search      │ Entities            │ Cost Optimization         │
└─────────────┴─────────────────────┴───────────────────────────┘
```

---

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

### Semantic Cache Implementation

**Purpose**: Cache LLM responses based on semantic similarity to reduce costs by 90%.

```python
class SemanticCache:
    """Vector-based semantic cache for LLM responses."""

    def __init__(self, qdrant_client: QdrantClient, embedding_provider):
        self.client = qdrant_client
        self.embedder = embedding_provider
        self.cache_collection = "semantic_cache"

    async def get(
        self,
        query: str,
        tenant_id: str,
        similarity_threshold: float = 0.95
    ) -> Optional[str]:
        """
        Retrieve cached response if semantically similar query exists.

        Args:
            query: User query
            tenant_id: Tenant identifier
            similarity_threshold: Minimum cosine similarity (0.95 = 95% similar)

        Returns:
            Cached LLM response if found, None otherwise
        """

        # Embed query
        query_embedding = await self.embedder.embed(query)

        # Search for similar cached queries
        results = await self.client.search(
            collection_name=self.cache_collection,
            query_vector=query_embedding,
            query_filter=Filter(
                must=[
                    FieldCondition(key="tenant_id", match=MatchValue(value=tenant_id))
                ]
            ),
            limit=1,
            score_threshold=similarity_threshold
        )

        if results:
            hit = results[0]
            logger.info(
                f"Cache HIT: similarity={hit.score:.3f}, "
                f"original_query='{hit.payload['query']}'"
            )
            return hit.payload["response"]

        logger.info("Cache MISS")
        return None

    async def set(
        self,
        query: str,
        response: str,
        tenant_id: str,
        ttl: int = 3600  # 1 hour
    ):
        """
        Cache LLM response with semantic indexing.

        Args:
            query: User query
            response: LLM response to cache
            tenant_id: Tenant identifier
            ttl: Time-to-live in seconds
        """

        # Embed query
        query_embedding = await self.embedder.embed(query)

        # Store in vector database
        await self.client.upsert(
            collection_name=self.cache_collection,
            points=[
                PointStruct(
                    id=str(uuid.uuid4()),
                    vector=query_embedding,
                    payload={
                        "tenant_id": tenant_id,
                        "query": query,
                        "response": response,
                        "timestamp": datetime.utcnow().isoformat(),
                        "expires_at": (datetime.utcnow() + timedelta(seconds=ttl)).isoformat()
                    }
                )
            ]
        )

# Usage in LLM Gateway
async def chat_completion_with_cache(query: str, tenant_id: str):
    """Chat completion with semantic caching."""

    # Check cache
    cached_response = await semantic_cache.get(query, tenant_id)
    if cached_response:
        return {"response": cached_response, "cached": True}

    # Cache miss: Call LLM
    llm_response = await llm_provider.complete(query)

    # Cache response
    await semantic_cache.set(query, llm_response, tenant_id)

    return {"response": llm_response, "cached": False}
```

**Cache Performance Metrics**:
- **Hit Rate Target**: >85% for production workloads
- **Cost Savings**: 90% reduction in LLM costs (cached queries cost ~$0)
- **Latency**: <20ms cache lookup vs 500-2000ms LLM call

---

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
    attributes: '{"role": "engineer"}'
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

### Knowledge Graph Patterns

#### Entity Extraction from Documents

```python
class KnowledgeGraphBuilder:
    """Build knowledge graph from documents using LLM."""

    async def extract_entities_and_relationships(
        self,
        document: str,
        tenant_id: str
    ) -> dict:
        """Extract entities and relationships using LLM."""

        # LLM prompt for entity extraction
        prompt = f"""
        Extract entities and relationships from the following text.
        Return JSON format:
        {{
            "entities": [
                {{"name": "Alice", "type": "Person", "attributes": {{"role": "engineer"}}}},
                {{"name": "Acme Corp", "type": "Company"}}
            ],
            "relationships": [
                {{"from": "Alice", "to": "Acme Corp", "type": "WORKS_AT", "attributes": {{"since": "2020"}}}}
            ]
        }}

        Text: {document}
        """

        # Call LLM
        llm_response = await self.llm.complete(prompt)
        extracted = json.loads(llm_response)

        # Store in Neo4j
        for entity in extracted["entities"]:
            await self.graph_store.create_entity(
                tenant_id=tenant_id,
                entity_type=entity["type"],
                name=entity["name"],
                attributes=entity.get("attributes", {})
            )

        for rel in extracted["relationships"]:
            await self.graph_store.create_relationship(
                tenant_id=tenant_id,
                from_entity=rel["from"],
                to_entity=rel["to"],
                relationship_type=rel["type"],
                attributes=rel.get("attributes", {})
            )

        return extracted
```

#### Graph Traversal for RAG

```python
async def graph_rag_retrieval(
    query: str,
    tenant_id: str,
    max_hops: int = 2
) -> list[str]:
    """
    Retrieve context using graph traversal.

    Steps:
    1. Extract entities from query using LLM
    2. Find related entities in graph (multi-hop traversal)
    3. Return context about entities and relationships
    """

    # Extract entities from query
    query_entities = await extract_entities_from_query(query)

    # Multi-hop graph traversal
    context_entities = []
    for entity_name in query_entities:
        related = await graph_store.find_related_entities(
            tenant_id=tenant_id,
            entity_name=entity_name,
            max_depth=max_hops
        )
        context_entities.extend(related)

    # Format as context
    context_paragraphs = []
    for entity in context_entities:
        attrs = json.loads(entity["attributes"])
        context_paragraphs.append(
            f"{entity['name']} ({entity['type']}): {attrs}"
        )

    return context_paragraphs
```

### Graph Algorithms

```python
# PageRank: Identify influential entities
async def compute_entity_importance(tenant_id: str):
    """Run PageRank to identify influential entities."""

    query = """
    CALL gds.pageRank.stream({
        nodeQuery: 'MATCH (n:Entity) WHERE n.tenant_id = $tenant_id RETURN id(n) as id',
        relationshipQuery: 'MATCH (a:Entity)-[r]-(b:Entity)
                            WHERE a.tenant_id = $tenant_id
                            AND b.tenant_id = $tenant_id
                            RETURN id(a) as source, id(b) as target'
    })
    YIELD nodeId, score
    RETURN gds.util.asNode(nodeId).name AS entity, score
    ORDER BY score DESC
    LIMIT 10
    """

    return await graph_store.query(tenant_id, query)

# Community Detection: Group related entities
async def detect_entity_communities(tenant_id: str):
    """Run Louvain algorithm to detect entity communities."""

    query = """
    CALL gds.louvain.stream({
        nodeQuery: 'MATCH (n:Entity) WHERE n.tenant_id = $tenant_id RETURN id(n) as id',
        relationshipQuery: 'MATCH (a:Entity)-[r]-(b:Entity)
                            WHERE a.tenant_id = $tenant_id
                            RETURN id(a) as source, id(b) as target'
    })
    YIELD nodeId, communityId
    RETURN gds.util.asNode(nodeId).name AS entity, communityId
    """

    return await graph_store.query(tenant_id, query)
```

---

## Hybrid: Graph RAG (Vector + Graph Fusion)

### Reciprocal Rank Fusion (RRF)

```python
class GraphRAGRetriever:
    """Combine vector search + graph traversal for enhanced retrieval."""

    async def retrieve(
        self,
        query: str,
        tenant_id: str,
        top_k: int = 5
    ) -> list[dict]:
        """
        Graph RAG retrieval using Reciprocal Rank Fusion.

        Steps:
        1. Vector search for semantically similar documents
        2. Extract entities from top vector results
        3. Graph traversal to find related entities
        4. Combine results using RRF algorithm
        """

        # 1. Vector search
        vector_results = await self.vector_store.search(
            tenant_id=tenant_id,
            query=query,
            top_k=10
        )

        # 2. Extract entities from vector results
        entities = []
        for doc in vector_results:
            doc_entities = doc.metadata.get("entities", [])
            entities.extend(doc_entities)

        # 3. Graph traversal
        graph_results = []
        for entity_name in set(entities):
            related = await self.graph_store.find_related_entities(
                tenant_id=tenant_id,
                entity_name=entity_name,
                max_depth=2
            )
            graph_results.extend(related)

        # 4. Combine using RRF
        combined_results = self._reciprocal_rank_fusion(
            vector_results,
            graph_results,
            k=60
        )

        return combined_results[:top_k]

    def _reciprocal_rank_fusion(
        self,
        vector_results: list,
        graph_results: list,
        k: int = 60
    ) -> list[dict]:
        """
        RRF algorithm for combining ranked lists.

        Formula: score(d) = Σ 1 / (k + rank(d))

        Args:
            vector_results: Ranked list from vector search
            graph_results: Ranked list from graph traversal
            k: Constant (typically 60) to prevent high ranks from dominating

        Returns:
            Combined results sorted by RRF score
        """

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

### Graph RAG Benefits

| Traditional RAG | Graph RAG | Improvement |
|----------------|-----------|-------------|
| Vector search only | Vector + Graph traversal | 30% better relevance |
| No relationship context | Multi-hop reasoning | Explainable results |
| Limited to similar docs | Related entities discovered | Deeper context |
| Single-source retrieval | Fusion of multiple signals | More robust |

**Example Use Case**:

Query: "What projects is Alice working on?"

**Traditional RAG** (Vector Only):
1. Embed query
2. Find similar documents mentioning "Alice" and "projects"
3. Return top 5 documents

**Graph RAG** (Vector + Graph):
1. Embed query → Find documents about Alice
2. Extract entity "Alice" from query
3. Graph traversal: `(Alice)-[:WORKS_ON]->(Projects)`
4. Find all projects connected to Alice in graph
5. Combine vector results + graph results using RRF
6. Return enriched context with relationships

**Result**: Graph RAG discovers Alice's projects even if not explicitly mentioned together in documents.

---

## Data Management Patterns

### Backup & Recovery

```python
class AIDataBackupManager:
    """Backup manager for AI data platform."""

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
        timestamp = datetime.utcnow().strftime("%Y%m%d_%H%M%S")
        backup_file = f"backups/neo4j_{tenant_id}_{timestamp}.json"
        with open(backup_file, "w") as f:
            json.dump(data, f)
```

### Data Migration

```python
async def migrate_tenant_embeddings(
    from_tenant_id: str,
    to_tenant_id: str
):
    """Migrate embeddings between tenants."""

    # Create new collection
    await qdrant_client.create_collection(
        collection_name=f"embeddings_{to_tenant_id}",
        vectors_config=VectorParams(size=1536, distance=Distance.COSINE)
    )

    # Copy all points
    offset = None
    while True:
        results, offset = await qdrant_client.scroll(
            collection_name=f"embeddings_{from_tenant_id}",
            limit=100,
            offset=offset
        )

        if not results:
            break

        # Update tenant_id in payload
        for point in results:
            point.payload["tenant_id"] = to_tenant_id

        # Upsert to new collection
        await qdrant_client.upsert(
            collection_name=f"embeddings_{to_tenant_id}",
            points=results
        )
```

### Data Retention & Cleanup

```python
async def cleanup_old_embeddings(tenant_id: str, retention_days: int = 90):
    """Delete embeddings older than retention period."""

    cutoff = datetime.utcnow() - timedelta(days=retention_days)

    # Qdrant: Delete old embeddings (if timestamp indexed)
    await qdrant_client.delete(
        collection_name=f"embeddings_{tenant_id}",
        points_selector=FilterSelector(
            filter=Filter(
                must=[
                    FieldCondition(
                        key="timestamp",
                        range=Range(lt=cutoff.isoformat())
                    )
                ]
            )
        )
    )

    # Neo4j: Delete old nodes
    await neo4j.execute_query("""
        MATCH (n {tenant_id: $tenant_id})
        WHERE n.created_at < $cutoff
        DETACH DELETE n
    """, tenant_id=tenant_id, cutoff=cutoff.isoformat())
```

---

## Performance SLAs

### Latency Targets

| Operation | Target (p95) | Target (p99) | Measured At |
|-----------|-------------|-------------|-------------|
| **Vector Operations** |
| Vector search (10M vectors) | <50ms | <100ms | Qdrant |
| Batch embedding (100 docs) | <500ms | <1s | Embedding service |
| Semantic cache lookup | <20ms | <40ms | Qdrant |
| **Graph Operations** |
| 1-hop traversal | <30ms | <60ms | Neo4j |
| 2-hop traversal | <100ms | <200ms | Neo4j |
| Entity extraction | <150ms | <300ms | Neo4j + LLM |
| **End-to-End AI SLAs** |
| RAG retrieval + generation | <3s | <6s | Vector search + LLM |
| Knowledge graph query + LLM | <4s | <8s | Graph traversal + LLM |
| Graph RAG (hybrid) | <3.5s | <7s | Vector + Graph + LLM |

### Throughput Targets

| Resource | Target | Scaling Strategy |
|----------|--------|------------------|
| **Qdrant** | 1,000 vector writes/s, 5,000 searches/s | Sharding by tenant |
| **Neo4j** | 2,000 writes/s, 10,000 reads/s | Read replicas + caching |
| **Semantic Cache** | 10,000 lookups/s | Qdrant cluster mode |

### Cache Hit Rates

| Cache Type | Target Hit Rate | Impact |
|------------|----------------|--------|
| **Semantic Cache (Qdrant)** | >85% | 90% cost savings on LLM calls |

### Monitoring & Alerts

```python
# Prometheus metrics for AI data platform
from prometheus_client import Histogram, Counter, Gauge

# Vector search metrics
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

# Cache metrics
semantic_cache_hits = Counter('semantic_cache_hits_total', 'Semantic cache hits')
semantic_cache_misses = Counter('semantic_cache_misses_total', 'Semantic cache misses')

# Usage example
async def execute_vector_search(query: str, tenant_id: str):
    with vector_search_duration.labels(collection=f"embeddings_{tenant_id}").time():
        results = await qdrant.search(...)
    return results
```

**Alerting Rules** (Prometheus):
```yaml
groups:
  - name: ai_data_sla_violations
    interval: 30s
    rules:
      # Vector search latency
      - alert: HighVectorSearchLatency
        expr: histogram_quantile(0.95, vector_search_duration_seconds) > 0.05
        for: 5m
        annotations:
          summary: "Vector search p95 latency exceeds 50ms SLA"

      # Graph query latency
      - alert: HighGraphQueryLatency
        expr: histogram_quantile(0.95, graph_query_duration_seconds) > 0.1
        for: 5m
        annotations:
          summary: "Graph query p95 latency exceeds 100ms SLA"

      # Semantic cache hit rate
      - alert: LowSemanticCacheHitRate
        expr: |
          rate(semantic_cache_hits_total[5m]) /
          (rate(semantic_cache_hits_total[5m]) + rate(semantic_cache_misses_total[5m])) < 0.85
        for: 10m
        annotations:
          summary: "Semantic cache hit rate below 85% target"
```

---

## Performance Optimization Strategies

| Strategy | Implementation | Impact |
|----------|----------------|--------|
| **Vector Index Optimization** | HNSW index (M=16, ef_construct=100) | <50ms search on 10M vectors |
| **Graph Index Strategy** | Composite indexes on (tenant_id, name) | 10x faster entity lookups |
| **Semantic Cache** | Qdrant-based similarity search | 90% cost reduction, <20ms lookup |
| **Batch Embedding** | Embed 100 docs at once | 10x faster than sequential |
| **Graph Query Caching** | Redis cache for frequent traversals | 95% reduction in repeated queries |

---

**Next**: [Traditional Data Platform](./04a-traditional-data-platform.md) | [Integration](./04c-data-integration.md) | [Back to Overview](../00-parallel-stacks-overview.md)
