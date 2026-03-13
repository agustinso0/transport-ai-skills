---
name: event-driven-transport
description: Event-driven architecture patterns for transportation systems including event publishing, consumption, saga patterns, and event sourcing for reliable distributed systems.
---

# Event-Driven Transport

## Description

This skill teaches how to build resilient event-driven transportation systems. It covers domain event design, event publishing patterns, event consumption strategies, saga pattern implementation for distributed transactions, event sourcing for complete audit trails, and dead-letter queue handling. This enables building loosely coupled services that reliably handle complex business processes and maintain data consistency across service boundaries.

## When to Activate

Activate this skill when:

- Designing event-driven transportation systems
- Implementing domain events for trips
- Setting up event publishers and subscribers
- Handling asynchronous event consumption
- Implementing saga patterns for distributed transactions
- Designing event sourcing for audit trails
- Handling event ordering and delivery guarantees
- Implementing dead-letter queues
- Designing event schema versioning
- Implementing event replays
- Handling idempotency in event handlers
- Designing event-driven databases
- Building real-time event dashboards
- Implementing complex workflow orchestration
- Handling temporal data and event history

## Engineering Principles

### Domain-Driven Events

Events represent facts that happened in the domain:

```typescript
// Good: Domain event
interface TripAssignedEvent {
  tripId: string;
  driverId: string;
  vehicleId: string;
  assignedAt: Date;
  estimatedPickupTime: Date;
}

// Bad: Service event (implementation detail)
interface DriverStatusUpdatedEvent {
  driverId: string;
  isNowOnTrip: boolean;
}
```

Events describe WHAT happened, not HOW:

```typescript
// Good: Fact-based event
export class TripStartedEvent {
  constructor(
    public readonly tripId: string,
    public readonly driverId: string,
    public readonly startTime: Date,
    public readonly location: Location,
  ) {}
}

// Bad: Imperative event
export class StartTripCommandExecuted {
  driverId: string;
  newStatus: string;
  // Doesn't convey the business fact
}
```

### Event Immutability

Events are immutable facts:

```typescript
export class TripCompletedEvent {
  private readonly _tripId: string;
  private readonly _driverId: string;
  private readonly _completedAt: Date;
  private readonly _distanceKm: number;

  constructor(
    tripId: string,
    driverId: string,
    completedAt: Date,
    distanceKm: number,
  ) {
    this._tripId = tripId;
    this._driverId = driverId;
    this._completedAt = completedAt;
    this._distanceKm = distanceKm;
  }

  get tripId(): string { return this._tripId; }
  get driverId(): string { return this._driverId; }
  get completedAt(): Date { return this._completedAt; }
  get distanceKm(): number { return this._distanceKm; }
}
```

### Event Ordering

Maintain causal ordering:

```
Trip 1:
1. trip.created (id: trip-1, seq: 1)
2. trip.assigned (id: trip-1, seq: 2)
3. trip.started (id: trip-1, seq: 3)
4. trip.completed (id: trip-1, seq: 4)

Trip 2:
1. trip.created (id: trip-2, seq: 1)
2. trip.cancelled (id: trip-2, seq: 2)

Events from different trips can be reordered, but
events from same trip must maintain order
```

### Idempotency

Handle duplicate event processing:

```typescript
@Injectable()
export class EventHandler {
  constructor(
    private readonly eventStore: EventStore,
    private readonly notificationService: NotificationService,
  ) {}

  async handleTripCompleted(event: TripCompletedEvent) {
    // Check if event already processed
    const processed = await this.eventStore.isProcessed(
      event.tripId,
      event.id,
    );

    if (processed) {
      return; // Already handled
    }

    // Process event
    await this.notificationService.notifyCompletion(event);

    // Mark as processed
    await this.eventStore.markProcessed(event.tripId, event.id);
  }
}
```

## Backend Architecture

### Event Bus Implementation

