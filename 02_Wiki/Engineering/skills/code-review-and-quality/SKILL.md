---
name: code-review-and-quality
description: Multi-axis code review of a diff, branch, PR, or finished feature. Use when the user asks to review code ("review kode ini", "cek PR", "review my changes"), before merging, or when evaluating code written by another agent or a human. Review is read-only unless the user asks for fixes. Not for refactoring (see code-simplification).
---

# Code Review and Quality

Review a change across five axes: correctness, readability, architecture, security, performance.

**Approval standard:** approve when the change definitely improves overall code health and follows project conventions, even if it is not how you would have written it. Block for real problems, not preference.

## Review is read-only by default

Report findings; do not edit files, commit, or push unless the user asks you to fix. Write the review in the user's language (Indonesian or English); keep code identifiers as written.

## Step 0: Get the change

Pick the source from what the user said:

| User asks to review | Get it with |
|---|---|
| Uncommitted work | `git status`, `git diff` and `git diff --staged` |
| Current branch vs main | `git diff <base>...HEAD` and `git log <base>..HEAD --oneline` |
| A specific commit | `git show <sha>` |
| A PR | `gh pr view <n>` and `gh pr diff <n>` (if `gh` is available), else ask for the branch |
| A file or folder | read it directly |

If the base branch is unclear, check `git branch -a` or ask. Then read the surrounding code of each changed file, not just the diff lines.

## Process

1. **Context**: what is this change for, what behavior should change, which spec/task/issue? If unclear, ask or state your assumption.
2. **Tests first**: do tests exist, do they check behavior (not internals), do they cover edge and error cases, would they catch a regression? Run them if possible, and say if you did not.
3. **Implementation**: walk each changed file through the five axes.
4. **Categorize and order findings** (below).
5. **Verification story**: what was run, what builds, UI screenshots, manual checks. State what you verified versus what you only read.

## The five axes

### 1. Correctness
- Matches the spec or task? Edge cases (null, empty, boundaries)? Error paths, not just the happy path?
- Off-by-one, races, state inconsistency, stale closures, missing `await`?
- Do the tests test the right thing?

### 2. Readability and simplicity
- Descriptive, consistent names; straightforward control flow; logical grouping.
- Could it be done in fewer lines? Do abstractions earn their complexity (do not generalize before the third use)?
- A new conditional bolted onto an unrelated flow, or repeated conditionals on the same shape, signals a missing model or dispatcher. That is a design smell, not a nit.
- Dead artifacts: unused variables, compatibility shims, `// removed` comments.

### 3. Architecture
- Follows existing patterns, or introduces a new one with justification? Clean module boundaries, correct dependency direction, no circulars?
- Does a refactor **reduce** complexity or only relocate it? Count the concepts a reader must hold; prefer the version where branches or layers disappear.
- Feature-specific logic leaking into shared modules? A near-duplicate of an existing helper?
- Explicit type boundaries: question gratuitous `any`, `!!`, `as` casts, `lateinit`, and silent fallbacks that hide an unclear invariant.

### 4. Security
Use the `security-and-hardening` skill for depth. Quick checks:
- Input validated at boundaries; queries parameterized; output encoded; secrets out of code and logs.
- **Authorization** checked on every protected endpoint, Server Action, function, and RPC (not just authentication).
- Data rules changed? Review Firebase Security Rules, Supabase RLS policies, and IAM changes for over-permissive access and for missing deny-path tests.
- Service-role/admin keys never reach client code or an APK; external data treated as untrusted.

### 5. Performance
Use the `performance-optimization` skill for depth. Quick checks:
- N+1 queries or reads (including Firestore reads per screen), unbounded fetches, missing pagination, missing indexes for new filters and RLS columns.
- React: unnecessary client components, request waterfalls, heavy re-renders. Go: missing timeouts, unbounded goroutines, leaked connections. Android: main-thread I/O, unstable Compose parameters, leaked listeners.

### Stack-specific smells (check when the change touches them)
- **Next.js/React**: `'use client'` placed too high, secrets in `NEXT_PUBLIC_*`, Server Action without auth check, `useEffect` data fetching that the server could do.
- **Go**: ignored errors, `context` not propagated, goroutine without exit path, shared state without synchronization, missing `defer rows.Close()`.
- **Kotlin/Android**: work on the main thread, hardcoded dispatchers that block testing, exported components without need, `GlobalScope`, leaked `Context`.
- **SQL/migrations**: destructive changes without a rollback plan, missing indexes, RLS left disabled on new tables.

## Structural remedies

