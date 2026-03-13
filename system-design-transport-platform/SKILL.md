---
name: system-design-transport-platform
description: System design patterns for dispatch, route optimization, and real-time tracking in transportation platforms
---

# System Design Transport Platform

Comprehensive system design patterns and strategies for building scalable transportation platforms with real-time dispatch, route optimization, trip lifecycle management, and location tracking capabilities.

# Description

This skill provides detailed system design guidance for the core operational components of transportation platforms. It covers the architecture and implementation strategies for dispatch systems that match drivers to trips, route optimization algorithms, complete trip lifecycle management from creation to completion, and real-time tracking systems. The skill focuses on scalability, reliability, and performance considerations specific to transportation operations.

# When to Activate

Activate this skill when:

- Designing a dispatch system for matching drivers to trips
- Implementing route optimization and planning algorithms
- Building trip lifecycle management and state transitions
- Creating real-time location tracking systems
- Scaling dispatch operations to handle high throughput
- Designing systems for real-time driver and vehicle location
- Implementing trip scheduling and assignment logic
- Building systems for trip history and analytics
- Optimizing performance for location-based queries
- Designing notification systems for trip updates

# Engineering Principles

## Real-Time First

Transportation platforms are inherently real-time systems:
- Trips happen now, not later
- Driver locations update continuously
- Dispatch decisions must be made in seconds
- Users expect immediate feedback

Design for low latency and real-time updates from the start.

## Eventual Consistency

Accept that perfect consistency is impossible:
- GPS coordinates may arrive out of order
- Network delays affect location updates
- Multiple systems may have slightly different views

Design for eventual consistency with conflict resolution strategies.

## Idempotency

Operations must be safely retryable:
- Trip assignment can be called multiple times safely
- Location updates should handle duplicates
- Notifications should not send twice for the same event

All state-changing operations must be idempotent.

## Graceful Degradation

Systems should degrade gracefully under failure:
- If route optimization is slow, fall back to simple distance
- If real-time tracking fails, use last known location
- If dispatch is overloaded, queue requests

Never block critical operations waiting for non-critical ones.

# Domain Modeling

## Dispatch Domain

### Core Concepts

```
Dispatch Request
├── id: uuid
├── trip_id: uuid
├── origin: geography (point)
├── destination: geography (point)
├── requested_at: timestamp
├── required_vehicle_type: enum
├── priority: integer
└── constraints: jsonb

Dispatch Assignment
├── id: uuid
├── dispatch_request_id: uuid
├── driver_id: uuid
├── vehicle_id: uuid
├── assigned_at: timestamp
├── driver_distance: decimal
├── estimated_arrival: interval
└── assignment_score: decimal

Dispatch Queue
├── id: uuid
├── trip_id: uuid
├── status: enum (queued, processing, assigned, failed)
├── retry_count: integer
├── priority: integer
├── queued_at: timestamp
└── processed_at: timestamp
```

## Route Domain

### Core Concepts

```
Route Plan
├── id: uuid
├── origin: geography (point)
├── destination: geography (point)
├── waypoints: geography[] (points)
├── total_distance: decimal
├── estimated_duration: interval
├── polyline: text (encoded)
├── algorithm_used: string
└── created_at: timestamp

Route Segment
├── id: uuid
├── route_plan_id: uuid
├── sequence: integer
├── start_point: geography (point)
├── end_point: geography (point)
├── distance: decimal
├── duration: interval
└── instructions: text

Route Optimization Result
├── id: uuid
├── input_waypoints: geography[]
├── optimized_sequence: integer[]
├── total_distance: decimal
├── total_duration: interval
├── savings_vs_original: decimal
└── computed_at: timestamp
```

## Trip Lifecycle Domain

### Core Concepts

```
Trip
├── id: uuid
├── route_plan_id: uuid
├── driver_id: uuid
├── vehicle_id: uuid
├── status: enum
├── scheduled_start: timestamp
├── actual_start: timestamp
├── estimated_end: timestamp
├── actual_end: timestamp
├── origin: geography (point)
├── destination: geography (point)
└── metadata: jsonb

Trip Timeline
├── id: uuid
├── trip_id: uuid
├── event_type: enum
├── occurred_at: timestamp
├── location: geography (point)
├── actor: uuid
├── details: jsonb
└── sequence: integer

Trip Metrics
├── trip_id: uuid (PK)
├── total_distance: decimal
├── total_duration: interval
├── idle_time: interval
├── average_speed: decimal
├── route_deviation: decimal
└── computed_at: timestamp
```

