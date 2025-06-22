# Zen MCP Server Architecture

## Executive Summary

The Zen MCP Server is a well-architected system that successfully bridges the stateless MCP (Model Context Protocol) with stateful AI tool orchestration. It employs a clean three-layer architecture (Protocol → Tool → Provider) with strong separation of concerns. While the architecture is solid for single-instance deployments, it faces critical scalability limitations due to its in-memory conversation storage design.

## Core Architecture

### Three-Layer MCP Bridge

```mermaid
graph TB
    subgraph "MCP Client"
        Claude[Claude Desktop/CLI]
    end
    
    subgraph "MCP Protocol Layer"
        Server[server.py]
        TD[Tool Discovery]
        RR[Request Routing]
        MR[Model Resolution]
        CM[Conversation Memory<br/>Reconstruction]
        
        Server --> TD
        Server --> RR
        Server --> MR
        Server --> CM
    end
    
    subgraph "Tool Layer"
        BT[BaseTool<br/>Abstract Base]
        ST[SimpleTool]
        WT[WorkflowTool]
        
        BT --> ST
        BT --> WT
        
        subgraph "Simple Tools"
            Chat[ChatTool]
            LM[ListModelsTool]
            Ver[VersionTool]
        end
        
        subgraph "Workflow Tools"
            Debug[DebugIssueTool]
            CR[CodeReviewTool]
            Plan[PlannerTool]
            Cons[ConsensusTool]
        end
        
        ST --> Chat
        ST --> LM
        ST --> Ver
        
        WT --> Debug
        WT --> CR
        WT --> Plan
        WT --> Cons
    end
    
    subgraph "Provider Layer"
        Reg[ModelProviderRegistry<br/>Singleton]
        
        subgraph "Native Providers"
            Gemini[GeminiProvider]
            OpenAI[OpenAIProvider]
            XAI[XAIProvider]
            DIAL[DIALProvider]
        end
        
        subgraph "Alternative Providers"
            Custom[CustomProvider]
            OR[OpenRouterProvider]
        end
        
        Reg --> Gemini
        Reg --> OpenAI
        Reg --> XAI
        Reg --> DIAL
        Reg --> Custom
        Reg --> OR
    end
    
    Claude -.->|stdio/JSON-RPC| Server
    RR --> BT
    BT --> Reg
    
    style Claude fill:#f9f,stroke:#333,stroke-width:2px
    style Server fill:#bbf,stroke:#333,stroke-width:2px
    style BT fill:#bfb,stroke:#333,stroke-width:2px
    style Reg fill:#fbf,stroke:#333,stroke-width:2px
```

### Request Flow Architecture

```mermaid
sequenceDiagram
    participant C as Claude
    participant S as server.py
    participant CM as ConversationMemory
    participant T as Tool
    participant P as Provider
    participant AI as AI Model
    
    C->>S: JSON-RPC Request<br/>(tool, args)
    
    S->>S: Model Resolution<br/>(validate/select provider)
    
    alt Has continuation_id
        S->>CM: Load Thread Context
        CM-->>S: Previous Turns + Files
        S->>S: Reconstruct Context
    end
    
    S->>T: Execute Tool<br/>(enriched context)
    
    T->>P: Get Provider Instance
    P->>AI: Generate Content
    AI-->>P: Response
    P-->>T: ModelResponse
    
    T->>T: Format Response
    
    alt Conversation Tool
        T->>CM: Store Turn
        CM-->>T: Thread ID
        T->>T: Generate Continuation
    end
    
    T-->>S: ToolOutput
    S-->>C: JSON-RPC Response
```

## Critical Architectural Decisions

### 1. Stateless-to-Stateful Bridge

The most significant architectural decision is bridging MCP's stateless protocol with stateful AI conversations:

- **Problem**: MCP treats each request independently
- **Solution**: In-memory conversation storage with UUID-based thread tracking
- **Trade-off**: Enables rich multi-turn conversations but limits horizontal scaling

