# Dataplane Architecture Documentation

## Overview

Dataplane is a high-performance data pipeline orchestration platform built with Go and React. It provides drag-and-drop pipeline building, distributed task execution, secrets management, and multi-environment support.

## System Architecture

### High-Level Architecture

```mermaid
graph TB
    subgraph "Frontend Layer"
        UI[React Frontend<br/>Port 9001]
    end

    subgraph "Main Application"
        API[Main App Server<br/>Go + Fiber<br/>Port 9001]
        GQL[GraphQL API]
        WS[WebSocket Server]
        AUTH[Authentication]
        SCHED[Scheduler]
    end

    subgraph "Data Layer"
        PG[(PostgreSQL<br/>TimescaleDB)]
        REDIS[(Redis<br/>Cache)]
        NATS[NATS<br/>Message Queue]
    end

    subgraph "Worker Layer"
        W1[Worker 1<br/>Python Dev]
        W2[Worker 2<br/>Python Prod]
        WN[Worker N<br/>Custom]
    end

    subgraph "Storage"
        DFS[Distributed File System<br/>DB/LocalFile/S3]
    end

    UI -->|GraphQL/WS| API
    API --> GQL
    API --> WS
    API --> AUTH
    API --> SCHED

    API <-->|SQL| PG
    API <-->|Cache| REDIS
    API <-->|Pub/Sub| NATS

    NATS <-->|Task Queue| W1
    NATS <-->|Task Queue| W2
    NATS <-->|Task Queue| WN

    W1 <-->|Metadata| PG
    W2 <-->|Metadata| PG
    WN <-->|Metadata| PG

    W1 <-->|Files| DFS
    W2 <-->|Files| DFS
    WN <-->|Files| DFS

    API <-->|Code Files| DFS
```

## Core Components

### 1. Main Application Server

```mermaid
graph LR
    subgraph "Main App (Port 9001)"
        FIBER[Fiber Server]

        subgraph "API Layer"
            PUB[Public GraphQL]
            PRIV[Private GraphQL]
            DESK[Desktop GraphQL]
            REST[REST Triggers]
        end

        subgraph "Business Logic"
            PIPE[Pipeline Manager]
            PERM[Permissions]
            CODE[Code Editor]
            SEC[Secrets Manager]
        end

        subgraph "Services"
            SCH[Scheduler Service]
            RWMGR[Remote Worker Mgr]
            LOGGER[Platform Logger]
        end

        FIBER --> PUB
        FIBER --> PRIV
        FIBER --> DESK
        FIBER --> REST

        PUB --> PIPE
        PRIV --> PIPE
        PRIV --> PERM
        PRIV --> CODE
        PRIV --> SEC

        PIPE --> SCH
        PIPE --> RWMGR
        PIPE --> LOGGER
    end
```

**Responsibilities:**
- GraphQL API serving (public, private, desktop variants)
- REST API for external pipeline triggers
- WebSocket connections for real-time updates
- User authentication and authorization
- Pipeline orchestration and scheduling
- Code editor backend
- Secrets management
- Static file serving for React frontend

**Technology:**
- Go with Fiber web framework
- gqlgen for GraphQL
- JWT authentication
- WebSocket for real-time updates

### 2. Worker System

```mermaid
graph TB
    subgraph "Worker Architecture"
        WSERVER[Worker Server<br/>Fiber - Port 9005]

        subgraph "Task Execution"
            LISTEN[Task Listener<br/>NATS Subscribe]
            RUNNER[Task Runner]
            EXEC[Code Executor<br/>Python/Bash/etc]
        end

        subgraph "Worker Services"
            HEALTH[Health Monitor]
            SECRETS[Secrets Cache]
            PACKAGES[Package Manager]
            METRICS[Resource Metrics]
        end

        subgraph "File Management"
            CACHE[Local File Cache]
            SYNC[File Sync]
        end

        WSERVER --> LISTEN
        LISTEN --> RUNNER
        RUNNER --> EXEC

        WSERVER --> HEALTH
        WSERVER --> SECRETS
        WSERVER --> PACKAGES
        WSERVER --> METRICS

        RUNNER --> CACHE
        CACHE --> SYNC
    end
```

