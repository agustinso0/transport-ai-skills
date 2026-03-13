---
name: transport-system-architecture
description: Architecture patterns and domain boundaries for transportation management systems
---

# Transport System Architecture

A comprehensive guide for architecting scalable, maintainable transportation management platforms with clear domain boundaries and service separation.

# Description

This skill provides architectural patterns specifically designed for transportation management systems. It covers domain-driven design principles, module boundaries, data ownership, and strategies for building systems that manage drivers, vehicles, routes, trips, and real-time operations. The guidance helps architects make informed decisions about monolithic vs microservices architectures while maintaining clear separation of concerns.

# When to Activate

Activate this skill when:

- Designing a new transportation management platform from scratch
- Refactoring an existing transport system that has unclear boundaries
- Making architectural decisions about service separation
- Evaluating modular monolith vs microservices approaches
- Defining domain models for drivers, vehicles, routes, and trips
- Establishing communication patterns between transport modules
- Planning data ownership and database schema organization
- Scaling a transportation platform beyond a single deployment

# Engineering Principles

## Domain-Driven Design

**Bounded Contexts**: Transportation systems have natural boundaries:
- Driver Management Context: driver profiles, availability, credentials, documents
- Vehicle Management Context: vehicle fleet, maintenance, inspections, capacity
- Route Context: route planning, optimization, waypoints, geographical data
- Trip Context: trip lifecycle, assignments, status transitions, history
- Dispatch Context: matching, assignment logic, queue management
- Tracking Context: real-time location, telemetry, trip progress
- Notification Context: alerts, messaging, push notifications

**Ubiquitous Language**: Use consistent terminology across all contexts:
- Use "Trip" not "Ride" or "Journey" interchangeably
- Use "Dispatch" not "Assignment" when referring to the matching process
- Use "Driver" for the person, "Vehicle" for the asset
- Use "Route" for planned paths, "Trip" for actual executions

## Single Responsibility

Each module should have ONE reason to change:
- Driver module changes when driver management rules change
- Vehicle module changes when fleet management needs evolve
- Trip module changes when trip lifecycle requirements change
- Route module changes when routing algorithms improve

## Data Ownership

**Golden Rule**: Each bounded context owns its data exclusively.

- Driver Context owns: drivers table
- Vehicle Context owns: vehicles table
- Route Context owns: routes, waypoints tables
- Trip Context owns: trips, trip_events tables
- Dispatch Context: May be stateless or own dispatch_queue table

**No Shared Tables**: Never have multiple contexts writing to the same table.

## Event-Driven Communication

For looseness coupling between contexts:
- Trip completed → Trigger driver availability update
- Vehicle maintenance scheduled → Trigger vehicle unavailability
- Driver assigned to trip → Trigger notification
- Trip location updated → Trigger tracking updates

# Domain Modeling

## Core Entities

### Driver Domain

```
Driver (Aggregate Root)
├── id: uuid
├── user_id: uuid (link to auth)
├── status: enum (active, inactive, suspended)
├── availability_status: enum (available, on_trip, offline)
├── license_number: string
├── license_expiry: date
├── rating: decimal
├── total_trips: integer
└── created_at: timestamp

Driver Documents
├── id: uuid
├── driver_id: uuid
├── document_type: enum
├── file_url: string
├── verified: boolean
└── expiry_date: date
```

### Vehicle Domain

```
Vehicle (Aggregate Root)
├── id: uuid
├── license_plate: string
├── make: string
├── model: string
├── year: integer
├── capacity: integer
├── vehicle_type: enum (sedan, suv, van, truck)
├── status: enum (active, maintenance, retired)
├── current_driver_id: uuid (nullable)
└── created_at: timestamp

Vehicle Maintenance
├── id: uuid
├── vehicle_id: uuid
├── maintenance_type: string
├── scheduled_date: timestamp
├── completed_date: timestamp
├── notes: text
└── cost: decimal
```

### Route Domain

