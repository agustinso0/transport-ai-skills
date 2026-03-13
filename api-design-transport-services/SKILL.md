---
name: api-design-transport-services
description: REST API design standards for transportation services with pagination, filtering, and versioning
---

# API Design Transport Services

Comprehensive REST API design patterns and best practices for transportation management platforms, covering resource modeling, endpoint design, pagination, filtering, versioning, and error handling.

# Description

This skill provides detailed guidance for designing RESTful APIs for transportation systems. It covers how to model resources specific to the transportation domain (drivers, vehicles, trips, routes), implement efficient pagination and filtering for large datasets, version APIs to support evolution, and provide consistent, helpful error responses. The skill emphasizes developer experience, performance, and maintainability.

# When to Activate

Activate this skill when:

- Designing REST API endpoints for transportation services
- Implementing pagination for list endpoints
- Adding filtering and search capabilities to APIs
- Planning API versioning strategy
- Standardizing error response formats
- Designing resource models for drivers, vehicles, trips, and routes
- Creating API documentation
- Implementing rate limiting and throttling
- Designing query parameters for complex searches
- Building webhooks for third-party integrations

# Engineering Principles

## RESTful Resource Modeling

Model APIs around resources, not actions:
- Use nouns for endpoints: `/drivers`, `/trips`, `/vehicles`
- Use HTTP methods for actions: GET, POST, PUT, PATCH, DELETE
- Nest resources logically: `/trips/{id}/events`, `/drivers/{id}/documents`

## Consistency

APIs should be predictable:
- Same patterns across all endpoints
- Consistent naming conventions
- Uniform error response format
- Standard pagination approach

## Developer Experience

Design for the API consumer:
- Self-documenting URLs
- Helpful error messages
- Comprehensive documentation
- Example requests and responses

## Performance

Optimize for common use cases:
- Pagination by default
- Selective field loading
- Efficient filtering
- Response compression

## Versioning

Plan for evolution:
- Version from day one
- Backward compatibility when possible
- Clear deprecation policy
- Migration guides for breaking changes

# Domain Modeling

## Resource Models

### Driver Resource

```typescript
interface Driver {
  id: string;
  userId: string;
  status: 'active' | 'inactive' | 'suspended';
  availabilityStatus: 'available' | 'on_trip' | 'offline';
  profile: {
    firstName: string;
    lastName: string;
    phone: string;
    email: string;
    photoUrl: string | null;
  };
  license: {
    number: string;
    expiryDate: string;
    verified: boolean;
  };
  rating: {
    average: number;
    count: number;
  };
  stats: {
    totalTrips: number;
    totalDistance: number;
    completionRate: number;
  };
  currentLocation: {
    lat: number;
    lng: number;
    accuracy: number;
    updatedAt: string;
  } | null;
  createdAt: string;
  updatedAt: string;
}
```

### Vehicle Resource

```typescript
interface Vehicle {
  id: string;
  licensePlate: string;
  make: string;
  model: string;
  year: number;
  color: string;
  vehicleType: 'sedan' | 'suv' | 'van' | 'truck';
  capacity: {
    passengers: number;
    cargo: number;
  };
  status: 'active' | 'maintenance' | 'retired';
  currentDriver: {
    id: string;
    name: string;
  } | null;
  features: string[];
  documents: {
    registration: DocumentInfo;
    insurance: DocumentInfo;
    inspection: DocumentInfo;
  };
  createdAt: string;
  updatedAt: string;
}

interface DocumentInfo {
  verified: boolean;
  expiryDate: string;
  documentUrl: string | null;
}
```

### Trip Resource

