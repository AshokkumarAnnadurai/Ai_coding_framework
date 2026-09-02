# Backend Standards: Testing Strategy

Rules for unit, integration, and e2e test scope, mocking discipline, fixtures, and keeping tests in sync with behavior.

## 1. Testing Pyramid & Scope

*   **Unit Tests:** Cover services, pure helpers, guards, pipes, and interceptors in isolation. Mock their dependencies (repositories, other services, HTTP clients).
*   **Integration Tests:** Cover repositories against a real (test) database to validate actual query behavior — mocking the ORM here defeats the purpose of the test.
*   **E2E Tests:** Cover full request/response flow through a controller (`supertest` against a bootstrapped `INestApplication`), including guards, pipes, filters, and the response envelope.
*   **Match Effort to Risk:** Business-critical and security-sensitive paths (auth, payments, permissions, PHI/PII handling) warrant the fullest coverage across all three layers. Trivial DTOs/getters do not need dedicated tests.

---

## 2. Test File Naming & Location

*   **Unit Tests:** Colocate `*.spec.ts` next to the file under test (`user.service.ts` → `user.service.spec.ts`).
*   **E2E Tests:** Place under `test/` at the project root as `*.e2e-spec.ts`, named after the feature/flow being exercised (`auth.e2e-spec.ts`).
*   **Integration Tests:** Colocate with the repository/provider under test, or in a dedicated `test/integration/` directory if they require special setup (test containers, migrations).

---

## 3. Mocking Strategy

*   **Mock at the Boundary, Not the Behavior:** In unit tests, mock the dependency's interface (repository methods, external HTTP client), not internal implementation details of the class under test.
*   **Use Nest's Testing Module:** Build test modules with `Test.createTestingModule({...}).overrideProvider(...).useValue(mockRepo).compile()` rather than manually constructing classes with `new`.
*   **Never Mock the Database in Integration/E2E Tests:** A mocked repository in an integration test only proves the mock works, not that the query is correct. Run integration/e2e tests against a real test database (a dedicated test schema, a disposable container, or an in-memory equivalent the ORM supports).
*   **Don't Over-Mock:** If a unit test needs five mocks wired together to pass, the class under test likely has too many responsibilities — reconsider its design rather than layering more mocks.

```typescript
// Good: unit test mocking the repository boundary
describe('AppointmentService', () => {
  let service: AppointmentService;
  let repository: jest.Mocked<AppointmentRepository>;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        AppointmentService,
        { provide: AppointmentRepository, useValue: { findById: jest.fn() } },
      ],
    }).compile();

    service = module.get(AppointmentService);
    repository = module.get(AppointmentRepository);
  });

  it('throws NotFoundException when appointment does not exist', async () => {
    repository.findById.mockResolvedValue(null);
    await expect(service.findById('missing-id')).rejects.toThrow(NotFoundException);
  });
});
```

---

## 4. Fixtures & Factories

*   **Factory Functions Over Duplicated Literals:** Build test data through factory functions (`createTestUser(overrides?)`) rather than repeating near-identical object literals across test files.
*   **Keep Factories Realistic but Minimal:** Factories should produce valid entities/DTOs by default, with the ability to override specific fields per test — avoid factories so complex they need their own tests.
*   **No Real PHI/PII in Fixtures:** Test fixtures must use clearly synthetic data (`test.user@example.com`, fake names) — never copy real user data into test files, even sanitized. See [06-logging-data-privacy.md](06-logging-data-privacy.md).

---

## 5. Test Independence & Cleanup

*   **No Shared Mutable State Between Tests:** Each test must set up and tear down its own state. Do not rely on test execution order.
*   **Isolate Database State:** Wrap integration/e2e test cases in a transaction that's rolled back afterward, or truncate/reset relevant tables between tests, so one test's data can't leak into another.
*   **Deterministic Time & Randomness:** Freeze or inject clock/UUID generation in tests that depend on them, rather than asserting against real wall-clock time.

---

## 6. Coverage & CI Gate

*   **Meaningful Coverage, Not Vanity Metrics:** Prioritize covering business logic branches (validation failures, authorization checks, edge cases) over chasing a raw coverage percentage.
*   **Tests Run in CI Before Merge:** All unit, integration, and e2e tests must pass in CI prior to merge — a change is not "done" until its tests are green in that environment, not just locally.
*   **Update Tests With Behavior Changes:** As covered in [08-agent-workflow-code-quality.md](08-agent-workflow-code-quality.md#6-testing), any change to behavior must update the tests describing that behavior in the same change.