```
Route (Aggregate Root)
├── id: uuid
├── name: string
├── origin: geography (point)
├── destination: geography (point)
├── estimated_distance: decimal
├── estimated_duration: integer
└── created_at: timestamp

Waypoint
├── id: uuid
├── route_id: uuid
├── sequence: integer
├── location: geography (point)
├── stop_duration: integer
└── instructions: text
```

### Trip Domain

```
Trip (Aggregate Root)
├── id: uuid
├── route_id: uuid
├── driver_id: uuid
├── vehicle_id: uuid
├── status: enum (pending, assigned, in_progress, completed, cancelled)
├── scheduled_start: timestamp
├── actual_start: timestamp
├── actual_end: timestamp
├── origin: geography (point)
├── destination: geography (point)
├── distance_traveled: decimal
└── created_at: timestamp

Trip Event
├── id: uuid
├── trip_id: uuid
├── event_type: enum (created, assigned, started, arrived, completed, cancelled)
├── occurred_at: timestamp
├── metadata: jsonb
└── recorded_by: uuid
```

## Aggregate Boundaries

**Rule**: Modifications to aggregates must go through the aggregate root.

- To update a driver's documents → Go through Driver aggregate
- To add a waypoint → Go through Route aggregate
- To record trip progress → Go through Trip aggregate
- Never directly modify child entities without the parent aggregate

# Architecture Guidelines

## Modular Monolith Approach (Recommended for Most Cases)

**Structure**:
```
src/
├── modules/
│   ├── drivers/
│   │   ├── drivers.service.ts
│   │   ├── drivers.repository.ts
│   │   ├── drivers.types.ts
│   │   └── drivers.routes.ts
│   ├── vehicles/
│   │   ├── vehicles.service.ts
│   │   ├── vehicles.repository.ts
│   │   ├── vehicles.types.ts
│   │   └── vehicles.routes.ts
│   ├── routes/
│   ├── trips/
│   ├── dispatch/
│   ├── tracking/
│   └── notifications/
├── shared/
│   ├── database/
│   ├── events/
│   └── types/
└── api/
```

**Benefits**:
- Single deployment
- Easier development and debugging
- Simpler data queries
- Lower operational complexity
- Can be split later if needed

**Rules**:
- Modules communicate via service interfaces, never direct database access
- Each module has its own service layer
- Shared code goes in `/shared`, not between modules
- Use TypeScript path aliases to prevent circular dependencies

## When to Consider Microservices

Split into microservices only when:
- You have distinct teams owning different domains
- Different modules have vastly different scaling requirements
- You need to deploy modules independently
- Technology stack needs differ by domain

**Services to Extract First** (in order):
1. Tracking Service (highest real-time load, different scaling needs)
2. Notification Service (independent, can use message queue)
3. Dispatch Service (complex algorithms, might need special infrastructure)
4. Driver/Vehicle/Trip Services (later, if organizational boundaries justify it)

## Database Organization

### Option 1: Single Database with Schema Separation (Recommended)

```sql
CREATE SCHEMA drivers;
CREATE SCHEMA vehicles;
CREATE SCHEMA routes;
CREATE SCHEMA trips;
CREATE SCHEMA dispatch;
CREATE SCHEMA tracking;
```

**Benefits**:
- Referential integrity across schemas
- Simpler transactions
- Easier queries across domains
- Lower operational overhead

**Rules**:
- Each module only accesses its own schema
- Cross-schema queries only via service layer
- Use views for read-only cross-schema access

### Option 2: Separate Databases per Domain

Only when true service separation is needed.

**Challenges**:
- No foreign keys across databases
- Distributed transactions complexity
- Data consistency is harder
- More infrastructure to manage

## Communication Patterns

### Synchronous (REST/GraphQL)

Use for:
- Read queries (get driver info, list trips)
- Commands that need immediate response
- User-initiated operations

### Asynchronous (Events)

Use for:
- Cross-domain notifications
- Operations that don't need immediate response
- Eventual consistency scenarios

