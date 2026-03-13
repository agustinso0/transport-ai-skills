---
name: microservices-transport
description: Microservices architecture patterns for transportation platforms including trip, driver, route, and notification services with service boundaries and communication patterns.
---

# Microservices Transport

## Description

This skill teaches how to architect and implement a microservices-based transportation platform. It covers designing independent services for trips, drivers, routes, and notifications with clear service boundaries, asynchronous communication patterns, distributed transactions, and deployment strategies. This enables building scalable, resilient transportation systems that can evolve independently.

## When to Activate

Activate this skill when:

- Designing microservices architecture for transportation
- Implementing trip management microservice
- Building driver management microservice
- Creating route calculation microservice
- Implementing notification/messaging service
- Designing service-to-service communication
- Handling distributed transactions
- Implementing saga patterns
- Managing service dependencies
- Designing API gateways
- Implementing service discovery
- Handling cross-service data consistency
- Scaling individual services
- Implementing circuit breakers
- Managing service deployments

## Engineering Principles

### Service Boundaries

Define clear service responsibilities:

**Trip Service**
- Owns trip data and lifecycle
- Manages trip state transitions
- Publishes trip events
- Responds to assignment requests

**Driver Service**
- Owns driver data and availability
- Manages driver status
- Publishes driver events
- Provides driver availability queries

**Route Service**
- Calculates optimal routes
- Manages route history
- Provides distance/duration estimates
- Handles route optimization requests

**Notification Service**
- Consumes events from other services
- Sends notifications (SMS, email, push)
- Manages notification templates
- Tracks notification delivery

**Vehicle Service**
- Owns vehicle data
- Manages vehicle availability
- Tracks vehicle location
- Publishes vehicle events

### Data Isolation

Each service owns its data:

```
Trip Service
├── trips table
├── trip_events table
└── trip_assignments table

Driver Service
├── drivers table
├── driver_status_history table
└── driver_documents table

Route Service
├── routes table
├── route_waypoints table
└── route_history table

Notification Service
├── notifications table
├── notification_templates table
└── notification_delivery_logs table
```

Services communicate through:
- REST APIs for synchronous queries
- Events for asynchronous notifications
- Event sourcing for audit trails

### Asynchronous Communication

Use event-driven patterns:

```
Trip Service publishes:
- trip.created
- trip.assigned
- trip.started
- trip.completed
- trip.cancelled

Driver Service publishes:
- driver.created
- driver.status_changed
- driver.available
- driver.unavailable

Route Service publishes:
- route.calculated
- route.optimized

Vehicle Service publishes:
- vehicle.location_updated
- vehicle.status_changed
```

## Backend Architecture

### Microservice Structure

Each service follows this structure:

```
trip-service/
├── src/
│   ├── domain/
│   │   ├── entities/
│   │   ├── value-objects/
│   │   └── services/
│   ├── application/
│   │   ├── dto/
│   │   ├── services/
│   │   └── events/
│   ├── infrastructure/
│   │   ├── persistence/
│   │   ├── messaging/
│   │   └── http/
│   ├── presentation/
│   │   └── controllers/
│   ├── config/
│   └── main.ts
├── Dockerfile
├── docker-compose.yml
└── package.json
```

### Service Discovery

Register services in a service registry:

```typescript
@Module({
  imports: [
    ConsulModule.forRoot({
      host: process.env.CONSUL_HOST,
      port: parseInt(process.env.CONSUL_PORT),
    }),
  ],
})
export class AppModule {
  constructor(
    private readonly consul: ConsulService,
  ) {
    this.registerService();
  }

  private registerService() {
    this.consul.agent.service.register({
      id: `trip-service-${process.env.NODE_ENV}`,
      name: 'trip-service',
      address: process.env.SERVICE_HOST,
      port: parseInt(process.env.SERVICE_PORT),
      check: {
        http: `http://${process.env.SERVICE_HOST}:${process.env.SERVICE_PORT}/health`,
        interval: '10s',
        timeout: '5s',
      },
    });
  }
}
```

### API Gateway

Route requests to appropriate services:

```typescript
@Controller()
export class ApiGatewayController {
  constructor(
    private readonly httpService: HttpService,
  ) {}