```mermaid
graph LR
    subgraph "Stateless MCP"
        R1[Request 1]
        R2[Request 2]
        R3[Request 3]
    end
    
    subgraph "Conversation Memory"
        T1[Thread UUID]
        T1 --> Turn1["Turn 1<br/>tool: analyze<br/>files: a.py"]
        T1 --> Turn2["Turn 2<br/>tool: debug<br/>context: preserved"]
        T1 --> Turn3["Turn 3<br/>tool: chat<br/>full history"]
    end
    
    R1 --> T1
    R2 --> T1
    R3 --> T1
    
    style T1 fill:#ffd,stroke:#333,stroke-width:2px
```

### 2. Tool Abstraction Hierarchy

```mermaid
classDiagram
    class BaseTool {
        <<abstract>>
        +name: str
        +description: str
        +execute(request) ToolOutput
        #prepare_prompt(request) str
        #get_conversation_context() dict
    }
    
    class SimpleTool {
        +execute(request) ToolOutput
        -_handle_large_prompt()
        -_prepare_model_request()
    }
    
    class WorkflowTool {
        +execute(request) ToolOutput
        +validate_request()
        +execute_workflow()
        +should_call_expert()
    }
    
    class BaseWorkflowMixin {
        +get_required_actions()
        +prepare_step_data()
        +handle_work_completion()
    }
    
    BaseTool <|-- SimpleTool
    BaseTool <|-- WorkflowTool
    BaseWorkflowMixin <|-- WorkflowTool
    
    SimpleTool <|-- ChatTool
    SimpleTool <|-- ListModelsTool
    SimpleTool <|-- VersionTool
    
    WorkflowTool <|-- DebugIssueTool
    WorkflowTool <|-- CodeReviewTool
    WorkflowTool <|-- PlannerTool
    WorkflowTool <|-- ConsensusTool
```

### 3. Provider Priority System

```mermaid
graph TD
    subgraph "Provider Selection Priority"
        A[Request with model name]
        B{Model available?}
        
        C[1. Native APIs<br/>Gemini, OpenAI, XAI, DIAL]
        D[2. Custom Provider<br/>Local models]
        E[3. OpenRouter<br/>Fallback]
        
        A --> B
        B -->|Check Priority 1| C
        C -->|Not Found| D
        D -->|Not Found| E
        E -->|Not Found| F[Error: Model not available]
        
        C -->|Found| G[Use Provider]
        D -->|Found| G
        E -->|Found| G
    end
    
    style C fill:#9f9,stroke:#333,stroke-width:2px
    style D fill:#ff9,stroke:#333,stroke-width:2px
    style E fill:#f99,stroke:#333,stroke-width:2px
```

## Key Architectural Patterns

### 1. Singleton Registry Pattern

The `ModelProviderRegistry` uses singleton pattern for global provider management:

```python
class ModelProviderRegistry:
    _instance = None
    
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance._providers = {}
            cls._instance._initialized_providers = {}
        return cls._instance
```

**Benefits**:
- Single source of truth for provider configuration
- Lazy initialization prevents unnecessary API client creation
- Thread-safe provider instance caching

### 2. Conversation Memory Architecture

```mermaid
graph LR
    subgraph "In-Memory Storage"
        ThreadDict["threads: Dict of uuid to Thread"]
        
        subgraph "Thread Structure"
            TID[id: UUID]
            Created[created_at: datetime]
            Turns["turns: List of Turn"]
            
            subgraph "Turn Structure"
                Tool[tool: str]
                Req[request: dict]
                Resp[response: dict]
                Files["files: List of str"]
                Images["images: List of str"]
            end
        end
        
        ThreadDict --> TID
        ThreadDict --> Created
        ThreadDict --> Turns
        Turns --> Tool
        Turns --> Req
        Turns --> Resp
        Turns --> Files
        Turns --> Images
    end
    
    style ThreadDict fill:#ffd,stroke:#333,stroke-width:2px
```

