# Advanced AI Patterns

## Overview

This document covers advanced AI platform patterns that extend the core architecture with sophisticated retrieval, coordination, evaluation, and observability capabilities.

**Topics Covered**:
- **Graph RAG Integration**: Hybrid retrieval combining vector similarity and knowledge graph traversal
- **Multi-Agent Coordination**: Swarm, hierarchical, and workflow-based agent patterns
- **Evaluation Framework**: LLM output quality metrics, prompt optimization, and regression testing
- **Advanced Observability**: Distributed tracing, LLM-specific metrics, and cost tracking

**Related Documents**:
- [03-service-layer.md](./03-service-layer.md) - Core AI services (Agent Orchestration, RAG, Knowledge Graph)
- [05-ai-platform-services.md](./05-ai-platform-services.md) - LLM Gateway, Unified Chat, Memory Management
- [04-data-plane.md](./04-data-plane.md) - Vector Store, Graph Database, OLAP analytics

---

## Graph RAG Integration

### Motivation

Traditional RAG (Retrieval-Augmented Generation) uses **vector similarity search** to find relevant documents. While effective for semantic matching, pure vector search has limitations:

- **Lacks structural understanding**: Cannot capture entity relationships or knowledge hierarchies
- **No multi-hop reasoning**: Cannot traverse conceptual connections (e.g., "What technologies does Company X use?" requires entity linking)
- **Limited explainability**: Returns similar chunks without showing *why* they're relevant

**Graph RAG** addresses these gaps by combining:
1. **Vector search** (semantic similarity)
2. **Knowledge graph traversal** (entity relationships, multi-hop reasoning)
3. **Hybrid fusion** (intelligently combining both approaches)

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     GraphRAG Retriever                       │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  Query: "What is GraphRAG?"                                  │
│         ↓                                                     │
│  ┌──────────────────────┐         ┌──────────────────────┐  │
│  │  Vector Search       │         │  Graph Search         │  │
│  │  (Qdrant)            │         │  (Neo4j)              │  │
│  │                      │         │                       │  │
│  │  1. Embed query      │         │  1. Extract concepts  │  │
│  │  2. Similarity       │         │     ("GraphRAG")      │  │
│  │     search           │         │  2. Multi-hop         │  │
│  │  3. Top-K results    │         │     traversal (2-3)   │  │
│  │     (semantic)       │         │  3. Find related      │  │
│  │                      │         │     documents         │  │
│  └──────┬───────────────┘         └──────────┬────────────┘  │
│         │                                    │                │
│         │         ┌─────────────────────────┐│                │
│         └────────→│ Reciprocal Rank Fusion  │←───────────────┘
│                   │ (RRF)                    │
│                   │                          │
│                   │ • Combine rankings       │
│                   │ • Weight vector vs graph │
│                   │ • Deduplicate results    │
│                   └──────────┬───────────────┘
│                              ↓
│                   ┌──────────────────────────┐
│                   │  Unified Result Set      │
│                   │  (with explainability)   │
│                   └──────────────────────────┘
└─────────────────────────────────────────────────────────────┘
```

### Implementation Patterns

#### 1. Hybrid Retrieval

**Code Example** (from `cortex/rag/retriever.py`):

```python
class Retriever:
    def __init__(
        self,
        embeddings: EmbeddingService,
        vector_store: VectorStore,
        graph_store: GraphStore | None = None,
    ):
        self.embeddings = embeddings
        self.vector_store = vector_store
        self.graph_store = graph_store

    async def graphrag_search(
        self,
        query: str,
        top_k: int = 5,
        vector_weight: float = 0.7,
        graph_weight: float = 0.3,
        max_hops: int = 2,
        tenant_id: str | None = None,
    ) -> list[SearchResult]:
        """
        Hybrid search combining vector and graph retrieval.

        Uses Reciprocal Rank Fusion (RRF) to combine results.
        """
        # 1. Vector search (semantic similarity)
        vector_results = await self.search(
            query=query,
            top_k=top_k * 2,  # Get more candidates for fusion
            tenant_id=tenant_id,
        )

        # 2. Graph search (entity relationships)
        concept_names = self._extract_concepts_from_query(query)
        graph_results = []

        for concept_name in concept_names[:3]:
            concept_results = await self.graph_search(
                concept_name=concept_name,
                max_hops=max_hops,
                tenant_id=tenant_id,
            )
            graph_results.extend(concept_results)

        # 3. Fuse results using RRF
        fused_results = self._reciprocal_rank_fusion(
            vector_results=vector_results,
            graph_results=graph_results,
            vector_weight=vector_weight,
            graph_weight=graph_weight,
        )

        return fused_results[:top_k]
```

**Key Decisions**:
- **Vector-first**: Start with vector search (70% weight by default) for semantic relevance
- **Graph augmentation**: Use graph traversal (30% weight) to add related entities and multi-hop reasoning
- **RRF fusion**: Combine rankings without needing normalized scores (robust to score scale differences)

#### 2. Multi-Hop Graph Traversal

**Cypher Query** (from `cortex/rag/retriever.py`):

```python
async def graph_search(
    self,
    concept_name: str,
    max_hops: int = 2,
    tenant_id: str | None = None,
) -> list[SearchResult]:
    """Find documents via multi-hop graph traversal."""

    tenant_filter = "AND c.tenant_id = $tenant_id" if tenant_id else ""

    query = f"""
    MATCH (c:Concept {{name: $concept_name}})
    WHERE 1=1 {tenant_filter}
    WITH c
    MATCH (c)-[:RELATES_TO*0..{max_hops}]-(related:Concept)
    WITH DISTINCT related
    MATCH (d:Document)-[m:MENTIONS]->(related)
    RETURN DISTINCT d.id as doc_id,
           d.content as content,
           COUNT(DISTINCT related) as concept_count,
           SUM(m.confidence) as total_confidence
    ORDER BY concept_count DESC, total_confidence DESC
    """

    # Execute and convert to SearchResult objects
    # ...
