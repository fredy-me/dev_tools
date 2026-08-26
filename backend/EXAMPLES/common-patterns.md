# Common Backend Development Patterns

## 1. Repository Pattern

Separates data access logic from business logic.

```typescript
// Interface
interface UserRepository {
  findById(id: string): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  findMany(params: FindManyParams): Promise<PaginatedResult<User>>;
  create(data: CreateUserDTO): Promise<User>;
  update(id: string, data: UpdateUserDTO): Promise<User>;
  delete(id: string): Promise<void>;
}

// Implementation
class PostgresUserRepository implements UserRepository {
  constructor(private db: PrismaClient) {}

  async findById(id: string): Promise<User | null> {
    return this.db.user.findUnique({ where: { id } });
  }

  async create(data: CreateUserDTO): Promise<User> {
    return this.db.user.create({ data });
  }
}

// Usage in service
class UserService {
  constructor(private repo: UserRepository) {}

  async getUser(id: string): Promise<User> {
    const user = await this.repo.findById(id);
    if (!user) throw new NotFoundError('User', id);
    return user;
  }
}
```

## 2. Service Layer Pattern

Encapsulates business logic and orchestrates operations.

```typescript
class OrderService {
  constructor(
    private orderRepo: OrderRepository,
    private productRepo: ProductRepository,
    private paymentService: PaymentService,
    private emailService: EmailService,
    private eventBus: EventBus,
  ) {}

  async createOrder(userId: string, input: CreateOrderInput): Promise<Order> {
    // 1. Validate products
    const products = await this.productRepo.findByIds(input.productIds);
    this.validateProducts(products, input);

    // 2. Check inventory
    await this.checkInventory(products, input);

    // 3. Calculate total
    const total = this.calculateTotal(products, input);

    // 4. Create order
    const order = await this.orderRepo.create({
      userId,
      items: input.items,
      total,
      status: 'pending',
    });

    // 5. Process payment
    await this.paymentService.charge(order.id, total);

    // 6. Update inventory
    await this.updateInventory(products, input);

    // 7. Send confirmation
    await this.emailService.sendOrderConfirmation(order);

    // 8. Emit event
    await this.eventBus.emit('order.created', { order });

    return order;
  }
}
```

## 3. Middleware Pattern

Composable request/response interceptors.

```typescript
// Authentication middleware
const authenticate = async (
  req: Request,
  res: Response,
  next: NextFunction,
) => {
  const token = req.headers.authorization?.split(' ')[1];

  if (!token) {
    return res.status(401).json({
      error: { code: 'UNAUTHORIZED', message: 'Token required' },
    });
  }

  try {
    const decoded = jwt.verify(token, publicKey);
    req.user = decoded;
    next();
  } catch (error) {
    return res.status(401).json({
      error: { code: 'INVALID_TOKEN', message: 'Invalid token' },
    });
  }
};

// Authorization middleware factory
const authorize = (...roles: string[]) => {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({
        error: { code: 'FORBIDDEN', message: 'Insufficient permissions' },
      });
    }
    next();
  };
};

// Usage
router.delete('/users/:id', authenticate, authorize('admin'), controller.delete);
```

## 4. Circuit Breaker Pattern

Prevents cascading failures in distributed systems.

```typescript
class CircuitBreaker {
  private failures = 0;
  private lastFailureTime = 0;
  private state: 'closed' | 'open' | 'half-open' = 'closed';

  constructor(
    private options: {
      failureThreshold: number;
      resetTimeout: number;
    },
  ) {}

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === 'open') {
      if (Date.now() - this.lastFailureTime > this.options.resetTimeout) {
        this.state = 'half-open';
      } else {
        throw new Error('Circuit breaker is open');
      }
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  private onSuccess() {
    this.failures = 0;
    this.state = 'closed';
  }

  private onFailure() {
    this.failures++;
    this.lastFailureTime = Date.now();

    if (this.failures >= this.options.failureThreshold) {
      this.state = 'open';
    }
  }
}

// Usage
const paymentBreaker = new CircuitBreaker({
  failureThreshold: 5,
  resetTimeout: 30000,
});

async function processPayment(orderId: string, amount: number) {
  return paymentBreaker.execute(async () => {
    return stripe.charges.create({ amount, currency: 'usd' });
  });
}
```

