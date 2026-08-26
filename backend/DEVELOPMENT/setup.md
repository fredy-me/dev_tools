# Backend Development Environment Setup

## Prerequisites

```yaml
system_requirements:
  os: "macOS 12+, Ubuntu 20.04+, Windows 11+ (WSL2)"
  ram: "16GB minimum, 32GB recommended"
  disk: "50GB free space"
  cpu: "4+ cores recommended"

tools:
  required:
    - name: "Node.js"
      version: "20 LTS"
      install: "nvm install 20 && nvm use 20"
    - name: "Docker"
      version: "24+"
      install: "https://docs.docker.com/get-docker/"
    - name: "Docker Compose"
      version: "2.20+"
      install: "Included with Docker Desktop"

  optional:
    - name: "Python"
      version: "3.11+"
      install: "pyenv install 3.11"
    - name: "Go"
      version: "1.21+"
      install: "https://go.dev/dl/"
    - name: "Java"
      version: "17+"
      install: "sdkman install java"
    - name: "PostgreSQL client"
      version: "15+"
      install: "brew install postgresql"
    - name: "Redis CLI"
      version: "7+"
      install: "brew install redis"
```

## Project Setup

### Quick Start (Node.js)

```bash
# Clone and setup
git clone https://github.com/[ORG]/[PROJECT].git
cd [PROJECT]

# Install dependencies
npm install

# Setup environment
cp .env.example .env
# Edit .env with your values

# Start infrastructure
docker-compose up -d

# Run migrations
npm run db:migrate

# Seed database
npm run db:seed

# Start development server
npm run dev
```

### Quick Start (Python/FastAPI)

```bash
# Clone and setup
git clone https://github.com/[ORG]/[PROJECT].git
cd [PROJECT]

# Create virtual environment
python -m venv venv
source venv/bin/activate  # Linux/macOS
# venv\Scripts\activate   # Windows

# Install dependencies
pip install -r requirements.txt
pip install -r requirements-dev.txt

# Setup environment
cp .env.example .env

# Start infrastructure
docker-compose up -d

# Run migrations
alembic upgrade head

# Seed database
python -m scripts.seed

# Start development server
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

## Environment Configuration

### .env.example

```bash
# Application
APP_NAME=[PROJECT_NAME]
APP_ENV=development
APP_PORT=3000
APP_URL=http://localhost:3000
API_URL=http://localhost:3000/api/v1

# Database
DATABASE_URL=postgresql://postgres:password@localhost:5432/[PROJECT]_dev
DATABASE_SSL=false
DATABASE_POOL_SIZE=10

# Redis
REDIS_URL=redis://localhost:6379
REDIS_PASSWORD=

# Authentication
JWT_SECRET=your-256-bit-secret-key
JWT_ALGORITHM=RS256
JWT_ACCESS_EXPIRY=15m
JWT_REFRESH_EXPIRY=7d

# OAuth2 (optional)
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
GITHUB_CLIENT_ID=
GITHUB_CLIENT_SECRET=

# Email
SMTP_HOST=smtp.ethereal.email
SMTP_PORT=587
SMTP_USER=
SMTP_PASS=

# Storage
S3_BUCKET=
S3_REGION=us-east-1
S3_ACCESS_KEY=
S3_SECRET_KEY=

# Monitoring
SENTRY_DSN=
LOG_LEVEL=debug

# Rate Limiting
RATE_LIMIT_FREE=100
RATE_LIMIT_PRO=1000
```

## Docker Compose

```yaml
# docker-compose.yml
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
      - NODE_ENV=development
      - DATABASE_URL=postgresql://postgres:password@db:5432/[PROJECT]_dev
      - REDIS_URL=redis://redis:6379
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    command: npm run dev

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
      - ./scripts/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5

  # Optional services
  mailhog:
    image: mailhog/mailhog
    ports:
      - "1025:1025"  # SMTP
      - "8025:8025"  # Web UI

  minio:
    image: minio/minio
    ports:
      - "9000:9000"
      - "9001:9001"
    volumes:
      - minio_data:/data
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    command: server /data --console-address ":9001"

volumes:
  postgres_data:
  redis_data:
  minio_data:
```

## IDE Configuration

### VS Code Extensions

```json
{
  "recommendations": [
    "dbaeumer.vscode-eslint",
    "esbenp.prettier-vscode",
    "bradlc.vscode-tailwindcss",
    "prisma.prisma",
    "mtxr.sqltools",
    "mtxr.sqltools-driver-pg",
    "humao.rest-client",
    "ms-azuretools.vscode-docker",
    "github.vscode-pull-request-github"
  ]
}
```

### VS Code Settings

```json
{
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit"
  },
  "typescript.preferences.importModuleSpecifier": "relative",
  "typescript.tsdk": "node_modules/typescript/lib"
}
```

## Database Setup

```bash
# Create database
createdb [PROJECT]_dev
createdb [PROJECT]_test

# Run migrations
npm run db:migrate

# Seed development data
npm run db:seed

# Reset database (dangerous!)
npm run db:reset

# View database
npm run db:studio  # Prisma Studio
```

## Git Hooks (Husky)

```bash
# Install husky
npx husky install

# Pre-commit: lint and format
npx husky add .husky/pre-commit "npx lint-staged"

# Commit message: conventional commits
npx husky add .husky/commit-msg "npx commitlint --edit $1"
```

### lint-staged Configuration

```json
{
  "*.{js,ts}": ["eslint --fix", "prettier --write"],
  "*.{json,md}": ["prettier --write"]
}
```

## Development Workflow

```mermaid
graph TB
    START[Start Development] --> BRANCH[Create Feature Branch]
    BRANCH --> CODE[Write Code]
    CODE --> TEST[Run Tests Locally]
    TEST --> LINT[Run Linter]
    LINT --> COMMIT[Commit Changes]
    COMMIT --> PUSH[Push to Remote]
    PUSH --> PR[Create Pull Request]
    PR --> CI[CI Pipeline Runs]
    CI --> REVIEW[Code Review]
    REVIEW --> MERGE[Merge to Main]
    MERGE --> DEPLOY[Deploy to Staging]
    DEPLOY --> E2E[Run E2E Tests]
    E2E --> PROD[Deploy to Production]
```

## Troubleshooting

```yaml
common_issues:
  - issue: "Port 5432 already in use"
    solution: "Stop existing PostgreSQL or change port in docker-compose.yml"

  - issue: "EACCES permission errors"
    solution: "Run: sudo chown -R $(whoami) ~/.docker"

  - issue: "Database connection refused"
    solution: "Ensure Docker is running: docker ps"

  - issue: "Module not found errors"
    solution: "Clear node_modules: rm -rf node_modules && npm install"

  - issue: "TypeScript compilation errors"
    solution: "Run: npx tsc --noEmit"
```