**Key Features**:
- UUID-based conversation thread identification
- Turn-by-turn history with tool attribution
- Cross-tool continuation support
- Newest-first file prioritization
- 20-turn maximum to prevent runaway conversations

### 3. Model Resolution Flow

```mermaid
flowchart TD
    A[Tool Request] --> B{Has model param?}
    B -->|No| C{DEFAULT_MODEL set?}
    B -->|Yes| D[Validate Model]
    
    C -->|auto| E[Select by Tool Category]
    C -->|specific| D
    C -->|not set| F[Error: Model required]
    
    E --> G{Tool Category}
    G -->|FAST_RESPONSE| H[Select Fast Model<br/>e.g., Flash]
    G -->|EXTENDED_REASONING| I[Select Powerful Model<br/>e.g., Pro, O3]
    
    D --> J{Model Available?}
    J -->|Yes| K[Get Provider]
    J -->|No| L[Error: Model not found]
    
    H --> K
    I --> K
    K --> M[Execute with Provider]
```

## Architectural Strengths

### 1. Clean Separation of Concerns

- **Protocol Layer**: Handles only MCP communication and routing
- **Tool Layer**: Business logic independent of transport or providers
- **Provider Layer**: Unified interface for diverse AI services

### 2. Extensibility Points

```mermaid
graph TD
    subgraph "Extension Points"
        NT[New Tool]
        NP[New Provider]
        NW[New Workflow]
        
        NT --> ST1[Inherit SimpleTool]
        NT --> WT1[Inherit WorkflowTool]
        
        NP --> MP[Implement ModelProvider]
        NP --> REG[Register in Registry]
        
        NW --> BWM[Extend BaseWorkflowMixin]
        NW --> SCHEMA[Define Request Schema]
    end
```

### 3. Developer Experience

- Consistent patterns across all tools
- Comprehensive type hints via Pydantic models
- Rich inline documentation
- Clear error messages with context

## Architectural Weaknesses

### 1. Scalability Bottleneck

**In-Memory Conversation Storage**:
- Cannot scale horizontally (no shared state)
- Data loss on process restart
- No disaster recovery capability

**Impact Analysis**:
```mermaid
graph TD
    A[Single Process] --> B[In-Memory Storage]
    B --> C[Cannot Load Balance]
    B --> D[Cannot Survive Restart]
    B --> E[Cannot Share State]
    
    C --> F[Scalability Limited]
    D --> G[Poor Reliability]
    E --> H[No Multi-Instance]
    
    style B fill:#f99,stroke:#333,stroke-width:2px
    style F fill:#fcc,stroke:#333,stroke-width:2px
    style G fill:#fcc,stroke:#333,stroke-width:2px
    style H fill:#fcc,stroke:#333,stroke-width:2px
```

### 2. Complex Inheritance

```mermaid
graph TD
    subgraph "Current (Complex)"
        WT2[WorkflowTool]
        BT2[BaseTool]
        BWM2[BaseWorkflowMixin]
        
        BT2 --> WT2
        BWM2 --> WT2
        
        WT2 --> MRO[Complex MRO]
        WT2 --> DUP[Duplicate Methods]
    end
    
    subgraph "Proposed (Simple)"
        WT3[WorkflowTool]
        BT3[BaseTool]
        WO[WorkflowOrchestrator]
        
        BT3 --> WT3
        WT3 --> WO
        
        WT3 --> COMP[Clean Composition]
    end
    
    style MRO fill:#f99,stroke:#333,stroke-width:2px
    style DUP fill:#f99,stroke:#333,stroke-width:2px
    style COMP fill:#9f9,stroke:#333,stroke-width:2px
```

### 3. Configuration Management

**Current State**: Environment variables accessed throughout codebase
**Problems**: No validation, no defaults management, scattered access

## Security & Operational Considerations

### Current Security Model

