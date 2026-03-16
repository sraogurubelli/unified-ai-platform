# AI Services

## Overview

The **AI Services** layer implements AI-native capabilities including LLM gateway, agent orchestration, RAG (Retrieval-Augmented Generation), memory management, and evaluation.

```
┌──────────────────────────────────────────────────────────────┐
│                  AI Services Layer                            │
├──────────────┬───────────────┬──────────────────────────────┤
│ LLM Gateway  │ Agent Orch    │ Evaluation Service            │
│ Chat Service │ RAG Service   │ Fine-tuning Service           │
│ Memory Mgmt  │ Tool Registry │ Embedding Service             │
└──────────────┴───────────────┴────────────────────────────────┘
```

---

## LLM Gateway

### Multi-Provider LLM Gateway

```python
class LLMGateway:
    """Unified gateway for multiple LLM providers."""

    def __init__(self):
        self.providers = {
            "openai": OpenAIProvider(),
            "anthropic": AnthropicProvider(),
            "cohere": CohereProvider(),
        }
        self.model_router = ModelRouter()
        self.semantic_cache = SemanticCache()

    async def complete(
        self,
        prompt: str,
        tenant_id: str,
        model: str = None,
        max_tokens: int = 1000,
        temperature: float = 0.7
    ) -> LLMResponse:
        """
        Complete prompt with LLM.

        Features:
        - Semantic caching (90% cost savings)
        - Smart model routing
        - Circuit breaker pattern
        - Cost tracking
        """

        # Check semantic cache
        cached_response = await self.semantic_cache.get(
            query=prompt,
            tenant_id=tenant_id
        )

        if cached_response:
            return LLMResponse(
                text=cached_response,
                cached=True,
                cost_saved=self._estimate_cost(prompt, model)
            )

        # Select model if not specified
        if not model:
            model = await self.model_router.route(prompt, tenant_id)

        # Get provider
        provider_name = self._get_provider_for_model(model)
        provider = self.providers[provider_name]

        # Execute with circuit breaker
        try:
            response = await self._execute_with_circuit_breaker(
                provider.complete,
                prompt=prompt,
                model=model,
                max_tokens=max_tokens,
                temperature=temperature
            )

            # Cache response
            await self.semantic_cache.set(
                query=prompt,
                response=response.text,
                tenant_id=tenant_id
            )

            # Track usage
            await llm_usage_tracker.record(
                tenant_id=tenant_id,
                model=model,
                prompt_tokens=response.usage.prompt_tokens,
                completion_tokens=response.usage.completion_tokens,
                cost_usd=self._calculate_cost(response.usage, model)
            )

            return response

        except CircuitBreakerOpen:
            # Fallback to different provider
            fallback_model = self._get_fallback_model(model)
            return await self.complete(
                prompt,
                tenant_id,
                model=fallback_model,
                max_tokens=max_tokens,
                temperature=temperature
            )

class OpenAIProvider:
    """OpenAI LLM provider."""

    async def complete(
        self,
        prompt: str,
        model: str,
        max_tokens: int,
        temperature: float
    ) -> LLMResponse:
        """Complete using OpenAI API."""

        response = await openai_client.chat.completions.create(
            model=model,
            messages=[{"role": "user", "content": prompt}],
            max_tokens=max_tokens,
            temperature=temperature
        )

        return LLMResponse(
            text=response.choices[0].message.content,
            usage=response.usage,
            model=model,
            cached=False
        )
```

---

## Chat Service

### Unified Chat Service

