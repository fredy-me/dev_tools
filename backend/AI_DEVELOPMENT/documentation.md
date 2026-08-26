# AI Documentation Standards

## Documentation Philosophy

AI agents should maintain documentation that is:

1. **Accurate**: Reflects current code state
2. **Concise**: Gets to the point quickly
3. **Actionable**: Helps developers accomplish tasks
4. **Complete**: Covers all necessary information
5. **Maintained**: Updated when code changes

## API Documentation

### OpenAPI/Swagger Standards

```yaml
# Every endpoint must have:
openapi: 3.1.0
paths:
  /users:
    post:
      summary: Create a new user
      description: |
        Creates a new user account with the provided information.

        **Rate Limit**: 10 requests per minute
        **Authentication**: Required (Bearer token)
        **Permissions**: `user:create`
      operationId: createUser
      tags: [Users]
      security:
        - bearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateUserRequest'
            examples:
              basic:
                summary: Basic user creation
                value:
                  email: john@example.com
                  name: John Doe
                  password: SecurePass123!
      responses:
        '201':
          description: User created successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/UserResponse'
        '400':
          $ref: '#/components/responses/ValidationError'
        '409':
          $ref: '#/components/responses/ConflictError'
```

### Endpoint Documentation Template

```typescript
/**
 * Creates a new user in the system.
 *
 * @description
 * This endpoint creates a new user with the provided information.
 * The email must be unique and the password must meet security requirements.
 *
 * @authentication Required - Bearer token
 * @authorization Required - `user:create` permission
 * @rateLimit 10 requests per minute
 *
 * @param {CreateUserRequest} body - User creation data
 * @returns {UserResponse} Created user object
 *
 * @throws {ValidationError} If input data is invalid
 * @throws {ConflictError} If email already exists
 * @throws {UnauthorizedError} If not authenticated
 * @throws {ForbiddenError} If insufficient permissions
 *
 * @example Request
 * ```json
 * {
 *   "email": "john@example.com",
 *   "name": "John Doe",
 *   "password": "SecurePass123!"
 * }
 * ```
 *
 * @example Response
 * ```json
 * {
 *   "data": {
 *     "id": "usr_abc123",
 *     "email": "john@example.com",
 *     "name": "John Doe",
 *     "role": "user",
 *     "created_at": "2024-01-15T10:00:00Z"
 *   }
 * }
 * ```
 *
 * @see {@link /docs/errors.md|Error Codes} for error details
 */
async function createUser(data: CreateUserDTO): Promise<User> {
  // Implementation
}
```

## Code Documentation

### Inline Comments

```typescript
// Use comments to explain WHY, not WHAT
// Bad: Increment counter
counter++;

// Good: Exponential backoff requires doubling the delay
counter *= 2;

// Use TODO comments sparingly and with tracking
// TODO(JIRA-1234): Remove this after migrating to new auth system
const legacyAuth = await oldAuthService.validate(token);

// Use FIXME for known issues that need attention
// FIXME: This is a workaround for upstream bug #5678
// Remove when service is updated to v2.1
if (response.headers['x-legacy-header']) {
  processLegacyResponse(response);
}
```

### Function Documentation

```typescript
/**
 * Calculates the total price for an order including tax and discounts.
 *
 * @param items - Array of order items with quantity and unit price
 * @param discountCode - Optional discount code to apply
 * @param taxRate - Tax rate as decimal (e.g., 0.1 for 10%)
 * @returns Total price in cents (e.g., 1999 for $19.99)
 *
 * @example
 * const total = calculateTotal(
 *   [{ price: 1000, quantity: 2 }],
 *   'SAVE10',
 *   0.08
 * );
 * // Returns: 1728 (meaning $17.28)
 */
function calculateTotal(
  items: OrderItem[],
  discountCode?: string,
  taxRate: number = 0,
): number {
  const subtotal = items.reduce(
    (sum, item) => sum + item.price * item.quantity,
    0,
  );

  const discount = discountCode
    ? applyDiscount(subtotal, discountCode)
    : 0;

  const tax = (subtotal - discount) * taxRate;

  return Math.round(subtotal - discount + tax);
}
```

## Architecture Documentation

### System Overview

