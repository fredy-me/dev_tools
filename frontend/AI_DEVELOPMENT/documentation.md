# AI Documentation Standards for Frontend

## Documentation Types

### Component Documentation

```tsx
/**
 * UserCard displays user information in a card format.
 *
 * @example
 * ```tsx
 * <UserCard
 *   user={{ id: '1', name: 'John Doe', email: 'john@example.com' }}
 *   onSelect={(id) => navigate(`/users/${id}`)}
 * />
 * ```
 *
 * @accessibility
 * - Uses semantic `<article>` element
 * - Click handler can be triggered via Enter/Space keys
 * - Includes aria-label for screen readers
 */
interface UserCardProps {
  /** User object containing display information */
  user: User;
  /** Callback when card is selected */
  onSelect: (id: string) => void;
  /** Optional additional CSS classes */
  className?: string;
}
```

### Hook Documentation

```tsx
/**
 * Custom hook for managing form state with validation.
 *
 * @param initialValues - Initial form values
 * @param validationSchema - Zod schema for validation
 * @returns Form state and handlers
 *
 * @example
 * ```tsx
 * const { values, errors, handleChange, handleSubmit } = useForm({
 *   initialValues: { email: '', password: '' },
 *   validationSchema: loginSchema,
 * });
 * ```
 */
function useForm<T>(options: UseFormOptions<T>): UseFormReturn<T>
```

### API Client Documentation

```typescript
/**
 * API client for user management endpoints.
 *
 * All methods include automatic token refresh and error handling.
 *
 * @example
 * ```typescript
 * // Fetch users with pagination
 * const users = await userApi.list({ page: 1, limit: 10 });
 *
 * // Create new user
 * const newUser = await userApi.create({ name: 'John', email: 'john@example.com' });
 * ```
 */
class UserApi {
  /**
   * Fetches a paginated list of users.
   *
   * @param params - Query parameters
   * @param params.page - Page number (1-indexed)
   * @param params.limit - Items per page (max 100)
   * @param params.search - Optional search term
   * @returns Paginated user list
   * @throws {ApiError} When request fails
   */
  async list(params: UserListParams): Promise<PaginatedResponse<User>> {
    return apiClient.get('/users', params);
  }
}
```

## README Template

```markdown
# Component Name

Brief description of what this component does.

## Usage

### Basic

​```tsx
import { ComponentName } from './components';

<ComponentName prop="value" />
​```

### With Variants

​```tsx
<ComponentName variant="primary" size="large">
  Content
</ComponentName>
​```

## Props

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| variant | 'primary' \| 'secondary' | 'primary' | Visual style |
| size | 'sm' \| 'md' \| 'lg' | 'md' | Component size |
| disabled | boolean | false | Disabled state |

## Accessibility

- Uses semantic HTML
- Keyboard navigable
- Screen reader friendly

## Examples

See [Storybook stories](./ComponentName.stories.tsx) for live examples.
```

## Generated Documentation Rules

1. **JSDoc**: All exported functions/components
2. **README**: For complex modules/features
3. **Types**: Self-documenting TypeScript interfaces
4. **Examples**: Usage examples in JSDoc
5. **Accessibility**: ARIA patterns and keyboard support

## What to Document

| Item | Required | Location |
|------|----------|----------|
| Component props | Yes | JSDoc/interface |
| Custom hooks | Yes | JSDoc |
| Utility functions | Yes | JSDoc |
| API endpoints | Yes | Code comments |
| Complex logic | Yes | Code comments |
| Type definitions | Yes | Type name + docs |
| Configuration | Yes | README |

## Documentation Anti-patterns

```typescript
// ❌ BAD: Redundant comment
// This function formats the date
function formatDate(date: Date) {
  return date.toLocaleDateString();
}

// ❌ BAD: Outdated comment
// Returns user by ID (created: 2022)
function getUser(id: string) { /* ... */ }

// ✅ GOOD: Explains why, not what
// Uses toLocaleDateString for locale-aware formatting
// Falls back to ISO format for unsupported locales
function formatDate(date: Date): string {
  return date.toLocaleDateString() ?? date.toISOString().split('T')[0];
}
```

## Auto-Generated Content

Let AI generate:
- Component prop tables
- Usage examples
- Accessibility notes
- Test descriptions
- CHANGELOG entries

Let humans write:
- Architecture decisions
- Business logic explanations
- Migration guides
- Deprecation notices