## Tracking Domain

### Core Concepts

```
Location Update
├── id: uuid
├── entity_type: enum (driver, vehicle, trip)
├── entity_id: uuid
├── location: geography (point)
├── accuracy: decimal
├── bearing: decimal
├── speed: decimal
├── recorded_at: timestamp
└── received_at: timestamp

Tracking Session
├── id: uuid
├── trip_id: uuid
├── started_at: timestamp
├── ended_at: timestamp
├── total_points: integer
├── average_accuracy: decimal
└── data_quality_score: decimal

Geofence
├── id: uuid
├── name: string
├── boundary: geography (polygon)
├── radius: decimal
├── center: geography (point)
└── active: boolean

Geofence Event
├── id: uuid
├── geofence_id: uuid
├── entity_id: uuid
├── event_type: enum (entered, exited)
├── occurred_at: timestamp
└── location: geography (point)
```

# Architecture Guidelines

## Dispatch Service Architecture

### High-Level Design

```
┌─────────────────────────────────────────────────┐
│             Trip Request                        │
└──────────────┬──────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────┐
│         Dispatch Orchestrator                   │
│  • Validates request                            │
│  • Determines priority                          │
│  • Adds to dispatch queue                       │
└──────────────┬──────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────┐
│          Dispatch Queue                         │
│  • Priority queue                               │
│  • Rate limiting                                │
│  • Retry logic                                  │
└──────────────┬──────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────┐
│        Candidate Selection                      │
│  • Query available drivers                      │
│  • Filter by vehicle type                       │
│  • Filter by location (radius)                  │
│  • Filter by rating (if required)               │
└──────────────┬──────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────┐
│         Scoring Engine                          │
│  • Distance score                               │
│  • ETA score                                    │
│  • Rating score                                 │
│  • Historical performance                       │
└──────────────┬──────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────┐
│       Assignment Logic                          │
│  • Select best candidate                        │
│  • Reserve driver (atomic)                      │
│  • Create assignment record                     │
│  • Notify driver                                │
└──────────────┬──────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────┐
│      Assignment Confirmation                    │
│  • Wait for driver acceptance                   │
│  • Timeout handling                             │
│  • Fallback to next candidate                   │
└─────────────────────────────────────────────────┘
```

### Database Schema for Dispatch

```sql
CREATE SCHEMA dispatch;

CREATE TABLE dispatch.requests (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  trip_id uuid NOT NULL,
  origin geography(POINT) NOT NULL,
  destination geography(POINT) NOT NULL,
  vehicle_type text NOT NULL,
  priority integer DEFAULT 0,
  status text NOT NULL CHECK (status IN ('pending', 'processing', 'assigned', 'failed', 'cancelled')),
  requested_at timestamptz DEFAULT now(),
  assigned_at timestamptz,
  failed_reason text,
  retry_count integer DEFAULT 0,
  max_retries integer DEFAULT 3,
  constraints jsonb DEFAULT '{}'::jsonb
);

CREATE INDEX idx_dispatch_requests_status ON dispatch.requests(status) WHERE status IN ('pending', 'processing');
CREATE INDEX idx_dispatch_requests_priority ON dispatch.requests(priority DESC, requested_at);

CREATE TABLE dispatch.assignments (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  request_id uuid REFERENCES dispatch.requests(id),
  driver_id uuid NOT NULL,
  vehicle_id uuid NOT NULL,
  assigned_at timestamptz DEFAULT now(),
  accepted_at timestamptz,
  rejected_at timestamptz,
  driver_distance decimal,
  estimated_arrival interval,
  score decimal,
  status text NOT NULL CHECK (status IN ('pending', 'accepted', 'rejected', 'expired'))
);

CREATE INDEX idx_dispatch_assignments_driver ON dispatch.assignments(driver_id);
CREATE INDEX idx_dispatch_assignments_request ON dispatch.assignments(request_id);
```

### Dispatch Algorithm Implementation