```python
class ChatService:
    """Manage chat conversations with context management."""

    async def create_conversation(
        self,
        tenant_id: str,
        project_id: str,
        principal_id: int
    ) -> Conversation:
        """Create new conversation."""

        conversation = Conversation(
            uid=f"conv_{uuid.uuid4().hex[:12]}",
            project_id=project_id,
            principal_id=principal_id,
            thread_id=str(uuid.uuid4()),
            created_at=datetime.utcnow()
        )

        await db.add(conversation)
        await db.commit()

        return conversation

    async def send_message(
        self,
        tenant_id: str,
        conversation_id: str,
        message: str,
        principal_id: int
    ) -> dict:
        """
        Send message and get AI response.

        Features:
        - Context retrieval (RAG)
        - Memory management
        - Streaming responses
        """

        # Get conversation
        conversation = await db.query(Conversation).filter_by(
            uid=conversation_id,
            tenant_id=tenant_id
        ).first()

        if not conversation:
            raise NotFoundError("Conversation not found")

        # Store user message
        user_message = Message(
            uid=f"msg_{uuid.uuid4().hex[:12]}",
            conversation_id=conversation.id,
            role="user",
            content=message
        )

        await db.add(user_message)
        await db.commit()

        # Retrieve relevant context (RAG)
        context = await rag_service.retrieve(
            query=message,
            tenant_id=tenant_id,
            project_id=conversation.project_id
        )

        # Get conversation history
        history = await self._get_conversation_history(
            conversation_id=conversation.id,
            limit=10
        )

        # Build prompt with context
        prompt = self._build_prompt_with_context(
            message=message,
            context=context,
            history=history
        )

        # Get LLM response
        llm_response = await llm_gateway.complete(
            prompt=prompt,
            tenant_id=tenant_id
        )

        # Store assistant message
        assistant_message = Message(
            uid=f"msg_{uuid.uuid4().hex[:12]}",
            conversation_id=conversation.id,
            role="assistant",
            content=llm_response.text
        )

        await db.add(assistant_message)
        await db.commit()

        return {
            "message_id": assistant_message.uid,
            "content": llm_response.text,
            "cached": llm_response.cached,
            "context_sources": [c["id"] for c in context]
        }
```

---

## RAG Service

### Retrieval-Augmented Generation

```python
class RAGService:
    """RAG with vector search and knowledge graph."""

    async def retrieve(
        self,
        query: str,
        tenant_id: str,
        project_id: str = None,
        top_k: int = 5,
        use_graph: bool = True
    ) -> list[dict]:
        """
        Retrieve relevant context for query.

        Modes:
        - Vector-only: Semantic search in Qdrant
        - Graph RAG: Vector + Knowledge graph fusion
        """

        if use_graph:
            # Graph RAG (hybrid)
            return await self._graph_rag_retrieve(
                query, tenant_id, project_id, top_k
            )
        else:
            # Vector-only
            return await self._vector_retrieve(
                query, tenant_id, project_id, top_k
            )

    async def _vector_retrieve(
        self,
        query: str,
        tenant_id: str,
        project_id: str,
        top_k: int
    ) -> list[dict]:
        """Vector-based retrieval."""

        # Generate query embedding
        query_embedding = await embedding_service.embed(query)

        # Search Qdrant
        results = await qdrant_client.search(
            collection_name=f"embeddings_{tenant_id}",
            query_vector=query_embedding,
            query_filter=Filter(
                must=[
                    FieldCondition(key="project_id", match=MatchValue(value=project_id))
                ]
            ) if project_id else None,
            limit=top_k,
            score_threshold=0.7
        )

        return [
            {
                "id": hit.id,
                "content": hit.payload.get("text"),
                "score": hit.score,
                "source": "vector"
            }
            for hit in results
        ]

    async def _graph_rag_retrieve(
        self,
        query: str,
        tenant_id: str,
        project_id: str,
        top_k: int
    ) -> list[dict]:
        """Graph RAG (vector + knowledge graph)."""

        # 1. Vector search
        vector_results = await self._vector_retrieve(
            query, tenant_id, project_id, top_k=10
        )

        # 2. Extract entities from query
        query_entities = await entity_extractor.extract(query)

        # 3. Graph traversal
        graph_results = []
        for entity in query_entities:
            related = await neo4j_client.find_related_entities(
                tenant_id=tenant_id,
                entity_name=entity["name"],
                max_depth=2
            )
            graph_results.extend(related)

        # 4. Combine using Reciprocal Rank Fusion
        combined = self._reciprocal_rank_fusion(
            vector_results,
            graph_results,
            k=60
        )

        return combined[:top_k]
```

---

## Agent Orchestration

### Multi-Agent Coordination

