---
name: transport-domain-expert
description: Domain knowledge for building transportation and fleet management systems including drivers, vehicles, routes, dispatching and trip lifecycle.
---

# Transport Domain Expert

## Description

This skill teaches the AI how transportation management systems work, enabling it to design correct domain models, services, and database schemas for fleet management, dispatching, and trip tracking platforms. It provides comprehensive domain knowledge about the transportation industry including entity relationships, business rules, state machines, and architectural patterns specific to transportation platforms.

## When to Activate

Activate this skill when:

- Designing transportation or fleet management systems
- Implementing backend services for transport platforms
- Modeling transportation databases and schemas
- Creating APIs related to trips, vehicles, drivers, routes, or dispatching
- Implementing dispatching and assignment logic
- Implementing trip lifecycle and state management
- Building real-time vehicle tracking systems
- Designing driver management systems
- Creating route planning and optimization features
- Implementing delivery or ride-sharing platforms

## Domain Overview

Transportation management platforms revolve around coordinating drivers, vehicles, and trips to move people or goods from origin to destination.

### Core Entities

**Driver**
- Represents a person authorized to operate vehicles
- Has status: available, on_trip, off_duty, unavailable
- Has certifications, ratings, and performance metrics
- May be assigned to one or more vehicles
- Can only have one active trip at a time

**Vehicle**
- Represents a transport asset (car, truck, van, etc.)
- Has status: available, in_use, maintenance, out_of_service
- Has type, capacity, and specifications
- Tracks maintenance history and mileage
- May have multiple drivers but one active driver at a time

**Trip**
- Represents a single journey from pickup to destination
- Has lifecycle states: created, assigned, started, in_progress, completed, cancelled
- Contains origin and destination information
- Tracks actual vs. planned route and times
- Records events throughout the journey

**Route**
- Defines the planned path between locations
- Contains waypoints and estimated times
- May be optimized for distance, time, or cost
- Can be static (predefined) or dynamic (calculated on-demand)

**Dispatch**
- Represents the assignment of driver + vehicle to trip
- Contains assignment logic and rules
- Tracks assignment history and changes
- May include priority and urgency levels

**LocationUpdate**
- Real-time position data from vehicles
- Includes coordinates, timestamp, speed, heading
- Used for tracking and ETA calculations
- High-volume data requiring efficient storage

**TripEvent**
- Audit trail of trip state changes
- Records timestamps for each lifecycle event
- Captures pickup time, dropoff time, delays
- Used for analytics and compliance

### Entity Relationships

```
Driver ──(assigned_to)──> Vehicle
Driver ──(performs)──> Trip
Vehicle ──(performs)──> Trip
Trip ──(follows)──> Route
Trip ──(has_many)──> LocationUpdate
Trip ──(has_many)──> TripEvent
Dispatch ──(assigns)──> Driver
Dispatch ──(assigns)──> Vehicle
Dispatch ──(creates)──> Trip
```

### Key Relationships

- A driver can be assigned to multiple vehicles but drives one at a time
- A vehicle can have multiple assigned drivers but one active driver
- A trip belongs to one driver and one vehicle
- A trip follows one route but may deviate
- Location updates belong to a vehicle and optionally a trip
- Trip events track the complete lifecycle of a trip

## Trip Lifecycle

Trips follow a strict state machine to ensure data consistency and business rule enforcement.

### States

**CREATED**
- Initial state when trip is requested
- Has origin, destination, and requirements
- Not yet assigned to driver/vehicle
- Can be cancelled without penalty

**ASSIGNED**
- Driver and vehicle have been allocated
- Driver notified and must accept
- Can be reassigned if driver rejects
- Transition to: started, cancelled

**STARTED**
- Driver accepted and is en route to pickup
- Cannot be unassigned without creating new trip
- Location tracking begins
- Transition to: in_progress, cancelled

**IN_PROGRESS**
- Pickup completed, en route to destination
- Active location tracking required
- Cannot be cancelled without reason
- Transition to: completed, cancelled

**COMPLETED**
- Destination reached, trip finished
- Final state (terminal)
- Triggers billing and rating
- Cannot transition to other states

