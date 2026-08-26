# Detailed Architecture & Data Flows

## Request Lifecycle

```mermaid
sequenceDiagram
    participant C as Client
    participant GW as API Gateway
    participant RL as Rate Limiter
    participant Auth as Auth Service
    participant Cache as Redis Cache
    participant App as Application
    participant Queue as Message Queue
    participant DB as Database
    participant Mon as Monitor

    C->>GW: HTTP Request
    GW->>GW: Parse & Validate Headers
    GW->>RL: Check Rate Limit
    alt Rate Limited
        RL-->>GW: 429 Too Many Requests
        GW-->>C: 429 Response
    else Allowed
        RL-->>GW: OK
        GW->>Auth: Verify Token
        alt Invalid Token
            Auth-->>GW: 401 Unauthorized
            GW-->>C: 401 Response
        else Valid Token
            Auth-->>GW: User Context
            GW->>Cache: Check Cache
            alt Cache Hit
                Cache-->>App: Cached Response
                App-->>C: 200 Response
            else Cache Miss
                GW->>App: Forward Request
                App->>DB: Query Database
                DB-->>App: Result
                App->>Cache: Store Result
                alt Async Work Needed
                    App->>Queue: Enqueue Job
                    App-->>C: 202 Accepted
                else Synchronous
                    App-->>C: 200 Response
                end
            end
        end
    end
    GW->>Mon: Log Request
```

## Service Interaction Matrix

```mermaid
graph TB
    subgraph "Synchronous Calls"
        direction LR
        USER_SVC[User Service] -->|REST/gRPC| ORDER_SVC[Order Service]
        ORDER_SVC -->|REST/gRPC| PAYMENT_SVC[Payment Service]
        PAYMENT_SVC -->|REST/gRPC| NOTIFICATION_SVC[Notification Service]
    end

    subgraph "Asynchronous Events"
        direction TB
        USER_SVC -->|user.created| EB[Event Bus]
        ORDER_SVC -->|order.placed| EB
        PAYMENT_SVC -->|payment.processed| EB
        EB -->|user.created| NOTIFICATION_SVC
        EB -->|order.placed| INVENTORY_SVC[Inventory Service]
        EB -->|payment.completed| ANALYTICS_SVC[Analytics Service]
    end
```

## Database Access Patterns

### Read/Write Separation

```mermaid
graph TB
    subgraph "Application"
        READ_REQ[Read Request]
        WRITE_REQ[Write Request]
    end

    subgraph "Connection Pool"
        R_POOL[Read Pool<br/>max: 20]
        W_POOL[Write Pool<br/>max: 5]
    end

    subgraph "Database Cluster"
        PRIMARY[(Primary<br/>Read/Write)]
        REPLICA1[(Replica 1<br/>Read Only)]
        REPLICA2[(Replica 2<br/>Read Only)]
    end

    READ_REQ --> R_POOL
    WRITE_REQ --> W_POOL
    R_POOL --> REPLICA1
    R_POOL --> REPLICA2
    W_POOL --> PRIMARY
    PRIMARY -->|Replication| REPLICA1
    PRIMARY -->|Replication| REPLICA2
```

### Caching Strategy

```mermaid
graph TB
    CLIENT[Client Request] --> CHECK{Cache<br/>Check}

    CHECK -->|Hit| RETURN[Return Cached]
    CHECK -->|Miss| DB[Query Database]
    DB --> COMPUTE[Compute Response]
    COMPUTE --> SET{Write<br/>Strategy}

    SET -->|Write-Through| CACHE_WT[(Cache + DB<br/>Simultaneous)]
    SET -->|Write-Behind| CACHE_WB[(Cache First<br/>DB Async)]
    SET -->|Cache-Aside| CACHE_CA[(Cache Only<br/>Next Read Cached)]

    CACHE_WT --> DONE[Response Ready]
    CACHE_WB --> DONE
    CACHE_CA --> DONE
    RETURN --> DONE

    style CHECK fill:#ffd,stroke:#333
    style DONE fill:#dfd,stroke:#333
```

### Cache Invalidation Patterns

| Pattern | Description | Use Case |
|---------|-------------|----------|
| TTL-based | Expire after N seconds | Non-critical data |
| Event-based | Invalidate on write event | User profiles |
| Version-based | Cache key includes version | API responses |
| Tag-based | Group invalidation by tag | Related entities |

## Event Processing Flow

```mermaid
graph TB
    subgraph "Event Production"
        SVC[Service] -->|Serialize| PRODUCER[Event Producer]
        PRODUCER -->|Publish| TOPIC[Topic/Queue]
    end

    subgraph "Event Processing"
        TOPIC -->|Dispatch| CONSUMER1[Consumer Group 1]
        TOPIC -->|Dispatch| CONSUMER2[Consumer Group 2]
        TOPIC -->|Dispatch| CONSUMER3[Consumer Group 3]

        CONSUMER1 -->|Process| HANDLER1[Handler A]
        CONSUMER2 -->|Process| HANDLER2[Handler B]
        CONSUMER3 -->|Process| HANDLER3[Handler C]
    end

    subgraph "Dead Letter Queue"
        HANDLER1 -->|Failure| DLQ[DLQ]
        HANDLER2 -->|Failure| DLQ
        HANDLER3 -->|Failure| DLQ
        DLQ -->|Retry| RETRY[Retry Logic]
        RETRY --> TOPIC
    end

    HANDLER1 --> STORE1[(Store A)]
    HANDLER2 --> STORE2[(Store B)]
    HANDLER3 --> STORE3[(Store C)]
```