```typescript
export interface DomainEvent {
  id: string;
  occurredAt: Date;
  aggregateId: string;
  aggregateType: string;
  eventType: string;
  version: number;
  correlationId?: string;
  causationId?: string;
  payload: unknown;
}

@Injectable()
export class EventBus {
  private subscriptions: Map<string, EventHandler[]> = new Map();

  subscribe(
    eventType: string,
    handler: (event: DomainEvent) => Promise<void>,
  ) {
    if (!this.subscriptions.has(eventType)) {
      this.subscriptions.set(eventType, []);
    }
    this.subscriptions.get(eventType).push(handler);
  }

  async publish(event: DomainEvent) {
    const handlers = this.subscriptions.get(event.eventType) || [];

    // Publish to message queue for persistence
    await this.messageQueue.publish(event);

    // Execute local handlers
    const promises = handlers.map(handler =>
      handler(event).catch(error => {
        this.logger.error(`Error handling ${event.eventType}`, error);
        // Send to dead-letter queue
        return this.deadLetterQueue.send(event, error);
      }),
    );

    await Promise.all(promises);
  }
}
```

### Event Store

```typescript
@Injectable()
export class EventStore {
  constructor(
    @InjectRepository(StoredEvent)
    private readonly eventRepository: Repository<StoredEvent>,
  ) {}

  async append(event: DomainEvent): Promise<void> {
    const stored = this.eventRepository.create({
      id: event.id,
      eventType: event.eventType,
      aggregateId: event.aggregateId,
      aggregateType: event.aggregateType,
      payload: event.payload,
      metadata: {
        occurredAt: event.occurredAt,
        correlationId: event.correlationId,
        causationId: event.causationId,
      },
      timestamp: event.occurredAt,
    });

    await this.eventRepository.save(stored);
  }

  async getEvents(aggregateId: string): Promise<DomainEvent[]> {
    return this.eventRepository.find({
      where: { aggregateId },
      order: { timestamp: 'ASC' },
    });
  }

  async getEventsSince(timestamp: Date): Promise<DomainEvent[]> {
    return this.eventRepository.find({
      where: { timestamp: MoreThan(timestamp) },
      order: { timestamp: 'ASC' },
    });
  }

  async isProcessed(aggregateId: string, eventId: string): Promise<boolean> {
    const processed = await this.eventRepository.count({
      where: {
        aggregateId,
        id: eventId,
        processedAt: Not(IsNull()),
      },
    });
    return processed > 0;
  }

  async markProcessed(aggregateId: string, eventId: string): Promise<void> {
    await this.eventRepository.update(
      { aggregateId, id: eventId },
      { processedAt: new Date() },
    );
  }
}
```

## Implementation Patterns

### Trip Domain Events

Define core trip events:

```typescript
// Trip lifecycle events
export class TripCreatedEvent {
  constructor(
    public readonly tripId: string,
    public readonly origin: Location,
    public readonly destination: Location,
    public readonly createdAt: Date,
  ) {}
}

export class TripAssignedEvent {
  constructor(
    public readonly tripId: string,
    public readonly driverId: string,
    public readonly vehicleId: string,
    public readonly estimatedPickupTime: Date,
  ) {}
}

export class TripStartedEvent {
  constructor(
    public readonly tripId: string,
    public readonly driverId: string,
    public readonly startedAt: Date,
    public readonly location: Location,
  ) {}
}

export class TripInProgressEvent {
  constructor(
    public readonly tripId: string,
    public readonly pickedUpAt: Date,
    public readonly pickupLocation: Location,
  ) {}
}

export class TripCompletedEvent {
  constructor(
    public readonly tripId: string,
    public readonly driverId: string,
    public readonly completedAt: Date,
    public readonly dropoffLocation: Location,
    public readonly distanceKm: number,
    public readonly durationMinutes: number,
    public readonly fare: number,
  ) {}
}

export class TripCancelledEvent {
  constructor(
    public readonly tripId: string,
    public readonly cancelledAt: Date,
    public readonly reason: string,
    public readonly cancellationFee?: number,
  ) {}
}
```

### Event Publishing in Services

