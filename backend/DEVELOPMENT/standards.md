# Backend Coding Standards

## General Principles

1. **Clarity over cleverness** - Write code that is easy to read and understand
2. **Fail fast** - Validate inputs early, throw errors early
3. **Single Responsibility** - Each function/class does one thing well
4. **DRY but pragmatic** - Don't repeat yourself, but don't over-abstract
5. **Secure by default** - Always sanitize, validate, and authorize

## Project Structure

```
src/
├── modules/               # Feature modules
│   ├── users/
│   │   ├── users.controller.ts
│   │   ├── users.service.ts
│   │   ├── users.repository.ts
│   │   ├── users.types.ts
│   │   ├── users.validation.ts
│   │   ├── users.test.ts
│   │   └── users.integration.test.ts
│   ├── orders/
│   │   └── ...
│   └── auth/
│       └── ...
├── shared/                # Shared utilities
│   ├── middleware/
│   ├── errors/
│   ├── logging/
│   └── utils/
├── config/               # Configuration
│   ├── database.ts
│   ├── redis.ts
│   └── app.ts
├── migrations/           # Database migrations
└── seeds/               # Seed data
```

## Naming Conventions

```typescript
// Files: kebab-case
users.controller.ts
users.service.ts
user.model.ts
auth.middleware.ts

// Classes: PascalCase
class UserService {}
class UsersController {}
class DatabaseConnection {}

// Functions/Methods: camelCase
async function findUserById(id: string): Promise<User> {}
async function createUser(data: CreateUserDTO): Promise<User> {}

// Variables: camelCase
const userName = 'John';
const MAX_RETRY_COUNT = 3;

// Constants: UPPER_SNAKE_CASE
const API_VERSION = 'v1';
const DEFAULT_PAGE_SIZE = 20;

// Types/Interfaces: PascalCase with descriptive names
interface CreateUserRequest {}
interface UserResponse {}
type UserRole = 'admin' | 'user' | 'viewer';

// Enums: PascalCase
enum OrderStatus {
  PENDING = 'pending',
  CONFIRMED = 'confirmed',
}

// Database columns: snake_case
// Map in ORM: userId -> user_id

// API responses: snake_case
{
  "user_id": "abc-123",
  "created_at": "2024-01-15T10:00:00Z"
}
```

## Code Style

### Function Organization

```typescript
// Order functions logically:
// 1. Exported functions (public API)
// 2. Helper functions
// 3. Types and interfaces

// Bad: Mixed order
export async function createUser() { ... }
type CreateUserDTO = { ... }
function validateEmail() { ... }
export async function deleteUser() { ... }

// Good: Organized
// --- Exported Functions ---
export async function createUser(data: CreateUserDTO): Promise<User> {
  validateEmail(data.email);
  const hash = await hashPassword(data.password);
  return db.users.create({ ...data, passwordHash: hash });
}

export async function deleteUser(id: string): Promise<void> {
  await db.users.softDelete(id);
}

// --- Helper Functions ---
function validateEmail(email: string): void {
  if (!emailRegex.test(email)) {
    throw new ValidationError('Invalid email format', [
      { field: 'email', message: 'Must be a valid email', code: 'INVALID_FORMAT' },
    ]);
  }
}

// --- Types ---
interface CreateUserDTO {
  email: string;
  name: string;
  password: string;
}
```

### Error Handling

```typescript
// Bad: Swallowing errors
try {
  await createUser(data);
} catch (e) {
  console.log(e);
}

// Bad: Generic catch
try {
  await createUser(data);
} catch (e) {
  throw new Error('Something went wrong');
}

// Good: Specific handling with context
try {
  await createUser(data);
} catch (error) {
  if (error instanceof AppError) {
    throw error; // Re-throw known errors
  }
  logger.error({ error, userId: data.email }, 'Failed to create user');
  throw new AppError('Failed to create user', 'USER_CREATE_FAILED', 500);
}

// Good: Using typed errors
async function findUser(id: string): Promise<User> {
  const user = await db.users.findById(id);
  if (!user) {
    throw new NotFoundError('User', id);
  }
  return user;
}
```

### Async/Await

```typescript
// Bad: Callback hell
getUser(id, (err, user) => {
  if (err) throw err;
  getOrders(user.id, (err, orders) => {
    if (err) throw err;
    processOrders(orders);
  });
});

// Bad: Promise chain
getUser(id)
  .then(user => getOrders(user.id))
  .then(orders => processOrders(orders))
  .catch(err => handleError(err));

// Good: Async/await
async function processUserOrders(id: string): Promise<void> {
  const user = await getUser(id);
  const orders = await getOrders(user.id);
  await processOrders(orders);
}

// Good: Parallel operations
async function getUserDashboard(id: string): Promise<Dashboard> {
  const [user, orders, analytics] = await Promise.all([
    getUser(id),
    getOrders(id),
    getAnalytics(id),
  ]);

  return { user, orders, analytics };
}
```

## TypeScript Specific

### Strict Type Checking

```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "exactOptionalPropertyTypes": true,
    "noPropertyAccessFromIndexSignature": true
  }
}
```

### Type Definitions

```typescript
// Use branded types for IDs
type UserId = string & { readonly __brand: 'UserId' };
type OrderId = string & { readonly __brand: 'OrderId' };

function createUserId(id: string): UserId {
  return id as UserId;
}

// Use discriminated unions for state
type OrderState =
  | { status: 'pending'; paymentPending: boolean }
  | { status: 'confirmed'; confirmedAt: Date }
  | { status: 'shipped'; trackingNumber: string }
  | { status: 'delivered'; deliveredAt: Date };

// Use utility types
type CreateUserInput = Omit<User, 'id' | 'createdAt' | 'updatedAt'>;
type UserResponse = Pick<User, 'id' | 'email' | 'name'>;
type PartialUser = Partial<User>;
```

