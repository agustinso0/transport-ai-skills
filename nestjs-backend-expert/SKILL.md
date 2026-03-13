---
name: nestjs-backend-transport
description: NestJS backend patterns for building transportation platform services including modules, DTOs, validation, repositories, and Swagger documentation.
---

# NestJS Backend Transport

## Description

This skill teaches how to build robust transportation backend services using NestJS framework. It covers modular architecture, dependency injection, Data Transfer Objects (DTOs), input validation, repository pattern, service layer design, and API documentation with Swagger. This enables building scalable, maintainable transportation services with proper separation of concerns.

## When to Activate

Activate this skill when:

- Building NestJS-based transportation services
- Implementing driver management modules
- Creating vehicle management APIs
- Building trip management services
- Designing repository patterns for data access
- Creating DTO validators for input
- Generating Swagger documentation
- Implementing error handling strategies
- Setting up service layer business logic
- Configuring database connections
- Designing request/response structures
- Creating custom decorators or guards
- Implementing middleware and pipes
- Setting up database migrations
- Configuring environment variables

## Engineering Principles

### Modular Architecture

Organize code into feature modules with clear boundaries:

```
src/
├── common/           # Shared utilities
│   ├── decorators/
│   ├── filters/
│   ├── guards/
│   ├── pipes/
│   ├── interceptors/
│   └── exceptions/
├── drivers/          # Driver feature module
│   ├── dto/
│   ├── entities/
│   ├── repositories/
│   ├── services/
│   ├── controllers/
│   └── drivers.module.ts
├── vehicles/         # Vehicle feature module
│   ├── dto/
│   ├── entities/
│   ├── repositories/
│   ├── services/
│   ├── controllers/
│   └── vehicles.module.ts
├── trips/            # Trip feature module
│   ├── dto/
│   ├── entities/
│   ├── repositories/
│   ├── services/
│   ├── controllers/
│   └── trips.module.ts
├── routes/           # Route feature module
├── config/           # Configuration module
├── database/         # Database setup
└── app.module.ts
```

### Single Responsibility Principle

Each class has one reason to change:

- **Controller**: Handle HTTP requests/responses, validate input
- **Service**: Implement business logic, orchestrate operations
- **Repository**: Data access only, query building
- **DTO**: Request/response structure and validation
- **Entity**: Database model definition

### Dependency Injection

Use NestJS built-in DI container:

```typescript
// Service depends on repository through constructor injection
@Injectable()
class DriverService {
  constructor(private readonly driverRepository: DriverRepository) {}
}

// Module provides dependencies
@Module({
  providers: [DriverService, DriverRepository],
})
export class DriversModule {}
```

### Error Handling

Use custom exception filters:

```typescript
@Catch(NotFoundException, BadRequestException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const status = exception.getStatus();
    const message = exception.getResponse();

    response.status(status).json({
      statusCode: status,
      message,
      timestamp: new Date().toISOString(),
    });
  }
}
```

## Backend Architecture

### Module Structure

Each feature module contains:

**Controller Layer**
- Handle HTTP requests
- Route requests to services
- Return HTTP responses
- Basic input validation (via pipes)

**Service Layer**
- Implement business logic
- Orchestrate operations
- Handle transactions
- Publish domain events

**Repository Layer**
- Data access abstraction
- Query building
- Database operations
- No business logic

**DTO Layer**
- Request structure
- Response structure
- Validation rules
- Serialization hints

**Entity Layer**
- Database schema definition
- Column definitions
- Relationships
- Constraints

### Database Integration

Use TypeORM with Supabase:

```typescript
@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: 'postgres',
      host: process.env.DB_HOST,
      port: parseInt(process.env.DB_PORT),
      username: process.env.DB_USER,
      password: process.env.DB_PASSWORD,
      database: process.env.DB_NAME,
      entities: ['src/**/*.entity.ts'],
      synchronize: false, // Use migrations
      logging: process.env.NODE_ENV === 'development',
    }),
  ],
})
export class DatabaseModule {}
```

### Configuration Management

Centralize configuration:

