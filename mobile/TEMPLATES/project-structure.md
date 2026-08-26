# Mobile Project Structure Template

## Directory Structure

```
{project-name}/
├── .github/
│   ├── workflows/
│   │   ├── ci.yml
│   │   └── release.yml
│   └── PULL_REQUEST_TEMPLATE.md
├── .vscode/
│   ├── settings.json
│   └── extensions.json
├── docs/
│   ├── ARCHITECTURE.md
│   ├── API.md
│   ├── CHANGELOG.md
│   └── screenshots/
├── fastlane/
│   ├── Fastfile
│   ├── Appfile
│   └── Matchfile
│
├── ios/                          # iOS specific
│   ├── MyApp/
│   │   ├── App/
│   │   │   ├── AppDelegate.swift
│   │   │   └── SceneDelegate.swift
│   │   ├── Features/
│   │   ├── Core/
│   │   ├── Shared/
│   │   ├── Resources/
│   │   └── Info.plist
│   ├── MyAppTests/
│   ├── MyAppUITests/
│   ├── MyApp.xcodeproj/
│   └── Podfile                  # If using CocoaPods
│
├── android/                      # Android specific
│   ├── app/
│   │   ├── src/
│   │   │   ├── main/
│   │   │   │   ├── java/com/example/myapp/
│   │   │   │   │   ├── di/
│   │   │   │   │   ├── data/
│   │   │   │   │   ├── domain/
│   │   │   │   │   ├── presentation/
│   │   │   │   │   └── MyApp.kt
│   │   │   │   ├── res/
│   │   │   │   ├── AndroidManifest.xml
│   │   │   │   └── network_security_config.xml
│   │   │   ├── test/
│   │   │   └── androidTest/
│   │   └── build.gradle.kts
│   ├── build.gradle.kts
│   ├── settings.gradle.kts
│   └── gradle.properties
│
├── src/                          # Shared / React Native / Flutter
│   ├── features/
│   │   ├── auth/
│   │   │   ├── components/
│   │   │   ├── screens/
│   │   │   ├── hooks/           # React Native
│   │   │   ├── store/           # State management
│   │   │   ├── types/
│   │   │   └── index.ts
│   │   ├── home/
│   │   └── profile/
│   ├── core/
│   │   ├── api/
│   │   ├── auth/
│   │   ├── storage/
│   │   ├── navigation/
│   │   ├── analytics/
│   │   ├── push/
│   │   └── utils/
│   ├── shared/
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── styles/
│   │   ├── constants/
│   │   └── types/
│   ├── assets/
│   │   ├── images/
│   │   ├── fonts/
│   │   └── animations/
│   ├── config/
│   │   ├── env.ts
│   │   └── feature-flags.ts
│   └── App.tsx                  # Entry point
│
├── __tests__/                    # Shared tests
│   ├── unit/
│   ├── integration/
│   └── fixtures/
│
├── scripts/
│   ├── setup.sh
│   ├── build.sh
│   └── release.sh
│
├── .env.example
├── .gitignore
├── .swiftlint.yml               # iOS
├── README.md
├── package.json                 # If React Native
├── tsconfig.json                # If TypeScript
└── dart_test.yaml               # If Flutter
```

## Feature Module Template

### iOS (Swift)

```
Features/
└── {FeatureName}/
    ├── Views/
    │   ├── {FeatureName}View.swift
    │   ├── {SubView}.swift
    │   └── Components/
    │       └── {Component}.swift
    ├── ViewModels/
    │   └── {FeatureName}ViewModel.swift
    ├── Models/
    │   ├── {FeatureName}.swift
    │   └── {FeatureName}Response.swift
    ├── Services/
    │   └── {FeatureName}Service.swift
    ├── Protocols/
    │   └── {FeatureName}Protocol.swift
    └── {FeatureName}Coordinator.swift
```

### Android (Kotlin)

