# AI Code Review Guidelines

## Review Scope

AI agents should review backend code for:

1. **Correctness**: Does the code do what it's supposed to do?
2. **Security**: Are there any vulnerabilities?
3. **Performance**: Are there any inefficiencies?
4. **Maintainability**: Is the code clean and readable?
5. **Testing**: Is the code properly tested?

## Security Review

### Critical Issues (Must Fix)

```typescript
// SQL Injection Vulnerability
// BAD
const query = `SELECT * FROM users WHERE id = '${userId}'`;
const result = await db.query(query);

// GOOD
const result = await db.query('SELECT * FROM users WHERE id = $1', [userId]);

// Path Traversal
// BAD
const filePath = path.join(uploadDir, req.params.filename);
const content = fs.readFileSync(filePath);

// GOOD
const filename = path.basename(req.params.filename); // Sanitize
const filePath = path.join(uploadDir, filename);
if (!filePath.startsWith(uploadDir)) {
  throw new Error('Invalid path');
}
const content = fs.readFileSync(filePath);

// NoSQL Injection
// BAD
const user = await db.users.findOne({ email: req.body.email });

// GOOD
const user = await db.users.findOne({
  email: { $eq: req.body.email } // Explicit operator
});

// Or better, use schema validation first
const validated = LoginSchema.parse(req.body);
const user = await db.users.findOne({ email: validated.email });
```

### Authentication Issues

```typescript
// Missing authentication
// BAD
router.delete('/users/:id', controller.delete);

// GOOD
router.delete('/users/:id', authenticate, controller.delete);

// Broken authentication
// BAD - Token not verified
const user = jwt.decode(token);

// GOOD - Token properly verified
const user = jwt.verify(token, publicKey, { algorithms: ['RS256'] });

// Weak password hashing
// BAD
const hash = await bcrypt.hash(password, 10);

// GOOD
const hash = await argon2.hash(password, {
  type: argon2.argon2id,
  memoryCost: 65536,
  timeCost: 3,
  parallelism: 4,
});
```

### Authorization Issues

```typescript
// Missing authorization check
// BAD - Any user can delete any order
router.delete('/orders/:id', authenticate, controller.delete);

// GOOD - Only owner or admin can delete
router.delete('/orders/:id',
  authenticate,
  authorize('order:delete'),
  controller.delete
);

// In controller
async delete(req: Request, res: Response) {
  const order = await this.orderService.findById(req.params.id);

  if (order.userId !== req.user.id && req.user.role !== 'admin') {
    throw new ForbiddenError('Cannot delete this order');
  }

  await this.orderService.delete(req.params.id);
  res.status(204).send();
}
```

## Performance Review

### N+1 Query Problems

```typescript
// BAD - N+1 queries
const users = await db.users.findMany();
for (const user of users) {
  user.orders = await db.orders.findByUserId(user.id); // N queries!
}

// GOOD - Single query with joins
const users = await db.users.findMany({
  include: { orders: true },
});

// GOOD - DataLoader for GraphQL
const userLoader = new DataLoader(async (ids) => {
  const users = await db.users.findByIds(ids);
  return ids.map(id => users.find(u => u.id === id));
});
```

### Missing Database Indexes

```typescript
// Review for missing indexes on:
// - Foreign keys
// - Columns used in WHERE clauses
// - Columns used in ORDER BY
// - Columns used in JOIN conditions

// BAD - Missing index
const orders = await db.orders.findMany({
  where: { userId: user.id, status: 'pending' },
  orderBy: { createdAt: 'desc' },
});

// GOOD - Add composite index
// CREATE INDEX idx_orders_user_status_created
// ON orders(user_id, status, created_at DESC);
```

### Inefficient Queries

```typescript
// BAD - Fetching all columns when only some needed
const users = await db.users.findMany();

// GOOD - Select only needed fields
const users = await db.users.findMany({
  select: { id: true, name: true, email: true },
});

// BAD - No pagination
const orders = await db.orders.findMany();

// GOOD - Always paginate
const orders = await db.orders.findMany({
  take: 20,
  skip: (page - 1) * 20,
  orderBy: { createdAt: 'desc' },
});
```

## Error Handling Review

### Missing Error Handling