```typescript
interface DispatchCandidate {
  driverId: string;
  vehicleId: string;
  location: GeoPoint;
  distance: number;
  eta: number;
  rating: number;
  totalTrips: number;
}

interface DispatchScore {
  driverId: string;
  totalScore: number;
  distanceScore: number;
  etaScore: number;
  ratingScore: number;
}

class DispatchService {
  async dispatch(request: DispatchRequest): Promise<Assignment> {
    // Step 1: Find available drivers within radius
    const candidates = await this.findCandidates(
      request.origin,
      request.vehicleType,
      request.radiusKm
    );

    if (candidates.length === 0) {
      throw new Error('No available drivers found');
    }

    // Step 2: Score candidates
    const scoredCandidates = this.scoreCandidates(candidates, request);

    // Step 3: Sort by score (highest first)
    scoredCandidates.sort((a, b) => b.totalScore - a.totalScore);

    // Step 4: Try to assign to best candidate
    for (const candidate of scoredCandidates.slice(0, 3)) {
      try {
        const assignment = await this.assignDriver(
          request,
          candidate.driverId
        );
        return assignment;
      } catch (error) {
        // Driver already assigned, try next
        continue;
      }
    }

    throw new Error('Failed to assign driver');
  }

  private scoreCandidates(
    candidates: DispatchCandidate[],
    request: DispatchRequest
  ): DispatchScore[] {
    return candidates.map(candidate => {
      // Distance score: closer is better (inverse relationship)
      // Max score of 40 points
      const maxDistance = 20; // km
      const distanceScore = Math.max(
        0,
        40 * (1 - candidate.distance / maxDistance)
      );

      // ETA score: faster is better
      // Max score of 30 points
      const maxEta = 30; // minutes
      const etaScore = Math.max(
        0,
        30 * (1 - candidate.eta / maxEta)
      );

      // Rating score: higher rating is better
      // Max score of 30 points
      const ratingScore = (candidate.rating / 5) * 30;

      const totalScore = distanceScore + etaScore + ratingScore;

      return {
        driverId: candidate.driverId,
        totalScore,
        distanceScore,
        etaScore,
        ratingScore
      };
    });
  }

  private async assignDriver(
    request: DispatchRequest,
    driverId: string
  ): Promise<Assignment> {
    // Atomic operation: reserve driver and create assignment
    const { data, error } = await supabase.rpc('assign_driver_atomic', {
      p_request_id: request.id,
      p_driver_id: driverId
    });

    if (error) throw error;

    // Emit event for other services
    eventBus.emit('dispatch.assigned', {
      requestId: request.id,
      driverId: driverId,
      assignedAt: new Date()
    });

    return data;
  }
}
```

### Atomic Assignment Function

```sql
CREATE OR REPLACE FUNCTION dispatch.assign_driver_atomic(
  p_request_id uuid,
  p_driver_id uuid
) RETURNS jsonb AS $$
DECLARE
  v_driver_available boolean;
  v_assignment_id uuid;
  v_vehicle_id uuid;
BEGIN
  -- Lock driver row for update
  SELECT
    availability_status = 'available',
    current_vehicle_id
  INTO v_driver_available, v_vehicle_id
  FROM drivers.drivers
  WHERE id = p_driver_id
  FOR UPDATE;

  -- Check if driver is available
  IF NOT v_driver_available THEN
    RAISE EXCEPTION 'Driver not available';
  END IF;

  -- Update driver status
  UPDATE drivers.drivers
  SET availability_status = 'on_trip'
  WHERE id = p_driver_id;

  -- Create assignment
  INSERT INTO dispatch.assignments (
    request_id,
    driver_id,
    vehicle_id,
    status
  ) VALUES (
    p_request_id,
    p_driver_id,
    v_vehicle_id,
    'pending'
  ) RETURNING id INTO v_assignment_id;

  -- Update request status
  UPDATE dispatch.requests
  SET
    status = 'assigned',
    assigned_at = now()
  WHERE id = p_request_id;

  RETURN jsonb_build_object(
    'assignmentId', v_assignment_id,
    'driverId', p_driver_id,
    'vehicleId', v_vehicle_id
  );
END;
$$ LANGUAGE plpgsql;
```

## Route Optimization

### Distance Matrix Calculation

