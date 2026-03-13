---
name: debugging-transport-system
description: Debug trip lifecycle, location updates, and queue processing issues
---

# Debugging Transport System

# Description

This skill provides systematic approaches to debugging common issues in transportation platforms, including trip lifecycle bugs, location update failures, queue processing errors, and real-time synchronization problems. It emphasizes root cause analysis and preventive measures.

# When to Activate

- Trips stuck in intermediate states
- Location updates not propagating to dashboards
- Queue messages not being processed
- Real-time subscriptions not firing
- Database inconsistencies in trip data
- Performance degradation during peak hours
- Integration failures with third-party services

# Engineering Principles

1. **Reproduce First**: Always reproduce the issue before attempting fixes
2. **Isolate the Problem**: Narrow down to specific component or service
3. **Check Logs**: Examine logs before making assumptions
4. **Validate Data**: Verify data integrity at each step
5. **Test Fixes**: Ensure fix resolves issue without introducing new problems
6. **Document Root Cause**: Record findings to prevent recurrence

# Implementation Patterns

## Debugging Setup

```typescript
src/
  utils/
    logger.ts
    debugger.ts
    errorTracker.ts
  middleware/
    requestLogger.ts
    errorHandler.ts
  tests/
    debug/
      trip-lifecycle.test.ts
      location-updates.test.ts
```

## Enhanced Logging System

```typescript
import winston from 'winston';

export enum LogLevel {
  ERROR = 'error',
  WARN = 'warn',
  INFO = 'info',
  DEBUG = 'debug'
}

interface LogContext {
  tripId?: string;
  driverId?: string;
  userId?: string;
  requestId?: string;
  [key: string]: any;
}

class TransportLogger {
  private logger: winston.Logger;

  constructor() {
    this.logger = winston.createLogger({
      level: process.env.LOG_LEVEL || 'info',
      format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.errors({ stack: true }),
        winston.format.json()
      ),
      transports: [
        new winston.transports.Console(),
        new winston.transports.File({
          filename: 'logs/error.log',
          level: 'error'
        }),
        new winston.transports.File({
          filename: 'logs/combined.log'
        })
      ]
    });
  }

  logTripEvent(
    level: LogLevel,
    event: string,
    context: LogContext,
    error?: Error
  ) {
    this.logger.log(level, event, {
      ...context,
      timestamp: new Date().toISOString(),
      error: error ? {
        message: error.message,
        stack: error.stack,
        name: error.name
      } : undefined
    });
  }

  logLocationUpdate(driverId: string, location: any) {
    this.logger.debug('Location update', {
      driverId,
      latitude: location.latitude,
      longitude: location.longitude,
      timestamp: location.timestamp,
      accuracy: location.accuracy
    });
  }

  logQueueEvent(queue: string, message: any, status: string) {
    this.logger.info('Queue event', {
      queue,
      messageId: message.id,
      status,
      attempts: message.attempts,
      timestamp: new Date().toISOString()
    });
  }
}

export const logger = new TransportLogger();
```

## Trip Lifecycle Debugger

