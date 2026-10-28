---
name: test-driven-development
description: Drives development with tests. Use when implementing logic with a test runner available, fixing a bug (reproduce with a test first), changing existing behavior, or when the user asks for tests ("buat test", "unit test"). Covers Vitest/Jest + React Testing Library + Playwright (Next.js), Firebase emulator and rules tests, Supabase pgTAP/RLS tests, AWS handlers, Go table-driven tests, and Kotlin/Android (JUnit, MockK, coroutines, Compose UI tests).
---

# Test-Driven Development

Write a failing test before the code that makes it pass. For bugs, reproduce the bug with a test before fixing it. "Seems right" is not done; a passing test is proof.

**Skip or relax** for pure config, docs, static content, throwaway spikes, and styling-only UI changes. For legacy code without tests, add characterization tests (pin current behavior) before refactoring.

## Discover the stack first

The cycle is universal; the commands are not. Before the first test, find how this repo tests and use its commands for RED, GREEN, and the final run:

- Build files: `package.json`, `go.mod`, `build.gradle(.kts)`, `supabase/`, `firebase.json`, `Makefile`
- Checked-in wrappers (`./gradlew`, `make test`, repo scripts) over global tools
- The test framework, and how to run **one** test vs the full suite
- Where tests live, how they are named, what neighboring tests look like
- README, CONTRIBUTING, and CI workflows show the commands that actually gate merges

Never assume `npm test`. Run the focused command during the loop and the full suite before finishing.

## The cycle

```
RED     write a test that fails (for the right reason)
GREEN   write the minimum code to pass
REFACTOR improve names, remove duplication; tests stay green
```

A test that passes on the first run proves nothing: make sure you saw it fail, and that it failed because the behavior is missing, not because of a typo.

### Prove-It pattern for bugs

```
Bug report -> write a test that reproduces it -> it FAILS (bug confirmed)
           -> fix -> it PASSES -> run the full suite (no regressions)
```

Do not start by editing the fix. If you cannot reproduce it in a test, say so and ask for details.

## What to test where (pyramid)

| Level | Use for | Share |
|---|---|---|
| Unit | Pure logic, transforms, validation, reducers/ViewModels | Most tests; ms each |
| Integration | Boundaries: API routes, DB queries, rules, repositories | Some; use real or emulated deps |
| E2E | Critical user flows only (sign-in, checkout) | Few |

Decision guide: pure logic means unit; crosses a boundary (HTTP, DB, rules, file system) means integration; critical user journey means E2E. Prefer the lowest level that can catch the bug.

## Stack playbooks

### Next.js / React / TypeScript

- **Runner**: Vitest (or Jest if the repo uses it). **Components**: React Testing Library plus `user-event`; query by role/label/text (what users see), not by class or test id unless nothing else works.
- **Network**: mock at the network edge with MSW rather than mocking `fetch` internals.
- **Next.js specifics**: async Server Components are hard to unit test with RTL; extract data and logic into plain functions and test those, and cover the rendered page with Playwright. Test Route Handlers and Server Actions as functions (call them with a `Request`/`FormData`), including the **unauthenticated and unauthorized** cases.
- **E2E**: Playwright for critical flows. Use role-based locators, rely on auto-waiting (no fixed `sleep`), isolate state per test (fresh user or seeded data).
- **Hooks**: `renderHook`. **Timers/dates**: fake timers and an injected clock, not real time.

### Firebase