**Event Examples**:
```typescript
// Trip completed
{
  event: 'trip.completed',
  tripId: 'uuid',
  driverId: 'uuid',
  vehicleId: 'uuid',
  completedAt: 'timestamp'
}

// Driver availability changed
{
  event: 'driver.availability_changed',
  driverId: 'uuid',
  previousStatus: 'on_trip',
  newStatus: 'available'
}
```

# Implementation Patterns

## Module Interface Pattern

Each module exposes a clean service interface:

```typescript
// modules/drivers/drivers.service.ts
export class DriversService {
  async getDriver(id: string): Promise<Driver | null> { }

  async updateDriverAvailability(
    id: string,
    status: AvailabilityStatus
  ): Promise<void> { }

  async getAvailableDrivers(
    location: GeoPoint,
    radius: number
  ): Promise<Driver[]> { }
}

// Other modules use this service, never access database directly
import { driversService } from '@/modules/drivers';

const driver = await driversService.getDriver(driverId);
```

## Repository Pattern

Isolate database access:

```typescript
// modules/trips/trips.repository.ts
export class TripsRepository {
  async create(trip: CreateTripData): Promise<Trip> {
    const { data, error } = await supabase
      .from('trips.trips')
      .insert(trip)
      .select()
      .single();

    if (error) throw new Error(error.message);
    return data;
  }

  async findById(id: string): Promise<Trip | null> {
    const { data } = await supabase
      .from('trips.trips')
      .select('*')
      .eq('id', id)
      .maybeSingle();

    return data;
  }
}
```

## Event Bus Pattern

Decouple modules with events:

```typescript
// shared/events/event-bus.ts
export class EventBus {
  private handlers: Map<string, Array<(data: any) => void>> = new Map();

  on(event: string, handler: (data: any) => void): void {
    if (!this.handlers.has(event)) {
      this.handlers.set(event, []);
    }
    this.handlers.get(event)!.push(handler);
  }

  async emit(event: string, data: any): Promise<void> {
    const handlers = this.handlers.get(event) || [];
    await Promise.all(handlers.map(h => h(data)));
  }
}

// Usage in modules
eventBus.on('trip.completed', async (data) => {
  await driversService.updateDriverAvailability(
    data.driverId,
    'available'
  );
});
```

## State Machine Pattern for Trip Lifecycle

```typescript
type TripStatus =
  | 'pending'
  | 'assigned'
  | 'in_progress'
  | 'completed'
  | 'cancelled';

type TripEvent =
  | 'assign'
  | 'start'
  | 'complete'
  | 'cancel';

const VALID_TRANSITIONS: Record<TripStatus, TripEvent[]> = {
  pending: ['assign', 'cancel'],
  assigned: ['start', 'cancel'],
  in_progress: ['complete', 'cancel'],
  completed: [],
  cancelled: []
};

export class TripStateMachine {
  canTransition(current: TripStatus, event: TripEvent): boolean {
    return VALID_TRANSITIONS[current]?.includes(event) ?? false;
  }

  transition(current: TripStatus, event: TripEvent): TripStatus {
    if (!this.canTransition(current, event)) {
      throw new Error(
        `Invalid transition: ${current} -> ${event}`
      );
    }

    switch (event) {
      case 'assign': return 'assigned';
      case 'start': return 'in_progress';
      case 'complete': return 'completed';
      case 'cancel': return 'cancelled';
    }
  }
}
```

# Anti-Patterns

## Shared Database Tables Between Modules

**NEVER DO THIS**:
```typescript
// ❌ Both drivers and trips modules writing to same table
// trips module
await supabase.from('driver_availability').update({ available: false });

// drivers module
await supabase.from('driver_availability').update({ available: true });
```

**DO THIS**:
```typescript
// ✅ Trips module calls drivers service
await driversService.updateAvailability(driverId, 'unavailable');
```

## Circular Dependencies Between Modules

