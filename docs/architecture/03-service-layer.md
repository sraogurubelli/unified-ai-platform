# Service Layer: Platform + AI Service Convergence

## Overview

The service layer contains all business logic and is organized by **domain**, not by "traditional vs AI". Services are designed to work seamlessly across both SaaS and AI use cases.

```
┌──────────────────────────────────────────────────────────────┐
│                      Service Layer                            │
├──────────────────────────┬───────────────────────────────────┤
│   Platform Services      │      AI Services                  │
│   (SaaS Patterns)        │      (AI-Native Patterns)         │
│                          │                                   │
│   - Auth Service         │      - Agent Orchestration        │
│   - Organization Service │      - RAG Service                │
│   - Project Service      │      - Knowledge Graph Service    │
│   - Webhook Service      │      - Model Management           │
│   - Analytics Service    │      - Fine-tuning Service        │
└──────────────────────────┴───────────────────────────────────┘
```

## Platform Services

### 1. Auth Service

Handles authentication, signup, and token management.

#### Signup Flow

```python
# POST /api/v1/auth/signup

class SignupRequest(BaseModel):
    email: str
    password: str
    organization_name: str
    subscription_tier: SubscriptionTier = SubscriptionTier.FREE

async def signup(request: SignupRequest) -> TokenResponse:
    """
    Create new tenant with:
    - Account (tenant)
    - Organization
    - Default project
    - Principal (owner)
    - RBAC memberships
    """

    # 1. Validate email not already registered
    existing = await db.query(Principal).filter_by(email=request.email).first()
    if existing:
        raise HTTPException(status_code=400, detail="Email already registered")

    # 2. Create principal
    salt = generate_salt()
    password_hash = hash_password(request.password, salt)

    principal = Principal(
        uid=f"user_{uuid.uuid4().hex[:12]}",
        email=request.email,
        display_name=request.email.split("@")[0],
        principal_type=PrincipalType.USER,
        salt=salt
    )
    await db.add(principal)
    await db.flush()

    # 3. Create account (tenant)
    account = Account(
        uid=f"acct_{uuid.uuid4().hex[:12]}",
        name=request.organization_name,
        billing_email=request.email,
        status=AccountStatus.TRIAL,
        subscription_tier=request.subscription_tier,
        owner_id=principal.id,
        trial_ends_at=datetime.utcnow() + timedelta(days=14)
    )
    await db.add(account)
    await db.flush()

    # 4. Create default organization
    org = Organization(
        uid=f"org_{uuid.uuid4().hex[:12]}",
        account_id=account.id,
        name=request.organization_name,
        owner_id=principal.id
    )
    await db.add(org)
    await db.flush()

    # 5. Create default project
    project = Project(
        uid=f"proj_{uuid.uuid4().hex[:12]}",
        organization_id=org.id,
        name="Default Project",
        owner_id=principal.id
    )
    await db.add(project)

    # 6. Create RBAC memberships (owner on all)
    for resource_type, resource_id in [
        ("account", account.uid),
        ("organization", org.uid),
        ("project", project.uid),
    ]:
        membership = Membership(
            principal_id=principal.id,
            resource_type=resource_type,
            resource_id=resource_id,
            role=Role.OWNER
        )
        await db.add(membership)

    await db.commit()

    # 7. Provision tenant resources (async)
    asyncio.create_task(provision_tenant(account))

    # 8. Generate JWT
    token = generate_jwt(principal, account)

    return TokenResponse(
        access_token=token,
        token_type="bearer",
        expires_in=86400
    )
```

### 2. Organization Service

Manages organizations and members.

```python
class OrganizationService:
    """Organization CRUD and member management."""

    async def create(
        self,
        account_id: int,
        name: str,
        owner_id: int
    ) -> Organization:
        """Create new organization."""

        org = Organization(
            uid=f"org_{uuid.uuid4().hex[:12]}",
            account_id=account_id,
            name=name,
            owner_id=owner_id
        )
        await db.add(org)

        # Grant owner membership
        membership = Membership(
            principal_id=owner_id,
            resource_type="organization",
            resource_id=org.uid,
            role=Role.OWNER
        )
        await db.add(membership)

        await db.commit()
        return org

    async def add_member(
        self,
        org_uid: str,
        principal_id: int,
        role: Role
    ) -> Membership:
        """Add member to organization."""

        membership = Membership(
            principal_id=principal_id,
            resource_type="organization",
            resource_id=org_uid,
            role=role
        )
        await db.add(membership)
        await db.commit()

        return membership

    async def list_members(self, org_uid: str) -> list[dict]:
        """List organization members with roles."""

        memberships = await db.query(Membership).join(Principal).filter(
            Membership.resource_type == "organization",
            Membership.resource_id == org_uid
        ).all()

        return [
            {
                "principal_id": m.principal.uid,
                "email": m.principal.email,
                "display_name": m.principal.display_name,
                "role": m.role
            }
            for m in memberships
        ]
```