```

**Traversal Strategy**:
- **0-hop**: Document directly mentions the concept
- **1-hop**: Document mentions concepts *related to* the query concept
- **2-hop**: Two degrees of separation (e.g., GraphRAG → Vector Databases → Qdrant)
- **3-hop**: Rarely needed; increases noise and latency

**Scoring**:
```python
# Combine concept count + confidence
score = min(1.0, (concept_count * 0.3 + total_confidence * 0.1))
```

#### 3. Reciprocal Rank Fusion (RRF)

**Algorithm** (from `cortex/rag/retriever.py`):

```python
def _reciprocal_rank_fusion(
    self,
    vector_results: list[SearchResult],
    graph_results: list[SearchResult],
    vector_weight: float = 0.7,
    graph_weight: float = 0.3,
    k: int = 60,  # RRF constant from literature
) -> list[SearchResult]:
    """
    Combine results using Reciprocal Rank Fusion.

    RRF Score: score(d) = Σ w_i / (k + rank_i(d))
    where w_i is weight for source i, rank_i is rank in that source.
    """
    # Build rank maps
    vector_ranks = {r.id: rank for rank, r in enumerate(vector_results, 1)}
    graph_ranks = {r.id: rank for rank, r in enumerate(graph_results, 1)}

    # Collect all unique document IDs
    all_doc_ids = set(vector_ranks.keys()) | set(graph_ranks.keys())

    # Calculate RRF scores
    doc_scores: dict[str, float] = {}

    for doc_id in all_doc_ids:
        rrf_score = 0.0

        # Vector contribution
        if doc_id in vector_ranks:
            rrf_score += vector_weight / (k + vector_ranks[doc_id])

        # Graph contribution
        if doc_id in graph_ranks:
            rrf_score += graph_weight / (k + graph_ranks[doc_id])

        doc_scores[doc_id] = rrf_score

    # Sort by RRF score
    sorted_docs = sorted(doc_scores.items(), key=lambda x: x[1], reverse=True)

    # Build final results with metadata
    fused_results = []
    for doc_id, rrf_score in sorted_docs:
        # Add metadata indicating source(s)
        metadata = {
            "rrf_score": rrf_score,
            "in_vector": doc_id in vector_ranks,
            "in_graph": doc_id in graph_ranks,
            "vector_rank": vector_ranks.get(doc_id),
            "graph_rank": graph_ranks.get(doc_id),
        }
        fused_results.append(SearchResult(..., metadata=metadata))

    return fused_results
```

**Why RRF?**:
- **Score-agnostic**: Works with different score ranges (vector: 0-1, graph: arbitrary)
- **Simple**: No normalization or calibration needed
- **Robust**: Rank-based fusion is less sensitive to outliers
- **Research-backed**: k=60 is empirically validated in IR literature

#### 4. Entity-Aware Context Building

**Workflow**:

```python
# 1. Extract entities from query
query = "What databases does Company X use for AI?"
entities = extract_entities(query)  # ["Company X", "databases", "AI"]

# 2. Expand via graph
expanded_concepts = []
for entity in entities:
    # Find related concepts in graph
    related = await graph_store.get_related_concepts(entity, max_hops=1)
    expanded_concepts.extend(related)

# 3. Retrieve documents mentioning expanded concepts
docs = await retriever.graphrag_search(
    query=query,
    vector_weight=0.6,  # Slightly lower - trust graph more for entity queries
    graph_weight=0.4,
)

# 4. Build context with entity highlights
context = format_context_with_entities(docs, entities)
```

### Explainability

**Graph Path Visualization**:

```python
# Return graph path with results
metadata = {
    "graph_path": [
        {"concept": "Company X", "type": "organization"},
        {"edge": "USES", "confidence": 0.95},
        {"concept": "PostgreSQL", "type": "technology"},
        {"edge": "RELATES_TO", "confidence": 0.85},
        {"concept": "Relational Databases", "type": "category"},
    ],
    "path_length": 2,
    "total_confidence": 0.81,
}
```

**UI Display**:
```
Query: "What databases does Company X use?"

Results:
1. [Score: 0.92] Company X Architecture Overview (vector + graph)
   └─ Graph path: Company X → USES → PostgreSQL (confidence: 0.95)

2. [Score: 0.87] PostgreSQL Performance Tuning (graph only)
   └─ Graph path: PostgreSQL ← USES ← Company X → USES → Redis
```

### Performance Optimization

**Strategies**:

1. **Vector-first filtering**:
   ```python
   # Get top-100 from vector search
   candidates = await vector_store.search(query, top_k=100)

   # Use graph to rerank only top candidates (not entire corpus)
   reranked = await graph_rerank(candidates, max_hops=1)
   ```

2. **Graph caching**:
   ```python
   # Cache frequent concept expansions in Redis
   cache_key = f"graph:expand:{concept_name}:{max_hops}"
   cached = await redis.get(cache_key)
   if cached:
       return json.loads(cached)
   ```

3. **Adaptive hop count**:
   ```python
   # Use fewer hops for broad queries, more for specific entity queries
   if query_has_named_entities(query):
       max_hops = 2  # Specific entity query
   else:
       max_hops = 1  # Broad conceptual query
   ```

---

## Multi-Agent Coordination Patterns

### Overview

Multi-agent systems enable complex workflows by decomposing tasks across specialized agents. The unified platform supports three coordination patterns:

1. **Swarm** (peer-to-peer handoff)
2. **Hierarchical** (supervisor delegates to workers)
3. **Workflow-based** (sequential pipeline with state transitions)

### Pattern 1: Swarm Coordination

**Architecture**:

```
┌────────────────────────────────────────────────────────────┐
│                    Swarm Orchestrator                       │
│                                                              │
│  User Query → [General Agent]                               │
│                      ↓                                       │
│               Analyzes request                              │
│                      ↓                                       │
│            ┌─────────┴─────────┐                            │
│            ↓                   ↓                             │
│     [Researcher Agent]   [Writer Agent]                     │
│      • Web search          • Draft content                  │
│      • Document retrieval  • Formatting                     │
│            ↓                   ↓                             │
│            └─────────┬─────────┘                            │
│                      ↓                                       │
│              [General Agent]                                │
│              Synthesizes final answer                       │
└────────────────────────────────────────────────────────────┘
```

**Implementation** (from `cortex/orchestration/swarm.py`):

```python
from cortex.orchestration import Swarm, ModelConfig