```typescript
interface Point {
  lat: number;
  lng: number;
}

class RouteOptimizer {
  async optimizeRoute(waypoints: Point[]): Promise<number[]> {
    if (waypoints.length <= 2) {
      return waypoints.map((_, i) => i);
    }

    // Calculate distance matrix
    const matrix = await this.calculateDistanceMatrix(waypoints);

    // Use nearest neighbor algorithm for small sets
    if (waypoints.length <= 10) {
      return this.nearestNeighbor(matrix);
    }

    // Use 2-opt for larger sets
    return this.twoOptOptimization(matrix);
  }

  private async calculateDistanceMatrix(
    points: Point[]
  ): Promise<number[][]> {
    const n = points.length;
    const matrix: number[][] = Array(n).fill(0).map(() => Array(n).fill(0));

    for (let i = 0; i < n; i++) {
      for (let j = i + 1; j < n; j++) {
        const distance = this.haversineDistance(points[i], points[j]);
        matrix[i][j] = distance;
        matrix[j][i] = distance;
      }
    }

    return matrix;
  }

  private nearestNeighbor(matrix: number[][]): number[] {
    const n = matrix.length;
    const visited = new Set<number>();
    const route: number[] = [0];
    visited.add(0);

    let current = 0;

    while (visited.size < n) {
      let nearest = -1;
      let minDistance = Infinity;

      for (let i = 0; i < n; i++) {
        if (!visited.has(i) && matrix[current][i] < minDistance) {
          nearest = i;
          minDistance = matrix[current][i];
        }
      }

      route.push(nearest);
      visited.add(nearest);
      current = nearest;
    }

    return route;
  }

  private twoOptOptimization(matrix: number[][]): number[] {
    let route = this.nearestNeighbor(matrix);
    let improved = true;

    while (improved) {
      improved = false;

      for (let i = 1; i < route.length - 2; i++) {
        for (let j = i + 1; j < route.length - 1; j++) {
          const currentDistance =
            matrix[route[i - 1]][route[i]] +
            matrix[route[j]][route[j + 1]];

          const newDistance =
            matrix[route[i - 1]][route[j]] +
            matrix[route[i]][route[j + 1]];

          if (newDistance < currentDistance) {
            // Reverse segment between i and j
            route = [
              ...route.slice(0, i),
              ...route.slice(i, j + 1).reverse(),
              ...route.slice(j + 1)
            ];
            improved = true;
          }
        }
      }
    }

    return route;
  }

  private haversineDistance(p1: Point, p2: Point): number {
    const R = 6371; // Earth radius in km
    const dLat = (p2.lat - p1.lat) * Math.PI / 180;
    const dLng = (p2.lng - p1.lng) * Math.PI / 180;

    const a =
      Math.sin(dLat / 2) * Math.sin(dLat / 2) +
      Math.cos(p1.lat * Math.PI / 180) *
      Math.cos(p2.lat * Math.PI / 180) *
      Math.sin(dLng / 2) * Math.sin(dLng / 2);

    const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
    return R * c;
  }
}
```

### Route Caching Strategy

```typescript
class RouteCacheService {
  private readonly CACHE_TTL = 3600; // 1 hour

  async getRoute(
    origin: Point,
    destination: Point
  ): Promise<RoutePlan | null> {
    const cacheKey = this.generateCacheKey(origin, destination);

    // Try to get from cache
    const cached = await this.cache.get(cacheKey);
    if (cached) {
      return JSON.parse(cached);
    }

    // Calculate new route
    const route = await this.calculateRoute(origin, destination);

    // Store in cache
    await this.cache.set(
      cacheKey,
      JSON.stringify(route),
      this.CACHE_TTL
    );

    return route;
  }

  private generateCacheKey(origin: Point, destination: Point): string {
    // Round coordinates to reduce cache size
    const roundedOrigin = {
      lat: Math.round(origin.lat * 1000) / 1000,
      lng: Math.round(origin.lng * 1000) / 1000
    };
    const roundedDest = {
      lat: Math.round(destination.lat * 1000) / 1000,
      lng: Math.round(destination.lng * 1000) / 1000
    };

    return `route:${roundedOrigin.lat},${roundedOrigin.lng}:${roundedDest.lat},${roundedDest.lng}`;
  }
}
```