```typescript
@Injectable()
export class ConfigService {
  get database() {
    return {
      host: process.env.DB_HOST,
      port: parseInt(process.env.DB_PORT),
      username: process.env.DB_USER,
      password: process.env.DB_PASSWORD,
      database: process.env.DB_NAME,
    };
  }

  get transport() {
    return {
      maxAssignmentWaitMs: parseInt(process.env.MAX_ASSIGNMENT_WAIT_MS || '30000'),
      maxTripDurationMinutes: parseInt(process.env.MAX_TRIP_DURATION_MINUTES || '480'),
    };
  }
}
```

## Implementation Patterns

### DTO Validation

Use class-validator decorators:

```typescript
import { IsEmail, IsString, IsEnum, Min, Max, ValidateNested } from 'class-validator';

export class CreateDriverDTO {
  @IsString()
  @IsNotEmpty()
  firstName: string;

  @IsString()
  @IsNotEmpty()
  lastName: string;

  @IsEmail()
  email: string;

  @IsEnum(['active', 'inactive', 'suspended'])
  status: 'active' | 'inactive' | 'suspended';

  @Min(0)
  @Max(5)
  rating: number;
}

export class UpdateDriverDTO extends PartialType(CreateDriverDTO) {}
```

### Repository Pattern

Abstraction for data access:

```typescript
@EntityRepository(Driver)
export class DriverRepository extends Repository<Driver> {
  async findAvailable(): Promise<Driver[]> {
    return this.find({
      where: { status: 'available' },
      order: { rating: 'DESC' },
      take: 50,
    });
  }

  async findByIdWithStats(id: string): Promise<Driver> {
    return this.createQueryBuilder('driver')
      .leftJoinAndSelect('driver.trips', 'trips')
      .where('driver.id = :id', { id })
      .select([
        'driver',
        'COUNT(trips.id) as trip_count',
      ])
      .getRawOne();
  }

  async findNearby(lat: number, lng: number, radiusKm: number): Promise<Driver[]> {
    return this.query(
      `SELECT * FROM drivers
       WHERE status = $1
       AND ST_Distance(
         ST_SetSRID(ST_Point($2, $3), 4326),
         location::geography
       ) < $4`,
      ['available', lng, lat, radiusKm * 1000]
    );
  }
}
```

### Service Layer

Business logic implementation:

```typescript
@Injectable()
export class DriverService {
  constructor(
    private readonly driverRepository: DriverRepository,
    private readonly eventEmitter: EventEmitter2,
  ) {}

  async createDriver(createDriverDTO: CreateDriverDTO): Promise<Driver> {
    // Validate business rules
    const existingDriver = await this.driverRepository.findOne({
      where: { email: createDriverDTO.email },
    });

    if (existingDriver) {
      throw new ConflictException('Driver with this email already exists');
    }

    // Create entity
    const driver = this.driverRepository.create(createDriverDTO);
    const savedDriver = await this.driverRepository.save(driver);

    // Publish event
    this.eventEmitter.emit('driver.created', {
      driverId: savedDriver.id,
      email: savedDriver.email,
    });

    return savedDriver;
  }

  async getAvailableDrivers(limit: number = 50): Promise<Driver[]> {
    return this.driverRepository.findAvailable();
  }

  async updateDriverStatus(
    driverId: string,
    status: DriverStatus,
  ): Promise<Driver> {
    const driver = await this.driverRepository.findOne({
      where: { id: driverId },
    });

    if (!driver) {
      throw new NotFoundException(`Driver ${driverId} not found`);
    }

    // Validate transition
    if (!this.isValidStatusTransition(driver.status, status)) {
      throw new BadRequestException(
        `Cannot transition from ${driver.status} to ${status}`,
      );
    }

    driver.status = status;
    const updated = await this.driverRepository.save(driver);

    // Publish event
    this.eventEmitter.emit('driver.status_changed', {
      driverId: updated.id,
      newStatus: status,
      previousStatus: driver.status,
    });

    return updated;
  }

  private isValidStatusTransition(
    from: DriverStatus,
    to: DriverStatus,
  ): boolean {
    const validTransitions: Record<DriverStatus, DriverStatus[]> = {
      available: ['on_trip', 'off_duty'],
      on_trip: ['available', 'off_duty'],
      off_duty: ['available'],
      unavailable: ['available'],
    };

    return validTransitions[from]?.includes(to) ?? false;
  }
}
```

