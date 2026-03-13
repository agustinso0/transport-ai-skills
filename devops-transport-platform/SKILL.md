---
name: devops-transport-platform
description: DevOps practices for dockerized transport services with CI/CD and monitoring
---

# DevOps Transport Platform

# Description

This skill provides comprehensive DevOps practices for managing a transportation platform, including containerization, CI/CD pipelines, environment configuration, monitoring, and deployment strategies. It ensures reliable, scalable infrastructure for transport services.

# When to Activate

- Setting up Docker containers for transport services
- Configuring CI/CD pipelines for automated deployments
- Managing environment variables and secrets
- Implementing monitoring and alerting
- Setting up logging and observability
- Configuring auto-scaling for peak demand
- Managing database migrations in production

# Engineering Principles

1. **Infrastructure as Code**: All infrastructure defined in version control
2. **Immutable Deployments**: Never modify running containers, always redeploy
3. **Zero Downtime**: Rolling deployments with health checks
4. **Fail Fast**: Quick detection and rollback of problematic deployments
5. **Observability First**: Comprehensive logging, metrics, and tracing
6. **Security by Default**: Secrets management, least privilege access

# Implementation Patterns

## Docker Structure

```
transport-platform/
  services/
    trip-service/
      Dockerfile
      src/
    location-service/
      Dockerfile
      src/
    notification-service/
      Dockerfile
      src/
  docker-compose.yml
  docker-compose.prod.yml
  .env.example
```

## Dockerfile Best Practices

```dockerfile
FROM node:20-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .
RUN npm run build

FROM node:20-alpine AS runner

RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

WORKDIR /app

COPY --from=builder --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nodejs:nodejs /app/dist ./dist
COPY --from=builder --chown=nodejs:nodejs /app/package.json ./

USER nodejs

EXPOSE 3000

ENV NODE_ENV=production

HEALTHCHECK --interval=30s --timeout=3s --start-period=40s \
  CMD node healthcheck.js

CMD ["node", "dist/index.js"]
```

## Docker Compose for Development

```yaml
version: '3.8'

services:
  trip-service:
    build:
      context: ./services/trip-service
      dockerfile: Dockerfile
    ports:
      - "3001:3000"
    environment:
      - NODE_ENV=development
      - DATABASE_URL=${DATABASE_URL}
      - SUPABASE_URL=${SUPABASE_URL}
      - SUPABASE_ANON_KEY=${SUPABASE_ANON_KEY}
    volumes:
      - ./services/trip-service/src:/app/src
    depends_on:
      - redis
    networks:
      - transport-network

  location-service:
    build:
      context: ./services/location-service
      dockerfile: Dockerfile
    ports:
      - "3002:3000"
    environment:
      - NODE_ENV=development
      - DATABASE_URL=${DATABASE_URL}
      - REDIS_URL=redis://redis:6379
    volumes:
      - ./services/location-service/src:/app/src
    depends_on:
      - redis
    networks:
      - transport-network

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    networks:
      - transport-network

volumes:
  redis-data:

networks:
  transport-network:
    driver: bridge
```

# Code Examples

## GitHub Actions CI/CD Pipeline

```yaml
name: Deploy Transport Services

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run linter
        run: npm run lint

      - name: Run type check
        run: npm run typecheck

      - name: Run tests
        run: npm test
        env:
          DATABASE_URL: ${{ secrets.TEST_DATABASE_URL }}

  build-and-push:
    needs: test
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'

    permissions:
      contents: read
      packages: write

    strategy:
      matrix:
        service: [trip-service, location-service, notification-service]

    steps:
      - uses: actions/checkout@v4

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/${{ matrix.service }}
          tags: |
            type=sha,prefix={{branch}}-
            type=ref,event=branch
            type=semver,pattern={{version}}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: ./services/${{ matrix.service }}
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest

    steps:
      - name: Deploy to production
        run: |
          echo "Deploying to production environment"
```

## Health Check Endpoint