## 5. Retry with Exponential Backoff

```typescript
async function retryWithBackoff<T>(
  fn: () => Promise<T>,
  options: {
    maxRetries: number;
    initialDelay: number;
    maxDelay: number;
    backoffMultiplier: number;
  } = {
    maxRetries: 3,
    initialDelay: 100,
    maxDelay: 5000,
    backoffMultiplier: 2,
  },
): Promise<T> {
  let lastError: Error;
  let delay = options.initialDelay;

  for (let attempt = 0; attempt <= options.maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error as Error;

      if (attempt < options.maxRetries) {
        // Add jitter
        const jitter = Math.random() * delay * 0.1;
        await sleep(delay + jitter);
        delay = Math.min(delay * options.backoffMultiplier, options.maxDelay);
      }
    }
  }

  throw lastError!;
}

// Usage
const data = await retryWithBackoff(
  () => fetchFromExternalAPI(url),
  { maxRetries: 3 },
);
```

## 6. Unit of Work Pattern

Groups multiple operations into a single transaction.

```typescript
class UnitOfWork {
  constructor(private prisma: PrismaClient) {}

  async execute<T>(fn: (tx: PrismaClient) => Promise<T>): Promise<T> {
    return this.prisma.$transaction(async (tx) => {
      try {
        const result = await fn(tx);
        return result;
      } catch (error) {
        // Transaction automatically rolls back
        throw error;
      }
    });
  }
}

// Usage
const order = await unitOfWork.execute(async (tx) => {
  // All these operations are in one transaction
  const order = await tx.order.create({ data: orderData });
  await tx.orderItem.createMany({ data: items });
  await tx.product.updateMany({
    where: { id: { in: productIds } },
    data: { stock: { decrement: 1 } },
  });
  return order;
});
```

## 7. Cache-Aside Pattern

Lazy loading with cache invalidation.

```typescript
class CachedUserService {
  private cacheTTL = 300; // 5 minutes

  constructor(
    private userRepo: UserRepository,
    private cache: RedisClient,
  ) {}

  async findById(id: string): Promise<User | null> {
    const cacheKey = `user:${id}`;

    // Try cache first
    const cached = await this.cache.get(cacheKey);
    if (cached) {
      return JSON.parse(cached);
    }

    // Fetch from database
    const user = await this.userRepo.findById(id);
    if (user) {
      await this.cache.setex(cacheKey, this.cacheTTL, JSON.stringify(user));
    }

    return user;
  }

  async update(id: string, data: UpdateUserDTO): Promise<User> {
    const user = await this.userRepo.update(id, data);

    // Invalidate cache
    await this.cache.del(`user:${id}`);

    return user;
  }
}
```

## 8. Event Sourcing Pattern

Store state changes as a sequence of events.

```typescript
interface Event {
  id: string;
  aggregateId: string;
  type: string;
  data: any;
  timestamp: Date;
  version: number;
}

class EventStore {
  constructor(private db: PrismaClient) {}

  async append(event: Omit<Event, 'id' | 'timestamp'>): Promise<void> {
    await this.db.event.create({
      data: {
        ...event,
        id: crypto.randomUUID(),
        timestamp: new Date(),
      },
    });
  }

  async getEvents(aggregateId: string): Promise<Event[]> {
    return this.db.event.findMany({
      where: { aggregateId },
      orderBy: { version: 'asc' },
    });
  }

  async getEventsByType(type: string): Promise<Event[]> {
    return this.db.event.findMany({
      where: { type },
      orderBy: { timestamp: 'asc' },
    });
  }
}

// Usage
class OrderAggregate {
  private events: Event[] = [];

  constructor(private eventStore: EventStore) {}

  async createOrder(data: CreateOrderInput): Promise<void> {
    // Validate and apply business rules
    this.apply('OrderCreated', data);

    // Store event
    await this.eventStore.append({
      aggregateId: this.id,
      type: 'OrderCreated',
      data,
      version: this.version,
    });
  }

  private apply(type: string, data: any) {
    // Apply event to state
    this.events.push({
      id: crypto.randomUUID(),
      aggregateId: this.id,
      type,
      data,
      timestamp: new Date(),
      version: this.version + 1,
    });
    this.version++;
  }
}
```