```mermaid
graph TD
    subgraph "Security Boundaries"
        MCP[MCP Client<br/>Trusted]
        Server[Zen Server<br/>No Auth]
        Files[File System<br/>Path Validation]
        API[AI APIs<br/>Key Protected]
    end
    
    MCP --> Server
    Server --> Files
    Server --> API
    
    style MCP fill:#9f9,stroke:#333,stroke-width:2px
    style Server fill:#ff9,stroke:#333,stroke-width:2px
    style Files fill:#ff9,stroke:#333,stroke-width:2px
    style API fill:#9f9,stroke:#333,stroke-width:2px
```

**Gaps**:
- No authentication mechanism
- No rate limiting
- No resource quotas
- File access via path validation only

### Observability

**Current**: Basic logging via Python logging module
**Missing**: Metrics, distributed tracing, health checks

## Performance Characteristics

### 1. Lazy Loading Strategy

```mermaid
sequenceDiagram
    participant R as Request
    participant Reg as Registry
    participant P as Provider
    
    R->>Reg: get_provider(type)
    
    alt Provider Not Initialized
        Reg->>Reg: Check API Key
        Reg->>P: Create Instance
        Reg->>Reg: Cache Instance
    end
    
    Reg-->>R: Return Cached Instance
```

### 2. Connection Pooling

All providers use httpx clients with connection pooling:
- Max connections: 10
- Keepalive: 30 seconds
- Timeout: Configurable per provider

### 3. Token Optimization

**Newest-First File Prioritization**:
- Recent file versions take precedence
- Older versions excluded when hitting token limits
- Maintains most relevant context

## Future Architecture Roadmap

### Immediate (1-2 weeks)

```mermaid
graph LR
    A[Current State] --> B[Storage Abstraction]
    B --> C[Redis Support]
    B --> D[Config Singleton]
    B --> E[Remove Duplicates]
    
    style B fill:#9f9,stroke:#333,stroke-width:2px
```

### Short-term (1-3 months)

```mermaid
graph LR
    A[After Immediate] --> B[Composition Refactor]
    B --> C[Add Auth/Rate Limit]
    B --> D[Metrics System]
    B --> E[Response Cache]
    
    style B fill:#99f,stroke:#333,stroke-width:2px
```

### Medium-term (3-6 months)

```mermaid
graph LR
    A[After Short-term] --> B[Async Architecture]
    B --> C[Plugin System]
    B --> D[WebSocket Support]
    B --> E[Health Monitoring]
    
    style B fill:#f9f,stroke:#333,stroke-width:2px
```

### Long-term Vision (6+ months)

```mermaid
graph TD
    subgraph "Microservices Architecture"
        GW[API Gateway]
        MCP_SVC[MCP Service]
        PROV_SVC[Provider Service]
        CONV_SVC[Conversation Service]
        
        GW --> MCP_SVC
        GW --> GraphQL[GraphQL API]
        MCP_SVC --> CONV_SVC
        MCP_SVC --> PROV_SVC
    end
    
    subgraph "Infrastructure"
        K8S[Kubernetes]
        Redis[Redis Cluster]
        Kafka[Event Stream]
    end
    
    MCP_SVC --> Redis
    PROV_SVC --> Kafka
    CONV_SVC --> Redis
```

## Conclusion

The Zen MCP Server demonstrates solid architectural foundations with clear separation of concerns, extensible design patterns, and thoughtful abstractions. The stateless-to-stateful bridge via conversation memory is an elegant solution to MCP's limitations, though it introduces scalability constraints.

The architecture is well-suited for its current use case (single-instance Claude Desktop integration) but requires evolution for production deployments. The proposed storage abstraction would immediately address the most critical limitation while maintaining backward compatibility.

Key architectural wins:
- Clean three-layer separation
- Provider-agnostic tool design  
- Extensible patterns throughout
- Strong typing and documentation

Primary challenges:
- In-memory state storage
- Complex inheritance hierarchies
- Scattered configuration
- Limited observability

