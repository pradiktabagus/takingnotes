---
name: code-simplification
description: Refactors code for clarity without changing behavior. Use when the user asks to simplify, clean up, refactor, or make code more readable ("rapikan", "sederhanakan"), or when working code has deep nesting, long functions, unclear names, or duplication. Not for code review (see code-review-and-quality), new features, or performance work.
---

# Code Simplification

Reduce complexity while preserving exact behavior. The goal is not fewer lines; it is code that is easier to read, change, and debug. Test every change with: "Would a new team member understand this faster than the original?"

**Do not use when** the code is already clear, you do not yet understand what it does, the "simpler" form would be measurably slower in a hot path, or the module is about to be rewritten.

## Five principles

### 1. Preserve behavior exactly
Same output for every input, same errors, same side effects and ordering. If unsure a change preserves behavior, do not make it.

Behavior traps that look harmless:
- `a || b` vs `a ?? b` (empty string, `0`, `false`): only swap when you know the value domain.
- Removing `async`/`await` changes where thrown errors surface (sync throw vs rejected promise) and stack traces.
- Reordering side effects, short-circuit evaluation, or evaluation of arguments.
- Go: changing `defer` order, shadowing `err`, nil vs empty slice/map. Kotlin: `?:` vs `if`, `lazy` timing, `let` on nullable returning differently.

### 2. Follow project conventions
Make code more consistent with the codebase, not with your taste. First read the project's rules (`AGENTS.md`, `CLAUDE.md`, steering files, `.rules`, lint and formatter config) and neighboring code: imports, naming, error handling, type annotation depth. Simplification that breaks consistency is churn.

### 3. Prefer clarity over cleverness
Explicit beats compact when the compact form needs a mental pause. Replace nested ternaries with `if`/`switch`/lookup; replace dense `reduce` chains with a loop and a named step.

### 4. Keep balance
Avoid over-simplifying: inlining a helper that named a concept, merging unrelated logic, removing an abstraction that exists for testability or extension, or optimizing for line count.

### 5. Scope to what changed
Default to recently modified code or the area the user named. No drive-by refactors of unrelated code.

## Process

### Step 1: Understand before touching (Chesterton's Fence)
Before changing or removing anything, answer: what is its responsibility, what calls it and what does it call, what are the edge and error paths, which tests define the behavior, why might it be written this way (performance, platform quirk, history)? Use `git blame` / `git log -p` on the lines. If you cannot answer, read more first.

### Step 2: Establish a safety net
Run the existing tests first. **If there are none** around the code, either add characterization tests that pin current behavior, or limit yourself to mechanical changes (renames, guard clauses) and say the safety net is missing. Do not claim behavior is preserved without evidence.

### Step 3: Find opportunities

| Pattern | Signal | Simplification |
|---|---|---|
| Deep nesting (3+) | Hard control flow | Guard clauses / early return, helper functions |
| Long function (50+ lines) | Several responsibilities | Split into named functions |
| Nested ternaries | Mental stack | `if`/`switch`/lookup map |
| Boolean flag parameters | `doThing(true, false)` | Options object or separate functions |
| Repeated conditionals | Same check in many places | Named predicate |
| Generic or misleading names | `data`, `tmp`, `get` that mutates | Rename to what it is |
| Comments saying "what" | `// increment counter` | Delete; keep comments that say **why** |
| Duplicated logic | Same 5+ lines repeated | Extract (only if it is truly the same concept) |
| Dead code | Unreachable, unused, commented-out | Remove after confirming it is dead (ask if unsure) |
| Needless abstraction | Wrapper adding nothing; factory of a factory | Inline or use the direct approach |
| Redundant assertions | Casts to an already inferred type | Remove |

### Step 4: Apply incrementally
One simplification at a time; run the tests after each. If tests fail, revert that step and reconsider. Do not batch changes you cannot attribute. Keep refactoring separate from feature or bug-fix work. Make a checkpoint commit only if the user has asked for commits. For very large mechanical changes (hundreds of lines), prefer a codemod or the language's refactoring tool (IDE rename, `gofmt -r`, `gopls` rename, Android Studio refactor) over manual edits.