**NEVER DO THIS**:
```typescript
// ❌ drivers.service.ts
import { tripsService } from '@/modules/trips';

// ❌ trips.service.ts
import { driversService } from '@/modules/drivers';
```

**DO THIS**:
```typescript
// ✅ Use events for cross-module communication
eventBus.emit('driver.availability_changed', { driverId, status });
```

## God Service Anti-Pattern

**NEVER DO THIS**:
```typescript
// ❌ One service doing everything
class TransportService {
  createDriver() { }
  createVehicle() { }
  createTrip() { }
  assignDriver() { }
  optimizeRoute() { }
  sendNotification() { }
  // ... 50 more methods
}
```

**DO THIS**:
```typescript
// ✅ Separate services by domain
class DriversService { }
class VehiclesService { }
class TripsService { }
class DispatchService { }
class NotificationService { }
```

## Anemic Domain Model

**NEVER DO THIS**:
```typescript
// ❌ No business logic in entities
interface Trip {
  id: string;
  status: string;
  driverId: string;
}

// Logic scattered in services
function canCancelTrip(trip: Trip): boolean {
  return trip.status === 'pending' || trip.status === 'assigned';
}
```

**DO THIS**:
```typescript
// ✅ Business logic in domain entities
class Trip {
  constructor(
    public id: string,
    public status: TripStatus,
    public driverId: string
  ) {}

  canCancel(): boolean {
    return this.status === 'pending' || this.status === 'assigned';
  }

  cancel(): void {
    if (!this.canCancel()) {
      throw new Error('Trip cannot be cancelled in current status');
    }
    this.status = 'cancelled';
  }
}
```

## Direct Database Access from Routes

**NEVER DO THIS**:
```typescript
// ❌ API route directly accessing database
app.get('/trips/:id', async (req, res) => {
  const { data } = await supabase
    .from('trips')
    .select('*')
    .eq('id', req.params.id);
  res.json(data);
});
```

**DO THIS**:
```typescript
// ✅ Route uses service layer
app.get('/trips/:id', async (req, res) => {
  const trip = await tripsService.getTrip(req.params.id);
  res.json(trip);
});
```

# Observability

## Metrics to Track

**Driver Metrics**:
- Active drivers count
- Available drivers count
- Average driver rating
- Driver utilization rate

**Vehicle Metrics**:
- Active vehicles count
- Vehicles in maintenance
- Average vehicle age
- Vehicle utilization rate

**Trip Metrics**:
- Total trips by status
- Average trip duration
- Average distance per trip
- Trip completion rate
- Trip cancellation rate by status

**Dispatch Metrics**:
- Average time to assignment
- Failed dispatch attempts
- Driver acceptance rate

**System Metrics**:
- API response times by endpoint
- Database query performance
- Event processing latency
- Error rates by module

## Logging Strategy

**Structured Logging**:
```typescript
logger.info('Trip assigned', {
  tripId: trip.id,
  driverId: driver.id,
  vehicleId: vehicle.id,
  dispatchLatency: 1200,
  module: 'dispatch'
});
```

**Key Events to Log**:
- Trip state transitions
- Driver availability changes
- Vehicle assignments
- Dispatch decisions
- Failed operations with context

## Distributed Tracing

Track requests across module boundaries:
```typescript
// Propagate trace context
const traceId = generateTraceId();

logger.info('Creating trip', { traceId, module: 'trips' });
await driversService.assignDriver(driverId, { traceId });
logger.info('Driver assigned', { traceId, module: 'dispatch' });
```

# Testing Strategy

## Unit Tests

Test individual components in isolation:

```typescript
describe('TripStateMachine', () => {
  it('should allow transitioning from pending to assigned', () => {
    const machine = new TripStateMachine();
    expect(machine.canTransition('pending', 'assign')).toBe(true);
  });

  it('should not allow transitioning from completed to started', () => {
    const machine = new TripStateMachine();
    expect(machine.canTransition('completed', 'start')).toBe(false);
  });
});
```