```typescript
@Injectable()
export class TripService {
  constructor(
    private readonly tripRepository: TripRepository,
    private readonly eventStore: EventStore,
    private readonly eventBus: EventBus,
  ) {}

  async createTrip(createTripDTO: CreateTripDTO): Promise<Trip> {
    // Create trip
    const trip = this.tripRepository.create(createTripDTO);
    const savedTrip = await this.tripRepository.save(trip);

    // Create domain event
    const event = new TripCreatedEvent(
      savedTrip.id,
      savedTrip.origin,
      savedTrip.destination,
      new Date(),
    );

    // Store event
    await this.eventStore.append({
      id: generateId(),
      occurredAt: event.occurredAt,
      aggregateId: event.tripId,
      aggregateType: 'Trip',
      eventType: 'trip.created',
      version: 1,
      payload: event,
    });

    // Publish event
    await this.eventBus.publish({
      id: generateId(),
      occurredAt: event.occurredAt,
      aggregateId: event.tripId,
      aggregateType: 'Trip',
      eventType: 'trip.created',
      version: 1,
      payload: event,
    });

    return savedTrip;
  }

  async completeTrip(
    tripId: string,
    completionData: TripCompletionData,
  ): Promise<Trip> {
    const trip = await this.tripRepository.findOne({ where: { id: tripId } });

    if (!trip || trip.status !== 'in_progress') {
      throw new BadRequestException('Trip cannot be completed');
    }

    // Update trip
    trip.status = 'completed';
    trip.completedAt = completionData.completedAt;
    trip.dropoffLocation = completionData.location;
    trip.distanceKm = completionData.distance;
    trip.durationMinutes = completionData.duration;

    const updated = await this.tripRepository.save(trip);

    // Create domain event
    const event = new TripCompletedEvent(
      updated.id,
      updated.driverId,
      updated.completedAt,
      updated.dropoffLocation,
      updated.distanceKm,
      updated.durationMinutes,
      this.calculateFare(updated.distanceKm),
    );

    // Store and publish event
    await this.eventStore.append({
      id: generateId(),
      occurredAt: event.completedAt,
      aggregateId: event.tripId,
      aggregateType: 'Trip',
      eventType: 'trip.completed',
      version: 2, // Increment version
      payload: event,
    });

    await this.eventBus.publish({
      id: generateId(),
      occurredAt: event.completedAt,
      aggregateId: event.tripId,
      aggregateType: 'Trip',
      eventType: 'trip.completed',
      version: 2,
      payload: event,
    });

    return updated;
  }

  private calculateFare(distanceKm: number): number {
    return 5 + (distanceKm * 1.5);
  }
}
```

### Event Saga Pattern

Handle distributed transactions:

```typescript
@Injectable()
export class TripAssignmentSaga {
  constructor(
    private readonly tripService: TripService,
    private readonly driverClient: DriverServiceClient,
    private readonly vehicleClient: VehicleServiceClient,
    private readonly eventBus: EventBus,
    private readonly sagaRepository: SagaRepository,
  ) {
    this.registerEventHandlers();
  }

  private registerEventHandlers() {
    // When trip is created, start assignment saga
    this.eventBus.subscribe('trip.created', (event) => {
      this.startAssignmentSaga(event.payload as TripCreatedEvent);
    });

    // When driver is assigned, request vehicle
    this.eventBus.subscribe('driver.assigned', (event) => {
      this.assignVehicle(event.payload);
    });

    // When vehicle is assigned, complete saga
    this.eventBus.subscribe('vehicle.assigned', (event) => {
      this.completeSaga(event.payload);
    });
  }

  private async startAssignmentSaga(event: TripCreatedEvent): Promise<void> {
    // Create saga record
    const saga = {
      id: generateId(),
      tripId: event.tripId,
      status: 'started',
      createdAt: new Date(),
    };

    await this.sagaRepository.save(saga);

    // Request driver assignment
    try {
      await this.driverClient.assignDriver({
        tripId: event.tripId,
        origin: event.origin,
      });
    } catch (error) {
      // Compensating transaction: cancel trip
      await this.tripService.cancelTrip(event.tripId, 'Driver assignment failed');
      saga.status = 'failed';
      await this.sagaRepository.update(saga.id, saga);
    }
  }

  private async assignVehicle(driverData: any): Promise<void> {
    const saga = await this.sagaRepository.findOne({
      where: { tripId: driverData.tripId },
    });

    try {
      await this.vehicleClient.assignVehicle({
        tripId: driverData.tripId,
        driverId: driverData.driverId,
      });
    } catch (error) {
      // Compensating transaction: unassign driver
      saga.status = 'failed';
      await this.sagaRepository.update(saga.id, saga);
    }
  }

  private async completeSaga(vehicleData: any): Promise<void> {
    const saga = await this.sagaRepository.findOne({
      where: { tripId: vehicleData.tripId },
    });

    saga.status = 'completed';
    saga.completedAt = new Date();
    await this.sagaRepository.update(saga.id, saga);
  }
}
```

