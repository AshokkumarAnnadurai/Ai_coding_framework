# Backend Standards: API Contracts, Responses & Middleware

Rules for standardized response shapes, global error handling, middleware weight, input validation, and route stability.

## 1. Standard Response Envelope

All API responses MUST follow a consistent shape so frontend clients can rely on a single parsing path.

*   **Success responses:**
    ```json
    { "statusCode": 200, "message": "Appointment created", "data": { "id": "..." } }
    ```
*   **Error responses:**
    ```json
    { "statusCode": 400, "error": "Validation failed", "data": null }
    ```
*   Apply this shape globally via a response interceptor and exception filter rather than building it manually inside every controller method.

```typescript
// Good: global response transform interceptor
@Injectable()
export class ResponseInterceptor<T> implements NestInterceptor<T, ApiResponse<T>> {
  intercept(context: ExecutionContext, next: CallHandler): Observable<ApiResponse<T>> {
    const statusCode = context.switchToHttp().getResponse().statusCode;
    return next.handle().pipe(
      map((data) => ({
        statusCode,
        message: (data && data.message) ?? 'Success',
        data: (data && data.data) ?? data ?? null,
      })),
    );
  }
}
```

```typescript
// Good: global exception filter
@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
  private readonly logger = new Logger(GlobalExceptionFilter.name);

  catch(exception: unknown, host: ArgumentsHost): void {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();

    const isHttpException = exception instanceof HttpException;
    const statusCode = isHttpException ? exception.getStatus() : HttpStatus.INTERNAL_SERVER_ERROR;
    const error = isHttpException ? exception.message : 'Internal server error';

    this.logger.error(isHttpException ? error : (exception as Error)?.stack ?? exception);

    response.status(statusCode).json({ statusCode, error, data: null });
  }
}
```

---

## 2. Global Exception Handling

*   **Use Nest's `HttpException` Hierarchy:** Throw `BadRequestException`, `NotFoundException`, `ForbiddenException`, `UnauthorizedException`, `ConflictException`, etc. instead of hand-rolled error objects.
*   **Never Leak Internals:** Error responses MUST NOT include stack traces, SQL/driver messages, internal file paths, connection strings, or environment details. Log the detailed error server-side; return a safe, generic `error` message to the client.
*   **One Global Filter:** Register a single global `ExceptionFilter` in `main.ts` rather than duplicating try/catch error-formatting logic in each controller.

---

## 3. Middleware Guidelines

*   **Keep Middleware Lightweight:** Middleware runs on every matched request before routing/guards resolve — avoid synchronous heavy computation, avoid unnecessary DB calls, and avoid large object allocations here. Push anything non-trivial to a guard, interceptor, or the service layer where it can be scoped more precisely.
*   **Prefer Guards/Interceptors Over Middleware for Business Rules:** Use middleware only for true cross-cutting transport concerns (request ID tagging, lightweight logging, header normalization). Use Guards for authorization decisions and Interceptors for response shaping/timing — they have access to Nest's execution context, which raw middleware does not.
*   **Order Matters:** Be explicit about middleware/guard/interceptor ordering when a request must be authenticated before another concern (e.g. rate limiting per-user) can run correctly.

---

## 4. Input Validation

*   **Global `ValidationPipe`:** Enable a global `ValidationPipe` with `whitelist: true` and `forbidNonWhitelisted: true` so unrecognized fields are stripped/rejected rather than silently passed through.
*   **DTO-Driven:** All validation rules live on DTOs via `class-validator` decorators — see [02-modules-controllers-services.md](02-modules-controllers-services.md#3-dtos-data-transfer-objects). Do not hand-write manual `if` validation chains in controllers for things a decorator already covers.
*   **Sanitization & Trimming:** See [05-security-auth.md](05-security-auth.md#3-input-sanitization--trimming) for input sanitization/trimming rules, including the password-trimming exception that requires explicit user confirmation.

---

## 5. Route Stability & Versioning

*   **Version Routes Explicitly:** Prefix routes with a version (e.g. `/api/v1/...`) using Nest's built-in URI versioning (`app.enableVersioning({ type: VersioningType.URI })`) rather than ad-hoc prefixes.
*   **Routes Are a Contract — Never Change Silently:** An existing route's path, HTTP method, or top-level response shape MUST NOT be changed, renamed, or removed without first explicitly warning the user that other services/clients may depend on it, and getting confirmation before proceeding. Prefer introducing a new versioned route over breaking an existing one.
*   **Deprecation Over Deletion:** When a route must be retired, mark it deprecated (documentation + response header) and coordinate a removal timeline with the user rather than deleting it outright.

---

## 6. API Documentation

*   **Keep Swagger/OpenAPI in Sync:** Decorate controllers and DTOs with `@nestjs/swagger` decorators (`@ApiTags`, `@ApiOperation`, `@ApiProperty`) so generated docs stay accurate as routes evolve. Treat undocumented or stale Swagger annotations as a defect when touching an endpoint.