```typescript
interface Trip {
  id: string;
  status: TripStatus;
  route: {
    origin: Location;
    destination: Location;
    waypoints: Location[];
    distance: number;
    estimatedDuration: number;
  };
  driver: {
    id: string;
    name: string;
    phone: string;
    rating: number;
  } | null;
  vehicle: {
    id: string;
    make: string;
    model: string;
    licensePlate: string;
    color: string;
  } | null;
  schedule: {
    requestedAt: string;
    scheduledStart: string;
    estimatedEnd: string;
    actualStart: string | null;
    actualEnd: string | null;
  };
  progress: {
    currentLocation: Location | null;
    distanceTraveled: number;
    percentComplete: number;
  };
  pricing: {
    currency: string;
    basefare: number;
    distanceCharge: number;
    timeCharge: number;
    total: number;
  };
  metadata: Record<string, any>;
  createdAt: string;
  updatedAt: string;
}

type TripStatus =
  | 'pending'
  | 'assigned'
  | 'accepted'
  | 'driver_arriving'
  | 'arrived'
  | 'in_progress'
  | 'completed'
  | 'cancelled';

interface Location {
  lat: number;
  lng: number;
  address: string;
  placeId: string | null;
}
```

### Route Resource

```typescript
interface Route {
  id: string;
  name: string;
  description: string;
  origin: Location;
  destination: Location;
  waypoints: Waypoint[];
  geometry: {
    polyline: string;
    bounds: {
      northeast: { lat: number; lng: number };
      southwest: { lat: number; lng: number };
    };
  };
  summary: {
    totalDistance: number;
    estimatedDuration: number;
    trafficDelay: number;
  };
  segments: RouteSegment[];
  createdAt: string;
  updatedAt: string;
}

interface Waypoint {
  sequence: number;
  location: Location;
  stopDuration: number;
  instructions: string;
}

interface RouteSegment {
  sequence: number;
  startPoint: Location;
  endPoint: Location;
  distance: number;
  duration: number;
  instructions: string[];
}
```

## Collection Responses

All list endpoints should use consistent wrapper:

```typescript
interface CollectionResponse<T> {
  data: T[];
  pagination: {
    page: number;
    pageSize: number;
    totalCount: number;
    totalPages: number;
    hasNext: boolean;
    hasPrevious: boolean;
  };
  links: {
    self: string;
    first: string;
    last: string;
    next: string | null;
    previous: string | null;
  };
}
```

## Error Responses

```typescript
interface ErrorResponse {
  error: {
    code: string;
    message: string;
    details: Record<string, any> | null;
    timestamp: string;
    path: string;
    requestId: string;
  };
}

interface ValidationErrorResponse {
  error: {
    code: 'validation_error';
    message: string;
    fields: Array<{
      field: string;
      message: string;
      value: any;
    }>;
    timestamp: string;
    path: string;
    requestId: string;
  };
}
```

# Architecture Guidelines

## URL Structure

### Versioning in URL

```
/api/v1/drivers
/api/v1/vehicles
/api/v1/trips
/api/v1/routes
```

### Resource Hierarchy

```
# Top-level resources
GET    /api/v1/drivers
GET    /api/v1/drivers/{id}
POST   /api/v1/drivers
PATCH  /api/v1/drivers/{id}
DELETE /api/v1/drivers/{id}

# Nested resources
GET    /api/v1/drivers/{id}/documents
POST   /api/v1/drivers/{id}/documents
GET    /api/v1/drivers/{id}/trips
GET    /api/v1/drivers/{id}/stats

# Actions (use sparingly)
POST   /api/v1/trips/{id}/assign
POST   /api/v1/trips/{id}/start
POST   /api/v1/trips/{id}/complete
POST   /api/v1/trips/{id}/cancel
```

### Query Parameters

```
# Pagination
GET /api/v1/trips?page=2&pageSize=50

# Filtering
GET /api/v1/trips?status=in_progress
GET /api/v1/drivers?availabilityStatus=available
GET /api/v1/vehicles?vehicleType=sedan&status=active

# Sorting
GET /api/v1/trips?sort=-createdAt
GET /api/v1/drivers?sort=rating,-totalTrips

# Field selection
GET /api/v1/trips?fields=id,status,driver,route.origin

# Search
GET /api/v1/drivers?search=john
GET /api/v1/vehicles?search=TOY

# Geographic queries
GET /api/v1/drivers?near=37.7749,-122.4194&radius=5000
GET /api/v1/trips?bounds=37.7,-122.5,37.8,-122.3

# Date ranges
GET /api/v1/trips?startDate=2024-01-01&endDate=2024-01-31
GET /api/v1/trips?createdAfter=2024-01-01T00:00:00Z

# Relationships
GET /api/v1/trips?include=driver,vehicle,route
```