```typescript
interface TripStateTransition {
  from: string;
  to: string;
  timestamp: Date;
  triggeredBy: string;
  metadata?: any;
}

class TripLifecycleDebugger {
  private transitions: Map<string, TripStateTransition[]> = new Map();

  recordTransition(tripId: string, transition: TripStateTransition) {
    if (!this.transitions.has(tripId)) {
      this.transitions.set(tripId, []);
    }
    this.transitions.get(tripId)!.push(transition);

    logger.logTripEvent(LogLevel.INFO, 'Trip state transition', {
      tripId,
      from: transition.from,
      to: transition.to,
      triggeredBy: transition.triggeredBy
    });
  }

  async validateTransition(tripId: string, newStatus: string): Promise<boolean> {
    const validTransitions = {
      pending: ['assigned', 'cancelled'],
      assigned: ['active', 'cancelled'],
      active: ['completed', 'cancelled'],
      completed: [],
      cancelled: []
    };

    const { data: trip } = await supabase
      .from('trips')
      .select('status')
      .eq('id', tripId)
      .maybeSingle();

    if (!trip) {
      logger.logTripEvent(LogLevel.ERROR, 'Trip not found', { tripId });
      return false;
    }

    const allowed = validTransitions[trip.status] || [];
    const isValid = allowed.includes(newStatus);

    if (!isValid) {
      logger.logTripEvent(LogLevel.WARN, 'Invalid state transition', {
        tripId,
        currentStatus: trip.status,
        attemptedStatus: newStatus,
        allowedTransitions: allowed
      });
    }

    return isValid;
  }

  getTransitionHistory(tripId: string): TripStateTransition[] {
    return this.transitions.get(tripId) || [];
  }

  async diagnoseStuckTrip(tripId: string) {
    const { data: trip } = await supabase
      .from('trips')
      .select('*, driver:drivers(*), vehicle:vehicles(*)')
      .eq('id', tripId)
      .maybeSingle();

    if (!trip) {
      return { error: 'Trip not found' };
    }

    const issues = [];
    const now = new Date();
    const createdAt = new Date(trip.created_at);
    const ageMinutes = (now.getTime() - createdAt.getTime()) / 60000;

    if (trip.status === 'pending' && ageMinutes > 10) {
      issues.push({
        type: 'STALE_PENDING',
        message: 'Trip pending for more than 10 minutes',
        age: ageMinutes
      });
    }

    if (trip.status === 'assigned' && !trip.driver_id) {
      issues.push({
        type: 'MISSING_DRIVER',
        message: 'Trip assigned but no driver assigned'
      });
    }

    if (trip.status === 'active' && ageMinutes > 120) {
      issues.push({
        type: 'LONG_ACTIVE',
        message: 'Trip active for more than 2 hours',
        age: ageMinutes
      });
    }

    const history = this.getTransitionHistory(tripId);

    return {
      trip,
      issues,
      transitionHistory: history,
      recommendations: this.generateRecommendations(issues)
    };
  }

  private generateRecommendations(issues: any[]): string[] {
    return issues.map(issue => {
      switch (issue.type) {
        case 'STALE_PENDING':
          return 'Consider auto-cancelling or reassigning trip';
        case 'MISSING_DRIVER':
          return 'Assign driver or reset trip to pending';
        case 'LONG_ACTIVE':
          return 'Contact driver to verify trip status';
        default:
          return 'Manual investigation required';
      }
    });
  }
}

export const tripDebugger = new TripLifecycleDebugger();
```

# Code Examples

## Location Update Debugging

```typescript
class LocationUpdateDebugger {
  private updateHistory: Map<string, any[]> = new Map();
  private readonly MAX_HISTORY = 100;

  async trackLocationUpdate(driverId: string, location: any) {
    const update = {
      ...location,
      timestamp: new Date(),
      receivedAt: Date.now()
    };

    if (!this.updateHistory.has(driverId)) {
      this.updateHistory.set(driverId, []);
    }

    const history = this.updateHistory.get(driverId)!;
    history.push(update);

    if (history.length > this.MAX_HISTORY) {
      history.shift();
    }

    const validation = this.validateLocation(location);
    if (!validation.valid) {
      logger.logTripEvent(LogLevel.WARN, 'Invalid location update', {
        driverId,
        location,
        errors: validation.errors
      });
    }

    const delay = this.calculateUpdateDelay(driverId);
    if (delay > 30000) {
      logger.logTripEvent(LogLevel.WARN, 'High location update delay', {
        driverId,
        delayMs: delay
      });
    }

    return validation;
  }

  validateLocation(location: any) {
    const errors = [];

    if (!location.latitude || !location.longitude) {
      errors.push('Missing coordinates');
    }

    if (location.latitude < -90 || location.latitude > 90) {
      errors.push('Invalid latitude');
    }

    if (location.longitude < -180 || location.longitude > 180) {
      errors.push('Invalid longitude');
    }

    if (location.accuracy && location.accuracy > 100) {
      errors.push('Low accuracy reading');
    }

    const age = Date.now() - new Date(location.timestamp).getTime();
    if (age > 60000) {
      errors.push('Stale location data');
    }

    return {
      valid: errors.length === 0,
      errors
    };
  }

  calculateUpdateDelay(driverId: string): number {
    const history = this.updateHistory.get(driverId);
    if (!history || history.length < 2) return 0;

    const latest = history[history.length - 1];
    const previous = history[history.length - 2];

    return latest.receivedAt - previous.receivedAt;
  }

  async diagnoseLocationIssues(driverId: string) {
    const history = this.updateHistory.get(driverId) || [];
    const recentUpdates = history.slice(-10);

    const avgDelay = recentUpdates.length > 1
      ? recentUpdates.reduce((sum, update, i) => {
          if (i === 0) return 0;
          return sum + (update.receivedAt - recentUpdates[i - 1].receivedAt);
        }, 0) / (recentUpdates.length - 1)
      : 0;

    const invalidUpdates = recentUpdates.filter(
      update => !this.validateLocation(update).valid
    );

    return {
      driverId,
      totalUpdates: history.length,
      recentUpdates: recentUpdates.length,
      averageDelay: avgDelay,
      invalidUpdates: invalidUpdates.length,
      lastUpdate: history[history.length - 1],
      issues: this.identifyIssues(avgDelay, invalidUpdates.length)
    };
  }

  private identifyIssues(avgDelay: number, invalidCount: number) {
    const issues = [];

    if (avgDelay > 30000) {
      issues.push({
        type: 'HIGH_LATENCY',
        message: 'Average update delay exceeds 30 seconds'
      });
    }

    if (invalidCount > 3) {
      issues.push({
        type: 'DATA_QUALITY',
        message: 'Multiple invalid location updates detected'
      });
    }

    return issues;
  }
}

export const locationDebugger = new LocationUpdateDebugger();
```