```typescript
import express from 'express';
import { supabase } from './lib/supabase';

const app = express();

app.get('/health', async (req, res) => {
  const checks = {
    status: 'healthy',
    timestamp: new Date().toISOString(),
    checks: {
      database: 'unknown',
      redis: 'unknown',
      memory: 'unknown'
    }
  };

  try {
    const { error } = await supabase.from('trips').select('id').limit(1);
    checks.checks.database = error ? 'unhealthy' : 'healthy';
  } catch (err) {
    checks.checks.database = 'unhealthy';
    checks.status = 'unhealthy';
  }

  const memUsage = process.memoryUsage();
  const memThreshold = 500 * 1024 * 1024;
  checks.checks.memory = memUsage.heapUsed < memThreshold ? 'healthy' : 'unhealthy';

  if (checks.checks.memory === 'unhealthy') {
    checks.status = 'unhealthy';
  }

  const statusCode = checks.status === 'healthy' ? 200 : 503;
  res.status(statusCode).json(checks);
});

app.get('/ready', async (req, res) => {
  try {
    await supabase.from('trips').select('id').limit(1);
    res.status(200).json({ ready: true });
  } catch (err) {
    res.status(503).json({ ready: false });
  }
});

export default app;
```

## Environment Configuration

```bash
.env.production
```

```bash
NODE_ENV=production

SUPABASE_URL=https://your-project.supabase.co
SUPABASE_ANON_KEY=your-anon-key
SUPABASE_SERVICE_ROLE_KEY=your-service-role-key

REDIS_URL=redis://redis:6379

LOG_LEVEL=info

RATE_LIMIT_WINDOW_MS=900000
RATE_LIMIT_MAX_REQUESTS=100

CORS_ORIGIN=https://yourdomain.com

SENTRY_DSN=https://your-sentry-dsn
```

## Secrets Management Script

```typescript
import { execSync } from 'child_process';

const secrets = [
  'SUPABASE_URL',
  'SUPABASE_ANON_KEY',
  'SUPABASE_SERVICE_ROLE_KEY',
  'REDIS_URL',
  'SENTRY_DSN'
];

function validateSecrets() {
  const missing = secrets.filter(key => !process.env[key]);

  if (missing.length > 0) {
    console.error('Missing required secrets:');
    missing.forEach(key => console.error(`  - ${key}`));
    process.exit(1);
  }

  console.log('All required secrets are present');
}

validateSecrets();
```

# Anti-Patterns

❌ **Hardcoded secrets in Dockerfile**
```dockerfile
ENV DATABASE_URL=postgresql://user:password@host/db
```

✅ **Use environment variables**
```dockerfile
ENV DATABASE_URL=${DATABASE_URL}
```

❌ **Running as root user**
```dockerfile
CMD ["node", "index.js"]
```

✅ **Use non-root user**
```dockerfile
USER nodejs
CMD ["node", "index.js"]
```

❌ **No health checks**
```yaml
services:
  app:
    image: myapp
```

✅ **Include health checks**
```yaml
services:
  app:
    image: myapp
    healthcheck:
      test: ["CMD", "node", "healthcheck.js"]
      interval: 30s
      timeout: 3s
```

❌ **Single-stage builds with dev dependencies**
```dockerfile
COPY . .
RUN npm install
CMD ["npm", "start"]
```

✅ **Multi-stage builds**
```dockerfile
FROM node:20-alpine AS builder
RUN npm ci --only=production
FROM node:20-alpine AS runner
COPY --from=builder /app/node_modules ./node_modules
```

# Performance Guidelines

1. **Layer Caching**: Order Dockerfile commands from least to most frequently changed
2. **Multi-stage Builds**: Reduce final image size by 60-80%
3. **Resource Limits**: Set memory and CPU limits for containers
4. **Connection Pooling**: Reuse database connections across requests
5. **Horizontal Scaling**: Add more service instances during peak times
6. **CDN Integration**: Serve static assets from CDN

```yaml
services:
  trip-service:
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 512M
        reservations:
          cpus: '0.5'
          memory: 256M
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
        order: start-first
```