```typescript
// BAD - No error handling
async function processPayment(orderId: string) {
  const order = await db.orders.findById(orderId);
  await paymentProvider.charge(order.total);
  await db.orders.update(orderId, { status: 'paid' });
}

// GOOD - Proper error handling
async function processPayment(orderId: string): Promise<void> {
  const order = await db.orders.findById(orderId);
  if (!order) {
    throw new NotFoundError('Order', orderId);
  }

  try {
    await paymentProvider.charge(order.total);
  } catch (error) {
    logger.error({ error, orderId }, 'Payment failed');
    throw new PaymentError('Payment processing failed');
  }

  try {
    await db.orders.update(orderId, { status: 'paid' });
  } catch (error) {
    logger.error({ error, orderId }, 'Failed to update order status');
    // Payment succeeded but update failed - need manual reconciliation
    await alertService.send({
      severity: 'critical',
      message: `Order ${orderId} paid but status not updated`,
    });
    throw new AppError('Order update failed', 'ORDER_UPDATE_FAILED', 500);
  }
}
```

### Swallowed Errors

```typescript
// BAD - Error silently ignored
try {
  await sendEmail(user.email, template);
} catch (e) {
  // Email failed, oh well
}

// GOOD - Proper error handling
try {
  await sendEmail(user.email, template);
} catch (error) {
  logger.warn({ error, userId: user.id, template }, 'Email delivery failed');
  // Queue for retry
  await emailQueue.add({ userId: user.id, template });
}
```

## Input Validation Review

### Missing Validation

```typescript
// BAD - No validation
router.post('/users', async (req, res) => {
  const user = await createUser(req.body);
  res.json(user);
});

// GOOD - Comprehensive validation
const CreateUserSchema = z.object({
  body: z.object({
    email: z.string().email().max(255),
    name: z.string().min(1).max(255).trim(),
    password: z
      .string()
      .min(12)
      .max(128)
      .regex(/[A-Z]/, 'Must contain uppercase')
      .regex(/[a-z]/, 'Must contain lowercase')
      .regex(/[0-9]/, 'Must contain number')
      .regex(/[^A-Za-z0-9]/, 'Must contain special character'),
    role: z.enum(['user', 'viewer']).default('user'),
  }),
});

router.post('/users', validate(CreateUserSchema), async (req, res) => {
  const user = await createUser(req.body);
  res.status(201).json({ data: user });
});
```

## Code Quality Review

### Code Smells

```typescript
// BAD - Long parameter list
function createUser(
  email: string,
  name: string,
  password: string,
  role: string,
  department: string,
  phone: string,
  address: string,
) { ... }

// GOOD - Use DTO object
interface CreateUserDTO {
  email: string;
  name: string;
  password: string;
  role?: string;
  department?: string;
  phone?: string;
  address?: string;
}

function createUser(data: CreateUserDTO) { ... }

// BAD - Magic numbers
if (retryCount > 3) { ... }
const timeout = 5000;

// GOOD - Named constants
const MAX_RETRY_ATTEMPTS = 3;
const REQUEST_TIMEOUT_MS = 5000;

if (retryCount > MAX_RETRY_ATTEMPTS) { ... }
const timeout = REQUEST_TIMEOUT_MS;
```

## Review Comments Template

### For Security Issues

```
**Security Issue**: [Description]

**Risk Level**: Critical/High/Medium/Low

**Location**: `file.ts:line`

**Current Code**:
```typescript
// problematic code
```

**Recommended Fix**:
```typescript
// fixed code
```

**References**: [OWASP link or security guideline]
```

### For Performance Issues

```
**Performance Issue**: [Description]

**Impact**: [Expected improvement]

**Location**: `file.ts:line`

**Issue**: [Explanation of why it's slow]

**Suggestion**: [How to fix]

**Expected Result**: [Performance improvement]
```

### For General Code Quality

```
**Code Quality**: [Issue type]

**Location**: `file.ts:line`

**Description**: [What's wrong]

**Suggestion**: [How to improve]

**Priority**: High/Medium/Low
```

## Automated Checks

Before manual review, ensure:

```yaml
automated_checks:
  - name: ESLint
    command: npm run lint
    must_pass: true

  - name: TypeScript
    command: npm run typecheck
    must_pass: true

  - name: Unit Tests
    command: npm run test
    must_pass: true

  - name: Security Scan
    command: npm audit
    must_pass: true

  - name: Code Coverage
    command: npm run test:coverage
    threshold: 80
```