## 9. CQRS Pattern

Separate read and write models.

```typescript
// Command side
class OrderCommandHandler {
  constructor(
    private writeDb: PrismaClient,
    private eventBus: EventBus,
  ) {}

  async createOrder(cmd: CreateOrderCommand): Promise<void> {
    // Write to normalized database
    const order = await this.writeDb.order.create({
      data: {
        userId: cmd.userId,
        items: cmd.items,
        status: 'pending',
      },
    });

    // Publish event
    await this.eventBus.publish('order.created', {
      orderId: order.id,
      userId: cmd.userId,
      total: order.total,
    });
  }
}

// Query side
class OrderQueryHandler {
  constructor(private readDb: ReadDatabase) {}

  async getOrder(id: string): Promise<OrderView> {
    // Read from denormalized view
    return this.readDb.orders.findById(id);
  }

  async listOrders(filters: OrderFilters): Promise<PaginatedResult<OrderView>> {
    return this.readDb.orders.findMany(filters);
  }
}

// Projection (updates read model from events)
class OrderProjection {
  constructor(private readDb: ReadDatabase) {}

  async handle(event: Event): Promise<void> {
    switch (event.type) {
      case 'order.created':
        await this.readDb.orders.create({
          id: event.data.orderId,
          userId: event.data.userId,
          total: event.data.total,
          status: 'pending',
        });
        break;
      case 'order.completed':
        await this.readDb.orders.update(event.data.orderId, {
          status: 'completed',
        });
        break;
    }
  }
}
```

## 10. Rate Limiter Pattern

```typescript
class RateLimiter {
  constructor(private redis: RedisClient) {}

  async check(
    key: string,
    limit: number,
    windowMs: number,
  ): Promise<{ allowed: boolean; remaining: number; resetAt: number }> {
    const now = Date.now();
    const windowStart = now - windowMs;

    // Remove old entries
    await this.redis.zremrangebyscore(key, 0, windowStart);

    // Count requests in window
    const count = await this.redis.zcard(key);

    if (count >= limit) {
      const resetAt = await this.redis.zrange(key, 0, 0, 'WITHSCORES');
      return {
        allowed: false,
        remaining: 0,
        resetAt: parseInt(resetAt[1]) + windowMs,
      };
    }

    // Add current request
    await this.redis.zadd(key, now, `${now}`);
    await this.redis.expire(key, windowMs / 1000);

    return {
      allowed: true,
      remaining: limit - count - 1,
      resetAt: now + windowMs,
    };
  }
}
```

## Pattern Selection Guide

| Pattern | Use When | Avoid When |
|---------|----------|------------|
| Repository | Multiple data sources, testing | Simple CRUD |
| Service Layer | Complex business logic | Simple operations |
| Middleware | Cross-cutting concerns | One-off logic |
| Circuit Breaker | External service calls | Single service |
| Retry | Transient failures | Permanent failures |
| Unit of Work | Multiple related operations | Single operations |
| Cache-Aside | Read-heavy, expensive queries | Write-heavy |
| Event Sourcing | Audit trail, complex state | Simple state |
| CQRS | Different read/write patterns | Simple CRUD |
| Rate Limiter | Public APIs | Internal services |
