# Backend Deployment & CI/CD

## Deployment Architecture

```mermaid
graph TB
    DEV[Developer] --> PUSH[Git Push]
    PUSH --> CI[CI Pipeline]

    subgraph "CI Pipeline"
        CI --> LINT[Lint & Format]
        LINT --> TEST[Unit Tests]
        TEST --> BUILD[Build]
        BUILD --> SECURITY[Security Scan]
        SECURITY --> IMAGE[Container Image]
    end

    IMAGE --> CD[CD Pipeline]

    subgraph "CD Pipeline"
        CD --> STAGING[Deploy to Staging]
        STAGING --> E2E[E2E Tests]
        E2E --> APPROVE{Manual<br/>Approval}
        APPROVE -->|Approved| PROD[Deploy to Production]
        APPROVE -->|Rejected| NOTIFY[Notify Team]
    end

    subgraph "Production"
        PROD --> LB[Load Balancer]
        LB --> APP1[App Server 1]
        LB --> APP2[App Server 2]
        LB --> APP3[App Server 3]
    end

    style CI fill:#e3f2fd
    style CD fill:#e8f5e9
```

## Dockerfile

```dockerfile
# Multi-stage build for Node.js
FROM node:20-alpine AS base
RUN apk add --no-cache libc6-compat
WORKDIR /app

# Dependencies
FROM base AS deps
COPY package.json package-lock.json ./
RUN npm ci --only=production && \
    cp -R node_modules /prod_modules && \
    npm ci

# Build
FROM base AS builder
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

# Production
FROM base AS runner
WORKDIR /app

ENV NODE_ENV production

# Create non-root user
RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 appuser

# Copy built assets
COPY --from=builder /app/dist ./dist
COPY --from=deps /prod_modules ./node_modules
COPY package.json ./

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

USER appuser

EXPOSE 3000

CMD ["node", "dist/server.js"]
```

### Docker Compose for Production

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
        reservations:
          cpus: '0.5'
          memory: 256M
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
    environment:
      - NODE_ENV=production
      - DATABASE_URL=${DATABASE_URL}
      - REDIS_URL=${REDIS_URL}
      - JWT_SECRET=${JWT_SECRET}
    networks:
      - backend
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy

  db:
    image: postgres:15-alpine
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    networks:
      - backend
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./certs:/etc/nginx/certs
    networks:
      - backend

networks:
  backend:

volumes:
  postgres_data:
  redis_data:
```

## CI/CD Pipeline

### GitHub Actions

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  NODE_VERSION: '20'
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck

  test:
    runs-on: ubuntu-latest
    needs: lint
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test
        ports: ['5432:5432']
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:7
        ports: ['6379:6379']
    env:
      DATABASE_URL: postgresql://test:test@localhost:5432/test
      REDIS_URL: redis://localhost:6379
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      - run: npm ci
      - run: npm run test:coverage
      - run: npm run test:integration

  build:
    runs-on: ubuntu-latest
    needs: test
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/metadata-action@v5
        id: meta
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha
            type=ref,event=branch
      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}

  security:
    runs-on: ubuntu-latest
    needs: build
    steps:
      - uses: actions/checkout@v4
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }}
          format: 'sarif'
          output: 'trivy-results.sarif'
      - name: Upload to GitHub Security
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'
```

### CD Pipeline

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production

jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: staging
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to staging
        run: |
          # Deploy to staging environment
          kubectl set image deployment/app \
            app=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }} \
            --namespace=staging
        env:
          KUBECONFIG: ${{ secrets.KUBE_CONFIG_STAGING }}

      - name: Run smoke tests
        run: |
          curl -f https://staging.api.[PROJECT_NAME].com/health
          # Add more smoke tests

      - name: Run E2E tests
        run: |
          npm run test:e2e -- --base-url=https://staging.api.[PROJECT_NAME].com

  deploy-production:
    runs-on: ubuntu-latest
    needs: deploy-staging
    environment:
      name: production
      url: https://api.[PROJECT_NAME].com
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to production (blue-green)
        run: |
          # Deploy to production using blue-green strategy
          kubectl set image deployment/app-blue \
            app=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }} \
            --namespace=production
          # Wait for rollout
          kubectl rollout status deployment/app-blue --namespace=production --timeout=300s
          # Switch traffic
          kubectl patch service app-svc -p '{"spec":{"selector":{"version":"blue"}}}' \
            --namespace=production

      - name: Verify deployment
        run: |
          # Health check
          for i in {1..30}; do
            if curl -sf https://api.[PROJECT_NAME].com/health; then
              echo "Deployment healthy!"
              exit 0
            fi
            sleep 10
          done
          echo "Deployment health check failed!"
          # Rollback
          kubectl rollout undo deployment/app-blue --namespace=production
          exit 1

      - name: Notify team
        if: always()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "Production deployment ${{ job.status }}: ${{ github.sha }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

## Kubernetes Deployment

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: app
          image: ghcr.io/[org]/[project]:latest
          ports:
            - containerPort: 3000
          env:
            - name: NODE_ENV
              value: production
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: database-url
          resources:
            requests:
              cpu: "500m"
              memory: "256Mi"
            limits:
              cpu: "1000m"
              memory: "512Mi"
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule

---
apiVersion: v1
kind: Service
metadata:
  name: app-svc
  namespace: production
spec:
  selector:
    app: api
  ports:
    - port: 80
      targetPort: 3000
  type: ClusterIP

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.[PROJECT_NAME].com
      secretName: api-tls
  rules:
    - host: api.[PROJECT_NAME].com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app-svc
                port:
                  number: 80
```

## Environment Management

```yaml
environments:
  development:
    replicas: 1
    resources:
      cpu: "250m"
      memory: "256Mi"
    database:
      host: localhost
      pool_size: 5

  staging:
    replicas: 2
    resources:
      cpu: "500m"
      memory: "512Mi"
    database:
      host: staging-db.internal
      pool_size: 10

  production:
    replicas: 3
    resources:
      cpu: "1000m"
      memory: "1Gi"
    database:
      host: prod-db.internal
      pool_size: 20
    autoscaling:
      enabled: true
      min_replicas: 3
      max_replicas: 10
      target_cpu: 70
```

## Monitoring & Alerting

```yaml
# prometheus/alerts.yml
groups:
  - name: api-alerts
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }}"

      - alert: HighLatency
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High latency detected"
          description: "95th percentile latency is {{ $value }}s"

      - alert: DatabaseConnectionPoolExhausted
        expr: pg_stat_activity_count > pg_settings_max_connections * 0.9
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Database connection pool nearly exhausted"

      - alert: MemoryHigh
        expr: process_resident_memory_bytes / 1024 / 1024 > 512
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage"
```

## Rollback Procedures

```bash
# Quick rollback - revert last deployment
kubectl rollout undo deployment/app --namespace=production

# Rollback to specific revision
kubectl rollout undo deployment/app --to-revision=3 --namespace=production

# Check rollout history
kubectl rollout history deployment/app --namespace=production

# Manual rollback via git
git revert HEAD
git push origin main
# CI/CD will trigger new deployment
```

## Deployment Checklist

```yaml
pre_deployment:
  - [ ] All tests passing in CI
  - [ ] Security scan completed
  - [ ] Database migrations tested
  - [ ] Environment variables configured
  - [ ] Secrets updated in vault
  - [ ] Backup created

deployment:
  - [ ] Deploy to staging first
  - [ ] Run smoke tests on staging
  - [ ] Verify staging health checks
  - [ ] Get deployment approval
  - [ ] Deploy to production (canary/blue-green)
  - [ ] Monitor error rates
  - [ ] Verify health endpoints

post_deployment:
  - [ ] Run production smoke tests
  - [ ] Check application logs
  - [ ] Verify metrics in Grafana
  - [ ] Update deployment documentation
  - [ ] Notify team of completion
  - [ ] Monitor for 30 minutes

rollback_criteria:
  - Error rate > 5% for 5 minutes
  - P95 latency > 2 seconds for 5 minutes
  - Health check failures
  - Critical bug reported
```
