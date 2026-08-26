# Detailed Component Architecture

## Component Hierarchy

```mermaid
graph TB
    subgraph "Root Level"
        App[App Component]
    end

    subgraph "Layout Level"
        Header[Header]
        Sidebar[Sidebar]
        Main[Main Content]
        Footer[Footer]
    end

    subgraph "Page Level"
        HomePage[Home Page]
        Dashboard[Dashboard]
        Settings[Settings]
        Profile[Profile]
    end

    subgraph "Feature Level"
        DataTable[Data Table]
        Charts[Charts]
        Forms[Forms]
        Modals[Modals]
    end

    subgraph "Primitive Level"
        Button[Button]
        Input[Input]
        Select[Select]
        Card[Card]
        Alert[Alert]
    end

    App --> Header
    App --> Sidebar
    App --> Main
    App --> Footer

    Main --> HomePage
    Main --> Dashboard
    Main --> Settings
    Main --> Profile

    Dashboard --> DataTable
    Dashboard --> Charts
    Settings --> Forms
    Profile --> Modals

    DataTable --> Button
    Forms --> Input
    Forms --> Select
    Modals --> Card
    Modals --> Alert
```

## Component Composition Pattern

```mermaid
graph TB
    subgraph "Smart Components (Containers)"
        SC1[UserListContainer]
        SC2[ProductDetailContainer]
        SC3[OrderFormContainer]
    end

    subgraph "State Management"
        Store[(Global Store)]
        LocalState[Local State]
    end

    subgraph "Presentational Components"
        PC1[UserCard]
        PC2[ProductGallery]
        PC3[OrderSummary]
        PC4[LoadingSpinner]
        PC5[ErrorMessage]
    end

    SC1 --> Store
    SC1 --> PC1
    SC1 --> PC4
    SC1 --> PC5

    SC2 --> Store
    SC2 --> PC2
    SC2 --> PC4

    SC3 --> Store
    SC3 --> LocalState
    SC3 --> PC3
    SC3 --> PC5
```

## React Component Example

```tsx
// Container Component
import { useSelector, useDispatch } from 'react-redux';
import { UserCard } from './UserCard';
import { LoadingSpinner } from './LoadingSpinner';
import { ErrorMessage } from './ErrorMessage';
import { fetchUsers } from '../store/userSlice';
import type { RootState, AppDispatch } from '../store';

interface UserListContainerProps {
  role?: 'admin' | 'user';
  limit?: number;
}

export function UserListContainer({ role, limit = 10 }: UserListContainerProps) {
  const dispatch = useDispatch<AppDispatch>();
  const { users, loading, error } = useSelector(
    (state: RootState) => state.users
  );

  useEffect(() => {
    dispatch(fetchUsers({ role, limit }));
  }, [dispatch, role, limit]);

  if (loading) return <LoadingSpinner />;
  if (error) return <ErrorMessage message={error} />;

  return (
    <div className="user-list">
      {users.map(user => (
        <UserCard
          key={user.id}
          user={user}
          onSelect={(id) => navigate(`/users/${id}`)}
        />
      ))}
    </div>
  );
}
```

## Vue Component Example

```vue
<template>
  <div class="user-list">
    <LoadingSpinner v-if="loading" />
    <ErrorMessage v-else-if="error" :message="error" />
    <UserCard
      v-for="user in users"
      :key="user.id"
      :user="user"
      @select="handleSelect"
    />
  </div>
</template>

<script setup lang="ts">
import { computed, onMounted } from 'vue';
import { useStore } from 'vuex';
import { useRouter } from 'vue-router';
import type { User } from '@/types';

interface Props {
  role?: 'admin' | 'user';
  limit?: number;
}

const props = withDefaults(defineProps<Props>(), {
  role: undefined,
  limit: 10
});

const store = useStore();
const router = useRouter();

const users = computed(() => store.state.users.list);
const loading = computed(() => store.state.users.loading);
const error = computed(() => store.state.users.error);

onMounted(() => {
  store.dispatch('users/fetchUsers', {
    role: props.role,
    limit: props.limit
  });
});

const handleSelect = (id: string) => {
  router.push(`/users/${id}`);
};
</script>
```

## Data Flow Diagram

```mermaid
sequenceDiagram
    participant User
    participant Component
    participant Action
    participant Reducer
    participant Store
    participant API
    participant Cache

    User->>Component: User Interaction
    Component->>Action: Dispatch Action
    Action->>API: API Request
    API-->>Cache: Cache Response
    API-->>Action: API Response
    Action->>Reducer: Update State
    Reducer->>Store: Store New State
    Store->>Component: State Changed
    Component->>User: Re-render UI
```