### Event Consumption

```typescript
@Controller('webhooks')
export class EventConsumerController {
  constructor(
    private readonly tripEventHandler: TripEventHandler,
    private readonly notificationEventHandler: NotificationEventHandler,
  ) {}

  @Post('trip-events')
  async handleTripEvent(@Body() event: DomainEvent): Promise<void> {
    switch (event.eventType) {
      case 'trip.completed':
        await this.tripEventHandler.handleCompleted(event);
        break;
      case 'trip.assigned':
        await this.notificationEventHandler.handleAssigned(event);
        break;
      case 'trip.started':
        await this.notificationEventHandler.handleStarted(event);
        break;
      default:
        this.logger.warn(`Unknown event type: ${event.eventType}`);
    }
  }
}

@Injectable()
export class NotificationEventHandler {
  constructor(
    private readonly notificationService: NotificationService,
  ) {}

  async handleAssigned(event: DomainEvent): Promise<void> {
    const tripEvent = event.payload as TripAssignedEvent;

    await this.notificationService.sendTripAssignmentNotification({
      driverId: tripEvent.driverId,
      tripId: tripEvent.tripId,
      estimatedPickupTime: tripEvent.estimatedPickupTime,
    });
  }

  async handleStarted(event: DomainEvent): Promise<void> {
    const tripEvent = event.payload as TripStartedEvent;

    await this.notificationService.sendTripStartedNotification({
      driverId: tripEvent.driverId,
      tripId: tripEvent.tripId,
      location: tripEvent.location,
    });
  }
}
```

### Dead-Letter Queue Handling

```typescript
@Injectable()
export class DeadLetterQueueService {
  constructor(
    private readonly dlqRepository: Repository<DeadLetterQueue>,
    private readonly logger: Logger,
  ) {}

  async handleFailedEvent(
    event: DomainEvent,
    error: Error,
    attempt: number,
  ): Promise<void> {
    const dlqEntry = this.dlqRepository.create({
      eventId: event.id,
      eventType: event.eventType,
      aggregateId: event.aggregateId,
      payload: event.payload,
      error: error.message,
      errorStack: error.stack,
      attempt,
      failedAt: new Date(),
      status: attempt >= 3 ? 'manual_review_required' : 'pending_retry',
    });

    await this.dlqRepository.save(dlqEntry);

    this.logger.error({
      type: 'dead_letter_queue',
      eventId: event.id,
      eventType: event.eventType,
      error: error.message,
      attempt,
    });

    // Alert if manual review needed
    if (attempt >= 3) {
      await this.alertAdmins(dlqEntry);
    }
  }

  async retryFailedEvent(dlqId: string): Promise<void> {
    const dlqEntry = await this.dlqRepository.findOne(dlqId);

    if (!dlqEntry) {
      throw new NotFoundException('DLQ entry not found');
    }

    // Recreate event from DLQ entry
    const event: DomainEvent = {
      id: dlqEntry.eventId,
      eventType: dlqEntry.eventType,
      aggregateId: dlqEntry.aggregateId,
      aggregateType: 'Trip',
      occurredAt: dlqEntry.failedAt,
      version: 1,
      payload: dlqEntry.payload,
    };

    // Republish event
    await this.eventBus.publish(event);

    dlqEntry.status = 'retried';
    dlqEntry.retriedAt = new Date();
    await this.dlqRepository.save(dlqEntry);
  }
}
```

### Event Projection