The modular design provides an excellent foundation for evolution while the clear abstraction boundaries enable incremental improvements without system-wide refactoring.

## Technology Stack

### Core Technologies

#### Python 3.10+ 
**Reasoning**: Modern Python features enable clean async support, type hints, and pattern matching. The MCP SDK requires Python 3.10+.

#### MCP (Model Context Protocol) SDK
**Library**: `mcp==1.3.2`  
**Purpose**: Protocol implementation for Claude integration  
**Reasoning**: Official Anthropic SDK ensures compatibility and protocol compliance

### Framework & Architecture Libraries

#### Pydantic v2
**Library**: `pydantic==2.10.4`  
**Purpose**: Data validation, serialization, and schema generation  
**Reasoning**: 
- Type-safe request/response models
- Automatic validation with clear error messages
- JSON Schema generation for MCP tool discovery
- Performance improvements in v2

#### httpx
**Library**: `httpx==0.28.1`  
**Purpose**: HTTP client with connection pooling  
**Reasoning**:
- Modern async-capable HTTP client
- Built-in connection pooling for provider efficiency
- Timeout and retry handling
- HTTP/2 support for better performance

### AI Provider SDKs

#### OpenAI Python SDK
**Library**: `openai==1.59.5`  
**Purpose**: OpenAI and OpenAI-compatible providers (DIAL)  
**Reasoning**:
- Industry standard for LLM interactions
- Supports OpenAI-compatible endpoints
- Streaming response support
- Well-maintained with regular updates

#### Google Generative AI
**Library**: `google-generativeai==0.8.3`  
**Purpose**: Gemini model integration  
**Reasoning**:
- Official Google SDK for Gemini
- Native support for Gemini-specific features (thinking mode)
- Handles multi-modal inputs (text + images)

### Development Tools

#### Testing Framework
**Libraries**:
- `pytest==8.3.4` - Test runner
- `pytest-asyncio==0.25.2` - Async test support
- `pytest-mock==3.14.1` - Mocking utilities

**Reasoning**:
- Pytest's fixture system enables clean test setup
- Async support crucial for protocol testing
- Rich assertion introspection
- Extensive plugin ecosystem

#### Code Quality
**Libraries**:
- `ruff==0.9.1` - Fast Python linter
- `black==24.10.0` - Code formatter
- `isort==5.13.2` - Import sorter
- `mypy==1.14.1` - Static type checker

**Reasoning**:
- Ruff combines multiple linters (flake8, pylint, etc.) with 10-100x speed
- Black ensures consistent formatting (no debates)
- Type checking catches errors early
- All tools work together harmoniously

### Utility Libraries

#### python-dotenv
**Library**: `python-dotenv==1.0.1`  
**Purpose**: Environment variable management  
**Reasoning**:
- Standard practice for configuration
- Supports .env files for local development
- Keeps secrets out of code

#### Pillow
**Library**: `Pillow==11.1.0`  
**Purpose**: Image processing for vision models  
**Reasoning**:
- De facto standard for Python image processing
- Supports all common formats
- Efficient base64 encoding for API calls

### Logging & Observability

#### Python Logging + RotatingFileHandler
**Built-in**: Standard library  
**Purpose**: Structured logging with rotation  
**Reasoning**:
- Zero dependencies for core functionality
- RotatingFileHandler prevents disk fill
- Structured logs enable parsing
- Compatible with any log aggregation system

**Configuration**:
```python
# 20MB max file size, 10 backups for server logs
# 20MB max file size, 5 backups for activity logs
```

### Architecture Patterns & Libraries

#### Singleton Pattern
**Implementation**: Custom `__new__` override  
**Used in**: ModelProviderRegistry  
**Reasoning**:
- Ensures single source of truth for providers
- Prevents duplicate API client initialization
- Thread-safe implementation

#### Abstract Base Classes
**Library**: `abc` (built-in)  
**Used in**: BaseTool, ModelProvider  
**Reasoning**:
- Enforces interface contracts
- Clear extension points
- IDE support for implementation checking