### Controller Pattern

HTTP request handling:

```typescript
@Controller('drivers')
@ApiTags('drivers')
export class DriversController {
  constructor(private readonly driverService: DriverService) {}

  @Post()
  @HttpCode(HttpStatus.CREATED)
  @ApiOperation({ summary: 'Create a new driver' })
  @ApiResponse({
    status: 201,
    description: 'Driver created',
    type: DriverResponseDTO,
  })
  async createDriver(
    @Body() createDriverDTO: CreateDriverDTO,
  ): Promise<DriverResponseDTO> {
    const driver = await this.driverService.createDriver(createDriverDTO);
    return this.mapToResponse(driver);
  }

  @Get()
  @ApiOperation({ summary: 'List all drivers' })
  @ApiQuery({ name: 'status', required: false })
  @ApiQuery({ name: 'limit', required: false, type: Number })
  async listDrivers(
    @Query('status') status?: string,
    @Query('limit') limit: number = 50,
  ): Promise<DriverResponseDTO[]> {
    let drivers: Driver[];

    if (status === 'available') {
      drivers = await this.driverService.getAvailableDrivers(limit);
    } else {
      drivers = await this.driverService.listAllDrivers(limit);
    }

    return drivers.map(d => this.mapToResponse(d));
  }

  @Get(':id')
  @ApiOperation({ summary: 'Get driver by ID' })
  @ApiParam({ name: 'id', type: String })
  @ApiResponse({ type: DriverResponseDTO })
  async getDriver(@Param('id') id: string): Promise<DriverResponseDTO> {
    const driver = await this.driverService.getDriver(id);
    return this.mapToResponse(driver);
  }

  @Patch(':id')
  @ApiOperation({ summary: 'Update driver' })
  @ApiParam({ name: 'id', type: String })
  @ApiResponse({ type: DriverResponseDTO })
  async updateDriver(
    @Param('id') id: string,
    @Body() updateDriverDTO: UpdateDriverDTO,
  ): Promise<DriverResponseDTO> {
    const driver = await this.driverService.updateDriver(id, updateDriverDTO);
    return this.mapToResponse(driver);
  }

  @Post(':id/status')
  @ApiOperation({ summary: 'Update driver status' })
  @ApiParam({ name: 'id', type: String })
  @ApiResponse({ type: DriverResponseDTO })
  async updateStatus(
    @Param('id') id: string,
    @Body() { status }: { status: DriverStatus },
  ): Promise<DriverResponseDTO> {
    const driver = await this.driverService.updateDriverStatus(id, status);
    return this.mapToResponse(driver);
  }

  @Delete(':id')
  @HttpCode(HttpStatus.NO_CONTENT)
  @ApiOperation({ summary: 'Delete driver' })
  @ApiParam({ name: 'id', type: String })
  async deleteDriver(@Param('id') id: string): Promise<void> {
    await this.driverService.deleteDriver(id);
  }

  private mapToResponse(driver: Driver): DriverResponseDTO {
    return {
      id: driver.id,
      firstName: driver.firstName,
      lastName: driver.lastName,
      email: driver.email,
      status: driver.status,
      rating: driver.rating,
      totalTrips: driver.totalTrips,
      createdAt: driver.createdAt,
      updatedAt: driver.updatedAt,
    };
  }
}
```

### Entity Definition

Database model with TypeORM:

```typescript
import { Column, Entity, PrimaryGeneratedColumn, CreateDateColumn, UpdateDateColumn, Index } from 'typeorm';

@Entity('drivers')
@Index(['status'])
@Index(['email'], { unique: true })
export class Driver {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ type: 'varchar' })
  firstName: string;

  @Column({ type: 'varchar' })
  lastName: string;

  @Column({ type: 'varchar', unique: true })
  email: string;

  @Column({ type: 'varchar' })
  phone: string;

  @Column({
    type: 'enum',
    enum: ['available', 'on_trip', 'off_duty', 'unavailable'],
    default: 'available',
  })
  status: DriverStatus;

  @Column({ type: 'decimal', precision: 2, scale: 1, default: 5 })
  rating: number;

  @Column({ type: 'integer', default: 0 })
  totalTrips: number;

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;
}
```

