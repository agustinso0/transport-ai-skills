---
name: performance-transport-system
description: Performance optimization strategies for transportation platforms focusing on trip queries, dispatch latency, and caching frequent lookups
---

# Performance Transport System

# Description

Comprehensive performance optimization strategies for transportation platforms. Focuses on reducing dispatch latency, optimizing database queries, implementing intelligent caching, and achieving sub-second response times for critical operations like driver matching and trip creation.

# When to Activate

- Experiencing slow trip creation or dispatch
- High database load during peak hours
- Need to scale to thousands of concurrent requests
- Optimizing driver matching algorithms
- Reducing API response times
- Implementing real-time location tracking at scale
- Building analytics dashboards with minimal impact

# Database Design

## Read Replicas Strategy
```javascript
const dbConfig = {
  primary: {
    host: process.env.DB_PRIMARY_HOST,
    max: 20,
    idleTimeoutMillis: 30000
  },
  replicas: [
    { host: process.env.DB_REPLICA_1_HOST, max: 50 },
    { host: process.env.DB_REPLICA_2_HOST, max: 50 }
  ]
};

function getConnection(readOnly = false) {
  if (readOnly) {
    const replica = dbConfig.replicas[
      Math.floor(Math.random() * dbConfig.replicas.length)
    ];
    return getPool(replica);
  }
  return getPool(dbConfig.primary);
}
```

## Connection Pooling
```javascript
import { Pool } from 'pg';

const pool = new Pool({
  max: 50,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 5000,
  statement_timeout: 10000,
  query_timeout: 10000
});

pool.on('error', (err) => {
  console.error('Unexpected database error:', err);
});
```

# Indexing Strategies

## Optimized Indexes for Trip Queries
```sql
CREATE INDEX CONCURRENTLY idx_trips_status_created
  ON trips (status, created_at DESC)
  WHERE status IN ('requested', 'searching', 'accepted');

CREATE INDEX CONCURRENTLY idx_trips_driver_active
  ON trips (driver_id, status)
  WHERE status NOT IN ('completed', 'cancelled');

CREATE INDEX CONCURRENTLY idx_drivers_online_location
  ON drivers (status)
  INCLUDE (current_location, id)
  WHERE status = 'online';
```

## Partial Indexes for Active Records
```sql
CREATE INDEX CONCURRENTLY idx_active_trips
  ON trips (id, driver_id, passenger_id, status, created_at)
  WHERE status IN ('in_progress', 'arriving');

CREATE INDEX CONCURRENTLY idx_recent_trip_events
  ON trip_events (trip_id, event_type, created_at DESC)
  WHERE created_at > NOW() - INTERVAL '7 days';
```

# Query Optimization

## Optimized Driver Matching
```javascript
async function findOptimalDriver(pickupLat, pickupLng, maxDistanceKm = 5) {
  const nearbyDrivers = await redis.georadius(
    'drivers:locations',
    pickupLng,
    pickupLat,
    maxDistanceKm,
    'km',
    'WITHDIST',
    'ASC',
    'COUNT',
    20
  );

  if (nearbyDrivers.length === 0) return null;

  const driverIds = nearbyDrivers.map(([id]) => id);

  const query = `
    SELECT
      d.id,
      d.first_name,
      d.rating,
      d.total_trips,
      ST_Distance(
        d.current_location,
        ST_MakePoint($1, $2)::geography
      ) as distance_meters
    FROM drivers d
    WHERE
      d.id = ANY($3)
      AND d.status = 'online'
      AND NOT EXISTS (
        SELECT 1 FROM trips t
        WHERE t.driver_id = d.id
        AND t.status IN ('accepted', 'in_progress', 'arriving')
      )
    ORDER BY
      d.rating DESC,
      distance_meters ASC
    LIMIT 1
  `;

  const result = await db.query(query, [pickupLng, pickupLat, driverIds]);
  return result.rows[0];
}
```

## Batch Trip Status Updates
```javascript
async function batchUpdateTripStatuses(updates) {
  const values = updates.map((u, i) =>
    `($${i * 3 + 1}, $${i * 3 + 2}, $${i * 3 + 3})`
  ).join(',');

  const params = updates.flatMap(u => [u.tripId, u.status, u.timestamp]);

  const query = `
    UPDATE trips t SET
      status = v.status,
      updated_at = v.timestamp
    FROM (VALUES ${values}) AS v(id, status, timestamp)
    WHERE t.id = v.id::uuid
    RETURNING t.id, t.status
  `;

  return await db.query(query, params);
}
```

