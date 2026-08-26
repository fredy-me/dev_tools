# AI Testing Approach for Frontend

## Testing Generation Rules

### Unit Test Generation

```typescript
// Input: Component source code
// Output: Test file with comprehensive coverage

// Pattern: Test file structure
import { render, screen, fireEvent } from '@testing-library/react';
import { ComponentName } from './ComponentName';

describe('ComponentName', () => {
  // Arrange - set up test data
  const defaultProps = {
    // ... realistic default props
  };

  beforeEach(() => {
    vi.clearAllMocks();
  });

  it('renders correctly', () => {
    render(<ComponentName {...defaultProps} />);
    // Assert
  });

  it('handles user interactions', async () => {
    render(<ComponentName {...defaultProps} />);
    // Act
    // Assert
  });

  it('handles error states', () => {
    // Test error scenarios
  });

  it('meets accessibility requirements', () => {
    // Test accessibility
  });
});
```

### Test Prioritization

| Priority | Type | Examples |
|----------|------|----------|
| P0 | Critical Path | Auth flows, payments, data submission |
| P1 | Core Features | CRUD operations, navigation, forms |
| P2 | Edge Cases | Empty states, errors, loading |
| P3 | UI Polish | Animations, transitions, tooltips |

### Mock Patterns

```typescript
// Mock API responses
import { http, HttpResponse } from 'msw';
import { server } from '../test/server';

// Mock specific endpoint
server.use(
  http.get('/api/users', () => {
    return HttpResponse.json({ data: mockUsers });
  })
);

// Mock error response
server.use(
  http.get('/api/users', () => {
    return new HttpResponse(null, { status: 500 });
  })
);

// Mock delayed response
server.use(
  http.get('/api/users', async () => {
    await new Promise(resolve => setTimeout(resolve, 2000));
    return HttpResponse.json({ data: mockUsers });
  })
);
```

### E2E Test Generation

```typescript
// Pattern: User journey test
import { test, expect } from '@playwright/test';

test.describe('Feature Name', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/feature');
  });

  test('completes user journey', async ({ page }) => {
    // Step 1: Initial state
    await expect(page.getByRole('heading')).toBeVisible();

    // Step 2: User action
    await page.getByRole('button', { name: 'Action' }).click();

    // Step 3: Verify result
    await expect(page.getByText('Success')).toBeVisible();
  });

  test('handles error scenario', async ({ page }) => {
    // Test error handling
  });

  test('maintains accessibility', async ({ page }) => {
    // Test keyboard navigation
    await page.keyboard.press('Tab');
    await expect(page.getByRole('button')).toBeFocused();
  });
});
```

## Coverage Requirements

```typescript
// vitest.config.ts
export default defineConfig({
  test: {
    coverage: {
      thresholds: {
        statements: 80,
        branches: 80,
        functions: 80,
        lines: 80,
      },
    },
  },
});
```

## Test Data Factories

```typescript
// src/test/factories.ts
interface User {
  id: string;
  name: string;
  email: string;
  role: 'admin' | 'user';
}

export function createMockUser(overrides: Partial<User> = {}): User {
  return {
    id: crypto.randomUUID(),
    name: 'Test User',
    email: 'test@example.com',
    role: 'user',
    ...overrides,
  };
}

export function createMockUsers(count: number): User[] {
  return Array.from({ length: count }, (_, i) =>
    createMockUser({
      id: `user-${i}`,
      name: `User ${i}`,
      email: `user${i}@example.com`,
    })
  );
}
```

## What AI Should Test

1. **Happy path**: Normal user flow
2. **Error states**: API failures, validation errors
3. **Loading states**: Skeletons, spinners
4. **Empty states**: No data, no results
5. **Edge cases**: Long text, special characters
6. **Accessibility**: Keyboard navigation, screen readers
7. **Responsiveness**: Mobile, tablet, desktop
8. **Authentication**: Login, logout, token refresh
9. **Permissions**: Role-based access
10. **Concurrent actions**: Rapid clicks, multiple requests

## Test Review Checklist

- [ ] Tests are independent (no shared state)
- [ ] Tests use realistic data
- [ ] Tests cover happy path and error cases
- [ ] Tests are readable (clear arrange/act/assert)
- [ ] Mocks are minimal (only external dependencies)
- [ ] Tests verify behavior, not implementation
- [ ] Tests include accessibility checks
- [ ] Tests have appropriate timeouts
