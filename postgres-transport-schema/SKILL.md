---
name: postgres-transport-schema
description: PostgreSQL schema design for transportation platforms with drivers, vehicles, trips, routes, and events
---

# PostgreSQL Transport Schema

# Description

Comprehensive database schema design for transportation platforms handling real-time driver dispatch, vehicle tracking, trip management, and route optimization. Designed for high-concurrency workloads with millions of trips and location updates.

# When to Activate

- Building ride-sharing, delivery, or logistics platforms
- Managing fleet operations with real-time tracking
- Implementing driver dispatch systems
- Tracking trip lifecycle and events
- Optimizing route planning and analytics

# Database Design

## Core Tables

### drivers
```sql
CREATE TABLE drivers (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid REFERENCES auth.users(id) ON DELETE CASCADE,
  first_name text NOT NULL,
  last_name text NOT NULL,
  phone text NOT NULL UNIQUE,
  email text NOT NULL UNIQUE,
  license_number text NOT NULL UNIQUE,
  license_expiry date NOT NULL,
  status text NOT NULL DEFAULT 'offline' CHECK (status IN ('online', 'offline', 'on_trip', 'suspended')),
  rating numeric(3,2) DEFAULT 5.00 CHECK (rating >= 0 AND rating <= 5),
  total_trips integer DEFAULT 0,
  current_location geography(POINT, 4326),
  last_location_update timestamptz,
  created_at timestamptz DEFAULT now(),
  updated_at timestamptz DEFAULT now()
);
```

### vehicles
```sql
CREATE TABLE vehicles (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  driver_id uuid REFERENCES drivers(id) ON DELETE SET NULL,
  make text NOT NULL,
  model text NOT NULL,
  year integer NOT NULL CHECK (year >= 1900 AND year <= EXTRACT(YEAR FROM CURRENT_DATE) + 1),
  color text NOT NULL,
  license_plate text NOT NULL UNIQUE,
  vin text NOT NULL UNIQUE,
  vehicle_type text NOT NULL CHECK (vehicle_type IN ('sedan', 'suv', 'van', 'truck', 'motorcycle', 'bicycle')),
  capacity integer NOT NULL DEFAULT 4 CHECK (capacity > 0),
  status text NOT NULL DEFAULT 'inactive' CHECK (status IN ('active', 'inactive', 'maintenance', 'retired')),
  last_inspection date,
  insurance_expiry date NOT NULL,
  created_at timestamptz DEFAULT now(),
  updated_at timestamptz DEFAULT now()
);
```

### routes
```sql
CREATE TABLE routes (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  name text NOT NULL,
  origin geography(POINT, 4326) NOT NULL,
  destination geography(POINT, 4326) NOT NULL,
  waypoints jsonb DEFAULT '[]',
  distance_meters numeric NOT NULL CHECK (distance_meters >= 0),
  estimated_duration_seconds integer NOT NULL CHECK (estimated_duration_seconds >= 0),
  route_geometry geography(LINESTRING, 4326),
  traffic_multiplier numeric DEFAULT 1.0 CHECK (traffic_multiplier >= 0.5 AND traffic_multiplier <= 3.0),
  created_at timestamptz DEFAULT now()
);
```

### trips
```sql
CREATE TABLE trips (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  driver_id uuid REFERENCES drivers(id) ON DELETE SET NULL,
  vehicle_id uuid REFERENCES vehicles(id) ON DELETE SET NULL,
  route_id uuid REFERENCES routes(id) ON DELETE SET NULL,
  passenger_id uuid REFERENCES auth.users(id) ON DELETE SET NULL,
  status text NOT NULL DEFAULT 'requested' CHECK (status IN (
    'requested', 'searching', 'accepted', 'arriving',
    'in_progress', 'completed', 'cancelled', 'failed'
  )),
  pickup_location geography(POINT, 4326) NOT NULL,
  dropoff_location geography(POINT, 4326) NOT NULL,
  pickup_address text NOT NULL,
  dropoff_address text NOT NULL,
  scheduled_pickup_time timestamptz,
  actual_pickup_time timestamptz,
  actual_dropoff_time timestamptz,
  distance_meters numeric,
  duration_seconds integer,
  fare_amount numeric(10,2),
  currency text DEFAULT 'USD',
  payment_status text DEFAULT 'pending' CHECK (payment_status IN ('pending', 'paid', 'failed', 'refunded')),
  rating integer CHECK (rating >= 1 AND rating <= 5),
  feedback text,
  created_at timestamptz DEFAULT now(),
  updated_at timestamptz DEFAULT now()
);
```

### trip_events
```sql
CREATE TABLE trip_events (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  trip_id uuid REFERENCES trips(id) ON DELETE CASCADE NOT NULL,
  event_type text NOT NULL CHECK (event_type IN (
    'trip_requested', 'driver_assigned', 'driver_arrived',
    'trip_started', 'trip_completed', 'trip_cancelled',
    'location_update', 'route_deviation', 'delay_detected'
  )),
  location geography(POINT, 4326),
  metadata jsonb DEFAULT '{}',
  created_at timestamptz DEFAULT now()
);
```

# Indexing Strategies

## Spatial Indexes
```sql
CREATE INDEX idx_drivers_location ON drivers USING GIST (current_location);
CREATE INDEX idx_trips_pickup_location ON trips USING GIST (pickup_location);
CREATE INDEX idx_trips_dropoff_location ON trips USING GIST (dropoff_location);
CREATE INDEX idx_routes_geometry ON routes USING GIST (route_geometry);
```

