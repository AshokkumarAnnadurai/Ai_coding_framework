# Backend Standards: Observability, Health & Graceful Shutdown

Rules for health/readiness checks, process lifecycle, metrics, and distributed tracing.

## 1. Health Checks

*   **Dedicated Health Endpoint:** Expose a `/health` endpoint using `@nestjs/terminus` that checks the app's own ability to serve traffic — database connectivity, critical external dependencies, disk/memory thresholds where relevant.
*   **Keep It Fast and Side-Effect Free:** Health checks run frequently (by load balancers/orchestrators) — they must be cheap, read-only, and must not trigger business logic or write operations.
*   **Don't Leak Internals:** As with any endpoint, a health check response must not expose connection strings, internal hostnames, or stack traces — a boolean/status-per-dependency is enough.

```typescript
@Controller('health')
export class HealthController {
  constructor(
    private readonly health: HealthCheckService,
    private readonly db: TypeOrmHealthIndicator,
  ) {}

  @Get()
  @HealthCheck()
  check(): Promise<HealthCheckResult> {
    return this.health.check([() => this.db.pingCheck('database')]);
  }
}
```

---

## 2. Readiness vs. Liveness

*   **Liveness:** "Is the process still running and not deadlocked?" — should almost never fail once the process has started; a failing liveness check tells the orchestrator to restart the container.
*   **Readiness:** "Is this instance currently able to serve traffic?" — should fail during startup (before DB connection is established), during graceful shutdown (draining), or when a critical dependency is down; a failing readiness check tells the orchestrator to stop routing traffic here, without restarting the process.
*   **Expose Both Separately** when the deployment target (Kubernetes, ECS, etc.) supports distinct probes, so a transient dependency outage doesn't cause unnecessary process restarts.

---

## 3. Graceful Shutdown

*   **Enable Shutdown Hooks:** Call `app.enableShutdownHooks()` in `main.ts` so Nest's lifecycle hooks (`OnModuleDestroy`, `beforeApplicationShutdown`) actually run on `SIGTERM`/`SIGINT`.
*   **Close Resources Explicitly:** Database connections/pools, queue consumers, and open sockets must be closed in `OnModuleDestroy` — don't rely on the process dying to clean these up.
*   **Drain In-Flight Requests:** On shutdown signal, stop accepting new requests (fail readiness immediately) while letting in-flight requests finish within a bounded timeout before the process exits, so deploys don't cut off active users.

---

## 4. Metrics

*   **Expose Operational Metrics:** Track request counts, latency distributions, and error rates per route, plus queue depth/processing time for background jobs where applicable. A Prometheus-compatible `/metrics` endpoint is the common convention.
*   **Label, Don't Explode Cardinality:** Tag metrics by route/status/method — avoid high-cardinality labels like raw user IDs or full URLs with query strings, which blow up metrics storage.

---

## 5. Tracing & Correlation

*   **Propagate Correlation IDs:** Reuse the request/correlation ID established for logging (see [06-logging-data-privacy.md](06-logging-data-privacy.md#1-logging-standards)) across outbound calls to downstream services so a single request can be traced end-to-end.
*   **Distributed Tracing:** For multi-service architectures, prefer a standard instrumentation layer (OpenTelemetry) over ad-hoc timing logs, so traces are consistent and queryable across services.