## Materialized View for Analytics
```sql
CREATE MATERIALIZED VIEW hourly_trip_stats AS
SELECT
  DATE_TRUNC('hour', created_at) as hour,
  status,
  COUNT(*) as trip_count,
  AVG(fare_amount) as avg_fare,
  AVG(duration_seconds) as avg_duration,
  PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY duration_seconds) as median_duration,
  PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY duration_seconds) as p95_duration
FROM trips
WHERE created_at >= NOW() - INTERVAL '30 days'
GROUP BY hour, status;

CREATE UNIQUE INDEX ON hourly_trip_stats (hour, status);

REFRESH MATERIALIZED VIEW CONCURRENTLY hourly_trip_stats;
```

# Anti-Patterns

## ❌ Avoid
- N+1 queries for related data
- Missing database connection pooling
- Synchronous API calls in critical path
- Full table scans on large tables
- Not using prepared statements
- Blocking operations on event loop
- Missing indexes on foreign keys
- Storing computed values without caching
- Sequential processing of independent operations
- Using `SELECT *` instead of specific columns

## ✅ Do
- Batch database operations when possible
- Implement connection pooling with limits
- Use async operations for external calls
- Add indexes for all WHERE/JOIN/ORDER BY columns
- Use prepared statements for repeated queries
- Leverage Promise.all for parallel operations
- Create indexes on all foreign keys
- Cache expensive computations in Redis
- Process independent tasks concurrently
- Select only required columns

# Performance Guidelines

## Multi-Layer Caching Strategy
```javascript
class CacheManager {
  constructor() {
    this.memoryCache = new Map();
    this.redis = redis;
    this.db = db;
  }

  async get(key, fetchFn, { ttl = 300, skipMemory = false } = {}) {
    if (!skipMemory && this.memoryCache.has(key)) {
      return this.memoryCache.get(key);
    }

    const cached = await this.redis.get(key);
    if (cached) {
      const value = JSON.parse(cached);
      if (!skipMemory) {
        this.memoryCache.set(key, value);
        setTimeout(() => this.memoryCache.delete(key), 10000);
      }
      return value;
    }

    const value = await fetchFn();

    await this.redis.set(key, JSON.stringify(value), 'EX', ttl);
    if (!skipMemory) {
      this.memoryCache.set(key, value);
      setTimeout(() => this.memoryCache.delete(key), 10000);
    }

    return value;
  }

  async invalidate(key) {
    this.memoryCache.delete(key);
    await this.redis.del(key);
  }
}
```

## Request Batching
```javascript
class RequestBatcher {
  constructor(batchFn, { maxBatchSize = 100, maxWaitMs = 50 } = {}) {
    this.batchFn = batchFn;
    this.maxBatchSize = maxBatchSize;
    this.maxWaitMs = maxWaitMs;
    this.queue = [];
    this.timer = null;
  }

  async add(item) {
    return new Promise((resolve, reject) => {
      this.queue.push({ item, resolve, reject });

      if (this.queue.length >= this.maxBatchSize) {
        this.flush();
      } else if (!this.timer) {
        this.timer = setTimeout(() => this.flush(), this.maxWaitMs);
      }
    });
  }

  async flush() {
    if (this.timer) {
      clearTimeout(this.timer);
      this.timer = null;
    }

    if (this.queue.length === 0) return;

    const batch = this.queue.splice(0, this.maxBatchSize);
    const items = batch.map(b => b.item);

    try {
      const results = await this.batchFn(items);
      batch.forEach((b, i) => b.resolve(results[i]));
    } catch (error) {
      batch.forEach(b => b.reject(error));
    }
  }
}

const driverBatcher = new RequestBatcher(async (driverIds) => {
  const query = 'SELECT * FROM drivers WHERE id = ANY($1)';
  const result = await db.query(query, [driverIds]);
  return driverIds.map(id => result.rows.find(r => r.id === id));
});
```

## Optimized Trip Creation
```javascript
async function createTrip(tripData) {
  const startTime = Date.now();

  const [driver, route] = await Promise.all([
    findOptimalDriver(tripData.pickupLat, tripData.pickupLng),
    calculateRoute(tripData.pickup, tripData.dropoff)
  ]);

  if (!driver) {
    throw new Error('No drivers available');
  }

  const trip = await db.transaction(async (client) => {
    const insertQuery = `
      INSERT INTO trips (
        driver_id, passenger_id, pickup_location, dropoff_location,
        pickup_address, dropoff_address, status, route_id, fare_amount
      ) VALUES ($1, $2, ST_MakePoint($3, $4), ST_MakePoint($5, $6), $7, $8, $9, $10, $11)
      RETURNING *
    `;

    const tripResult = await client.query(insertQuery, [
      driver.id, tripData.passengerId,
      tripData.pickupLng, tripData.pickupLat,
      tripData.dropoffLng, tripData.dropoffLat,
      tripData.pickupAddress, tripData.dropoffAddress,
      'accepted', route.id, route.estimatedFare
    ]);

    await client.query(
      'UPDATE drivers SET status = $1 WHERE id = $2',
      ['on_trip', driver.id]
    );

    return tripResult.rows[0];
  });

  await Promise.all([
    redis.hset(`trip:active:${trip.id}`, trip),
    redis.expire(`trip:active:${trip.id}`, 3600),
    redis.publish('trip:created', JSON.stringify(trip))
  ]);

  console.log(`Trip created in ${Date.now() - startTime}ms`);

  return trip;
}
```

