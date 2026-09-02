# Backend Standards: Database Access, Migrations & Data Integrity

Rules for safe database interaction, migration handling, transactions, and avoiding conflicting sources of truth.

## 1. Never Execute Live Queries Directly

*   **No Direct Query Execution:** If database credentials are discoverable (in `.env`, config files, secret managers, or the shell environment), NEVER run a query directly against that database — not for debugging, not for "just checking," not even a `SELECT`.
*   **Always Warn First:** Before proposing any operation that would touch a live database, explicitly tell the user:
    1.  What the query/operation would do.
    2.  Which tables/rows are affected and roughly how many.
    3.  Whether it is reversible.
    4.  What environment the credentials appear to point to (local/staging/production), if determinable.
*   **Read-Only Exploration:** Prefer reading schema/model definitions in code over querying a live database to understand structure. If live inspection is genuinely necessary, ask the user to run it (or explicitly approve it) rather than executing it autonomously.

---

## 2. Migrations

*   **Generate, Never Run:** Migrations MUST be generated as files (e.g. via the ORM's CLI: `typeorm migration:generate`, `prisma migrate dev --create-only`, `sequelize-cli migration:generate`) and left for the user to review and apply. Never execute a migration against any database, even when credentials are present and even in what appears to be a local/dev environment.
*   **Explicit Migration Scope Warning:** Every generated migration MUST be accompanied by a plain-language summary covering:
    *   Whether it is additive (new table/column) or destructive (drop/rename/type-narrowing column, drop table).
    *   Potential data loss (e.g. narrowing a column, dropping a default, dropping a column with existing data).
    *   Whether it requires a backfill or data migration step in addition to the schema change.
    *   Estimated blast radius (e.g. "this rewrites every row in `appointments`") and lock/downtime implications on large tables.
*   **Reversibility:** Always generate a corresponding `down`/rollback migration where the tool supports it, and call out if a migration is not cleanly reversible (e.g. a dropped column with data loss).

---

## 3. Single Source of Truth

*   **No Duplicated Derived Data:** Avoid persisting values that are simply derived from other stored fields (e.g. storing `fullName` when `firstName`/`lastName` already exist) unless there is a proven, documented performance reason — and if so, document how it stays in sync.
*   **Computed at Read Time:** Prefer computing derived values in the service layer or via a database view rather than maintaining a second, potentially stale, copy.
*   **One Repository per Entity:** Each entity/table should have exactly one repository/data-access provider responsible for reading and writing it. Do not scatter direct queries against the same table across multiple services.

---

## 4. Transactions & Data Integrity

*   **Wrap Multi-Step Writes in Transactions:** Any operation that writes to more than one table, or performs a read-modify-write that must stay consistent, MUST be wrapped in a database transaction so partial failures roll back cleanly.
*   **Keep Transactions Short:** Do not perform slow I/O (HTTP calls, email sending, file uploads) inside an open transaction — fetch/prepare data first, then transact only the database writes.
*   **Concurrency-Safe Updates:** Use row-level locking (`SELECT ... FOR UPDATE`), optimistic concurrency (version columns), or atomic update expressions (`increment`/`decrement`) instead of read-then-write patterns for values that can be updated concurrently (balances, counters, seat/slot availability).

```typescript
// Good: transactional multi-table write
async transferCredits(fromId: string, toId: string, amount: number): Promise<void> {
  await this.dataSource.transaction(async (manager) => {
    await manager.decrement(AccountEntity, { id: fromId }, 'balance', amount);
    await manager.increment(AccountEntity, { id: toId }, 'balance', amount);
    await manager.insert(LedgerEntryEntity, { fromId, toId, amount });
  });
}
```

---

## 5. Repository Layer & DI

*   **Injected, Singleton by Default:** DB context/repository providers are registered and injected via Nest's DI container using the default singleton scope — see [02-modules-controllers-services.md](02-modules-controllers-services.md#2-dependency-injection--provider-scope). Do not open ad-hoc connections outside of the framework's connection/pool management.
*   **No Query Logic Outside Repositories:** Services orchestrate; repositories query. If a service is building raw SQL or complex query-builder chains inline, that logic belongs in the repository instead.
