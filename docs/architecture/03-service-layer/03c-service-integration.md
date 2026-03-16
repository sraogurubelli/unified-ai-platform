# Service Layer Integration

## Overview

The **Service Integration** layer connects Traditional and AI services through event-driven patterns, shared data flows, and API composition.

```
┌──────────────────────────────────────────────────────────────┐
│              Traditional Services                             │
│   Auth · Projects · Documents · Webhooks · Analytics         │
└────────────────────────┬─────────────────────────────────────┘
                         │
            Integration Points:
            • Event Bus (Kafka)
            • Shared Database
            • API Composition
            • Async Processing
                         │
┌────────────────────────▼─────────────────────────────────────┐
│                   AI Services                                 │
│   LLM Gateway · RAG · Agents · Memory · Evaluation           │
└──────────────────────────────────────────────────────────────┘
```

---

## Event-Driven Integration

### Document Upload → Embedding Generation

**Flow**: Traditional document service → AI embedding service

```python
# Traditional Service: Document upload
class DocumentService:
    async def upload_document(self, file: UploadFile, tenant_id: str):
        """Upload document and trigger embedding generation."""

        # Upload to S3 (Traditional)
        s3_key = await self._upload_to_s3(file, tenant_id)

        # Store metadata (Traditional)
        document = await db.insert(Document, {
            "uid": f"doc_{uuid.uuid4().hex[:12]}",
            "tenant_id": tenant_id,
            "s3_key": s3_key,
            "filename": file.filename
        })

        # Publish event for AI processing
        await event_bus.publish("document.uploaded", {
            "document_id": document.uid,
            "tenant_id": tenant_id,
            "s3_key": s3_key
        })

        return document

# AI Service: Embedding generation consumer
class EmbeddingConsumer:
    async def consume_document_events(self):
        """Listen for document uploads and generate embeddings."""

        consumer = KafkaConsumer(
            'document.uploaded',
            bootstrap_servers='kafka:9092',
            group_id='embedding-pipeline'
        )

        for message in consumer:
            event = json.loads(message.value)

            # Download document
            content = await self._download_from_s3(event["s3_key"])

            # Chunk text
            chunks = self._chunk_text(content)

            # Generate embeddings
            embeddings = await embedding_service.embed_batch(chunks)

            # Store in Qdrant
            await qdrant_client.upsert(
                collection_name=f"embeddings_{event['tenant_id']}",
                points=[
                    PointStruct(
                        id=f"{event['document_id']}_{i}",
                        vector=emb,
                        payload={"text": chunk, "document_id": event["document_id"]}
                    )
                    for i, (chunk, emb) in enumerate(zip(chunks, embeddings))
                ]
            )

            logger.info(
                "document_embedded",
                document_id=event["document_id"],
                chunks=len(chunks)
            )
```

### Conversation Created → Knowledge Graph Update

**Flow**: AI chat service → AI knowledge graph service

```python
# AI Service: Chat creates conversation
class ChatService:
    async def send_message(self, conversation_id: str, message: str, tenant_id: str):
        """Send message and trigger knowledge graph extraction."""

        # Store message
        await db.insert(Message, {
            "conversation_id": conversation_id,
            "content": message,
            "role": "user"
        })

        # Get AI response
        response = await llm_gateway.complete(message, tenant_id)

        # Store response
        await db.insert(Message, {
            "conversation_id": conversation_id,
            "content": response.text,
            "role": "assistant"
        })

        # Trigger entity extraction
        await event_bus.publish("message.created", {
            "conversation_id": conversation_id,
            "tenant_id": tenant_id,
            "content": message,
            "response": response.text
        })

        return response

# AI Service: Entity extraction consumer
class EntityExtractionConsumer:
    async def consume_message_events(self):
        """Extract entities and update knowledge graph."""

        consumer = KafkaConsumer(
            'message.created',
            bootstrap_servers='kafka:9092',
            group_id='entity-extraction-pipeline'
        )

        for message in consumer:
            event = json.loads(message.value)

            # Extract entities using LLM
            entities = await entity_extractor.extract(
                text=event["content"],
                tenant_id=event["tenant_id"]
            )

            # Update knowledge graph
            for entity in entities:
                await neo4j_client.create_entity(
                    tenant_id=event["tenant_id"],
                    entity_type=entity["type"],
                    name=entity["name"],
                    attributes=entity.get("attributes", {})
                )

            # Create relationships
            for relationship in entities.get("relationships", []):
                await neo4j_client.create_relationship(
                    tenant_id=event["tenant_id"],
                    from_entity=relationship["from"],
                    to_entity=relationship["to"],
                    relationship_type=relationship["type"]
                )

            logger.info(
                "entities_extracted",
                conversation_id=event["conversation_id"],
                entity_count=len(entities)
            )
```