## Trip Lifecycle Management

### State Machine Implementation

```typescript
type TripStatus =
  | 'pending'
  | 'assigned'
  | 'accepted'
  | 'driver_arriving'
  | 'arrived'
  | 'in_progress'
  | 'completed'
  | 'cancelled';

type TripTransition =
  | 'assign'
  | 'accept'
  | 'driver_en_route'
  | 'arrive'
  | 'start'
  | 'complete'
  | 'cancel';

const TRANSITION_GRAPH: Record<TripStatus, TripTransition[]> = {
  pending: ['assign', 'cancel'],
  assigned: ['accept', 'cancel'],
  accepted: ['driver_en_route', 'cancel'],
  driver_arriving: ['arrive', 'cancel'],
  arrived: ['start', 'cancel'],
  in_progress: ['complete', 'cancel'],
  completed: [],
  cancelled: []
};

const NEXT_STATUS: Record<TripTransition, TripStatus> = {
  assign: 'assigned',
  accept: 'accepted',
  driver_en_route: 'driver_arriving',
  arrive: 'arrived',
  start: 'in_progress',
  complete: 'completed',
  cancel: 'cancelled'
};

class TripLifecycleService {
  async transitionTrip(
    tripId: string,
    transition: TripTransition,
    metadata?: Record<string, any>
  ): Promise<Trip> {
    // Get current trip
    const trip = await this.getTrip(tripId);

    // Validate transition
    if (!this.canTransition(trip.status, transition)) {
      throw new Error(
        `Invalid transition: ${trip.status} -> ${transition}`
      );
    }

    const newStatus = NEXT_STATUS[transition];

    // Update trip status
    const updatedTrip = await this.updateTripStatus(
      tripId,
      newStatus,
      metadata
    );

    // Record timeline event
    await this.recordTimelineEvent(tripId, {
      eventType: transition,
      fromStatus: trip.status,
      toStatus: newStatus,
      metadata
    });

    // Emit event
    eventBus.emit('trip.status_changed', {
      tripId,
      previousStatus: trip.status,
      newStatus,
      transition
    });

    // Execute side effects
    await this.executeSideEffects(transition, updatedTrip);

    return updatedTrip;
  }

  private canTransition(
    currentStatus: TripStatus,
    transition: TripTransition
  ): boolean {
    return TRANSITION_GRAPH[currentStatus]?.includes(transition) ?? false;
  }

  private async executeSideEffects(
    transition: TripTransition,
    trip: Trip
  ): Promise<void> {
    switch (transition) {
      case 'complete':
        // Update driver availability
        await driversService.updateAvailability(
          trip.driverId,
          'available'
        );
        // Calculate metrics
        await this.calculateTripMetrics(trip);
        break;

      case 'cancel':
        // Release driver if assigned
        if (trip.driverId) {
          await driversService.updateAvailability(
            trip.driverId,
            'available'
          );
        }
        break;

      case 'start':
        // Start tracking session
        await trackingService.startSession(trip.id);
        break;
    }
  }
}
```

### Trip Timeline Tracking

```sql
CREATE TABLE trips.timeline (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  trip_id uuid NOT NULL REFERENCES trips.trips(id),
  event_type text NOT NULL,
  from_status text,
  to_status text,
  occurred_at timestamptz DEFAULT now(),
  location geography(POINT),
  metadata jsonb DEFAULT '{}'::jsonb,
  sequence serial
);

CREATE INDEX idx_trips_timeline_trip ON trips.timeline(trip_id, sequence);
CREATE INDEX idx_trips_timeline_occurred ON trips.timeline(occurred_at);
```

## Real-Time Tracking System

### Location Update Handler