#### UUID Generation
**Library**: `uuid` (built-in)  
**Purpose**: Conversation thread identification  
**Reasoning**:
- Globally unique identifiers
- No coordination required
- URL-safe string representation

### Missing Technologies (By Design)

#### No Web Framework
**Reasoning**: MCP uses stdio, not HTTP. Adding Flask/FastAPI would add complexity without benefit.

#### No Database
**Current**: In-memory dictionaries  
**Reasoning**: Simplicity for initial implementation, though this limits scalability  
**Future**: Redis/PostgreSQL for persistence

#### No Message Queue
**Current**: Synchronous request/response  
**Reasoning**: MCP protocol is inherently synchronous  
**Future**: Could add for provider parallelization

#### No Container Orchestration
**Current**: Single process design  
**Reasoning**: Desktop integration focus  
**Future**: Kubernetes for cloud deployment

### Technology Decisions Matrix

| Component | Technology | Alternative Considered | Decision Reasoning |
|-----------|------------|----------------------|-------------------|
| Protocol | MCP SDK | Custom JSON-RPC | Official SDK ensures compatibility |
| Validation | Pydantic v2 | attrs, dataclasses | Schema generation, validation |
| HTTP Client | httpx | requests, aiohttp | Modern, connection pooling |
| Testing | pytest | unittest | Fixtures, async support |
| Linting | ruff | flake8, pylint | Speed, all-in-one |
| Formatting | black | yapf, autopep8 | Community standard |
| Images | Pillow | opencv-python | Lighter weight, sufficient |
| Config | python-dotenv | configparser | .env is standard practice |

### Performance Characteristics

#### Memory Usage
- Base: ~50MB Python interpreter
- Per provider: ~10-20MB (client libraries)
- Per conversation: ~1-5KB (metadata + turns)
- Images: Temporarily loaded, then gc'd

#### Startup Time
- Cold start: 2-3 seconds (import time)
- Provider init: Lazy (on first use)
- Tool discovery: <100ms (pre-computed)

#### Request Latency
- MCP overhead: <5ms
- Provider calls: 500ms-30s (model dependent)
- Memory operations: <1ms
- File I/O: Variable (size dependent)

### Security Considerations

#### Dependency Management
- All dependencies pinned to exact versions
- Regular security updates via Dependabot
- No use of deprecated libraries
- Minimal dependency tree

#### Secret Management
- API keys via environment variables
- No secrets in code or logs
- .env file in .gitignore
- Keys redacted in error messages

#### Input Validation
- All inputs validated via Pydantic
- Path traversal prevention
- File size limits for images
- Request size limits (MCP protocol)

### Technology Stack Layers

```mermaid
graph TD
    subgraph "Application Layer"
        Tools[Tools<br/>SimpleTool, WorkflowTool]
        Providers[Providers<br/>Gemini, OpenAI, DIAL]
        Server[MCP Server<br/>server.py]
    end
    
    subgraph "Framework Layer"
        Pydantic[Pydantic v2<br/>Validation & Schemas]
        MCP[MCP SDK<br/>Protocol]
        ABC[Abstract Base Classes<br/>Interfaces]
    end
    
    subgraph "Library Layer"
        HTTPX[httpx<br/>HTTP Client]
        OpenAI_SDK[OpenAI SDK<br/>LLM Client]
        Google_SDK[Google AI SDK<br/>Gemini Client]
        Pillow[Pillow<br/>Image Processing]
    end
    
    subgraph "Infrastructure Layer"
        Logging[Python Logging<br/>+ Rotation]
        DotEnv[python-dotenv<br/>Config]
        UUID[UUID<br/>Thread IDs]
    end
    
    subgraph "Development Layer"
        Pytest[pytest<br/>Testing]
        Ruff[ruff<br/>Linting]
        Black[black<br/>Formatting]
        MyPy[mypy<br/>Type Checking]
    end
    
    Tools --> Pydantic
    Tools --> ABC
    Providers --> HTTPX
    Providers --> OpenAI_SDK
    Providers --> Google_SDK
    Server --> MCP
    Server --> Logging
    
    style Application fill:#f9f,stroke:#333,stroke-width:2px
    style Framework fill:#9ff,stroke:#333,stroke-width:2px
    style Library fill:#ff9,stroke:#333,stroke-width:2px
    style Infrastructure fill:#9f9,stroke:#333,stroke-width:2px
    style Development fill:#f99,stroke:#333,stroke-width:2px
```

