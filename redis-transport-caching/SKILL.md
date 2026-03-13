---
name: redis-transport-caching
description: Redis caching strategies for transportation platforms including vehicle locations, rate limiting, distributed locks, and queue processing
---

# Redis Transport Caching

# Description

High-performance caching layer using Redis for transportation platforms. Handles real-time vehicle location updates, driver availability, rate limiting for API calls, distributed locks for dispatch coordination, and queue processing for asynchronous tasks.

# When to Activate

- Caching frequently accessed driver/vehicle data
- Implementing real-time location tracking
- Rate limiting API endpoints
- Coordinating distributed driver dispatch
- Processing background jobs (notifications, analytics)
- Managing session data for drivers and passengers
- Implementing pub/sub for real-time updates

# Database Design

## Redis Data Structures

### Vehicle Location Cache
```javascript
// Key pattern: vehicle:location:{vehicle_id}
// Type: Hash
// TTL: 60 seconds
{
  "lat": "37.7749",
  "lng": "-122.4194",
  "heading": "90",
  "speed": "45.5",
  "updated_at": "1710345600"
}
```

### Driver Availability Set
```javascript
// Key: drivers:online
// Type: Sorted Set (score = last_update_timestamp)
// Members: driver_id
ZADD drivers:online 1710345600 "driver-uuid-123"
```

### Active Trip Cache
```javascript
// Key pattern: trip:active:{trip_id}
// Type: Hash
// TTL: 3600 seconds
{
  "driver_id": "driver-uuid-123",
  "passenger_id": "passenger-uuid-456",
  "status": "in_progress",
  "pickup_lat": "37.7749",
  "pickup_lng": "-122.4194",
  "dropoff_lat": "37.8044",
  "dropoff_lng": "-122.2712"
}
```

### Geospatial Index
```javascript
// Key: drivers:locations
// Type: Geo
GEOADD drivers:locations -122.4194 37.7749 "driver-uuid-123"
```

### Rate Limiting
```javascript
// Key pattern: ratelimit:{endpoint}:{user_id}
// Type: String (counter)
// TTL: 60 seconds
SET ratelimit:api/trips:user-123 1 EX 60
```

### Distributed Lock
```javascript
// Key pattern: lock:trip:dispatch:{trip_id}
// Type: String
// TTL: 30 seconds
SET lock:trip:dispatch:trip-123 "worker-id-456" NX EX 30
```

# Indexing Strategies

## Geospatial Indexing
```javascript
async function updateDriverLocation(driverId, lat, lng) {
  const multi = redis.multi();

  multi.geoadd('drivers:locations', lng, lat, driverId);

  multi.hset(`vehicle:location:${driverId}`, {
    lat,
    lng,
    heading: 0,
    speed: 0,
    updated_at: Date.now()
  });

  multi.expire(`vehicle:location:${driverId}`, 60);

  multi.zadd('drivers:online', Date.now(), driverId);

  await multi.exec();
}
```

## Finding Nearby Drivers
```javascript
async function findNearbyDrivers(lat, lng, radiusKm = 5, limit = 10) {
  const drivers = await redis.georadius(
    'drivers:locations',
    lng,
    lat,
    radiusKm,
    'km',
    'WITHDIST',
    'ASC',
    'COUNT',
    limit
  );

  const driverData = await Promise.all(
    drivers.map(async ([driverId, distance]) => {
      const location = await redis.hgetall(`vehicle:location:${driverId}`);
      const isOnline = await redis.zscore('drivers:online', driverId);

      return {
        driverId,
        distance: parseFloat(distance),
        location,
        isOnline: !!isOnline
      };
    })
  );

  return driverData.filter(d => d.isOnline);
}
```

# Query Optimization

## Pipelining for Bulk Operations
```javascript
async function updateMultipleLocations(locations) {
  const pipeline = redis.pipeline();

  for (const { driverId, lat, lng } of locations) {
    pipeline.geoadd('drivers:locations', lng, lat, driverId);
    pipeline.hset(`vehicle:location:${driverId}`, {
      lat,
      lng,
      updated_at: Date.now()
    });
    pipeline.expire(`vehicle:location:${driverId}`, 60);
  }

  await pipeline.exec();
}
```

## Caching Trip Details
```javascript
async function getTripDetails(tripId) {
  const cacheKey = `trip:active:${tripId}`;

  let trip = await redis.hgetall(cacheKey);

  if (!trip || Object.keys(trip).length === 0) {
    trip = await db.query('SELECT * FROM trips WHERE id = $1', [tripId]);

    if (trip) {
      await redis.hset(cacheKey, trip);
      await redis.expire(cacheKey, 3600);
    }
  }

  return trip;
}
```

## Cache Invalidation
```javascript
async function updateTripStatus(tripId, status) {
  await db.query('UPDATE trips SET status = $1 WHERE id = $2', [status, tripId]);

  await redis.del(`trip:active:${tripId}`);

  await redis.publish('trip:updates', JSON.stringify({
    tripId,
    status,
    timestamp: Date.now()
  }));
}
```

# Anti-Patterns

## ❌ Avoid
- Storing large objects without compression
- Not setting TTLs on temporary data
- Using Redis as primary database
- Blocking operations on main thread
- Not handling connection failures gracefully
- Storing sensitive data unencrypted
- Using KEYS command in production
- Not using pipelining for bulk operations
- Missing error handling on cache misses

