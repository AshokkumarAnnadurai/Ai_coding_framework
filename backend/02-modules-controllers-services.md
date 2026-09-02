# Backend Standards: Modules, Controllers, Services & DTOs

This document outlines the layered architecture, dependency injection rules, DTO/entity typing, and helper-naming discipline for NestJS modules.

## 1. Layered Architecture

*   **Controllers:** Handle HTTP concerns only — route binding, param/body extraction, delegating to a service, and returning a result. Controllers MUST NOT contain business logic, validation logic beyond DTO binding, or direct database/query calls.
*   **Services:** Own business logic and orchestration. Services call repositories (or other services) and apply domain rules. Services MUST NOT know about `Request`/`Response` objects — keep them transport-agnostic.
*   **Repositories:** Own data access exclusively (queries, ORM calls, transactions). No business logic belongs here — only data shape and persistence concerns.
*   **Always Use a Repository or Data-Access Service:** Never inject the raw database client/ORM connection directly into a controller. Every entity/table MUST be accessed through a dedicated repository or data-access provider that the service layer depends on.

```typescript
// Good: controller → service → repository
@Controller('appointments')
export class AppointmentController {
  constructor(private readonly appointmentService: AppointmentService) {}

  @Get(':id')
  async getOne(@Param('id', ParseUUIDPipe) id: string): Promise<AppointmentResponseDto> {
    return this.appointmentService.findById(id);
  }
}

@Injectable()
export class AppointmentService {
  constructor(private readonly appointmentRepository: AppointmentRepository) {}

  async findById(id: string): Promise<AppointmentResponseDto> {
    const appointment = await this.appointmentRepository.findById(id);
    if (!appointment) {
      throw new NotFoundException('Appointment not found');
    }
    return AppointmentResponseDto.fromEntity(appointment);
  }
}

@Injectable()
export class AppointmentRepository {
  constructor(
    @InjectRepository(AppointmentEntity)
    private readonly repo: Repository<AppointmentEntity>,
  ) {}

  findById(id: string): Promise<AppointmentEntity | null> {
    return this.repo.findOne({ where: { id } });
  }
}
```

---

## 2. Dependency Injection & Provider Scope

*   **Use Nest's DI Container:** Never manually instantiate a service, repository, or provider with `new`. Always inject it through the constructor so Nest manages its lifecycle.
*   **Default to Singleton Scope:** Providers (including DB-context/repository providers) MUST use Nest's default singleton scope unless there is a specific, justified need for `Scope.REQUEST` or `Scope.TRANSIENT` (e.g. per-request tenant context). Request-scoped providers have a real performance cost — do not opt into them casually.
*   **One Provider per Concern:** Do not let a single service accumulate unrelated responsibilities (e.g. a `UserService` that also sends emails and generates PDFs). Split into focused, composable providers.

---

## 3. DTOs (Data Transfer Objects)

*   **Every Input and Output is Typed:** All request bodies, query params, and response payloads MUST have a dedicated DTO class — never accept or return raw `any`/`object`.
*   **Validation Decorators:** Use `class-validator` decorators (`@IsString()`, `@IsEmail()`, `@IsUUID()`, `@IsOptional()`, etc.) on every DTO field so the global `ValidationPipe` can enforce them.
*   **Separate Input/Output Shapes:** Do not reuse a request DTO as the response shape or return raw entities from controllers. Map entities to response DTOs explicitly so internal columns (password hashes, internal flags, soft-delete markers) are never accidentally serialized.
*   **Transformation:** Use `class-transformer` (`@Expose()`, `@Exclude()`, `plainToInstance`) or explicit mapper functions to convert entities to response DTOs.

```typescript
export class CreateUserDto {
  @IsEmail()
  email: string;

  @IsString()
  @MinLength(8)
  password: string;

  @IsString()
  @IsNotEmpty()
  fullName: string;
}

export class UserResponseDto {
  @Expose() id: string;
  @Expose() email: string;
  @Expose() fullName: string;
  @Exclude() passwordHash: string; // never leaves the service layer
}
```

---

## 4. Base Entities

*   **Shared Base Entity:** Define a `BaseEntity` (or abstract mapped superclass) in `src/common/entities/` that provides `id`, `createdAt`, `updatedAt`, and, where applicable, `deletedAt` for soft deletes. All domain entities extend it.
*   **No Business Logic in Entities:** Entities describe shape and persistence mapping only. Domain behavior belongs in services.
*   **Explicit Column Typing:** Every column MUST declare its type, nullability, and constraints explicitly — do not rely on ORM type inference for anything that touches validation or migrations.

```typescript
export abstract class BaseEntity {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;
}
```

---

## 5. Helper & Utility Naming

*   **Accurate Naming:** A helper's name MUST precisely describe what it does — no vague names like `handleData`, `doStuff`, or `processItem`. Prefer `formatAppointmentTimeRange`, `calculateProrationAmount`, `isEligibleForDiscount`.
*   **Pure Where Possible:** Helpers and utility functions should be pure — no reading from request context, no mutating arguments, no hidden I/O. If a function needs external state (DB, config, request), it belongs in a service, not a `utils/` helper.
*   **No Duplicate Sources of Truth:** Before adding a new helper or constant, check whether an equivalent already exists in `src/common/` or `src/shared/`. Do not create a second implementation of the same calculation or formatting logic.