```typescript
@Injectable()
export class TripProjectionService {
  constructor(
    @InjectRepository(TripProjection)
    private readonly projectionRepository: Repository<TripProjection>,
  ) {}

  async handleTripCreated(event: TripCreatedEvent): Promise<void> {
    const projection = this.projectionRepository.create({
      tripId: event.tripId,
      status: 'created',
      origin: event.origin,
      destination: event.destination,
      createdAt: event.createdAt,
      updatedAt: event.createdAt,
    });

    await this.projectionRepository.save(projection);
  }

  async handleTripAssigned(event: TripAssignedEvent): Promise<void> {
    await this.projectionRepository.update(
      { tripId: event.tripId },
      {
        status: 'assigned',
        driverId: event.driverId,
        vehicleId: event.vehicleId,
        estimatedPickupTime: event.estimatedPickupTime,
        updatedAt: new Date(),
      },
    );
  }

  async handleTripCompleted(event: TripCompletedEvent): Promise<void> {
    await this.projectionRepository.update(
      { tripId: event.tripId },
      {
        status: 'completed',
        completedAt: event.completedAt,
        distanceKm: event.distanceKm,
        fare: event.fare,
        updatedAt: event.completedAt,
      },
    );
  }

  async getActiveTripsByDriver(driverId: string): Promise<TripProjection[]> {
    return this.projectionRepository.find({
      where: {
        driverId,
        status: In(['assigned', 'started', 'in_progress']),
      },
    });
  }
}
```

## Anti-Patterns

### Losing Events

**DON'T:**
```typescript
// Publishing without persistence
async publishEvent(event: DomainEvent) {
  // If service crashes here, event is lost
  await this.eventBus.publish(event);
  // Database update after
  await this.repository.update(event.aggregateId, data);
}
```

**DO:**
```typescript
// Persist first, then publish
async publishEvent(event: DomainEvent) {
  // Store event first
  await this.eventStore.append(event);

  // Then publish (can be retried if fails)
  try {
    await this.eventBus.publish(event);
  } catch {
    // Event is already stored, can be replayed
    throw;
  }
}
```

### Non-Idempotent Event Handlers

**DON'T:**
```typescript
async handleTripCompleted(event: TripCompletedEvent) {
  // No idempotency check
  // If message is redelivered, fare is charged twice
  await this.billingService.chargeFare(event.tripId, event.fare);
}
```

**DO:**
```typescript
async handleTripCompleted(event: TripCompletedEvent) {
  // Check if already processed
  if (await this.isEventProcessed(event.id)) {
    return;
  }

  await this.billingService.chargeFare(event.tripId, event.fare);

  // Mark as processed
  await this.markEventProcessed(event.id);
}
```

### Event Ordering Violations

**DON'T:**
```typescript
// Parallel processing loses order
events.forEach(event => {
  // Each runs in parallel, order not guaranteed
  this.handleEvent(event);
});
```

**DO:**
```typescript
// Sequential processing per aggregate
for (const event of events) {
  // Same aggregate events processed sequentially
  if (event.aggregateId === currentAggregateId) {
    await this.handleEvent(event);
  }
}
```

### Tight Coupling to Event Details

**DON'T:**
```typescript
// Handler tightly coupled to event fields
async handleTripCompleted(event: any) {
  const fare = event.fare; // Direct field access
  const distance = event.distanceKm;
  // Breaks if event schema changes
}
```

**DO:**
```typescript
// Handler uses typed events
async handleTripCompleted(event: TripCompletedEvent) {
  const fare = event.fare; // Type-safe access
  const distance = event.distanceKm;
  // Change event = change type = catch at compile time
}
```

## Performance Guidelines

### Batch Event Processing

```typescript
@Injectable()
export class BatchEventProcessor {
  private batch: DomainEvent[] = [];
  private batchSize = 100;

  async addEvent(event: DomainEvent) {
    this.batch.push(event);

    if (this.batch.length >= this.batchSize) {
      await this.processBatch();
    }
  }

  private async processBatch() {
    const batch = this.batch.splice(0, this.batchSize);

    // Bulk insert all events
    await this.eventStore.appendBatch(batch);

    // Publish all at once
    await this.messageQueue.publishBatch(batch);
  }
}
```

### Event Replay Optimization