## Integration Tests

Test module interactions:

```typescript
describe('Trip Assignment Flow', () => {
  it('should update driver availability when trip is assigned', async () => {
    const trip = await tripsService.createTrip(tripData);
    const driver = await driversService.getAvailableDrivers()[0];

    await dispatchService.assignTrip(trip.id, driver.id);

    const updatedDriver = await driversService.getDriver(driver.id);
    expect(updatedDriver.availabilityStatus).toBe('on_trip');
  });
});
```

## Contract Tests

Ensure module interfaces remain stable:

```typescript
describe('DriversService Contract', () => {
  it('should expose getDriver method with correct signature', () => {
    expect(typeof driversService.getDriver).toBe('function');
    expect(driversService.getDriver.length).toBe(1);
  });

  it('should return Driver or null', async () => {
    const result = await driversService.getDriver('test-id');
    expect(result === null || 'id' in result).toBe(true);
  });
});
```

## End-to-End Tests

Test complete workflows:

```typescript
describe('Complete Trip Workflow', () => {
  it('should handle full trip lifecycle', async () => {
    // Create trip
    const trip = await api.post('/trips', tripData);

    // Assign driver
    await api.post(`/trips/${trip.id}/assign`, { driverId });

    // Start trip
    await api.post(`/trips/${trip.id}/start`);

    // Complete trip
    await api.post(`/trips/${trip.id}/complete`);

    const finalTrip = await api.get(`/trips/${trip.id}`);
    expect(finalTrip.status).toBe('completed');
  });
});
```

# Code Review Checklist

## Architecture Review

- [ ] Are module boundaries clear and well-defined?
- [ ] Does each module have a single, well-defined responsibility?
- [ ] Are there any circular dependencies between modules?
- [ ] Is data ownership clearly established?
- [ ] Are shared concerns properly placed in `/shared`?

## Domain Modeling Review

- [ ] Are aggregates properly defined with clear roots?
- [ ] Do entities contain business logic, not just data?
- [ ] Are value objects used where appropriate?
- [ ] Is the ubiquitous language used consistently?
- [ ] Are domain events properly defined?

## Communication Review

- [ ] Do modules communicate through service interfaces?
- [ ] Are events used for loose coupling?
- [ ] Is there any direct database access across module boundaries?
- [ ] Are synchronous calls appropriate, or should they be async?
- [ ] Are event names descriptive and follow naming conventions?

## Data Access Review

- [ ] Is database access isolated in repository layer?
- [ ] Are queries optimized with appropriate indexes?
- [ ] Is connection pooling properly configured?
- [ ] Are transactions used appropriately?
- [ ] Is RLS enabled on all tables?

## State Management Review

- [ ] Are state transitions validated?
- [ ] Is the state machine pattern used for complex lifecycles?
- [ ] Are invalid state transitions prevented?
- [ ] Are state changes logged for audit purposes?

## Testing Review

- [ ] Are unit tests covering business logic?
- [ ] Are integration tests covering module interactions?
- [ ] Are contract tests ensuring interface stability?
- [ ] Are end-to-end tests covering critical workflows?
- [ ] Is test coverage above 80% for business logic?

## Security Review

- [ ] Are authentication checks in place?
- [ ] Are authorization rules properly enforced?
- [ ] Is input validation comprehensive?
- [ ] Are SQL injection vectors eliminated?
- [ ] Are sensitive data fields encrypted?

## Performance Review

- [ ] Are database queries indexed appropriately?
- [ ] Is pagination implemented for list endpoints?
- [ ] Are N+1 query problems avoided?
- [ ] Is caching used where appropriate?
- [ ] Are background jobs used for long-running tasks?

## Observability Review

- [ ] Are key metrics being tracked?
- [ ] Is logging structured and consistent?
- [ ] Are errors logged with sufficient context?
- [ ] Are traces propagated across module boundaries?
- [ ] Are alerts configured for critical failures?