- Use the **Emulator Suite**; never test against production. Run suites with `firebase emulators:exec "<test command>"`.
- **Security Rules**: `@firebase/rules-unit-testing` with `assertSucceeds` / `assertFails`. For every important rule, write the **denied** case (other user's data, unauthenticated, wrong fields, oversized write), not just the happy path.
- **Cloud Functions**: test the business logic as plain functions; integration-test triggers against the emulators.
- Seed and clear emulator data between tests.

### Supabase / Postgres

- Run locally with `supabase start`. Test DB logic, functions, and **RLS policies** with pgTAP via `supabase test db` (files in `supabase/tests/`). For RLS, run assertions as different roles/users (anon, user A, user B, service role) and verify cross-user access fails.
- Client integration tests: sign in as distinct test users and exercise real queries against the local stack; reset the DB between tests (`supabase db reset` or transactions that roll back).
- Migrations: test that they apply cleanly from scratch.

### AWS

- Keep handler logic separate from the Lambda wrapper; unit test the logic as a pure function.
- Mock the AWS SDK v3 at the client boundary (`aws-sdk-client-mock` in JS); for real service behavior use LocalStack or SAM local in integration tests, and say so when behavior may differ from real AWS.
- Infrastructure as code: assert on synthesized templates (for example CDK assertions) for IAM least privilege and public-access settings.

### Go

- **Table-driven tests** with `t.Run(name, ...)` subtests; call `t.Parallel()` where state is isolated; use `t.Helper()`, `t.Cleanup()`, `t.TempDir()`.
- Run `go test ./...`; add `-race` for concurrency code, `-count=1` to bypass the test cache, `-run TestName` for focus, `-cover` when tracked.
- **HTTP**: `net/http/httptest` for handlers and clients. **Dependencies**: define small interfaces where they are consumed and use hand-written fakes. **Real DB**: `testcontainers-go` (or a test schema) for repository tests instead of mocking SQL.
- Use **fuzz tests** (`go test -fuzz`) for parsers and validators; golden files for large outputs.
- Compare with `cmp.Diff` (or `testify` if the repo already uses it); keep error assertions specific (`errors.Is` / `errors.As`).

### Kotlin / Android

- **Local unit tests** (`./gradlew testDebugUnitTest`): JUnit, MockK (or hand-written fakes), `kotlinx-coroutines-test` (`runTest`), and Turbine for `Flow`. **Inject dispatchers** so tests control them; do not hardcode `Dispatchers.IO` in code under test.
- **ViewModels**: assert on emitted UI state for a given input/event, not on internal calls.
- **Compose UI tests**: `createComposeRule`, find nodes by semantics (`onNodeWithText`, `onNodeWithContentDescription`, test tags as a fallback), assert state, and use `waitUntil`/idling rather than sleeps.
- **Room**: in-memory database for DAO tests. **Android-framework code on the JVM**: Robolectric. **Instrumented tests** (`./gradlew connectedDebugAndroidTest`) need a device or emulator; keep them for what truly needs the framework.
- Prefer fakes (fake repository, fake API) over deep mock chains.

## Writing good tests

- **Test behavior and state, not implementation.** Assert outcomes; do not assert which internal methods were called. Tests that mirror the implementation break on refactors.
- **Prefer real, then fake, then stub, then mock.** Mock only what is slow, non-deterministic, or has side effects (external APIs, email, payments). Over-mocking yields tests that pass while production breaks.
- **DAMP over DRY.** A test should read as a self-contained specification; a little duplication is fine.
- **Arrange, Act, Assert**, one behavior per test.
- **Name tests as specifications**: `rejects empty title`, not `works` or `test 3`.
- **Deterministic**: no real time, randomness, network, or shared state between tests; each test sets up and cleans up its own data.
- **Beyonce rule**: if you liked it, put a test on it. Tests, not infrastructure changes, are responsible for catching your regressions.

## Anti-patterns

| Anti-pattern | Fix |
|---|---|
| Testing implementation details | Test inputs and outputs |
| Flaky tests (timing, order, shared state) | Deterministic waits, isolation, injected clocks |
| Testing library/framework behavior | Test your code only |
| Large unreviewed snapshots | Use sparingly; review every change |
| Happy-path only (especially auth, rules, RLS) | Always include denied, empty, and error cases |
| Mocking everything | Real or fake at the boundary |
| Tests that need production services | Emulators, local stacks, containers |
| Deleting or skipping a test to go green | Fix the code or the test, and say why |

## Common rationalizations

| Rationalization | Reality |
|---|---|
| "I'll write tests after" | You will not, and after-the-fact tests mirror the implementation. |
| "Too simple to test" | Simple code grows; the test documents intent. |
| "I tested it manually" | Manual checks do not persist. |
| "It's a prototype" | Prototypes become production. |
| "Security rules/RLS are config, not code" | They are the access control. Test the deny paths. |
| "Run the suite again just to be sure" | Re-running on unchanged code adds nothing; run again after edits. |

## Red flags

- New logic or a bug fix with no test
- Defaulting to `npm test` without checking the repo
- Tests that pass on the first run, or "all pass" with none actually run
- Rules, RLS, or authorization changes with no deny-path test
- Tests hitting real Firebase, Supabase, or AWS resources
- Skipped or disabled tests to make the suite pass
- Flaky tests tolerated or retried instead of fixed

## Verification

- [ ] Every new behavior has a test; bug fixes have a reproduction test that failed before the fix
- [ ] The full suite passes using the repo's own command (`npm test` / `pnpm test`, `go test ./...`, `./gradlew test`, `supabase test db`, `firebase emulators:exec ...`)
- [ ] Deny, empty, and error cases are covered for auth, rules, and RLS changes
- [ ] No test skipped or disabled; no test depends on production services
- [ ] Test names describe behavior
- [ ] Coverage did not drop (if tracked)

Run each test command after a change that could affect the result; do not repeat it on unchanged code.
