# Advanced AI Patterns

## Overview

Advanced patterns for AI Stack including Graph RAG, multi-agent coordination, semantic caching optimization, LLM evaluation, and prompt engineering.

---

## Graph RAG with Reciprocal Rank Fusion

### Architecture

```
User Query → Embed Query → Parallel Search
                              ├─ Vector Search (Qdrant)
                              └─ Graph Search (Neo4j)
                                    ↓
                            Reciprocal Rank Fusion
                                    ↓
                            Rerank with LLM
                                    ↓
                            Generate Response
```

### Reciprocal Rank Fusion (RRF)

```python
class GraphRAG:
    """Graph RAG with Reciprocal Rank Fusion."""

    async def retrieve(
        self,
        query: str,
        tenant_id: str,
        k: int = 10
    ) -> list[Document]:
        """Retrieve with Graph RAG."""

        # 1. Embed query
        query_embedding = await self.embedder.embed(query)

        # 2. Parallel retrieval
        vector_results, graph_results = await asyncio.gather(
            self._vector_search(query_embedding, tenant_id, k=20),
            self._graph_search(query, tenant_id, k=20)
        )

        # 3. Reciprocal Rank Fusion
        fused_results = self._reciprocal_rank_fusion(
            [vector_results, graph_results],
            k=k
        )

        # 4. Rerank with LLM (optional)
        if self.rerank_enabled:
            fused_results = await self._rerank_with_llm(
                query=query,
                documents=fused_results,
                top_k=k
            )

        return fused_results[:k]

    def _reciprocal_rank_fusion(
        self,
        result_sets: list[list[Document]],
        k: int = 60
    ) -> list[Document]:
        """Fuse multiple result sets using RRF.

        Formula: RRF_score(d) = Σ 1/(k + rank_i(d))
        where rank_i(d) is the rank of document d in result set i
        """

        # Calculate RRF scores
        rrf_scores = defaultdict(float)

        for result_set in result_sets:
            for rank, doc in enumerate(result_set, start=1):
                rrf_scores[doc.id] += 1.0 / (k + rank)

        # Sort by RRF score (descending)
        doc_map = {doc.id: doc for result_set in result_sets for doc in result_set}
        sorted_docs = sorted(
            rrf_scores.items(),
            key=lambda x: x[1],
            reverse=True
        )

        return [doc_map[doc_id] for doc_id, _ in sorted_docs]

    async def _rerank_with_llm(
        self,
        query: str,
        documents: list[Document],
        top_k: int = 10
    ) -> list[Document]:
        """Rerank documents using LLM as judge."""

        prompt = f"""Query: {query}

Rank the following documents by relevance (1 = most relevant):

{self._format_documents(documents)}

Return only a JSON array of document IDs in order of relevance.
"""

        response = await llm_gateway.complete(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": prompt}],
            response_format={"type": "json_object"}
        )

        ranked_ids = json.loads(response.content)
        doc_map = {doc.id: doc for doc in documents}

        return [doc_map[doc_id] for doc_id in ranked_ids[:top_k]]
```

### Graph Search with Entity Expansion

```python
async def _graph_search(
    self,
    query: str,
    tenant_id: str,
    k: int = 20
) -> list[Document]:
    """Graph search with entity expansion."""

    # 1. Extract entities from query
    entities = await self._extract_entities(query)

    # 2. Find related entities in knowledge graph
    cypher_query = """
    MATCH (e:Entity)-[r]-(related:Entity)
    WHERE e.name IN $entity_names
      AND e.tenant_id = $tenant_id
    RETURN related, r, e
    ORDER BY r.weight DESC
    LIMIT $limit
    """

    results = await neo4j.query(
        cypher_query,
        {
            "entity_names": [e.name for e in entities],
            "tenant_id": tenant_id,
            "limit": k * 2
        }
    )

    # 3. Get documents linked to entities
    document_ids = set()
    for record in results:
        related_entity = record["related"]
        # Get documents mentioning this entity
        docs = await self._get_documents_for_entity(
            related_entity["id"],
            tenant_id
        )
        document_ids.update(doc.id for doc in docs)

    # 4. Retrieve full documents
    documents = await db.query(Document).filter(
        Document.id.in_(document_ids)
    ).all()

    return documents[:k]
```

---

## Multi-Agent Coordination Patterns

### Swarm Pattern

