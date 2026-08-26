# Frontend Coding Standards

## Project Structure

```
src/
├── api/                  # API client and endpoints
│   ├── client.ts
│   └── endpoints/
├── assets/               # Static assets (images, fonts)
├── components/           # Reusable UI components
│   ├── ui/               # Primitive components (Button, Input, etc.)
│   ├── layout/           # Layout components (Header, Sidebar, etc.)
│   └── features/         # Feature-specific components
├── constants/            # App constants
├── hooks/                # Custom React hooks
├── lib/                  # Utility functions
├── pages/                # Route-level components
├── services/             # Business logic services
├── store/                # State management
├── styles/               # Global styles
├── types/                # TypeScript type definitions
├── utils/                # Utility functions
└── App.tsx               # Root component
```

## Naming Conventions

### Files

| Type | Convention | Example |
|------|-----------|---------|
| Components | PascalCase | `UserProfile.tsx` |
| Hooks | camelCase, `use` prefix | `useAuth.ts` |
| Utils | camelCase | `formatDate.ts` |
| Types | PascalCase | `types.ts` |
| Constants | UPPER_SNAKE_CASE | `API_ENDPOINTS.ts` |

### Variables & Functions

```typescript
// Components: PascalCase
function UserCard() {}
const UserProfile = () => {};

// Hooks: camelCase with `use` prefix
function useUserData() {}
const useAuth = () => {};

// Utils: camelCase
function formatDate() {}
const calculateTotal = () => {};

// Constants: UPPER_SNAKE_CASE
const API_BASE_URL = 'https://api.example.com';
const MAX_RETRY_COUNT = 3;

// Types/Interfaces: PascalCase
interface UserProps {}
type UserRole = 'admin' | 'user';

// Boolean variables: `is`, `has`, `should`, `can` prefix
const isLoading = true;
const hasError = false;
const shouldRedirect = true;
const canEdit = false;
```

## TypeScript Standards

```typescript
// Use explicit types for function parameters and returns
function processUser(user: User): ProcessedUser {
  return {
    id: user.id,
    displayName: `${user.firstName} ${user.lastName}`,
  };
}

// Use interfaces for object shapes
interface User {
  id: string;
  firstName: string;
  lastName: string;
  email: string;
  role: UserRole;
  createdAt: Date;
}

// Use type for unions and intersections
type UserRole = 'admin' | 'user' | 'viewer';
type UserWithPosts = User & { posts: Post[] };

// Use enums for constant sets
enum HttpMethod {
  GET = 'GET',
  POST = 'POST',
  PUT = 'PUT',
  DELETE = 'DELETE',
}

// Avoid `any` - use `unknown` and narrow
function processData(data: unknown) {
  if (typeof data === 'string') {
    return data.toUpperCase();
  }
  if (Array.isArray(data)) {
    return data.map(processData);
  }
  throw new Error('Invalid data');
}

// Use discriminated unions for state
type AppState =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: User[] }
  | { status: 'error'; error: string };
```

## Component Standards

```tsx
// Component structure
import { memo } from 'react';
import { cn } from '@/lib/utils';

interface UserCardProps {
  user: User;
  onSelect: (id: string) => void;
  className?: string;
}

// Memoize components that receive stable props
export const UserCard = memo(function UserCard({
  user,
  onSelect,
  className,
}: UserCardProps) {
  const fullName = `${user.firstName} ${user.lastName}`;

  return (
    <article
      className={cn('rounded-lg border p-4', className)}
      onClick={() => onSelect(user.id)}
    >
      <h3>{fullName}</h3>
      <p className="text-sm text-neutral-500">{user.email}</p>
    </article>
  );
});

UserCard.displayName = 'UserCard';
```

## CSS Standards

```css
/* Use CSS custom properties for theming */
:root {
  --color-primary: #3b82f6;
}

/* BEM naming for custom classes */
.block {}
.block__element {}
.block--modifier {}

/* Mobile-first responsive design */
.component {
  /* Base styles (mobile) */
  padding: 1rem;

  /* Tablet */
  @media (min-width: 768px) {
    padding: 1.5rem;
  }

  /* Desktop */
  @media (min-width: 1024px) {
    padding: 2rem;
  }
}

/* Use logical properties for RTL support */
.card {
  margin-inline-start: 1rem;
  padding-inline: 1.5rem;
  border-inline-start: 4px solid var(--color-primary);
}
```

