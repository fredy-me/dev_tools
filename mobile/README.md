# Mobile Development Template

Comprehensive template for mobile application development across iOS, Android, and cross-platform frameworks.

## Directory Structure

```
mobile/
├── ARCHITECTURE/          # System architecture & design
│   ├── high-level.md      # High-level architecture diagrams
│   ├── detailed.md        # Component-level architecture
│   └── integration.md     # API contracts & service boundaries
├── SECURITY/              # Security patterns & compliance
│   ├── authentication.md  # Auth patterns (OAuth, biometric, etc.)
│   ├── authorization.md   # Access control & permissions
│   ├── compliance.md      # GDPR, CCPA, mobile compliance
│   └── best-practices.md  # OWASP Mobile Top 10 & checklists
├── DESIGN/                # UI/UX guidelines
│   ├── ui-guidelines.md   # Platform-specific UX principles
│   ├── design-system.md   # Tokens, components, theming
│   └── accessibility.md   # TalkBack, VoiceOver, a11y
├── DEVELOPMENT/           # Dev setup & workflow
│   ├── setup.md           # Environment configuration
│   ├── standards.md       # Coding standards & conventions
│   ├── testing.md         # Testing strategy
│   └── deployment.md      # CI/CD & app store deployment
├── AI_DEVELOPMENT/        # AI agent instructions
│   ├── AGENTS.md          # AI agent guidelines for mobile
│   ├── code-review.md     # AI-assisted code review rules
│   ├── testing-strategy.md# AI testing approach
│   └── documentation.md   # Documentation standards
├── TEMPLATES/             # Reusable templates
│   ├── project-structure.md # Project scaffolding
│   └── checklist.md       # Development checklist
└── EXAMPLES/              # Reference implementations
    ├── sample-project.md  # Example customized template
    └── common-patterns.md # Proven mobile patterns
```

## Platform Support

| Platform | Frameworks Covered |
|----------|-------------------|
| iOS | Swift/SwiftUI, Objective-C, UIKit |
| Android | Kotlin/Jetpack Compose, Java, XML |
| Cross-Platform | React Native, Flutter, Xamarin, Kotlin Multiplatform |
| Web-Adjacent | PWA, Capacitor, Ionic |

## Quick Start

1. Review `ARCHITECTURE/high-level.md` to plan your app structure
2. Configure your project using `TEMPLATES/project-structure.md`
3. Set up your environment with `DEVELOPMENT/setup.md`
4. Follow `SECURITY/best-practices.md` for security baseline
5. Use `TEMPLATES/checklist.md` to track progress

## Customization

This template is designed to be customized. Search for `[CONFIGURE]` markers throughout the documents to find sections requiring project-specific configuration.

## Performance Targets

| Metric | Target | Tool |
|--------|--------|------|
| App Launch (Cold) | < 2s | Xcode Instruments, Android Profiler |
| Screen Transition | < 300ms | Lottie, Reanimated |
| Crash Rate | < 0.1% | Firebase Crashlytics, Sentry |
| ANR Rate (Android) | < 0.5% | Google Play Console |
| Memory Usage | < 200MB baseline | Instruments, Android Studio |
| Battery Impact | Minimal | Battery Historian, Energy Diagnostics |