### Module Definition

Feature module setup:

```typescript
@Module({
  imports: [TypeOrmModule.forFeature([Driver])],
  controllers: [DriversController],
  providers: [DriverService, DriverRepository],
  exports: [DriverService],
})
export class DriversModule {}
```

## Code Examples

### Custom Validation Pipe

```typescript
@Injectable()
export class ValidationPipe implements PipeTransform {
  async transform(value: any, metadata: ArgumentMetadata) {
    const { type, metatype } = metadata;

    if (type !== 'body' || !metatype) {
      return value;
    }

    const object = plainToClass(metatype, value);
    const errors = await validate(object);

    if (errors.length > 0) {
      throw new BadRequestException({
        message: 'Validation failed',
        errors: errors.map(error => ({
          field: error.property,
          errors: Object.values(error.constraints),
        })),
      });
    }

    return value;
  }
}
```

### Auth Guard

```typescript
@Injectable()
export class AuthGuard implements CanActivate {
  constructor(private readonly jwtService: JwtService) {}

  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    const token = this.extractToken(request);

    if (!token) {
      throw new UnauthorizedException('No token provided');
    }

    try {
      const payload = this.jwtService.verify(token);
      request.user = payload;
      return true;
    } catch {
      throw new UnauthorizedException('Invalid token');
    }
  }

  private extractToken(request: any): string | undefined {
    const authHeader = request.headers.authorization;
    return authHeader?.split(' ')[1];
  }
}
```

### Pagination Decorator

```typescript
@Injectable()
export class PaginationPipe implements PipeTransform {
  transform(
    value: any,
    metadata: ArgumentMetadata,
  ): { page: number; limit: number } {
    const page = Math.max(1, parseInt(value?.page || '1'));
    const limit = Math.min(100, Math.max(1, parseInt(value?.limit || '10')));

    return { page, limit };
  }
}
```

## Anti-Patterns

### Business Logic in Controllers

**DON'T:**
```typescript
@Controller('drivers')
export class DriversController {
  @Get(':id')
  async getDriver(@Param('id') id: string) {
    // Business logic in controller!
    const driver = await db.query(
      'SELECT * FROM drivers WHERE id = $1',
      [id],
    );

    if (!driver) {
      return { error: 'Not found' };
    }

    return driver;
  }
}
```

**DO:**
```typescript
@Controller('drivers')
export class DriversController {
  constructor(private readonly driverService: DriverService) {}

  @Get(':id')
  async getDriver(@Param('id') id: string) {
    return this.driverService.getDriver(id);
  }
}
```

### Direct Database Access in Services

**DON'T:**
```typescript
@Injectable()
export class DriverService {
  constructor(private db: DataSource) {}

  async getDriver(id: string) {
    // Direct query, no abstraction
    return this.db.query('SELECT * FROM drivers WHERE id = $1', [id]);
  }
}
```

**DO:**
```typescript
@Injectable()
export class DriverService {
  constructor(private readonly driverRepository: DriverRepository) {}

  async getDriver(id: string) {
    return this.driverRepository.findOne({ where: { id } });
  }
}
```

### Incomplete Input Validation

**DON'T:**
```typescript
export class CreateDriverDTO {
  firstName: string;
  email: string;
  status: string;
}
```

**DO:**
```typescript
export class CreateDriverDTO {
  @IsString()
  @IsNotEmpty()
  @MinLength(2)
  firstName: string;

  @IsEmail()
  email: string;

  @IsEnum(['available', 'on_trip', 'off_duty'])
  status: DriverStatus;
}
```

### Synchronous I/O in Services

**DON'T:**
```typescript
async getDriver(id: string) {
  // Blocking synchronous operations
  const driver = driverCache.get(id); // sync
  if (driver) return driver;

  return driverRepository.findOne(id); // async
}
```

**DO:**
```typescript
async getDriver(id: string) {
  // All async operations
  const cachedDriver = await this.cache.get(id);
  if (cachedDriver) return cachedDriver;

  const driver = await this.driverRepository.findOne({
    where: { id },
  });
  await this.cache.set(id, driver);
  return driver;
}
```

### Missing Error Handling