## Status and Temporal Indexes
```sql
CREATE INDEX idx_drivers_status ON drivers (status) WHERE status = 'online';
CREATE INDEX idx_trips_status ON trips (status) WHERE status IN ('requested', 'searching', 'accepted');
CREATE INDEX idx_trips_created_at ON trips (created_at DESC);
CREATE INDEX idx_trip_events_trip_id_created_at ON trip_events (trip_id, created_at DESC);
```

## Composite Indexes for Common Queries
```sql
CREATE INDEX idx_trips_driver_status ON trips (driver_id, status, created_at DESC);
CREATE INDEX idx_trips_passenger_status ON trips (passenger_id, status, created_at DESC);
CREATE INDEX idx_vehicles_driver_status ON vehicles (driver_id, status);
```

# Query Optimization

## Finding Nearest Available Drivers
```sql
SELECT
  d.id,
  d.first_name,
  d.last_name,
  ST_Distance(d.current_location, ST_GeogFromText('POINT(-122.4194 37.7749)')) as distance_meters
FROM drivers d
WHERE
  d.status = 'online'
  AND d.current_location IS NOT NULL
  AND ST_DWithin(
    d.current_location,
    ST_GeogFromText('POINT(-122.4194 37.7749)'),
    5000
  )
ORDER BY distance_meters
LIMIT 10;
```

## Trip Analytics with Aggregations
```sql
SELECT
  DATE_TRUNC('hour', created_at) as hour,
  status,
  COUNT(*) as trip_count,
  AVG(fare_amount) as avg_fare,
  AVG(duration_seconds) as avg_duration
FROM trips
WHERE created_at >= NOW() - INTERVAL '24 hours'
GROUP BY hour, status
ORDER BY hour DESC;
```

## Driver Performance Metrics
```sql
SELECT
  d.id,
  d.first_name,
  d.last_name,
  d.rating,
  COUNT(t.id) as completed_trips,
  AVG(t.fare_amount) as avg_fare,
  SUM(t.fare_amount) as total_earnings
FROM drivers d
LEFT JOIN trips t ON t.driver_id = d.id AND t.status = 'completed'
WHERE t.created_at >= NOW() - INTERVAL '30 days'
GROUP BY d.id, d.first_name, d.last_name, d.rating
ORDER BY completed_trips DESC;
```

# Anti-Patterns

## ❌ Avoid
- Storing location as separate lat/lng columns instead of geography type
- Missing indexes on status columns for filtering
- Not using partial indexes for active records
- Storing route geometry as JSON instead of PostGIS types
- Not partitioning trip_events by time range
- Using SERIAL instead of UUID for distributed systems
- Missing ON DELETE CASCADE/SET NULL on foreign keys
- Not validating status transitions in application layer

## ✅ Do
- Use PostGIS geography type for location data
- Create partial indexes on frequently filtered statuses
- Partition large tables (trips, trip_events) by time
- Use UUIDs for primary keys in distributed environments
- Implement proper foreign key constraints
- Use CHECK constraints for status validation
- Add triggers for updated_at timestamps
- Use EXPLAIN ANALYZE for query optimization

# Performance Guidelines

## Table Partitioning
```sql
CREATE TABLE trips_partitioned (
  LIKE trips INCLUDING ALL
) PARTITION BY RANGE (created_at);

CREATE TABLE trips_2024_01 PARTITION OF trips_partitioned
  FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
```

## Updated Timestamp Trigger
```sql
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_drivers_updated_at
  BEFORE UPDATE ON drivers
  FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

## Materialized Views for Analytics
```sql
CREATE MATERIALIZED VIEW driver_stats AS
SELECT
  d.id,
  d.first_name,
  d.last_name,
  COUNT(t.id) as total_trips,
  AVG(t.rating) as avg_rating,
  SUM(t.fare_amount) as total_earnings
FROM drivers d
LEFT JOIN trips t ON t.driver_id = d.id AND t.status = 'completed'
GROUP BY d.id, d.first_name, d.last_name;

CREATE UNIQUE INDEX ON driver_stats (id);
```

# Observability

## Query Performance Monitoring
```sql
SELECT
  query,
  calls,
  total_exec_time,
  mean_exec_time,
  max_exec_time
FROM pg_stat_statements
WHERE query LIKE '%trips%'
ORDER BY mean_exec_time DESC
LIMIT 10;
```

## Index Usage Statistics
```sql
SELECT
  schemaname,
  tablename,
  indexname,
  idx_scan,
  idx_tup_read,
  idx_tup_fetch
FROM pg_stat_user_indexes
WHERE schemaname = 'public'
ORDER BY idx_scan DESC;
```

# Testing Strategy

## Data Integrity Tests
- Verify foreign key constraints prevent orphaned records
- Test CHECK constraints reject invalid status values
- Validate geography type stores coordinates correctly
- Ensure triggers update timestamps properly

## Performance Tests
- Benchmark nearest driver queries under load
- Test trip creation throughput (target: 1000+ req/s)
- Validate index usage with EXPLAIN ANALYZE
- Measure query performance with 10M+ trips

## Spatial Query Tests
- Verify ST_DWithin returns correct nearby drivers
- Test route geometry calculations
- Validate distance calculations accuracy
- Ensure spatial indexes are utilized

## Data Migration Tests
- Test partition creation for historical data
- Validate data integrity during migrations
- Ensure zero downtime during schema changes
- Verify rollback procedures work correctly