**CANCELLED**
- Trip terminated before completion
- Final state (terminal)
- Requires cancellation reason
- May trigger cancellation fee

### State Transitions

```
CREATED → ASSIGNED (when driver/vehicle assigned)
ASSIGNED → STARTED (when driver accepts and begins)
ASSIGNED → CANCELLED (if rejected or timed out)
STARTED → IN_PROGRESS (when pickup confirmed)
STARTED → CANCELLED (with penalty/reason)
IN_PROGRESS → COMPLETED (when destination reached)
IN_PROGRESS → CANCELLED (with penalty/reason)
```

### Transition Rules

- All transitions must be recorded as TripEvents
- Cannot skip states (must follow sequence)
- Terminal states (completed, cancelled) are immutable
- Each transition validates business rules before executing
- Failed transitions must rollback and log errors

## Core Business Rules

Transportation platforms must enforce strict business rules to maintain operational integrity.

### Driver Rules

1. A driver can only have ONE active trip at any time
2. A driver must be in "available" status to receive assignments
3. A driver cannot start a trip without vehicle assignment
4. A driver must have valid certifications for vehicle type
5. A driver's working hours must comply with regulations
6. A driver must accept/reject assignment within timeout period

### Vehicle Rules

1. A vehicle cannot be assigned to multiple trips simultaneously
2. A vehicle must be in "available" status to be dispatched
3. A vehicle in "maintenance" status cannot accept trips
4. A vehicle must pass safety checks before trip starts
5. A vehicle's capacity must meet trip requirements
6. A vehicle must have valid insurance and registration

### Trip Rules

1. A trip must have both origin and destination
2. A trip cannot start without driver AND vehicle assignment
3. A trip must follow the defined lifecycle sequence
4. A trip cannot be completed without reaching destination
5. Location updates must be recorded during active trips
6. Trip state changes must be logged as events

### Route Rules

1. A route must have at least origin and destination
2. Waypoints must be in logical sequence
3. Route distance and duration must be calculated
4. Actual route may deviate but must be tracked
5. Route changes during trip must be logged

### Assignment Rules

1. Driver must be within acceptable distance from pickup
2. Vehicle type must match trip requirements
3. Driver rating must meet minimum threshold (if applicable)
4. Vehicle capacity must accommodate trip needs
5. Driver's schedule must have availability
6. Dispatch algorithm must optimize for efficiency

## Service Boundaries

Transportation platforms typically decompose into specialized services with clear boundaries.

### Driver Service
**Responsibilities:**
- Driver registration and profile management
- Status management (available, on_trip, off_duty)
- Certification and document verification
- Performance metrics and ratings
- Working hours and compliance tracking

**Exposes:**
- Driver CRUD operations
- Status updates
- Availability queries
- Performance reports

### Vehicle Service
**Responsibilities:**
- Vehicle registration and specifications
- Status management (available, in_use, maintenance)
- Maintenance scheduling and history
- Capacity and capability tracking
- Insurance and compliance

**Exposes:**
- Vehicle CRUD operations
- Status updates
- Availability queries
- Maintenance records

### Trip Service
**Responsibilities:**
- Trip creation and management
- Lifecycle state management
- Trip history and analytics
- Event logging
- Billing calculation

**Exposes:**
- Trip CRUD operations
- State transitions
- Trip queries and filters
- Event history

### Route Service
**Responsibilities:**
- Route planning and optimization
- Distance and duration calculation
- Waypoint management
- Route deviation detection
- Historical route analysis

**Exposes:**
- Route calculation
- Route optimization
- Distance/duration APIs
- Route history

### Dispatch Service
**Responsibilities:**
- Driver-vehicle-trip matching
- Assignment optimization algorithms
- Assignment notifications
- Reassignment handling
- Queue management

**Exposes:**
- Assignment requests
- Assignment status
- Queue queries
- Optimization controls

### Tracking Service
**Responsibilities:**
- Real-time location ingestion
- Location history storage
- ETA calculation
- Geofencing
- Speed and behavior monitoring

**Exposes:**
- Location update endpoints
- Real-time tracking queries
- ETA calculations
- Location history

## Architecture Guidelines