## ✅ Do
- Set appropriate TTLs for all cached data
- Use Redis for ephemeral/derived data only
- Implement connection pooling
- Use async/await for non-blocking operations
- Encrypt sensitive cached data
- Use SCAN instead of KEYS
- Batch operations with pipelining
- Implement cache-aside pattern with fallback to DB
- Monitor memory usage and eviction policies

# Performance Guidelines

## Rate Limiting Implementation
```javascript
async function checkRateLimit(userId, endpoint, limit = 100, window = 60) {
  const key = `ratelimit:${endpoint}:${userId}`;

  const current = await redis.incr(key);

  if (current === 1) {
    await redis.expire(key, window);
  }

  if (current > limit) {
    throw new Error('Rate limit exceeded');
  }

  return {
    allowed: true,
    remaining: limit - current,
    reset: await redis.ttl(key)
  };
}
```

## Distributed Lock for Dispatch
```javascript
async function acquireDispatchLock(tripId, workerId, ttl = 30) {
  const lockKey = `lock:trip:dispatch:${tripId}`;

  const acquired = await redis.set(
    lockKey,
    workerId,
    'NX',
    'EX',
    ttl
  );

  return acquired === 'OK';
}

async function releaseDispatchLock(tripId, workerId) {
  const lockKey = `lock:trip:dispatch:${tripId}`;

  const script = `
    if redis.call("get", KEYS[1]) == ARGV[1] then
      return redis.call("del", KEYS[1])
    else
      return 0
    end
  `;

  return await redis.eval(script, 1, lockKey, workerId);
}
```

## Queue Processing
```javascript
async function enqueueNotification(tripId, type, data) {
  const payload = JSON.stringify({
    tripId,
    type,
    data,
    timestamp: Date.now()
  });

  await redis.lpush('queue:notifications', payload);
}

async function processNotificationQueue() {
  while (true) {
    const item = await redis.brpop('queue:notifications', 5);

    if (item) {
      const [, payload] = item;
      const notification = JSON.parse(payload);

      try {
        await sendNotification(notification);
      } catch (error) {
        await redis.lpush('queue:notifications:dlq', payload);
      }
    }
  }
}
```

## Pub/Sub for Real-Time Updates
```javascript
const subscriber = redis.duplicate();

await subscriber.subscribe('trip:updates', 'driver:location');

subscriber.on('message', (channel, message) => {
  const data = JSON.parse(message);

  if (channel === 'trip:updates') {
    broadcastToPassenger(data.tripId, data);
  } else if (channel === 'driver:location') {
    updateMapForPassenger(data.driverId, data.location);
  }
});

async function publishLocationUpdate(driverId, location) {
  await redis.publish('driver:location', JSON.stringify({
    driverId,
    location,
    timestamp: Date.now()
  }));
}
```

# Observability

## Monitoring Redis Performance
```javascript
async function getRedisStats() {
  const info = await redis.info('stats');
  const memory = await redis.info('memory');

  return {
    totalConnections: parseInfo(info, 'total_connections_received'),
    totalCommands: parseInfo(info, 'total_commands_processed'),
    opsPerSecond: parseInfo(info, 'instantaneous_ops_per_sec'),
    usedMemory: parseInfo(memory, 'used_memory_human'),
    evictedKeys: parseInfo(info, 'evicted_keys'),
    keyspaceHits: parseInfo(info, 'keyspace_hits'),
    keyspaceMisses: parseInfo(info, 'keyspace_misses')
  };
}
```

## Cache Hit Rate Monitoring
```javascript
async function getCacheHitRate() {
  const info = await redis.info('stats');
  const hits = parseInt(parseInfo(info, 'keyspace_hits'));
  const misses = parseInt(parseInfo(info, 'keyspace_misses'));

  const total = hits + misses;
  const hitRate = total > 0 ? (hits / total) * 100 : 0;

  return {
    hits,
    misses,
    total,
    hitRate: hitRate.toFixed(2) + '%'
  };
}
```

## Key Space Analysis
```javascript
async function analyzeKeyspace() {
  const keys = {};
  let cursor = '0';

  do {
    const [newCursor, batch] = await redis.scan(
      cursor,
      'MATCH',
      '*',
      'COUNT',
      1000
    );
    cursor = newCursor;

    for (const key of batch) {
      const type = key.split(':')[0];
      keys[type] = (keys[type] || 0) + 1;
    }
  } while (cursor !== '0');

  return keys;
}
```

# Testing Strategy

## Unit Tests
```javascript
describe('Vehicle Location Cache', () => {
  it('should update driver location', async () => {
    await updateDriverLocation('driver-123', 37.7749, -122.4194);

    const location = await redis.hgetall('vehicle:location:driver-123');
    expect(location.lat).toBe('37.7749');
    expect(location.lng).toBe('-122.4194');
  });

  it('should find nearby drivers within radius', async () => {
    await updateDriverLocation('driver-1', 37.7749, -122.4194);
    await updateDriverLocation('driver-2', 37.7750, -122.4195);

    const nearby = await findNearbyDrivers(37.7749, -122.4194, 1);
    expect(nearby.length).toBeGreaterThan(0);
  });
});
```

## Integration Tests
- Test cache invalidation on database updates
- Verify pub/sub message delivery
- Test distributed lock acquisition and release
- Validate rate limiting across multiple requests
- Ensure queue processing handles failures

## Load Tests
- Benchmark location updates (target: 10000+ updates/s)
- Test geospatial queries under load
- Measure pub/sub throughput
- Validate cache hit rates under production load
- Test connection pool behavior under stress

## Failover Tests
- Verify graceful degradation on cache failure
- Test automatic reconnection
- Validate data consistency after Redis restart
- Ensure no data loss during failover