**DON'T:**
```typescript
async createDriver(dto: CreateDriverDTO) {
  // Unhandled errors
  return this.driverRepository.save(dto);
}
```

**DO:**
```typescript
async createDriver(dto: CreateDriverDTO) {
  try {
    const existing = await this.driverRepository.findOne({
      where: { email: dto.email },
    });

    if (existing) {
      throw new ConflictException('Email already registered');
    }

    return this.driverRepository.save(dto);
  } catch (error) {
    if (error instanceof ConflictException) throw error;

    this.logger.error('Failed to create driver', error);
    throw new InternalServerErrorException('Failed to create driver');
  }
}
```

## Performance Guidelines

### Query Optimization

```typescript
// Load only needed relations
async getDriver(id: string) {
  return this.driverRepository.findOne({
    where: { id },
    relations: ['vehicle', 'trips'],
  });
}

// Limit result sets
async listDrivers(page: number = 1, limit: number = 20) {
  return this.driverRepository.find({
    take: limit,
    skip: (page - 1) * limit,
  });
}

// Use pagination cursor for large datasets
async listDriversWithCursor(cursor?: string, limit: number = 20) {
  let query = this.driverRepository.createQueryBuilder();

  if (cursor) {
    query = query.where('drivers.id > :cursor', { cursor });
  }

  return query.take(limit).getMany();
}
```

### Caching Strategy

```typescript
@Injectable()
export class DriverService {
  constructor(
    private readonly driverRepository: DriverRepository,
    private readonly cacheManager: Cache,
  ) {}

  async getDriver(id: string): Promise<Driver> {
    // Check cache first
    const cached = await this.cacheManager.get(`driver:${id}`);
    if (cached) return cached;

    // Fetch from database
    const driver = await this.driverRepository.findOne({ where: { id } });

    if (!driver) {
      throw new NotFoundException();
    }

    // Cache for 5 minutes
    await this.cacheManager.set(`driver:${id}`, driver, 300000);

    return driver;
  }

  async updateDriver(id: string, dto: UpdateDriverDTO): Promise<Driver> {
    const driver = await this.driverRepository.update(id, dto);

    // Invalidate cache
    await this.cacheManager.del(`driver:${id}`);

    return driver;
  }
}
```

### Batch Operations

```typescript
async bulkCreateDrivers(drivers: CreateDriverDTO[]): Promise<Driver[]> {
  return this.driverRepository
    .createQueryBuilder()
    .insert()
    .into(Driver)
    .values(drivers)
    .orIgnore() // Skip duplicates
    .execute()
    .then(() => this.driverRepository.find({
      where: { email: In(drivers.map(d => d.email)) }
    }));
}
```

## Observability

### Structured Logging

```typescript
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  constructor(private readonly logger: Logger) {}

  intercept(context: ExecutionContext, next: CallHandler) {
    const request = context.switchToHttp().getRequest();
    const { method, url, body } = request;
    const startTime = Date.now();

    return next.handle().pipe(
      tap(response => {
        const duration = Date.now() - startTime;
        this.logger.log({
          type: 'http_request',
          method,
          url,
          status: context.switchToHttp().getResponse().statusCode,
          duration_ms: duration,
          timestamp: new Date().toISOString(),
        });
      }),
      catchError(error => {
        const duration = Date.now() - startTime;
        this.logger.error({
          type: 'http_error',
          method,
          url,
          error: error.message,
          duration_ms: duration,
          timestamp: new Date().toISOString(),
        });
        throw error;
      }),
    );
  }
}
```

### Request ID Tracking

```typescript
@Injectable()
export class RequestIdMiddleware implements NestMiddleware {
  constructor(private readonly logger: Logger) {}

  use(req: any, res: any, next: () => void) {
    req.id = req.id || crypto.randomUUID();
    res.setHeader('X-Request-ID', req.id);

    this.logger.debug({
      type: 'request_start',
      request_id: req.id,
      method: req.method,
      url: req.url,
    });

    next();
  }
}
```

### Application Metrics