# Initialize swarm
swarm = Swarm(model="gpt-4o")

# Add agents with handoff capabilities
swarm.add_agent(
    name="general",
    description="General assistant for routing and synthesis",
    system_prompt="You route queries to specialists and synthesize results.",
    can_handoff_to=["researcher", "writer"],
)

swarm.add_agent(
    name="researcher",
    description="Research and gather information",
    tools=["web_search", "document_retrieval"],
    system_prompt="You research topics and gather relevant information.",
    can_handoff_to=["writer", "general"],
)

swarm.add_agent(
    name="writer",
    description="Write and format content",
    tools=["markdown_formatter", "grammar_checker"],
    system_prompt="You write clear, well-formatted content.",
    can_handoff_to=["general"],
)

# Compile with checkpointer
from cortex.orchestration.session import get_checkpointer

graph = swarm.compile(checkpointer=get_checkpointer())

# Run
result = await graph.ainvoke(
    {"messages": [HumanMessage(content="Write a blog post about GraphRAG")]},
    config={"configurable": {"thread_id": "session-1"}},
)
```

**Handoff Mechanism**:

```python
# Automatically injected by swarm
handoff_to_researcher = create_handoff_tool(
    agent_name="researcher",
    description="Transfer to researcher: Research and gather information",
)

# Agent calls it to transfer control
await handoff_to_researcher.ainvoke({"reason": "Need to research GraphRAG"})
```

**When to Use**:
- ✅ Flexible task decomposition (agents decide who to hand off to)
- ✅ Peer collaboration (multiple agents contribute)
- ✅ Uncertain workflows (cannot predict exact sequence)
- ❌ Strict ordering required (use workflow pattern instead)

### Pattern 2: Hierarchical Coordination

**Architecture**:

```
┌──────────────────────────────────────────────────────────┐
│                   Supervisor Agent                        │
│  • Decomposes task                                        │
│  • Delegates to workers                                   │
│  • Aggregates results                                     │
└────┬─────────────────┬────────────────────┬──────────────┘
     │                 │                    │
     ↓                 ↓                    ↓
┌─────────┐      ┌─────────┐         ┌─────────┐
│ Worker 1│      │ Worker 2│         │ Worker 3│
│ (SQL)   │      │ (Python)│         │ (Cloud) │
└─────────┘      └─────────┘         └─────────┘
```

**Implementation**:

```python
# Supervisor agent
supervisor = Agent(
    name="supervisor",
    model=ModelConfig(model="gpt-4o"),
    system_prompt="""
    You are a supervisor that decomposes tasks and delegates to workers.

    Available workers:
    - sql_expert: SQL queries and database operations
    - python_expert: Python code generation and execution
    - cloud_expert: Cloud infrastructure and deployment

    For each task:
    1. Analyze requirements
    2. Delegate to appropriate worker(s)
    3. Aggregate results
    4. Provide final answer
    """,
    tools=[delegate_to_sql, delegate_to_python, delegate_to_cloud],
)

# Worker agents (run in parallel when possible)
workers = {
    "sql_expert": Agent(...),
    "python_expert": Agent(...),
    "cloud_expert": Agent(...),
}

# Execution
async def run_hierarchical(task: str):
    # Supervisor plans
    plan = await supervisor.run(f"Plan: {task}")

    # Execute worker tasks (parallelized)
    worker_tasks = extract_delegations(plan)
    results = await asyncio.gather(*[
        workers[w].run(task_desc)
        for w, task_desc in worker_tasks
    ])

    # Supervisor aggregates
    final = await supervisor.run(f"Aggregate: {results}")
    return final
```

**When to Use**:
- ✅ Clear task decomposition (supervisor knows what to delegate)
- ✅ Parallel execution (workers are independent)
- ✅ Aggregation needed (combine multiple outputs)
- ❌ Complex state sharing (use shared memory pool instead)

### Pattern 3: Workflow-Based Coordination

**Architecture** (LangGraph StateGraph):

```
┌──────────────────────────────────────────────────────────┐
│                  Workflow State Machine                   │
│                                                            │
│   [START] → [Classify] → [Route]                          │
│                            ↓                               │
│              ┌─────────────┼─────────────┐                │
│              ↓             ↓             ↓                 │
│           [FAQ]       [Technical]    [Sales]              │
│              ↓             ↓             ↓                 │
│              └─────────────┼─────────────┘                │
│                            ↓                               │
│                      [Format] → [END]                      │
└──────────────────────────────────────────────────────────┘
```

**Implementation**:

```python
from langgraph.graph import StateGraph, END

# Define state
class SupportState(TypedDict):
    query: str
    category: str
    response: str
    formatted: str

# Build workflow
workflow = StateGraph(SupportState)

# Add nodes
workflow.add_node("classify", classify_query)
workflow.add_node("faq", handle_faq)
workflow.add_node("technical", handle_technical)
workflow.add_node("sales", handle_sales)
workflow.add_node("format", format_response)

# Add edges
workflow.add_edge("classify", "route")

# Conditional routing
def route(state: SupportState) -> str:
    category = state["category"]
    if category == "faq":
        return "faq"
    elif category == "technical":
        return "technical"
    else:
        return "sales"

workflow.add_conditional_edges("route", route)
workflow.add_edge("faq", "format")
workflow.add_edge("technical", "format")
workflow.add_edge("sales", "format")
workflow.add_edge("format", END)