### Step 5: Verify the whole
Compare before and after: genuinely easier to understand? new patterns inconsistent with the codebase? diff clean and reviewable? If the result is harder to follow or review, revert. Not every attempt succeeds.

## Examples

### TypeScript / React

```typescript
// Guard clauses instead of nesting (same behavior)
function process(data?: Data) {
  if (!data) throw new TypeError('Data is required');
  if (!data.isValid()) throw new ValueError('Invalid data');
  if (!data.hasPermission()) throw new PermissionError('No permission');
  return doWork(data);
}

// Redundant boolean return
const isValid = (s: string) => s.length > 0 && s.length < 100;

// Truthiness vs nullish: NOT equivalent for '' or 0.
// Only replace `if (x) a = x; else a = y` with `x || y`; use `??` when only null/undefined should fall back.
```

In React, if the "simpler" form recomputes values or hides intent, leave it. Prop drilling through several components is a judgment call (context or composition): flag it, do not auto-refactor.

### Go

```go
// Remove else after return; wrap errors with context
if err != nil {
    return fmt.Errorf("load config: %w", err)
}
return cfg, nil
```
- Prefer early returns, small functions, and clear names over clever one-liners. Keep errors explicit; do not hide them to shorten code.
- Replace manual loops with `slices`/`maps` helpers only when the repo's Go version supports them and clarity improves.
- Run `gofmt`/`goimports` and `go vet`; never remove `defer Close()` or context propagation in the name of brevity.

### Kotlin

```kotlin
// when instead of an if/else-if chain
fun label(item: Item) = when {
    item.isNew -> "New"
    item.isUpdated -> "Updated"
    item.isArchived -> "Archived"
    else -> "Active"
}
```
- Use `data class`, `sealed interface`, `?.`/`?:`, and `map`/`filter` where they read better. Avoid stacking scope functions (`let`/`run`/`apply`/`also`) into dense chains.
- Keep coroutine structure (dispatchers, cancellation, `supervisorScope`) intact; changing it changes behavior.
- In Compose, do not "simplify" by moving state reads or object creation in a way that changes recomposition.

## Common rationalizations

| Rationalization | Reality |
|---|---|
| "It works, don't touch it" | Hard-to-read working code is hard to fix when it breaks. |
| "Fewer lines is simpler" | A nested one-line ternary is not simpler than a 5-line `if`. |
| "I'll tidy this unrelated code too" | Unscoped changes make noisy diffs and risk regressions. |
| "This abstraction might be useful later" | Speculative abstractions are cost without value; remove and re-add when needed. |
| "The original author must have had a reason" | Check blame (Chesterton's Fence), but accumulated complexity often has no reason. |
| "I'll refactor while adding this feature" | Mixed changes are harder to review and revert. |

## Red flags

- Tests had to be modified to pass (you probably changed behavior)
- The "simpler" version is longer or harder to follow
- Renaming to personal taste instead of project convention
- Removing error handling or `defer`/cleanup to look cleaner
- Simplifying code you do not understand
- Large batched change with no incremental verification
- Touching code outside the requested scope
- Claiming "behavior preserved" with no tests and no reasoning

## Verification

- [ ] Existing tests pass without modification (or characterization tests were added first, or the missing safety net was stated)
- [ ] Build and linter/formatter pass with no new warnings
- [ ] Each change is incremental and reviewable; the diff has no unrelated edits
- [ ] Follows project conventions (checked against `AGENTS.md`/`CLAUDE.md`/steering and neighboring code)
- [ ] No error handling, cleanup, or `defer`/cancellation was removed or weakened
- [ ] No dead code left behind (unused imports, unreachable branches)
- [ ] The result is a net improvement that a teammate would approve