---

## API Composition

### Chat with Document Context (Hybrid Service)

**Combines**: Traditional document service + AI RAG service

```python
class HybridChatService:
    """Chat service that combines traditional and AI services."""

    async def chat_with_documents(
        self,
        tenant_id: str,
        project_id: str,
        message: str,
        document_ids: list[str] = None
    ) -> dict:
        """
        Chat with document context.

        Flow:
        1. Get documents (Traditional)
        2. Retrieve relevant chunks (AI RAG)
        3. Generate response (AI LLM)
        """

        # 1. Get documents from project (Traditional Service)
        if document_ids:
            documents = await document_service.get_documents(
                tenant_id=tenant_id,
                document_ids=document_ids
            )
        else:
            documents = await document_service.list_documents(
                tenant_id=tenant_id,
                project_id=project_id
            )

        # 2. Retrieve relevant context (AI RAG Service)
        context = await rag_service.retrieve(
            query=message,
            tenant_id=tenant_id,
            document_ids=[d.uid for d in documents],
            top_k=5
        )

        # 3. Generate response with context (AI LLM Service)
        prompt = self._build_prompt(message, context)
        response = await llm_gateway.complete(
            prompt=prompt,
            tenant_id=tenant_id
        )

        # 4. Store conversation (Traditional Service)
        await db.insert(Message, {
            "tenant_id": tenant_id,
            "content": message,
            "role": "user"
        })

        await db.insert(Message, {
            "tenant_id": tenant_id,
            "content": response.text,
            "role": "assistant",
            "metadata": json.dumps({
                "context_sources": [c["id"] for c in context],
                "cached": response.cached
            })
        })

        return {
            "response": response.text,
            "sources": [
                {
                    "document_id": c["document_id"],
                    "document_name": next(
                        (d.filename for d in documents if d.uid == c["document_id"]),
                        "Unknown"
                    ),
                    "relevance_score": c["score"]
                }
                for c in context
            ],
            "cached": response.cached
        }
```

### Project Analytics with AI Insights

**Combines**: Traditional analytics + AI summarization

```python
class HybridAnalyticsService:
    """Analytics service enhanced with AI insights."""

    async def get_project_insights(
        self,
        tenant_id: str,
        project_id: str,
        date_range: str = "30d"
    ) -> dict:
        """
        Get project analytics with AI-generated insights.

        Flow:
        1. Get raw metrics (Traditional Analytics)
        2. Generate insights (AI LLM)
        """

        # 1. Get traditional metrics
        metrics = await analytics_service.get_project_stats(
            tenant_id=tenant_id,
            project_id=project_id,
            date_range=date_range
        )

        # 2. Generate AI insights
        metrics_summary = json.dumps(metrics, indent=2)

        insights_prompt = f"""
        Analyze the following project metrics and provide insights:

        {metrics_summary}

        Provide:
        1. Key trends (2-3 bullet points)
        2. Anomalies or concerns (if any)
        3. Recommendations (2-3 actionable items)

        Format as JSON:
        {{
            "trends": ["...", "..."],
            "concerns": ["..."],
            "recommendations": ["...", "..."]
        }}
        """

        insights_response = await llm_gateway.complete(
            prompt=insights_prompt,
            tenant_id=tenant_id,
            model="gpt-4o"
        )

        insights = json.loads(insights_response.text)

        return {
            "metrics": metrics,
            "ai_insights": insights,
            "generated_at": datetime.utcnow().isoformat()
        }
```