```python
class AgentSwarm:
    """Coordinate multiple specialized agents."""

    def __init__(self):
        self.agents = {
            "researcher": ResearchAgent(),
            "coder": CodingAgent(),
            "reviewer": ReviewAgent(),
            "writer": WriterAgent()
        }

    async def execute(
        self,
        task: str,
        tenant_id: str
    ) -> AgentResult:
        """Execute task with agent swarm."""

        # 1. Decompose task
        subtasks = await self._decompose_task(task)

        # 2. Assign agents to subtasks
        assignments = self._assign_agents(subtasks)

        # 3. Execute in parallel
        results = await asyncio.gather(*[
            agent.execute(subtask, tenant_id)
            for agent, subtask in assignments
        ])

        # 4. Synthesize results
        final_result = await self._synthesize(task, results)

        return final_result

    def _assign_agents(
        self,
        subtasks: list[Subtask]
    ) -> list[tuple[Agent, Subtask]]:
        """Assign best agent for each subtask."""

        assignments = []
        for subtask in subtasks:
            # Score each agent for this subtask
            scores = {
                name: agent.score_suitability(subtask)
                for name, agent in self.agents.items()
            }

            # Assign best agent
            best_agent_name = max(scores, key=scores.get)
            assignments.append((self.agents[best_agent_name], subtask))

        return assignments
```

### Hierarchical Pattern

```python
class HierarchicalAgents:
    """Hierarchical agent coordination."""

    async def execute(
        self,
        task: str,
        tenant_id: str
    ) -> AgentResult:
        """Execute with hierarchical coordination."""

        # Manager agent plans and delegates
        manager = ManagerAgent()
        plan = await manager.create_plan(task)

        results = []
        for step in plan.steps:
            if step.type == "delegate":
                # Delegate to worker agent
                worker = self._get_worker(step.worker_type)
                result = await worker.execute(step.task, tenant_id)
                results.append(result)

            elif step.type == "review":
                # Review previous results
                reviewer = ReviewAgent()
                review = await reviewer.review(results[-1])

                if not review.approved:
                    # Retry with feedback
                    worker = self._get_worker(step.worker_type)
                    result = await worker.execute(
                        step.task,
                        tenant_id,
                        feedback=review.feedback
                    )
                    results[-1] = result

        # Manager synthesizes final result
        final_result = await manager.synthesize(task, results)
        return final_result
```

---

## Semantic Caching Optimization

### Cache Key Generation

```python
class SemanticCacheOptimizer:
    """Optimize semantic cache hit rates."""

    async def get_cache_key(
        self,
        query: str,
        context: dict = None,
        similarity_threshold: float = 0.95
    ) -> str:
        """Generate cache key with normalization."""

        # 1. Normalize query
        normalized_query = self._normalize_query(query)

        # 2. Generate embedding
        embedding = await self.embedder.embed(normalized_query)

        # 3. Search for similar cached queries
        results = await qdrant.search(
            collection_name="semantic_cache",
            query_vector=embedding,
            limit=1,
            score_threshold=similarity_threshold
        )

        if results:
            # Cache hit - return existing key
            return results[0].payload["cache_key"]

        # Cache miss - generate new key
        cache_key = hashlib.sha256(
            normalized_query.encode()
        ).hexdigest()

        return cache_key

    def _normalize_query(self, query: str) -> str:
        """Normalize query for better cache hits."""

        # Remove extra whitespace
        query = re.sub(r'\s+', ' ', query.strip())

        # Lowercase
        query = query.lower()

        # Remove punctuation (except meaningful ones)
        query = re.sub(r'[^\w\s\?\.\!]', '', query)

        # Stemming (optional, can reduce hit rate)
        # query = self._stem(query)

        return query
```

### Cache Warming Strategy

```python
class CacheWarmer:
    """Warm semantic cache with common queries."""

    async def warm_cache(
        self,
        tenant_id: str,
        common_queries: list[str]
    ):
        """Pre-populate cache with common queries."""

        for query in common_queries:
            # Check if already cached
            cache_key = await cache.get_cache_key(query)
            cached_response = await cache.get(cache_key, tenant_id)

            if not cached_response:
                # Generate and cache response
                response = await llm_gateway.complete(
                    model="gpt-4o",
                    messages=[{"role": "user", "content": query}],
                    tenant_id=tenant_id
                )

                await cache.set(
                    cache_key=cache_key,
                    query=query,
                    response=response,
                    tenant_id=tenant_id,
                    ttl=3600 * 24  # 24 hours
                )

                logger.info(
                    "cache_warmed",
                    tenant_id=tenant_id,
                    query=query[:50]
                )
```

### Cache Invalidation