## HTTP Methods

### GET - Retrieve Resources

```typescript
// Get single resource
GET /api/v1/trips/123
Response: 200 OK
{
  "data": { /* trip object */ }
}

// Get collection
GET /api/v1/trips
Response: 200 OK
{
  "data": [ /* array of trips */ ],
  "pagination": { /* pagination info */ }
}

// Resource not found
GET /api/v1/trips/invalid
Response: 404 Not Found
{
  "error": {
    "code": "resource_not_found",
    "message": "Trip not found"
  }
}
```

### POST - Create Resources

```typescript
// Create resource
POST /api/v1/trips
Content-Type: application/json
{
  "route": {
    "origin": { "lat": 37.7749, "lng": -122.4194 },
    "destination": { "lat": 37.8044, "lng": -122.2712 }
  },
  "scheduledStart": "2024-01-15T10:00:00Z"
}

Response: 201 Created
Location: /api/v1/trips/123
{
  "data": { /* created trip object */ }
}

// Validation error
Response: 400 Bad Request
{
  "error": {
    "code": "validation_error",
    "message": "Invalid request data",
    "fields": [
      {
        "field": "route.origin.lat",
        "message": "Latitude must be between -90 and 90",
        "value": 100
      }
    ]
  }
}
```

### PATCH - Partial Update

```typescript
// Update resource
PATCH /api/v1/drivers/123
Content-Type: application/json
{
  "availabilityStatus": "offline"
}

Response: 200 OK
{
  "data": { /* updated driver object */ }
}

// Conflict
Response: 409 Conflict
{
  "error": {
    "code": "conflict",
    "message": "Driver is currently on a trip and cannot be set offline"
  }
}
```

### DELETE - Remove Resources

```typescript
// Delete resource
DELETE /api/v1/vehicles/123

Response: 204 No Content

// Cannot delete (has dependencies)
Response: 409 Conflict
{
  "error": {
    "code": "conflict",
    "message": "Vehicle cannot be deleted while assigned to active trips"
  }
}
```

## Pagination

### Cursor-Based Pagination (Recommended for Real-Time Data)

```typescript
interface CursorPaginationParams {
  cursor?: string;
  limit: number;
}

interface CursorPaginationResponse<T> {
  data: T[];
  pagination: {
    nextCursor: string | null;
    previousCursor: string | null;
    hasNext: boolean;
    hasPrevious: boolean;
    limit: number;
  };
}

// Request
GET /api/v1/trips?limit=50&cursor=eyJpZCI6MTIzfQ==

// Response
{
  "data": [ /* trips */ ],
  "pagination": {
    "nextCursor": "eyJpZCI6MTczfQ==",
    "previousCursor": null,
    "hasNext": true,
    "hasPrevious": false,
    "limit": 50
  }
}
```

### Offset-Based Pagination (For Static Reports)

```typescript
interface OffsetPaginationParams {
  page: number;
  pageSize: number;
}

// Request
GET /api/v1/trips?page=2&pageSize=50

// Response
{
  "data": [ /* trips */ ],
  "pagination": {
    "page": 2,
    "pageSize": 50,
    "totalCount": 1234,
    "totalPages": 25,
    "hasNext": true,
    "hasPrevious": true
  },
  "links": {
    "self": "/api/v1/trips?page=2&pageSize=50",
    "first": "/api/v1/trips?page=1&pageSize=50",
    "last": "/api/v1/trips?page=25&pageSize=50",
    "next": "/api/v1/trips?page=3&pageSize=50",
    "previous": "/api/v1/trips?page=1&pageSize=50"
  }
}
```

