# AI Testing Approach for Backend

## Testing Philosophy

AI agents should:

1. **Write meaningful tests** that verify behavior, not implementation
2. **Cover edge cases** that humans might miss
3. **Ensure security** through security-focused tests
4. **Validate performance** through load test suggestions
5. **Maintain test quality** as code evolves

## Test Generation Guidelines

### When Writing Unit Tests

```typescript
// Structure tests using AAA pattern
describe('UserService', () => {
  describe('createUser', () => {
    // Happy path
    it('should create user with valid data', async () => {
      // Arrange
      const input = {
        email: 'test@example.com',
        name: 'Test User',
        password: 'SecurePass123!',
      };
      const mockUser = { id: '1', ...input, createdAt: new Date() };
      mockRepo.create.mockResolvedValue(mockUser);

      // Act
      const result = await service.createUser(input);

      // Assert
      expect(result).toEqual(mockUser);
      expect(mockRepo.create).toHaveBeenCalledWith(
        expect.objectContaining({ email: input.email }),
      );
    });

    // Validation tests
    it('should reject invalid email format', async () => {
      const input = { email: 'not-an-email', name: 'Test', password: 'pass' };

      await expect(service.createUser(input))
        .rejects.toThrow(ValidationError);
    });

    it('should reject weak password', async () => {
      const input = { email: 'test@example.com', name: 'Test', password: '123' };

      await expect(service.createUser(input))
        .rejects.toThrow(ValidationError);
    });

    // Edge cases
    it('should handle database connection error', async () => {
      const input = { email: 'test@example.com', name: 'Test', password: 'SecurePass123!' };
      mockRepo.create.mockRejectedValue(new Error('Connection refused'));

      await expect(service.createUser(input))
        .rejects.toThrow(AppError);
    });

    it('should handle concurrent duplicate email', async () => {
      const input = { email: 'test@example.com', name: 'Test', password: 'SecurePass123!' };
      mockRepo.create.mockRejectedValue({ code: '23505' }); // Unique constraint

      await expect(service.createUser(input))
        .rejects.toThrow(ConflictError);
    });
  });
});
```

### When Writing Integration Tests

```typescript
// Test complete API flows
describe('POST /api/v1/users', () => {
  it('should create user and return 201', async () => {
    const response = await request(app)
      .post('/api/v1/users')
      .send({
        email: 'new@example.com',
        name: 'New User',
        password: 'SecurePass123!',
      })
      .expect(201);

    expect(response.body.data).toHaveProperty('id');
    expect(response.body.data.email).toBe('new@example.com');
    expect(response.body.data).not.toHaveProperty('password');
    expect(response.body.data).not.toHaveProperty('passwordHash');
  });

  it('should return 400 for invalid data', async () => {
    const response = await request(app)
      .post('/api/v1/users')
      .send({
        email: 'invalid',
        name: '',
        password: 'weak',
      })
      .expect(400);

    expect(response.body.error.code).toBe('VALIDATION_ERROR');
    expect(response.body.error.details).toHaveLength(3);
  });

  it('should return 409 for duplicate email', async () => {
    // Create first user
    await request(app)
      .post('/api/v1/users')
      .send({
        email: 'existing@example.com',
        name: 'Existing',
        password: 'SecurePass123!',
      });

    // Try to create duplicate
    await request(app)
      .post('/api/v1/users')
      .send({
        email: 'existing@example.com',
        name: 'Duplicate',
        password: 'SecurePass123!',
      })
      .expect(409);
  });
});
```

### Security Test Cases

```typescript
describe('Security Tests', () => {
  describe('Authentication', () => {
    it('should reject request without token', async () => {
      await request(app)
        .get('/api/v1/users')
        .expect(401);
    });

    it('should reject invalid token', async () => {
      await request(app)
        .get('/api/v1/users')
        .set('Authorization', 'Bearer invalid-token')
        .expect(401);
    });

    it('should reject expired token', async () => {
      const expiredToken = generateExpiredToken();
      await request(app)
        .get('/api/v1/users')
        .set('Authorization', `Bearer ${expiredToken}`)
        .expect(401);
    });
  });

  describe('Authorization', () => {
    it('should enforce RBAC', async () => {
      const viewerToken = generateToken({ role: 'viewer' });

      await request(app)
        .post('/api/v1/users')
        .set('Authorization', `Bearer ${viewerToken}`)
        .send({ email: 'test@example.com', name: 'Test', password: 'Pass123!' })
        .expect(403);
    });

    it('should prevent horizontal privilege escalation', async () => {
      const user1Token = generateToken({ id: 'user-1' });
      const user2Id = 'user-2';

      await request(app)
        .delete(`/api/v1/users/${user2Id}`)
        .set('Authorization', `Bearer ${user1Token}`)
        .expect(403);
    });
  });

  describe('Input Validation', () => {
    it('should prevent SQL injection', async () => {
      await request(app)
        .get("/api/v1/users?q='; DROP TABLE users; --")
        .set('Authorization', `Bearer ${adminToken}`)
        .expect(200); // Should not error

      // Verify table still exists
      const response = await request(app)
        .get('/api/v1/users')
        .set('Authorization', `Bearer ${adminToken}`)
        .expect(200);

      expect(response.body.data).toBeDefined();
    });

    it('should prevent XSS in responses', async () => {
      const xssPayload = '<script>alert("xss")</script>';
      await request(app)
        .post('/api/v1/users')
        .send({ email: 'test@example.com', name: xssPayload, password: 'Pass123!' });

      const response = await request(app)
        .get('/api/v1/users')
        .set('Authorization', `Bearer ${adminToken}`);

      expect(response.body.data[0].name).not.toContain('<script>');
    });

    it('should enforce request size limits', async () => {
      const largePayload = 'x'.repeat(10 * 1024 * 1024); // 10MB

      await request(app)
        .post('/api/v1/users')
        .send({ data: largePayload })
        .expect(413);
    });
  });
});
```

