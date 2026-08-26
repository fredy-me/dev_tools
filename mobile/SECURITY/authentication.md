# Mobile Authentication Patterns

## Authentication Flow Overview

```mermaid
sequenceDiagram
    participant User
    participant App
    participant AuthService
    participant Provider as OAuth Provider
    participant Biometric as Biometric Module
    participant Keychain as Secure Storage

    User->>App: Open App
    App->>Keychain: Check for stored tokens
    
    alt No Tokens
        App->>User: Show Login Screen
        User->>App: Enter credentials
        App->>AuthService: POST /auth/login
        AuthService-->>App: Tokens + User
        App->>Keychain: Store tokens
    else Tokens Exist
        alt Token Valid
            App->>App: Proceed to home
        else Token Expired
            App->>AuthService: POST /auth/refresh
            alt Refresh Success
                AuthService-->>App: New tokens
                App->>Keychain: Update tokens
            else Refresh Failed
                App->>User: Redirect to login
            end
        end
    end
```

## Biometric Authentication

### iOS (Face ID / Touch ID)

```swift
import LocalAuthentication

class BiometricAuthManager {
    func authenticate(reason: String) async throws -> Bool {
        let context = LAContext()
        var error: NSError?
        
        guard context.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, error: &error) else {
            throw BiometricError.notAvailable
        }
        
        let success = try await context.evaluatePolicy(
            .deviceOwnerAuthenticationWithBiometrics,
            localizedReason: reason
        )
        return success
    }
    
    func biometricType() -> BiometricType {
        let context = LAContext()
        var error: NSError?
        guard context.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, error: &error) else {
            return .none
        }
        switch context.biometryType {
        case .faceID: return .faceID
        case .touchID: return .touchID
        case .opticID: return .opticID
        default: return .none
        }
    }
}

enum BiometricType {
    case none, faceID, touchID, opticID
}

enum BiometricError: Error {
    case notAvailable
    case cancelled
    case fallback
    case lockout
    case passcodeRequired
}
```

### Android (BiometricPrompt)

```kotlin
import androidx.biometric.BiometricPrompt
import androidx.fragment.app.FragmentActivity

class BiometricAuthManager(private val activity: FragmentActivity) {
    
    fun authenticate(
        title: String,
        subtitle: String,
        onSuccess: () -> Unit,
        onError: (Int, String) -> Unit,
        onFailed: () -> Unit
    ) {
        val executor = ContextCompat.getMainExecutor(activity)
        
        val callback = object : BiometricPrompt.AuthenticationCallback() {
            override fun onAuthenticationSucceeded(result: BiometricPrompt.AuthenticationResult) {
                super.onAuthenticationSucceeded(result)
                onSuccess()
            }
            
            override fun onAuthenticationError(errorCode: Int, errString: CharSequence) {
                super.onAuthenticationError(errorCode, errString)
                onError(errorCode, errString.toString())
            }
            
            override fun onAuthenticationFailed() {
                super.onAuthenticationFailed()
                onFailed()
            }
        }
        
        val promptInfo = BiometricPrompt.PromptInfo.Builder()
            .setTitle(title)
            .setSubtitle(subtitle)
            .setAllowedAuthenticators(BiometricManager.Authenticators.BIOMETRIC_STRONG)
            .setNegativeButtonText("Use Password")
            .build()
        
        val biometricPrompt = BiometricPrompt(activity, executor, callback)
        biometricPrompt.authenticate(promptInfo)
    }
}
```

## OAuth 2.0 + PKCE Flow

```mermaid
sequenceDiagram
    participant App
    participant AuthService
    participant OAuthProvider

    App->>App: Generate code_verifier & code_challenge
    Note right of App: code_verifier = random(32-96 chars)<br/>code_challenge = SHA256(code_verifier)

    App->>OAuthProvider: Authorization Request
    Note right of App: client_id, redirect_uri,<br/>code_challenge, code_challenge_method=S256,<br/>scope, state

    OAuthProvider->>User: Login & Consent Screen
    User->>OAuthProvider: Approve

    OAuthProvider-->>App: Authorization Code + state
    Note right of App: redirect_uri?code=xxx&state=yyy

    App->>App: Verify state matches
    App->>AuthService: Exchange Code
    Note right of App: POST /auth/oauth/token<br/>grant_type=authorization_code,<br/>code, code_verifier, client_id

    AuthService->>OAuthProvider: Validate code + verifier
    OAuthProvider-->>AuthService: Tokens
    AuthService-->>App: App Tokens (access + refresh)
```

### PKCE Implementation