### Dependency Relationships

```mermaid
graph LR
    subgraph "Core Dependencies"
        MCP_SDK[mcp>=1.0.0]
        Pydantic[pydantic>=2.0.0]
        DotEnv[python-dotenv>=1.0.0]
    end
    
    subgraph "Provider Dependencies"
        OpenAI[openai>=1.55.2]
        GoogleAI[google-genai>=1.19.0]
        HTTPX[httpx via openai]
        Pillow[Pillow - images]
    end
    
    subgraph "Dev Dependencies"
        Pytest[pytest>=7.4.0]
        AsyncIO[pytest-asyncio>=0.21.0]
        Mock[pytest-mock>=3.11.0]
        Black[black>=23.0.0]
        Ruff[ruff>=0.1.0]
        ISort[isort>=5.12.0]
    end
    
    OpenAI --> HTTPX
    GoogleAI --> Pillow
    Pytest --> AsyncIO
    Pytest --> Mock
```

### Build & Deployment Tools

#### Shell Scripts
**Scripts**:
- `run-server.sh` - Setup and configuration
- `code_quality_checks.sh` - CI/CD quality gates
- `run_integration_tests.sh` - Integration test runner

**Reasoning**:
- Cross-platform compatibility (with WSL on Windows)
- Self-documenting installation process
- Automated environment setup
- Consistent developer experience

#### Virtual Environment Management
**Tool**: Python venv (built-in)  
**Location**: `.zen_venv/`  
**Reasoning**:
- No additional dependencies
- Standard Python practice
- Isolates project dependencies
- Works across all platforms

#### Configuration Management
**Files**:
- `.env` - Local environment variables
- `conf/custom_models.json` - Model configurations
- `systemprompts/` - System prompt templates

**Reasoning**:
- Clear separation of config from code
- Easy to customize without code changes
- Version control friendly (templates checked in)

### CI/CD Pipeline

```mermaid
graph LR
    subgraph "Local Development"
        Edit[Code Changes]
        Quality[./code_quality_checks.sh]
        Test[Unit Tests]
    end
    
    subgraph "Integration Testing"
        Integration[./run_integration_tests.sh]
        Simulator[Simulator Tests]
    end
    
    subgraph "Deployment"
        Setup[./run-server.sh]
        Config[MCP Configuration]
        Run[Claude Integration]
    end
    
    Edit --> Quality
    Quality --> Test
    Test --> Integration
    Integration --> Simulator
    Simulator --> Setup
    Setup --> Config
    Config --> Run
    
    style Quality fill:#9f9,stroke:#333,stroke-width:2px
    style Test fill:#9f9,stroke:#333,stroke-width:2px
    style Integration fill:#ff9,stroke:#333,stroke-width:2px
```

### Future Technology Considerations

#### Short-term Additions
- **Redis**: For conversation persistence
- **Prometheus client**: For metrics
- **structlog**: For structured logging
- **tenacity**: For advanced retry logic

#### Medium-term Evolution
- **FastAPI**: If adding REST API
- **SQLAlchemy**: For relational data
- **Celery**: For async task processing
- **OpenTelemetry**: For distributed tracing

#### Long-term Platform
- **gRPC**: For service communication
- **Kubernetes operators**: For deployment
- **Apache Kafka**: For event streaming
- **GraphQL**: For flexible querying