```python
class CacheInvalidator:
    """Invalidate stale cache entries."""

    async def invalidate_by_document(
        self,
        document_id: str,
        tenant_id: str
    ):
        """Invalidate cache entries related to document."""

        # 1. Find cached queries mentioning this document
        # (requires tracking document → cache_key mapping)
        cache_keys = await self._get_cache_keys_for_document(
            document_id,
            tenant_id
        )

        # 2. Delete cache entries
        for cache_key in cache_keys:
            await cache.delete(cache_key, tenant_id)

        logger.info(
            "cache_invalidated",
            document_id=document_id,
            invalidated_count=len(cache_keys)
        )

    async def invalidate_by_age(
        self,
        max_age_hours: int = 24
    ):
        """Invalidate cache entries older than threshold."""

        cutoff = datetime.utcnow() - timedelta(hours=max_age_hours)

        # Delete old entries from Qdrant
        await qdrant.delete(
            collection_name="semantic_cache",
            points_selector={
                "filter": {
                    "must": [{
                        "key": "cached_at",
                        "range": {"lt": cutoff.isoformat()}
                    }]
                }
            }
        )
```

---

## LLM Evaluation Patterns

### LLM-as-Judge

```python
class LLMEvaluator:
    """Evaluate LLM outputs using LLM-as-Judge."""

    async def evaluate_response(
        self,
        query: str,
        response: str,
        context: str = None
    ) -> EvaluationResult:
        """Evaluate response quality."""

        prompt = f"""Evaluate the following AI response on a scale of 1-5 for:
1. Accuracy: Is the response factually correct?
2. Relevance: Does it answer the question?
3. Completeness: Is the answer comprehensive?
4. Clarity: Is it well-written and clear?

Query: {query}

Response: {response}

{f"Context: {context}" if context else ""}

Return a JSON object with scores and reasoning:
{{
  "accuracy": {{"score": 1-5, "reasoning": "..."}},
  "relevance": {{"score": 1-5, "reasoning": "..."}},
  "completeness": {{"score": 1-5, "reasoning": "..."}},
  "clarity": {{"score": 1-5, "reasoning": "..."}},
  "overall_score": 1-5,
  "pass": true/false
}}
"""

        evaluation = await llm_gateway.complete(
            model="gpt-4o",  # Use best model for evaluation
            messages=[{"role": "user", "content": prompt}],
            response_format={"type": "json_object"},
            temperature=0.0  # Deterministic evaluation
        )

        result = json.loads(evaluation.content)
        return EvaluationResult(**result)
```

### RAG Evaluation Metrics

```python
class RAGEvaluator:
    """Evaluate RAG performance."""

    async def evaluate_retrieval(
        self,
        query: str,
        retrieved_docs: list[Document],
        ground_truth_docs: list[Document]
    ) -> RetrievalMetrics:
        """Evaluate retrieval quality."""

        retrieved_ids = {doc.id for doc in retrieved_docs}
        ground_truth_ids = {doc.id for doc in ground_truth_docs}

        # Precision: How many retrieved docs are relevant?
        precision = len(retrieved_ids & ground_truth_ids) / len(retrieved_ids)

        # Recall: How many relevant docs were retrieved?
        recall = len(retrieved_ids & ground_truth_ids) / len(ground_truth_ids)

        # F1 Score
        f1 = 2 * (precision * recall) / (precision + recall) if (precision + recall) > 0 else 0

        # Mean Reciprocal Rank (MRR)
        mrr = self._calculate_mrr(retrieved_docs, ground_truth_ids)

        # NDCG (Normalized Discounted Cumulative Gain)
        ndcg = self._calculate_ndcg(retrieved_docs, ground_truth_ids)

        return RetrievalMetrics(
            precision=precision,
            recall=recall,
            f1=f1,
            mrr=mrr,
            ndcg=ndcg
        )

    async def evaluate_generation(
        self,
        query: str,
        generated_response: str,
        ground_truth: str,
        context: list[Document]
    ) -> GenerationMetrics:
        """Evaluate generation quality."""

        # 1. Factual consistency (using LLM-as-Judge)
        factual_score = await self._check_factual_consistency(
            generated_response,
            context
        )

        # 2. Answer relevance
        relevance_score = await self._check_answer_relevance(
            query,
            generated_response
        )

        # 3. Context precision (how much context is actually used?)
        context_precision = await self._check_context_precision(
            generated_response,
            context
        )

        # 4. BLEU/ROUGE (if ground truth available)
        bleu_score = self._calculate_bleu(generated_response, ground_truth)
        rouge_score = self._calculate_rouge(generated_response, ground_truth)

        return GenerationMetrics(
            factual_consistency=factual_score,
            answer_relevance=relevance_score,
            context_precision=context_precision,
            bleu=bleu_score,
            rouge=rouge_score
        )
```

