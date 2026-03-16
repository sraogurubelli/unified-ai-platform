# Architecture Diagrams for Presentations

This document contains Mermaid diagrams that can be rendered in presentations. Use tools like:
- **Mermaid Live Editor**: https://mermaid.live
- **VS Code Extension**: Markdown Preview Mermaid Support
- **Export**: PNG/SVG for PowerPoint/Keynote

---

## 1. 4-Layer Architecture Overview

```mermaid
graph TB
    subgraph Gateway["🚪 GATEWAY LAYER"]
        API[API Gateway<br/>REST, GraphQL]
        MCP[MCP Gateway<br/>AI Tools, Agents]
    end

    subgraph Control["🎛️ CONTROL PLANE"]
        IAM[IAM & RBAC]
        Tenant[Multi-Tenancy]
        Obs[Observability]
        Policy[Policy Enforcement]
    end

    subgraph Services["⚙️ SERVICE LAYER"]
        LLM[LLM Gateway]
        Chat[Unified Chat]
        Memory[Memory Management]
        Agent[Agent Orchestration]
        RAG[Graph RAG]
    end

    subgraph Data["💾 DATA PLANE"]
        OLTP[(PostgreSQL<br/>OLTP)]
        OLAP[(ClickHouse<br/>OLAP)]
        Vector[(Qdrant<br/>Vector)]
        Graph[(Neo4j<br/>Graph)]
        Cache[(Redis<br/>Cache)]
    end

    API --> IAM
    MCP --> IAM
    IAM --> LLM
    Tenant --> Chat
    Obs --> Memory
    Policy --> Agent
    LLM --> OLTP
    Chat --> Vector
    Memory --> Cache
    Agent --> Graph
    RAG --> OLAP

    style Gateway fill:#e1f5ff
    style Control fill:#fff4e1
    style Services fill:#f0e1ff
    style Data fill:#e1ffe1
```

---

## 2. Multi-Tenancy Hierarchy

```mermaid
graph TD
    Account[Account<br/>Billing Entity]
    Org1[Organization A]
    Org2[Organization B]
    Proj1[Project 1]
    Proj2[Project 2]
    Proj3[Project 3]
    Conv1[Conversations]
    Conv2[Conversations]
    Msg1[Messages]
    Msg2[Messages]

    Account --> Org1
    Account --> Org2
    Org1 --> Proj1
    Org1 --> Proj2
    Org2 --> Proj3
    Proj1 --> Conv1
    Proj2 --> Conv2
    Conv1 --> Msg1
    Conv2 --> Msg2

    style Account fill:#ff6b6b
    style Org1 fill:#ffd93d
    style Org2 fill:#ffd93d
    style Proj1 fill:#6bcf7f
    style Proj2 fill:#6bcf7f
    style Proj3 fill:#6bcf7f
    style Conv1 fill:#4d96ff
    style Conv2 fill:#4d96ff
```

---

## 3. LLM Gateway Request Flow

```mermaid
sequenceDiagram
    participant Client
    participant Gateway as LLM Gateway
    participant Cache as Semantic Cache
    participant Router as Intelligent Router
    participant CB as Circuit Breaker
    participant LLM as LLM Provider

    Client->>Gateway: Chat Request
    Gateway->>Cache: Check semantic similarity

    alt Cache Hit (similarity > 0.95)
        Cache-->>Gateway: Cached Response
        Gateway-->>Client: Response (90% cost saved)
    else Cache Miss
        Gateway->>Router: Route to provider
        Router->>Router: Calculate: cost, latency, quality
        Router->>CB: Check provider health

        alt Provider Healthy
            CB->>LLM: Forward request
            LLM-->>CB: LLM Response
            CB-->>Gateway: Response
            Gateway->>Cache: Store in cache
            Gateway-->>Client: Response
        else Provider Unhealthy
            CB->>CB: Open circuit
            CB->>Router: Failover to backup
            Router->>LLM: Retry with backup
        end
    end
```

---

## 4. Memory Management - 3-Layer System