### 3. Project Service

Manages projects (workspaces).

```python
class ProjectService:
    """Project CRUD and resource management."""

    async def create(
        self,
        organization_id: int,
        name: str,
        owner_id: int
    ) -> Project:
        """Create new project."""

        project = Project(
            uid=f"proj_{uuid.uuid4().hex[:12]}",
            organization_id=organization_id,
            name=name,
            owner_id=owner_id
        )
        await db.add(project)

        # Grant owner membership
        membership = Membership(
            principal_id=owner_id,
            resource_type="project",
            resource_id=project.uid,
            role=Role.OWNER
        )
        await db.add(membership)

        await db.commit()
        return project

    async def get_stats(self, project_id: int) -> dict:
        """Get project statistics."""

        # Count conversations
        conversation_count = await db.query(Conversation).filter_by(
            project_id=project_id
        ).count()

        # Count messages
        message_count = await db.query(Message).join(Conversation).filter(
            Conversation.project_id == project_id
        ).count()

        # Count documents (from vector store)
        document_count = await vector_store.count_documents(
            collection_name=f"project_{project_id}"
        )

        return {
            "conversations": conversation_count,
            "messages": message_count,
            "documents": document_count
        }
```

### 4. Webhook Service

Event-driven integrations.

```python
class WebhookService:
    """Webhook management and delivery."""

    async def create_webhook(
        self,
        tenant_id: str,
        url: str,
        events: list[str],
        secret: str = None
    ) -> Webhook:
        """Register webhook endpoint."""

        webhook = Webhook(
            uid=f"hook_{uuid.uuid4().hex[:12]}",
            tenant_id=tenant_id,
            url=url,
            events=json.dumps(events),
            secret=secret or secrets.token_urlsafe(32),
            active=True
        )
        await db.add(webhook)
        await db.commit()

        return webhook

    async def trigger(
        self,
        tenant_id: str,
        event: str,
        payload: dict
    ):
        """Trigger webhooks for event."""

        # Find active webhooks for this event
        webhooks = await db.query(Webhook).filter(
            Webhook.tenant_id == tenant_id,
            Webhook.active == True
        ).all()

        for webhook in webhooks:
            events = json.loads(webhook.events)
            if event in events or "*" in events:
                await self._deliver_webhook(webhook, event, payload)

    async def _deliver_webhook(
        self,
        webhook: Webhook,
        event: str,
        payload: dict
    ):
        """Deliver webhook via HTTP POST."""

        # Build payload
        webhook_payload = {
            "event": event,
            "timestamp": datetime.utcnow().isoformat(),
            "data": payload
        }

        # Sign payload
        signature = hmac.new(
            webhook.secret.encode(),
            json.dumps(webhook_payload).encode(),
            hashlib.sha256
        ).hexdigest()

        # Send HTTP POST
        try:
            async with httpx.AsyncClient() as client:
                response = await client.post(
                    webhook.url,
                    json=webhook_payload,
                    headers={
                        "X-Webhook-Signature": signature,
                        "X-Webhook-Event": event
                    },
                    timeout=10.0
                )

            if response.status_code >= 200 and response.status_code < 300:
                logger.info("webhook_delivered", webhook_id=webhook.uid, event=event)
            else:
                logger.error(
                    "webhook_failed",
                    webhook_id=webhook.uid,
                    status=response.status_code
                )
        except Exception as e:
            logger.error("webhook_error", webhook_id=webhook.uid, error=str(e))
```

## AI Services

### 1. Agent Orchestration Service

Multi-agent coordination and execution.

#### Single Agent Execution