## API Design Standards

### Request Validation

```typescript
// Zod schemas for validation
const CreateUserSchema = z.object({
  body: z.object({
    email: z.string().email(),
    name: z.string().min(1).max(255),
    password: z.string().min(12).max(128),
    role: z.enum(['user', 'viewer']).default('user'),
  }),
  query: z.object({}).optional(),
  params: z.object({}).optional(),
});

// Type inference
type CreateUserInput = z.infer<typeof CreateUserSchema>['body'];

// Middleware usage
router.post('/users', validate(CreateUserSchema), controller.create);
```

### Response Format

```typescript
// Standard response types
interface ApiResponse<T> {
  data: T;
  meta?: {
    pagination?: PaginationMeta;
    request_id: string;
  };
}

interface PaginationMeta {
  page: number;
  limit: number;
  total: number;
  total_pages: number;
}

// Controller response helpers
class ResponseHelper {
  static success<T>(res: Response, data: T, statusCode = 200): void {
    res.status(statusCode).json({
      data,
      meta: {
        request_id: res.req.requestId,
      },
    });
  }

  static created<T>(res: Response, data: T): void {
    this.success(res, data, 201);
  }

  static paginated<T>(
    res: Response,
    data: T[],
    pagination: PaginationMeta,
  ): void {
    res.status(200).json({
      data,
      meta: {
        pagination,
        request_id: res.req.requestId,
      },
    });
  }
}
```

## Database Standards

### Query Patterns

```typescript
// Bad: Raw SQL injection vulnerable
const query = `SELECT * FROM users WHERE id = '${userId}'`;

// Good: Parameterized query
const query = 'SELECT * FROM users WHERE id = $1';
const result = await db.query(query, [userId]);

// Good: Using ORM
const user = await prisma.user.findUnique({
  where: { id: userId },
  select: { id: true, email: true, name: true },
});

// Good: Transaction
async function transferFunds(fromId: string, toId: string, amount: number) {
  return db.transaction(async (tx) => {
    const from = await tx.account.findUnique({ where: { id: fromId } });
    const to = await tx.account.findUnique({ where: { id: toId } });

    if (from.balance < amount) {
      throw new InsufficientFundsError();
    }

    await tx.account.update({
      where: { id: fromId },
      data: { balance: { decrement: amount } },
    });

    await tx.account.update({
      where: { id: toId },
      data: { balance: { increment: amount } },
    });
  });
}
```

### Migration Standards

```typescript
// Migration template
export default {
  up: async (queryInterface: QueryInterface, DataTypes: typeof Sequelize) => {
    await queryInterface.createTable('users', {
      id: {
        type: DataTypes.UUID,
        defaultValue: DataTypes.UUIDV4,
        primaryKey: true,
      },
      email: {
        type: DataTypes.STRING(255),
        allowNull: false,
        unique: true,
      },
      created_at: {
        type: DataTypes.DATE,
        allowNull: false,
        defaultValue: DataTypes.NOW,
      },
    });

    // Add indexes
    await queryInterface.addIndex('users', ['email']);
    await queryInterface.addIndex('users', ['created_at']);
  },

  down: async (queryInterface: QueryInterface) => {
    await queryInterface.dropTable('users');
  },
};
```

## Testing Standards

```typescript
// Test file naming: *.test.ts or *.spec.ts
// Test organization: describe > it/test

describe('UserService', () => {
  describe('createUser', () => {
    it('should create a user with valid data', async () => {
      const input = { email: 'test@example.com', name: 'Test' };
      const user = await userService.createUser(input);

      expect(user).toHaveProperty('id');
      expect(user.email).toBe(input.email);
    });

    it('should throw ValidationError for invalid email', async () => {
      const input = { email: 'invalid', name: 'Test' };

      await expect(userService.createUser(input))
        .rejects.toThrow(ValidationError);
    });

    it('should throw ConflictError for duplicate email', async () => {
      const input = { email: 'existing@example.com', name: 'Test' };
      await userService.createUser(input);

      await expect(userService.createUser(input))
        .rejects.toThrow(ConflictError);
    });
  });
});
```

## Security Standards

```typescript
// Always validate and sanitize input
const sanitized = DOMPurify.sanitize(userInput);

// Always use parameterized queries
db.query('SELECT * FROM users WHERE id = $1', [userId]);

// Always hash passwords with Argon2id
const hash = await argon2.hash(password, {
  type: argon2.argon2id,
  memoryCost: 65536,
  timeCost: 3,
  parallelism: 4,
});

// Always set security headers
app.use(helmet());

// Always rate limit sensitive endpoints
app.use('/api/auth/login', rateLimit({ max: 5, windowMs: 900000 }));

// Never log sensitive data
logger.info({ userId: user.id }, 'User logged in');
// NOT: logger.info({ user }, 'User logged in') // leaks password hash
```

## Documentation Standards

```typescript
/**
 * Creates a new user in the system.
 *
 * @param data - User creation data
 * @returns Created user object
 * @throws ValidationError - If input data is invalid
 * @throws ConflictError - If email already exists
 *
 * @example
 * ```typescript
 * const user = await createUser({
 *   email: 'john@example.com',
 *   name: 'John Doe',
 *   password: 'securePassword123!',
 * });
 * ```
 */
async function createUser(data: CreateUserDTO): Promise<User> {
  // Implementation
}
```
