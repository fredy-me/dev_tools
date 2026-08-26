# Integration Architecture

## API Contract Overview

```mermaid
graph TB
    subgraph "Mobile Client"
        AUTH_M[Auth Module]
        CORE_M[Core Module]
        MEDIA_M[Media Module]
        PUSH_M[Push Module]
    end

    subgraph "Backend Services"
        AUTH_S[Auth Service<br/>/api/v1/auth/*]
        USER_S[User Service<br/>/api/v1/users/*]
        CORE_S[Core Service<br/>/api/v1/core/*]
        MEDIA_S[Media Service<br/>/api/v1/media/*]
        NOTIF_S[Notification Service<br/>/api/v1/notifications/*]
        PAY_S[Payment Service<br/>/api/v1/payments/*]
    end

    subgraph "Message Bus"
        EVENTS[Event Bus / Message Queue]
    end

    AUTH_M -->|JWT Tokens| AUTH_S
    AUTH_M -->|User Profile| USER_S
    CORE_M -->|Business Data| CORE_S
    MEDIA_M -->|Upload/Download| MEDIA_S
    PUSH_M -->|Device Token| NOTIF_S
    CORE_S -->|Payment Events| PAY_S
    NOTIF_S -->|Push Events| EVENTS
    CORE_S -->|Domain Events| EVENTS
```

## REST API Contract

### Authentication Endpoints

```yaml
# POST /api/v1/auth/register
Request:
  body:
    email: string (required)
    password: string (required, min 8 chars)
    name: string (required)
    platform: enum [ios, android, web]
    push_token: string (optional)
    device_id: string (required)
  headers:
    Content-Type: application/json

Response 201:
  body:
    user:
      id: uuid
      email: string
      name: string
      created_at: iso8601
    tokens:
      access_token: jwt
      refresh_token: string
      expires_in: integer (seconds)

Response 409:
  body:
    error:
      code: "USER_EXISTS"
      message: "Email already registered"

# POST /api/v1/auth/login
Request:
  body:
    email: string (required)
    password: string (required)
    platform: enum [ios, android, web]
    push_token: string (optional)

Response 200:
  body:
    user:
      id: uuid
      email: string
      name: string
    tokens:
      access_token: jwt
      refresh_token: string
      expires_in: integer

# POST /api/v1/auth/refresh
Request:
  body:
    refresh_token: string (required)

Response 200:
  body:
    tokens:
      access_token: jwt
      refresh_token: string
      expires_in: integer

# POST /api/v1/auth/logout
Request:
  headers:
    Authorization: Bearer {access_token}
  body:
    push_token: string (optional, for single device logout)

# POST /api/v1/auth/forgot-password
Request:
  body:
    email: string (required)

Response 200:
  body:
    message: "If email exists, reset link sent"
```

### Resource Endpoints

```yaml
# GET /api/v1/resources
Request:
  headers:
    Authorization: Bearer {access_token}
  query:
    page: integer (default: 1)
    per_page: integer (default: 20, max: 100)
    sort: string (default: "created_at")
    order: enum [asc, desc]
    search: string (optional)
    filter[field]: string (optional)

Response 200:
  body:
    data: Resource[]
    meta:
      current_page: integer
      total_pages: integer
      total_count: integer
      per_page: integer
    links:
      self: string
      next: string | null
      prev: string | null

# GET /api/v1/resources/:id
Response 200:
  body:
    data: Resource

Response 404:
  body:
    error:
      code: "NOT_FOUND"
      message: "Resource not found"

# POST /api/v1/resources
Request:
  headers:
    Authorization: Bearer {access_token}
    Content-Type: application/json
  body:
    name: string (required)
    description: string (optional)

Response 201:
  body:
    data: Resource

# PUT /api/v1/resources/:id
# DELETE /api/v1/resources/:id
```

### Standard Error Response

```yaml
Response 4xx/5xx:
  body:
    error:
      code: string (machine-readable)
      message: string (human-readable)
      details: object (optional)
        field_errors:
          - field: string
            message: string
            code: string
      request_id: uuid
      timestamp: iso8601
```

## GraphQL Contract

```graphql
type Query {
  # User queries
  me: User!
  user(id: ID!): User
  
  # Core queries
  resources(filter: ResourceFilter, page: Int, perPage: Int): ResourceConnection!
  resource(id: ID!): Resource
  
  # Search
  search(query: String!, type: SearchType): SearchResult!
}

type Mutation {
  # Auth mutations
  login(input: LoginInput!): AuthPayload!
  register(input: RegisterInput!): AuthPayload!
  refreshToken(refreshToken: String!): AuthPayload!
  
  # Resource mutations
  createResource(input: CreateResourceInput!): Resource!
  updateResource(id: ID!, input: UpdateResourceInput!): Resource!
  deleteResource(id: ID!): Boolean!
}

type Subscription {
  # Real-time updates
  resourceUpdated(id: ID!): Resource!
  newNotification: Notification!
}

type User {
  id: ID!
  email: String!
  name: String!
  avatar: String
  createdAt: DateTime!
  updatedAt: DateTime!
}

type AuthPayload {
  user: User!
  accessToken: String!
  refreshToken: String!
  expiresIn: Int!
}

input LoginInput {
  email: String!
  password: String!
  platform: Platform!
  pushToken: String
}

enum Platform {
  IOS
  ANDROID
  WEB
}
```