# Observability

## Structured Logging

```typescript
import winston from 'winston';

const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  defaultMeta: {
    service: 'trip-service',
    environment: process.env.NODE_ENV
  },
  transports: [
    new winston.transports.Console({
      format: winston.format.simple()
    })
  ]
});

export function logTripEvent(event: string, tripId: string, metadata?: any) {
  logger.info('Trip event', {
    event,
    tripId,
    ...metadata
  });
}
```

## Metrics Collection

```typescript
import { Counter, Histogram, register } from 'prom-client';

const tripCounter = new Counter({
  name: 'trips_total',
  help: 'Total number of trips',
  labelNames: ['status']
});

const tripDuration = new Histogram({
  name: 'trip_duration_seconds',
  help: 'Trip duration in seconds',
  buckets: [300, 600, 900, 1800, 3600]
});

export function recordTripMetrics(trip: Trip) {
  tripCounter.inc({ status: trip.status });

  if (trip.status === 'completed') {
    const duration = (trip.completed_at - trip.started_at) / 1000;
    tripDuration.observe(duration);
  }
}

app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});
```

## Distributed Tracing

```typescript
import { trace, context } from '@opentelemetry/api';

const tracer = trace.getTracer('trip-service');

export async function createTrip(tripData: TripData) {
  const span = tracer.startSpan('createTrip');

  try {
    span.setAttribute('trip.pickup', tripData.pickup);
    span.setAttribute('trip.destination', tripData.destination);

    const trip = await supabase.from('trips').insert(tripData);

    span.setStatus({ code: 0 });
    return trip;
  } catch (error) {
    span.setStatus({ code: 2, message: error.message });
    throw error;
  } finally {
    span.end();
  }
}
```

# Testing Strategy

## Integration Tests for Services

```typescript
import { describe, test, expect, beforeAll, afterAll } from 'vitest';
import { createTestContainer, stopTestContainer } from './test-utils';

describe('Trip Service Integration', () => {
  let container;

  beforeAll(async () => {
    container = await createTestContainer('trip-service');
  });

  afterAll(async () => {
    await stopTestContainer(container);
  });

  test('health endpoint returns 200', async () => {
    const response = await fetch('http://localhost:3001/health');
    expect(response.status).toBe(200);
  });

  test('creates trip successfully', async () => {
    const response = await fetch('http://localhost:3001/trips', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        pickup: '123 Main St',
        destination: '456 Oak Ave'
      })
    });

    expect(response.status).toBe(201);
    const trip = await response.json();
    expect(trip.id).toBeDefined();
  });
});
```

## Load Testing

```typescript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '2m', target: 100 },
    { duration: '5m', target: 100 },
    { duration: '2m', target: 200 },
    { duration: '5m', target: 200 },
    { duration: '2m', target: 0 }
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],
    http_req_failed: ['rate<0.01']
  }
};

export default function() {
  const response = http.post('http://localhost:3001/trips', JSON.stringify({
    pickup: '123 Main St',
    destination: '456 Oak Ave'
  }), {
    headers: { 'Content-Type': 'application/json' }
  });

  check(response, {
    'status is 201': (r) => r.status === 201,
    'response time < 500ms': (r) => r.timings.duration < 500
  });

  sleep(1);
}
```

## Monitoring Alerts Configuration

```yaml
alerts:
  - name: HighErrorRate
    condition: error_rate > 0.05
    duration: 5m
    severity: critical
    message: "Error rate above 5% for 5 minutes"

  - name: HighResponseTime
    condition: p95_response_time > 1000
    duration: 5m
    severity: warning
    message: "95th percentile response time above 1s"

  - name: LowDatabaseConnections
    condition: db_connections < 5
    duration: 2m
    severity: warning
    message: "Database connection pool below threshold"

  - name: HighMemoryUsage
    condition: memory_usage > 0.85
    duration: 5m
    severity: critical
    message: "Memory usage above 85%"
```
