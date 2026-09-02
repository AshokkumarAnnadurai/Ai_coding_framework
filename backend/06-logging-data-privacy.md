# Backend Standards: Logging & Data Privacy

Rules for structured logging, severity labeling, and keeping PHI/PII out of logs, responses, and agent context.

## 1. Logging Standards

*   **Structured, Labeled Logs:** Every log entry MUST carry an accurate severity level so issues can be triaged quickly:
    *   `error` — a failure that needs attention (unhandled exceptions, failed external calls after retries, data integrity violations).
    *   `warn` — a recoverable or unexpected condition worth reviewing (fallback path taken, deprecated route hit, retry succeeded).
    *   `log` / `info` — normal operational events (request completed, job started/finished).
    *   `debug` — verbose detail useful only during active debugging; disabled by default in production.
    *   `verbose` — the noisiest tier, reserved for deep tracing.
*   **Use Nest's `Logger` (or a Configured Provider):** Use Nest's built-in `Logger` class or a wired-up structured logger (`pino`, `winston`) — do not use raw `console.log`.
*   **Correlation IDs:** Attach a request/correlation ID (generated in middleware, propagated through async context) to every log line for a request so a single failure can be traced end-to-end across services.
*   **Contextual, Not Verbose by Default:** Include enough context to debug (module, operation, relevant non-sensitive identifiers) without dumping entire request/response payloads at `log` level.

---

## 2. PHI/PII Protection

*   **Never Log PHI/PII:** Names, addresses, dates of birth, government IDs, medical record details, diagnosis/treatment data, and similar identifiers MUST NOT appear in log output at any level.
*   **Never Return More Than Necessary in API Responses:** Response DTOs must only include fields the consuming client actually needs — do not serialize full entities "for convenience" if that leaks PHI/PII the caller doesn't require. See [02-modules-controllers-services.md](02-modules-controllers-services.md#3-dtos-data-transfer-objects).
*   **Keep PHI/PII Out of Agent Context:** If a log excerpt, database row, or API response containing PHI/PII would otherwise be pulled into the working context (pasted into a message, read from a file, returned from a query result) — exclude or redact it, and explicitly warn the user that sensitive data was found and withheld rather than silently proceeding.
*   **Redact, Don't Guess:** When sensitive fields must be referenced for debugging, mask them (`j***@example.com`, `***-**-1234`) rather than omitting context entirely or fabricating a placeholder that looks real.

```typescript
// Good: redact sensitive fields before logging
this.logger.log(
  `User lookup succeeded for userId=${user.id}`, // no email, name, or DOB
  UserService.name,
);

// Bad: leaks PII into logs
this.logger.log(`User found: ${JSON.stringify(user)}`);
```

---

## 3. Error Responses to Clients

*   **Generic, Safe Messages:** As covered in [04-api-contracts-responses-middleware.md](04-api-contracts-responses-middleware.md#2-global-exception-handling), client-facing error responses MUST NOT include stack traces, internal file paths, database/driver error text, or configuration details — these can leak both implementation details and, in DB error messages, occasionally fragments of stored data.
*   **Server-Side Detail, Client-Side Summary:** Log the full error detail server-side (with correlation ID) and return only a safe, actionable message to the client.

---

## 4. Audit Considerations

*   **Sensitive Operations Get an Audit Trail:** Actions that create, modify, or access PHI/PII, financial data, or permissions changes should write an audit record (actor, action, target, timestamp — not the sensitive payload itself) separate from application logs, so access can be reviewed without needing to mine debug logs.