  @Post('drivers')
  async createDriver(@Body() dto: CreateDriverDTO) {
    return this.httpService.post(
      `${process.env.DRIVER_SERVICE_URL}/drivers`,
      dto,
    ).toPromise();
  }

  @Post('trips')
  async createTrip(@Body() dto: CreateTripDTO) {
    return this.httpService.post(
      `${process.env.TRIP_SERVICE_URL}/trips`,
      dto,
    ).toPromise();
  }

  @Get('trips/:id')
  async getTrip(@Param('id') id: string) {
    return this.httpService.get(
      `${process.env.TRIP_SERVICE_URL}/trips/${id}`,
    ).toPromise();
  }
}
```

### Message Broker Configuration

Use message broker for service communication:

```typescript
@Module({
  imports: [
    RabbitMQModule.forRoot({
      exchanges: [
        {
          name: 'transport.events',
          type: 'topic',
          options: { durable: true },
        },
      ],
      uri: process.env.RABBITMQ_URI,
    }),
  ],
})
export class MessagingModule {}
```

## Implementation Patterns

### Trip Service

**Responsibilities:**
- Create and manage trips
- Enforce trip lifecycle
- Publish trip events
- Handle assignments

**Implementation:**

```typescript
@Injectable()
export class TripService {
  constructor(
    private readonly tripRepository: TripRepository,
    private readonly eventPublisher: EventPublisher,
    private readonly driverClient: DriverServiceClient,
  ) {}

  async createTrip(createTripDTO: CreateTripDTO): Promise<Trip> {
    // Create trip
    const trip = this.tripRepository.create({
      ...createTripDTO,
      status: 'created',
    });
    const savedTrip = await this.tripRepository.save(trip);

    // Publish event
    await this.eventPublisher.publish('trip.created', {
      tripId: savedTrip.id,
      origin: savedTrip.origin,
      destination: savedTrip.destination,
      createdAt: savedTrip.createdAt,
    });

    return savedTrip;
  }

  async assignTrip(tripId: string, driverId: string, vehicleId: string): Promise<Trip> {
    const trip = await this.tripRepository.findOne({ where: { id: tripId } });

    if (!trip || trip.status !== 'created') {
      throw new BadRequestException('Trip cannot be assigned');
    }

    // Update trip with assignment
    trip.status = 'assigned';
    trip.driverId = driverId;
    trip.vehicleId = vehicleId;
    trip.assignedAt = new Date();

    const updatedTrip = await this.tripRepository.save(trip);

    // Publish assignment event
    await this.eventPublisher.publish('trip.assigned', {
      tripId: updatedTrip.id,
      driverId,
      vehicleId,
      assignedAt: updatedTrip.assignedAt,
    });

    return updatedTrip;
  }

  async startTrip(tripId: string): Promise<Trip> {
    const trip = await this.tripRepository.findOne({ where: { id: tripId } });

    if (!trip || trip.status !== 'assigned') {
      throw new BadRequestException('Trip cannot be started');
    }

    // Start trip
    trip.status = 'started';
    trip.startedAt = new Date();
    const updatedTrip = await this.tripRepository.save(trip);

    // Publish event
    await this.eventPublisher.publish('trip.started', {
      tripId: updatedTrip.id,
      driverId: updatedTrip.driverId,
      vehicleId: updatedTrip.vehicleId,
      startedAt: updatedTrip.startedAt,
    });

    return updatedTrip;
  }