```typescript
interface LocationUpdate {
  entityType: 'driver' | 'vehicle' | 'trip';
  entityId: string;
  location: Point;
  accuracy: number;
  bearing: number;
  speed: number;
  recordedAt: Date;
}

class TrackingService {
  private readonly MAX_QUEUE_SIZE = 10000;
  private readonly BATCH_SIZE = 100;
  private readonly BATCH_INTERVAL = 1000; // ms

  private updateQueue: LocationUpdate[] = [];
  private processingBatch = false;

  async recordLocation(update: LocationUpdate): Promise<void> {
    // Validate update
    this.validateLocationUpdate(update);

    // Add to queue
    this.updateQueue.push(update);

    // Start batch processing if not already running
    if (!this.processingBatch) {
      this.startBatchProcessing();
    }

    // Publish to real-time channels immediately
    await this.publishRealtimeUpdate(update);
  }

  private async startBatchProcessing(): Promise<void> {
    this.processingBatch = true;

    while (this.updateQueue.length > 0) {
      const batch = this.updateQueue.splice(0, this.BATCH_SIZE);

      try {
        await this.processBatch(batch);
      } catch (error) {
        console.error('Failed to process location batch', error);
        // Put failed items back in queue
        this.updateQueue.unshift(...batch);
      }

      await new Promise(resolve =>
        setTimeout(resolve, this.BATCH_INTERVAL)
      );
    }

    this.processingBatch = false;
  }

  private async processBatch(
    updates: LocationUpdate[]
  ): Promise<void> {
    // Batch insert to database
    await supabase.from('tracking.location_updates').insert(
      updates.map(u => ({
        entity_type: u.entityType,
        entity_id: u.entityId,
        location: `POINT(${u.location.lng} ${u.location.lat})`,
        accuracy: u.accuracy,
        bearing: u.bearing,
        speed: u.speed,
        recorded_at: u.recordedAt,
        received_at: new Date()
      }))
    );

    // Update current location cache
    await this.updateLocationCache(updates);

    // Check geofences
    await this.checkGeofences(updates);
  }

  private async publishRealtimeUpdate(
    update: LocationUpdate
  ): Promise<void> {
    // Publish to Supabase Realtime
    const channel = `location:${update.entityType}:${update.entityId}`;
    await supabase.channel(channel).send({
      type: 'broadcast',
      event: 'location_update',
      payload: update
    });
  }

  private async updateLocationCache(
    updates: LocationUpdate[]
  ): Promise<void> {
    // Group by entity
    const byEntity = new Map<string, LocationUpdate>();

    for (const update of updates) {
      const key = `${update.entityType}:${update.entityId}`;
      const existing = byEntity.get(key);

      if (!existing || update.recordedAt > existing.recordedAt) {
        byEntity.set(key, update);
      }
    }

    // Update cache with latest locations
    const cacheUpdates = Array.from(byEntity.entries()).map(
      ([key, update]) => ({
        key: `location:${key}`,
        value: JSON.stringify(update),
        ttl: 3600 // 1 hour
      })
    );

    await this.cache.mset(cacheUpdates);
  }
}
```

### Geofencing Implementation

```typescript
class GeofenceService {
  async checkGeofence(
    location: Point,
    entityId: string
  ): Promise<GeofenceEvent[]> {
    // Get active geofences
    const geofences = await this.getActiveGeofences();

    const events: GeofenceEvent[] = [];

    for (const geofence of geofences) {
      const isInside = this.isPointInPolygon(
        location,
        geofence.boundary
      );

      const wasInside = await this.wasEntityInGeofence(
        entityId,
        geofence.id
      );

      if (isInside && !wasInside) {
        // Entered geofence
        events.push({
          geofenceId: geofence.id,
          entityId,
          eventType: 'entered',
          location,
          occurredAt: new Date()
        });

        await this.recordGeofenceState(entityId, geofence.id, true);
      } else if (!isInside && wasInside) {
        // Exited geofence
        events.push({
          geofenceId: geofence.id,
          entityId,
          eventType: 'exited',
          location,
          occurredAt: new Date()
        });

        await this.recordGeofenceState(entityId, geofence.id, false);
      }
    }

    return events;
  }

  private isPointInPolygon(point: Point, polygon: Point[]): boolean {
    let inside = false;

    for (let i = 0, j = polygon.length - 1; i < polygon.length; j = i++) {
      const xi = polygon[i].lng;
      const yi = polygon[i].lat;
      const xj = polygon[j].lng;
      const yj = polygon[j].lat;

      const intersect =
        ((yi > point.lat) !== (yj > point.lat)) &&
        (point.lng < (xj - xi) * (point.lat - yi) / (yj - yi) + xi);

      if (intersect) inside = !inside;
    }

    return inside;
  }
}
```

### Spatial Queries with PostGIS

