---
name: incremental-implementation
description: "Delivers multi-file features in thin, verifiable slices instead of one big change. Use when implementing a feature that spans layers or several files (DB + API + UI, Supabase migration + RLS + client, Go handler + repository, Android data/domain/UI), when a task feels too big to land at once, or when you are about to write 100+ lines before testing. Not needed for single-function edits. Indonesian triggers: kerjakan bertahap, fitur besar, pecah tugas, implementasi per langkah, terlalu besar untuk sekali jalan."
---

# Incremental Implementation

Build in thin vertical slices: implement one piece, test it, verify it, then expand. Each slice leaves the system working and testable. Never implement a whole feature in one pass.

**Skip** for single-file, single-function changes with minimal scope.

## The slice cycle

For each slice:

1. **Implement** the smallest complete piece of functionality.
2. **Test**: run the existing tests; write a test for the new behavior (see `test-driven-development`).
3. **Verify**: tests pass, build succeeds, and the slice works (manual check or stated limitation).
4. **Checkpoint**: if the user has asked you to commit, make an atomic commit with a descriptive message (see `git-workflow-and-versioning`). Otherwise stop at a clean, working state and say what changed. Never commit unasked.
5. **Next slice**: carry forward, do not restart.

## Slicing strategies

**Vertical slices (preferred)**: one complete path through the stack per slice.
```
Slice 1: create a task (DB + API + minimal UI)   -> user can create a task
Slice 2: list tasks                               -> user can see tasks
Slice 3: edit a task                              -> user can modify
Slice 4: delete a task (with confirmation)        -> full CRUD
```

**Contract-first**: when backend and client are built in parallel, slice 0 defines the contract (types, OpenAPI, Firestore document shape, Supabase table + RLS policy, Kotlin interface), then each side builds against it, then integrate.

**Risk-first**: do the riskiest or most uncertain piece first (a new SDK, a real-time connection, a permissions model) so failure shows up before you invest in the rest.

### Stack examples

- **Next.js + Supabase**: (1) migration + RLS policy + deny-path test, (2) server function or Route Handler with validation and auth check, (3) minimal UI with loading/empty/error states, (4) polish.
- **Firebase**: (1) Security Rules + emulator tests, (2) data access layer, (3) UI, (4) Cloud Function trigger if needed.
- **Go service**: (1) handler + request validation + table-driven test with a fake repository, (2) real repository with a test DB, (3) wiring and config, (4) observability.
- **Android (Kotlin)**: (1) domain model + use case with unit tests, (2) repository/data source, (3) ViewModel state, (4) Compose screen with UI test.
- **AWS**: (1) handler logic as a pure function with tests, (2) infrastructure definition, (3) permissions (least privilege), (4) deploy to a dev stage.

## Implementation rules

### Simplicity first
Ask "what is the simplest thing that could work?" before writing, and check after: could it be fewer lines, do abstractions earn their complexity, am I building for a hypothetical future? Three similar lines beat a premature abstraction. Build the naive, obviously-correct version first and optimize only after tests prove correctness.

```
Generic event bus with middleware for one notification  ->  a function call
Abstract factory for two similar components             ->  two components + shared util
Config-driven form builder for three forms              ->  three form components
```

### Scope discipline
Touch only what the task requires. Do not tidy adjacent code, reshuffle imports in files you are not changing, remove comments you do not understand, add "useful" extras, or modernize syntax in files you only read. If you notice something worth fixing, note it instead:

```
NOTICED BUT NOT TOUCHING:
- src/utils/format.ts has an unused import (unrelated)
- auth middleware error messages could be clearer (separate task)
-> Want me to create tasks for these?
```

### One thing at a time
Each increment changes one logical thing. A new component, a refactor of another, and a build-config change are three changes.

### Keep it compilable
After each increment the project builds and existing tests pass. Do not leave it broken between slices.

### Feature flags for unfinished work
If an increment must merge before the feature is ready, hide it behind a flag so users never see incomplete work: an environment/config flag, Firebase Remote Config, or a database-backed flag. Choose what the project already uses.

### Rollback-friendly
Each increment should be independently revertable. Additive changes are easy to revert; keep modifications to existing code small. Database migrations (Supabase/Postgres, Room, SQL) need a rollback or forward-fix plan, and destructive changes (drop column, rewrite data) are separate steps done after the new path is live. Do not delete and replace something in the same change.

## Increment checklist

After each increment, verify with the repository's own commands (discover them; do not assume npm):

- [ ] The change does one thing and does it completely
- [ ] Existing tests pass (`npm|pnpm test`, `go test ./...`, `./gradlew test`, `supabase test db`, ...)
- [ ] Build, type check, and lint pass where the stack has them
- [ ] The new behavior works as expected
- [ ] Auth, rules, and RLS implications were considered for anything touching data access

Run each verification after a change that could affect it; do not repeat a successful command on unchanged code.

## Common rationalizations

| Rationalization | Reality |
|---|---|
| "I'll test it all at the end" | A bug in slice 1 makes slices 2-5 wrong. Test each slice. |
| "It's faster to do it all at once" | Until something breaks and you cannot tell which of 500 changed lines did it. |
| "These changes are too small to separate" | Small slices are cheap; large ones hide bugs and make rollback painful. |
| "I'll add the feature flag later" | If it is incomplete it should not be user-visible. Add the flag now. |
| "This refactor is small enough to include" | Mixed changes are harder to review and debug. |

## Red flags

- 100+ lines written without running tests
- Several unrelated changes in one increment
- "Let me just add this too" scope expansion; touching files outside the task
- Skipping test or verify to move faster
- Build or tests broken between increments
- Abstractions built before a third use case needs them
- Committing without being asked

## Verification

- [ ] Each increment was individually tested
- [ ] The full suite passes and the build is clean
- [ ] The feature works end to end as specified (or the limits of verification were stated)
- [ ] Out-of-scope observations were reported, not silently fixed