Transportation platforms require careful architectural decisions based on scale and requirements.

### Deployment Models

**Modular Monolith** (Recommended for small-medium deployments)
- Single deployable with clear module boundaries
- Shared database with logical separation
- Lower operational complexity
- Easier transactions across modules
- Suitable for: startups, small fleets, single-region operations

**Microservices** (Recommended for large platforms)
- Independent deployable services
- Database per service pattern
- Higher operational complexity
- Better scalability and resilience
- Suitable for: large fleets, multi-region, high traffic

### Event-Driven Communication

Use event-driven patterns for:

**Trip Events:**
- trip.created
- trip.assigned
- trip.started
- trip.in_progress
- trip.completed
- trip.cancelled

**Driver Events:**
- driver.status_changed
- driver.assigned
- driver.location_updated

**Vehicle Events:**
- vehicle.status_changed
- vehicle.maintenance_scheduled
- vehicle.location_updated

**Benefits:**
- Loose coupling between services
- Async processing for non-critical operations
- Event sourcing for audit trails
- Easier integration with third-party systems

### Synchronous vs Asynchronous

**Use Synchronous (REST/GraphQL) for:**
- Trip creation (immediate response needed)
- Driver/vehicle queries
- Status updates requiring confirmation
- User-facing operations

**Use Asynchronous (Events/Messages) for:**
- Location updates (high volume)
- Analytics and reporting
- Notifications
- Non-critical background tasks

### Scalability Considerations

**High Volume Data:**
- Location updates: time-series database or partitioned tables
- Trip events: append-only tables with indexes
- Use read replicas for queries

**Real-time Requirements:**
- WebSocket connections for live tracking
- Redis/caching for active trip data
- Message queues for location processing

**Geographic Distribution:**
- Regional databases for latency
- Geo-partitioning for compliance
- CDN for static assets

## Database Modeling

Proper database design is critical for performance and maintainability in transportation systems.

### Recommended Schema

**drivers table**
```sql
id (uuid, primary key)
email (text, unique)
first_name (text)
last_name (text)
phone (text)
status (enum: available, on_trip, off_duty, unavailable)
rating (decimal)
total_trips (integer)
created_at (timestamptz)
updated_at (timestamptz)
```

**vehicles table**
```sql
id (uuid, primary key)
license_plate (text, unique)
make (text)
model (text)
year (integer)
vehicle_type (text)
capacity (integer)
status (enum: available, in_use, maintenance, out_of_service)
current_driver_id (uuid, nullable, FK to drivers)
created_at (timestamptz)
updated_at (timestamptz)
```

**trips table**
```sql
id (uuid, primary key)
driver_id (uuid, FK to drivers)
vehicle_id (uuid, FK to vehicles)
route_id (uuid, nullable, FK to routes)
status (enum: created, assigned, started, in_progress, completed, cancelled)
origin_lat (decimal)
origin_lng (decimal)
origin_address (text)
destination_lat (decimal)
destination_lng (decimal)
destination_address (text)
scheduled_pickup_time (timestamptz, nullable)
actual_pickup_time (timestamptz, nullable)
actual_dropoff_time (timestamptz, nullable)
distance_km (decimal)
duration_minutes (integer)
fare (decimal, nullable)
created_at (timestamptz)
updated_at (timestamptz)
```

**routes table**
```sql
id (uuid, primary key)
name (text)
waypoints (jsonb)
total_distance_km (decimal)
estimated_duration_minutes (integer)
is_optimized (boolean)
created_at (timestamptz)
```

**trip_events table**
```sql
id (uuid, primary key)
trip_id (uuid, FK to trips)
event_type (text)
from_status (text, nullable)
to_status (text)
metadata (jsonb)
created_at (timestamptz)
```

**location_updates table**
```sql
id (uuid, primary key)
vehicle_id (uuid, FK to vehicles)
trip_id (uuid, nullable, FK to trips)
latitude (decimal)
longitude (decimal)
speed_kmh (decimal, nullable)
heading (decimal, nullable)
accuracy_meters (decimal, nullable)
timestamp (timestamptz)
created_at (timestamptz)
```