# Compile
graph = workflow.compile()
```

**When to Use**:
- ✅ Deterministic flow (known state transitions)
- ✅ Complex branching logic
- ✅ State persistence across steps
- ❌ Unpredictable task decomposition (use swarm instead)

### Shared Memory Pool (Cross-Agent Memory)

**Problem**: Agents in a swarm need to share context without re-explaining previous work.

**Solution** (from `cortex/orchestration/swarm.py`):

```python
class SharedMemoryPool:
    """
    Shared memory accessible by all agents in a swarm.

    Stored in Redis with tenant isolation.
    """
    def __init__(self, redis_client: Redis, tenant_id: str, session_id: str):
        self.redis = redis_client
        self.tenant_id = tenant_id
        self.session_id = session_id
        self.key_prefix = f"swarm:{tenant_id}:{session_id}"

    async def set(self, key: str, value: Any, ttl: int = 3600):
        """Store value in shared memory."""
        redis_key = f"{self.key_prefix}:{key}"
        await self.redis.setex(
            redis_key,
            ttl,
            json.dumps(value),
        )

    async def get(self, key: str) -> Any | None:
        """Retrieve value from shared memory."""
        redis_key = f"{self.key_prefix}:{key}"
        data = await self.redis.get(redis_key)
        return json.loads(data) if data else None

    async def append(self, key: str, value: Any):
        """Append to list in shared memory."""
        redis_key = f"{self.key_prefix}:{key}"
        await self.redis.rpush(redis_key, json.dumps(value))

# Usage in swarm
memory_pool = SharedMemoryPool(redis, tenant_id="acme", session_id="session-1")

# Researcher stores findings
await memory_pool.set("research_findings", {
    "topic": "GraphRAG",
    "sources": [...],
    "key_points": [...],
})

# Writer retrieves findings
findings = await memory_pool.get("research_findings")
```

---

## Evaluation Framework

### Overview

LLM outputs are non-deterministic and require systematic evaluation. The evaluation framework provides:

1. **Output quality metrics** (accuracy, relevance, coherence)
2. **Prompt optimization** (A/B testing, regression detection)
3. **Cost tracking** (token usage, latency, throughput)

### Evaluation Metrics

#### 1. LLM-as-Judge

**Pattern**: Use a strong LLM (e.g., GPT-4o) to evaluate other LLM outputs.

```python
from cortex.evaluation import LLMJudge

judge = LLMJudge(model="gpt-4o")

# Evaluate response quality
result = await judge.evaluate(
    prompt="What is GraphRAG?",
    response="GraphRAG combines vector search with knowledge graphs...",
    criteria={
        "accuracy": "Is the response factually correct?",
        "relevance": "Does it directly answer the question?",
        "coherence": "Is it well-structured and clear?",
    },
)

# Result:
# {
#   "accuracy": {"score": 0.95, "reasoning": "Response is accurate..."},
#   "relevance": {"score": 1.0, "reasoning": "Directly answers..."},
#   "coherence": {"score": 0.9, "reasoning": "Clear structure..."},
#   "overall": 0.95
# }
```

#### 2. Reference-Based Metrics

**Pattern**: Compare output to golden reference answers.

```python
from cortex.evaluation import calculate_similarity

# Exact match
exact_match = (response.strip().lower() == reference.strip().lower())

# Semantic similarity (embedding-based)
similarity = await calculate_similarity(
    response_embedding=embed(response),
    reference_embedding=embed(reference),
)

# BLEU/ROUGE for text generation
from cortex.evaluation.metrics import bleu_score, rouge_score

bleu = bleu_score(response, reference)
rouge = rouge_score(response, reference)
```

#### 3. Task-Specific Metrics

**RAG Evaluation**:

```python
from cortex.evaluation.rag import evaluate_rag

result = await evaluate_rag(
    query="What is GraphRAG?",
    retrieved_docs=[...],
    response="...",
    ground_truth_docs=[...],  # Optional golden set
)

# Metrics:
# {
#   "retrieval_precision": 0.8,   # Relevant docs / Retrieved docs
#   "retrieval_recall": 0.9,      # Relevant docs / Total relevant
#   "context_relevance": 0.85,    # Are retrieved docs useful?
#   "answer_faithfulness": 0.9,   # Does answer match context?
#   "answer_relevance": 0.95,     # Does answer match query?
# }
```

**Code Generation Evaluation**:

```python
from cortex.evaluation.code import evaluate_code

result = await evaluate_code(
    prompt="Write a function to reverse a list",
    generated_code="def reverse(lst): return lst[::-1]",
    test_cases=[
        {"input": [[1, 2, 3]], "expected": [3, 2, 1]},
        {"input": [[]], "expected": []},
    ],
)

# Metrics:
# {
#   "syntax_valid": True,
#   "tests_passed": 2,
#   "tests_failed": 0,
#   "execution_time_ms": 0.5,
# }
```

### Prompt Optimization

#### A/B Testing

```python
from cortex.evaluation import PromptExperiment

experiment = PromptExperiment(
    name="graphrag_explanation",
    variants={
        "v1": "Explain GraphRAG in simple terms.",
        "v2": "What is GraphRAG and how does it work?",
        "v3": "Describe GraphRAG, including its benefits over traditional RAG.",
    },
    evaluation_criteria=["clarity", "completeness", "conciseness"],
)

# Run experiment
results = await experiment.run(
    model="gpt-4o",
    num_trials=10,  # Run each variant 10 times
)

# Analyze results
best_variant = experiment.get_winner()
# {
#   "variant": "v3",
#   "avg_score": 0.92,
#   "confidence": 0.95,
#   "sample_response": "..."
# }
```

#### Regression Testing

```python
from cortex.evaluation import RegressionSuite

suite = RegressionSuite(
    test_cases=[
        {
            "id": "graphrag_basic",
            "prompt": "What is GraphRAG?",
            "min_score": 0.8,
            "reference": "GraphRAG combines...",
        },
        # ... more test cases
    ]
)

# Run on new prompt/model
report = await suite.run(
    model="gpt-4o",
    prompt_template=new_template,
)