  async completeTrip(tripId: string): Promise<Trip> {
    const trip = await this.tripRepository.findOne({ where: { id: tripId } });

    if (!trip || trip.status !== 'in_progress') {
      throw new BadRequestException('Trip cannot be completed');
    }

    // Complete trip
    trip.status = 'completed';
    trip.completedAt = new Date();
    const updatedTrip = await this.tripRepository.save(trip);

    // Publish completion event
    await this.eventPublisher.publish('trip.completed', {
      tripId: updatedTrip.id,
      driverId: updatedTrip.driverId,
      vehicleId: updatedTrip.vehicleId,
      completedAt: updatedTrip.completedAt,
      distanceKm: updatedTrip.distanceKm,
      durationMinutes: updatedTrip.durationMinutes,
    });

    // Update driver availability (async)
    await this.eventPublisher.publish('driver.available', {
      driverId: updatedTrip.driverId,
    });

    return updatedTrip;
  }
}
```

### Driver Service

**Responsibilities:**
- Manage driver profiles
- Track driver availability
- Publish driver events
- Handle status changes

**Implementation:**

```typescript
@Injectable()
export class DriverService {
  constructor(
    private readonly driverRepository: DriverRepository,
    private readonly eventPublisher: EventPublisher,
  ) {}

  async updateDriverStatus(driverId: string, status: DriverStatus): Promise<Driver> {
    const driver = await this.driverRepository.findOne({
      where: { id: driverId },
    });

    if (!driver) {
      throw new NotFoundException('Driver not found');
    }

    const previousStatus = driver.status;
    driver.status = status;
    driver.statusChangedAt = new Date();

    const updated = await this.driverRepository.save(driver);

    // Publish status change event
    await this.eventPublisher.publish('driver.status_changed', {
      driverId: updated.id,
      previousStatus,
      newStatus: status,
      changedAt: updated.statusChangedAt,
    });

    // Publish availability change
    if (status === 'available') {
      await this.eventPublisher.publish('driver.available', {
        driverId: updated.id,
        latitude: updated.currentLatitude,
        longitude: updated.currentLongitude,
      });
    }

    return updated;
  }

  async getAvailableDrivers(): Promise<Driver[]> {
    return this.driverRepository.find({
      where: { status: 'available' },
      order: { rating: 'DESC' },
    });
  }
}

@Controller('drivers')
@ApiTags('drivers')
export class DriverController {
  constructor(private readonly driverService: DriverService) {}

  @Patch(':id/status')
  async updateStatus(
    @Param('id') driverId: string,
    @Body() { status }: { status: DriverStatus },
  ): Promise<Driver> {
    return this.driverService.updateDriverStatus(driverId, status);
  }

  @Get('available')
  async getAvailableDrivers(): Promise<Driver[]> {
    return this.driverService.getAvailableDrivers();
  }
}
```

### Route Service

**Responsibilities:**
- Calculate optimal routes
- Estimate distance and duration
- Provide route optimization
- Handle route requests from Trip Service

**Implementation:**

```typescript
@Injectable()
export class RouteService {
  constructor(
    private readonly routeRepository: RouteRepository,
    private readonly mapClient: MapServiceClient,
    private readonly eventPublisher: EventPublisher,
  ) {}

  async calculateRoute(
    origin: Location,
    destination: Location,
    waypoints?: Location[],
  ): Promise<Route> {
    // Call external mapping service (Google Maps, OSRM, etc.)
    const routeData = await this.mapClient.getDirections({
      origin,
      destination,
      waypoints,
    });

    // Create route entity
    const route = this.routeRepository.create({
      origin,
      destination,
      waypoints,
      distance: routeData.distance,
      duration: routeData.duration,
      polyline: routeData.polyline,
      isOptimized: false,
    });

    const savedRoute = await this.routeRepository.save(route);

    // Publish event
    await this.eventPublisher.publish('route.calculated', {
      routeId: savedRoute.id,
      origin,
      destination,
      distanceKm: savedRoute.distance,
      durationMinutes: savedRoute.duration,
    });

    return savedRoute;
  }

