# Backend Standards: Async Handling, Concurrency & Performance

Rules for promise handling, race-condition avoidance, and keeping the request path fast and dependency-lean.

## 1. Promise & Async Handling

*   **No Floating Promises:** Every promise MUST be `await`ed, returned, or explicitly handled (`.catch(...)`). Enable and respect the `no-floating-promises` lint rule — an unawaited promise can silently swallow an error or let a request complete before a write finishes.
*   **Parallelize Independent Work:** Use `Promise.all()` (or `Promise.allSettled()` when partial failure is acceptable) for independent async operations instead of awaiting them sequentially.
*   **Handle Rejections Explicitly:** Wrap external calls (HTTP, queue, third-party SDKs) in try/catch and translate failures into the appropriate `HttpException`, rather than letting a rejection bubble up as an unhandled 500 with no context.

```typescript
// Good: independent async work runs in parallel
const [user, preferences] = await Promise.all([
  this.userRepository.findById(userId),
  this.preferencesRepository.findByUserId(userId),
]);

// Bad: unnecessary sequential await
const user = await this.userRepository.findById(userId);
const preferences = await this.preferencesRepository.findByUserId(userId);
```

---

## 2. Race Conditions & Partial Writes

*   **No Read-Modify-Write on Shared State:** Avoid patterns where a value is read, modified in application code, and written back without protection — a concurrent request can interleave and silently drop an update. Use atomic DB operations (`increment`/`decrement`), row locks, or optimistic concurrency (version columns) instead. See [03-database-migrations-data-integrity.md](03-database-migrations-data-integrity.md#4-transactions--data-integrity).
*   **Multi-Step Operations Are Transactional:** Any operation with more than one write that must succeed or fail together MUST be wrapped in a transaction — partial writes left behind by a failed second step are a data-integrity bug, not an edge case.
*   **Idempotency for Retryable Operations:** Endpoints that may be retried by a client or a queue consumer (payments, webhook handlers, job processors) should be idempotent — use an idempotency key or existence check so a retry doesn't duplicate the effect.
*   **Guard Against Duplicate Concurrent Requests:** For actions where two near-simultaneous requests could both "succeed" incorrectly (e.g. double-booking the last slot), rely on a unique constraint or locked read at the database level rather than an application-level check-then-act.

---

## 3. Performance

*   **Lightweight Middleware:** As covered in [04-api-contracts-responses-middleware.md](04-api-contracts-responses-middleware.md#3-middleware-guidelines), middleware runs on the hot path for every request — keep it fast and non-blocking.
*   **Avoid N+1 Queries:** When loading a collection with related data, use joins, `relations`/`include`, or batched `DataLoader`-style loading instead of issuing one query per row inside a loop.
*   **Paginate by Default:** Any endpoint returning a potentially large collection MUST support pagination (limit/offset or cursor-based) rather than returning the full table.
*   **Cache Deliberately:** Cache expensive, frequently-read, low-volatility data (via Nest's `CacheModule` or a dedicated cache store) — but only after confirming staleness is acceptable for that data, and never cache PHI/PII without an explicit, reviewed reason.
*   **Don't Block the Event Loop:** Avoid heavy synchronous computation (large in-memory sorts/transforms, sync crypto on big payloads) on the request path; move it to a background job/queue if it's non-trivial.

---

## 4. Dependency Minimalism

*   **Prefer Built-In/Native Solutions First:** Before adding a new npm package, check whether Node's standard library or NestJS's existing feature set already covers the need. Extra packages add bundle weight, transitive dependencies, and a larger vulnerability surface.
*   **Vet Before Adding:** If a package is genuinely needed, check its maintenance status, weekly download count, bundle/dependency footprint, and known-vulnerability history before introducing it — and flag the addition to the user rather than adding it silently.
*   **Avoid Overlapping Packages:** Don't add a second library that does roughly what an existing dependency already does (e.g. a second date library, a second HTTP client).