**Responsibilities:**
- Execute pipeline tasks (Python, Bash, etc.)
- Manage code packages and dependencies
- Monitor resource usage (CPU, memory)
- Cache code files locally
- Report health status
- Handle secrets securely
- Stream execution logs

**Worker Types:**
- VM workers (virtual machine execution)
- Container workers (Docker containers)
- Process workers (direct process execution)

**Worker Groups:**
- Environment-specific (Development, Production, Staging)
- Language-specific (Python, Go, etc.)
- Custom worker groups per use case

### 3. Communication Layer

```mermaid
sequenceDiagram
    participant F as Frontend
    participant A as Main App
    participant N as NATS
    participant W as Worker
    participant D as Database

    F->>A: GraphQL: Create Pipeline
    A->>D: Store Pipeline Definition
    A->>W: Cache Invalidation (NATS)
    W->>D: Fetch Code Files

    F->>A: GraphQL: Run Pipeline
    A->>D: Create Pipeline Run
    A->>N: Publish: pipeline-run-next
    N->>A: Subscribe: Process Dependencies
    A->>N: Publish: Task Queue
    N->>W: Deliver: Task
    W->>W: Execute Task
    W->>N: Publish: taskupdate.*
    N->>A: Receive: Status Update
    A->>F: WebSocket: Live Status
    W->>D: Store Results
    W->>N: Task Complete
    N->>A: Next Task Trigger
```

**Communication Patterns:**

1. **Frontend ↔ Main App:**
   - GraphQL for data operations
   - WebSocket for real-time updates
   - REST API for external triggers

2. **Main App ↔ Workers:**
   - NATS for task distribution
   - WebSocket for remote workers
   - Database for metadata sync

3. **Main App ↔ Data Layer:**
   - PostgreSQL for persistent storage
   - Redis for caching and sessions
   - NATS for pub/sub messaging

### 4. Data Layer Architecture

```mermaid
graph TB
    subgraph "PostgreSQL (TimescaleDB)"
        USERS[Users & Permissions]
        PIPES[Pipelines]
        RUNS[Pipeline Runs]
        TASKS[Tasks]
        WORKERS[Workers]
        SECRETS[Encrypted Secrets]
        FILES[Code Files Metadata]
        ENVS[Environments]
    end

    subgraph "Redis"
        CACHE[API Cache]
        SESSIONS[User Sessions]
        TOKENS[Worker Tokens]
        TRIGGERS[API Trigger Data]
    end

    subgraph "NATS Topics"
        RUN[pipeline-run-next]
        UPDATE[taskupdate.*]
        SCHED[pipeline-scheduler]
        DIST[distributed-storage]
        SEC[secrets-update]
    end
```

**Database Schema (Key Tables):**
- `users`: User accounts and authentication
- `permissions`: Fine-grained access control
- `environments`: Development, Production, etc.
- `pipelines`: Pipeline definitions and configurations
- `pipeline_runs`: Execution instances
- `tasks`: Individual task executions
- `workers`: Worker registration and health
- `code_files`: Code and file storage
- `secrets`: Encrypted secret values
- `deployments`: Version deployments

## Pipeline Execution Flow

```mermaid
stateDiagram-v2
    [*] --> Trigger

    Trigger --> CreateRun: User/API/Schedule
    CreateRun --> QueueTasks: Parse Pipeline DAG

    QueueTasks --> SelectWorker: Load Balancing
    SelectWorker --> DownloadCode: Worker Selected
    DownloadCode --> Execute: Files Cached

    Execute --> Running: Task Started
    Running --> CheckDependency: Task Complete

    CheckDependency --> QueueNext: Dependencies Met
    CheckDependency --> Wait: Dependencies Pending
    Wait --> QueueNext: All Dependencies Done

    QueueNext --> SelectWorker: More Tasks
    CheckDependency --> Complete: All Tasks Done

    Complete --> [*]

    Running --> Failed: Error Occurred
    Failed --> [*]

    note right of Execute
        Real-time logging via
        WebSocket & NATS
    end note

    note right of SelectWorker
        Round-robin or
        least-loaded strategy
    end note
```

