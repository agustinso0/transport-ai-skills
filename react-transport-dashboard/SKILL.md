---
name: react-transport-dashboard
description: Build React dashboards for trip monitoring, driver management, and vehicle tracking
---

# React Transport Dashboard

# Description

This skill guides the development of React-based dashboards for transportation platforms, focusing on real-time trip monitoring, driver dashboards, vehicle map visualization, and advanced filtering capabilities. It emphasizes performance, real-time data handling, and intuitive UX for transport operations.

# When to Activate

- Building dashboards for trip monitoring and management
- Creating driver-facing interfaces for job management
- Implementing vehicle tracking and map visualization
- Adding filtering, search, and data table functionality
- Displaying real-time location updates and trip status
- Building analytics views for transport operations

# Engineering Principles

1. **Real-time First**: Design components to handle live data updates efficiently
2. **Responsive Design**: Ensure dashboards work across desktop, tablet, and mobile
3. **Progressive Loading**: Load critical data first, defer secondary information
4. **State Management**: Keep component state minimal, derive values when possible
5. **Accessibility**: Ensure keyboard navigation and screen reader support
6. **Error Resilience**: Handle network failures and stale data gracefully

# Implementation Patterns

## Component Organization

```
src/
  components/
    dashboard/
      TripMonitor/
      DriverDashboard/
      VehicleMap/
    shared/
      DataTable/
      FilterPanel/
      SearchBar/
  hooks/
    useRealtimeTrips.ts
    useVehicleLocations.ts
  services/
    tripService.ts
    locationService.ts
  types/
    trip.ts
    driver.ts
    vehicle.ts
```

## Real-time Data Pattern

Use Supabase real-time subscriptions for live updates:

```typescript
import { useEffect, useState } from 'react';
import { supabase } from '../lib/supabase';

export function useRealtimeTrips(status?: string) {
  const [trips, setTrips] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const fetchTrips = async () => {
      const query = supabase
        .from('trips')
        .select('*, driver:drivers(*), vehicle:vehicles(*)');

      if (status) {
        query.eq('status', status);
      }

      const { data, error } = await query;
      if (!error && data) {
        setTrips(data);
      }
      setLoading(false);
    };

    fetchTrips();

    const subscription = supabase
      .channel('trips-changes')
      .on('postgres_changes',
        { event: '*', schema: 'public', table: 'trips' },
        (payload) => {
          if (payload.eventType === 'INSERT') {
            setTrips(prev => [...prev, payload.new]);
          } else if (payload.eventType === 'UPDATE') {
            setTrips(prev => prev.map(t =>
              t.id === payload.new.id ? payload.new : t
            ));
          } else if (payload.eventType === 'DELETE') {
            setTrips(prev => prev.filter(t => t.id !== payload.old.id));
          }
        }
      )
      .subscribe();

    return () => {
      subscription.unsubscribe();
    };
  }, [status]);

  return { trips, loading };
}
```

## Filter and Search Pattern

```typescript
interface FilterState {
  status?: string;
  driverId?: string;
  dateFrom?: string;
  dateTo?: string;
  search?: string;
}

export function TripMonitor() {
  const [filters, setFilters] = useState<FilterState>({});
  const { trips, loading } = useRealtimeTrips();

  const filteredTrips = useMemo(() => {
    return trips.filter(trip => {
      if (filters.status && trip.status !== filters.status) return false;
      if (filters.driverId && trip.driver_id !== filters.driverId) return false;
      if (filters.search) {
        const searchLower = filters.search.toLowerCase();
        return (
          trip.id.toLowerCase().includes(searchLower) ||
          trip.driver?.name.toLowerCase().includes(searchLower) ||
          trip.pickup_address.toLowerCase().includes(searchLower)
        );
      }
      return true;
    });
  }, [trips, filters]);

  return (
    <div className="space-y-4">
      <FilterPanel filters={filters} onChange={setFilters} />
      <TripTable trips={filteredTrips} loading={loading} />
    </div>
  );
}
```

# Code Examples

## Map Visualization Component

```typescript
import { useEffect, useRef } from 'react';
import { MapPin } from 'lucide-react';

interface Vehicle {
  id: string;
  plate: string;
  latitude: number;
  longitude: number;
  status: 'active' | 'idle' | 'offline';
}

export function VehicleMap({ vehicles }: { vehicles: Vehicle[] }) {
  const mapRef = useRef<HTMLDivElement>(null);

  return (
    <div className="relative h-[600px] bg-gray-100 rounded-lg overflow-hidden">
      <div ref={mapRef} className="absolute inset-0">
        {vehicles.map(vehicle => (
          <VehicleMarker key={vehicle.id} vehicle={vehicle} />
        ))}
      </div>
    </div>
  );
}

function VehicleMarker({ vehicle }: { vehicle: Vehicle }) {
  const statusColors = {
    active: 'text-green-600',
    idle: 'text-yellow-600',
    offline: 'text-gray-400'
  };

  return (
    <div
      className="absolute transform -translate-x-1/2 -translate-y-1/2 cursor-pointer"
      style={{
        left: `${vehicle.longitude}%`,
        top: `${vehicle.latitude}%`
      }}
    >
      <MapPin className={`w-6 h-6 ${statusColors[vehicle.status]}`} />
      <span className="text-xs font-medium">{vehicle.plate}</span>
    </div>
  );
}
```

## Driver Dashboard

