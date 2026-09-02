# Backend Standards: External Integrations & Resilience

Rules for calling third-party services and downstream APIs safely, without letting their failures become your outage.

## 1. Adapter Pattern for Third-Party Services

*   **Wrap, Don't Scatter:** Every third-party SDK/API integration must sit behind a dedicated injectable service (an adapter) that exposes a domain-shaped interface. Do not call third-party SDKs directly from controllers or scatter calls across multiple services.
*   **Testability:** Because the integration sits behind an interface, it can be mocked cleanly in unit tests without touching the real third-party service — see [09-testing-strategy.md](09-testing-strategy.md#3-mocking-strategy).
*   **Isolate Vendor-Specific Types:** Keep the third-party SDK's request/response types inside the adapter. Translate to/from your own DTOs at the boundary so a vendor change doesn't ripple through the whole codebase.

---

## 2. Timeouts

*   **Always Set an Explicit Timeout:** Every outbound HTTP/RPC call must have a timeout. An unbounded call can hang a request (or a whole worker) indefinitely if the downstream service stalls.
*   **Timeout Budget:** Keep outbound timeouts meaningfully shorter than your own endpoint's expected response time, so a slow dependency fails fast enough to still return a controlled error to your caller.

```typescript
// Good: bounded outbound call with rxjs
async fetchExchangeRate(currency: string): Promise<number> {
  const response = await firstValueFrom(
    this.httpService.get(`/rates/${currency}`).pipe(
      timeout(3000),
      catchError((err) => {
        this.logger.warn(`Exchange rate lookup failed for ${currency}`);
        throw new ServiceUnavailableException('Exchange rate service unavailable');
      }),
    ),
  );
  return response.data.rate;
}
```

---

## 3. Retries With Backoff

*   **Retry Only Transient, Idempotent Failures:** Retry network errors, timeouts, and `5xx` responses on operations that are safe to repeat (typically `GET`, or any operation guarded by an idempotency key). Never blindly retry a non-idempotent write.
*   **Exponential Backoff With Jitter:** Space retries out (e.g. 200ms, 800ms, 2s, with randomized jitter) rather than retrying immediately in a tight loop, to avoid hammering a struggling dependency.
*   **Cap Retry Attempts:** Set a hard maximum retry count; after that, fail and surface a clear error rather than retrying indefinitely.

---

## 4. Circuit Breakers

*   **Fail Fast When a Dependency Is Down:** For dependencies that are called frequently, use a circuit-breaker pattern so that after a threshold of consecutive failures, subsequent calls fail immediately (without waiting out a timeout) for a cooldown period, instead of piling up slow, doomed requests.
*   **Half-Open Recovery:** Once the cooldown elapses, allow a small number of trial requests through to detect recovery before fully closing the circuit again.

---

## 5. Failure Isolation

*   **A Downstream Outage Is Not Your Outage:** Design so that a failing third-party dependency degrades only the feature that depends on it, not the whole service. Return a clear, safe error (or a sensible fallback/default) for that feature rather than letting the failure propagate into an unrelated request path.
*   **Don't Block Critical Paths on Non-Critical Dependencies:** If a feature (e.g. sending an analytics event) is not essential to completing the user's request, it should not be able to fail the request — fire it asynchronously or catch and log its failure without surfacing it to the caller.
