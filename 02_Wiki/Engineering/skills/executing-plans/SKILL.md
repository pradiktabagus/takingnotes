---
name: executing-plans
description: "Use when executing a written implementation plan (from writing-plans) in the current session - works task by task with TDD, tracks progress by ticking the plan checklist, never commits, and ends with a fresh-context review. Indonesian triggers: jalankan rencana, eksekusi plan, kerjakan sesuai rencana, lanjutkan plan."
---

# Executing Plans

Execute a plan from `writing-plans` yourself, task by task, in this session. Progress lives in the plan file: ticked checkboxes plus lines in its Progress section. Nothing is committed; every change stays in the working tree for the user to review and commit.

**Core principle:** The plan already did the thinking. Execute it exactly, prove each step with a test you watched fail and then pass, and leave a record in the plan that survives your own forgetting.

**Narration:** between tool calls, at most one short line. The plan's checkboxes and the tool results carry the record.

**Continuous execution:** once the user approved execution, do not pause between tasks to ask "should I continue?". Execute every task.

**Rulings, not stalls.** Conflicts, ambiguities, plan defects: decide them. The spec (or the plan's Intent section) is the binding authority, the plan is its argument, and your judgment settles what neither answers. Record each decision in the plan's Progress section as `Task <N>: Ruling: <what you decided> — <why> — <cost if wrong>` and keep going. Deviating from the plan without a recorded ruling is a decision made in secret.

**Five things stop you, and only these:**

1. an irreversible or destructive operation
2. a security-sensitive action (auth, rules, RLS, IAM, secrets)
3. any git state change: commit, push, branch, merge, stash, reset (this workflow never does them; propose instead)
4. a side effect outside the repo (deploy, publish, migration on a shared database, paid API calls)
5. a plan so broken that every path forward is a guess

For those, stop and ask.

## When to Use

- The user approved a plan from `writing-plans` and asked you to execute it, or asked you to resume one ("lanjutkan plan").
- Not for work without a plan: use `incremental-implementation` for that, or `writing-plans` first.

## Setup

1. **Read the plan once**, note its Global Constraints and Review Focus, and create a todo per task. Read the spec it names, or its Intent section. A plan with neither gets a Progress note saying so; rulings made without one are provisional.
2. **Check the Progress section.** Tasks with a `Task <N>: complete` line are done: do not redo them, resume at the first task without one. After compaction, trust the Progress section, the ticked boxes, and `git diff` over your recollection. Spot-check that a "complete" task's files really contain its changes before building on them.
3. **Record the baseline** on a fresh run: `git rev-parse HEAD` and `git status --short` go on the `Baseline:` line. If the working tree already has uncommitted changes, tell the user before Task 1 and suggest they commit or stash first; you never do it for them. If they continue anyway, list those files in Baseline. A task that must edit one of them is a stop: their changes and yours would become inseparable.
4. **Load `test-driven-development`** before Task 1. It governs every step below; a plan whose steps already say "write the failing test first" does not exempt you from reading it.
5. **Pre-flight scan.** For every task that consumes what an earlier task produces (the Interfaces blocks), compare the two and add one Progress row per pair with what you found. Rule on each conflict with the spec as the authority. A plan whose tasks share nothing gets `Pre-flight: no shared interfaces`.

## The Task Loop

Everything you print stays in your context for the rest of the session. Redirect long test output to a temp file outside the repo and read its tail.

### 1. Take the task

Re-read the task's own text in the plan, even if you remember it: what you remember is a summary, the plan has the exact values, signatures, and test cases. Mark its todo in progress.

### 2. Work the steps

The steps are already in RED-GREEN order; follow that order. A test step's code is written and run first. Watching it fail is a step, not a formality: a test that passes before the implementation exists is a finding about the test.

Every step that runs a command has an `Expected:` line. Run it, read the output, compare:

- **Matches:** tick the box, next step.
- **The code is wrong:** use `debugging-and-error-recovery`. Find the cause; never patch the symptom to make the output match.
- **The plan is wrong** (contradicts the spec, an interface doesn't match what an earlier task produced, a command that cannot work): rule on the smallest change that satisfies the spec, record the ruling, continue. Later tasks touching the same interface read the ruling from Progress.

Tick boxes in the same tool call as the work they record, not in a call of their own.

### 3. The completion contract

Before writing a task's completion line, all of these are true, with evidence from this session (not inferred from the diff looking right):

- Every test the task names exists and ran in this task, and you read the output.
- The task's final test run (its Checkpoint step) passed.
- Every `Expected:` line was compared against real output.
- Every deviation has a `Ruling:` line.
- No file outside the task's Files list changed, except as a recorded ruling (check `git status --short` against Baseline).

`verification-before-completion` governs the claim. If anything is missing, the task is not complete: finish it.

### 4. Complete the task

Append `Task <N>: complete — tests: <command> → <result line>` to Progress, tick the Checkpoint box, mark the todo complete, take the next task. A failing run records nothing.

## Final Review

When every task is complete, get one fresh-context review of the whole change:

- **Scope:** `git diff <Baseline sha>` plus new untracked files (`git status --short`), excluding files listed in Baseline as pre-existing changes.
- **With a subagent tool:** dispatch one reviewer on the most capable available model. Give it the diff scope, the plan and spec paths, the plan's Review Focus verbatim, and a pointer to the Progress `Ruling:` lines. Tell it to follow `code-review-and-quality` (read-only, Set aside list, severity labels). Do not give it your session history, and do not replace it with your own read of the diff.
- **Without a subagent tool:** do that review yourself as a separate pass, write `Final review: self-review (no subagent tool)` in Progress, and say so in your final message. A self-review by the author is weaker; the user decides whether it is enough.

Sort the findings before acting on any of them. The reviewer's labels are advice; the gate is yours. Re-grade by effect: what a reasonable user of this software gets if it ships, not whether the spec mentions the trigger.

- **Critical and Required:** fix them yourself in ONE pass. Each fix gets a test that reproduces the finding, watched failing, then passing, then the full suite. Record `Final: fixed <finding> — <test> RED→GREEN, suite <result>`. No second review and no second fix pass.
- **Optional, Nit, FYI:** record `Final: deferred: <one line>`. Do not fix them; the user decides.
- **A finding you decide not to fix**, and every line in the reviewer's Set aside list, is a ruling: `Final: Ruling: <finding> — <why the code stands> — <cost if wrong>`.

## Finish

Your final message carries, exhaustively and in order:

- **What changed:** files, grouped by task.
- **Rulings I made:** every `Ruling:` line from Progress, each with its cost if wrong. This is the only place the decisions you took on the user's behalf reach them.
- **Deferred:** every `Final: deferred` line.
- **Verification:** the last full-suite command and result, plus anything you could not verify (UI, device, production).
- **Suggested commits:** one message per task or logical group, in the repo's style (`git log --oneline -15`). Do not commit.

Leave the plan file and its Progress section in place; it is the record.

## Common Rationalizations

| Excuse | Reality |
|--------|---------|
| "I remember what Task N says" | You remember a summary. The plan has the exact values. Re-read it. |
| "The plan's code is right, skip watching the test fail" | A test you never saw fail proves nothing. It is one step. Run it. |
| "I'll run the full suite at the end instead of per step" | Per-step runs show which step broke it. The Checkpoint run is the contract, not a substitute. |
| "The plan is wrong here, I'll just do the right thing" | Do the right thing and record the ruling. |
| "I'll update Progress after a few tasks" | Compaction does not wait. One line per task, in the same call as the work. |
| "Let me check in before the next task" | The user already approved execution. Only the five stops stop you. |
| "A quick commit per task would make this cleaner" | Git state is the user's. Propose commit messages at the end. |
| "I read my own diff carefully; the final reviewer is redundant" | Same author, same blind spots. The reviewer is the one fresh context this run buys. |
| "The reviewer said Optional, so it's Optional" | Re-grade by what the user of the software gets, then gate. |
| "The fix is obvious, no need for a failing test first" | The failing test is the only proof the finding was real and is now gone. |
| "I'll fix the nits too while I'm in there" | Each one is a change the user didn't ask for. Record it as deferred. |

## Red Flags

- Any `git commit`, `git stash`, `git checkout -b`, or `git reset` during execution
- A task marked complete with no test output read in this session
- Ticked boxes with no matching Progress line, or the reverse
- Changes in files outside the task's Files list with no ruling
- Editing a file that had the user's pre-existing changes
- Skipping the final review, or replacing it with your own read without saying so