### Indexing Strategies

**Critical Indexes:**

```sql
-- Active trip queries
CREATE INDEX idx_trips_status ON trips(status) WHERE status IN ('started', 'in_progress');
CREATE INDEX idx_trips_driver_id ON trips(driver_id);
CREATE INDEX idx_trips_vehicle_id ON trips(vehicle_id);
CREATE INDEX idx_trips_created_at ON trips(created_at DESC);

-- Driver availability
CREATE INDEX idx_drivers_status ON drivers(status) WHERE status = 'available';

-- Vehicle availability
CREATE INDEX idx_vehicles_status ON vehicles(status) WHERE status = 'available';

-- Location tracking
CREATE INDEX idx_location_updates_vehicle_trip ON location_updates(vehicle_id, trip_id, timestamp DESC);
CREATE INDEX idx_location_updates_timestamp ON location_updates(timestamp DESC);

-- Trip events audit
CREATE INDEX idx_trip_events_trip_id ON trip_events(trip_id, created_at DESC);
```

**Partitioning for Scale:**

```sql
-- Partition location_updates by time (monthly)
CREATE TABLE location_updates_2024_01 PARTITION OF location_updates
  FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

-- Partition trip_events by time (quarterly)
CREATE TABLE trip_events_2024_q1 PARTITION OF trip_events
  FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');
```

### Data Retention

- Active trips: keep indefinitely
- Completed trips: keep 2-5 years for compliance
- Location updates: keep 90 days for active analysis, archive older
- Trip events: keep indefinitely for audit trails

## Implementation Patterns

Transportation platforms should follow established patterns for maintainability and scalability.

### Backend Architecture Pattern

**Controller → Service → Repository**

```
Controller Layer:
- HTTP request/response handling
- Input validation and DTO mapping
- Authentication and authorization
- NO business logic

Service Layer:
- Business logic implementation
- Transaction management
- Service orchestration
- Domain rule enforcement

Repository Layer:
- Data access abstraction
- Query optimization
- Database operations
- NO business logic
```

### DTO Validation

Always validate input at API boundaries:

```typescript
// Trip creation DTO
interface CreateTripDTO {
  origin: {
    lat: number;    // Required, -90 to 90
    lng: number;    // Required, -180 to 180
    address: string;
  };
  destination: {
    lat: number;
    lng: number;
    address: string;
  };
  scheduledPickupTime?: Date;
  requirements?: {
    vehicleType?: string;
    capacity?: number;
  };
}
```

### Domain Services

Encapsulate complex business logic:

```typescript
// TripAssignmentService
class TripAssignmentService {
  async assignDriverAndVehicle(tripId: string): Promise<Assignment> {
    // 1. Validate trip is in correct state
    // 2. Find available drivers within range
    // 3. Find available vehicles matching requirements
    // 4. Apply assignment algorithm
    // 5. Create assignment
    // 6. Update driver/vehicle status
    // 7. Publish assignment event
    // 8. Return assignment
  }
}
```

### Transaction Management

Use transactions for multi-step operations:

```typescript
async startTrip(tripId: string, driverId: string) {
  return await db.transaction(async (trx) => {
    // 1. Update trip status
    await trx.trips.update(tripId, { status: 'started' });

    // 2. Create trip event
    await trx.tripEvents.create({
      tripId,
      eventType: 'trip_started',
      toStatus: 'started'
    });

    // 3. Update driver status
    await trx.drivers.update(driverId, { status: 'on_trip' });

    // All or nothing
  });
}
```

### Event Publishing

Publish events after successful operations:

```typescript
async completeTrip(tripId: string) {
  const trip = await tripService.complete(tripId);

  // Publish event for other services
  await eventBus.publish('trip.completed', {
    tripId: trip.id,
    driverId: trip.driverId,
    vehicleId: trip.vehicleId,
    completedAt: trip.actualDropoffTime,
    distance: trip.distanceKm,
    duration: trip.durationMinutes
  });
}
```

## Anti-Patterns

Avoid these common mistakes in transportation platform development.

### Business Logic in Controllers