```mermaid
graph TB
    subgraph Short["🔥 Short-Term Memory (Redis)"]
        ST1[Conversation Window]
        ST2[Recent Context]
        ST3[TTL: Minutes]
    end

    subgraph Long["💾 Long-Term Memory (PostgreSQL)"]
        LT1[User Preferences]
        LT2[Facts & Knowledge]
        LT3[Permanent Storage]
    end

    subgraph Semantic["🧠 Semantic Memory (Qdrant)"]
        SM1[Vector Embeddings]
        SM2[Similarity Search]
        SM3[Cross-Session Recall]
    end

    Query[Agent Query] --> MemMgr[Memory Manager]
    MemMgr --> Short
    MemMgr --> Long
    MemMgr --> Semantic

    Short --> Fusion[Fusion & Ranking]
    Long --> Fusion
    Semantic --> Fusion
    Fusion --> Response[Unified Memory Context]

    style Short fill:#ff6b6b
    style Long fill:#4d96ff
    style Semantic fill:#6bcf7f
```

---

## 5. Graph RAG - Hybrid Retrieval

```mermaid
graph LR
    Query[User Query:<br/>'What is GraphRAG?']

    subgraph Vector["Vector Search (Qdrant)"]
        VE[Embed Query]
        VS[Similarity Search]
        VR[Top-K Documents]
    end

    subgraph Graph["Graph Search (Neo4j)"]
        GE[Extract Concepts:<br/>'GraphRAG']
        GT[Multi-Hop Traversal<br/>max_hops=2]
        GR[Related Documents]
    end

    Query --> VE
    Query --> GE
    VE --> VS
    VS --> VR
    GE --> GT
    GT --> GR

    VR --> RRF[Reciprocal Rank Fusion]
    GR --> RRF

    RRF --> Final[Unified Results<br/>with Explainability]

    style Vector fill:#e1f5ff
    style Graph fill:#f0e1ff
    style RRF fill:#ffd93d
```

---

## 6. Multi-Agent Swarm Coordination

```mermaid
graph TD
    User[User Query] --> General[General Agent]

    General -->|Handoff: Research needed| Researcher[Researcher Agent]
    General -->|Handoff: Write content| Writer[Writer Agent]

    Researcher -->|Web Search Tool| Web[Web Search]
    Researcher -->|Document Tool| Docs[Document Retrieval]

    Writer -->|Markdown Tool| MD[Markdown Formatter]
    Writer -->|Grammar Tool| Grammar[Grammar Checker]

    Researcher -->|Handoff: Draft ready| Writer
    Writer -->|Handoff: Review| General

    subgraph SharedMemory["Shared Memory Pool (Redis)"]
        Findings[Research Findings]
        Draft[Content Draft]
        Context[Session Context]
    end

    Researcher -.->|Store| Findings
    Writer -.->|Read| Findings
    Writer -.->|Store| Draft
    General -.->|Read| Draft

    General --> Response[Final Response]

    style General fill:#ffd93d
    style Researcher fill:#6bcf7f
    style Writer fill:#4d96ff
    style SharedMemory fill:#ffe1f5
```

---

## 7. Data Plane - Storage Selection

```mermaid
graph TB
    subgraph Transactional["OLTP - PostgreSQL"]
        Users[Users & Auth]
        Projects[Projects]
        Conversations[Conversations]
        Messages[Messages]
    end

    subgraph Analytical["OLAP - ClickHouse / StarRocks"]
        Usage[Usage Events]
        Analytics[Analytics]
        Metrics[Metrics]
    end

    subgraph VectorDB["Vector - Qdrant"]
        Embeddings[Document Embeddings]
        Semantic[Semantic Search]
    end

    subgraph GraphDB["Graph - Neo4j"]
        Entities[Entities]
        Relations[Relationships]
        Knowledge[Knowledge Graph]
    end

    subgraph CacheLayer["Cache - Redis"]
        Sessions[Session State]
        RateLimit[Rate Limiting]
        Queue[Task Queues]
    end

    App[Application] --> Transactional
    App --> Analytical
    App --> VectorDB
    App --> GraphDB
    App --> CacheLayer

    style Transactional fill:#e1f5ff
    style Analytical fill:#fff4e1
    style VectorDB fill:#f0e1ff
    style GraphDB fill:#e1ffe1
    style CacheLayer fill:#ffe1e1
```

---

## 8. Deployment Models Comparison

