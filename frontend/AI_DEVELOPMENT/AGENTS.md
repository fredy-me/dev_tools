# AI Agent Guidelines for Frontend Development

## Overview

This document provides guidelines for AI agents working on frontend codebases. Follow these rules to maintain consistency and quality.

## Code Generation Rules

### Framework-Specific Rules

#### React
- Use functional components with hooks
- Use TypeScript for all components
- Prefer named exports over default exports
- Use React.memo for performance-critical components
- Avoid inline styles; use CSS modules or Tailwind

```tsx
// GOOD
interface UserCardProps {
  user: User;
  onSelect: (id: string) => void;
}

export function UserCard({ user, onSelect }: UserCardProps) {
  return (
    <article onClick={() => onSelect(user.id)}>
      <h3>{user.name}</h3>
    </article>
  );
}
```

#### Vue
- Use Composition API with `<script setup>`
- Use TypeScript with defineProps/defineEmits
- Prefer composables for reusable logic
- Use Pinia for state management

```vue
<script setup lang="ts">
interface Props {
  user: User;
}

const props = defineProps<Props>();
const emit = defineEmits<{
  select: [id: string];
}>();
</script>
```

#### Angular
- Use standalone components
- Use signals for reactive state
- Follow Angular style guide conventions
- Use OnPush change detection

### State Management

```typescript
// GOOD: Local state for component-specific data
const [isOpen, setIsOpen] = useState(false);

// GOOD: Global state for shared data
const user = useSelector((state: RootState) => state.auth.user);

// BAD: Prop drilling through many levels
// Use context or state management instead
```

### API Integration

```typescript
// GOOD: Custom hook for data fetching
export function useUsers() {
  return useQuery({
    queryKey: ['users'],
    queryFn: () => apiClient.get('/users'),
  });
}

// GOOD: Error handling
const { data, error, isLoading } = useUsers();
if (error) return <ErrorMessage error={error} />;
```

### Component Structure

```
ComponentName/
├── ComponentName.tsx       # Main component
├── ComponentName.test.tsx  # Tests
├── ComponentName.stories.tsx # Storybook stories (optional)
├── index.ts               # Public exports
└── types.ts               # Component-specific types
```

## File Naming Conventions

| Type | Convention | Example |
|------|-----------|---------|
| Components | PascalCase | `UserProfile.tsx` |
| Hooks | camelCase with `use` | `useAuth.ts` |
| Utils | camelCase | `formatDate.ts` |
| Types | PascalCase | `User.ts` |
| Constants | UPPER_SNAKE | `API_ENDPOINTS.ts` |

## What NOT to Do

- Never use `any` type
- Never use `console.log` in production code
- Never hardcode API URLs
- Never store secrets in client code
- Never skip error handling
- Never use index as key for dynamic lists
- Never mutate state directly

## Response Format

When generating code:

1. Include TypeScript types
2. Add JSDoc comments for complex functions
3. Include error handling
4. Add accessibility attributes
5. Follow existing code style in the project

## Testing Requirements

- Write tests for new components
- Test user interactions, not implementation
- Use data-testid sparingly; prefer role/label queries
- Mock API calls with MSW
- Test error states and loading states