# Report:
# {
#   "passed": 45,
#   "failed": 2,
#   "degraded": 3,  # Score dropped > 10%
#   "failures": [
#       {"id": "graphrag_basic", "expected": 0.8, "actual": 0.65}
#   ]
# }
```

---

## Advanced Observability

### Distributed Tracing

**Architecture** (from `cortex/orchestration/observability/telemetry.py`):

```
┌────────────────────────────────────────────────────────┐
│                 OpenTelemetry                           │
│  ┌──────────────────────────────────────────────────┐  │
│  │ TracerProvider                                    │  │
│  │ • Resource attributes (service.name, version)    │  │
│  │ • BaggageSpanProcessor (context propagation)     │  │
│  │ • BatchSpanProcessor (buffering)                 │  │
│  └──────────────────────────────────────────────────┘  │
│                        ↓                                │
│  ┌──────────────────────────────────────────────────┐  │
│  │ OTLP Exporter                                     │  │
│  │ • HTTP/gRPC protocol                             │  │
│  │ • Batching (every 2s, max 100 spans)            │  │
│  │ • Circuit breaker (fallback if collector down)  │  │
│  └──────────────────────────────────────────────────┘  │
│                        ↓                                │
└────────────────────────┼───────────────────────────────┘
                         ↓
            ┌────────────────────────┐
            │  Collector             │
            │  • Jaeger              │
            │  • Grafana Tempo       │
            │  • Datadog APM         │
            └────────────────────────┘
```

**Implementation**:

```python
from cortex.orchestration.observability import initialize_telemetry, get_tracer

# Initialize once at startup
initialize_telemetry(
    service_name="cortex-api",
    service_version="1.0.0",
    deployment_env="production",
    otlp_endpoint="https://tempo.example.com:4318",
)

# Get tracer
tracer = get_tracer(__name__)

# Create spans
@tracer.start_as_current_span("chat_request")
async def handle_chat(request: ChatRequest):
    span = trace.get_current_span()
    span.set_attribute("user.id", request.user_id)
    span.set_attribute("model", request.model)

    # Nested span for RAG
    with tracer.start_as_current_span("rag_retrieval") as rag_span:
        docs = await retriever.search(request.message)
        rag_span.set_attribute("docs.retrieved", len(docs))

    # Nested span for LLM
    with tracer.start_as_current_span("llm_generation") as llm_span:
        response = await llm.generate(request.message, context=docs)
        llm_span.set_attribute("tokens.prompt", response.usage.prompt_tokens)
        llm_span.set_attribute("tokens.completion", response.usage.completion_tokens)

    return response
```

**Trace Visualization** (Jaeger/Tempo):

```
chat_request [500ms]
├─ rag_retrieval [150ms]
│  ├─ vector_search [100ms]
│  └─ graph_search [50ms]
├─ llm_generation [300ms]
│  ├─ prompt_build [10ms]
│  ├─ llm_call [280ms]
│  └─ response_parse [10ms]
└─ response_format [50ms]
```

### LLM-Specific Metrics

**Token Usage Tracking** (from `cortex/orchestration/usage_tracking.py`):

```python
from cortex.orchestration import ModelUsageTracker

tracker = ModelUsageTracker()

# Automatic tracking from LangGraph events
async for event in agent.astream_events(message, version="v2"):
    tracker.record_from_event(event)
    # Emits SSE to client
    await stream_writer.write(event)

# Get aggregated usage
usage = tracker.get_usage()
# {
#   "gpt-4o": {
#     "prompt_tokens": 1500,
#     "completion_tokens": 800,
#     "total_tokens": 2300,
#     "cache": {
#       "cache_read": 500,      # Anthropic prompt caching
#       "cache_creation": 1000
#     }
#   }
# }

# Track cost
cost = calculate_cost(usage)
# {
#   "gpt-4o": {"input": 0.015, "output": 0.024, "total": 0.039},
#   "total": 0.039
# }
```

**Latency Tracking**:

```python
from cortex.orchestration.observability import LatencyTracker

latency = LatencyTracker()

with latency.track("llm_generation"):
    response = await llm.generate(prompt)

metrics = latency.get_metrics()
# {
#   "llm_generation": {
#     "p50": 280,  # ms
#     "p95": 450,
#     "p99": 600,
#     "avg": 320,
#     "count": 1000
#   }
# }
```

### Cost Tracking & Budgets

```python
from cortex.platform.billing import CostTracker

cost_tracker = CostTracker(tenant_id="acme")

# Track LLM costs
await cost_tracker.record(
    service="llm",
    model="gpt-4o",
    usage={
        "prompt_tokens": 1500,
        "completion_tokens": 800,
    },
    cost=0.039,
)

# Check budget
budget = await cost_tracker.get_budget(tenant_id="acme")
# {
#   "monthly_limit": 1000.00,
#   "current_spend": 245.67,
#   "remaining": 754.33,
#   "alert_threshold": 800.00,
#   "alerts": []
# }

# Alert if exceeded
if budget["current_spend"] > budget["alert_threshold"]:
    await send_alert("Budget threshold exceeded")
```

---

## Deployment Considerations

### SaaS Deployment

**Multi-Tenant Observability**:

```python
# Inject tenant context into all spans
from opentelemetry import baggage

# Set baggage at request entry
baggage.set_baggage("tenant.id", tenant_id)
baggage.set_baggage("organization.id", org_id)
baggage.set_baggage("project.id", project_id)

# Automatically copied to all spans via BaggageSpanProcessor
# Filter traces in Jaeger/Tempo by tenant.id
```

**Cost Attribution**:

```python
# Track costs per tenant
costs = await cost_tracker.get_tenant_costs(
    tenant_id="acme",
    start_date="2024-01-01",
    end_date="2024-01-31",
)

# Breakdown by service
# {
#   "llm": 450.23,
#   "embedding": 12.34,
#   "vector_search": 5.67,
#   "total": 468.24
# }
```

### Hybrid Deployment

**Trace forwarding** (customer data plane → SaaS control plane):

```python
# Customer data plane: Send traces to local collector
initialize_telemetry(
    service_name="cortex-customer",
    otlp_endpoint="http://localhost:4318",  # Local collector
)