### Pipeline Execution Steps

1. **Trigger**: Pipeline started via UI, API, or scheduler
2. **Task Creation**: Pipeline nodes converted to executable tasks
3. **Dependency Resolution**: Task graph analyzed for execution order
4. **Queue Management**: Tasks queued to appropriate worker groups
5. **Worker Selection**: Load balancer assigns tasks to workers
6. **File Distribution**: Code files downloaded to worker cache
7. **Execution**: Worker executes task with environment and secrets
8. **Status Updates**: Real-time progress via NATS and WebSocket
9. **Dependency Check**: Next tasks triggered when dependencies complete
10. **Completion**: Results stored, notifications sent

## Distributed File System

```mermaid
graph LR
    subgraph "Code Editor"
        EDIT[Edit Files]
        SAVE[Save Changes]
    end

    subgraph "Storage Backend"
        DB[(Database Storage)]
        LOCAL[Local File System]
        S3[S3 Storage]
    end

    subgraph "Distribution"
        COMPRESS[Compress Files]
        TRANSFER[Transfer to Workers]
    end

    subgraph "Worker Cache"
        W1C[Worker 1 Cache]
        W2C[Worker 2 Cache]
        WNC[Worker N Cache]
    end

    EDIT --> SAVE
    SAVE -->|Store| DB
    SAVE -->|Store| LOCAL
    SAVE -->|Store| S3

    DB --> COMPRESS
    LOCAL --> COMPRESS
    S3 --> COMPRESS

    COMPRESS --> TRANSFER
    TRANSFER -->|Download| W1C
    TRANSFER -->|Download| W2C
    TRANSFER -->|Download| WNC

    W1C -->|Invalidate| TRANSFER
    W2C -->|Invalidate| TRANSFER
    WNC -->|Invalidate| TRANSFER
```

**Features:**
- **Storage Options**: Database, Local File System, S3
- **Caching**: Node-level file caching with automatic invalidation
- **Compression**: Files compressed before transfer
- **Versioning**: Version control for code deployments
- **Multi-Environment**: Separate file spaces per environment

## Authentication & Authorization

```mermaid
graph TB
    subgraph "Authentication Methods"
        LOGIN[Username/Password]
        OIDC[OpenID Connect]
        APIKEY[API Keys]
        WORKER[Worker Tokens]
    end

    subgraph "Token Generation"
        JWT[JWT Generation]
        SESSION[Session Creation]
    end

    subgraph "Permission System"
        CHECK[Permission Check]

        subgraph "Resource Types"
            PLAT[Platform Admin]
            ENV[Environment Access]
            PIPE[Pipeline Access]
            WGRP[Worker Group Access]
        end

        subgraph "Operations"
            READ[Read]
            WRITE[Write]
            DELETE[Delete]
            EXECUTE[Execute]
        end
    end

    LOGIN --> JWT
    OIDC --> JWT
    APIKEY --> SESSION
    WORKER --> SESSION

    JWT --> CHECK
    SESSION --> CHECK

    CHECK --> PLAT
    CHECK --> ENV
    CHECK --> PIPE
    CHECK --> WGRP

    PLAT --> READ
    ENV --> READ
    PIPE --> READ
    WGRP --> READ

    READ --> WRITE
    WRITE --> DELETE
    DELETE --> EXECUTE
```

**Security Layers:**

1. **Authentication:**
   - JWT tokens for web sessions
   - API keys for external integrations
   - OpenID Connect for enterprise SSO
   - Session tokens for remote workers

2. **Authorization:**
   - Platform-level permissions (admin access)
   - Environment-level permissions (dev/prod isolation)
   - Pipeline-level permissions (specific pipeline access)
   - Resource-based access control