## Component Communication Patterns

### Parent to Child (Props)

```mermaid
graph LR
    Parent[Parent Component] -->|Props| Child[Child Component]
    Parent -->|Callback| Child
```

### Child to Parent (Events)

```mermaid
graph RL
    Child[Child Component] -->|Emit Event| Parent[Parent Component]
```

### Sibling Communication (Shared State)

```mermaid
graph TB
    Sibling1[Sibling A] -->|Dispatch| Store[Shared Store]
    Store -->|Select| Sibling2[Sibling B]
```

### Cross-Component Communication (Context/Provide-Inject)

```mermaid
graph TB
    Provider[Provider Component] -->|Provide| Context[Context]
    Context -->|Inject| Consumer1[Consumer A]
    Context -->|Inject| Consumer2[Consumer B]
```

## Atomic Design Pattern

```mermaid
graph TB
    subgraph "Atoms"
        A1[Button]
        A2[Input]
        A3[Label]
        A4[Icon]
    end

    subgraph "Molecules"
        M1[Search Field]
        M2[Form Group]
        M3[Card Header]
    end

    subgraph "Organisms"
        O1[Navigation Bar]
        O2[Data Table]
        O3[Form Section]
    end

    subgraph "Templates"
        T1[Dashboard Layout]
        T2[Form Layout]
        T3[Detail Layout]
    end

    subgraph "Pages"
        P1[Home Page]
        P2[Settings Page]
        P3[Profile Page]
    end

    A1 --> M1
    A2 --> M1
    A2 --> M2
    A3 --> M2
    A4 --> M3

    M1 --> O1
    M2 --> O3
    M3 --> O3

    O1 --> T1
    O2 --> T1
    O3 --> T2

    T1 --> P1
    T2 --> P2
    T3 --> P3
```

## Lazy Loading Implementation

```mermaid
graph TB
    subgraph "Bundle Splitting"
        Main[Main Bundle]
        FeatureA[Feature A Chunk]
        FeatureB[Feature B Chunk]
        FeatureC[Feature C Chunk]
    end

    subgraph "Loading States"
        Loading[Loading Fallback]
        Skeleton[Skeleton UI]
        Placeholder[Placeholder]
    end

    Main -->|Route Change| Loading
    Loading -->|Chunk Loaded| FeatureA
    Loading -->|Chunk Loaded| FeatureB
    Loading -->|Chunk Loaded| FeatureC

    FeatureA --> Skeleton
    FeatureB --> Placeholder
```

## Error Boundary Pattern

```mermaid
graph TB
    subgraph "Error Handling"
        EB[Error Boundary]
        Fallback[Fallback UI]
        Recovery[Recovery Action]
    end

    subgraph "Component Tree"
        Parent[Parent]
        Child1[Child 1]
        Child2[Child 2]
        Child3[Child 3 - Error]
    end

    EB --> Parent
    Child3 -.->|Error| EB
    EB --> Fallback
    Fallback --> Recovery
    Recovery -->|Retry| Parent
```

## Performance Optimization Map

```mermaid
graph LR
    subgraph "Rendering"
        Memo[Memoization]
        Pure[Pure Components]
        Virtual[List Virtualization]
    end

    subgraph "Bundle"
        CodeSplit[Code Splitting]
        TreeShake[Tree Shaking]
        Compression[Gzip/Brotli]
    end

    subgraph "Runtime"
        Debounce[Debounce]
        Throttle[Throttle]
        RAF[requestAnimationFrame]
    end

    subgraph "Network"
        Prefetch[Prefetch]
        Preload[Preload]
        HTTP2[HTTP/2 Push]
    end

    Memo --> Virtual
    CodeSplit --> TreeShake
    Debounce --> RAF
    Prefetch --> HTTP2
```

## Component Testing Strategy

```mermaid
graph TB
    subgraph "Unit Tests"
        UT1[Component Tests]
        UT2[Hook Tests]
        UT3[Utility Tests]
    end

    subgraph "Integration Tests"
        IT1[Feature Tests]
        IT2[Page Tests]
        IT3[Form Tests]
    end

    subgraph "E2E Tests"
        ET1[User Journeys]
        ET2[Critical Paths]
        ET3[Visual Regression]
    end

    UT1 --> IT1
    IT1 --> ET1
    UT2 --> IT2
    IT2 --> ET2
    UT3 --> IT3
```