```mermaid
graph TB
    subgraph SaaS["☁️ SaaS Deployment"]
        SaaSCP[Control Plane<br/>Cloud]
        SaaSDP[Data Plane<br/>Shared Cloud]
        SaaSDB[(Shared Database<br/>Logical Isolation)]
    end

    subgraph Hybrid["🔀 Hybrid Deployment"]
        HybridCP[Control Plane<br/>Cloud]
        HybridDP[Data Plane<br/>Customer Premises]
        HybridDB[(Customer Database<br/>Physical Isolation)]
    end

    subgraph OnPrem["🏢 On-Premise Deployment"]
        OnPremCP[Control Plane<br/>Customer Premises]
        OnPremDP[Data Plane<br/>Customer Premises]
        OnPremDB[(Customer Database<br/>Air-Gapped)]
    end

    Client1[Small Business] --> SaaS
    Client2[Enterprise<br/>Regulated Industry] --> Hybrid
    Client3[Government<br/>Defense] --> OnPrem

    style SaaS fill:#e1f5ff
    style Hybrid fill:#fff4e1
    style OnPrem fill:#f0e1ff
```

---

## 9. Chat Request End-to-End Flow

```mermaid
sequenceDiagram
    participant User
    participant API as API Gateway
    participant Auth as Auth Service
    participant Chat as Chat Service
    participant RAG as Graph RAG
    participant Agent as Agent Orchestrator
    participant LLM as LLM Provider
    participant Stream as SSE Stream

    User->>API: POST /chat
    API->>Auth: Validate JWT
    Auth-->>API: Principal + Permissions

    API->>Chat: Create Conversation
    Chat->>RAG: Retrieve Context
    RAG-->>Chat: Relevant Documents

    Chat->>Agent: Build Agent with Context
    Agent->>LLM: Stream Request

    loop Streaming Response
        LLM-->>Agent: Token
        Agent-->>Stream: SSE Event
        Stream-->>User: Real-time Update
    end

    Agent-->>Chat: Final Response
    Chat->>Chat: Store Message
    Chat-->>User: 200 OK
```

---

## 10. Observability Stack

```mermaid
graph TB
    subgraph App["Application Layer"]
        API[API Services]
        Agents[Agent Services]
        RAG[RAG Services]
    end

    subgraph Instrumentation["Instrumentation"]
        OTel[OpenTelemetry SDK]
        Metrics[Prometheus Client]
        Logs[Structured Logging]
    end

    subgraph Collectors["Collectors"]
        OTLP[OTLP Collector]
        PromCollect[Prometheus]
        LogCollect[Loki]
    end

    subgraph Backends["Backends"]
        Jaeger[Jaeger<br/>Distributed Tracing]
        Grafana[Grafana<br/>Dashboards]
        Alerts[AlertManager]
    end

    API --> OTel
    Agents --> OTel
    RAG --> OTel

    API --> Metrics
    Agents --> Logs

    OTel --> OTLP
    Metrics --> PromCollect
    Logs --> LogCollect

    OTLP --> Jaeger
    PromCollect --> Grafana
    LogCollect --> Grafana
    Grafana --> Alerts

    style App fill:#e1f5ff
    style Instrumentation fill:#fff4e1
    style Collectors fill:#f0e1ff
    style Backends fill:#e1ffe1
```

---

## Usage Instructions

### Converting to Images

**Option 1: Mermaid Live Editor**
1. Copy diagram code
2. Go to https://mermaid.live
3. Paste code
4. Export as PNG or SVG

**Option 2: VS Code**
1. Install "Markdown Preview Mermaid Support" extension
2. Open this file
3. Preview with Cmd+Shift+V (Mac) or Ctrl+Shift+V (Windows)
4. Right-click diagram → Copy as PNG

**Option 3: CLI Tool**
```bash
npm install -g @mermaid-js/mermaid-cli
mmdc -i diagrams.md -o diagrams.png
```

### Customizing Diagrams

- **Colors**: Change `fill:#RRGGBB` in `style` directives
- **Layout**: Adjust `TB` (top-bottom), `LR` (left-right), `RL`, `BT`
- **Node Shapes**:
  - `[]` Rectangle
  - `()` Rounded
  - `[()]` Stadium
  - `{}` Diamond
  - `[()]` Cylinder (database)

---

## Diagram Best Practices for Presentations

1. **High Contrast**: Use dark text on light backgrounds
2. **Large Fonts**: Increase default font size for projector readability
3. **Limited Colors**: Stick to 4-5 colors max per diagram
4. **Progressive Disclosure**: Show diagrams step-by-step (use animation)
5. **Annotations**: Add callouts for key points in presentation software
6. **Consistency**: Use same color scheme across all diagrams
7. **Simplicity**: Remove unnecessary details for executive presentations

---

**Next**: Import these diagrams into your presentation deck and add speaker notes.