3. **Secrets Management:**
   - AES encryption with platform keys
   - Environment-scoped secrets
   - Secure injection to workers
   - No plaintext storage
   - Log redaction

## Scheduler System

```mermaid
graph TB
    subgraph "Scheduler Architecture"
        LEADER[Leader Election]
        CRON[Cron Engine<br/>gocron]
        PARSER[RRule Parser]
        TRIGGER[Pipeline Trigger]
    end

    subgraph "Schedule Management"
        CREATE[Create Schedule]
        UPDATE[Update Schedule]
        DELETE[Delete Schedule]
        LISTEN[NATS Listener<br/>pipeline-scheduler]
    end

    subgraph "Execution"
        CHECK[Check Schedule]
        EXEC[Execute Pipeline]
        LOG[Log Execution]
    end

    LEADER --> CRON

    CREATE --> LISTEN
    UPDATE --> LISTEN
    DELETE --> LISTEN

    LISTEN --> CRON

    CRON --> CHECK
    CHECK --> EXEC
    EXEC --> TRIGGER
    EXEC --> LOG
```

**Features:**
- RRule-based recurring schedules
- Multi-timezone support
- Dynamic schedule updates via NATS
- Leader election (single active scheduler)
- Cron expression support
- Schedule versioning per deployment

## Worker Health Monitoring

```mermaid
graph LR
    subgraph "Worker Metrics"
        CPU[CPU Usage]
        MEM[Memory Usage]
        LOAD[System Load]
        TASKS[Active Tasks]
    end

    subgraph "Health Check"
        BEAT[Heartbeat<br/>1s Interval]
        STATUS[Status Report]
        ALERT[Alert Threshold]
    end

    subgraph "Main App"
        COLLECT[Metrics Collection]
        STORE[Store in DB]
        DISPLAY[Display in UI]
    end

    CPU --> BEAT
    MEM --> BEAT
    LOAD --> BEAT
    TASKS --> BEAT

    BEAT --> STATUS
    STATUS --> ALERT

    ALERT -->|NATS| COLLECT
    COLLECT --> STORE
    STORE --> DISPLAY
```

**Monitoring Capabilities:**
- Real-time resource metrics (CPU, memory, load)
- Task execution tracking
- Worker availability status
- Heartbeat monitoring (1-second intervals)
- Historical metric storage
- Alert thresholds

## Technology Stack

### Backend
- **Language**: Go 1.x
- **Web Framework**: Fiber (FastHTTP-based)
- **GraphQL**: gqlgen (schema-first)
- **Database ORM**: GORM
- **Message Queue**: NATS
- **Cache**: Redis
- **Authentication**: JWT
- **Scheduler**: gocron

### Frontend
- **Framework**: React
- **Communication**: GraphQL + WebSocket
- **Build**: Embedded static files

### Infrastructure
- **Database**: PostgreSQL with TimescaleDB
- **Cache**: Redis 7
- **Message Queue**: NATS 2.9
- **Containers**: Docker

### Worker Runtimes
- Python (primary)
- Bash
- Custom language support

## Deployment Architecture

```mermaid
graph TB
    subgraph "Docker Compose Deployment"
        MAIN[Main App Container<br/>Port 9001]
        WDEV[Worker Dev Container<br/>Replicas: N]
        WPROD[Worker Prod Container<br/>Replicas: N]

        PG[(PostgreSQL<br/>TimescaleDB)]
        REDIS[(Redis)]
        NATS[NATS Server]
    end

    subgraph "External Access"
        USER[Users]
        API_CLIENT[API Clients]
    end

    USER -->|HTTP/WS| MAIN
    API_CLIENT -->|REST API| MAIN

    MAIN <--> PG
    MAIN <--> REDIS
    MAIN <--> NATS

    WDEV <--> PG
    WDEV <--> NATS

    WPROD <--> PG
    WPROD <--> NATS
```

**Deployment Options:**
- **Docker Compose**: Quick start and development
- **Kubernetes**: Production scalability
- **Distributed Mode**: Multi-node deployment
- **High Availability**: Multiple replicas per component