```python
# cortex/orchestration/agent.py

class Agent:
    """LangGraph-based agent."""

    def __init__(
        self,
        name: str,
        system_prompt: str,
        model: ModelConfig,
        tool_registry: ToolRegistry,
        checkpointer: BaseCheckpointSaver,
        max_iterations: int = 25
    ):
        self.name = name
        self.system_prompt = system_prompt
        self.model = model
        self.tools = tool_registry.get_tools()
        self.checkpointer = checkpointer
        self.max_iterations = max_iterations

    async def run(
        self,
        message: str,
        thread_id: str,
        context: dict = None
    ) -> AgentResult:
        """Execute agent."""

        # Build LangGraph workflow
        graph = self._build_graph()

        # Execute
        config = {
            "configurable": {
                "thread_id": thread_id,
                "checkpoint_ns": f"agent_{self.name}"
            }
        }

        result = await graph.ainvoke(
            {"messages": [HumanMessage(content=message)]},
            config=config
        )

        return AgentResult(
            response=result["messages"][-1].content,
            messages=result["messages"],
            token_usage=result.get("usage", {}),
            tool_calls=self._extract_tool_calls(result["messages"])
        )

    def _build_graph(self) -> CompiledGraph:
        """Build LangGraph workflow."""

        # Create model with tools
        model_with_tools = self.model.bind_tools(self.tools)

        # Define agent node
        async def agent_node(state: AgentState):
            messages = state["messages"]
            response = await model_with_tools.ainvoke(messages)
            return {"messages": messages + [response]}

        # Define tool node
        async def tool_node(state: AgentState):
            messages = state["messages"]
            last_message = messages[-1]

            # Execute tools
            tool_results = []
            for tool_call in last_message.tool_calls:
                tool = self._get_tool(tool_call["name"])
                result = await tool.ainvoke(tool_call["args"])
                tool_results.append(
                    ToolMessage(
                        content=result,
                        tool_call_id=tool_call["id"]
                    )
                )

            return {"messages": messages + tool_results}

        # Build graph
        workflow = StateGraph(AgentState)
        workflow.add_node("agent", agent_node)
        workflow.add_node("tools", tool_node)

        workflow.set_entry_point("agent")
        workflow.add_conditional_edges(
            "agent",
            self._should_continue,
            {
                "continue": "tools",
                "end": END
            }
        )
        workflow.add_edge("tools", "agent")

        # Compile with checkpointer
        return workflow.compile(checkpointer=self.checkpointer)

    def _should_continue(self, state: AgentState) -> str:
        """Decide whether to continue or end."""
        messages = state["messages"]
        last_message = messages[-1]

        if last_message.tool_calls:
            return "continue"
        return "end"
```

#### Multi-Agent Swarm

```python
# cortex/orchestration/swarm.py

class SwarmOrchestrator:
    """Coordinate multiple agents."""

    def __init__(self, agents: list[Agent]):
        self.agents = {agent.name: agent for agent in agents}

    async def run(
        self,
        message: str,
        entry_agent: str,
        thread_id: str
    ) -> SwarmResult:
        """Execute swarm with agent handoffs."""

        current_agent = entry_agent
        conversation_history = []
        handoffs = []

        for iteration in range(10):  # Max 10 handoffs
            agent = self.agents[current_agent]

            # Execute current agent
            result = await agent.run(
                message=message,
                thread_id=thread_id,
                context={"history": conversation_history}
            )

            conversation_history.extend(result.messages)

            # Check for handoff
            handoff = self._extract_handoff(result.messages)
            if handoff:
                handoffs.append({
                    "from": current_agent,
                    "to": handoff["target_agent"],
                    "reason": handoff["reason"]
                })
                current_agent = handoff["target_agent"]
                message = handoff["context"]
            else:
                # No more handoffs, return result
                return SwarmResult(
                    response=result.response,
                    messages=conversation_history,
                    handoffs=handoffs,
                    final_agent=current_agent
                )

        raise RuntimeError("Max handoffs exceeded")

    def _extract_handoff(self, messages: list) -> dict:
        """Extract handoff intent from messages."""

        last_message = messages[-1]

        # Check for handoff tool call
        for tool_call in last_message.tool_calls:
            if tool_call["name"] == "handoff_to_agent":
                return {
                    "target_agent": tool_call["args"]["agent_name"],
                    "reason": tool_call["args"]["reason"],
                    "context": tool_call["args"]["context"]
                }

        return None
```

### 2. RAG Service

Document processing and retrieval.

