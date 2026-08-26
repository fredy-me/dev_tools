# Sample Frontend Project Example

## Project: Task Management App

### Technology Stack

- **Framework**: React 18 + TypeScript
- **Styling**: Tailwind CSS
- **State**: Zustand + React Query
- **Routing**: React Router v6
- **Testing**: Vitest + Playwright
- **Build**: Vite

### Project Structure

```
task-manager/
├── src/
│   ├── api/
│   │   ├── client.ts
│   │   └── tasks.ts
│   ├── components/
│   │   ├── ui/
│   │   │   ├── Button.tsx
│   │   │   ├── Input.tsx
│   │   │   └── Card.tsx
│   │   └── features/
│   │       ├── TaskList/
│   │       │   ├── TaskList.tsx
│   │       │   ├── TaskItem.tsx
│   │       │   └── TaskList.test.tsx
│   │       └── TaskForm/
│   │           ├── TaskForm.tsx
│   │           └── TaskForm.test.tsx
│   ├── hooks/
│   │   └── useTasks.ts
│   ├── store/
│   │   └── useTaskStore.ts
│   ├── types/
│   │   └── task.ts
│   ├── pages/
│   │   ├── Dashboard.tsx
│   │   └── Login.tsx
│   ├── App.tsx
│   └── main.tsx
├── package.json
└── README.md
```

### Type Definitions

```typescript
// src/types/task.ts
export interface Task {
  id: string;
  title: string;
  description: string;
  status: 'todo' | 'in-progress' | 'done';
  priority: 'low' | 'medium' | 'high';
  assigneeId: string | null;
  dueDate: string | null;
  createdAt: string;
  updatedAt: string;
}

export interface CreateTaskInput {
  title: string;
  description?: string;
  priority?: Task['priority'];
  assigneeId?: string;
  dueDate?: string;
}

export interface UpdateTaskInput extends Partial<CreateTaskInput> {
  status?: Task['status'];
}
```

### API Client

```typescript
// src/api/tasks.ts
import { apiClient } from './client';
import type { Task, CreateTaskInput, UpdateTaskInput } from '../types/task';

export const tasksApi = {
  list: async (params?: { status?: string; assigneeId?: string }) => {
    const { data } = await apiClient.get<{ data: Task[] }>('/tasks', { params });
    return data.data;
  },

  get: async (id: string) => {
    const { data } = await apiClient.get<{ data: Task }>(`/tasks/${id}`);
    return data.data;
  },

  create: async (input: CreateTaskInput) => {
    const { data } = await apiClient.post<{ data: Task }>('/tasks', input);
    return data.data;
  },

  update: async (id: string, input: UpdateTaskInput) => {
    const { data } = await apiClient.patch<{ data: Task }>(`/tasks/${id}`, input);
    return data.data;
  },

  delete: async (id: string) => {
    await apiClient.delete(`/tasks/${id}`);
  },
};
```

### Custom Hook

```typescript
// src/hooks/useTasks.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { tasksApi } from '../api/tasks';
import type { CreateTaskInput, UpdateTaskInput } from '../types/task';

export function useTasks(filters?: { status?: string }) {
  return useQuery({
    queryKey: ['tasks', filters],
    queryFn: () => tasksApi.list(filters),
  });
}

export function useTask(id: string) {
  return useQuery({
    queryKey: ['tasks', id],
    queryFn: () => tasksApi.get(id),
    enabled: !!id,
  });
}

export function useCreateTask() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: tasksApi.create,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['tasks'] });
    },
  });
}

export function useUpdateTask() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ id, input }: { id: string; input: UpdateTaskInput }) =>
      tasksApi.update(id, input),
    onSuccess: (_, variables) => {
      queryClient.invalidateQueries({ queryKey: ['tasks'] });
      queryClient.invalidateQueries({ queryKey: ['tasks', variables.id] });
    },
  });
}

export function useDeleteTask() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: tasksApi.delete,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['tasks'] });
    },
  });
}
```

### Component Example