```python
class AgentOrchestrator:
    """Orchestrate multi-agent workflows."""

    async def execute_swarm(
        self,
        task: str,
        tenant_id: str,
        agents: list[Agent]
    ) -> dict:
        """
        Execute task using swarm coordination.

        Pattern: Agents hand off to each other based on capabilities.
        """

        # Start with first agent
        current_agent = agents[0]
        context = {"task": task, "history": []}

        max_handoffs = 5
        handoff_count = 0

        while handoff_count < max_handoffs:
            # Execute current agent
            result = await current_agent.execute(
                task=context["task"],
                context=context
            )

            context["history"].append({
                "agent": current_agent.name,
                "result": result
            })

            # Check if handoff needed
            if result.get("handoff_to"):
                # Find next agent
                next_agent_name = result["handoff_to"]
                current_agent = next(
                    (a for a in agents if a.name == next_agent_name),
                    None
                )

                if not current_agent:
                    break

                handoff_count += 1
            else:
                # Task complete
                break

        return {
            "result": result,
            "handoff_count": handoff_count,
            "agents_used": [h["agent"] for h in context["history"]]
        }

    async def execute_hierarchical(
        self,
        task: str,
        tenant_id: str,
        supervisor_agent: Agent,
        worker_agents: list[Agent]
    ) -> dict:
        """
        Execute task using hierarchical coordination.

        Pattern: Supervisor delegates subtasks to workers in parallel.
        """

        # Supervisor breaks down task
        plan = await supervisor_agent.plan(task)

        # Execute subtasks in parallel
        subtask_results = await asyncio.gather(*[
            worker_agents[i % len(worker_agents)].execute(subtask)
            for i, subtask in enumerate(plan["subtasks"])
        ])

        # Supervisor synthesizes results
        final_result = await supervisor_agent.synthesize(subtask_results)

        return {
            "result": final_result,
            "subtasks": len(plan["subtasks"]),
            "workers_used": len(worker_agents)
        }
```

---

## Memory Management

### 3-Layer Memory System

```python
class MemoryManager:
    """Manage agent memory across 3 layers."""

    def __init__(self):
        self.short_term = RedisMemory()    # Seconds to minutes
        self.long_term = PostgresMemory()  # Permanent
        self.semantic = QdrantMemory()     # Vector-based recall

    async def store(
        self,
        tenant_id: str,
        agent_id: str,
        memory_type: str,
        content: dict,
        ttl: int = None
    ):
        """Store memory in appropriate layer."""

        if memory_type == "conversation":
            # Short-term: Recent conversation window
            await self.short_term.store(
                key=f"agent:{agent_id}:conversation",
                value=content,
                ttl=ttl or 3600  # 1 hour
            )

        elif memory_type == "fact":
            # Long-term: User preferences, facts
            await self.long_term.store(
                tenant_id=tenant_id,
                agent_id=agent_id,
                memory_type="fact",
                content=content
            )

        # Always store in semantic memory for retrieval
        embedding = await embedding_service.embed(str(content))

        await self.semantic.store(
            collection_name=f"memory_{tenant_id}",
            vector=embedding,
            payload={
                "agent_id": agent_id,
                "memory_type": memory_type,
                "content": content,
                "timestamp": datetime.utcnow().isoformat()
            }
        )

    async def recall(
        self,
        tenant_id: str,
        agent_id: str,
        query: str,
        limit: int = 5
    ) -> list[dict]:
        """Recall relevant memories."""

        # Semantic search in memory
        query_embedding = await embedding_service.embed(query)

        results = await qdrant_client.search(
            collection_name=f"memory_{tenant_id}",
            query_vector=query_embedding,
            query_filter=Filter(
                must=[
                    FieldCondition(key="agent_id", match=MatchValue(value=agent_id))
                ]
            ),
            limit=limit
        )

        return [hit.payload["content"] for hit in results]
```

---

## Evaluation Service

### LLM-as-Judge Evaluation