## Error Propagation Flow

```mermaid
graph TB
    ERR[Error Origin] --> CLASSIFY{Error<br/>Classification}

    CLASSIFY -->|Transient| RETRY[Retry Strategy<br/>Exponential Backoff]
    CLASSIFY -->|Permanent| FAIL[Return Error<br/>to Client]
    CLASSIFY -->|Degraded| FALLBACK[Fallback<br/>Response]

    RETRY --> MAX{Max Retries<br/>Reached?}
    MAX -->|No| ERR
    MAX -->|Yes| CIRCUIT[Circuit Breaker<br/>Open]
    CIRCUIT --> FALLBACK

    FALLBACK --> LOG[Log Error]
    FAIL --> LOG
    CIRCUIT --> LOG

    LOG --> MONITOR[Send to<br/>Monitoring]
    LOG --> ALERT{Severity<br/>Check}
    ALERT -->|Critical| PAGE[Page On-Call]
    ALERT -->|Warning| TICKET[Create Ticket]
```

## Circuit Breaker States

```mermaid
stateDiagram-v2
    [*] --> Closed: Initial State

    Closed --> Open: Failure Threshold Exceeded
    Closed --> Closed: Request Success

    Open --> HalfOpen: Timeout Expires
    Open --> Open: Request Rejected

    HalfOpen --> Closed: Probe Request Success
    HalfOpen --> Open: Probe Request Failure
```

### Circuit Breaker Configuration

```yaml
circuit_breaker:
  failure_threshold: 5       # failures before opening
  success_threshold: 3       # successes before closing
  timeout: 30s              # time before half-open
  half_open_max_calls: 10   # probe requests in half-open
  monitoring_window: 60s    # window for failure counting
```

## Data Flow: Order Processing

```mermaid
graph TB
    START[Order Received] --> VALIDATE[Validate Order]
    VALIDATE -->|Invalid| REJECT[Return 400]
    VALIDATE -->|Valid| CHECK_INV{Check<br/>Inventory}

    CHECK_INV -->|Available| RESERVE[Reserve Inventory]
    CHECK_INV -->|Unavailable| NOTIFY_UNAV[Notify: Out of Stock]

    RESERVE --> CHARGE[Process Payment]
    CHARGE -->|Success| CONFIRM[Confirm Order]
    CHARGE -->|Failure| RELEASE[Release Inventory]

    CONFIRM --> SEND_EVENT[Send order.confirmed Event]
    SEND_EVENT --> NOTIFY[Send Confirmation Email]
    SEND_EVENT --> ANALYTICS[Update Analytics]
    SEND_EVENT --> INVENTORY_ADJ[Adjust Inventory Counts]

    RELEASE --> SEND_FAIL[Send order.failed Event]
    SEND_FAIL --> NOTIFY_FAIL[Notify Customer of Failure]

    NOTIFY_UNAV --> DONE[Order Complete]
    NOTIFY --> DONE
    ANALYTICS --> DONE
    INVENTORY_ADJ --> DONE
    NOTIFY_FAIL --> DONE

    style START fill:#4CAF50,color:white
    style REJECT fill:#f44336,color:white
    style DONE fill:#2196F3,color:white
```

## Service Mesh Configuration

```mermaid
graph TB
    subgraph "Service Mesh (Istio/Linkerd)"
        subgraph "Sidecar Proxies"
            SP1[Envoy Proxy 1]
            SP2[Envoy Proxy 2]
            SP3[Envoy Proxy 3]
        end

        subgraph "Control Plane"
            CP[Service Mesh Control]
            MTLS[mTLS Certificate Manager]
            TRAFFIC[Traffic Manager]
        end

        SVC1[Service A] <--> SP1
        SVC2[Service B] <--> SP2
        SVC3[Service C] <--> SP3

        SP1 <-->|mTLS| SP2
        SP2 <-->|mTLS| SP3
        SP1 <-->|mTLS| SP3

        CP --> SP1
        CP --> SP2
        CP --> SP3
        MTLS --> SP1
        MTLS --> SP2
        MTLS --> SP3
    end
```

## Performance Optimization Layers

```mermaid
graph TB
    subgraph "Layer 1: CDN / Edge"
        CDN[CloudFront/Cloudflare]
        EDGE[Edge Functions]
    end

    subgraph "Layer 2: API Gateway"
        GW_CACHE[Response Caching]
        GW_COMPRESS[Compression]
        GW_THROTTLE[Request Throttling]
    end

    subgraph "Layer 3: Application"
        APP_CACHE[In-Memory Cache]
        APP_POOL[Connection Pooling]
        APP_BATCH[Batch Processing]
    end

    subgraph "Layer 4: Database"
        DB_INDEX[Indexing]
        DB_PART[Partitioning]
        DB_READ_REPLICA[Read Replicas]
    end

    CDN --> GW_CACHE
    GW_CACHE --> APP_CACHE
    APP_CACHE --> DB_INDEX
```