  async optimizeRoute(routeId: string, stops: Location[]): Promise<Route> {
    const route = await this.routeRepository.findOne({
      where: { id: routeId },
    });

    // Use optimization algorithm
    const optimizedOrder = await this.mapClient.optimizeWaypoints({
      origin: route.origin,
      destination: route.destination,
      waypoints: stops,
    });

    route.waypoints = optimizedOrder.waypoints;
    route.isOptimized = true;

    const updated = await this.routeRepository.save(route);

    // Publish event
    await this.eventPublisher.publish('route.optimized', {
      routeId: updated.id,
      newOrder: optimizedOrder.waypoints,
    });

    return updated;
  }

  async estimateRoute(origin: Location, destination: Location): Promise<RouteEstimate> {
    const routeData = await this.mapClient.getDirections({
      origin,
      destination,
    });

    return {
      origin,
      destination,
      estimatedDistance: routeData.distance,
      estimatedDuration: routeData.duration,
      estimatedFare: this.calculateFare(routeData.distance),
    };
  }

  private calculateFare(distanceKm: number): number {
    const baseFare = 5;
    const perKmRate = 1.5;
    return baseFare + (distanceKm * perKmRate);
  }
}

@Controller('routes')
@ApiTags('routes')
export class RouteController {
  constructor(private readonly routeService: RouteService) {}

  @Post('calculate')
  async calculateRoute(
    @Body() { origin, destination, waypoints }: CalculateRouteDTO,
  ): Promise<Route> {
    return this.routeService.calculateRoute(origin, destination, waypoints);
  }

  @Post(':id/optimize')
  async optimizeRoute(
    @Param('id') routeId: string,
    @Body() { stops }: { stops: Location[] },
  ): Promise<Route> {
    return this.routeService.optimizeRoute(routeId, stops);
  }

  @Post('estimate')
  async estimateRoute(
    @Body() { origin, destination }: EstimateRouteDTO,
  ): Promise<RouteEstimate> {
    return this.routeService.estimateRoute(origin, destination);
  }
}
```

### Notification Service

**Responsibilities:**
- Consume events from other services
- Send notifications
- Manage notification templates
- Track delivery status

**Implementation:**

```typescript
@Injectable()
export class NotificationService {
  constructor(
    private readonly notificationRepository: NotificationRepository,
    private readonly smsProvider: SmsProvider,
    private readonly emailProvider: EmailProvider,
    private readonly templateService: TemplateService,
  ) {}

  async sendTripAssignmentNotification(
    event: TripAssignedEvent,
  ): Promise<void> {
    const driver = await this.getDriver(event.driverId);

    const template = await this.templateService.getTemplate(
      'trip_assigned',
    );

    const message = this.templateService.render(template, {
      driverName: driver.firstName,
      pickupAddress: event.pickupAddress,
      dropoffAddress: event.dropoffAddress,
    });

    // Send SMS
    const notification = this.notificationRepository.create({
      driverId: event.driverId,
      tripId: event.tripId,
      type: 'sms',
      message,
      status: 'pending',
    });

    await this.notificationRepository.save(notification);

    try {
      await this.smsProvider.send({
        to: driver.phone,
        body: message,
      });

      notification.status = 'sent';
      notification.sentAt = new Date();
    } catch (error) {
      notification.status = 'failed';
      notification.error = error.message;
    }

    await this.notificationRepository.save(notification);
  }

  async sendTripCompletionNotification(
    event: TripCompletedEvent,
  ): Promise<void> {
    const driver = await this.getDriver(event.driverId);

    const template = await this.templateService.getTemplate(
      'trip_completed',
    );

    const message = this.templateService.render(template, {
      driverName: driver.firstName,
      fare: event.fare,
      rating: event.rating,
    });

    // Send email
    const notification = this.notificationRepository.create({
      driverId: event.driverId,
      tripId: event.tripId,
      type: 'email',
      message,
      status: 'pending',
    });

    await this.notificationRepository.save(notification);

    try {
      await this.emailProvider.send({
        to: driver.email,
        subject: 'Trip Completed',
        body: message,
      });

      notification.status = 'sent';
      notification.sentAt = new Date();
    } catch (error) {
      notification.status = 'failed';
      notification.error = error.message;
    }

    await this.notificationRepository.save(notification);
  }