## Filtering

### Simple Filters

```typescript
// Status filter
GET /api/v1/trips?status=in_progress

// Multiple values (OR)
GET /api/v1/trips?status=in_progress,completed

// Multiple filters (AND)
GET /api/v1/trips?status=completed&driverId=123
```

### Comparison Operators

```typescript
// Greater than
GET /api/v1/drivers?totalTrips[gte]=100

// Less than
GET /api/v1/drivers?rating[lte]=3.5

// Date ranges
GET /api/v1/trips?createdAt[gte]=2024-01-01&createdAt[lte]=2024-01-31

// Between
GET /api/v1/trips?distance[between]=5,20
```

### Geographic Filters

```typescript
// Near point with radius
GET /api/v1/drivers?near=37.7749,-122.4194&radius=5000&radiusUnit=meters

// Within bounding box
GET /api/v1/trips?bounds=37.7,-122.5,37.8,-122.3

// Within polygon
POST /api/v1/search/drivers
{
  "polygon": [
    [37.7, -122.5],
    [37.8, -122.5],
    [37.8, -122.3],
    [37.7, -122.3],
    [37.7, -122.5]
  ]
}
```

### Text Search

```typescript
// Simple search across multiple fields
GET /api/v1/drivers?search=john

// Field-specific search
GET /api/v1/vehicles?licensePlate[contains]=ABC

// Case-insensitive search
GET /api/v1/drivers?email[icontains]=@gmail.com
```

## Sorting

```typescript
// Single field ascending
GET /api/v1/trips?sort=createdAt

// Single field descending
GET /api/v1/trips?sort=-createdAt

// Multiple fields
GET /api/v1/drivers?sort=rating,-totalTrips,lastName
```

## Field Selection

```typescript
// Select specific fields
GET /api/v1/trips?fields=id,status,driver.name,route.origin

Response:
{
  "data": [
    {
      "id": "123",
      "status": "in_progress",
      "driver": {
        "name": "John Doe"
      },
      "route": {
        "origin": { "lat": 37.7749, "lng": -122.4194 }
      }
    }
  ]
}

// Exclude fields
GET /api/v1/drivers?exclude=metadata,stats
```

## Relationship Inclusion

```typescript
// Include related resources
GET /api/v1/trips?include=driver,vehicle

Response:
{
  "data": {
    "id": "123",
    "status": "in_progress",
    "driverId": "456",
    "vehicleId": "789",
    "driver": {
      "id": "456",
      "name": "John Doe",
      /* full driver object */
    },
    "vehicle": {
      "id": "789",
      "make": "Toyota",
      /* full vehicle object */
    }
  }
}

// Nested includes
GET /api/v1/trips?include=driver.vehicle,route.waypoints
```

# Implementation Patterns

## Base Controller Pattern