---

## Prompt Engineering Patterns

### Few-Shot Learning

```python
class FewShotPromptBuilder:
    """Build few-shot prompts for better performance."""

    def build_prompt(
        self,
        task: str,
        examples: list[Example],
        query: str
    ) -> str:
        """Build few-shot prompt."""

        prompt = f"""Task: {task}

Here are some examples:

"""

        # Add examples
        for i, example in enumerate(examples, start=1):
            prompt += f"""Example {i}:
Input: {example.input}
Output: {example.output}

"""

        # Add actual query
        prompt += f"""Now complete this:
Input: {query}
Output:"""

        return prompt
```

### Chain-of-Thought (CoT)

```python
class ChainOfThoughtPrompt:
    """Generate prompts with chain-of-thought reasoning."""

    def build_prompt(
        self,
        task: str,
        query: str,
        include_examples: bool = True
    ) -> str:
        """Build CoT prompt."""

        prompt = f"""Task: {task}

Let's approach this step-by-step:
"""

        if include_examples:
            prompt += """
Example:
Question: What is 15% of 80?
Reasoning:
1. Convert percentage to decimal: 15% = 0.15
2. Multiply: 0.15 × 80 = 12
Answer: 12

"""

        prompt += f"""
Question: {query}
Reasoning:"""

        return prompt
```

### Prompt Templating

```python
class PromptTemplate:
    """Template-based prompt generation."""

    def __init__(self, template: str):
        self.template = template

    def render(self, **kwargs) -> str:
        """Render template with variables."""

        # Use Jinja2 for advanced templating
        from jinja2 import Template
        template = Template(self.template)
        return template.render(**kwargs)


# Example usage
EXTRACTION_TEMPLATE = """Extract entities from the following text.

Text: {{ text }}

Extract the following entity types:
{% for entity_type in entity_types %}
- {{ entity_type }}
{% endfor %}

Return a JSON object with entity types as keys and arrays of entities as values.
"""

prompt_template = PromptTemplate(EXTRACTION_TEMPLATE)
prompt = prompt_template.render(
    text="Apple Inc. was founded by Steve Jobs in Cupertino, California.",
    entity_types=["Organization", "Person", "Location"]
)
```

---

## Agent Memory Patterns

### Sliding Window Memory

```python
class SlidingWindowMemory:
    """Keep only recent N messages in context."""

    def __init__(self, window_size: int = 10):
        self.window_size = window_size

    async def get_context(
        self,
        conversation_id: str,
        tenant_id: str
    ) -> list[Message]:
        """Get recent messages."""

        messages = await db.query(Message).filter_by(
            conversation_id=conversation_id,
            tenant_id=tenant_id
        ).order_by(Message.created_at.desc()).limit(self.window_size).all()

        return list(reversed(messages))  # Chronological order
```

### Summarization Memory

```python
class SummarizationMemory:
    """Summarize old messages to save context."""

    async def get_context(
        self,
        conversation_id: str,
        tenant_id: str
    ) -> str:
        """Get summarized context."""

        # Get all messages
        messages = await db.query(Message).filter_by(
            conversation_id=conversation_id,
            tenant_id=tenant_id
        ).order_by(Message.created_at).all()

        # Recent messages (full context)
        recent_messages = messages[-10:]

        # Old messages (summarize)
        old_messages = messages[:-10]

        if old_messages:
            # Summarize old conversation
            summary = await llm_gateway.complete(
                model="gpt-4o-mini",
                messages=[{
                    "role": "user",
                    "content": f"Summarize this conversation:\n\n{self._format_messages(old_messages)}"
                }],
                tenant_id=tenant_id
            )

            context = f"Previous conversation summary: {summary.content}\n\nRecent messages:\n"
        else:
            context = "Recent messages:\n"

        context += self._format_messages(recent_messages)
        return context
```

### Semantic Memory (Entity-based)