```markdown
# System Architecture

## Overview
[Brief description of the system]

## Components
- **API Gateway**: Routes requests, handles auth, rate limiting
- **User Service**: Manages user accounts and authentication
- **Order Service**: Handles order processing and management
- **Payment Service**: Integrates with payment providers

## Data Flow
[Mermaid diagram showing request flow]

## Technology Stack
- Runtime: Node.js 20 LTS
- Framework: Express.js
- Database: PostgreSQL 15
- Cache: Redis 7
- Message Queue: RabbitMQ

## Deployment
- Containerized with Docker
- Orchestrated with Kubernetes
- CI/CD via GitHub Actions
```

### Component Documentation

```markdown
# User Service

## Responsibilities
- User registration and authentication
- Profile management
- Role-based access control

## API Endpoints
- `POST /api/v1/users` - Create user
- `GET /api/v1/users/:id` - Get user
- `PATCH /api/v1/users/:id` - Update user
- `DELETE /api/v1/users/:id` - Delete user

## Database Tables
- `users` - User accounts
- `user_sessions` - Active sessions
- `user_roles` - Role assignments

## External Dependencies
- Email service (for verification)
- OAuth providers (Google, GitHub)

## Configuration
See `.env.example` for required environment variables.
```

## Runbook Documentation

```markdown
# Runbook: User Service Issues

## High Error Rate

### Symptoms
- Error rate > 5% in Grafana
- Users reporting login failures

### Investigation
1. Check service logs: `kubectl logs -l app=user-service --tail=100`
2. Check database connections: `SELECT count(*) FROM pg_stat_activity`
3. Check Redis connectivity: `redis-cli ping`

### Common Causes
1. **Database connection pool exhausted**
   - Solution: Restart service or increase pool size
2. **Invalid JWT signing key**
   - Solution: Verify JWT_SECRET environment variable
3. **External service down**
   - Solution: Check email service status

### Resolution
1. If database issue: `kubectl rollout restart deployment/user-service`
2. If config issue: Update ConfigMap and restart
3. If external service: Enable fallback mode

### Prevention
- Set up connection pool monitoring
- Configure circuit breakers for external services
- Regular secret rotation
```

## Changelog Documentation

```markdown
# Changelog

## [2.1.0] - 2024-01-15

### Added
- User profile picture upload
- OAuth2 login with Google
- Rate limiting per API key

### Changed
- Upgraded to Node.js 20 LTS
- Migrated from Jest to Vitest

### Fixed
- Fixed race condition in order processing
- Fixed memory leak in WebSocket handler

### Security
- Updated dependencies for CVE-2024-XXXX
- Added CSRF protection for state-changing operations

## [2.0.0] - 2024-01-01

### Breaking Changes
- Removed v1 API endpoints
- Changed authentication to JWT-only
- Removed deprecated `username` field

See [Migration Guide](./MIGRATION-v2.md) for upgrade instructions.
```

## README Standards

```markdown
# Project Name

[![CI](https://github.com/org/project/actions/workflows/ci.yml/badge.svg)](https://github.com/org/project/actions)
[![Coverage](https://codecov.io/gh/org/project/branch/main/graph/badge.svg)](https://codecov.io/gh/org/project)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

Brief description of what this project does.

## Quick Start

### Prerequisites
- Node.js 20+
- Docker & Docker Compose
- PostgreSQL 15+

### Installation
```bash
git clone https://github.com/org/project.git
cd project
npm install
```

### Development
```bash
# Start infrastructure
docker-compose up -d

# Run migrations
npm run db:migrate

# Start dev server
npm run dev
```

### Testing
```bash
npm run test           # Unit tests
npm run test:integration  # Integration tests
npm run test:e2e      # E2E tests
```

## API Documentation

- [OpenAPI Spec](./docs/openapi.yaml)
- [API Guide](./docs/api-guide.md)
- [Authentication](./docs/authentication.md)

## Architecture

- [System Overview](./docs/architecture.md)
- [Database Schema](./docs/database.md)
- [Deployment](./docs/deployment.md)

## Contributing

See [CONTRIBUTING.md](./CONTRIBUTING.md) for guidelines.

## License

[MIT](./LICENSE)
```

## Documentation Checklist

```yaml
code_changes:
  - [ ] JSDoc/function docs updated
  - [ ] README updated (if setup changes)
  - [ ] CHANGELOG updated
  - [ ] API docs updated (if endpoints changed)
  - [ ] Migration guide (if breaking changes)

new_features:
  - [ ] Feature documented in README
  - [ ] API endpoint documented in OpenAPI
  - [ ] Configuration documented
  - [ ] Examples provided
  - [ ] Troubleshooting section added

bug_fixes:
  - [ ] Root cause documented
  - [ ] Fix documented in CHANGELOG
  - [ ] Runbook updated (if operational)
```
