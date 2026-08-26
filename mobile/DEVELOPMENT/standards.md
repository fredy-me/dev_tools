# Mobile Coding Standards

## General Principles

```yaml
principles:
  - clean_code: "Code should be readable, not clever"
  - single_responsibility: "Each module does one thing well"
  - dependency_inversion: "Depend on abstractions, not concretions"
  - dry: "Don't repeat yourself, but prefer duplication over wrong abstraction"
  - yagni: "You ain't gonna need it - build only what's needed now"
  - kiss: "Keep it simple, stupid"
```

## Project Structure

### iOS (Swift)

```
MyApp/
├── App/
│   ├── AppDelegate.swift
│   ├── SceneDelegate.swift
│   └── AppDependencies.swift
├── Features/
│   ├── Auth/
│   │   ├── Views/
│   │   ├── ViewModels/
│   │   ├── Models/
│   │   └── Services/
│   ├── Home/
│   └── Profile/
├── Core/
│   ├── Networking/
│   ├── Storage/
│   ├── Security/
│   └── Extensions/
├── Shared/
│   ├── Components/
│   ├── Helpers/
│   └── Resources/
├── Resources/
│   ├── Assets.xcassets
│   ├── Localizable.xcstrings
│   └── LaunchScreen.storyboard
└── Tests/
    ├── UnitTests/
    ├── IntegrationTests/
    └── UITests/
```

### Android (Kotlin)

```
com.example.myapp/
├── di/                          # Dependency Injection
│   └── AppModule.kt
├── data/                        # Data Layer
│   ├── local/
│   │   ├── dao/
│   │   ├── entity/
│   │   └── database/
│   ├── remote/
│   │   ├── api/
│   │   ├── dto/
│   │   └── interceptor/
│   ├── repository/
│   └── mapper/
├── domain/                      # Domain Layer
│   ├── model/
│   ├── repository/
│   ├── usecase/
│   └── util/
├── presentation/                # Presentation Layer
│   ├── home/
│   │   ├── HomeScreen.kt
│   │   ├── HomeViewModel.kt
│   │   └── HomeState.kt
│   ├── auth/
│   └── settings/
├── util/
│   ├── extension/
│   └── constant/
└── MyApp.kt                    # Application class
```

## Naming Conventions

```yaml
naming:
  files:
    ios:
      views: "{FeatureName}View.swift"
      view_models: "{FeatureName}ViewModel.swift"
      services: "{ServiceName}Service.swift"
      models: "{ModelName}.swift"
      extensions: "{Type}+{Feature}.swift"
      
    android:
      screens: "{FeatureName}Screen.kt"
      view_models: "{FeatureName}ViewModel.kt"
      repositories: "{Name}Repository.kt"
      use_cases: "{Action}UseCase.kt"
      models: "{ModelName}.kt"
      
    react_native:
      components: "{ComponentName}.tsx"
      screens: "{ScreenName}Screen.tsx"
      hooks: "use{HookName}.ts"
      services: "{serviceName}.ts"
      
    flutter:
      screens: "{feature_name}_screen.dart"
      widgets: "{widget_name}.dart"
      models: "{model_name}.dart"
      services: "{service_name}.dart"

  classes:
    format: PascalCase
    examples:
      - UserRepository
      - LoginViewModel
      - NetworkService
      
  methods:
    format: camelCase
    prefixes:
      - get: "retrieve a value"
      - set: "update a value"
      - is/has/can: "boolean check"
      - fetch: "async network request"
      - handle: "event handler"
    examples:
      - getUserProfile()
      - isUserLoggedIn()
      - fetchNotifications()
      - handleDeepLink()
      
  variables:
    format: camelCase
    constants: SCREAMING_SNAKE_CASE
    examples:
      - userProfile
      - MAX_RETRY_COUNT
      - isLoading
      
  files:
    format: snake_case (files), PascalCase (classes)
    examples:
      - user_repository.dart
      - UserRepository.kt
      - user_repository.swift
```

## Code Style

### Swift

```swift
// MARK: - Types

/// Manages user authentication and token lifecycle.
final class AuthManager {
    
    // MARK: - Properties
    
    private let keychain: KeychainService
    private let apiClient: APIClient
    
    // MARK: - Initialization
    
    init(keychain: KeychainService, apiClient: APIClient) {
        self.keychain = keychain
        self.apiClient = apiClient
    }
    
    // MARK: - Public Methods
    
    /// Authenticates user with email and password.
    /// - Parameters:
    ///   - email: User's email address.
    ///   - password: User's password (min 8 characters).
    /// - Returns: Authentication token pair.
    func login(email: String, password: String) async throws -> TokenPair {
        let request = LoginRequest(email: email, password: password)
        let response: LoginResponse = try await apiClient.request(.login(request))
        
        try keychain.store(tokens: response.tokens)
        return response.tokens
    }
}

// MARK: - Extensions

extension AuthManager: AuthManagerProtocol {
    // Protocol conformance
}
```