```typescript
abstract class BaseController<T> {
  protected abstract service: BaseService<T>;

  async list(req: Request, res: Response): Promise<void> {
    try {
      const params = this.parsePaginationParams(req);
      const filters = this.parseFilters(req);
      const sort = this.parseSort(req);

      const result = await this.service.list({
        ...params,
        filters,
        sort
      });

      res.json({
        data: result.data,
        pagination: result.pagination,
        links: this.generatePaginationLinks(req, result.pagination)
      });
    } catch (error) {
      this.handleError(error, res);
    }
  }

  async get(req: Request, res: Response): Promise<void> {
    try {
      const id = req.params.id;
      const include = this.parseInclude(req);

      const item = await this.service.get(id, { include });

      if (!item) {
        res.status(404).json({
          error: {
            code: 'resource_not_found',
            message: `${this.resourceName} not found`,
            timestamp: new Date().toISOString(),
            path: req.path,
            requestId: req.id
          }
        });
        return;
      }

      res.json({ data: item });
    } catch (error) {
      this.handleError(error, res);
    }
  }

  async create(req: Request, res: Response): Promise<void> {
    try {
      const data = req.body;
      const errors = await this.validate(data);

      if (errors.length > 0) {
        res.status(400).json({
          error: {
            code: 'validation_error',
            message: 'Invalid request data',
            fields: errors,
            timestamp: new Date().toISOString(),
            path: req.path,
            requestId: req.id
          }
        });
        return;
      }

      const item = await this.service.create(data);

      res.status(201)
        .location(`${req.baseUrl}/${item.id}`)
        .json({ data: item });
    } catch (error) {
      this.handleError(error, res);
    }
  }

  async update(req: Request, res: Response): Promise<void> {
    try {
      const id = req.params.id;
      const data = req.body;

      const item = await this.service.update(id, data);

      if (!item) {
        res.status(404).json({
          error: {
            code: 'resource_not_found',
            message: `${this.resourceName} not found`
          }
        });
        return;
      }

      res.json({ data: item });
    } catch (error) {
      this.handleError(error, res);
    }
  }

  protected parsePaginationParams(req: Request): PaginationParams {
    const page = parseInt(req.query.page as string) || 1;
    const pageSize = Math.min(
      parseInt(req.query.pageSize as string) || 50,
      100 // Max page size
    );

    return { page, pageSize };
  }

  protected parseFilters(req: Request): Record<string, any> {
    const filters: Record<string, any> = {};

    for (const [key, value] of Object.entries(req.query)) {
      if (['page', 'pageSize', 'sort', 'fields', 'include'].includes(key)) {
        continue;
      }

      filters[key] = value;
    }

    return filters;
  }

  protected parseSort(req: Request): string[] {
    const sort = req.query.sort as string;
    if (!sort) return [];

    return sort.split(',').map(s => s.trim());
  }

  protected parseInclude(req: Request): string[] {
    const include = req.query.include as string;
    if (!include) return [];

    return include.split(',').map(i => i.trim());
  }

  protected generatePaginationLinks(
    req: Request,
    pagination: PaginationInfo
  ): PaginationLinks {
    const baseUrl = `${req.protocol}://${req.get('host')}${req.baseUrl}${req.path}`;
    const queryParams = new URLSearchParams(req.query as any);

    const buildUrl = (page: number): string => {
      queryParams.set('page', page.toString());
      return `${baseUrl}?${queryParams.toString()}`;
    };

    return {
      self: buildUrl(pagination.page),
      first: buildUrl(1),
      last: buildUrl(pagination.totalPages),
      next: pagination.hasNext ? buildUrl(pagination.page + 1) : null,
      previous: pagination.hasPrevious ? buildUrl(pagination.page - 1) : null
    };
  }

  protected abstract get resourceName(): string;
  protected abstract validate(data: any): Promise<ValidationError[]>;
  protected abstract handleError(error: any, res: Response): void;
}
```

## Query Builder Pattern

```typescript
class QueryBuilder<T> {
  private query: any;

  constructor(private table: string) {
    this.query = supabase.from(table).select('*');
  }

  filter(filters: Record<string, any>): this {
    for (const [key, value] of Object.entries(filters)) {
      if (Array.isArray(value)) {
        this.query = this.query.in(key, value);
      } else if (typeof value === 'object') {
        this.applyOperator(key, value);
      } else {
        this.query = this.query.eq(key, value);
      }
    }
    return this;
  }

  private applyOperator(field: string, operator: Record<string, any>): void {
    for (const [op, value] of Object.entries(operator)) {
      switch (op) {
        case 'gte':
          this.query = this.query.gte(field, value);
          break;
        case 'lte':
          this.query = this.query.lte(field, value);
          break;
        case 'gt':
          this.query = this.query.gt(field, value);
          break;
        case 'lt':
          this.query = this.query.lt(field, value);
          break;
        case 'contains':
          this.query = this.query.ilike(field, `%${value}%`);
          break;
        case 'startsWith':
          this.query = this.query.ilike(field, `${value}%`);
          break;
      }
    }
  }