# Local collector forwards to SaaS control plane
# (via OTLP HTTP exporter with authentication)
```

### On-Premise Deployment

**Self-hosted observability stack**:

```yaml
# docker-compose.yml
version: "3.8"
services:
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"  # UI
      - "4318:4318"    # OTLP HTTP

  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
```

---

## Performance Targets & SLAs

### AI Platform Performance Metrics

This section defines specific, measurable performance targets for advanced AI patterns. These targets complement the [Data Plane SLAs](./04-data-plane.md#performance-slas) with AI-specific metrics.

#### Graph RAG Performance

| Operation | Target (p95) | Target (p99) | Notes |
|-----------|-------------|-------------|-------|
| **Vector Search** | <50ms | <100ms | 10M vectors, HNSW index |
| **Graph Traversal (1-hop)** | <30ms | <60ms | Neo4j with indexed tenant_id |
| **Graph Traversal (2-hop)** | <100ms | <200ms | Cached concept expansions |
| **Hybrid Fusion (RRF)** | <150ms | <300ms | Parallel vector + graph execution |
| **Entity Extraction** | <200ms | <400ms | LLM-based NER |
| **End-to-End Graph RAG** | <3s | <6s | Retrieval + LLM generation |

**Optimization Strategies**:
```python
# Parallel execution reduces latency by 50%
vector_task = asyncio.create_task(vector_search(query))
graph_task = asyncio.create_task(graph_search(query))

# Wait for both (runs in parallel, not sequential)
vector_results, graph_results = await asyncio.gather(vector_task, graph_task)
# Total latency: max(vector, graph) instead of sum(vector, graph)
```

**Cache Hit Rates**:
- **Concept expansion cache**: 80% hit rate (Redis, 5-minute TTL)
- **Entity linking cache**: 70% hit rate (common entities)
- **Graph query results**: 60% hit rate (parameterized queries)

#### Multi-Agent Performance

| Pattern | Latency Target | Throughput | Notes |
|---------|---------------|-----------|-------|
| **Sequential Handoff** | <5s per agent | 20 agents/sec | Swarm pattern, 3 agents |
| **Parallel Workers** | <3s total | 50 tasks/sec | Hierarchical, 5 workers |
| **Workflow Execution** | <10s per workflow | 10 workflows/sec | State machine, 7 nodes |
| **Shared Memory Access** | <5ms | 10K ops/sec | Redis-backed memory pool |
| **Agent Tool Call** | <500ms | 100 calls/sec | Function execution |
| **Inter-Agent Message** | <10ms | 1K messages/sec | In-memory queue |

**Agent Scaling Limits**:
- **Max concurrent agents per tenant**: 100 (to prevent resource exhaustion)
- **Max swarm size**: 10 agents (beyond this, use hierarchical)
- **Max workflow depth**: 50 nodes (beyond this, split into sub-workflows)
- **Message queue depth**: 1,000 messages (backpressure threshold)

**Cost Optimization**:
```python
# Agent pooling reduces initialization overhead
agent_pool = {
    "researcher": Agent(...),  # Reused across requests
    "writer": Agent(...),
}

# Before: 500ms initialization per request
# After: 50ms warmup + <1ms retrieval from pool
# Improvement: 10x faster agent startup
```

#### LLM Gateway Performance

| Metric | Target | Notes |
|--------|--------|-------|
| **Semantic Cache Hit Rate** | >85% | Qdrant similarity threshold 0.95 |
| **Cache Lookup Latency** | <20ms (p95) | Vector search in Qdrant |
| **Routing Decision** | <10ms | Cost-based or latency-based routing |
| **Rate Limit Check** | <3ms | Redis sorted set (sliding window) |
| **Provider Failover** | <200ms | Circuit breaker, 3 failed requests |
| **Streaming First Token** | <200ms (p95) | Time to first LLM token |
| **Streaming Throughput** | >50 tokens/sec | SSE stream delivery |

**Cost Savings via Semantic Cache**:
```
Without Cache:
- 10,000 requests/day
- $0.01 per request (GPT-4 average)
- Monthly cost: $3,000

With 85% Cache Hit Rate:
- 8,500 cache hits (free)
- 1,500 cache misses ($0.01 each)
- Monthly cost: $450
- Savings: $2,550/month (85% reduction)
```

#### Evaluation Framework Performance

| Operation | Target Latency | Throughput | Notes |
|-----------|---------------|-----------|-------|
| **LLM-as-Judge Evaluation** | <5s | 20 evals/sec | GPT-4 as judge |
| **RAGAS Metric Calculation** | <10s | 10 evals/sec | Answer relevancy, faithfulness |
| **A/B Test Assignment** | <1ms | 100K assigns/sec | Hash-based bucketing |
| **Regression Test Suite** | <5 minutes | 100 tests | Full test suite execution |
| **Prompt Version Comparison** | <30s | 5 comparisons/min | Side-by-side evaluation |

**Evaluation Cost Optimization**:
- **Batch evaluations**: Process 100 samples in parallel (10x throughput)
- **Sample-based testing**: Evaluate 10% of production traffic (10x cost reduction)
- **Cached judgments**: Reuse LLM judge results for identical inputs (50% cost reduction)

#### Observability Performance

| Component | Overhead | Target | Notes |
|-----------|----------|--------|-------|
| **Tracing (10% sample)** | <3% latency | <50ms overhead | OpenTelemetry instrumentation |
| **Metrics Collection** | <1% CPU | <10ms/metric | Prometheus scraping |
| **Log Aggregation** | <5% CPU | <100MB/sec | Structured JSON logs |
| **Span Export** | Batched | 2s batch interval | OTLP exporter, 100 spans/batch |
| **Cost Tracking** | <1ms | 1M events/sec | Redis increment operations |

**Trace Sampling Strategy**:
```python
# Always trace errors, sample successful requests
from opentelemetry.sdk.trace.sampling import ParentBased, AlwaysOn, TraceIdRatioBased