**DON'T:**
```typescript
// Controller with business logic
async createTrip(req, res) {
  const driver = await db.drivers.findById(req.body.driverId);
  if (driver.status !== 'available') {
    return res.error('Driver not available');
  }
  // More business logic...
}
```

**DO:**
```typescript
// Controller delegates to service
async createTrip(req, res) {
  const trip = await tripService.create(req.body);
  return res.json(trip);
}
```

### Shared Database Across Microservices

**DON'T:**
- Multiple services directly accessing same tables
- Creates tight coupling
- Prevents independent scaling

**DO:**
- Database per service
- Services communicate via APIs/events
- Duplicate data where needed (denormalization)

### Synchronous Service Chains

**DON'T:**
```typescript
// Service A calls B calls C calls D (waterfall)
await serviceB.process(await serviceA.start(data));
```

**DO:**
```typescript
// Event-driven async processing
await eventBus.publish('trip.created', data);
// Services react independently
```

### Missing Indexes on High-Volume Tables

**DON'T:**
- Querying location_updates without indexes
- Filtering trips without status index
- Full table scans on large tables

**DO:**
- Index all foreign keys
- Index frequently filtered columns
- Use composite indexes for common queries
- Monitor and optimize slow queries

### Ignoring Concurrent Updates

**DON'T:**
```typescript
// Race condition: multiple dispatchers assign same driver
const driver = await getDriver(id);
if (driver.status === 'available') {
  await assignTrip(driver.id, tripId);
}
```

**DO:**
```typescript
// Optimistic locking with version
await db.drivers.update({
  where: { id, status: 'available', version },
  data: { status: 'on_trip', version: version + 1 }
});
```

### Storing Location Updates Synchronously

**DON'T:**
- Blocking API calls for each location update
- Storing directly in main database

**DO:**
- Queue location updates for async processing
- Use time-series database or partitioned tables
- Batch inserts for efficiency

## Observability

Transportation platforms require comprehensive monitoring and logging.

### Trip Events Logging

Log all trip state transitions:
- Trip created with origin/destination
- Driver/vehicle assigned
- Trip started
- Pickup completed
- Trip completed
- Trip cancelled (with reason)

Include context: trip_id, driver_id, vehicle_id, timestamp, metadata

### Driver Activity Monitoring

Track driver metrics:
- Time spent available vs on_trip
- Trip acceptance rate
- Trip completion rate
- Average rating
- Violations or incidents

Alert on:
- Extended inactive periods
- Low acceptance rates
- Safety violations

### Location Update Tracking

Monitor location data:
- Update frequency per vehicle
- Gaps in location data
- Anomalous movements (speed, direction)
- Geofence violations

Alert on:
- Missing location data
- Vehicle stopped unexpectedly
- Route deviations

### System Health Metrics

Track platform health:
- Active trips count
- Available drivers/vehicles count
- Average assignment time
- API response times
- Database query performance
- Event processing lag

Dashboard views:
- Real-time operations view
- Historical trends
- Geographic heat maps
- Resource utilization

### Error Tracking

Monitor and alert on:
- Failed trip assignments
- State transition errors
- Database errors
- Integration failures
- Timeout errors

Include stack traces, request IDs, and context for debugging.

## Testing Strategy

Comprehensive testing ensures reliability and correctness.

### Unit Tests for Domain Logic

Test business rules in isolation:

```typescript
describe('TripService', () => {
  test('cannot start trip without driver assignment', async () => {
    const trip = createTrip({ status: 'created' });
    await expect(tripService.start(trip.id))
      .rejects.toThrow('Trip must be assigned before starting');
  });

  test('driver can only have one active trip', async () => {
    const driver = createDriver();
    await tripService.create({ driverId: driver.id, status: 'started' });
    await expect(tripService.create({ driverId: driver.id }))
      .rejects.toThrow('Driver already has active trip');
  });
});
```

### Integration Tests for Services

Test service interactions:

```typescript
describe('Trip Assignment Flow', () => {
  test('assigns available driver and vehicle to trip', async () => {
    const driver = await createDriver({ status: 'available' });
    const vehicle = await createVehicle({ status: 'available' });
    const trip = await createTrip({ status: 'created' });

    const assignment = await dispatchService.assign(trip.id);

    expect(assignment.driverId).toBe(driver.id);
    expect(assignment.vehicleId).toBe(vehicle.id);

    const updatedTrip = await tripService.findById(trip.id);
    expect(updatedTrip.status).toBe('assigned');
  });
});
```