  sort(sortFields: string[]): this {
    for (const field of sortFields) {
      const ascending = !field.startsWith('-');
      const fieldName = ascending ? field : field.substring(1);

      this.query = this.query.order(fieldName, { ascending });
    }
    return this;
  }

  paginate(page: number, pageSize: number): this {
    const start = (page - 1) * pageSize;
    const end = start + pageSize - 1;

    this.query = this.query.range(start, end);
    return this;
  }

  async execute(): Promise<{ data: T[]; count: number }> {
    const { data, error, count } = await this.query;

    if (error) throw error;

    return { data, count: count || 0 };
  }
}
```

## Validation Pattern

```typescript
interface ValidationRule {
  field: string;
  validate: (value: any, data: any) => string | null;
}

class Validator {
  constructor(private rules: ValidationRule[]) {}

  async validate(data: any): Promise<ValidationError[]> {
    const errors: ValidationError[] = [];

    for (const rule of this.rules) {
      const value = this.getNestedValue(data, rule.field);
      const error = await rule.validate(value, data);

      if (error) {
        errors.push({
          field: rule.field,
          message: error,
          value
        });
      }
    }

    return errors;
  }

  private getNestedValue(obj: any, path: string): any {
    return path.split('.').reduce((curr, key) => curr?.[key], obj);
  }
}

// Usage
const tripValidator = new Validator([
  {
    field: 'route.origin.lat',
    validate: (value) => {
      if (value < -90 || value > 90) {
        return 'Latitude must be between -90 and 90';
      }
      return null;
    }
  },
  {
    field: 'route.origin.lng',
    validate: (value) => {
      if (value < -180 || value > 180) {
        return 'Longitude must be between -180 and 180';
      }
      return null;
    }
  },
  {
    field: 'scheduledStart',
    validate: (value) => {
      const date = new Date(value);
      if (date < new Date()) {
        return 'Scheduled start must be in the future';
      }
      return null;
    }
  }
]);
```

# Anti-Patterns

## Exposing Internal IDs

**NEVER DO THIS**:
```typescript
// ❌ Exposing database structure
GET /api/v1/trips?driverId=123&vehicleId=456

Response:
{
  "user_id": 789,
  "driver_id": 123,
  "vehicle_id": 456,
  "created_by": 1,
  "updated_by": 1
}
```

**DO THIS**:
```typescript
// ✅ Clean external API
GET /api/v1/trips?driver=123&vehicle=456

Response:
{
  "id": "abc-123",
  "driver": { "id": "driver-123", "name": "John" },
  "vehicle": { "id": "vehicle-456", "make": "Toyota" }
}
```

## Nested Resource Explosion

**NEVER DO THIS**:
```typescript
// ❌ Too deeply nested
GET /api/v1/drivers/123/vehicles/456/maintenance/789/parts/101
```

**DO THIS**:
```typescript
// ✅ Flat with filters
GET /api/v1/maintenance-parts/101?vehicle=456
GET /api/v1/maintenance/789/parts
```

## Custom Query Languages

**NEVER DO THIS**:
```typescript
// ❌ Custom query syntax
GET /api/v1/trips?query=status:completed AND driver.rating>4.5
```

**DO THIS**:
```typescript
// ✅ Standard query parameters
GET /api/v1/trips?status=completed&driverRating[gt]=4.5
```

## Inconsistent Error Formats

**NEVER DO THIS**:
```typescript
// ❌ Different error formats
{ "error": "Not found" }
{ "message": "Invalid input", "code": 400 }
{ "errors": [{ "msg": "Required field" }] }
```

**DO THIS**:
```typescript
// ✅ Consistent error format
{
  "error": {
    "code": "resource_not_found",
    "message": "Trip not found",
    "timestamp": "2024-01-15T10:00:00Z",
    "path": "/api/v1/trips/123",
    "requestId": "req-123"
  }
}
```

## Returning Too Much Data

**NEVER DO THIS**:
```typescript
// ❌ Always returning full objects
GET /api/v1/trips