### Performance Test Suggestions

```typescript
// Load test scenarios to consider
const loadTestScenarios = {
  'list_users': {
    method: 'GET',
    path: '/api/v1/users',
    targetRPS: 100,
    threshold: { p95: 200 }, // ms
  },
  'get_user': {
    method: 'GET',
    path: '/api/v1/users/:id',
    targetRPS: 500,
    threshold: { p95: 100 },
  },
  'create_user': {
    method: 'POST',
    path: '/api/v1/users',
    targetRPS: 50,
    threshold: { p95: 500 },
  },
};

// Suggested k6 test
const loadTestScript = `
import http from 'k6/http';
import { check } from 'k6';

export const options = {
  stages: [
    { duration: '1m', target: 50 },
    { duration: '5m', target: 50 },
    { duration: '1m', target: 100 },
    { duration: '5m', target: 100 },
    { duration: '1m', target: 0 },
  ],
  thresholds: {
    http_req_duration: ['p(95)<200'],
  },
};

export default function () {
  const res = http.get('http://localhost:3000/api/v1/users');
  check(res, { 'status is 200': (r) => r.status === 200 });
}
`;
```

## Test Data Management

### Factory Pattern

```typescript
// test/factories/user.factory.ts
class UserFactory {
  static build(overrides: Partial<CreateUserDTO> = {}): CreateUserDTO {
    return {
      email: `user-${Date.now()}@example.com`,
      name: `Test User ${Math.random().toString(36).slice(2)}`,
      password: 'SecurePass123!',
      role: 'user',
      ...overrides,
    };
  }

  static async create(overrides: Partial<CreateUserDTO> = {}): Promise<User> {
    const data = this.build(overrides);
    return userService.create(data);
  }

  static async createMany(
    count: number,
    overrides: Partial<CreateUserDTO> = {},
  ): Promise<User[]> {
    return Promise.all(
      Array.from({ length: count }, () => this.create(overrides)),
    );
  }
}

// Usage in tests
const admin = await UserFactory.create({ role: 'admin' });
const users = await UserFactory.createMany(10);
```

### Test Database Management

```typescript
// test/helpers/database.ts
class TestDatabase {
  static async reset(): Promise<void> {
    // Reset to clean state
    await db.$executeRaw`TRUNCATE TABLE users CASCADE`;
    await db.$executeRaw`TRUNCATE TABLE orders CASCADE`;
  }

  static async seed(): Promise<void> {
    // Add test data
    await UserFactory.create({ email: 'admin@example.com', role: 'admin' });
    await UserFactory.create({ email: 'user@example.com', role: 'user' });
  }

  static async transaction<T>(fn: () => Promise<T>): Promise<T> {
    return db.$transaction(async (tx) => {
      const result = await fn();
      await tx.$rollback(); // Rollback after test
      return result;
    });
  }
}
```

## Mocking Guidelines

### When to Mock

- **External services** (payment providers, email services)
- **Time-dependent code** (for deterministic tests)
- **Random values** (UUIDs, tokens)
- **File system** (for unit tests)

### When NOT to Mock

- **Database** (use test database instead)
- **Internal service calls** (test integration)
- **Authentication** (use real tokens)

### Mock Examples

```typescript
// Mock external service
jest.mock('@/services/email', () => ({
  sendEmail: jest.fn().mockResolvedValue({ success: true }),
}));

// Mock time
jest.useFakeTimers();
jest.setSystemTime(new Date('2024-01-15'));

// Mock random values
jest.mock('uuid', () => ({
  v4: jest.fn(() => 'test-uuid-123'),
}));
```

## CI Integration

```yaml
# .github/workflows/test.yml
test:
  runs-on: ubuntu-latest
  services:
    postgres:
      image: postgres:15
      env:
        POSTGRES_DB: test
        POSTGRES_USER: test
        POSTGRES_PASSWORD: test
    redis:
      image: redis:7

  steps:
    - name: Run Unit Tests
      run: npm run test:unit

    - name: Run Integration Tests
      run: npm run test:integration
      env:
        DATABASE_URL: postgresql://test:test@localhost:5432/test
        REDIS_URL: redis://localhost:6379

    - name: Upload Coverage
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage/lcov.info
```

## Test Quality Checklist

```yaml
test_quality:
  - [ ] Tests are independent (no shared state)
  - [ ] Tests are deterministic (no flaky tests)
  - [ ] Tests cover happy path and error cases
  - [ ] Tests include edge cases
  - [ ] Tests use meaningful assertions
  - [ ] Tests have clear names describing behavior
  - [ ] Tests don't test implementation details
  - [ ] Mocks are minimal and focused

coverage:
  - [ ] Unit test coverage > 80%
  - [ ] Integration test coverage for all endpoints
  - [ ] Security tests for auth/authz
  - [ ] Error handling tests
  - [ ] Performance benchmarks documented
```
