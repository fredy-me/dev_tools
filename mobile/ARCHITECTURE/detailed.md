# Detailed Mobile Architecture

## Component Diagram

```mermaid
graph TB
    subgraph "App Shell"
        ROOT[App Root]
        NAV[Navigation Container]
        PROVIDER[Global Providers]
    end

    subgraph "Feature: Authentication"
        LOGIN[Login Screen]
        REGISTER[Register Screen]
        FORGOT[Forgot Password]
        ONBOARD[Onboarding Flow]
    end

    subgraph "Feature: Home"
        FEED[Feed Screen]
        DETAIL[Detail Screen]
        SEARCH_F[Search Screen]
    end

    subgraph "Feature: Profile"
        PROFILE[Profile Screen]
        SETTINGS[Settings Screen]
        EDIT[Edit Profile]
    end

    subgraph "Shared Module"
        COMPONENTS[UI Components]
        HOOKS[Custom Hooks / Extensions]
        UTILS[Utilities]
        CONSTANTS[Constants]
        MODELS[Data Models]
    end

    subgraph "Core Module"
        API_CORE[API Client]
        AUTH_CORE[Auth Manager]
        STORAGE[Storage Manager]
        PUSH_CORE[Push Manager]
        ANALYTICS_CORE[Analytics]
        CRASH[Crash Reporter]
        FEATURE_FLAG[Feature Flags]
    end

    ROOT --> PROVIDER
    PROVIDER --> NAV
    NAV --> LOGIN
    NAV --> FEED
    NAV --> PROFILE
    LOGIN --> ONBOARD
    FEED --> DETAIL
    PROFILE --> SETTINGS
    FEED --> SEARCH_F
    LOGIN --> AUTH_CORE
    FEED --> API_CORE
    PROFILE --> STORAGE
    DETAIL --> API_CORE
    SETTINGS --> STORAGE
    API_CORE --> AUTH_CORE
    AUTH_CORE --> STORAGE
    API_CORE --> FEATURE_FLAG
    FEED --> ANALYTICS_CORE
    ROOT --> CRASH
```

## State Management Architecture

```mermaid
graph TB
    subgraph "Global State"
        USER_S[User State]
        CONFIG_S[Config State]
        THEME_S[Theme State]
    end

    subgraph "Feature State"
        HOME_S[Home Feature State]
        AUTH_S[Auth Feature State]
        NOTIF_S[Notification State]
    end

    subgraph "UI State"
        FORM_S[Form State]
        MODAL_S[Modal State]
        TOAST_S[Toast State]
    end

    subgraph "Persistence"
        MMKV[MMKV / UserDefaults]
        SECURE[Keychain / Keystore]
        ASYNC_STORE[AsyncStorage / DataStore]
    end

    USER_S --> MMKV
    USER_S --> SECURE
    CONFIG_S --> ASYNC_STORE
    THEME_S --> MMKV
    HOME_S --> API_CORE_S[API Core]
    AUTH_S --> SECURE
```

## Network Layer Architecture

```mermaid
classDiagram
    class APIClient {
        +baseURL: String
        +interceptors: List~Interceptor~
        +request(endpoint) Response
        +upload(endpoint, file) Response
        +download(endpoint) File
    }

    class Interceptor {
        <<interface>>
        +intercept(request) Request
    }

    class AuthInterceptor {
        +tokenProvider: TokenProvider
        +intercept(request) Request
    }

    class RetryInterceptor {
        +maxRetries: Int
        +intercept(request) Request
    }

    class CacheInterceptor {
        +cachePolicy: CachePolicy
        +intercept(request) Request
    }

    class Endpoint {
        +path: String
        +method: HTTPMethod
        +headers: Map
        +body: Encodable?
        +query: Map?
    }

    class Response~T~ {
        +data: T?
        +error: APIError?
        +statusCode: Int
        +headers: Map
    }

    class APIError {
        +code: Int
        +message: String
        +details: Map?
    }

    class TokenProvider {
        <<interface>>
        +getAccessToken() String
        +getRefreshToken() String
        +refreshToken() TokenPair
        +clearTokens()
    }

    APIClient --> Interceptor
    AuthInterceptor ..|> Interceptor
    RetryInterceptor ..|> Interceptor
    CacheInterceptor ..|> Interceptor
    APIClient --> Endpoint
    APIClient ..> Response
    Response ..> APIError
    AuthInterceptor --> TokenProvider
```

## Local Storage Architecture

