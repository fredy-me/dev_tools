# Frontend Development Checklist

## Project Setup

- [ ] Initialize project with framework CLI
- [ ] Configure TypeScript with strict mode
- [ ] Set up ESLint and Prettier
- [ ] Configure Tailwind CSS
- [ ] Set up Git hooks (Husky + lint-staged)
- [ ] Create environment variable files
- [ ] Configure path aliases (@/)
- [ ] Set up testing framework (Vitest)
- [ ] Configure E2E testing (Playwright)
- [ ] Add README with setup instructions

## Development Workflow

### Before Starting
- [ ] Review requirements and acceptance criteria
- [ ] Check for existing similar components
- [ ] Review design system documentation
- [ ] Identify API endpoints needed

### During Development
- [ ] Follow naming conventions
- [ ] Write TypeScript types first
- [ ] Build reusable components
- [ ] Handle loading states
- [ ] Handle error states
- [ ] Handle empty states
- [ ] Add proper accessibility attributes
- [ ] Write unit tests
- [ ] Test on mobile viewport

### Before Committing
- [ ] Run typecheck (`npm run typecheck`)
- [ ] Run linter (`npm run lint`)
- [ ] Run formatter (`npm run format`)
- [ ] Run tests (`npm test`)
- [ ] Check bundle size impact
- [ ] Review code changes

## Component Checklist

- [ ] TypeScript interface defined
- [ ] Props have meaningful names
- [ ] Default props specified
- [ ] Component is properly exported
- [ ] Accessible (ARIA, keyboard nav)
- [ ] Responsive design
- [ ] Loading state handled
- [ ] Error state handled
- [ ] Empty state handled
- [ ] Tests written

## Performance

- [ ] Images optimized (WebP, lazy loading)
- [ ] Code splitting configured
- [ ] Bundle size within budget
- [ ] No unnecessary re-renders
- [ ] Memoization where needed
- [ ] Lazy loading for routes
- [ ] Service worker configured (if PWA)
- [ ] Core Web Vitals pass

## Security

- [ ] No secrets in client code
- [ ] Input validation implemented
- [ ] XSS prevention (sanitization)
- [ ] CSRF protection enabled
- [ ] Authentication flow secure
- [ ] Authorization checks in place
- [ ] CSP headers configured
- [ ] Dependencies audited

## Accessibility

- [ ] Semantic HTML used
- [ ] Color contrast >= 4.5:1
- [ ] Keyboard navigation works
- [ ] Screen reader tested
- [ ] Focus management correct
- [ ] Form labels associated
- [ ] Error messages accessible
- [ ] Skip navigation link

## Testing

- [ ] Unit tests for utilities
- [ ] Component tests written
- [ ] Integration tests for features
- [ ] E2E tests for critical paths
- [ ] Test coverage meets threshold
- [ ] Tests are independent
- [ ] Mocks are minimal
- [ ] Edge cases covered

## Deployment

- [ ] Environment variables set
- [ ] Build succeeds
- [ ] CI/CD pipeline configured
- [ ] Preview deploy tested
- [ ] Staging deploy tested
- [ ] Production deploy tested
- [ ] Monitoring configured
- [ ] Rollback plan ready

## Documentation

- [ ] README updated
- [ ] Component docs added
- [ ] API integration documented
- [ ] Environment variables documented
- [ ] Deployment steps documented
- [ ] Changelog updated

## Pre-Launch

- [ ] All tests passing
- [ ] No console errors
- [ ] Performance audit passed
- [ ] Security audit passed
- [ ] Accessibility audit passed
- [ ] Cross-browser tested
- [ ] Mobile tested
- [ ] Analytics configured
- [ ] Error tracking configured
- [ ] SEO meta tags set