### End-to-End Tests for Trip Lifecycle

Test complete user journeys:

```typescript
describe('Complete Trip Lifecycle', () => {
  test('trip from creation to completion', async () => {
    // 1. Create trip
    const trip = await api.post('/trips', tripData);
    expect(trip.status).toBe('created');

    // 2. Assign driver/vehicle
    await api.post(`/trips/${trip.id}/assign`);
    const assigned = await api.get(`/trips/${trip.id}`);
    expect(assigned.status).toBe('assigned');

    // 3. Start trip
    await api.post(`/trips/${trip.id}/start`);
    const started = await api.get(`/trips/${trip.id}`);
    expect(started.status).toBe('started');

    // 4. Mark in progress
    await api.post(`/trips/${trip.id}/pickup`);
    const inProgress = await api.get(`/trips/${trip.id}`);
    expect(inProgress.status).toBe('in_progress');

    // 5. Complete trip
    await api.post(`/trips/${trip.id}/complete`);
    const completed = await api.get(`/trips/${trip.id}`);
    expect(completed.status).toBe('completed');
    expect(completed.actualDropoffTime).toBeDefined();
  });
});
```

### Load Testing

Test under realistic conditions:
- Concurrent trip creation
- Simultaneous location updates
- Burst assignment requests
- High query volume

Monitor:
- Response times
- Error rates
- Database performance
- Resource utilization

## Code Review Checklist

Use this checklist when reviewing transportation platform code.

### Domain Model Correctness

- [ ] Entities match transportation domain requirements
- [ ] Relationships are properly defined (FK constraints)
- [ ] Status enums cover all valid states
- [ ] Required fields are enforced (NOT NULL)
- [ ] Data types are appropriate for scale

### Trip Lifecycle Enforcement

- [ ] State transitions follow defined sequence
- [ ] Terminal states cannot be modified
- [ ] All transitions create trip events
- [ ] Invalid transitions are rejected with clear errors
- [ ] Concurrent updates are handled safely

### Driver Assignment Logic

- [ ] Only available drivers can be assigned
- [ ] Driver can only have one active trip
- [ ] Driver certifications are validated
- [ ] Assignment timeout is enforced
- [ ] Reassignment handling is correct

### Route Consistency

- [ ] Routes have valid origin and destination
- [ ] Waypoints are in logical order
- [ ] Distance and duration are calculated
- [ ] Route deviations are tracked
- [ ] Actual vs planned is recorded

### Database Performance

- [ ] Foreign keys have indexes
- [ ] Frequently filtered columns are indexed
- [ ] High-volume tables use partitioning
- [ ] Queries use EXPLAIN ANALYZE for optimization
- [ ] N+1 queries are avoided

### Security and Authorization

- [ ] RLS policies protect trip data
- [ ] Drivers can only see their trips
- [ ] Admin endpoints require proper authorization
- [ ] Sensitive data is not logged
- [ ] API rate limiting is implemented

### Error Handling

- [ ] Business rule violations return clear errors
- [ ] Database errors are caught and logged
- [ ] Timeout errors are handled gracefully
- [ ] Retry logic for transient failures
- [ ] Error messages are user-friendly

### Testing Coverage

- [ ] Unit tests for business rules
- [ ] Integration tests for service interactions
- [ ] E2E tests for critical flows
- [ ] Edge cases are covered
- [ ] Test data is realistic

### Observability

- [ ] Trip events are logged
- [ ] Errors include context for debugging
- [ ] Metrics are tracked
- [ ] Alerts are configured for failures
- [ ] Dashboards show key metrics

### Code Quality

- [ ] Business logic in service layer
- [ ] Controllers are thin
- [ ] Repositories handle data access only
- [ ] Code follows DRY principle
- [ ] Functions have single responsibility
- [ ] Naming is clear and consistent
- [ ] Comments explain "why" not "what"
