# High-Level Backend Architecture

## System Overview

```mermaid
graph TB
    subgraph Clients
        WEB[Web App]
        MOB[Mobile App]
        EXT[Third-Party]
    end

    subgraph "API Gateway"
        GW[API Gateway / Load Balancer]
        RATE[Rate Limiter]
        AUTH[Auth Middleware]
    end

    subgraph "Application Layer"
        direction TB
        SVC1[Service A<br/>User Management]
        SVC2[Service B<br/>Business Logic]
        SVC3[Service C<br/>Notifications]
        SVC4[Service D<br/>Analytics]
    end

    subgraph "Data Layer"
        DB1[(Primary DB<br/>PostgreSQL)]
        DB2[(Cache<br/>Redis)]
        DB3[(Search<br/>Elasticsearch)]
        MQ[Message Queue<br/>RabbitMQ/Kafka]
        OBJ[Object Storage<br/>S3/MinIO]
    end

    subgraph "Infrastructure"
        MON[Monitoring<br/>Prometheus/Grafana]
        LOG[Logging<br/>ELK Stack]
        TRACE[Tracing<br/>Jaeger/Zipkin]
    end

    WEB --> GW
    MOB --> GW
    EXT --> GW
    GW --> RATE
    GW --> AUTH
    AUTH --> SVC1
    AUTH --> SVC2
    AUTH --> SVC3
    AUTH --> SVC4
    SVC1 --> DB1
    SVC1 --> DB2
    SVC2 --> DB1
    SVC2 --> MQ
    SVC3 --> MQ
    SVC3 --> OBJ
    SVC4 --> DB3
    SVC1 -.-> MON
    SVC2 -.-> MON
    SVC3 -.-> LOG
    SVC4 -.-> TRACE
```

## Architecture Patterns

### Monolith Architecture

Best for: Small teams, MVPs, domains with tight coupling.

```mermaid
graph TB
    subgraph "Single Deployable Unit"
        LB[Load Balancer]
        APP[Application Server]
        CACHE[(Cache Layer)]
        DB[(Database)]
    end

    LB --> APP
    APP --> CACHE
    APP --> DB
```

**Characteristics:**
- Single codebase and deployment
- Shared database
- In-process communication
- Simpler DevOps and debugging

### Microservices Architecture

Best for: Large teams, independent scaling, diverse tech stacks.

```mermaid
graph TB
    GW[API Gateway]

    subgraph "Services"
        SVC_A[User Service]
        SVC_B[Order Service]
        SVC_C[Payment Service]
        SVC_D[Inventory Service]
    end

    subgraph "Databases"
        DB_A[(User DB)]
        DB_B[(Order DB)]
        DB_C[(Payment DB)]
        DB_D[(Inventory DB)]
    end

    subgraph "Messaging"
        KAFKA[Event Bus<br/>Kafka]
    end

    GW --> SVC_A
    GW --> SVC_B
    GW --> SVC_C

    SVC_A --> DB_A
    SVC_B --> DB_B
    SVC_C --> DB_C
    SVC_D --> DB_D

    SVC_A --> KAFKA
    SVC_B --> KAFKA
    SVC_C --> KAFKA
    SVC_D --> KAFKA

    KAFKA --> SVC_A
    KAFKA --> SVC_B
    KAFKA --> SVC_D
```

**Characteristics:**
- Each service owns its data
- Inter-service communication via events
- Independent deployment and scaling
- Bounded contexts per domain

### Event-Driven Architecture

Best for: Async workflows, real-time processing, decoupled systems.

```mermaid
graph LR
    PROD[Producers] -->|Events| EB[Event Bus<br/>Kafka/RabbitMQ]
    EB -->|Events| CONS1[Consumer A]
    EB -->|Events| CONS2[Consumer B]
    EB -->|Events| CONS3[Consumer C]

    CONS1 -->|Write| DB1[(Store A)]
    CONS2 -->|Write| DB2[(Store B)]
    CONS3 -->|Write| DB3[(Store C)]

    style EB fill:#f9f,stroke:#333,stroke-width:2px
```

### CQRS Pattern

Best for: Read-heavy systems, complex queries, performance optimization.

```mermaid
graph TB
    CLIENT[Client]

    subgraph "Write Side (Commands)"
        CMD[Command Handler]
        WRITE_DB[(Write DB<br/>Normalized)]
    end

    subgraph "Read Side (Queries)"
        QRY[Query Handler]
        READ_DB[(Read DB<br/>Denormalized)]
        PROJ[Projection Service]
    end

    CLIENT -->|Commands| CMD
    CLIENT -->|Queries| QRY
    CMD --> WRITE_DB
    CMD -->|Events| EB[Event Bus]
    EB --> PROJ
    PROJ --> READ_DB
    QRY --> READ_DB
```

## Scaling Patterns

```mermaid
graph TB
    subgraph "Level 1: Single Server"
        S1[App + DB]
    end

    subgraph "Level 2: Vertical Scaling"
        S2[Powerful Server]
    end

    subgraph "Level 3: Horizontal Scaling"
        LB2[Load Balancer]
        APP1[App Server 1]
        APP2[App Server 2]
        APP3[App Server 3]
        DB2[(Shared DB)]
    end

    subgraph "Level 4: Database Scaling"
        LB3[Load Balancer]
        APP4[App Servers]
        PRIMARY[(Primary DB)]
        REPLICA1[(Read Replica 1)]
        REPLICA2[(Read Replica 2)]
    end

    subgraph "Level 5: Full Distribution"
        GW5[API Gateway]
        SVC集群[Microservices Cluster]
        DB集群[(Distributed DB)]
        CACHE集群[(Cache Cluster)]
        MQ集群[Message Queue Cluster]
    end

    S1 --> S2
    S2 --> S3
    S3 --> S4
    S4 --> S5
```

## Technology Stack Matrix

| Layer | Node.js | Python | Go | Java |
|-------|---------|--------|-----|------|
| **Framework** | NestJS / Express | FastAPI / Django | Gin / Echo | Spring Boot |
| **ORM** | Prisma / TypeORM | SQLAlchemy / Tortoise | GORM / sqlx | Hibernate / JPA |
| **Cache** | ioredis | redis-py | go-redis | Spring Data Redis |
| **Queue** | Bull / BullMQ | Celery / arq | Asynq / machinery | Spring AMQP |
| **Testing** | Jest / Supertest | pytest / httpx | testify / gomock | JUnit / Mockito |
| **Validation** | Zod / Joi | Pydantic / Marshmallow | validator | Bean Validation |

## Decision Matrix

| Factor | Monolith | Microservices | Serverless |
|--------|----------|---------------|------------|
| Team size | 1-10 | 10+ | 1-5 |
| Complexity | Low | High | Medium |
| Deployment | Simple | Complex | Simple |
| Scaling | Vertical | Horizontal | Auto |
| Cost (low traffic) | Low | High | Very Low |
| Cost (high traffic) | Medium | Medium | High |
| Latency | Low | Medium | Variable |
| Debugging | Easy | Hard | Hard |

## Next Steps

- Review [detailed.md](detailed.md) for component-level diagrams
- Review [integration.md](integration.md) for API contracts
- Select pattern based on team size and requirements