sampler = ParentBased(
    root=TraceIdRatioBased(0.1),  # 10% of successful requests
    remote_parent_sampled=AlwaysOn(),  # Always trace if parent traced
    remote_parent_not_sampled=TraceIdRatioBased(0.1),
)

# Override for errors
if is_error:
    force_sampling()  # Always trace errors for debugging
```

### Performance Budgets & Monitoring

#### Latency Budgets

**End-to-End User Experience**:
```
Chat Request (Standard RAG):
┌─────────────────────────────────────────────────────────┐
│ API Gateway Auth:       50ms (p95)                       │
│ Vector Search:          50ms (p95)                       │
│ Semantic Cache Lookup:  20ms (p95)                       │
│ LLM Generation:        1500ms (p95)                      │
│ Response Serialization: 10ms (p95)                       │
│ ───────────────────────────────────────────────────────  │
│ Total:                 1630ms (p95) ✓ Target: <2000ms   │
└─────────────────────────────────────────────────────────┘

Chat Request (Graph RAG):
┌─────────────────────────────────────────────────────────┐
│ API Gateway Auth:       50ms (p95)                       │
│ Vector Search:          50ms (p95)  ┐                    │
│ Graph Search (2-hop):  100ms (p95)  ├─ Parallel (100ms) │
│ Hybrid Fusion (RRF):    20ms (p95)  ┘                    │
│ Semantic Cache Lookup:  20ms (p95)                       │
│ LLM Generation:        1500ms (p95)                      │
│ Response Serialization: 10ms (p95)                       │
│ ───────────────────────────────────────────────────────  │
│ Total:                 1750ms (p95) ✓ Target: <3000ms   │
└─────────────────────────────────────────────────────────┘

Multi-Agent Workflow (3 agents):
┌─────────────────────────────────────────────────────────┐
│ Orchestrator Planning: 200ms (p95)                       │
│ Agent 1 (Research):   2000ms (p95)                       │
│ Agent 2 (Analysis):   2000ms (p95)                       │
│ Agent 3 (Summary):    1500ms (p95)                       │
│ Result Aggregation:    100ms (p95)                       │
│ ───────────────────────────────────────────────────────  │
│ Total:                5800ms (p95) ✓ Target: <10000ms   │
└─────────────────────────────────────────────────────────┘
```

#### Resource Budgets

| Resource | Limit | Alert Threshold | Scaling Action |
|----------|-------|-----------------|----------------|
| **LLM Tokens/Request** | 4,000 tokens | 3,500 tokens | Compress context |
| **Agents/Tenant** | 100 concurrent | 80 concurrent | Queue new requests |
| **Memory Usage/Agent** | 512MB | 400MB | Evict old conversations |
| **Graph Query Depth** | 2 hops | 2 hops | Reject deeper queries |
| **Evaluation Batch Size** | 100 samples | 80 samples | Split into smaller batches |

#### SLA Guarantees

**Production SaaS**:
- **Availability**: 99.9% uptime (8.76 hours downtime/year)
- **Latency**: p95 < 2s for standard chat, p95 < 3s for Graph RAG
- **Error Rate**: <0.1% (excluding upstream LLM provider errors)
- **Data Retention**: 90 days for traces, 1 year for metrics

**Enterprise Self-Hosted**:
- **Availability**: 99.95% (customer-managed infrastructure)
- **Latency**: Configurable based on deployment resources
- **Data Retention**: Customer-defined (1 year default)

### Performance Regression Testing

**Automated Performance Tests**:

```python
# tests/performance/test_graph_rag_latency.py

import pytest
from cortex.rag.graph_rag import GraphRAGRetriever

@pytest.mark.performance
async def test_graph_rag_latency_p95():
    """Ensure Graph RAG p95 latency stays below 150ms."""

    retriever = GraphRAGRetriever(...)
    latencies = []

    # Run 100 queries
    for query in test_queries:
        start = time.time()
        await retriever.retrieve(query, max_hops=2)
        latency = (time.time() - start) * 1000  # ms

        latencies.append(latency)

    # Calculate p95
    p95_latency = np.percentile(latencies, 95)

    # Assert SLA
    assert p95_latency < 150, f"Graph RAG p95 latency {p95_latency}ms exceeds 150ms SLA"

@pytest.mark.performance
async def test_semantic_cache_hit_rate():
    """Ensure semantic cache hit rate stays above 85%."""

    cache = SemanticCache(...)
    hits = 0
    total = 0

    for query in test_queries:
        # First call: cache miss
        await cache.get_or_set(query, lambda: llm.generate(query))
        total += 1

        # Second call: should be cache hit
        result = await cache.get(query)
        if result:
            hits += 1
        total += 1

    hit_rate = hits / total

    # Assert SLA
    assert hit_rate > 0.85, f"Semantic cache hit rate {hit_rate:.1%} below 85% target"
```

**CI/CD Performance Gates**:

```yaml
# .github/workflows/performance.yml
name: Performance Tests

on:
  pull_request:
    branches: [main]

jobs:
  performance:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run performance tests
        run: pytest tests/performance/ --performance

      - name: Check performance regression
        run: |
          # Compare current p95 latencies with baseline
          python scripts/check_performance_regression.py \
            --baseline=performance_baseline.json \
            --current=performance_results.json \
            --threshold=10%  # Fail if >10% regression

      - name: Update performance dashboard
        run: |
          # Push metrics to monitoring dashboard
          python scripts/update_performance_metrics.py
```

### Cost Monitoring & Optimization

**Cost Tracking by Component**:

```python
# cortex/platform/observability/cost_tracker.py