## Interactive Flow: Creating and Running a Pipeline

```mermaid
sequenceDiagram
    actor User
    participant UI as Frontend
    participant API as Main App
    participant DB as PostgreSQL
    participant MQ as NATS
    participant W as Worker
    participant DFS as File System

    Note over User,DFS: Pipeline Creation Phase
    User->>UI: Design Pipeline (Drag & Drop)
    UI->>API: GraphQL: createPipeline
    API->>DB: Save Pipeline Definition
    API->>DFS: Store Code Files
    DB-->>API: Pipeline ID
    API-->>UI: Success
    UI-->>User: Pipeline Created

    Note over User,DFS: Deployment Phase
    User->>UI: Deploy Pipeline
    UI->>API: GraphQL: deployPipeline
    API->>DB: Create Deployment Record
    API->>MQ: Publish: distributed-storage
    MQ->>W: Invalidate Cache
    W->>DFS: Fetch New Files
    DFS-->>W: Code Files
    API-->>UI: Deployed

    Note over User,DFS: Execution Phase
    User->>UI: Run Pipeline
    UI->>API: GraphQL: runPipeline
    API->>DB: Create Pipeline Run
    API->>DB: Create Tasks
    API->>MQ: Publish: pipeline-run-next

    loop For Each Task
        MQ->>API: Process Next Task
        API->>DB: Check Dependencies
        API->>MQ: Queue Task (taskqueue.*)
        MQ->>W: Deliver Task
        W->>W: Load Secrets
        W->>W: Execute Code

        loop During Execution
            W->>MQ: Publish: taskupdate.*
            MQ->>API: Forward Update
            API->>UI: WebSocket: Status
            UI->>User: Live Progress
        end

        W->>DB: Save Task Results
        W->>MQ: Task Complete
    end

    API->>DB: Mark Run Complete
    API->>UI: WebSocket: Complete
    UI-->>User: Pipeline Finished
```

## Key Design Patterns

### 1. Event-Driven Architecture
- NATS pub/sub for asynchronous communication
- Real-time WebSocket updates
- Decoupled component interaction

### 2. Distributed Task Queue
- NATS-based task distribution
- Load balancing across workers
- Automatic retry and failure handling

### 3. Caching Strategy
- Redis for API-level caching
- Worker-level file caching
- Distributed cache invalidation

### 4. Multi-Tenancy
- Environment-based isolation
- Resource-level permissions
- Separate deployment spaces

### 5. Microservices Communication
- Message queue for async operations
- REST/GraphQL for synchronous calls
- WebSocket for real-time data

## Scalability Considerations

### Horizontal Scaling
- **Workers**: Add more worker containers/VMs
- **Main App**: Multiple instances with load balancer
- **Database**: PostgreSQL replication
- **NATS**: NATS clustering

### Vertical Scaling
- Increase worker resources for heavy workloads
- Scale database for larger datasets
- Expand Redis cache size

### Performance Optimization
- Worker file caching reduces I/O
- Redis caching reduces database load
- NATS messaging enables async processing
- Compiled Go code for high performance

## Monitoring and Observability

### Metrics Collected
- Worker CPU, memory, load
- Task execution times
- Pipeline success/failure rates
- Active connections
- Message queue depth

### Logging
- Structured logging with levels
- Real-time log streaming via WebSocket
- Log retention policies
- Secret redaction in logs

### Health Checks
- Worker heartbeat monitoring
- Database connection health
- NATS connectivity
- Redis availability

## Conclusion

Dataplane implements a robust, scalable architecture for data pipeline orchestration. Key strengths include:

- **Performance**: Go-based for low latency and high throughput
- **Scalability**: Distributed worker architecture with horizontal scaling
- **Security**: Multi-layered authentication and encrypted secrets
- **Real-time**: WebSocket and NATS for live updates
- **Flexibility**: Multi-environment support and custom worker groups
- **Reliability**: Comprehensive health monitoring and error handling

The architecture supports enterprise requirements including multi-tenancy, role-based access control, audit logging, and high availability deployment patterns.
