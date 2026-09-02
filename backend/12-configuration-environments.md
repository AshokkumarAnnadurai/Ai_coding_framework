# Backend Standards: Configuration & Environment Management

Rules for loading, validating, and separating configuration across environments, plus feature-flag conventions.

## 1. Config Loading

*   **Single Source: `ConfigModule`:** Load all environment variables through Nest's `ConfigModule` (`ConfigModule.forRoot({ isGlobal: true, ... })`) and inject `ConfigService` — never read `process.env` directly scattered throughout the codebase.
*   **Typed Config Access:** Wrap raw config access in typed getter methods or a strongly-typed config object, so consumers get compile-time safety instead of stringly-typed `configService.get('SOME_KEY')` calls sprinkled everywhere.

---

## 2. Fail Fast on Missing/Invalid Config

*   **Validate at Boot:** Define a validation schema (via `Joi`, or a `class-validator`-decorated config class) and pass it to `ConfigModule.forRoot({ validationSchema })`. The application MUST refuse to start if a required environment variable is missing or malformed.
*   **Never Fail Late:** A missing secret or malformed config value should surface as a startup crash with a clear message — not as an intermittent runtime error discovered when a specific code path first executes in production.

```typescript
ConfigModule.forRoot({
  isGlobal: true,
  validationSchema: Joi.object({
    NODE_ENV: Joi.string().valid('development', 'staging', 'production').required(),
    DATABASE_URL: Joi.string().uri().required(),
    JWT_SECRET: Joi.string().min(32).required(),
    AUTH_PASSWORD_PEPPER: Joi.string().min(16).required(),
  }),
});
```

---

## 3. Per-Environment Separation

*   **Environment-Specific Files/Values:** Keep `development`, `staging`, and `production` configuration distinct (separate `.env` files locally, separate secret stores per environment in deployment). Never point a lower environment at production data, and never let a production deploy silently fall back to a lower environment's defaults.
*   **No Shared Secrets Across Environments:** Production credentials (DB, JWT signing keys, third-party API keys) must be unique to production and not reused in staging/dev.
*   **`.env` Files Stay Local and Uncommitted:** Only a `.env.example` (with placeholder, non-real values) is committed. See [05-security-auth.md](05-security-auth.md#4-secrets--sensitive-files).

---

## 4. Feature Flags

*   **Centralized Flag Definitions:** Define feature flags in one typed place (a config section or dedicated flags service) rather than scattering raw `process.env.FEATURE_X === 'true'` checks throughout business logic.
*   **Flags Are Temporary:** Treat a feature flag as scaffolding for a rollout, not permanent branching logic — once a feature is fully rolled out (or rolled back), remove the flag and the dead branch it guarded.

---

## 5. No Hardcoded Environment-Specific Values

*   **Everything Environment-Dependent Comes From Config:** URLs, timeouts, rate limits, third-party endpoints, and similar values must be sourced from configuration — never hardcoded literals that silently differ between what's in source control and what's actually needed per environment.