### Kotlin

```kotlin
/**
 * Manages user authentication and token lifecycle.
 */
class AuthManager @Inject constructor(
    private val keychain: KeychainService,
    private val apiClient: APIClient
) {
    
    // region Public Methods
    
    /**
     * Authenticates user with email and password.
     *
     * @param email User's email address.
     * @param password User's password (min 8 characters).
     * @return Authentication token pair.
     * @throws AuthException If credentials are invalid.
     */
    suspend fun login(
        email: String,
        password: String
    ): TokenPair {
        val request = LoginRequest(email = email, password = password)
        val response = apiClient.login(request)
        
        keychain.store(response.tokens)
        return response.tokens
    }
    
    // endregion
}
```

### TypeScript/React Native

```typescript
/**
 * Manages user authentication and token lifecycle.
 */
export class AuthManager {
  constructor(
    private readonly keychain: KeychainService,
    private readonly apiClient: APIClient
  ) {}

  /**
   * Authenticates user with email and password.
   * @param email - User's email address.
   * @param password - User's password (min 8 characters).
   * @returns Authentication token pair.
   */
  async login(email: string, password: string): Promise<TokenPair> {
    const request: LoginRequest = { email, password };
    const response = await this.apiClient.login(request);
    
    await this.keychain.store(response.tokens);
    return response.tokens;
  }
}
```

## Error Handling

```yaml
error_handling:
  principles:
    - fail_fast_with_clear_messages
    - handle_errors_at_appropriate_level
    - never_silently_swallow_errors
    - log_all_errors_for_debugging
    - provide_user_facing_error_messages
    
  error_types:
    network:
      timeout: "Connection timed out. Please try again."
      no_internet: "No internet connection. Please check your network."
      server_error: "Something went wrong. Please try again later."
      
    auth:
      invalid_credentials: "Invalid email or password."
      account_locked: "Account locked. Please reset your password."
      session_expired: "Session expired. Please log in again."
      
    validation:
      required_field: "This field is required."
      invalid_email: "Please enter a valid email address."
      password_weak: "Password must be at least 8 characters."
      
    permission:
      denied: "Permission denied. Please enable in Settings."
      restricted: "This feature is not available in your region."
```

## Performance Guidelines

```yaml
performance:
  image_loading:
    - use_async_image_loading
    - implement_image_caching
    - provide_placeholders
    - compress_for_display_size
    - use_webp_format_when_possible
    
  list_performance:
    - implement_lazy_loading
    - use_cell_reuse_patterns
    - limit_initial_load_count
    - prefetch_visible_area
    - optimize_list_item_height
    
  network:
    - implement_request_caching
    - compress_api_responses
    - use_websockets_for_realtime
    - batch_api_requests
    - implement_offline_queue
    
  memory:
    - release_unused_resources
    - avoid_retain_cycles
    - use_weak_references
    - profile_memory_usage_regularly
    
  startup:
    - defer_non_critical_initialization
    - use_splash_screen
    - implement_lazy_loading
    - optimize_app_size
```

## Git Conventions

```yaml
branching:
  strategy: git_flow
  branches:
    main: "production-ready code"
    develop: "integration branch"
    feature/*: "new features"
    bugfix/*: "bug fixes"
    hotfix/*: "production fixes"
    release/*: "release preparation"

commits:
  format: conventional_commits
  types:
    - feat: "new feature"
    - fix: "bug fix"
    - docs: "documentation"
    - style: "formatting, no code change"
    - refactor: "code restructuring"
    - perf: "performance improvement"
    - test: "adding tests"
    - chore: "maintenance"
  examples:
    - "feat(auth): add biometric login support"
    - "fix(profile): resolve image upload crash"
    - "perf(list): implement virtual scrolling"

pull_requests:
  template: |
    ## Description
    [What this PR does]
    
    ## Changes
    - [Change 1]
    - [Change 2]
    
    ## Testing
    - [ ] Unit tests pass
    - [ ] UI tests pass
    - [ ] Manual testing completed
    
    ## Screenshots
    [If applicable]
    
  requirements:
    - at_least_one_reviewer
    - all_ci_checks_pass
    - no_conflicts_with_main
    - updated_documentation_if_needed
```

## Configuration

[CONFIGURE] Update for your project:
- Primary language and framework
- Naming convention preferences
- Code style (SwiftLint rules, ktlint, ESLint)
- Git branching strategy
- PR review requirements
- Performance budgets