```sql
-- Enable PostGIS
CREATE EXTENSION IF NOT EXISTS postgis;

-- Find drivers within radius
CREATE OR REPLACE FUNCTION dispatch.find_drivers_in_radius(
  p_location geography,
  p_radius_meters integer,
  p_vehicle_type text DEFAULT NULL
) RETURNS TABLE (
  driver_id uuid,
  vehicle_id uuid,
  distance_meters decimal,
  location geography
) AS $$
BEGIN
  RETURN QUERY
  SELECT
    d.id,
    d.current_vehicle_id,
    ST_Distance(l.location, p_location)::decimal,
    l.location
  FROM drivers.drivers d
  INNER JOIN tracking.current_locations l
    ON l.entity_id = d.id AND l.entity_type = 'driver'
  LEFT JOIN vehicles.vehicles v
    ON v.id = d.current_vehicle_id
  WHERE
    d.availability_status = 'available'
    AND ST_DWithin(l.location, p_location, p_radius_meters)
    AND (p_vehicle_type IS NULL OR v.vehicle_type = p_vehicle_type)
  ORDER BY ST_Distance(l.location, p_location);
END;
$$ LANGUAGE plpgsql;

-- Create spatial index
CREATE INDEX idx_locations_geography
  ON tracking.current_locations
  USING GIST(location);
```

# Implementation Patterns

## Dispatch Queue Pattern

Use a priority queue for managing dispatch requests:

```typescript
class DispatchQueue {
  async enqueue(request: DispatchRequest): Promise<void> {
    await supabase.from('dispatch.requests').insert({
      trip_id: request.tripId,
      origin: request.origin,
      destination: request.destination,
      vehicle_type: request.vehicleType,
      priority: request.priority,
      status: 'pending'
    });

    // Trigger processing
    this.processQueue();
  }

  private async processQueue(): Promise<void> {
    // Get next pending request (highest priority first)
    const { data: request } = await supabase
      .from('dispatch.requests')
      .select('*')
      .eq('status', 'pending')
      .order('priority', { ascending: false })
      .order('requested_at', { ascending: true })
      .limit(1)
      .maybeSingle();

    if (!request) return;

    // Mark as processing
    await supabase
      .from('dispatch.requests')
      .update({ status: 'processing' })
      .eq('id', request.id);

    try {
      await dispatchService.dispatch(request);
    } catch (error) {
      await this.handleFailure(request);
    }
  }

  private async handleFailure(request: DispatchRequest): Promise<void> {
    const retryCount = request.retry_count + 1;

    if (retryCount >= request.max_retries) {
      await supabase
        .from('dispatch.requests')
        .update({
          status: 'failed',
          failed_reason: 'Max retries exceeded'
        })
        .eq('id', request.id);
    } else {
      await supabase
        .from('dispatch.requests')
        .update({
          status: 'pending',
          retry_count: retryCount
        })
        .eq('id', request.id);

      // Retry after delay
      setTimeout(() => this.processQueue(), 5000);
    }
  }
}
```

## Real-Time Subscription Pattern

```typescript
class RealtimeTrackingClient {
  private subscriptions: Map<string, RealtimeChannel> = new Map();

  subscribeToTrip(
    tripId: string,
    callback: (location: LocationUpdate) => void
  ): () => void {
    const channel = supabase
      .channel(`location:trip:${tripId}`)
      .on('broadcast', { event: 'location_update' }, (payload) => {
        callback(payload.payload);
      })
      .subscribe();

    this.subscriptions.set(tripId, channel);

    // Return unsubscribe function
    return () => {
      channel.unsubscribe();
      this.subscriptions.delete(tripId);
    };
  }

  subscribeToDriver(
    driverId: string,
    callback: (location: LocationUpdate) => void
  ): () => void {
    const channel = supabase
      .channel(`location:driver:${driverId}`)
      .on('broadcast', { event: 'location_update' }, (payload) => {
        callback(payload.payload);
      })
      .subscribe();

    this.subscriptions.set(driverId, channel);

    return () => {
      channel.unsubscribe();
      this.subscriptions.delete(driverId);
    };
  }
}
```

# Anti-Patterns

## Synchronous Dispatch