```tsx
// src/components/features/TaskList/TaskItem.tsx
import type { Task } from '../../../types/task';
import { cn } from '../../../lib/utils';
import { useUpdateTask } from '../../../hooks/useTasks';

interface TaskItemProps {
  task: Task;
  onSelect: (id: string) => void;
}

const priorityStyles = {
  low: 'border-l-neutral-400',
  medium: 'border-l-warning-500',
  high: 'border-l-error-500',
};

const statusStyles = {
  todo: 'bg-neutral-100 text-neutral-600',
  'in-progress': 'bg-primary-100 text-primary-700',
  done: 'bg-success-100 text-success-700',
};

export function TaskItem({ task, onSelect }: TaskItemProps) {
  const updateTask = useUpdateTask();

  const handleStatusChange = (newStatus: Task['status']) => {
    updateTask.mutate({ id: task.id, input: { status: newStatus } });
  };

  return (
    <article
      className={cn(
        'rounded-lg border-l-4 bg-surface p-4 shadow-sm cursor-pointer',
        'hover:shadow-md transition-shadow',
        priorityStyles[task.priority]
      )}
      onClick={() => onSelect(task.id)}
      role="button"
      tabIndex={0}
      onKeyDown={(e) => e.key === 'Enter' && onSelect(task.id)}
    >
      <div className="flex items-start justify-between">
        <div>
          <h3 className="font-medium">{task.title}</h3>
          {task.description && (
            <p className="mt-1 text-sm text-neutral-500 line-clamp-2">
              {task.description}
            </p>
          )}
        </div>

        <span
          className={cn(
            'rounded-full px-2 py-1 text-xs font-medium',
            statusStyles[task.status]
          )}
        >
          {task.status}
        </span>
      </div>

      {task.dueDate && (
        <p className="mt-2 text-xs text-neutral-400">
          Due: {new Date(task.dueDate).toLocaleDateString()}
        </p>
      )}

      <div className="mt-3 flex gap-2">
        {(['todo', 'in-progress', 'done'] as const).map((status) => (
          <button
            key={status}
            onClick={(e) => {
              e.stopPropagation();
              handleStatusChange(status);
            }}
            className={cn(
              'text-xs px-2 py-1 rounded',
              task.status === status
                ? 'bg-primary-600 text-white'
                : 'bg-neutral-100 hover:bg-neutral-200'
            )}
          >
            {status}
          </button>
        ))}
      </div>
    </article>
  );
}
```

### Test Example

```typescript
// src/components/features/TaskList/TaskItem.test.tsx
import { render, screen, fireEvent } from '@testing-library/react';
import { TaskItem } from './TaskItem';
import type { Task } from '../../../types/task';

const mockTask: Task = {
  id: '1',
  title: 'Test Task',
  description: 'Test Description',
  status: 'todo',
  priority: 'high',
  assigneeId: null,
  dueDate: '2024-12-31',
  createdAt: '2024-01-01',
  updatedAt: '2024-01-01',
};

describe('TaskItem', () => {
  it('renders task title and description', () => {
    render(<TaskItem task={mockTask} onSelect={vi.fn()} />);

    expect(screen.getByText('Test Task')).toBeInTheDocument();
    expect(screen.getByText('Test Description')).toBeInTheDocument();
  });

  it('displays priority indicator', () => {
    render(<TaskItem task={mockTask} onSelect={vi.fn()} />);
    const article = screen.getByRole('article');
    expect(article.className).toContain('border-l-error-500');
  });

  it('calls onSelect when clicked', async () => {
    const onSelect = vi.fn();
    render(<TaskItem task={mockTask} onSelect={onSelect} />);

    await fireEvent.click(screen.getByRole('article'));
    expect(onSelect).toHaveBeenCalledWith('1');
  });

  it('allows status change', async () => {
    render(<TaskItem task={mockTask} onSelect={vi.fn()} />);

    await fireEvent.click(screen.getByText('in-progress'));
    expect(screen.getByText('in-progress')).toHaveClass('bg-primary-600');
  });
});
```

### Dashboard Page

```tsx
// src/pages/Dashboard.tsx
import { useState } from 'react';
import { useTasks } from '../hooks/useTasks';
import { TaskItem } from '../components/features/TaskList/TaskItem';
import { TaskForm } from '../components/features/TaskForm/TaskForm';
import { Button } from '../components/ui/Button';

export function Dashboard() {
  const [showForm, setShowForm] = useState(false);
  const [filter, setFilter] = useState<string | undefined>();
  const { data: tasks, isLoading, error } = useTasks({ status: filter });

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error loading tasks</div>;

  return (
    <div className="container mx-auto px-4 py-8">
      <div className="flex items-center justify-between mb-6">
        <h1 className="text-2xl font-bold">Tasks</h1>
        <Button onClick={() => setShowForm(true)}>Add Task</Button>
      </div>

      <div className="flex gap-2 mb-4">
        {['all', 'todo', 'in-progress', 'done'].map((status) => (
          <button
            key={status}
            onClick={() => setFilter(status === 'all' ? undefined : status)}
            className={`px-3 py-1 rounded-full text-sm ${
              (filter || 'all') === status
                ? 'bg-primary-600 text-white'
                : 'bg-neutral-100'
            }`}
          >
            {status}
          </button>
        ))}
      </div>

      <div className="grid gap-4 md:grid-cols-2 lg:grid-cols-3">
        {tasks?.map((task) => (
          <TaskItem
            key={task.id}
            task={task}
            onSelect={(id) => console.log('Selected:', id)}
          />
        ))}
      </div>

      {showForm && (
        <TaskForm onClose={() => setShowForm(false)} />
      )}
    </div>
  );
}
```
