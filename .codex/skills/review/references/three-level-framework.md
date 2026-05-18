# Three-Level Review Framework

Every PR review walks all three levels for every module.
Levels are not severity classes — they are different lenses on the same code.
A single hunk can produce findings at all three levels.

## Level 1: Intent and scope

The PR exists for a stated reason.
Every change must justify itself against that reason.

| Check | What to look for |
|-------|------------------|
| Alignment | Does each hunk advance the stated or inferred PR intent? Could the PR ship without this hunk and still achieve its goal? |
| Scope creep | Unrelated refactors, opportunistic cleanups, dependency bumps, or formatting changes that obscure the real diff. |
| Missing pieces | Claims in the PR body or linked issues that have no corresponding code. Half-finished features behind a flag. |
| Surprise | Changes that are unexpected given the goal: new public API, schema migration, telemetry, vendored code. |
| Test alignment | Tests target the stated behavior, not only the code that was touched. |

Flag intent-vs-scope mismatches early.
A logically correct change that does not belong in this PR is still a problem.

## Level 2: Logic and behavior

Walk the new and modified control flow as if executing it.
Check the obvious paths and the paths the author hoped you would not check.

| Check | What to look for |
|-------|------------------|
| Control flow | Off-by-one, fence-post errors, wrong branch taken, missing else, fallthrough, early return that skips cleanup. |
| Nil/null/zero | Dereference of values that can be unset, default values that mean "absent" but are treated as valid, empty slice vs nil slice. |
| Error paths | Errors swallowed, returned but not handled, wrapped without context, surfaced to wrong layer. Partial failures left in inconsistent state. |
| Invariants | Pre/postconditions preserved across the change. State machine transitions still valid. Type contracts honored. |
| Concurrency | Race conditions, check-then-act without atomicity, shared mutable state without synchronization, goroutine/thread leaks, deadlocks. |
| Ordering | Operations that must happen in a specific order (init before use, lock before mutate, validate before persist). |
| Edge cases | Empty input, single element, large input, unicode, timezone, negative numbers, exact zero, off-by-one boundaries. |
| Backward compatibility | Public API signature changes, wire format changes, DB schema changes, config keys renamed or removed, default values flipped. |
| Old/new interactions | Newly added code path coexists with the old one. Migration steps. Feature flag transitions. Rollback safety. |
| Idempotency | Operations that may retry must produce the same result. Side effects guarded. |

## Level 3: Code quality and maintainability

The change is correct but does the next reader pay for it?
Apply this level after correctness is verified.

| Check | What to look for |
|-------|------------------|
| Duplication | Repeated constants, copy-pasted blocks, parallel hierarchies that should share. Three is the threshold for extraction. |
| Magic values | Hardcoded numbers, strings, timeouts without a named constant or comment explaining the value. |
| Dead code | Functions, branches, or imports left behind. Comments referencing removed code. TODOs older than the file. |
| Abstractions | Leaky abstractions that expose internals. Premature abstractions with one caller. Wrong abstraction (forced into a pattern that does not fit). |
| Naming | Stuttering (`user.UserService`), generic names (`Manager`, `Helper`, `Util`), misleading names that suggest behavior the code does not have. |
| Patterns | Inconsistent with sibling files in the same module. New pattern introduced without removing the old one. |
| Error handling | Granularity and consistency across the layer. User-facing messages do not leak internals. Logs include enough context to debug. |
| Security | Injection (SQL, shell, HTML), insecure deserialization, secrets in code or logs, broken auth/authz, regression in input validation. |
| Performance | N+1 queries, allocations on hot paths, blocking I/O in async code, redundant work in loops, missing pagination on large reads. |
| Test quality | Tests assert behavior, not implementation. Meaningful assertions over presence checks. No over-mocking. Edge cases covered. Tests fail for the right reason. |
| Test seams | New code is testable without setting up the world. Dependencies inverted at the boundary that needs to be mocked. |

## Severity tagging

Tag every finding with severity and confidence:

| Severity | Meaning |
|----------|---------|
| Critical | Blocks merge. Bug, security flaw, broken contract, data loss risk, breaking change without migration path. |
| Major | Should be addressed in this PR. Significant code smell, missing test for new behavior, performance regression, unclear scope. |
| Minor | Suggestion or nit. Style, naming, optional refactor, documentation gap. |

| Confidence | Meaning |
|------------|---------|
| HIGH | Definite issue with code evidence. The reviewer can point at the line and explain why it fails. |
| MEDIUM | Likely issue based on patterns or context. Probable but not proven. |
| LOW | Subjective observation or preference. Alternative approach. |

A finding can be Critical/MEDIUM (likely a bug, would block merge if confirmed) or Minor/HIGH (a definite nit).
The combinations are independent.
