# Backend Standards: Agent Workflow & Code Quality

Rules for how an agent should plan, scope, and communicate backend changes, plus code-quality and version-control hygiene expectations.

## 1. Pre-Implementation Analysis

*   **Analyze Before Coding:** Before implementing any non-trivial change, deeply analyze the problem and the proposed solution against industry best practices, existing patterns in the codebase, and security implications.
*   **Lay Out the Plan First:** Present the intended approach to the user before writing code — what will change, why, and what alternatives were considered. Get the user's input on gaps rather than silently assuming an answer to an open question.
*   **Security-Auditor Bar:** For anything touching auth, data access, PHI/PII, or external input, the proposed solution should be one that would hold up to a third-party security review — see [05-security-auth.md](05-security-auth.md#5-pre-implementation-security-review).

---

## 2. Scope Discipline

*   **Confirm Before Going Out of Scope:** If accomplishing the requested task requires a change outside its original scope, stop and explain to the user why it's necessary before making it.
*   **Analyze Downstream Impact:** Before making an out-of-scope or structural change, identify what else depends on the code being touched, and confirm that dependents won't break as a result.
*   **Prefer the Smallest Sufficient Change:** When a task can be accomplished without touching existing shared code, do that. Only extend or modify shared code when there's no reasonable way to meet the requirement otherwise.

---

## 3. Backward Compatibility When Extending

*   **Don't Break Existing Callers:** Extending an existing class, service, or function must not change its existing behavior or signature in a way that breaks current callers.
*   **Warn Before Altering Existing Functionality:** If implementing a new feature genuinely requires changing existing behavior, warn the user first, explain concretely why reuse or non-breaking extension isn't possible, and get confirmation before proceeding.
*   **Additive Over Destructive:** Prefer adding a new method/overload/optional parameter over changing the meaning of an existing one.

---

## 4. Comments & Naming

*   **No AI-Sounding Comments:** Comments must read like something a developer on this team would actually write — not generic, over-explained, or restating what the code already says. Avoid filler phrasing ("This function is responsible for...", "Here we are..."). Only comment non-obvious *why*, not *what*.
*   **No Personal Names From Source Material:** When the user pastes in ticket text, task descriptions, or chat excerpts that reference people by name, do not carry those names into code, comments, commit messages, or documentation. Refer to the work item or behavior, not the person.
*   **Accurate Helper Names:** As covered in [02-modules-controllers-services.md](02-modules-controllers-services.md#5-helper--utility-naming), a function's name must precisely reflect what it does.

---

## 5. Dead Code & Unused Declarations

*   **Flag, Don't Silently Delete:** When unreachable code, unused variables, or unused imports/exports are found — whether pre-existing or introduced by a change — point them out to the user rather than silently leaving them in or silently deleting code that might be intentionally kept (e.g. a public export used by another package).
*   **Clean Up What You Introduce:** Any dead code or unused declarations introduced as a byproduct of the current change should be removed as part of that change, not left for later.

---

## 6. Testing

*   **Update Tests With Behavior:** Whenever a function's, service's, or endpoint's behavior changes, its corresponding unit/e2e tests MUST be updated in the same change — not left describing the old behavior.
*   **New Behavior Gets New Coverage:** New services, guards, pipes, or non-trivial helpers should ship with tests covering the primary path and the meaningful edge cases (validation failures, not-found, unauthorized).
*   **Don't Weaken Tests to Pass Them:** If a test fails after a change, fix the underlying issue or update the assertion to reflect a deliberate, understood behavior change — never loosen an assertion just to make a red test green.

---

## 7. Version Control Hygiene

*   **Check Staged Files Before Committing:** Before any commit, review the staged file list for `.env` files, key/certificate material, credential dumps, or any file whose name or content looks like it could carry a secret — even if the filename looks innocuous — and flag it to the user rather than committing it. See [05-security-auth.md](05-security-auth.md#4-secrets--sensitive-files).
*   **Review Broad Adds:** After a broad `git add`, review `git status`/`git diff` for the actual file list before committing — don't assume a wildcard add only picked up the intended files.