```typescript
import { useEffect, useState } from 'react';
import { supabase } from '../lib/supabase';
import { Clock, MapPin, DollarSign } from 'lucide-react';

interface DriverStats {
  activeTrips: number;
  completedToday: number;
  earningsToday: number;
  hoursOnline: number;
}

export function DriverDashboard({ driverId }: { driverId: string }) {
  const [stats, setStats] = useState<DriverStats | null>(null);
  const [currentTrip, setCurrentTrip] = useState(null);

  useEffect(() => {
    const fetchStats = async () => {
      const today = new Date().toISOString().split('T')[0];

      const { data: trips } = await supabase
        .from('trips')
        .select('*')
        .eq('driver_id', driverId)
        .gte('created_at', today);

      const activeTrips = trips?.filter(t => t.status === 'active').length || 0;
      const completedToday = trips?.filter(t => t.status === 'completed').length || 0;
      const earningsToday = trips
        ?.filter(t => t.status === 'completed')
        .reduce((sum, t) => sum + (t.fare || 0), 0) || 0;

      setStats({
        activeTrips,
        completedToday,
        earningsToday,
        hoursOnline: 0
      });
    };

    fetchStats();
  }, [driverId]);

  if (!stats) return <div>Loading...</div>;

  return (
    <div className="space-y-6">
      <div className="grid grid-cols-1 md:grid-cols-4 gap-4">
        <StatCard
          icon={<Clock className="w-5 h-5" />}
          label="Hours Online"
          value={`${stats.hoursOnline}h`}
        />
        <StatCard
          icon={<MapPin className="w-5 h-5" />}
          label="Active Trips"
          value={stats.activeTrips}
        />
        <StatCard
          icon={<MapPin className="w-5 h-5" />}
          label="Completed Today"
          value={stats.completedToday}
        />
        <StatCard
          icon={<DollarSign className="w-5 h-5" />}
          label="Earnings Today"
          value={`$${stats.earningsToday.toFixed(2)}`}
        />
      </div>

      {currentTrip && <CurrentTripCard trip={currentTrip} />}
    </div>
  );
}

function StatCard({ icon, label, value }) {
  return (
    <div className="bg-white p-4 rounded-lg shadow">
      <div className="flex items-center gap-2 text-gray-600 mb-2">
        {icon}
        <span className="text-sm font-medium">{label}</span>
      </div>
      <div className="text-2xl font-bold">{value}</div>
    </div>
  );
}
```

# Anti-Patterns

❌ **Fetching all data on mount**
```typescript
useEffect(() => {
  supabase.from('trips').select('*').then(({ data }) => setTrips(data));
}, []);
```

✅ **Use pagination and filtering**
```typescript
const { data } = await supabase
  .from('trips')
  .select('*')
  .eq('status', 'active')
  .order('created_at', { ascending: false })
  .limit(50);
```

❌ **Polling for updates**
```typescript
setInterval(() => fetchTrips(), 5000);
```

✅ **Use real-time subscriptions**
```typescript
supabase.channel('trips').on('postgres_changes', ...).subscribe();
```

❌ **Filtering in component render**
```typescript
return trips.filter(t => t.status === 'active').map(...);
```

✅ **Use useMemo for expensive operations**
```typescript
const activeTrips = useMemo(() =>
  trips.filter(t => t.status === 'active'),
  [trips]
);
```

# Performance Guidelines

1. **Virtualize Long Lists**: Use virtual scrolling for >100 items
2. **Debounce Search**: Wait 300ms after user stops typing
3. **Memoize Calculations**: Use `useMemo` for derived data
4. **Lazy Load Maps**: Only render map when tab is active
5. **Batch Updates**: Group multiple state updates together
6. **Code Split**: Lazy load dashboard routes

```typescript
const DriverDashboard = lazy(() => import('./components/DriverDashboard'));
const VehicleMap = lazy(() => import('./components/VehicleMap'));
```

# Observability

## Error Boundaries

```typescript
class DashboardErrorBoundary extends React.Component {
  state = { hasError: false };

  static getDerivedStateFromError(error) {
    return { hasError: true };
  }

  componentDidCatch(error, errorInfo) {
    console.error('Dashboard error:', error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      return <ErrorFallback onReset={() => window.location.reload()} />;
    }
    return this.props.children;
  }
}
```

## Performance Monitoring

```typescript
useEffect(() => {
  const start = performance.now();

  fetchDashboardData().then(() => {
    const duration = performance.now() - start;
    if (duration > 1000) {
      console.warn(`Dashboard loaded slowly: ${duration}ms`);
    }
  });
}, []);
```

# Testing Strategy

## Component Testing

```typescript
import { render, screen, waitFor } from '@testing-library/react';
import { TripMonitor } from './TripMonitor';

jest.mock('../hooks/useRealtimeTrips', () => ({
  useRealtimeTrips: () => ({
    trips: [
      { id: '1', status: 'active', driver: { name: 'John' } },
      { id: '2', status: 'completed', driver: { name: 'Jane' } }
    ],
    loading: false
  })
}));

test('displays active trips', async () => {
  render(<TripMonitor />);

  await waitFor(() => {
    expect(screen.getByText('John')).toBeInTheDocument();
  });
});
```

## Integration Testing

```typescript
test('filters trips by status', async () => {
  const { user } = render(<TripMonitor />);

  const statusFilter = screen.getByLabelText('Status');
  await user.selectOptions(statusFilter, 'active');

  expect(screen.getByText('John')).toBeInTheDocument();
  expect(screen.queryByText('Jane')).not.toBeInTheDocument();
});
```

## E2E Testing

```typescript
test('driver can view and update current trip', async () => {
  await page.goto('/driver/dashboard');

  await page.waitForSelector('[data-testid="current-trip"]');
  await page.click('[data-testid="complete-trip"]');

  await expect(page.locator('.success-message')).toBeVisible();
});
```