```python
class RAGService:
    """Retrieval-Augmented Generation service."""

    def __init__(
        self,
        vector_store: VectorStore,
        embedding_provider: EmbeddingProvider
    ):
        self.vector_store = vector_store
        self.embedding_provider = embedding_provider

    async def ingest_document(
        self,
        tenant_id: str,
        project_id: str,
        document: Document
    ):
        """Ingest document into vector store."""

        # 1. Chunk document
        chunks = self._chunk_document(document.content)

        # 2. Generate embeddings
        embeddings = await self.embedding_provider.embed_batch(
            texts=[chunk.text for chunk in chunks]
        )

        # 3. Store in vector database
        points = [
            {
                "id": str(uuid.uuid4()),
                "vector": embedding,
                "payload": {
                    "tenant_id": tenant_id,
                    "project_id": project_id,
                    "document_id": document.id,
                    "chunk_index": i,
                    "text": chunk.text,
                    "metadata": document.metadata
                }
            }
            for i, (chunk, embedding) in enumerate(zip(chunks, embeddings))
        ]

        await self.vector_store.upsert(
            collection_name=f"embeddings_{tenant_id}",
            points=points
        )

    def _chunk_document(
        self,
        content: str,
        chunk_size: int = 512,
        overlap: int = 50
    ) -> list[Chunk]:
        """Split document into overlapping chunks."""

        # Simple sentence-based chunking
        sentences = content.split(". ")
        chunks = []
        current_chunk = []
        current_size = 0

        for sentence in sentences:
            sentence_size = len(sentence.split())

            if current_size + sentence_size > chunk_size and current_chunk:
                # Save current chunk
                chunks.append(Chunk(text=". ".join(current_chunk)))

                # Start new chunk with overlap
                overlap_sentences = current_chunk[-overlap:] if overlap > 0 else []
                current_chunk = overlap_sentences + [sentence]
                current_size = sum(len(s.split()) for s in current_chunk)
            else:
                current_chunk.append(sentence)
                current_size += sentence_size

        if current_chunk:
            chunks.append(Chunk(text=". ".join(current_chunk)))

        return chunks

    async def retrieve(
        self,
        tenant_id: str,
        project_id: str,
        query: str,
        top_k: int = 5
    ) -> list[RetrievedChunk]:
        """Retrieve relevant chunks for query."""

        # 1. Generate query embedding
        query_embedding = await self.embedding_provider.embed(query)

        # 2. Search vector store
        results = await self.vector_store.search(
            collection_name=f"embeddings_{tenant_id}",
            query_vector=query_embedding,
            filters={
                "tenant_id": tenant_id,
                "project_id": project_id
            },
            limit=top_k
        )

        # 3. Return results
        return [
            RetrievedChunk(
                text=result.payload["text"],
                score=result.score,
                document_id=result.payload["document_id"],
                metadata=result.payload["metadata"]
            )
            for result in results
        ]

    async def generate_answer(
        self,
        query: str,
        context_chunks: list[RetrievedChunk],
        model: ModelConfig
    ) -> str:
        """Generate answer using RAG."""

        # Build context
        context = "\n\n".join([
            f"[Document {i+1}]\n{chunk.text}"
            for i, chunk in enumerate(context_chunks)
        ])

        # Build prompt
        prompt = f"""
Answer the question based on the context below.

Context:
{context}

Question: {query}

Answer:
"""

        # Call LLM
        llm = model.get_llm()
        response = await llm.ainvoke(prompt)

        return response.content
```

### 3. Knowledge Graph Service

Entity extraction and graph queries.

```python
class KnowledgeGraphService:
    """Knowledge graph management."""

    def __init__(self, neo4j_driver: Neo4jDriver):
        self.neo4j = neo4j_driver

    async def extract_entities(
        self,
        tenant_id: str,
        document_id: str,
        content: str
    ):
        """Extract entities and relationships from document."""

        # Use LLM to extract entities
        extraction_prompt = f"""
Extract entities and relationships from the text below.
Return JSON format:
{{
    "entities": [
        {{"type": "Person", "name": "Alice", "attributes": {{}}}},
        {{"type": "Company", "name": "Acme Corp", "attributes": {{}}}}
    ],
    "relationships": [
        {{"from": "Alice", "to": "Acme Corp", "type": "WORKS_AT"}}
    ]
}}

Text:
{content}
"""

        llm = get_llm("gpt-4o")
        response = await llm.ainvoke(extraction_prompt)

        # Parse extraction result
        extraction = json.loads(response.content)

        # Create nodes and relationships in Neo4j
        for entity in extraction["entities"]:
            await self._create_entity(
                tenant_id=tenant_id,
                entity_type=entity["type"],
                name=entity["name"],
                attributes=entity["attributes"]
            )

        for rel in extraction["relationships"]:
            await self._create_relationship(
                tenant_id=tenant_id,
                from_name=rel["from"],
                to_name=rel["to"],
                rel_type=rel["type"]
            )

    async def _create_entity(
        self,
        tenant_id: str,
        entity_type: str,
        name: str,
        attributes: dict
    ):
        """Create entity node."""

        query = f"""
        MERGE (e:{entity_type} {{tenant_id: $tenant_id, name: $name}})
        SET e += $attributes
        RETURN e
        """

        await self.neo4j.execute_query(
            query,
            tenant_id=tenant_id,
            name=name,
            attributes=attributes
        )

    async def _create_relationship(
        self,
        tenant_id: str,
        from_name: str,
        to_name: str,
        rel_type: str
    ):
        """Create relationship between entities."""

        query = f"""
        MATCH (a {{tenant_id: $tenant_id, name: $from_name}})
        MATCH (b {{tenant_id: $tenant_id, name: $to_name}})
        MERGE (a)-[r:{rel_type}]->(b)
        RETURN r
        """

        await self.neo4j.execute_query(
            query,
            tenant_id=tenant_id,
            from_name=from_name,
            to_name=to_name
        )

    async def query_graph(
        self,
        tenant_id: str,
        cypher: str,
        params: dict = None
    ) -> list[dict]:
        """Execute Cypher query with tenant filter."""

        # Inject tenant filter
        cypher_with_tenant = f"""
        MATCH (n)
        WHERE n.tenant_id = $tenant_id
        {cypher}
        """

        results = await self.neo4j.execute_query(
            cypher_with_tenant,
            tenant_id=tenant_id,
            **(params or {})
        )

        return results
```

