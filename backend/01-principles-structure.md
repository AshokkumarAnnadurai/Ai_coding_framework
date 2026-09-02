# Backend Standards: Core Principles & Structure

This document outlines the foundation of NestJS backend engineering, project layouts, module-based architecture, and expected agent behavior.

## 1. Backend Engineering Principles

*   **Production-Quality Code:** Code must be robust, reliable, and ready for enterprise scale. Avoid quick hacks, `console.log` debugging, or stubbed-out logic in production paths.
*   **Maintainability & Readability:** Prioritize self-documenting code with meaningful names, clear module boundaries, and strict TypeScript patterns.
*   **Simplicity:** Prefer simple solutions over complex abstractions. Do not introduce new npm packages or helper layers for behavior the language or NestJS already provides.
*   **Consistency:** Adhere strictly to workspace conventions for module layout, naming, and route design.
*   **Separation of Concerns:** Controllers handle HTTP concerns only. Business logic belongs in services. Data access belongs in repositories/providers. Never blend these layers.
*   **DRY (Don't Repeat Yourself):** Extract shared logic into services, utilities, or base classes rather than duplicating it across modules.
*   **SOLID Principles:** Apply Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, and Dependency Inversion when designing classes, providers, and modules.
*   **Pure Functions:** Prefer pure functions for helpers, mappers, and calculations — same input always produces the same output, no hidden side effects, no mutation of shared state.
*   **Composition Over Duplication:** Favor composing smaller services/providers over building monolithic "god services" with too many responsibilities.
*   **Backward Compatibility:** Ensure changes do not break existing controllers, services, DTOs, database schemas, or consumers of existing API routes.

---

## 2. Backend Project Structure

We follow a modular layout under `/src`:

```
src/
├── main.ts                # Bootstrap, global pipes/filters/interceptors
├── app.module.ts           # Root module
├── config/                 # Environment config, validation schemas
├── common/
│   ├── decorators/         # Custom parameter/method decorators
│   ├── dto/                # Shared base DTOs (pagination, envelopes)
│   ├── entities/           # Shared base entities (BaseEntity, timestamps)
│   ├── filters/             # Global exception filters
│   ├── guards/              # Auth/role guards
│   ├── interceptors/        # Response transform, logging interceptors
│   ├── middleware/          # Lightweight cross-cutting middleware
│   └── pipes/                # Custom validation/transformation pipes
├── database/
│   ├── migrations/          # Generated migration files (never auto-run)
│   └── seeds/                # Seed scripts, non-production only
├── modules/                 # Feature/domain modules (self-contained)
└── shared/                   # Cross-module services/utilities (no domain logic)
```

### Module Ownership
*   Code belonging to a specific business domain (e.g., `users`, `billing`, `appointments`) MUST reside within its own module folder.
*   Do NOT put domain-specific DTOs, entities, or providers inside `common/` or `shared/`. Keep global directories framework-level only.

---

## 3. Module-Based Architecture

Inside `src/modules/`, organize code by domain, following NestJS conventions:

```
modules/
└── <domain-name>/
    ├── <domain-name>.module.ts
    ├── <domain-name>.controller.ts
    ├── <domain-name>.service.ts
    ├── <domain-name>.repository.ts     # or a dedicated data-access provider
    ├── dto/
    │   ├── create-<domain-name>.dto.ts
    │   ├── update-<domain-name>.dto.ts
    │   └── <domain-name>-response.dto.ts
    ├── entities/
    │   └── <domain-name>.entity.ts
    └── <domain-name>.spec.ts / *.controller.spec.ts / *.service.spec.ts
```

### Isolation Rules
*   **Module Boundaries:** A module may import shared providers from `src/common/` or `src/shared/`.
*   **Cross-Module Imports:** Modules SHOULD NOT reach directly into another module's internals (e.g. importing another domain's repository). Expose a public service and import that module properly through Nest's module system instead.
*   **Circular Dependencies:** Keep module import chains linear. Use `forwardRef()` only as a last resort, and prefer restructuring the dependency instead.

---

## 4. NestJS Naming & Routing Conventions

*   **File Naming:** Use kebab-case with the NestJS suffix convention: `user.controller.ts`, `user.service.ts`, `user.repository.ts`, `user.module.ts`, `create-user.dto.ts`, `user.entity.ts`, `roles.guard.ts`, `logging.interceptor.ts`.
*   **Class Naming:** PascalCase matching the file's role: `UserController`, `UserService`, `CreateUserDto`, `RolesGuard`.
*   **Folder Naming:** Lowercase kebab-case, singular or plural consistent with the rest of the codebase (pick one and stay consistent — do not mix `user/` and `users/`).
*   **Route Naming:** Lowercase, kebab-case, plural resource nouns, versioned under a common prefix (e.g. `/api/v1/appointment-slots`). Avoid verbs in route paths — let HTTP methods express the action.
*   **Route Stability:** See [04-api-contracts-responses-middleware.md](04-api-contracts-responses-middleware.md) — routes are a contract with other services and MUST NOT change without explicit prior warning to the user.

---

## 5. Backend Agent Behavior

When working on backend code:
1.  **Inspect First:** Analyze existing modules, shared providers, DTOs, and database schema before writing new code.
2.  **Reuse Existing Providers:** Check `src/common/` and `src/shared/` for existing guards, interceptors, pipes, and utility services before writing new ones.
3.  **Preserve Functionality:** Do not remove existing logic, change method signatures, or refactor unrelated modules unless instructed. See [08-agent-workflow-code-quality.md](08-agent-workflow-code-quality.md) for the full change-scope protocol.
4.  **Plan Before Implementing:** For any non-trivial change, analyze the problem, outline a plan aligned with NestJS and security best practices, and share it with the user before writing code — see [08-agent-workflow-code-quality.md](08-agent-workflow-code-quality.md).
5.  **Never Execute Destructive or Live Operations Autonomously:** Never run raw queries or migrations against a real database, even when credentials are available in the environment. See [03-database-migrations-data-integrity.md](03-database-migrations-data-integrity.md).

---

## 6. Definition of Done Checklist

- [ ] Controller / service / repository layering is respected (no business logic in controllers, no query logic outside repositories).
- [ ] No `any` TS types are present; DTOs and entities are fully typed.
- [ ] New/changed endpoints follow the standard response envelope.
- [ ] No PHI/PII is present in logs, error responses, or committed code.
- [ ] Migrations are generated but not executed; migration scope was explained to the user.
- [ ] Relevant unit/e2e tests are added or updated to match behavior changes.
- [ ] No sensitive files (`.env`, keys, credentials) are staged for commit.
- [ ] App builds and lints successfully without warnings or compile exceptions.