Response: [
  {
    // 200 fields including rarely-used metadata
    "metadata": { /* huge object */ }
  }
]
```

**DO THIS**:
```typescript
// ✅ Return essential fields, allow selection
GET /api/v1/trips?fields=id,status,driver,route.origin

// Or use summary/detail endpoints
GET /api/v1/trips (returns summary)
GET /api/v1/trips/123 (returns full details)
```

# Observability

## Request Logging

```typescript
app.use((req, res, next) => {
  const start = Date.now();

  res.on('finish', () => {
    const duration = Date.now() - start;

    logger.info('API request', {
      method: req.method,
      path: req.path,
      query: req.query,
      status: res.statusCode,
      duration,
      requestId: req.id,
      userId: req.user?.id
    });

    // Track metrics
    metrics.histogram('api.request.duration', duration, {
      method: req.method,
      path: req.route?.path,
      status: res.statusCode
    });
  });

  next();
});
```

## Rate Limiting

```typescript
const rateLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // Max requests per window
  message: {
    error: {
      code: 'rate_limit_exceeded',
      message: 'Too many requests, please try again later'
    }
  },
  standardHeaders: true,
  legacyHeaders: false
});

app.use('/api/', rateLimiter);
```

# Testing Strategy

## API Contract Tests

```typescript
describe('GET /api/v1/trips', () => {
  it('should return paginated trips', async () => {
    const response = await request(app)
      .get('/api/v1/trips')
      .expect(200);

    expect(response.body).toHaveProperty('data');
    expect(response.body).toHaveProperty('pagination');
    expect(Array.isArray(response.body.data)).toBe(true);

    expect(response.body.pagination).toMatchObject({
      page: expect.any(Number),
      pageSize: expect.any(Number),
      totalCount: expect.any(Number),
      hasNext: expect.any(Boolean),
      hasPrevious: expect.any(Boolean)
    });
  });

  it('should filter by status', async () => {
    const response = await request(app)
      .get('/api/v1/trips?status=completed')
      .expect(200);

    expect(response.body.data.every(t => t.status === 'completed')).toBe(true);
  });

  it('should return 404 for invalid trip', async () => {
    const response = await request(app)
      .get('/api/v1/trips/invalid')
      .expect(404);

    expect(response.body.error.code).toBe('resource_not_found');
  });
});
```

## Validation Tests

```typescript
describe('POST /api/v1/trips', () => {
  it('should reject invalid latitude', async () => {
    const response = await request(app)
      .post('/api/v1/trips')
      .send({
        route: {
          origin: { lat: 100, lng: -122 },
          destination: { lat: 37, lng: -122 }
        }
      })
      .expect(400);

    expect(response.body.error.code).toBe('validation_error');
    expect(response.body.error.fields).toContainEqual({
      field: 'route.origin.lat',
      message: expect.any(String),
      value: 100
    });
  });
});
```

# Code Review Checklist

- [ ] Are resource names plural nouns?
- [ ] Are HTTP methods used correctly?
- [ ] Is pagination implemented on all list endpoints?
- [ ] Are max page size limits enforced?
- [ ] Are filters working correctly?
- [ ] Is sorting implemented properly?
- [ ] Are error responses consistent?
- [ ] Do validation errors include field details?
- [ ] Are 404s returned for missing resources?
- [ ] Are 201s returned for successful creations?
- [ ] Is the Location header set on 201 responses?
- [ ] Are related resources properly included when requested?
- [ ] Is field selection working?
- [ ] Are geographic queries using spatial indexes?
- [ ] Is rate limiting configured?
- [ ] Are all requests logged with duration?
- [ ] Is API versioning in place?
- [ ] Is documentation up to date?
- [ ] Are example requests/responses provided?
- [ ] Is authentication/authorization enforced?