  async subscribeToEvents() {
    // Subscribe to trip events
    this.eventBus.subscribe('trip.assigned', (event) => {
      this.sendTripAssignmentNotification(event);
    });

    this.eventBus.subscribe('trip.completed', (event) => {
      this.sendTripCompletionNotification(event);
    });
  }
}

// In app.module.ts
@Module({
  providers: [NotificationService],
})
export class NotificationModule {
  constructor(private readonly notificationService: NotificationService) {
    this.notificationService.subscribeToEvents();
  }
}
```

## Anti-Patterns

### Tight Coupling Between Services

**DON'T:**
```typescript
// Trip Service directly calls Driver Service
@Injectable()
export class TripService {
  constructor(private readonly driverService: DriverService) {}

  async assignTrip(tripId: string, driverId: string) {
    // Direct synchronous call
    const driver = await this.driverService.getDriver(driverId);
    // Tight coupling, cannot work independently
  }
}
```

**DO:**
```typescript
// Trip Service publishes event, Driver Service subscribes
@Injectable()
export class TripService {
  constructor(private readonly eventPublisher: EventPublisher) {}

  async assignTrip(tripId: string, driverId: string) {
    const trip = await this.tripRepository.save(assignment);

    // Publish event, driver service reacts independently
    await this.eventPublisher.publish('trip.assigned', {
      tripId,
      driverId,
    });
  }
}
```

### Shared Database

**DON'T:**
```typescript
// Multiple services accessing same database
// Trip Service
SELECT * FROM trips WHERE driver_id = ?

// Driver Service
UPDATE drivers SET status = 'on_trip'

// Both access same 'drivers' table
```

**DO:**
```typescript
// Each service owns its data
// Trip Service database
- trips
- trip_events
- trip_assignments

// Driver Service database
- drivers
- driver_status_history
- driver_documents

// Services call each other through APIs
```

### Synchronous Service Chains

**DON'T:**
```typescript
// Long chains of synchronous calls
Trip.create() → Driver.assign() → Vehicle.assign() → Notification.send()

// If any service is slow/down, entire chain blocks
```

**DO:**
```typescript
// Event-driven async chain
Trip.created (event) →
  Driver service reacts independently →
  Vehicle service reacts independently →
  Notification service reacts independently
```

### Lost Updates

**DON'T:**
```typescript
// Two services reading and writing same entity
const trip = await tripService.getTrip(id); // status: 'created'
const trip2 = await tripService.getTrip(id); // status: 'created'

await tripService.assignTrip(trip.id, driver1); // status: 'assigned'
await tripService.assignTrip(trip2.id, driver2); // OVERWRITE!

// trip is assigned to driver2, driver1 assignment lost
```

**DO:**
```typescript
// Use event sourcing or optimistic locking
const trip = await tripService.getTrip(id, version: 1);

try {
  await tripService.assignTrip(id, driver1, version: 1);
  // Updates version to 2
} catch {
  // Retry or handle conflict
}

// Concurrent update will fail with version mismatch
```

## Performance Guidelines

### Event Batching

```typescript
@Injectable()
export class EventPublisher {
  private eventQueue: DomainEvent[] = [];
  private batchSize = 100;
  private flushInterval = 5000; // 5 seconds

  async publish(event: DomainEvent) {
    this.eventQueue.push(event);

    if (this.eventQueue.length >= this.batchSize) {
      await this.flush();
    }
  }

  private async flush() {
    if (this.eventQueue.length === 0) return;

    const batch = this.eventQueue.splice(0, this.batchSize);
    await this.messageQueue.publishBatch(batch);
  }
}
```

### Service-to-Service Caching

```typescript
@Injectable()
export class DriverServiceClient {
  constructor(private readonly cache: CacheService) {}