---

## Shared Data Flows

### User Activity → Memory + Analytics

**Shared by**: Traditional analytics + AI memory management

```python
class ActivityTracker:
    """Track user activity for both analytics and AI memory."""

    async def track_activity(
        self,
        tenant_id: str,
        principal_id: int,
        activity_type: str,
        metadata: dict
    ):
        """
        Record activity for both analytics and AI memory.

        Destinations:
        - Traditional: ClickHouse (analytics)
        - AI: Memory system (agent context)
        """

        activity = {
            "timestamp": datetime.utcnow(),
            "tenant_id": tenant_id,
            "principal_id": principal_id,
            "activity_type": activity_type,
            "metadata": metadata
        }

        # Store in analytics database (Traditional)
        await clickhouse.insert("user_activities", [activity])

        # Store in AI memory (for agent personalization)
        if activity_type in ["document_viewed", "search_query", "conversation_topic"]:
            await memory_manager.store(
                tenant_id=tenant_id,
                agent_id="user_assistant",
                memory_type="user_preference",
                content={
                    "activity": activity_type,
                    "details": metadata
                }
            )

        # Publish event for other consumers
        await event_bus.publish("user.activity", activity)
```

---

## Async Processing Pipelines

### Document Processing Pipeline

**Multi-stage pipeline** across Traditional and AI services

```python
class DocumentProcessingPipeline:
    """
    Complete document processing pipeline.

    Stages:
    1. Upload (Traditional: S3)
    2. Extract text (Traditional: OCR/parsing)
    3. Generate embeddings (AI: Qdrant)
    4. Extract entities (AI: Neo4j)
    5. Update search index (Traditional: Elasticsearch)
    """

    async def process_document(self, document_id: str, tenant_id: str):
        """Execute complete pipeline."""

        # Stage 1: Get document
        document = await db.query(Document).filter_by(
            uid=document_id,
            tenant_id=tenant_id
        ).first()

        # Stage 2: Extract text
        content = await self._extract_text(document.s3_key)

        # Stage 3: Generate embeddings (AI)
        await embedding_service.embed_document(document_id, tenant_id)

        # Stage 4: Extract entities (AI)
        entities = await entity_extractor.extract(content, tenant_id)
        for entity in entities:
            await neo4j_client.create_entity(tenant_id, **entity)

        # Stage 5: Update search index (Traditional)
        await elasticsearch.index("documents", {
            "id": document_id,
            "tenant_id": tenant_id,
            "content": content,
            "filename": document.filename
        })

        # Update document status
        document.processing_status = "completed"
        await db.commit()

        # Notify user (Traditional)
        await notification_service.send_email(
            to_email=document.uploaded_by_email,
            subject="Document processing complete",
            body=f"Document '{document.filename}' is ready for search."
        )
```

---

## Cross-Service API Calls

### AI Service → Traditional Service

**Use Case**: Agent needs to create project

```python
class ProjectCreationAgent:
    """AI agent that can create projects via Traditional API."""

    async def execute(self, task: str, tenant_id: str):
        """Execute project creation task."""

        # Parse intent
        intent = await llm_gateway.complete(
            prompt=f"""
            Extract project details from: "{task}"

            Return JSON:
            {{
                "name": "project name",
                "description": "description"
            }}
            """,
            tenant_id=tenant_id
        )

        project_details = json.loads(intent.text)

        # Call Traditional API to create project
        async with httpx.AsyncClient() as client:
            response = await client.post(
                f"{API_GATEWAY_URL}/api/v1/projects",
                json={
                    "name": project_details["name"],
                    "description": project_details["description"]
                },
                headers={
                    "Authorization": f"Bearer {self.auth_token}"
                }
            )

            response.raise_for_status()

            project = response.json()

        return {
            "message": f"Created project '{project['name']}'",
            "project_id": project["id"]
        }
```