**NEVER DO THIS**:
```typescript
// ❌ Blocking dispatch call
app.post('/trips', async (req, res) => {
  const trip = await tripsService.createTrip(req.body);
  const assignment = await dispatchService.dispatch(trip); // Blocks!
  res.json({ trip, assignment });
});
```

**DO THIS**:
```typescript
// ✅ Asynchronous dispatch
app.post('/trips', async (req, res) => {
  const trip = await tripsService.createTrip(req.body);

  // Dispatch asynchronously
  dispatchQueue.enqueue({ tripId: trip.id });

  res.json({ trip, status: 'dispatch_pending' });
});
```

## Polling for Location Updates

**NEVER DO THIS**:
```typescript
// ❌ Polling every second
setInterval(async () => {
  const location = await api.get(`/drivers/${driverId}/location`);
  updateMap(location);
}, 1000);
```

**DO THIS**:
```typescript
// ✅ Real-time subscription
trackingClient.subscribeToDriver(driverId, (location) => {
  updateMap(location);
});
```

## N+1 Queries in Dispatch

**NEVER DO THIS**:
```typescript
// ❌ N+1 queries
const drivers = await getAvailableDrivers();
for (const driver of drivers) {
  const vehicle = await getVehicle(driver.vehicleId);
  const distance = await calculateDistance(origin, driver.location);
}
```

**DO THIS**:
```typescript
// ✅ Single query with joins
const { data } = await supabase
  .from('drivers')
  .select(`
    *,
    vehicle:vehicles(*),
    location:current_locations(*)
  `)
  .eq('availability_status', 'available');
```

# Observability

## Key Metrics

**Dispatch Metrics**:
- Time to assignment (p50, p95, p99)
- Assignment success rate
- Driver acceptance rate
- Queue depth
- Failed dispatch attempts

**Route Metrics**:
- Route calculation time
- Cache hit rate
- Route deviation from plan
- ETA accuracy

**Trip Metrics**:
- Trip duration by status
- Status transition times
- Completion rate
- Cancellation rate by status

**Tracking Metrics**:
- Location update frequency
- Location accuracy
- Update latency (recorded vs received)
- Geofence event rate

## Logging Strategy

```typescript
logger.info('Dispatch assigned', {
  dispatchRequestId: request.id,
  tripId: request.tripId,
  driverId: assignment.driverId,
  distance: assignment.driverDistance,
  score: assignment.score,
  candidatesEvaluated: candidates.length,
  timeToAssign: Date.now() - request.requestedAt.getTime()
});
```

# Testing Strategy

## Unit Tests

```typescript
describe('DispatchService', () => {
  it('should score closer drivers higher', () => {
    const candidates = [
      { driverId: '1', distance: 5, eta: 10, rating: 4.5 },
      { driverId: '2', distance: 2, eta: 5, rating: 4.0 }
    ];

    const scores = service.scoreCandidates(candidates, request);

    expect(scores[1].totalScore).toBeGreaterThan(scores[0].totalScore);
  });
});
```

## Integration Tests

```typescript
describe('Trip Lifecycle', () => {
  it('should transition through all states successfully', async () => {
    const trip = await tripsService.createTrip(tripData);

    await lifecycleService.transitionTrip(trip.id, 'assign');
    expect(await getTrip(trip.id)).toHaveProperty('status', 'assigned');

    await lifecycleService.transitionTrip(trip.id, 'accept');
    expect(await getTrip(trip.id)).toHaveProperty('status', 'accepted');

    await lifecycleService.transitionTrip(trip.id, 'start');
    expect(await getTrip(trip.id)).toHaveProperty('status', 'in_progress');

    await lifecycleService.transitionTrip(trip.id, 'complete');
    expect(await getTrip(trip.id)).toHaveProperty('status', 'completed');
  });
});
```

# Code Review Checklist

- [ ] Is dispatch logic idempotent?
- [ ] Are location updates batched for performance?
- [ ] Is real-time data published immediately?
- [ ] Are spatial indexes used for location queries?
- [ ] Is route optimization appropriate for dataset size?
- [ ] Are trip state transitions validated?
- [ ] Is geofencing efficient (not checking all geofences)?
- [ ] Are failed dispatches retried appropriately?
- [ ] Is driver availability updated atomically?
- [ ] Are metrics collected for all key operations?