### 4. Model Management Service

Multi-provider LLM management.

```python
class ModelManager:
    """Manage LLM providers and routing."""

    def __init__(self):
        self.providers = {
            "openai": OpenAIProvider(),
            "anthropic": AnthropicProvider(),
            "google": GoogleProvider()
        }

    async def get_completion(
        self,
        model: str,
        messages: list[dict],
        **kwargs
    ) -> CompletionResult:
        """Route completion request to appropriate provider."""

        # Parse model string
        provider_name, model_name = self._parse_model(model)

        # Get provider
        provider = self.providers.get(provider_name)
        if not provider:
            raise ValueError(f"Unknown provider: {provider_name}")

        # Call provider with fallback
        try:
            result = await provider.chat_completion(
                model=model_name,
                messages=messages,
                **kwargs
            )
            return result
        except Exception as e:
            logger.error(f"Provider {provider_name} failed: {e}")

            # Fallback to alternative provider
            fallback_provider = self._get_fallback_provider(provider_name)
            if fallback_provider:
                logger.info(f"Falling back to {fallback_provider}")
                return await self.get_completion(
                    model=fallback_provider,
                    messages=messages,
                    **kwargs
                )
            else:
                raise

    def _parse_model(self, model: str) -> tuple[str, str]:
        """Parse model string into provider and model name."""

        # Format: "provider/model" or just "model"
        if "/" in model:
            provider, model_name = model.split("/", 1)
        else:
            # Infer provider from model name
            if model.startswith("gpt"):
                provider = "openai"
            elif model.startswith("claude"):
                provider = "anthropic"
            elif model.startswith("gemini"):
                provider = "google"
            else:
                provider = "openai"  # default

            model_name = model

        return provider, model_name

    def _get_fallback_provider(self, primary: str) -> str:
        """Get fallback provider."""

        fallbacks = {
            "openai": "anthropic/claude-sonnet-4",
            "anthropic": "openai/gpt-4o",
            "google": "openai/gpt-4o"
        }

        return fallbacks.get(primary)
```

## Event-Driven Integration

### Overview

The platform uses **event-driven architecture** for service communication, enabling loose coupling, scalability, and asynchronous processing. Services publish events when state changes, and other services subscribe to react to those changes.

**Benefits**:
- **Decoupling**: Services don't need to know about each other
- **Scalability**: Multiple consumers can process events in parallel
- **Resilience**: Failed consumers can replay events
- **Audit Trail**: All state changes captured as events

### Event Streaming Technologies

| Technology | Use Case | Latency | Throughput | Persistence |
|------------|----------|---------|------------|-------------|
| **Kafka** | High-volume event streaming | 5-10ms | 1M+ msg/sec | Days to weeks |
| **Redis Streams** | Real-time events, session data | 1-3ms | 100K+ msg/sec | Hours to days |
| **PostgreSQL** | ACID event sourcing | 10-50ms | 10K msg/sec | Permanent |

**Recommended Stack**:
- **Kafka**: Enterprise deployments (high volume, strict ordering)
- **Redis Streams**: Startup/scale-up (lower complexity, good performance)
- **PostgreSQL**: Event sourcing with strong consistency guarantees