```typescript
@Injectable()
export class MetricsService {
  private readonly httpRequestDuration = new Histogram({
    name: 'http_request_duration_seconds',
    help: 'HTTP request latency',
    labelNames: ['method', 'route', 'status_code'],
  });

  private readonly driversCreated = new Counter({
    name: 'drivers_created_total',
    help: 'Total drivers created',
  });

  recordRequestDuration(
    method: string,
    route: string,
    statusCode: number,
    duration: number,
  ) {
    this.httpRequestDuration
      .labels(method, route, statusCode.toString())
      .observe(duration / 1000);
  }

  recordDriverCreated() {
    this.driversCreated.inc();
  }
}
```

## Testing Strategy

### Unit Tests for Services

```typescript
describe('DriverService', () => {
  let service: DriverService;
  let repository: DriverRepository;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        DriverService,
        {
          provide: DriverRepository,
          useValue: {
            findOne: jest.fn(),
            find: jest.fn(),
            create: jest.fn(),
            save: jest.fn(),
          },
        },
      ],
    }).compile();

    service = module.get<DriverService>(DriverService);
    repository = module.get<DriverRepository>(DriverRepository);
  });

  describe('getDriver', () => {
    it('should return driver when found', async () => {
      const driverId = 'driver-123';
      const driver = { id: driverId, firstName: 'John', status: 'available' };

      jest.spyOn(repository, 'findOne').mockResolvedValue(driver as any);

      const result = await service.getDriver(driverId);

      expect(result).toEqual(driver);
      expect(repository.findOne).toHaveBeenCalledWith({
        where: { id: driverId },
      });
    });

    it('should throw NotFoundException when driver not found', async () => {
      jest.spyOn(repository, 'findOne').mockResolvedValue(null);

      await expect(service.getDriver('non-existent')).rejects.toThrow(
        NotFoundException,
      );
    });
  });

  describe('createDriver', () => {
    it('should create driver with valid input', async () => {
      const createDto: CreateDriverDTO = {
        firstName: 'Jane',
        lastName: 'Doe',
        email: 'jane@example.com',
        status: 'available',
        rating: 5,
      };

      const driver = { id: 'new-id', ...createDto };

      jest.spyOn(repository, 'findOne').mockResolvedValue(null);
      jest.spyOn(repository, 'create').mockReturnValue(driver as any);
      jest.spyOn(repository, 'save').mockResolvedValue(driver as any);

      const result = await service.createDriver(createDto);

      expect(result).toEqual(driver);
      expect(repository.save).toHaveBeenCalled();
    });

    it('should throw ConflictException when email exists', async () => {
      const createDto: CreateDriverDTO = {
        firstName: 'Jane',
        lastName: 'Doe',
        email: 'jane@example.com',
        status: 'available',
        rating: 5,
      };

      jest.spyOn(repository, 'findOne').mockResolvedValue({} as any);

      await expect(service.createDriver(createDto)).rejects.toThrow(
        ConflictException,
      );
    });
  });
});
```

### Integration Tests for Controllers

```typescript
describe('DriversController (e2e)', () => {
  let app: INestApplication;

  beforeAll(async () => {
    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [DriversModule],
    }).compile();

    app = moduleFixture.createNestApplication();
    await app.init();
  });

  afterAll(async () => {
    await app.close();
  });

  describe('POST /drivers', () => {
    it('should create driver', () => {
      return request(app.getHttpServer())
        .post('/drivers')
        .send({
          firstName: 'John',
          lastName: 'Doe',
          email: 'john@example.com',
          status: 'available',
          rating: 5,
        })
        .expect(201)
        .expect(res => {
          expect(res.body).toHaveProperty('id');
          expect(res.body.email).toBe('john@example.com');
        });
    });

    it('should reject invalid email', () => {
      return request(app.getHttpServer())
        .post('/drivers')
        .send({
          firstName: 'John',
          lastName: 'Doe',
          email: 'invalid-email',
          status: 'available',
          rating: 5,
        })
        .expect(400);
    });
  });

  describe('GET /drivers/:id', () => {
    it('should return driver by id', () => {
      const driverId = 'driver-123';

      return request(app.getHttpServer())
        .get(`/drivers/${driverId}`)
        .expect(200)
        .expect(res => {
          expect(res.body.id).toBe(driverId);
        });
    });

    it('should return 404 for non-existent driver', () => {
      return request(app.getHttpServer())
        .get('/drivers/non-existent')
        .expect(404);
    });
  });
});
```