```typescript
@Injectable()
export class EventReplayer {
  async replayEvents(
    aggregateId: string,
    fromVersion?: number,
  ): Promise<Trip> {
    const events = await this.eventStore.getEvents(aggregateId);
    let trip = new Trip();

    for (const event of events) {
      if (fromVersion && event.version < fromVersion) {
        continue;
      }

      // Apply event to aggregate
      trip = this.applyEvent(trip, event);

      // Snapshot every 100 events for faster replay
      if (event.version % 100 === 0) {
        await this.snapshotRepository.save({
          aggregateId,
          version: event.version,
          state: trip,
        });
      }
    }

    return trip;
  }

  private applyEvent(aggregate: Trip, event: DomainEvent): Trip {
    switch (event.eventType) {
      case 'trip.created':
        return this.applyTripCreated(aggregate, event);
      case 'trip.assigned':
        return this.applyTripAssigned(aggregate, event);
      // ... more event types
      default:
        return aggregate;
    }
  }
}
```

## Observability

### Event Metrics

```typescript
@Injectable()
export class EventMetrics {
  private readonly eventsPublished = new Counter({
    name: 'events_published_total',
    help: 'Total events published',
    labelNames: ['event_type', 'aggregate_type'],
  });

  private readonly eventProcessingDuration = new Histogram({
    name: 'event_processing_duration_seconds',
    help: 'Event processing latency',
    labelNames: ['event_type', 'handler'],
  });

  recordPublished(eventType: string, aggregateType: string) {
    this.eventsPublished.labels(eventType, aggregateType).inc();
  }

  recordProcessing(
    eventType: string,
    handler: string,
    durationMs: number,
  ) {
    this.eventProcessingDuration
      .labels(eventType, handler)
      .observe(durationMs / 1000);
  }
}
```

### Event Tracing

```typescript
@Injectable()
export class EventTracing {
  async traceEventProcessing<T>(
    event: DomainEvent,
    handler: (event: DomainEvent) => Promise<T>,
  ): Promise<T> {
    const span = this.tracer.startSpan('event.process', {
      tags: {
        'event.type': event.eventType,
        'event.id': event.id,
        'aggregate.id': event.aggregateId,
        'correlation.id': event.correlationId,
      },
    });

    try {
      return await handler(event);
    } catch (error) {
      span.setTag('error', true);
      span.log({ message: error.message });
      throw error;
    } finally {
      span.finish();
    }
  }
}
```

## Testing Strategy

### Event Test Fixtures

```typescript
describe('Trip Events', () => {
  let eventBus: EventBus;
  let tripService: TripService;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        TripService,
        EventBus,
        EventStore,
        {
          provide: Repository,
          useValue: {
            create: jest.fn(),
            save: jest.fn(),
          },
        },
      ],
    }).compile();

    eventBus = module.get<EventBus>(EventBus);
    tripService = module.get<TripService>(TripService);
  });

  test('should publish trip.created event', async () => {
    const publishSpy = jest.spyOn(eventBus, 'publish');

    await tripService.createTrip({
      origin: { lat: 0, lng: 0 },
      destination: { lat: 1, lng: 1 },
    });

    expect(publishSpy).toHaveBeenCalledWith(
      expect.objectContaining({
        eventType: 'trip.created',
        aggregateId: expect.any(String),
      }),
    );
  });
});
```

### Event Handler Testing

```typescript
describe('Trip Event Handlers', () => {
  let handler: NotificationEventHandler;
  let notificationService: NotificationService;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        NotificationEventHandler,
        {
          provide: NotificationService,
          useValue: {
            sendNotification: jest.fn(),
          },
        },
      ],
    }).compile();

    handler = module.get<NotificationEventHandler>(NotificationEventHandler);
    notificationService = module.get<NotificationService>(NotificationService);
  });

  test('should send notification when trip is assigned', async () => {
    const event: TripAssignedEvent = {
      tripId: 'trip-123',
      driverId: 'driver-123',
      vehicleId: 'vehicle-123',
      estimatedPickupTime: new Date(),
    };

    await handler.handleAssigned(event);

    expect(notificationService.sendNotification).toHaveBeenCalledWith(
      expect.objectContaining({
        driverId: 'driver-123',
      }),
    );
  });
});
```