class CostTracker:
    """Track costs across AI platform components."""

    async def track_llm_call(
        self,
        tenant_id: str,
        model: str,
        prompt_tokens: int,
        completion_tokens: int,
        cached: bool = False
    ):
        """Track LLM API call costs."""

        if cached:
            cost = 0  # Semantic cache hit = zero cost
        else:
            # Calculate cost based on model pricing
            cost = self._calculate_llm_cost(model, prompt_tokens, completion_tokens)

        # Record cost metric
        await self.metrics.record(
            metric="llm_cost_usd",
            value=cost,
            labels={
                "tenant_id": tenant_id,
                "model": model,
                "cached": str(cached)
            }
        )

        # Check budget alerts
        total_cost = await self.get_tenant_cost_mtd(tenant_id)
        budget = await self.get_tenant_budget(tenant_id)

        if total_cost > budget * 0.8:  # 80% threshold
            await self.send_budget_alert(tenant_id, total_cost, budget)

    async def get_cost_breakdown(
        self,
        tenant_id: str,
        start_date: str,
        end_date: str
    ) -> dict:
        """Get cost breakdown by component."""

        return {
            "llm_calls": await self._get_llm_costs(tenant_id, start_date, end_date),
            "embeddings": await self._get_embedding_costs(tenant_id, start_date, end_date),
            "vector_search": await self._get_vector_search_costs(tenant_id, start_date, end_date),
            "graph_queries": await self._get_graph_query_costs(tenant_id, start_date, end_date),
            "total": sum([...])
        }

# Monthly cost dashboard
{
    "tenant_id": "acct_123",
    "period": "2024-01",
    "costs": {
        "llm_calls": {
            "total": 450.23,
            "cached_savings": 2550.00,  # 85% cache hit rate
            "breakdown": {
                "gpt-4o": 350.00,
                "gpt-4o-mini": 100.23
            }
        },
        "embeddings": {
            "total": 12.34,
            "volume": 1.2M  # tokens embedded
        },
        "vector_search": {
            "total": 5.67,
            "queries": 100K
        },
        "graph_queries": {
            "total": 8.90,
            "queries": 50K
        },
        "total": 477.14,
        "budget": 1000.00,
        "utilization": "47.7%"
    }
}
```

**Cost Optimization Recommendations**:

1. **Semantic Caching**: 85% hit rate → 90% cost savings on LLM calls
2. **Model Selection**: Use `gpt-4o-mini` for simple queries (10x cheaper than `gpt-4o`)
3. **Prompt Compression**: Reduce token count by 30% via compression techniques
4. **Batch Processing**: Group embeddings into batches of 100 (5x throughput)
5. **Query Result Caching**: Cache graph query results (60% hit rate)

---

## Performance & Scaling

### Graph RAG Optimization

**Challenges**:
- Graph queries can be slow (multi-hop traversal)
- Vector + graph = 2x latency

**Solutions**:

1. **Parallel execution**:
   ```python
   # Run vector and graph searches in parallel
   vector_task = asyncio.create_task(vector_store.search(...))
   graph_task = asyncio.create_task(graph_store.search(...))

   vector_results, graph_results = await asyncio.gather(vector_task, graph_task)
   ```

2. **Graph query caching**:
   ```python
   # Cache frequent concept expansions
   cache_key = f"graph:concepts:{concept_name}:{max_hops}"
   cached = await redis.get(cache_key)
   if cached:
       return json.loads(cached)
   ```

3. **Adaptive hop limits**:
   ```python
   # Use 1-hop for broad queries, 2-hop for entity-specific
   max_hops = 2 if has_named_entities(query) else 1
   ```

### Multi-Agent Scaling

**Challenges**:
- Each agent consumes LLM tokens
- Sequential handoffs increase latency

**Solutions**:

1. **Agent pooling**:
   ```python
   # Reuse agent instances across requests
   agent_pool = {
       "researcher": Agent(...),
       "writer": Agent(...),
   }

   # Concurrent requests use same agent instances
   ```

2. **Parallel agent execution** (when independent):
   ```python
   # Supervisor delegates to workers in parallel
   results = await asyncio.gather(
       worker1.run(task1),
       worker2.run(task2),
       worker3.run(task3),
   )
   ```

3. **Streaming handoffs**:
   ```python
   # Stream tokens from Agent A → Agent B without waiting for completion
   async for token in agent_a.stream(message):
       await stream_writer.write(token)
       await agent_b.process_token(token)  # Real-time handoff
   ```

### Observability Performance

**Tracing overhead**: ~1-3% latency increase

**Mitigation**:

```python
# Sample traces (don't trace every request)
from opentelemetry.sdk.trace.sampling import ParentBasedTraceIdRatio

sampler = ParentBasedTraceIdRatio(0.1)  # Sample 10% of traces
tracer_provider = TracerProvider(sampler=sampler)
```

**Batch exports**:

```python
# Buffer spans and export in batches
BatchSpanProcessor(
    exporter,
    schedule_delay_millis=2000,    # Export every 2s
    max_export_batch_size=100,     # Batch size
)
```

---

## Summary

**Graph RAG Integration**:
- Combines vector similarity + knowledge graph traversal
- Reciprocal Rank Fusion (RRF) for result fusion
- Multi-hop reasoning with explainability
- Parallel execution for performance

**Multi-Agent Coordination**:
- Swarm (peer handoff), Hierarchical (supervisor/worker), Workflow (state machine)
- Shared memory pool for cross-agent context
- Automatic handoff tool injection

**Evaluation Framework**:
- LLM-as-judge for quality metrics
- A/B testing for prompt optimization
- Regression testing for quality assurance
- Task-specific metrics (RAG, code generation)

**Advanced Observability**:
- OpenTelemetry distributed tracing
- LLM-specific metrics (tokens, latency, cost)
- Multi-tenant trace filtering
- Cost tracking and budget alerts

**Next Steps**:
- Review [05-ai-platform-services.md](./05-ai-platform-services.md) for LLM Gateway, Unified Chat, and Memory Management
- Explore [cortex-ai reference implementation](../../cortex-ai/) for working code examples
- Deploy observability stack (Jaeger, Prometheus, Grafana) for production monitoring
