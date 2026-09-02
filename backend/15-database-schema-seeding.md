# Backend Standards: Database Schema, Indexing & Seeding

Rules for schema naming conventions, indexing strategy, database-level constraints, soft-delete details, and seed data.

## 1. Naming Conventions

*   **Tables:** Lowercase `snake_case`, consistently either singular or plural across the whole schema (pick one convention and stay consistent — do not mix `user` and `appointments`).
*   **Columns:** Lowercase `snake_case` (`created_at`, `first_name`), even when the ORM entity exposes them as `camelCase` in TypeScript — configure the ORM's naming strategy once, globally, rather than annotating every column.
*   **Foreign Keys:** Name as `<referenced_entity>_id` (e.g. `user_id`, `appointment_id`). Name the constraint itself descriptively (`fk_appointments_user_id`) rather than relying on an auto-generated name.
*   **Indexes:** Name descriptively (`idx_appointments_user_id_status`) so their purpose is clear from a schema dump alone.

---

## 2. Indexing Strategy

*   **Index Foreign Keys:** Every foreign key column should have an index — most databases do not create one automatically.
*   **Index What You Actually Filter/Sort/Join On:** Add indexes based on real query patterns (`WHERE`, `ORDER BY`, `JOIN` columns) — do not index every column speculatively, since each index adds write overhead and storage cost.
*   **Composite Indexes, Ordered by Selectivity:** When a query filters on multiple columns together, use a composite index ordered with the most selective/most-frequently-equality-filtered column first.
*   **Verify With `EXPLAIN`:** For any query on a table expected to grow large, confirm it's actually using the intended index rather than assuming — but only ever run this against a non-production/read-replica database, and only after checking with the user per [03-database-migrations-data-integrity.md](03-database-migrations-data-integrity.md#1-never-execute-live-queries-directly).

---

## 3. Constraints at the Database Level

*   **The Database Is the Last Line of Defense:** `NOT NULL`, `UNIQUE`, foreign key, and check constraints must be enforced in the schema itself — application-level (DTO) validation is a UX layer, not a substitute for DB-level integrity guarantees.
*   **Explicit Cascade Behavior:** Foreign key `ON DELETE`/`ON UPDATE` behavior (`CASCADE`, `RESTRICT`, `SET NULL`) must be a deliberate choice per relationship, documented in the migration, not left to the ORM's default.

---

## 4. Soft Deletes

*   **Standard `deletedAt` Pattern:** Entities that support soft delete extend the shared `BaseEntity` pattern from [02-modules-controllers-services.md](02-modules-controllers-services.md#4-base-entities) with a nullable `deletedAt` timestamp.
*   **Queries Exclude Soft-Deleted Rows by Default:** Repository read methods must filter out soft-deleted rows unless a caller explicitly requests to include them (e.g. an admin "restore" view).
*   **Unique Constraints Must Account for Soft Deletes:** A plain `UNIQUE` constraint on a soft-deletable table will block re-creating a record with the same natural key after a "delete." Use a partial/filtered unique index (`WHERE deleted_at IS NULL`) where the database supports it, or handle the conflict explicitly in the service layer.

---

## 5. Seed Data

*   **Seeds Are Non-Production Only:** Seed scripts populate local/dev/test databases with representative data. They MUST be gated behind an explicit environment check (or simply never invoked against anything other than a local/test connection) — never run against a database that could be staging or production.
*   **Synthetic Data Only:** Seed data must be clearly fake (`test.user@example.com`, placeholder names) — never real user records or anything resembling real PHI/PII, even from an old export. See [06-logging-data-privacy.md](06-logging-data-privacy.md).
*   **Idempotent Seeds:** Seed scripts should be safe to re-run without creating duplicates (upsert semantics or a "has this already been seeded" check).