## Queue Processing Debugger

```typescript
interface QueueMessage {
  id: string;
  type: string;
  payload: any;
  attempts: number;
  createdAt: Date;
  processedAt?: Date;
  error?: string;
}

class QueueDebugger {
  private messageLog: Map<string, QueueMessage> = new Map();
  private deadLetterQueue: QueueMessage[] = [];

  async processMessage(message: QueueMessage) {
    this.messageLog.set(message.id, message);

    try {
      logger.logQueueEvent('processing', message, 'started');

      const startTime = Date.now();
      await this.handleMessage(message);
      const duration = Date.now() - startTime;

      message.processedAt = new Date();

      logger.logQueueEvent('processing', {
        ...message,
        duration
      }, 'completed');

      return { success: true };
    } catch (error) {
      message.attempts++;
      message.error = error.message;

      logger.logTripEvent(LogLevel.ERROR, 'Queue processing failed', {
        messageId: message.id,
        type: message.type,
        attempts: message.attempts,
        error: error.message
      }, error);

      if (message.attempts >= 3) {
        this.moveToDeadLetter(message);
      }

      return { success: false, error: error.message };
    }
  }

  private async handleMessage(message: QueueMessage) {
    switch (message.type) {
      case 'trip.created':
        await this.handleTripCreated(message.payload);
        break;
      case 'trip.assigned':
        await this.handleTripAssigned(message.payload);
        break;
      case 'location.updated':
        await this.handleLocationUpdated(message.payload);
        break;
      default:
        throw new Error(`Unknown message type: ${message.type}`);
    }
  }

  private async handleTripCreated(payload: any) {
    const { data, error } = await supabase
      .from('trips')
      .select('*')
      .eq('id', payload.tripId)
      .maybeSingle();

    if (error || !data) {
      throw new Error(`Trip not found: ${payload.tripId}`);
    }
  }

  private async handleTripAssigned(payload: any) {
  }

  private async handleLocationUpdated(payload: any) {
  }

  private moveToDeadLetter(message: QueueMessage) {
    this.deadLetterQueue.push(message);

    logger.logTripEvent(LogLevel.ERROR, 'Message moved to dead letter queue', {
      messageId: message.id,
      type: message.type,
      attempts: message.attempts,
      error: message.error
    });
  }

  getDeadLetterQueue(): QueueMessage[] {
    return this.deadLetterQueue;
  }

  async diagnoseQueueHealth() {
    const messages = Array.from(this.messageLog.values());
    const recent = messages.filter(m => {
      const age = Date.now() - m.createdAt.getTime();
      return age < 3600000;
    });

    const failed = recent.filter(m => m.attempts > 0);
    const avgProcessingTime = recent
      .filter(m => m.processedAt)
      .reduce((sum, m) => {
        return sum + (m.processedAt!.getTime() - m.createdAt.getTime());
      }, 0) / recent.filter(m => m.processedAt).length;

    return {
      totalMessages: messages.length,
      recentMessages: recent.length,
      failedMessages: failed.length,
      deadLetterCount: this.deadLetterQueue.length,
      averageProcessingTime: avgProcessingTime,
      failureRate: recent.length > 0 ? failed.length / recent.length : 0
    };
  }
}

export const queueDebugger = new QueueDebugger();
```

## Real-time Subscription Debugger