### Event Types

```python
class EventType(str, Enum):
    # Resource lifecycle events
    PROJECT_CREATED = "project.created"
    PROJECT_UPDATED = "project.updated"
    PROJECT_DELETED = "project.deleted"

    # AI service events
    CONVERSATION_CREATED = "conversation.created"
    MESSAGE_SENT = "message.sent"
    LLM_CALL_COMPLETED = "llm.call.completed"

    # Analytics events
    USAGE_RECORDED = "usage.recorded"
    QUOTA_EXCEEDED = "quota.exceeded"

    # Knowledge graph events
    ENTITY_EXTRACTED = "entity.extracted"
    RELATIONSHIP_CREATED = "relationship.created"
```

### Event Schema

**Standard Event Envelope**:

```python
from pydantic import BaseModel
from datetime import datetime

class Event(BaseModel):
    """Standard event envelope."""

    event_id: str  # Unique event ID (UUID)
    event_type: str  # Event type (e.g., "conversation.created")
    timestamp: datetime  # When event occurred
    tenant_id: str  # Tenant context
    actor_id: str  # Who triggered the event (user/service)
    resource_type: str  # Type of resource (e.g., "conversation")
    resource_id: str  # Resource identifier
    payload: dict  # Event-specific data
    metadata: dict = {}  # Optional metadata (trace_id, span_id, etc.)
```

**Example Event**:

```json
{
  "event_id": "evt_abc123",
  "event_type": "conversation.created",
  "timestamp": "2024-01-15T10:30:00Z",
  "tenant_id": "acct_xyz789",
  "actor_id": "user_456",
  "resource_type": "conversation",
  "resource_id": "conv_789",
  "payload": {
    "project_id": "proj_123",
    "title": "AI Platform Discussion",
    "model": "gpt-4o"
  },
  "metadata": {
    "trace_id": "trace-abc",
    "source": "chat-service"
  }
}
```

### Event Bus Implementation

#### Option 1: Redis Streams (Recommended for Startups)

```python
import redis.asyncio as redis
import json
from typing import AsyncGenerator

class RedisEventBus:
    """Event bus using Redis Streams."""

    def __init__(self, redis_url: str):
        self.redis = redis.from_url(redis_url)

    async def publish(self, event: Event) -> str:
        """
        Publish event to Redis stream.

        Returns:
            str: Message ID
        """
        stream_key = f"events:{event.event_type}"

        message_id = await self.redis.xadd(
            stream_key,
            {
                "event_id": event.event_id,
                "data": event.json()
            }
        )

        # Also publish to global event stream for analytics
        await self.redis.xadd(
            "events:all",
            {"data": event.json()}
        )

        return message_id

    async def subscribe(
        self,
        event_type: str,
        consumer_group: str,
        consumer_name: str
    ) -> AsyncGenerator[Event, None]:
        """
        Subscribe to events.

        Args:
            event_type: Type of events to subscribe to
            consumer_group: Consumer group name (for load balancing)
            consumer_name: Unique consumer identifier
        """
        stream_key = f"events:{event_type}"

        # Create consumer group (idempotent)
        try:
            await self.redis.xgroup_create(
                stream_key,
                consumer_group,
                id="0",
                mkstream=True
            )
        except redis.ResponseError:
            pass  # Group already exists

        # Read events
        while True:
            # Read new events (block for 1 second if no events)
            events = await self.redis.xreadgroup(
                consumer_group,
                consumer_name,
                {stream_key: ">"},
                count=10,
                block=1000
            )

            for stream, messages in events:
                for message_id, data in messages:
                    event = Event.parse_raw(data[b"data"])
                    yield event

                    # Acknowledge processing
                    await self.redis.xack(stream, consumer_group, message_id)
```

#### Option 2: Kafka (Recommended for Enterprise)