When you flag a structural problem, propose the move, not just the problem:
- Replace a chain of conditionals with a typed model or dispatcher.
- Collapse duplicate branches into one flow.
- Separate orchestration from business logic.
- Move feature-specific logic to the package that owns it; reuse the canonical helper.
- Make a type boundary explicit so downstream branching disappears.
- Delete a pass-through wrapper; extract a helper or split an oversized file.

Prefer the remedy that removes moving parts over one that spreads the same complexity around.

## Severity labels (use these exactly)

| Label | Meaning | Author action |
|---|---|---|
| **Critical** | Security hole, data loss, broken functionality | Blocks merge |
| **Required** | Real defect or clear regression, not cosmetic | Fix before merge |
| **Optional** | Worthwhile suggestion | Author's call |
| **Nit** | Style or taste | May ignore |
| **FYI** | Context only | No action |

**Lead with what matters.** Order by leverage: correctness and security, then structural problems, then the rest. A few high-conviction comments beat a long list; if there is one structural problem and ten nits, the structural problem is the review.

**Presumptive Required** (propose the simpler design; escalate to Required if the change makes structure worse): a refactor that relocates rather than reduces complexity; growing an already very large file without decomposition; feature logic in a shared module; a near-duplicate of an existing helper; a silent fallback hiding an unclear invariant.

## Output format

```
## Review: <title or branch>
Scope: <what was reviewed, how obtained>. Verified: <tests/build run, or "read only">

### Critical / Required
1. <file:line>: <problem>. <why it matters, quantified if possible>. Suggest: <fix>.

### Optional / Nit / FYI
- <file:line>: ...

### Verdict
Approve | Approve with comments | Request changes: <one-line reason>
```

Reference locations as `path/to/file:line`. If nothing is wrong, say what you checked rather than a bare "LGTM".

## Change sizing

~100 lines is easy to review, ~300 acceptable for one logical change, ~1000 should be split. Watch file size too: a small diff that pushes a file past ~1000 total lines is a signal to extract first. Splitting strategies: stack small dependent changes, vertical slices of a feature, shared code first then consumers, or separate groups for different reviewers. Large deletions and automated refactors are acceptable when intent is the only thing to verify. A refactor plus new behavior is two changes; ask for them to be separated.

## Dead code hygiene

After a refactor, list newly unreachable code (function, component, constant, with the reason) and **ask before deleting** anything you are not sure about.

## Honesty in review

- Do not rubber-stamp; "LGTM" with no evidence of review helps no one.
- Do not soften real issues; say plainly when something will hit production.
- Quantify when possible ("N+1 adds one query per list item") instead of "could be slow".
- Push back on approaches with clear problems and propose alternatives; do not flatter. If the author has context and disagrees, defer on judgment calls, not on facts.
- Comment on code, not people. Technical facts and data outrank preference; the project's style guide is authoritative on style.

## Dependency review

Before adding a dependency: does the existing stack already solve it, how big is it, is it maintained, any known vulnerabilities (`npm audit`, `govulncheck`, Gradle dependency check), is the license compatible? Prefer the standard library and existing utilities.

For upgrades:
1. Read the changelog; a "patch" can change behavior. Read migration notes for majors.
2. One dependency per change (or small related groups), so a break is attributable and revertable.
3. A green suite before and after is the verification; thin coverage around the dependency is the real finding.
4. Review the lockfile diff for transitive changes; never hand-edit lockfiles.

Supply-chain risk triage belongs to `security-and-hardening`.

## Common rationalizations

| Rationalization | Reality |
|---|---|
| "It works, good enough" | Unreadable, insecure, or mis-architected code compounds as debt. |
| "I wrote it, so I know it's correct" | Authors are blind to their own assumptions. |
| "AI-generated code is probably fine" | It is confident and plausible even when wrong; scrutinize more. |
| "Tests pass, so it's good" | Tests do not catch architecture, security, or readability problems. |
| "The refactor makes it cleaner" | Relocated complexity is not reduced complexity. |
| "It's just a version bump" | A bump is a behavior change you did not write. |

## Red flags

- Review that only checks that tests pass; "LGTM" without evidence
- Security-sensitive changes (auth, rules, RLS, IAM) with no deny-path tests
- Bug fix without a regression test
- Findings with no severity, or a real issue buried under nits
- A refactor that moves code without reducing concepts; new conditionals scattered into unrelated paths
- Bulk dependency bumps with no changelog review; hand-edited lockfiles
- You edited files during a review nobody asked you to fix

## Verification

- [ ] The change was actually retrieved (diff or files), and surrounding code read
- [ ] Each finding has a location, severity label, reason, and a suggested fix
- [ ] Critical and Required items are clearly separated from Optional, Nit, and FYI
- [ ] You stated what was run (tests, build) versus only read
- [ ] A verdict was given
- [ ] No files were modified unless the user asked