## Database Query Monitoring
```javascript
async function monitorSlowQueries() {
  const query = `
    SELECT
      query,
      calls,
      total_exec_time,
      mean_exec_time,
      max_exec_time,
      stddev_exec_time
    FROM pg_stat_statements
    WHERE mean_exec_time > 100
    ORDER BY mean_exec_time DESC
    LIMIT 20
  `;

  const result = await db.query(query);
  return result.rows;
}
```

# Observability

## Performance Metrics Collection
```javascript
class MetricsCollector {
  constructor() {
    this.metrics = new Map();
  }

  recordLatency(operation, durationMs) {
    if (!this.metrics.has(operation)) {
      this.metrics.set(operation, []);
    }
    this.metrics.get(operation).push(durationMs);
  }

  getStats(operation) {
    const values = this.metrics.get(operation) || [];
    if (values.length === 0) return null;

    values.sort((a, b) => a - b);

    return {
      count: values.length,
      mean: values.reduce((a, b) => a + b, 0) / values.length,
      median: values[Math.floor(values.length / 2)],
      p95: values[Math.floor(values.length * 0.95)],
      p99: values[Math.floor(values.length * 0.99)],
      max: values[values.length - 1]
    };
  }
}

const metrics = new MetricsCollector();

async function timedOperation(name, fn) {
  const start = Date.now();
  try {
    return await fn();
  } finally {
    metrics.recordLatency(name, Date.now() - start);
  }
}
```

## Real-Time Performance Dashboard
```javascript
async function getPerformanceMetrics() {
  const [dbStats, redisStats, cacheHitRate] = await Promise.all([
    getDbConnectionStats(),
    getRedisStats(),
    getCacheHitRate()
  ]);

  return {
    database: dbStats,
    cache: redisStats,
    cacheHitRate,
    operations: {
      tripCreation: metrics.getStats('trip_creation'),
      driverMatch: metrics.getStats('driver_match'),
      routeCalculation: metrics.getStats('route_calculation')
    }
  };
}
```

# Testing Strategy

## Load Testing
```javascript
import { check } from 'k6';
import http from 'k6/http';

export const options = {
  stages: [
    { duration: '2m', target: 100 },
    { duration: '5m', target: 1000 },
    { duration: '2m', target: 0 }
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],
    http_req_failed: ['rate<0.01']
  }
};

export default function() {
  const payload = JSON.stringify({
    passengerId: 'test-passenger',
    pickupLat: 37.7749,
    pickupLng: -122.4194,
    dropoffLat: 37.8044,
    dropoffLng: -122.2712
  });

  const res = http.post('http://api/trips', payload, {
    headers: { 'Content-Type': 'application/json' }
  });

  check(res, {
    'status is 201': (r) => r.status === 201,
    'response time < 500ms': (r) => r.timings.duration < 500
  });
}
```

## Performance Benchmarks
- Trip creation: < 200ms (p95)
- Driver matching: < 100ms (p95)
- Location updates: < 50ms (p95)
- Database queries: < 100ms (p95)
- Cache hit rate: > 80%
- API throughput: > 1000 req/s
- Database connections: < 70% pool usage

## Stress Testing
- Test with 10x expected peak load
- Verify graceful degradation under load
- Test database failover scenarios
- Validate cache failure handling
- Ensure no memory leaks during extended runs
- Test connection pool exhaustion recovery

## Profiling
```javascript
import { performance } from 'perf_hooks';

function profile(fn) {
  return async function(...args) {
    const start = performance.now();
    const memStart = process.memoryUsage();

    try {
      return await fn.apply(this, args);
    } finally {
      const duration = performance.now() - start;
      const memEnd = process.memoryUsage();

      console.log({
        function: fn.name,
        duration: `${duration.toFixed(2)}ms`,
        heapUsed: `${((memEnd.heapUsed - memStart.heapUsed) / 1024 / 1024).toFixed(2)}MB`
      });
    }
  };
}
```
