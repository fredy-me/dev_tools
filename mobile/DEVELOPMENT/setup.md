# Mobile Development Environment Setup

## Prerequisites

### macOS (Required for iOS Development)

```bash
# Install Homebrew
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install essential tools
brew install --cask xcode
brew install --cask android-studio
brew install node
brew install --cask java  # JDK 17+
brew install git
brew install fastlane
brew install cocoapods  # or use SPM
```

### Cross-Platform Tools

```bash
# Node.js & package manager
brew install node
npm install -g yarn

# React Native CLI
npm install -g react-native-cli
npm install -g expo-cli  # if using Expo

# Flutter
brew install --cask flutter
flutter doctor  # verify installation

# Kotlin Multiplatform
brew install kotlin

# Dart
brew install dart
```

## iOS Development Setup

### Xcode Configuration

```bash
# Verify Xcode installation
xcode-select -p
# Should output: /Applications/Xcode.app/Contents/Developer

# Accept license
sudo xcodebuild -license accept

# Install command line tools
xcode-select --install

# Verify iOS simulators
xcrun simctl list devices
```

### iOS Project Setup

```bash
# Using Swift Package Manager (recommended)
# No additional setup needed, SPM is integrated

# Using CocoaPods (legacy projects)
pod init
pod install
open *.xcworkspace
```

### iOS Certificates & Profiles

```yaml
certificate_setup:
  development:
    - create_apple_id_at_developer_apple_com
    - generate_certificate_signing_request_csr
    - create_development_certificate
    - register_device_udids
    - create_development_provisioning_profile
    
  distribution:
    - create_distribution_certificate
    - create_distribution_provisioning_profile
    - app_store_connect_api_key_for_ci
    
  recommended:
    - use_automatic_signing_in_xcode
    - let_xcode_manage_profiles
    - store_certificates_in_keychain_for_ci
    - use_fastlane_match_for_teams
```

## Android Development Setup

### Android Studio Configuration

```bash
# Install Android Studio
brew install --cask android-studio

# Install Android SDK components
# Via Android Studio SDK Manager:
# - Android API 34 (latest stable)
# - Android SDK Build-Tools 34.0.0
# - Android SDK Platform-Tools
# - Android Emulator
# - Intel HAXM or Hypervisor Framework

# Verify installation
adb --version
sdkmanager --version
```

### Android SDK & Environment

```bash
# Set environment variables (add to ~/.zshrc or ~/.bashrc)
export ANDROID_HOME=$HOME/Library/Android/sdk
export PATH=$PATH:$ANDROID_HOME/emulator
export PATH=$PATH:$ANDROID_HOME/platform-tools
export PATH=$PATH:$ANDROID_HOME/tools
export PATH=$PATH:$ANDROID_HOME/tools/bin

# Verify
echo $ANDROID_HOME
adb devices
```

### Android Signing

```yaml
keystore_setup:
  debug_keystore:
    location: ~/.android/debug.keystore
    password: android
    alias: androiddebugkey
    
  release_keystore:
    generation: |
      keytool -genkey -v \
        -keystore release.keystore \
        -alias my-key-alias \
        -keyalg RSA \
        -keysize 2048 \
        -validity 10000
    store_in: keystore.properties (gitignored)
    config: |
      storeFile=release.keystore
      storePassword=***
      keyAlias=my-key-alias
      keyPassword=***
```

## React Native Setup

```bash
# Create new project
npx react-native init MyApp --template react-native-template-typescript

# OR with Expo
npx create-expo-app MyApp
cd MyApp
npx expo start

# Install dependencies
npm install

# iOS setup
cd ios && pod install && cd ..

# Run
npx react-native run-ios
npx react-native run-android
```

### React Native Dependencies

```yaml
essential_packages:
  navigation:
    - @react-navigation/native
    - @react-navigation/native-stack
    - react-native-screens
    - react-native-safe-area-context
    
  state_management:
    - @reduxjs/toolkit  # or zustand, mobx
    - react-redux
    
  networking:
    - axios  # or react-query/tanstack-query
    
  storage:
    - @react-native-async-storage/async-storage
    - react-native-keychain  # secure storage
    
  ui:
    - react-native-reanimated
    - react-native-gesture-handler
    
  device:
    - react-native-permissions
    - react-native-device-info
    
  testing:
    - @testing-library/react-native
    - jest
```

## Flutter Setup

```bash
# Verify Flutter installation
flutter doctor

# Create new project
flutter create my_app
cd my_app

# Run
flutter run
flutter run --release
```

### Flutter Dependencies

```yaml
essential_packages:
  state_management:
    - flutter_riverpod  # or bloc, provider
    
  navigation:
    - go_router
    
  networking:
    - dio
    - retrofit  # or chopper
    
  storage:
    - shared_preferences
    - flutter_secure_storage
    - hive  # or isar
    
  ui:
    - flutter_svg
    - shimmer
    - lottie
    
  device:
    - permission_handler
    - device_info_plus
    
  testing:
    - mockito
    - bloc_test
    - golden_toolkit
```

## IDE Configuration

### VS Code

```json
// .vscode/settings.json
{
  "dart.flutterSdkPath": "/path/to/flutter",
  "dart.analyzePaths": ["lib"],
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.fixAll": true,
    "source.organizeImports": true
  },
  "dart.lineLength": 80,
  "files.associations": {
    "*.dart": "dart"
  }
}
```

```json
// .vscode/extensions.json
{
  "recommendations": [
    "Dart-Code.flutter",
    "Dart-Code.dart-code",
    "ms-vscode.vscode-typescript-next",
    "bradlc.vscode-tailwindcss"
  ]
}
```

### Android Studio

```yaml
plugins:
  - Dart
  - Flutter
  - Kotlin
  - ADB Ideas
  - Firebase

settings:
  run_intellij_inspections_on_build: true
  auto_import_on_paste: true
  tab_size: 2
```

## CI/CD Setup

```yaml
# .github/workflows/mobile.yml
name: Mobile CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  ios:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - name: Install dependencies
        run: cd ios && pod install
      - name: Run tests
        run: xcodebuild test -workspace MyApp.xcworkspace -scheme MyApp -destination 'platform=iOS Simulator,name=iPhone 15'
      - name: Build
        run: xcodebuild build -workspace MyApp.xcworkspace -scheme MyApp -configuration Release

  android:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
      - name: Build Debug
        run: ./gradlew assembleDebug
      - name: Run Tests
        run: ./gradlew test
```

## Environment Variables

```bash
# .env.development
API_BASE_URL=https://dev-api.example.com
API_KEY=dev_key_xxx
SENTRY_DSN=https://xxx@sentry.io/xxx
SEGMENT_WRITE_KEY=dev_key

# .env.staging
API_BASE_URL=https://staging-api.example.com
API_KEY=staging_key_xxx
SENTRY_DSN=https://xxx@sentry.io/xxx
SEGMENT_WRITE_KEY=staging_key

# .env.production
API_BASE_URL=https://api.example.com
API_KEY=prod_key_xxx
SENTRY_DSN=https://xxx@sentry.io/xxx
SEGMENT_WRITE_KEY=prod_key
```

## Configuration

[CONFIGURE] Update for your project:
- Target platforms and minimum OS versions
- Development team certificates
- API endpoints per environment
- Analytics and monitoring keys
- CI/CD platform choice
- IDE preference