```python
class SemanticMemory:
    """Track entities and facts across conversation."""

    async def update_memory(
        self,
        conversation_id: str,
        message: str,
        tenant_id: str
    ):
        """Extract and store entities/facts."""

        # Extract entities
        entities = await self._extract_entities(message)

        for entity in entities:
            # Store in Neo4j knowledge graph
            await neo4j.query("""
                MERGE (e:Entity {name: $name, type: $type, tenant_id: $tenant_id})
                ON CREATE SET e.first_mentioned = timestamp()
                ON MATCH SET e.last_mentioned = timestamp()

                MERGE (c:Conversation {id: $conversation_id, tenant_id: $tenant_id})
                MERGE (c)-[:MENTIONS]->(e)
            """, {
                "name": entity.name,
                "type": entity.type,
                "tenant_id": tenant_id,
                "conversation_id": conversation_id
            })

    async def recall(
        self,
        query: str,
        conversation_id: str,
        tenant_id: str
    ) -> list[Fact]:
        """Recall relevant facts from memory."""

        # Extract entities from query
        query_entities = await self._extract_entities(query)

        # Find related facts in graph
        facts = []
        for entity in query_entities:
            cypher = """
            MATCH (e:Entity {name: $name, tenant_id: $tenant_id})-[r]-(related)
            WHERE (related)-[:MENTIONS]-(:Conversation {id: $conversation_id})
            RETURN e, r, related
            LIMIT 10
            """

            results = await neo4j.query(cypher, {
                "name": entity.name,
                "tenant_id": tenant_id,
                "conversation_id": conversation_id
            })

            facts.extend(self._format_facts(results))

        return facts
```

---

## Performance Optimization

### Batch Processing

```python
class BatchProcessor:
    """Process multiple requests in batches."""

    async def embed_batch(
        self,
        texts: list[str],
        batch_size: int = 100
    ) -> list[list[float]]:
        """Embed texts in batches."""

        embeddings = []

        for i in range(0, len(texts), batch_size):
            batch = texts[i:i + batch_size]

            # Batch embed
            batch_embeddings = await embedding_service.embed_batch(batch)
            embeddings.extend(batch_embeddings)

        return embeddings
```

### Parallel Execution

```python
async def parallel_rag_search(
    queries: list[str],
    tenant_id: str
) -> list[list[Document]]:
    """Execute multiple RAG searches in parallel."""

    results = await asyncio.gather(*[
        rag_service.search(query, tenant_id)
        for query in queries
    ])

    return results
```

### Streaming Responses

```python
async def stream_llm_response(
    query: str,
    tenant_id: str
) -> AsyncIterator[str]:
    """Stream LLM response for better UX."""

    stream = await llm_gateway.stream(
        model="gpt-4o",
        messages=[{"role": "user", "content": query}],
        tenant_id=tenant_id
    )

    async for chunk in stream:
        if chunk.choices[0].delta.content:
            yield chunk.choices[0].delta.content
```

---

## Cost Optimization

### Model Selection Strategy

```python
class CostOptimizedRouter:
    """Route to cheapest model that meets requirements."""

    MODEL_COSTS = {
        "gpt-4o": {"input": 2.50, "output": 10.00},  # per 1M tokens
        "gpt-4o-mini": {"input": 0.15, "output": 0.60},
        "claude-3-5-sonnet": {"input": 3.00, "output": 15.00},
        "claude-3-haiku": {"input": 0.25, "output": 1.25}
    }

    def select_model(
        self,
        task_complexity: str,  # 'simple', 'medium', 'complex'
        max_cost_per_1k_tokens: float = None
    ) -> str:
        """Select most cost-effective model."""

        if task_complexity == "simple":
            # Use cheapest model
            return "gpt-4o-mini"

        elif task_complexity == "medium":
            # Balance cost and quality
            if max_cost_per_1k_tokens and max_cost_per_1k_tokens < 0.001:
                return "gpt-4o-mini"
            return "claude-3-haiku"

        else:  # complex
            # Use best model
            return "gpt-4o"
```

### Token Optimization

```python
class TokenOptimizer:
    """Optimize prompts to reduce token usage."""

    def compress_prompt(self, prompt: str) -> str:
        """Compress prompt while preserving meaning."""

        # Remove extra whitespace
        prompt = re.sub(r'\s+', ' ', prompt.strip())

        # Remove unnecessary words
        prompt = re.sub(r'\b(please|kindly|very|really)\b', '', prompt, flags=re.IGNORECASE)

        return prompt

    def truncate_context(
        self,
        context: str,
        max_tokens: int = 4000
    ) -> str:
        """Truncate context to fit token budget."""

        # Estimate tokens (rough: 1 token ≈ 4 characters)
        estimated_tokens = len(context) // 4

        if estimated_tokens <= max_tokens:
            return context

        # Truncate to max tokens
        max_chars = max_tokens * 4
        return context[:max_chars] + "..."
```

---

**Previous**: [Traditional Patterns](./06a-traditional-patterns.md) | [Back to Overview](../00-parallel-stacks-overview.md)