  async getDriver(id: string): Promise<Driver> {
    // Check cache first
    const cached = await this.cache.get(`driver:${id}`);
    if (cached) return cached;

    // Call service
    const driver = await this.http.get(`${process.env.DRIVER_SERVICE_URL}/drivers/${id}`);

    // Cache for 5 minutes
    await this.cache.set(`driver:${id}`, driver, 300000);

    return driver;
  }
}
```

### Circuit Breaker Pattern

```typescript
@Injectable()
export class CircuitBreaker {
  private failureCount = 0;
  private lastFailureTime: Date;
  private state: 'closed' | 'open' | 'half-open' = 'closed';

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === 'open') {
      if (Date.now() - this.lastFailureTime.getTime() > 60000) {
        this.state = 'half-open';
      } else {
        throw new Error('Circuit breaker is open');
      }
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  private onSuccess() {
    this.failureCount = 0;
    this.state = 'closed';
  }

  private onFailure() {
    this.failureCount++;
    this.lastFailureTime = new Date();

    if (this.failureCount >= 5) {
      this.state = 'open';
    }
  }
}
```

## Observability

### Distributed Tracing

```typescript
@Injectable()
export class TracingService {
  constructor(private readonly tracer: Tracer) {}

  createSpan(name: string, parentSpan?: Span): Span {
    return this.tracer.startSpan(name, {
      childOf: parentSpan,
    });
  }

  async traceAsync<T>(
    name: string,
    fn: (span: Span) => Promise<T>,
  ): Promise<T> {
    const span = this.createSpan(name);

    try {
      return await fn(span);
    } catch (error) {
      span.setTag('error', true);
      span.log({ event: 'error', message: error.message });
      throw error;
    } finally {
      span.finish();
    }
  }
}
```

### Cross-Service Correlation

```typescript
@Injectable()
export class CorrelationMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    const correlationId = req.headers['x-correlation-id'] || generateId();
    req['correlationId'] = correlationId;
    res.setHeader('x-correlation-id', correlationId);
    next();
  }
}

@Injectable()
export class CorrelationService {
  async publishEvent(event: any, correlationId: string) {
    await this.eventBus.publish({
      ...event,
      correlationId,
      timestamp: new Date(),
    });
  }
}
```

## Testing Strategy

### Contract Testing

```typescript
describe('Trip Service → Driver Service Contract', () => {
  let tripService: TripService;
  let driverClient: DriverServiceClient;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        TripService,
        {
          provide: DriverServiceClient,
          useValue: {
            getDriver: jest.fn().mockResolvedValue({
              id: 'driver-123',
              status: 'available',
            }),
          },
        },
      ],
    }).compile();

    tripService = module.get<TripService>(TripService);
    driverClient = module.get<DriverServiceClient>(DriverServiceClient);
  });

  test('Trip Service expects Driver Service to return driver with status field', async () => {
    const driver = await driverClient.getDriver('driver-123');

    expect(driver).toHaveProperty('id');
    expect(driver).toHaveProperty('status');
    expect(['available', 'on_trip', 'off_duty']).toContain(driver.status);
  });
});
```

### Event Testing

```typescript
describe('Event Publishing', () => {
  let tripService: TripService;
  let eventPublisher: EventPublisher;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        TripService,
        {
          provide: EventPublisher,
          useValue: {
            publish: jest.fn(),
          },
        },
      ],
    }).compile();

    tripService = module.get<TripService>(TripService);
    eventPublisher = module.get<EventPublisher>(EventPublisher);
  });

  test('should publish trip.assigned event when trip is assigned', async () => {
    const tripId = 'trip-123';
    const driverId = 'driver-123';

    await tripService.assignTrip(tripId, driverId);

    expect(eventPublisher.publish).toHaveBeenCalledWith(
      'trip.assigned',
      expect.objectContaining({
        tripId,
        driverId,
      }),
    );
  });
});
```
