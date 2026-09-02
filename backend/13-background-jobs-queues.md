# Backend Standards: Background Jobs, Queues & Scheduled Tasks

Rules for offloading slow work, running recurring tasks, and communicating between modules asynchronously.

## 1. When to Use a Queue

*   **Offload Anything Slow or Non-Critical to the Response:** Work that doesn't need to complete before the HTTP response is returned — sending emails/notifications, generating reports/PDFs, processing uploaded media, batch/bulk operations — belongs in a background job, not inline in the request handler.
*   **Offload Anything That Needs Retry Semantics:** If an operation might transiently fail and should be retried with backoff, a queue's built-in retry handling is a better fit than ad-hoc retry loops inside a request.

---

## 2. Queue Library & Structure

*   **Use `@nestjs/bullmq` (or `@nestjs/bull`) With Redis:** Prefer the framework-integrated queue module over a hand-rolled polling mechanism.
*   **Dedicated Processor Classes:** Define job processing logic in its own `@Processor()` class, not inline inside the service that enqueues the job. The producer (service) should only be responsible for adding jobs to the queue.
*   **Typed Job Payloads:** Give every job a typed payload interface/DTO — don't pass loosely-shaped objects into `queue.add()`.

```typescript
// Producer
@Injectable()
export class NotificationService {
  constructor(@InjectQueue('notifications') private readonly queue: Queue) {}

  async queueWelcomeEmail(userId: string): Promise<void> {
    await this.queue.add('welcome-email', { userId } satisfies WelcomeEmailJob, {
      attempts: 3,
      backoff: { type: 'exponential', delay: 2000 },
    });
  }
}

// Processor
@Processor('notifications')
export class NotificationProcessor extends WorkerHost {
  async process(job: Job<WelcomeEmailJob>): Promise<void> {
    await this.emailService.sendWelcomeEmail(job.data.userId);
  }
}
```

---

## 3. Job Idempotency & Retries

*   **Jobs Must Be Safe to Re-Run:** A job may be retried or, in rare failure modes, processed more than once — design job handlers so re-processing the same job doesn't duplicate side effects (duplicate emails, duplicate charges). Use an idempotency check (e.g. "has this notification already been sent for this ID?") where the side effect isn't naturally idempotent.
*   **Bounded Retries With Backoff:** Configure `attempts` and a `backoff` strategy per job type rather than letting a broken job retry forever.
*   **Dead-Letter / Failed-Job Visibility:** Jobs that exhaust their retries must be visible (failed-job queue, alert, or logged at `error` severity) for manual review — a silently-dropped failed job is a data-loss bug.

---

## 4. Scheduled Tasks

*   **Use `@nestjs/schedule` for Cron-Like Work:** Prefer `@Cron()`, `@Interval()`, or `@Timeout()` decorators over external cron scripts calling into the API.
*   **Keep Scheduled Handlers Thin:** A `@Cron()` method should delegate to a service method immediately — the scheduling decorator is wiring, not a place for business logic.
*   **Guard Against Overlapping Runs:** For jobs that might run longer than their interval, ensure a lock/flag prevents a new run from starting while a previous run is still in progress, unless concurrent runs are explicitly safe.

---

## 5. Event-Driven Communication

*   **`@nestjs/event-emitter` for In-Process Decoupling:** When one module needs to react to something happening in another (e.g. `user.created` triggering a welcome email) without being tightly coupled, emit a domain event and let listeners subscribe — rather than injecting the notification service directly into unrelated domain logic.
*   **Document Events as a Contract:** Event names and payload shapes should be defined as typed constants/interfaces in one place, since they function as an internal contract between decoupled modules.
*   **Don't Use Events to Hide Required Synchronous Logic:** If a step must complete before the operation can be considered successful (e.g. a required data write), it belongs in the main flow — events are for genuinely optional/decoupled side effects, not a way to avoid explicit sequencing.