```python
from aiokafka import AIOKafkaProducer, AIOKafkaConsumer
import json

class KafkaEventBus:
    """Event bus using Apache Kafka."""

    def __init__(self, bootstrap_servers: list[str]):
        self.bootstrap_servers = bootstrap_servers
        self.producer = None
        self.consumers = {}

    async def start(self):
        """Initialize Kafka producer."""
        self.producer = AIOKafkaProducer(
            bootstrap_servers=self.bootstrap_servers,
            value_serializer=lambda v: json.dumps(v).encode()
        )
        await self.producer.start()

    async def publish(self, event: Event):
        """
        Publish event to Kafka topic.

        Topic naming: events.{event_type}
        e.g., "events.conversation.created"
        """
        topic = f"events.{event.event_type}"

        await self.producer.send_and_wait(
            topic,
            value=event.dict(),
            key=event.tenant_id.encode()  # Partition by tenant
        )

    async def subscribe(
        self,
        event_type: str,
        consumer_group: str
    ) -> AsyncGenerator[Event, None]:
        """Subscribe to events from Kafka topic."""

        topic = f"events.{event_type}"

        consumer = AIOKafkaConsumer(
            topic,
            bootstrap_servers=self.bootstrap_servers,
            group_id=consumer_group,
            value_deserializer=lambda m: json.loads(m.decode()),
            enable_auto_commit=False  # Manual commit for reliability
        )

        await consumer.start()

        try:
            async for message in consumer:
                event = Event(**message.value)
                yield event

                # Commit offset after successful processing
                await consumer.commit()
        finally:
            await consumer.stop()
```

### Service Integration Patterns

#### Pattern 1: Analytics Pipeline

**LLM Gateway publishes usage events → ClickHouse consumes for analytics**

```python
# LLM Gateway (Publisher)
class LLMGateway:
    async def generate(self, prompt: str, model: str, tenant_id: str):
        response = await llm_provider.generate(prompt, model)

        # Publish usage event
        await event_bus.publish(Event(
            event_id=str(uuid.uuid4()),
            event_type="llm.call.completed",
            timestamp=datetime.utcnow(),
            tenant_id=tenant_id,
            actor_id=current_principal.uid,
            resource_type="llm_call",
            resource_id=str(uuid.uuid4()),
            payload={
                "model": model,
                "prompt_tokens": response.usage.prompt_tokens,
                "completion_tokens": response.usage.completion_tokens,
                "cost_usd": calculate_cost(response.usage, model)
            }
        ))

        return response

# Analytics Service (Consumer)
class AnalyticsConsumer:
    async def process_llm_events(self):
        """Consume LLM events and write to ClickHouse."""

        async for event in event_bus.subscribe(
            "llm.call.completed",
            consumer_group="analytics"
        ):
            await clickhouse.execute("""
                INSERT INTO llm_usage (
                    timestamp, tenant_id, model,
                    prompt_tokens, completion_tokens, cost_usd
                ) VALUES
            """, [(
                event.timestamp,
                event.tenant_id,
                event.payload["model"],
                event.payload["prompt_tokens"],
                event.payload["completion_tokens"],
                event.payload["cost_usd"]
            )])
```

#### Pattern 2: Knowledge Graph Enrichment

**Chat Service publishes conversation events → Knowledge Graph extracts entities**

```python
# Chat Service (Publisher)
async def create_message(conversation_id: str, content: str):
    message = Message(conversation_id=conversation_id, content=content)
    await db.add(message)

    # Publish event for knowledge graph processing
    await event_bus.publish(Event(
        event_type="message.sent",
        tenant_id=tenant_id,
        resource_type="message",
        resource_id=message.uid,
        payload={
            "conversation_id": conversation_id,
            "content": content
        }
    ))

# Knowledge Graph Service (Consumer)
class KnowledgeGraphConsumer:
    async def process_messages(self):
        """Extract entities from new messages."""

        async for event in event_bus.subscribe(
            "message.sent",
            consumer_group="knowledge-graph"
        ):
            content = event.payload["content"]

            # Extract entities using NER
            entities = await entity_extractor.extract(content)

            for entity in entities:
                await neo4j.run("""
                    MERGE (e:Entity {name: $name, type: $type})
                    MERGE (d:Document {id: $message_id})
                    MERGE (d)-[:MENTIONS {confidence: $confidence}]->(e)
                """, name=entity.name, type=entity.type,
                    message_id=event.resource_id,
                    confidence=entity.confidence)
```

#### Pattern 3: Rate Limiting Updates

**Usage Service publishes quota events → LLM Gateway updates rate limits**