## Service Boundaries

```mermaid
graph TB
    subgraph "Auth Boundary"
        A1[Token Management]
        A2[Session Management]
        A3[Password Policy]
        A4[OAuth Providers]
        A5[MFA]
    end

    subgraph "User Boundary"
        U1[Profile Management]
        U2[Preferences]
        U3[Privacy Settings]
        U4[Account Deletion]
    end

    subgraph "Core Business Boundary"
        C1[Domain Logic]
        C2[Workflows]
        C3[Business Rules]
        C4[Validation]
    end

    subgraph "Media Boundary"
        M1[Upload Processing]
        M2[Image Resizing]
        M3[CDN Management]
        M4[Content Moderation]
    end

    subgraph "Notification Boundary"
        N1[Push Delivery]
        N2[Email Delivery]
        N3[SMS Delivery]
        N4[In-App Notifications]
        N5[Notification Preferences]
    end

    A1 -.->|Authenticates| C1
    U1 -.->|Profile Data| C1
    C1 -.->|Media Requests| M1
    C1 -.->|Triggers| N1
    M3 -.->|CDN URLs| C1
```

## Sync Protocol

```mermaid
sequenceDiagram
    participant Client
    participant API
    participant DB

    Note over Client: App opens or comes to foreground
    Client->>API: GET /sync?last_sync={timestamp}&device_id={id}
    
    API->>DB: Query changes since timestamp
    DB-->>API: Changed records
    
    API-->>Client: Sync Response
    Note right of Client: { changes: [...], server_time: timestamp }
    
    loop For each local change
        Client->>API: POST /sync/push
        Note right of Client: { entity: "resource", operation: "upsert", data: {...} }
        
        alt Conflict Detected
            API-->>Client: 409 Conflict
            Note right of Client: { server_version: {...}, conflict: "field_mismatch" }
            Client->>Client: Resolve conflict
            Client->>API: POST /sync/push (with resolution)
        else No Conflict
            API-->>Client: 200 OK
            API->>DB: Persist change
        end
    end
    
    Client->>Client: Update local sync timestamp
```

## WebSocket Events

```yaml
# Connection
WebSocket: wss://api.example.com/ws?token={jwt}

# Client -> Server Events
events:
  - type: "ping"
  - type: "subscribe"
    payload:
      channel: "user:{user_id}"
  - type: "unsubscribe"
    payload:
      channel: "user:{user_id}"

# Server -> Client Events
events:
  - type: "pong"
  - type: "notification"
    payload:
      id: uuid
      type: enum [info, warning, alert]
      title: string
      body: string
      data: object (optional)
      created_at: iso8601
  - type: "resource.updated"
    payload:
      id: uuid
      changes: object
  - type: "presence"
    payload:
      user_id: uuid
      status: enum [online, offline]
```

## SDK Integration Contracts

### Analytics SDK

```typescript
interface AnalyticsSDK {
  identify(userId: string, traits: Record<string, any>): void;
  track(event: string, properties?: Record<string, any>): void;
  screen(name: string, properties?: Record<string, any>): void;
  flush(): Promise<void>;
  reset(): void;
}
```

### Crash Reporting SDK

```typescript
interface CrashReporter {
  setUserId(userId: string): void;
  setCustomKey(key: string, value: string | number | boolean): void;
  log(message: string, level: 'debug' | 'info' | 'warning' | 'error'): void;
  recordError(error: Error, context?: Record<string, any>): void;
  start(): void;
}
```

### Feature Flag SDK

```typescript
interface FeatureFlagSDK {
  isEnabled(flag: string, defaultValue?: boolean): boolean;
  getVariant(flag: string, defaultValue?: string): string;
  getConfiguration<T>(flag: string, defaultValue: T): T;
  onFlagChanged(flag: string, callback: (value: any) => void): () => void;
}
```

## Configuration

[CONFIGURE] Update for your project:
- API base URLs and versioning strategy
- Authentication provider (Firebase, Auth0, custom)
- GraphQL schema if using GraphQL
- Sync strategy based on offline requirements
- WebSocket events for real-time features
- SDK choices for analytics, crash reporting, feature flags