```typescript
class RealtimeDebugger {
  private subscriptions: Map<string, any> = new Map();
  private eventCounts: Map<string, number> = new Map();

  monitorSubscription(channel: string, config: any) {
    const wrappedConfig = {
      ...config,
      onSubscribe: () => {
        logger.logTripEvent(LogLevel.INFO, 'Subscription established', {
          channel
        });
        config.onSubscribe?.();
      },
      onError: (error: Error) => {
        logger.logTripEvent(LogLevel.ERROR, 'Subscription error', {
          channel,
          error: error.message
        }, error);
        config.onError?.(error);
      }
    };

    const subscription = supabase
      .channel(channel)
      .on('postgres_changes', config.filter, (payload) => {
        this.recordEvent(channel, payload.eventType);

        logger.logTripEvent(LogLevel.DEBUG, 'Realtime event received', {
          channel,
          eventType: payload.eventType,
          table: payload.table
        });

        config.callback(payload);
      })
      .subscribe();

    this.subscriptions.set(channel, subscription);
    return subscription;
  }

  private recordEvent(channel: string, eventType: string) {
    const key = `${channel}:${eventType}`;
    this.eventCounts.set(key, (this.eventCounts.get(key) || 0) + 1);
  }

  getSubscriptionStats() {
    return {
      activeSubscriptions: this.subscriptions.size,
      eventCounts: Object.fromEntries(this.eventCounts)
    };
  }

  async testSubscription(channel: string, table: string) {
    let eventReceived = false;
    const timeout = 10000;

    const testChannel = supabase
      .channel(`test-${channel}`)
      .on('postgres_changes',
        { event: '*', schema: 'public', table },
        () => {
          eventReceived = true;
        }
      )
      .subscribe();

    await new Promise(resolve => setTimeout(resolve, 1000));

    const { error } = await supabase
      .from(table)
      .select('id')
      .limit(1);

    await new Promise(resolve => setTimeout(resolve, timeout));

    testChannel.unsubscribe();

    return {
      success: eventReceived,
      message: eventReceived
        ? 'Subscription working correctly'
        : 'No events received within timeout period'
    };
  }
}

export const realtimeDebugger = new RealtimeDebugger();
```

# Anti-Patterns

❌ **Debugging in production without logging**
```typescript
try {
  await updateTrip(tripId);
} catch (error) {
  console.log(error);
}
```

✅ **Proper error logging with context**
```typescript
try {
  await updateTrip(tripId);
} catch (error) {
  logger.logTripEvent(LogLevel.ERROR, 'Failed to update trip', {
    tripId,
    operation: 'updateTrip'
  }, error);
  throw error;
}
```

❌ **Ignoring validation errors**
```typescript
const location = await getLocation();
await saveLocation(location);
```

✅ **Validate before processing**
```typescript
const location = await getLocation();
const validation = locationDebugger.validateLocation(location);
if (!validation.valid) {
  throw new Error(`Invalid location: ${validation.errors.join(', ')}`);
}
await saveLocation(location);
```

# Performance Guidelines

1. **Enable Debug Mode Conditionally**: Only in development or when explicitly needed
2. **Limit Log Retention**: Rotate logs daily, keep max 7 days
3. **Sample High-Volume Events**: Log 1% of location updates in production
4. **Use Async Logging**: Don't block request processing for logging
5. **Index Log Fields**: Ensure queryable fields are indexed

# Observability

## Debug Dashboard

```typescript
app.get('/debug/trips/:tripId', async (req, res) => {
  const { tripId } = req.params;
  const diagnosis = await tripDebugger.diagnoseStuckTrip(tripId);
  res.json(diagnosis);
});

app.get('/debug/driver/:driverId/location', async (req, res) => {
  const { driverId } = req.params;
  const diagnosis = await locationDebugger.diagnoseLocationIssues(driverId);
  res.json(diagnosis);
});

app.get('/debug/queue/health', async (req, res) => {
  const health = await queueDebugger.diagnoseQueueHealth();
  res.json(health);
});

app.get('/debug/realtime/stats', async (req, res) => {
  const stats = realtimeDebugger.getSubscriptionStats();
  res.json(stats);
});
```

# Testing Strategy

## Debugging Test Suite

```typescript
import { describe, test, expect } from 'vitest';
import { tripDebugger } from './tripDebugger';

describe('Trip Lifecycle Debugger', () => {
  test('detects invalid state transitions', async () => {
    const isValid = await tripDebugger.validateTransition(
      'trip-123',
      'completed'
    );

    expect(isValid).toBe(false);
  });

  test('identifies stuck trips', async () => {
    const diagnosis = await tripDebugger.diagnoseStuckTrip('trip-123');

    expect(diagnosis.issues).toContainEqual(
      expect.objectContaining({ type: 'STALE_PENDING' })
    );
  });
});

describe('Location Update Debugger', () => {
  test('validates location coordinates', () => {
    const validation = locationDebugger.validateLocation({
      latitude: 91,
      longitude: 0,
      timestamp: new Date()
    });

    expect(validation.valid).toBe(false);
    expect(validation.errors).toContain('Invalid latitude');
  });
});
```