```python
# Usage Service (Publisher)
async def record_usage(tenant_id: str, quota_type: str, amount: int):
    # Record in database
    await db.add(Usage(...))

    # Check if quota exceeded
    total = await db.query(Usage).filter(
        Usage.tenant_id == tenant_id,
        Usage.quota_type == quota_type,
        Usage.period == current_month()
    ).sum(Usage.amount)

    tier_limit = get_tier_limit(tenant_id, quota_type)

    if total > tier_limit:
        # Publish quota exceeded event
        await event_bus.publish(Event(
            event_type="quota.exceeded",
            tenant_id=tenant_id,
            resource_type="quota",
            resource_id=quota_type,
            payload={
                "quota_type": quota_type,
                "limit": tier_limit,
                "current": total
            }
        ))

# LLM Gateway (Consumer)
class LLMGatewayConsumer:
    async def process_quota_events(self):
        """Update rate limits when quotas exceeded."""

        async for event in event_bus.subscribe(
            "quota.exceeded",
            consumer_group="llm-gateway"
        ):
            tenant_id = event.tenant_id
            quota_type = event.payload["quota_type"]

            # Update Redis rate limiter
            await redis.set(
                f"rate_limit:{tenant_id}:{quota_type}",
                "exceeded",
                ex=3600  # Block for 1 hour
            )
```

### Data Consistency Patterns

#### Eventual Consistency

Most events use **eventual consistency** - subscribers process events asynchronously:

```python
# Order of operations:
# 1. Database write (source of truth)
# 2. Event published
# 3. Consumers update derived state

async def create_project(project: Project):
    # 1. Write to PostgreSQL
    await db.add(project)
    await db.commit()

    # 2. Publish event
    await event_bus.publish(Event(event_type="project.created", ...))

    # 3. Consumers eventually update:
    #    - Search index (Qdrant)
    #    - Analytics (ClickHouse)
    #    - Cache (Redis)
```

#### Strong Consistency (Event Sourcing)

For critical workflows, use **event sourcing** with PostgreSQL:

```python
# All state changes stored as events
class EventStore:
    async def append(self, event: Event):
        """Append event to event log."""

        await db.execute("""
            INSERT INTO event_log (
                event_id, event_type, aggregate_id,
                data, timestamp
            ) VALUES ($1, $2, $3, $4, $5)
        """, event.event_id, event.event_type,
             event.resource_id, event.json(),
             event.timestamp)

    async def get_events(self, aggregate_id: str) -> list[Event]:
        """Get all events for aggregate."""

        rows = await db.fetch("""
            SELECT data FROM event_log
            WHERE aggregate_id = $1
            ORDER BY timestamp ASC
        """, aggregate_id)

        return [Event.parse_raw(row["data"]) for row in rows]

# Rebuild state from events
async def get_conversation_state(conversation_id: str):
    events = await event_store.get_events(conversation_id)

    conversation = Conversation(id=conversation_id)
    for event in events:
        conversation.apply(event)  # Apply event to update state

    return conversation
```

### Monitoring & Observability

**Event Metrics**:

```python
from prometheus_client import Counter, Histogram

events_published = Counter(
    "events_published_total",
    "Total events published",
    ["event_type", "tenant_id"]
)

event_processing_time = Histogram(
    "event_processing_seconds",
    "Time to process event",
    ["event_type", "consumer_group"]
)

# Track metrics
events_published.labels(
    event_type="conversation.created",
    tenant_id=tenant_id
).inc()

with event_processing_time.labels(
    event_type="conversation.created",
    consumer_group="analytics"
).time():
    await process_event(event)
```

**Event Tracing**:

```python
from opentelemetry import trace

tracer = trace.get_tracer(__name__)

# Propagate trace context via event metadata
with tracer.start_as_current_span("publish_event") as span:
    event.metadata["trace_id"] = format(span.get_span_context().trace_id, '032x')
    event.metadata["span_id"] = format(span.get_span_context().span_id, '016x')

    await event_bus.publish(event)

# Consumer creates child span
with tracer.start_as_current_span(
    "process_event",
    context=trace_context_from_event(event)  # Parent span
) as span:
    await process_event(event)
```

### Comparison: Synchronous vs Event-Driven

| Aspect | Synchronous (HTTP/gRPC) | Event-Driven (Kafka/Redis) |
|--------|------------------------|----------------------------|
| **Coupling** | Tight (caller knows callee) | Loose (publisher doesn't know consumers) |
| **Latency** | Low (5-50ms) | Higher (50-500ms) |
| **Reliability** | Fails if service down | Resilient (replay events) |
| **Scalability** | Limited (point-to-point) | High (many consumers) |
| **Use Case** | Real-time requests (get user profile) | Async workflows (send email, update analytics) |

**When to Use Each**:

✅ **Synchronous**: User-facing requests, critical path operations
✅ **Event-Driven**: Analytics, notifications, cross-service state sync, audit logging

---

**Next**: [Data Plane](./04-data-plane.md) | [Previous](./02-control-plane.md) | [Back to Overview](./00-convergence-overview.md)