## Import Order

```typescript
// 1. External libraries
import React from 'react';
import { useNavigate } from 'react-router-dom';
import { useQuery } from '@tanstack/react-query';

// 2. Internal modules (absolute paths)
import { apiClient } from '@/api/client';
import { Button } from '@/components/ui/Button';
import { useAuth } from '@/hooks/useAuth';

// 3. Relative imports
import { formatDate } from './utils';
import type { UserProps } from './types';

// 4. Types (always last)
import type { User } from '@/types';
```

## Error Handling

```typescript
// Use typed errors
class AppError extends Error {
  constructor(
    message: string,
    public code: string,
    public statusCode?: number
  ) {
    super(message);
    this.name = 'AppError';
  }
}

// Async error handling
async function fetchData<T>(url: string): Promise<T> {
  try {
    const response = await fetch(url);
    if (!response.ok) {
      throw new AppError(
        `HTTP error ${response.status}`,
        'HTTP_ERROR',
        response.status
      );
    }
    return response.json();
  } catch (error) {
    if (error instanceof AppError) throw error;
    throw new AppError('Network error', 'NETWORK_ERROR');
  }
}

// React error boundaries
interface ErrorBoundaryProps {
  children: ReactNode;
  fallback?: ReactNode;
}

interface ErrorBoundaryState {
  hasError: boolean;
  error?: Error;
}

class ErrorBoundary extends Component<ErrorBoundaryProps, ErrorBoundaryState> {
  constructor(props: ErrorBoundaryProps) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(error: Error) {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    console.error('ErrorBoundary caught:', error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback || <div>Something went wrong.</div>;
    }
    return this.props.children;
  }
}
```

## Comments & Documentation

```typescript
// ✅ GOOD: JSDoc for public APIs
/**
 * Formats a date string into a localized format.
 * @param date - ISO date string or Date object
 * @param options - Intl.DateTimeFormat options
 * @returns Formatted date string
 */
export function formatDate(
  date: string | Date,
  options?: Intl.DateTimeFormatOptions
): string {
  return new Intl.DateTimeFormat('en-US', options).format(new Date(date));
}

// ❌ BAD: Redundant comments
// This function formats the date
function formatDate(date: string) {
  return new Date(date).toLocaleDateString(); // Format as date string
}

// ✅ GOOD: Explain WHY, not WHAT
// Debounce to prevent excessive API calls during rapid typing
const debouncedSearch = debounce(search, 300);
```

## Testing Standards

```typescript
// Test file naming: ComponentName.test.tsx
import { render, screen, fireEvent } from '@testing-library/react';
import { UserCard } from './UserCard';

describe('UserCard', () => {
  const mockUser = {
    id: '1',
    firstName: 'John',
    lastName: 'Doe',
    email: 'john@example.com',
  };

  it('renders user name and email', () => {
    render(<UserCard user={mockUser} onSelect={vi.fn()} />);

    expect(screen.getByText('John Doe')).toBeInTheDocument();
    expect(screen.getByText('john@example.com')).toBeInTheDocument();
  });

  it('calls onSelect with user id when clicked', async () => {
    const onSelect = vi.fn();
    render(<UserCard user={mockUser} onSelect={onSelect} />);

    await fireEvent.click(screen.getByRole('article'));

    expect(onSelect).toHaveBeenCalledWith('1');
  });
});
```

## Code Review Checklist

- [ ] TypeScript strict mode enabled
- [ ] No `any` types
- [ ] Components have proper TypeScript interfaces
- [ ] Props have meaningful names
- [ ] Error handling implemented
- [ ] Tests cover main scenarios
- [ ] Accessibility attributes present
- [ ] No hardcoded values (use constants)
- [ ] Imports follow order convention
- [ ] No unused imports or variables