```
presentation/{feature}/
├── {Feature}Screen.kt
├── {Feature}ViewModel.kt
├── {Feature}State.kt
├── {Feature}Event.kt
├── components/
│   └── {Component}.kt
└── navigation/
    └── {Feature}Navigation.kt

domain/{feature}/
├── model/
│   └── {Feature}Model.kt
├── repository/
│   └── {Feature}Repository.kt
└── usecase/
    ├── Get{Feature}UseCase.kt
    └── Update{Feature}UseCase.kt

data/{feature}/
├── remote/
│   ├── {Feature}Api.kt
│   └── {Feature}Dto.kt
├── local/
│   ├── {Feature}Dao.kt
│   └── {Feature}Entity.kt
└── repository/
    └── {Feature}RepositoryImpl.kt
```

### React Native (TypeScript)

```
src/features/{feature}/
├── screens/
│   ├── {Feature}Screen.tsx
│   └── {Feature}DetailScreen.tsx
├── components/
│   ├── {Component}.tsx
│   └── {Component}.styles.ts
├── hooks/
│   ├── use{Feature}.ts
│   └── use{Feature}Mutations.ts
├── store/
│   ├── {feature}Slice.ts
│   └── {feature}Api.ts
├── types/
│   └── index.ts
├── utils/
│   └── helpers.ts
└── index.ts
```

### Flutter

```
lib/features/{feature}/
├── presentation/
│   ├── screens/
│   │   └── {feature}_screen.dart
│   ├── widgets/
│   │   └── {widget}.dart
│   └── providers/
│       └── {feature}_provider.dart
├── domain/
│   ├── models/
│   │   └── {feature}.dart
│   ├── repositories/
│   │   └── {feature}_repository.dart
│   └── usecases/
│       └── get_{feature}.dart
├── data/
│   ├── datasources/
│   │   ├── {feature}_remote.dart
│   │   └── {feature}_local.dart
│   ├── models/
│   │   └── {feature}_model.dart
│   └── repositories/
│       └── {feature}_repository_impl.dart
└── {feature}_module.dart
```

## Configuration Files

### Environment Configuration

```yaml
# .env.example
# API Configuration
API_BASE_URL=https://api.example.com/v1
API_KEY=your_api_key_here

# Authentication
AUTH_PROVIDER=firebase  # firebase, auth0, custom
OAUTH_CLIENT_ID=your_client_id
OAUTH_REDIRECT_URI=com.example.myapp://callback

# Analytics
SEGMENT_WRITE_KEY=your_segment_key
MIXPANEL_TOKEN=your_mixpanel_token

# Crash Reporting
SENTRY_DSN=your_sentry_dsn
DATADOG_CLIENT_TOKEN=your_datadog_token

# Feature Flags
LAUNCHDARKLY_SDK_KEY=your_ld_key

# Push Notifications
FCM_SERVER_KEY=your_fcm_key  # Android only
```

### Build Configuration

```yaml
# ios/fastlane/Appfile
app_identifier("com.example.myapp")
apple_id("developer@example.com")
team_id("XXXXXXXXXX")
itc_team_id("XXXXXXXXXX")

# android/fastlane/Appfile
json_key_file("path/to/google-play-key.json")
package_name("com.example.myapp")
```

## Getting Started Commands

```bash
# 1. Clone and setup
git clone https://github.com/your-org/your-app.git
cd your-app

# 2. Install dependencies
# iOS
cd ios && pod install && cd ..
# Android
cd android && ./gradlew build && cd ..
# React Native
npm install && cd ios && pod install && cd ..
# Flutter
flutter pub get

# 3. Configure environment
cp .env.example .env
# Edit .env with your values

# 4. Run
# iOS
npx react-native run-ios
# Android
npx react-native run-android
# Flutter
flutter run
```

## Configuration

[CONFIGURE] Update for your project:
- Project name and bundle IDs
- Team IDs and certificates
- Feature modules
- Navigation structure
- Build configurations
- Environment variables
