# AI Code Review Guidelines for Frontend

## Review Checklist

### TypeScript Quality

```typescript
// ❌ BAD: Using any
function processData(data: any) {
  return data.map((item: any) => item.value);
}

// ✅ GOOD: Proper types
interface DataItem {
  id: string;
  value: string;
}

function processData(data: DataItem[]): string[] {
  return data.map((item) => item.value);
}
```

### React Patterns

```tsx
// ❌ BAD: Missing key or using index
{items.map((item, index) => (
  <ListItem key={index} item={item} />
))}

// ✅ GOOD: Using stable unique ID
{items.map((item) => (
  <ListItem key={item.id} item={item} />
))}

// ❌ BAD: Inline function causing re-renders
<Button onClick={() => handleClick(id)} />

// ✅ GOOD: Memoized callback
const handleClick = useCallback((id: string) => {
  navigate(`/items/${id}`);
}, [navigate]);

<Button onClick={() => handleClick(id)} />
```

### Performance Issues

```tsx
// ❌ BAD: Unnecessary re-renders
function Parent() {
  const [count, setCount] = useState(0);
  return (
    <div>
      <ExpensiveComponent /> {/* Re-renders on count change */}
      <button onClick={() => setCount(c => c + 1)}>+</button>
    </div>
  );
}

// ✅ GOOD: Memoized expensive component
const MemoizedExpensive = memo(ExpensiveComponent);

function Parent() {
  const [count, setCount] = useState(0);
  return (
    <div>
      <MemoizedExpensive />
      <button onClick={() => setCount(c => c + 1)}>+</button>
    </div>
  );
}
```

### Security Issues

```tsx
// ❌ BAD: XSS vulnerability
<div dangerouslySetInnerHTML={{ __html: userInput }} />

// ❌ BAD: Exposed secrets
const API_KEY = 'sk_live_abc123';

// ✅ GOOD: Sanitized content
import DOMPurify from 'dompurify';
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(userInput) }} />

// ✅ GOOD: Environment variables
const API_KEY = import.meta.env.VITE_API_KEY;
```

### Accessibility Issues

```tsx
// ❌ BAD: No label association
<input type="email" placeholder="Email" />

// ✅ GOOD: Proper label
<label htmlFor="email">Email</label>
<input id="email" type="email" />

// ❌ BAD: Non-descriptive button
<button onClick={handleClick}>Click here</button>

// ✅ GOOD: Descriptive button
<button onClick={handleClick} aria-label="Delete user account">
  Delete Account
</button>
```

## Review Rules

### Error Handling

```typescript
// ❌ BAD: Silent failure
try {
  await fetchData();
} catch (e) {
  // silently fail
}

// ✅ GOOD: Proper error handling
try {
  await fetchData();
} catch (error) {
  console.error('Failed to fetch data:', error);
  toast.error('Failed to load data. Please try again.');
  throw error;
}
```

### API Calls

```typescript
// ❌ BAD: No error handling or loading state
useEffect(() => {
  fetch('/api/users').then(res => res.json()).then(setUsers);
}, []);

// ✅ GOOD: Complete async handling
const { data: users, isLoading, error } = useQuery({
  queryKey: ['users'],
  queryFn: () => apiClient.get('/users'),
});

if (isLoading) return <LoadingSpinner />;
if (error) return <ErrorMessage error={error} />;
```

### CSS/Styling

```css
/* ❌ BAD: !important usage */
.button {
  color: red !important;
}

/* ❌ BAD: Magic numbers */
.card {
  margin-top: 37px;
}

/* ✅ GOOD: Using design tokens */
.button {
  color: var(--color-primary);
}

.card {
  margin-top: var(--space-8);
}
```

## Common Issues to Flag

1. **Memory leaks**: Missing cleanup in useEffect
2. **Race conditions**: Unhandled concurrent requests
3. **Stale closures**: Variables captured in callbacks
4. **Bundle size**: Large imports that could be tree-shaken
5. **Hardcoded values**: Magic numbers/strings
6. **Missing tests**: New functions without tests
7. **Documentation**: Complex logic without comments
8. **Consistency**: Code style inconsistent with project

## Positive Feedback

- Well-structured components
- Good error handling patterns
- Clear naming conventions
- Proper TypeScript usage
- Accessibility considerations
- Performance optimizations
- Comprehensive tests