```typescript
// Crypto utilities for PKCE
async function generatePKCE() {
  const verifier = generateRandomString(64);
  const challenge = await sha256(verifier);
  return {
    codeVerifier: verifier,
    codeChallenge: base64UrlEncode(challenge),
  };
}

function generateRandomString(length: number): string {
  const array = new Uint8Array(length);
  crypto.getRandomValues(array);
  return base64UrlEncode(array);
}

async function sha256(plain: string): Promise<ArrayBuffer> {
  const encoder = new TextEncoder();
  const data = encoder.encode(plain);
  return crypto.subtle.digest('SHA-256', data);
}

// OAuth flow
async function startOAuthFlow(provider: 'google' | 'apple' | 'github') {
  const { codeVerifier, codeChallenge } = await generatePKCE();
  const state = generateRandomString(32);
  
  // Store PKCE values securely
  await secureStore.set('pkce_verifier', codeVerifier);
  await secureStore.set('oauth_state', state);
  
  const params = new URLSearchParams({
    client_id: CONFIG.oauth[provider].clientId,
    redirect_uri: CONFIG.oauth.redirectUri,
    response_type: 'code',
    code_challenge: codeChallenge,
    code_challenge_method: 'S256',
    scope: CONFIG.oauth[provider].scopes.join(' '),
    state,
  });
  
  openBrowser(`${CONFIG.oauth[provider].authUrl}?${params}`);
}
```

## Token Management

```mermaid
graph TB
    subgraph "Token Lifecycle"
        AC[Access Token<br/>Short-lived: 15min<br/>Stored: Memory only]
        RT[Refresh Token<br/>Long-lived: 30 days<br/>Stored: Secure Storage]
        AT[App Token<br/>Medium-lived: 24h<br/>Stored: Secure Storage]
    end

    subgraph "Refresh Strategy"
        EXPIRE[Token Expiring]
        SILENT[Silent Refresh]
        RETRY[Retry Logic]
        REVOKED[Token Revoked]
    end

    subgraph "Security Measures"
        ROTATION[Token Rotation]
        BINDING[Device Binding]
        PINNING[Certificate Pinning]
    end

    EXPIRE --> SILENT
    SILENT -->|Success| AC
    SILENT -->|Fail| RETRY
    RETRY -->|Max retries| REVOKED
    REVOKED --> Login[Re-authenticate]
    ROTATION --> RT
    BINDING --> AC
    BINDING --> RT
```

### Token Storage Rules

```yaml
access_token:
  storage: In-memory only
  transmission: Authorization header
  lifetime: 900 seconds (15 min)
  refresh: Silent refresh before expiry

refresh_token:
  storage: Keychain (iOS) / EncryptedSharedPreferences (Android)
  transmission: POST body only
  lifetime: 2592000 seconds (30 days)
  refresh: Rotated on each use
  revocation: On logout, revoke server-side

app_token:
  storage: Keychain / EncryptedSharedPreferences
  transmission: Authorization header
  lifetime: 86400 seconds (24 hours)
  refresh: On app foreground if expired
```

## Session Management

```mermaid
stateDiagram-v2
    [*] --> Unauthenticated

    Unauthenticated --> Authenticating: User submits credentials
    Authenticating --> Authenticated: Success
    Authenticating --> Unauthenticated: Failure

    Authenticated --> Foreground: App becomes active
    Foreground --> Validating: Check token validity
    Validating --> Foreground: Token valid
    Validating --> Refreshing: Token expired
    Refreshing --> Foreground: Refresh success
    Refreshing --> Unauthenticated: Refresh failed

    Foreground --> Background: App goes to background
    Background --> Foreground: App resumes (< 5 min)
    Background --> Locked: Inactive > 5 min
    Locked --> Unauthenticated: Sensitive action required

    Authenticated --> LoggedOut: User logout
    LoggedOut --> Unauthenticated: Cleanup complete
```

## MFA Implementation

```yaml
mfa_methods:
  sms:
    description: SMS verification code
    setup: POST /auth/mfa/sms/setup
    verify: POST /auth/mfa/sms/verify
    backup: true
    
  totp:
    description: Time-based one-time password (Google Authenticator)
    setup: POST /auth/mfa/totp/setup
      response: { secret, qr_code_url, recovery_codes }
    verify: POST /auth/mfa/totp/verify
    
  push:
    description: Push notification approval
    setup: POST /auth/mfa/push/setup
    verify: POST /auth/mfa/push/verify
    
  biometric:
    description: Biometric re-verification
    verify: Local biometric prompt + server verification

mfa_flow:
  1: User logs in with email/password
  2: Server requires MFA
  3: App prompts for MFA method
  4: User provides MFA
  5: Server issues full tokens
  6: App stores tokens
```

## Passwordless Authentication

```mermaid
sequenceDiagram
    participant User
    participant App
    participant AuthService

    User->>App: Enter email
    App->>AuthService: POST /auth/magic-link
    AuthService->>User: Send magic link email
    AuthService-->>App: Check email for link
    
    User->>User: Click email link
    User->>App: App opens via deep link
    Note right of App: myapp://auth/verify?token=xxx
    
    App->>AuthService: POST /auth/verify-magic-link
    AuthService-->>App: Tokens
    App->>App: Store tokens, navigate home
```

## Configuration

[CONFIGURE] Update for your project:
- OAuth providers (Google, Apple, GitHub, etc.)
- Biometric requirements (optional vs required)
- Token lifetimes based on security needs
- MFA methods supported
- Session timeout policies
- Password policy requirements
