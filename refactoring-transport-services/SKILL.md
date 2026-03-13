---
name: refactoring-transport-services
description: Refactor transport services for better modularity and maintainability
---

# Refactoring Transport Services

# Description

This skill guides the systematic refactoring of transportation platform services to improve code quality, modularity, testability, and maintainability. It focuses on extracting domain logic, eliminating duplication, and establishing clear architectural boundaries.

# When to Activate

- Code has become difficult to understand or modify
- Multiple services contain duplicated logic
- Business logic is tightly coupled to framework code
- Functions exceed 100 lines or have multiple responsibilities
- Adding new features requires changing many files
- Test coverage is low due to tight coupling
- Performance issues from inefficient code organization

# Engineering Principles

1. **Single Responsibility**: Each module has one clear purpose
2. **Domain-Driven Design**: Organize code around business concepts
3. **Dependency Inversion**: Depend on abstractions, not implementations
4. **DRY (Don't Repeat Yourself)**: Extract common logic to shared utilities
5. **Progressive Refactoring**: Make small, incremental improvements
6. **Test Coverage**: Add tests before refactoring critical paths

# Implementation Patterns

## Clean Architecture Structure

```
src/
  domain/
    trip/
      Trip.ts
      TripService.ts
      TripRepository.ts
    driver/
      Driver.ts
      DriverService.ts
      DriverRepository.ts
    location/
      Location.ts
      LocationService.ts
      LocationTracker.ts
  infrastructure/
    database/
      SupabaseClient.ts
      repositories/
        SupabaseTripRepository.ts
        SupabaseDriverRepository.ts
    queue/
      QueueClient.ts
      MessageHandler.ts
  application/
    usecases/
      CreateTrip.ts
      AssignDriver.ts
      UpdateLocation.ts
    dto/
      TripDTO.ts
      DriverDTO.ts
  presentation/
    controllers/
      TripController.ts
      DriverController.ts
    middleware/
      AuthMiddleware.ts
```

## Domain Entity Pattern

```typescript
export class Trip {
  private constructor(
    public readonly id: string,
    public status: TripStatus,
    public pickupLocation: Location,
    public destinationLocation: Location,
    public driverId?: string,
    public vehicleId?: string,
    public fare?: number,
    public createdAt: Date = new Date(),
    public updatedAt: Date = new Date()
  ) {}

  static create(
    pickupLocation: Location,
    destinationLocation: Location
  ): Trip {
    return new Trip(
      crypto.randomUUID(),
      TripStatus.PENDING,
      pickupLocation,
      destinationLocation
    );
  }

  static fromDatabase(data: any): Trip {
    return new Trip(
      data.id,
      data.status,
      Location.fromCoordinates(data.pickup_lat, data.pickup_lng),
      Location.fromCoordinates(data.destination_lat, data.destination_lng),
      data.driver_id,
      data.vehicle_id,
      data.fare,
      new Date(data.created_at),
      new Date(data.updated_at)
    );
  }

  canTransitionTo(newStatus: TripStatus): boolean {
    const validTransitions: Record<TripStatus, TripStatus[]> = {
      [TripStatus.PENDING]: [TripStatus.ASSIGNED, TripStatus.CANCELLED],
      [TripStatus.ASSIGNED]: [TripStatus.ACTIVE, TripStatus.CANCELLED],
      [TripStatus.ACTIVE]: [TripStatus.COMPLETED, TripStatus.CANCELLED],
      [TripStatus.COMPLETED]: [],
      [TripStatus.CANCELLED]: []
    };

    return validTransitions[this.status]?.includes(newStatus) ?? false;
  }

  assignDriver(driverId: string, vehicleId: string): void {
    if (!this.canTransitionTo(TripStatus.ASSIGNED)) {
      throw new Error(`Cannot assign driver in ${this.status} status`);
    }

    this.driverId = driverId;
    this.vehicleId = vehicleId;
    this.status = TripStatus.ASSIGNED;
    this.updatedAt = new Date();
  }

  start(): void {
    if (!this.canTransitionTo(TripStatus.ACTIVE)) {
      throw new Error(`Cannot start trip in ${this.status} status`);
    }

    if (!this.driverId) {
      throw new Error('Cannot start trip without assigned driver');
    }

    this.status = TripStatus.ACTIVE;
    this.updatedAt = new Date();
  }

  complete(fare: number): void {
    if (!this.canTransitionTo(TripStatus.COMPLETED)) {
      throw new Error(`Cannot complete trip in ${this.status} status`);
    }

    this.fare = fare;
    this.status = TripStatus.COMPLETED;
    this.updatedAt = new Date();
  }

  calculateEstimatedFare(): number {
    const distance = this.pickupLocation.distanceTo(this.destinationLocation);
    const baseRate = 3.0;
    const perKmRate = 1.5;
    return baseRate + (distance * perKmRate);
  }
}

export enum TripStatus {
  PENDING = 'pending',
  ASSIGNED = 'assigned',
  ACTIVE = 'active',
  COMPLETED = 'completed',
  CANCELLED = 'cancelled'
}
```

## Repository Pattern

```typescript
export interface ITripRepository {
  findById(id: string): Promise<Trip | null>;
  findByStatus(status: TripStatus): Promise<Trip[]>;
  findByDriver(driverId: string): Promise<Trip[]>;
  save(trip: Trip): Promise<void>;
  update(trip: Trip): Promise<void>;
  delete(id: string): Promise<void>;
}

export class SupabaseTripRepository implements ITripRepository {
  constructor(private supabase: SupabaseClient) {}

  async findById(id: string): Promise<Trip | null> {
    const { data, error } = await this.supabase
      .from('trips')
      .select('*')
      .eq('id', id)
      .maybeSingle();

    if (error || !data) return null;
    return Trip.fromDatabase(data);
  }

  async findByStatus(status: TripStatus): Promise<Trip[]> {
    const { data, error } = await this.supabase
      .from('trips')
      .select('*')
      .eq('status', status)
      .order('created_at', { ascending: false });

    if (error || !data) return [];
    return data.map(Trip.fromDatabase);
  }

  async findByDriver(driverId: string): Promise<Trip[]> {
    const { data, error } = await this.supabase
      .from('trips')
      .select('*')
      .eq('driver_id', driverId)
      .order('created_at', { ascending: false });

    if (error || !data) return [];
    return data.map(Trip.fromDatabase);
  }

  async save(trip: Trip): Promise<void> {
    const { error } = await this.supabase
      .from('trips')
      .insert(this.toDatabase(trip));

    if (error) {
      throw new Error(`Failed to save trip: ${error.message}`);
    }
  }

  async update(trip: Trip): Promise<void> {
    const { error } = await this.supabase
      .from('trips')
      .update(this.toDatabase(trip))
      .eq('id', trip.id);

    if (error) {
      throw new Error(`Failed to update trip: ${error.message}`);
    }
  }

  async delete(id: string): Promise<void> {
    const { error } = await this.supabase
      .from('trips')
      .delete()
      .eq('id', id);

    if (error) {
      throw new Error(`Failed to delete trip: ${error.message}`);
    }
  }

  private toDatabase(trip: Trip) {
    return {
      id: trip.id,
      status: trip.status,
      pickup_lat: trip.pickupLocation.latitude,
      pickup_lng: trip.pickupLocation.longitude,
      destination_lat: trip.destinationLocation.latitude,
      destination_lng: trip.destinationLocation.longitude,
      driver_id: trip.driverId,
      vehicle_id: trip.vehicleId,
      fare: trip.fare,
      created_at: trip.createdAt.toISOString(),
      updated_at: trip.updatedAt.toISOString()
    };
  }
}
```

# Code Examples

## Service Layer Pattern

```typescript
export class TripService {
  constructor(
    private tripRepository: ITripRepository,
    private driverRepository: IDriverRepository,
    private notificationService: INotificationService
  ) {}

  async createTrip(
    pickupLocation: Location,
    destinationLocation: Location,
    userId: string
  ): Promise<Trip> {
    const trip = Trip.create(pickupLocation, destinationLocation);

    await this.tripRepository.save(trip);

    return trip;
  }

  async assignDriverToTrip(
    tripId: string,
    driverId: string
  ): Promise<Trip> {
    const trip = await this.tripRepository.findById(tripId);
    if (!trip) {
      throw new Error('Trip not found');
    }

    const driver = await this.driverRepository.findById(driverId);
    if (!driver) {
      throw new Error('Driver not found');
    }

    if (!driver.isAvailable()) {
      throw new Error('Driver is not available');
    }

    trip.assignDriver(driverId, driver.vehicleId);
    await this.tripRepository.update(trip);

    await this.notificationService.notifyDriverAssigned(trip, driver);

    return trip;
  }

  async startTrip(tripId: string, driverId: string): Promise<Trip> {
    const trip = await this.tripRepository.findById(tripId);
    if (!trip) {
      throw new Error('Trip not found');
    }

    if (trip.driverId !== driverId) {
      throw new Error('Only assigned driver can start trip');
    }

    trip.start();
    await this.tripRepository.update(trip);

    return trip;
  }

  async completeTrip(
    tripId: string,
    driverId: string,
    fare: number
  ): Promise<Trip> {
    const trip = await this.tripRepository.findById(tripId);
    if (!trip) {
      throw new Error('Trip not found');
    }

    if (trip.driverId !== driverId) {
      throw new Error('Only assigned driver can complete trip');
    }

    trip.complete(fare);
    await this.tripRepository.update(trip);

    await this.notificationService.notifyTripCompleted(trip);

    return trip;
  }

  async getActiveTrips(): Promise<Trip[]> {
    return this.tripRepository.findByStatus(TripStatus.ACTIVE);
  }

  async getDriverTrips(driverId: string): Promise<Trip[]> {
    return this.tripRepository.findByDriver(driverId);
  }
}
```

## Use Case Pattern

```typescript
interface CreateTripRequest {
  userId: string;
  pickupAddress: string;
  pickupLat: number;
  pickupLng: number;
  destinationAddress: string;
  destinationLat: number;
  destinationLng: number;
}

interface CreateTripResponse {
  tripId: string;
  status: string;
  estimatedFare: number;
}

export class CreateTripUseCase {
  constructor(
    private tripService: TripService,
    private geocodingService: IGeocodingService
  ) {}

  async execute(request: CreateTripRequest): Promise<CreateTripResponse> {
    const pickupLocation = Location.fromCoordinates(
      request.pickupLat,
      request.pickupLng,
      request.pickupAddress
    );

    const destinationLocation = Location.fromCoordinates(
      request.destinationLat,
      request.destinationLng,
      request.destinationAddress
    );

    this.validateLocations(pickupLocation, destinationLocation);

    const trip = await this.tripService.createTrip(
      pickupLocation,
      destinationLocation,
      request.userId
    );

    return {
      tripId: trip.id,
      status: trip.status,
      estimatedFare: trip.calculateEstimatedFare()
    };
  }

  private validateLocations(pickup: Location, destination: Location): void {
    const minDistance = 0.5;
    const maxDistance = 100;

    const distance = pickup.distanceTo(destination);

    if (distance < minDistance) {
      throw new Error('Trip distance too short');
    }

    if (distance > maxDistance) {
      throw new Error('Trip distance exceeds maximum allowed');
    }
  }
}
```

## Value Object Pattern

```typescript
export class Location {
  private constructor(
    public readonly latitude: number,
    public readonly longitude: number,
    public readonly address?: string
  ) {
    this.validate();
  }

  static fromCoordinates(
    latitude: number,
    longitude: number,
    address?: string
  ): Location {
    return new Location(latitude, longitude, address);
  }

  private validate(): void {
    if (this.latitude < -90 || this.latitude > 90) {
      throw new Error('Invalid latitude');
    }

    if (this.longitude < -180 || this.longitude > 180) {
      throw new Error('Invalid longitude');
    }
  }

  distanceTo(other: Location): number {
    const R = 6371;
    const dLat = this.toRad(other.latitude - this.latitude);
    const dLon = this.toRad(other.longitude - this.longitude);

    const a =
      Math.sin(dLat / 2) * Math.sin(dLat / 2) +
      Math.cos(this.toRad(this.latitude)) *
        Math.cos(this.toRad(other.latitude)) *
        Math.sin(dLon / 2) *
        Math.sin(dLon / 2);

    const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
    return R * c;
  }

  private toRad(degrees: number): number {
    return degrees * (Math.PI / 180);
  }

  equals(other: Location): boolean {
    return (
      this.latitude === other.latitude &&
      this.longitude === other.longitude
    );
  }
}
```

## Extract Common Logic

Before refactoring:

```typescript
async function createTrip(data) {
  const trip = await supabase.from('trips').insert(data).select().single();
  await supabase.from('notifications').insert({
    type: 'trip_created',
    trip_id: trip.id
  });
  return trip;
}

async function assignDriver(tripId, driverId) {
  const trip = await supabase.from('trips').update({
    driver_id: driverId,
    status: 'assigned'
  }).eq('id', tripId).select().single();

  await supabase.from('notifications').insert({
    type: 'driver_assigned',
    trip_id: trip.id
  });

  return trip;
}
```

After refactoring:

```typescript
class TripEventPublisher {
  async publishTripCreated(trip: Trip): Promise<void> {
    await this.publish('trip_created', { tripId: trip.id });
  }

  async publishDriverAssigned(trip: Trip, driver: Driver): Promise<void> {
    await this.publish('driver_assigned', {
      tripId: trip.id,
      driverId: driver.id
    });
  }

  private async publish(eventType: string, payload: any): Promise<void> {
    await supabase.from('notifications').insert({
      type: eventType,
      ...payload
    });
  }
}

class TripService {
  constructor(
    private repository: ITripRepository,
    private eventPublisher: TripEventPublisher
  ) {}

  async createTrip(data: CreateTripData): Promise<Trip> {
    const trip = Trip.create(data.pickupLocation, data.destinationLocation);
    await this.repository.save(trip);
    await this.eventPublisher.publishTripCreated(trip);
    return trip;
  }

  async assignDriver(tripId: string, driverId: string): Promise<Trip> {
    const trip = await this.repository.findById(tripId);
    const driver = await this.driverRepository.findById(driverId);

    trip.assignDriver(driverId, driver.vehicleId);
    await this.repository.update(trip);
    await this.eventPublisher.publishDriverAssigned(trip, driver);

    return trip;
  }
}
```

# Anti-Patterns

❌ **God Classes** - Classes that do everything
```typescript
class TripManager {
  createTrip() {}
  updateTrip() {}
  deleteTrip() {}
  assignDriver() {}
  calculateFare() {}
  sendNotification() {}
  processPayment() {}
}
```

✅ **Single Responsibility Classes**
```typescript
class TripService { /* trip operations */ }
class DriverAssignmentService { /* driver assignment */ }
class FareCalculator { /* fare calculation */ }
class NotificationService { /* notifications */ }
```

❌ **Tight Coupling to Framework**
```typescript
async function createTrip(req, res) {
  const trip = await supabase.from('trips').insert(req.body);
  res.json(trip);
}
```

✅ **Layered Architecture**
```typescript
class TripController {
  async createTrip(req, res) {
    const useCase = new CreateTripUseCase(tripService);
    const result = await useCase.execute(req.body);
    res.json(result);
  }
}
```

❌ **Anemic Domain Model**
```typescript
interface Trip {
  id: string;
  status: string;
}

function canAssignDriver(trip: Trip): boolean {
  return trip.status === 'pending';
}
```

✅ **Rich Domain Model**
```typescript
class Trip {
  canAssignDriver(): boolean {
    return this.status === TripStatus.PENDING;
  }
}
```

# Performance Guidelines

1. **Lazy Loading**: Load related entities only when needed
2. **Batch Operations**: Group database operations
3. **Caching**: Cache frequently accessed data
4. **Async Processing**: Move heavy operations to background jobs
5. **Query Optimization**: Select only needed fields

# Observability

Add logging to track refactoring impact:

```typescript
class TripService {
  async createTrip(data: CreateTripData): Promise<Trip> {
    const startTime = performance.now();

    const trip = await this.innerCreateTrip(data);

    const duration = performance.now() - startTime;
    logger.info('Trip created', {
      tripId: trip.id,
      duration,
      method: 'createTrip'
    });

    return trip;
  }
}
```

# Testing Strategy

## Unit Tests for Domain Logic

```typescript
describe('Trip', () => {
  test('can transition from pending to assigned', () => {
    const trip = Trip.create(pickup, destination);
    expect(trip.canTransitionTo(TripStatus.ASSIGNED)).toBe(true);
  });

  test('cannot transition from completed to active', () => {
    const trip = Trip.create(pickup, destination);
    trip.complete(10);
    expect(trip.canTransitionTo(TripStatus.ACTIVE)).toBe(false);
  });

  test('calculates estimated fare correctly', () => {
    const trip = Trip.create(
      Location.fromCoordinates(0, 0),
      Location.fromCoordinates(0.01, 0.01)
    );
    const fare = trip.calculateEstimatedFare();
    expect(fare).toBeGreaterThan(3);
  });
});
```

## Integration Tests for Services

```typescript
describe('TripService', () => {
  let service: TripService;
  let repository: ITripRepository;

  beforeEach(() => {
    repository = new InMemoryTripRepository();
    service = new TripService(repository, eventPublisher);
  });

  test('creates trip and publishes event', async () => {
    const trip = await service.createTrip(pickup, destination, 'user-1');

    expect(trip.id).toBeDefined();
    expect(trip.status).toBe(TripStatus.PENDING);
    expect(eventPublisher.publishTripCreated).toHaveBeenCalledWith(trip);
  });
});
```
