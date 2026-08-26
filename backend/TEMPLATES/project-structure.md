# Backend Project Structure Template

## Directory Layout

```
[PROJECT_NAME]/
├── .github/
│   └── workflows/
│       ├── ci.yml
│       ├── deploy.yml
│       └── security.yml
├── docs/
│   ├── api/
│   │   └── openapi.yaml
│   ├── architecture/
│   │   ├── system.md
│   │   └── database.md
│   └── guides/
│       ├── setup.md
│       ├── deployment.md
│       └── contributing.md
├── k8s/
│   ├── base/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── ingress.yaml
│   └── overlays/
│       ├── staging/
│       └── production/
├── scripts/
│   ├── seed.ts
│   ├── migrate.ts
│   └── backup.ts
├── src/
│   ├── config/
│   │   ├── app.ts
│   │   ├── database.ts
│   │   ├── redis.ts
│   │   └── index.ts
│   ├── middleware/
│   │   ├── auth.ts
│   │   ├── error-handler.ts
│   │   ├── rate-limiter.ts
│   │   ├── validator.ts
│   │   └── logger.ts
│   ├── modules/
│   │   ├── auth/
│   │   │   ├── auth.controller.ts
│   │   │   ├── auth.service.ts
│   │   │   ├── auth.routes.ts
│   │   │   ├── auth.types.ts
│   │   │   ├── auth.validation.ts
│   │   │   └── auth.test.ts
│   │   ├── users/
│   │   │   ├── users.controller.ts
│   │   │   ├── users.service.ts
│   │   │   ├── users.repository.ts
│   │   │   ├── users.routes.ts
│   │   │   ├── users.types.ts
│   │   │   ├── users.validation.ts
│   │   │   ├── users.test.ts
│   │   │   └── users.integration.test.ts
│   │   └── orders/
│   │       ├── orders.controller.ts
│   │       ├── orders.service.ts
│   │       ├── orders.repository.ts
│   │       ├── orders.routes.ts
│   │       ├── orders.types.ts
│   │       ├── orders.validation.ts
│   │       └── orders.test.ts
│   ├── shared/
│   │   ├── errors/
│   │   │   ├── app-error.ts
│   │   │   ├── not-found.ts
│   │   │   ├── validation.ts
│   │   │   └── index.ts
│   │   ├── utils/
│   │   │   ├── crypto.ts
│   │   │   ├── date.ts
│   │   │   └── response.ts
│   │   └── types/
│   │       ├── index.ts
│   │       └── api.ts
│   ├── app.ts
│   └── server.ts
├── test/
│   ├── setup.ts
│   ├── helpers/
│   │   ├── database.ts
│   │   ├── auth.ts
│   │   └── factories/
│   │       └── user.factory.ts
│   └── fixtures/
│       └── users.json
├── .env.example
├── .gitignore
├── .eslintrc.js
├── .prettierrc
├── docker-compose.yml
├── Dockerfile
├── Dockerfile.dev
├── jest.config.ts
├── package.json
├── README.md
├── tsconfig.json
└── vitest.config.ts
```

## Template Files

### package.json

```json
{
  "name": "[PROJECT_NAME]",
  "version": "1.0.0",
  "description": "[PROJECT_DESCRIPTION]",
  "main": "dist/server.js",
  "scripts": {
    "dev": "tsx watch src/server.ts",
    "build": "tsc",
    "start": "node dist/server.js",
    "lint": "eslint src --ext .ts",
    "lint:fix": "eslint src --ext .ts --fix",
    "typecheck": "tsc --noEmit",
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "test:integration": "jest --config jest.integration.config.ts",
    "test:e2e": "jest --config jest.e2e.config.ts",
    "db:migrate": "prisma migrate dev",
    "db:generate": "prisma generate",
    "db:seed": "tsx scripts/seed.ts",
    "db:reset": "prisma migrate reset",
    "db:studio": "prisma studio",
    "prepare": "husky install"
  },
  "dependencies": {
    "express": "^4.18.2",
    "prisma": "^5.7.0",
    "@prisma/client": "^5.7.0",
    "zod": "^3.22.4",
    "jsonwebtoken": "^9.0.2",
    "argon2": "^0.31.2",
    "ioredis": "^5.3.2",
    "helmet": "^7.1.0",
    "cors": "^2.8.5",
    "pino": "^8.17.2",
    "pino-pretty": "^10.3.1",
    "uuid": "^9.0.0"
  },
  "devDependencies": {
    "typescript": "^5.3.3",
    "@types/node": "^20.10.6",
    "@types/express": "^4.17.21",
    "@types/jsonwebtoken": "^9.0.5",
    "@types/cors": "^2.8.17",
    "@types/uuid": "^9.0.7",
    "tsx": "^4.7.0",
    "jest": "^29.7.0",
    "ts-jest": "^29.1.1",
    "@types/jest": "^29.5.11",
    "supertest": "^6.3.3",
    "@types/supertest": "^6.0.2",
    "eslint": "^8.56.0",
    "@typescript-eslint/eslint-plugin": "^6.17.0",
    "@typescript-eslint/parser": "^6.17.0",
    "prettier": "^3.1.1",
    "husky": "^8.0.3",
    "lint-staged": "^15.2.0"
  }
}
```

### tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
```

### app.ts

```typescript
import express from 'express';
import cors from 'cors';
import helmet from 'helmet';
import { pino } from 'pino';

import { authRouter } from '@/modules/auth/auth.routes';
import { usersRouter } from '@/modules/users/users.routes';
import { ordersRouter } from '@/modules/orders/orders.routes';
import { errorHandler } from '@/middleware/error-handler';
import { requestLogger } from '@/middleware/logger';
import { rateLimiter } from '@/middleware/rate-limiter';

const app = express();
const logger = pino({ level: process.env.LOG_LEVEL || 'info' });

// Security middleware
app.use(helmet());
app.use(cors({
  origin: process.env.CORS_ORIGIN || 'http://localhost:3000',
  credentials: true,
}));

// Body parsing
app.use(express.json({ limit: '1mb' }));
app.use(express.urlencoded({ extended: true, limit: '1mb' }));

// Logging
app.use(requestLogger);

// Rate limiting
app.use(rateLimiter);

// Health check
app.get('/health', (req, res) => {
  res.json({ status: 'healthy', timestamp: new Date().toISOString() });
});

// API routes
app.use('/api/v1/auth', authRouter);
app.use('/api/v1/users', usersRouter);
app.use('/api/v1/orders', ordersRouter);

// Error handling
app.use(errorHandler);

export default app;
```

### server.ts

```typescript
import app from './app';
import { prisma } from '@/config/database';
import { redis } from '@/config/redis';

const PORT = process.env.PORT || 3000;

async function start() {
  try {
    // Connect to database
    await prisma.$connect();
    console.log('Connected to database');

    // Connect to Redis
    await redis.connect();
    console.log('Connected to Redis');

    // Start server
    app.listen(PORT, () => {
      console.log(`Server running on port ${PORT}`);
    });
  } catch (error) {
    console.error('Failed to start server:', error);
    process.exit(1);
  }
}

// Graceful shutdown
process.on('SIGTERM', async () => {
  console.log('SIGTERM received, shutting down...');
  await prisma.$disconnect();
  await redis.disconnect();
  process.exit(0);
});

start();
```

### .env.example

```bash
# Application
APP_NAME=[PROJECT_NAME]
APP_ENV=development
APP_PORT=3000
APP_URL=http://localhost:3000
CORS_ORIGIN=http://localhost:3000

# Database
DATABASE_URL=postgresql://postgres:password@localhost:5432/[PROJECT]_dev

# Redis
REDIS_URL=redis://localhost:6379

# Authentication
JWT_SECRET=your-secret-key-here
JWT_ALGORITHM=RS256
JWT_ACCESS_EXPIRY=15m
JWT_REFRESH_EXPIRY=7d

# Logging
LOG_LEVEL=debug
```

### .gitignore

```gitignore
# Dependencies
node_modules/

# Build
dist/
build/

# Environment
.env
.env.local
.env.*.local

# IDE
.vscode/
.idea/
*.swp
*.swo

# OS
.DS_Store
Thumbs.db

# Testing
coverage/

# Logs
*.log
logs/

# Database
*.db
```

### docker-compose.yml

```yaml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile.dev
    ports:
      - "3000:3000"
    volumes:
      - .:/app
      - /app/node_modules
    environment:
      - DATABASE_URL=postgresql://postgres:password@db:5432/[PROJECT]_dev
      - REDIS_URL=redis://redis:6379
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy

  db:
    image: postgres:15-alpine
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
      POSTGRES_DB: [PROJECT]_dev
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
```