### Traditional Service → AI Service

**Use Case**: Webhook triggers AI analysis

```python
class WebhookService:
    """Traditional webhook service that can trigger AI analysis."""

    async def handle_github_webhook(
        self,
        tenant_id: str,
        event_type: str,
        payload: dict
    ):
        """
        Handle GitHub webhook and trigger AI code review.

        Flow:
        1. Store event (Traditional)
        2. Trigger AI code review (AI)
        3. Post results as comment (Traditional)
        """

        # Store webhook event (Traditional)
        await db.insert(WebhookEvent, {
            "tenant_id": tenant_id,
            "event_type": event_type,
            "payload": json.dumps(payload)
        })

        if event_type == "pull_request":
            # Get PR diff
            pr_diff = payload["pull_request"]["diff_url"]

            # Trigger AI code review (AI Service)
            review = await ai_code_reviewer.review(
                tenant_id=tenant_id,
                pr_url=pr_diff
            )

            # Post review as comment (Traditional: GitHub API)
            await github_client.create_pr_comment(
                repo=payload["repository"]["full_name"],
                pr_number=payload["pull_request"]["number"],
                body=review["summary"]
            )
```

---

## Unified Error Handling

### Cross-Service Error Propagation

```python
class ServiceError(Exception):
    """Base exception for all services."""

    def __init__(self, message: str, service: str, code: str):
        self.message = message
        self.service = service  # "traditional" or "ai"
        self.code = code
        super().__init__(message)

# Traditional service error
class DocumentNotFoundError(ServiceError):
    def __init__(self, document_id: str):
        super().__init__(
            message=f"Document {document_id} not found",
            service="traditional",
            code="DOCUMENT_NOT_FOUND"
        )

# AI service error
class EmbeddingGenerationError(ServiceError):
    def __init__(self, reason: str):
        super().__init__(
            message=f"Embedding generation failed: {reason}",
            service="ai",
            code="EMBEDDING_FAILED"
        )

# Hybrid service with error handling
async def hybrid_service_with_error_handling():
    """Handle errors from both Traditional and AI services."""

    try:
        # Traditional service call
        document = await document_service.get_document("doc_123", "acct_456")

        # AI service call
        embeddings = await embedding_service.embed_document(
            document_id=document.uid,
            tenant_id="acct_456"
        )

    except DocumentNotFoundError as e:
        logger.error(
            "hybrid_service_error",
            service=e.service,
            code=e.code,
            message=e.message
        )
        raise HTTPException(status_code=404, detail=e.message)

    except EmbeddingGenerationError as e:
        logger.error(
            "hybrid_service_error",
            service=e.service,
            code=e.code,
            message=e.message
        )
        # Fallback: Continue without embeddings
        logger.warning("Proceeding without embeddings")
```

---

## Performance Optimization

### Parallel Service Calls

```python
async def get_complete_project_data(tenant_id: str, project_id: str):
    """Fetch data from multiple services in parallel."""

    # Call Traditional and AI services in parallel
    project, documents, conversations, rag_context = await asyncio.gather(
        # Traditional services
        project_service.get_project(tenant_id, project_id),
        document_service.list_documents(tenant_id, project_id),

        # AI services
        chat_service.list_conversations(tenant_id, project_id),
        rag_service.get_project_knowledge_graph(tenant_id, project_id)
    )

    return {
        "project": project,
        "documents": documents,
        "conversations": conversations,
        "knowledge_graph": rag_context
    }
```

---

**Previous**: [Traditional Services](./03a-traditional-services.md) | [AI Services](./03b-ai-services.md) | [Back to Overview](../00-parallel-stacks-overview.md)