```mermaid
graph TB
    subgraph "Storage Abstraction"
        STORAGE_MGR[Storage Manager]
    end

    subgraph "Storage Backends"
        SECURE_S[Secure Storage]
        PERSIST_S[Persistent Storage]
        CACHE_S[Cache Storage]
        PREF_S[Preferences Storage]
    end

    subgraph "Platform Implementations"
        KC_S[iOS Keychain]
        KS_S[Android Keystore]

        CD_S[iOS CoreData]
        ROOM_S[Android Room]

        NS_S[iOS NSCache]
        LRU_S[LRU Cache - Both]

        UD_S[iOS UserDefaults]
        DS_S[Android DataStore]

        MMKV_S[MMKV - Cross Platform]
    end

    STORAGE_MGR --> SECURE_S
    STORAGE_MGR --> PERSIST_S
    STORAGE_MGR --> CACHE_S
    STORAGE_MGR --> PREF_S

    SECURE_S --> KC_S
    SECURE_S --> KS_S
    PERSIST_S --> CD_S
    PERSIST_S --> ROOM_S
    CACHE_S --> NS_S
    CACHE_S --> LRU_S
    PREF_S --> UD_S
    PREF_S --> DS_S
    PERSIST_S --> MMKV_S
    PREF_S --> MMKV_S
```

## Push Notification Architecture

```mermaid
sequenceDiagram
    participant Server
    participant APNS as APNs (iOS)
    participant FCM as FCM (Android)
    participant App
    participant Local as Local Notification Manager

    Server->>APNS: Send Push (iOS)
    Server->>FCM: Send Push (Android)
    APNS-->>App: Delivery
    FCM-->>App: Delivery

    App->>App: Parse Notification
    App->>App: Handle Deep Link

    alt App in Foreground
        App->>App: Show In-App Notification
    else App in Background
        App->>Local: Schedule Local Notification
        Local->>App: Show System Notification
    end

    App->>App: Update Badge Count
    App->>App: Sync Unread State
```

## Error Handling Architecture

```mermaid
graph TB
    subgraph "Error Layers"
        NETWORK_E[Network Errors]
        DOMAIN_E[Domain Errors]
        UI_E[UI Errors]
    end

    subgraph "Network Error Types"
        TIMEOUT[TimeoutError]
        CONN[ConnectionError]
        HTTP_4xx[HTTP 4xx Client Error]
        HTTP_5xx[HTTP 5xx Server Error]
        PARSE[ParseError]
    end

    subgraph "Domain Error Types"
        AUTH_E[AuthError]
        VALIDATION_E[ValidationError]
        BIZ_E[BusinessRuleError]
        SYNC_E[SyncError]
    end

    subgraph "UI Error Presentation"
        TOAST[Toast Message]
        SNACK[Snackbar]
        ALERT[Alert Dialog]
        FULLSCREEN[Full Screen Error]
        RETRY_U[Retry with UI]
    end

    NETWORK_E --> TIMEOUT
    NETWORK_E --> CONN
    NETWORK_E --> HTTP_4xx
    NETWORK_E --> HTTP_5xx
    NETWORK_E --> PARSE

    DOMAIN_E --> AUTH_E
    DOMAIN_E --> VALIDATION_E
    DOMAIN_E --> BIZ_E
    DOMAIN_E --> SYNC_E

    TIMEOUT --> RETRY_U
    CONN --> RETRY_U
    HTTP_4xx --> ALERT
    HTTP_5xx --> TOAST
    AUTH_E --> ALERT
    VALIDATION_E --> SNACK
    BIZ_E --> TOAST
```

## Deep Linking Architecture

```mermaid
graph LR
    subgraph "Deep Link Sources"
        EMAIL[Email Link]
        QR[QR Code]
        SOCIAL[Social Share]
        ADS[Ad Campaign]
        WEB[Web Link]
    end

    subgraph "iOS URL Handling"
        UNIVERSAL[Universal Links]
        URL_SCHEME[Custom URL Scheme]
        ASSOCIATED[Associated Domains]
    end

    subgraph "Android URL Handling"
        INTENT_F[Intent Filters]
        APP_LINKS[App Links]
        DYN_LINKS[Dynamic Links]
    end

    subgraph "Router"
        PARSER[Link Parser]
        VALIDATOR[Link Validator]
        ROUTER_M[Router Manager]
    end

    subgraph "Destinations"
        DEEP[Deep Screen]
        FEATURE_R[Feature Entry]
        EXTERNAL[External Redirect]
    end

    EMAIL --> UNIVERSAL
    EMAIL --> INTENT_F
    QR --> URL_SCHEME
    SOCIAL --> APP_LINKS
    ADS --> DYN_LINKS
    WEB --> ASSOCIATED

    UNIVERSAL --> PARSER
    URL_SCHEME --> PARSER
    INTENT_F --> PARSER
    APP_LINKS --> PARSER
    DYN_LINKS --> PARSER

    PARSER --> VALIDATOR
    VALIDATOR --> ROUTER_M
    ROUTER_M --> DEEP
    ROUTER_M --> FEATURE_R
    ROUTER_M --> EXTERNAL
```

## Configuration

[CONFIGURE] Update for your project:
- Feature modules based on your app's domain
- State management solution (Redux, BLoC, MobX, etc.)
- Network layer library choices
- Storage backend selections
- Error handling strategy
- Deep link URL patterns
