# Mobile Development Checklist

## Project Setup

- [ ] Project repository created
- [ ] Branch protection rules configured
- [ ] README with setup instructions
- [ ] .gitignore configured for mobile
- [ ] Environment variables template (.env.example)
- [ ] CI/CD pipeline configured
- [ ] Code signing configured
- [ ] App icons and splash screens

## Architecture

- [ ] Architecture pattern chosen (MVVM, MVI, Clean)
- [ ] Directory structure defined
- [ ] State management solution selected
- [ ] Navigation system configured
- [ ] Dependency injection implemented
- [ ] Error handling strategy defined
- [ ] Logging framework integrated
- [ ] Analytics framework integrated

## Security

- [ ] Secure storage implemented (Keychain/Keystore)
- [ ] Certificate pinning configured
- [ ] OAuth/PKCE flow implemented
- [ ] Token management implemented
- [ ] Biometric authentication configured
- [ ] Jailbreak/root detection (if needed)
- [ ] Input validation on all endpoints
- [ ] No secrets in source code
- [ ] Network security configuration
- [ ] OWASP Mobile Top 10 reviewed

## UI/UX

- [ ] Design system implemented
- [ ] Dark mode support
- [ ] Dynamic type / font scaling
- [ ] Platform-specific navigation patterns
- [ ] Loading states for all screens
- [ ] Error states for all screens
- [ ] Empty states for all screens
- [ ] Pull-to-refresh where appropriate
- [ ] Keyboard handling (dismiss, avoid overlap)
- [ ] Haptic feedback (where appropriate)
- [ ] Animations (smooth, non-blocking)

## Accessibility

- [ ] All images have alt text
- [ ] All buttons have labels
- [ ] All form fields have labels
- [ ] Touch targets ≥ 44pt/48dp
- [ ] Color contrast ≥ 4.5:1
- [ ] Screen reader tested (VoiceOver/TalkBack)
- [ ] Dynamic text size tested
- [ ] Reduce motion respected
- [ ] Focus order is logical
- [ ] No information conveyed by color alone

## Performance

- [ ] App launch < 2 seconds (cold)
- [ ] Screen transitions < 300ms
- [ ] Lists use lazy loading
- [ ] Images properly sized and cached
- [ ] Network responses cached
- [ ] Memory usage monitored
- [ ] No blocking main thread operations
- [ ] App size optimized
- [ ] Startup time optimized
- [ ] Battery impact minimized

## Testing

- [ ] Unit tests (80%+ coverage)
- [ ] Integration tests
- [ ] UI/E2E tests for critical paths
- [ ] Performance tests
- [ ] Accessibility tests
- [ ] Security testing completed
- [ ] Edge cases tested
- [ ] Offline mode tested
- [ ] Error scenarios tested

## Data & Privacy

- [ ] Privacy policy published
- [ ] Data collection documented
- [ ] Consent management implemented
- [ ] Data retention policy defined
- [ ] Right to deletion implemented
- [ ] Data export feature (GDPR)
- [ ] COPPA compliance (if applicable)
- [ ] HIPAA compliance (if applicable)
- [ ] Platform privacy labels configured

## Push Notifications

- [ ] Push notification setup (APNs/FCM)
- [ ] Notification permissions requested
- [ ] Deep linking from notifications
- [ ] Notification preferences screen
- [ ] Badge count management
- [ ] Quiet hours support
- [ ] Notification categories defined

## Analytics & Monitoring

- [ ] Analytics events defined
- [ ] Screen tracking implemented
- [ ] Error tracking configured
- [ ] Performance monitoring setup
- [ ] Crash reporting configured
- [ ] User properties tracked
- [ ] A/B testing framework (if needed)

## App Store Preparation

- [ ] App description written
- [ ] Keywords optimized
- [ ] Screenshots captured (all devices)
- [ ] App preview video (optional)
- [ ] Privacy nutrition labels (iOS)
- [ ] Data safety section (Android)
- [ ] Content rating configured
- [ ] Pricing configured
- [ ] Release notes written
- [ ] Support URL configured
- [ ] Marketing URL configured

## Post-Launch

- [ ] Crash rate monitoring
- [ ] User feedback collection
- [ ] Analytics review
- [ ] Performance monitoring
- [ ] A/B test results
- [ ] User support process
- [ ] Hotfix process defined
- [ ] Update cadence established

## Configuration

[CONFIGURE] Mark items as N/A or add project-specific items:
- Remove irrelevant checklist items
- Add project-specific requirements
- Set priority levels (P0, P1, P2)
- Assign owners
- Set deadlines