```python
class EvaluationService:
    """Evaluate LLM responses using LLM-as-Judge."""

    async def evaluate_response(
        self,
        query: str,
        response: str,
        context: list[str],
        tenant_id: str
    ) -> dict:
        """
        Evaluate response quality.

        Metrics:
        - Relevance: Does response answer the query?
        - Faithfulness: Is response grounded in context?
        - Completeness: Is response comprehensive?
        """

        eval_prompt = f"""
        Evaluate the following AI response on a scale of 1-5:

        Query: {query}

        Context:
        {chr(10).join(context)}

        Response: {response}

        Rate the response on:
        1. Relevance (1-5): Does it answer the query?
        2. Faithfulness (1-5): Is it grounded in the context?
        3. Completeness (1-5): Is it comprehensive?

        Respond in JSON format:
        {{
            "relevance": <score>,
            "faithfulness": <score>,
            "completeness": <score>,
            "explanation": "<brief explanation>"
        }}
        """

        eval_response = await llm_gateway.complete(
            prompt=eval_prompt,
            tenant_id=tenant_id,
            model="gpt-4o"
        )

        scores = json.loads(eval_response.text)

        return {
            "overall_score": (
                scores["relevance"] +
                scores["faithfulness"] +
                scores["completeness"]
            ) / 3,
            "metrics": scores
        }

    async def run_evaluation_suite(
        self,
        tenant_id: str,
        test_cases: list[dict]
    ) -> dict:
        """Run evaluation on test dataset."""

        results = []

        for test_case in test_cases:
            # Generate response
            response = await chat_service.send_message(
                tenant_id=tenant_id,
                conversation_id=test_case["conversation_id"],
                message=test_case["query"],
                principal_id=1  # System
            )

            # Evaluate
            scores = await self.evaluate_response(
                query=test_case["query"],
                response=response["content"],
                context=response.get("context_sources", []),
                tenant_id=tenant_id
            )

            results.append({
                "test_case_id": test_case["id"],
                "scores": scores
            })

        return {
            "total_tests": len(test_cases),
            "avg_score": sum(r["scores"]["overall_score"] for r in results) / len(results),
            "results": results
        }
```

---

## Embedding Service

### Document Embedding

```python
class EmbeddingService:
    """Generate embeddings for text."""

    async def embed(self, text: str) -> list[float]:
        """Generate embedding for single text."""

        response = await openai_client.embeddings.create(
            model="text-embedding-3-small",
            input=text
        )

        return response.data[0].embedding

    async def embed_batch(
        self,
        texts: list[str],
        batch_size: int = 100
    ) -> list[list[float]]:
        """Generate embeddings for batch of texts."""

        embeddings = []

        for i in range(0, len(texts), batch_size):
            batch = texts[i:i + batch_size]

            response = await openai_client.embeddings.create(
                model="text-embedding-3-small",
                input=batch
            )

            batch_embeddings = [data.embedding for data in response.data]
            embeddings.extend(batch_embeddings)

        return embeddings

    async def embed_document(
        self,
        document_id: str,
        tenant_id: str
    ):
        """Generate and store embeddings for document."""

        # Get document
        document = await db.query(Document).filter_by(
            uid=document_id,
            tenant_id=tenant_id
        ).first()

        # Download from S3
        obj = s3_client.get_object(Bucket=BUCKET_NAME, Key=document.s3_key)
        content = obj['Body'].read().decode('utf-8')

        # Chunk document
        chunks = self._chunk_text(content, chunk_size=500, overlap=50)

        # Generate embeddings
        embeddings = await self.embed_batch(chunks)

        # Store in Qdrant
        points = [
            PointStruct(
                id=f"{document_id}_{i}",
                vector=embedding,
                payload={
                    "document_id": document_id,
                    "tenant_id": tenant_id,
                    "chunk_index": i,
                    "text": chunk
                }
            )
            for i, (chunk, embedding) in enumerate(zip(chunks, embeddings))
        ]

        await qdrant_client.upsert(
            collection_name=f"embeddings_{tenant_id}",
            points=points
        )
```

---

**Next**: [Traditional Services](./03a-traditional-services.md) | [Service Integration](./03c-service-integration.md) | [Back to Overview](../00-parallel-stacks-overview.md)
