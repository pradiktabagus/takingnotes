---
name: git-workflow-and-versioning
description: "Git practice for commits, branches, merge conflicts, undoing changes safely, worktrees, tags, semantic versioning, and changelogs. Use when the user asks to commit, branch, push, merge, rebase, resolve conflicts, revert, release, tag, or write a changelog or commit message. Not needed for ordinary code edits where no git action is requested. Indonesian triggers: commit, buat branch, merge conflict, gabungkan branch, batalkan perubahan, rilis versi, changelog."
---

# Git Workflow and Versioning

Commits are save points, branches are sandboxes, history is documentation. Disciplined version control keeps changes reviewable and reversible.

## Agent safety rules (read first)

- **Commit, push, tag, merge, or open a PR only when the user asks.** Otherwise leave changes in the working tree and, if useful, propose a commit message.
- **Never run destructive commands without explicit approval**: `git reset --hard`, `git clean -fd`, `git checkout -- .`, `git push --force` (use `--force-with-lease` only when the user asked), `git branch -D`, history rewrites on shared branches.
- **Undo only your own work.** The working tree may contain the user's uncommitted changes. To undo what you changed, restore specific files (`git restore <path>`), or `git stash` first. Check `git status` and `git diff` before and after.
- **Never skip hooks** (`--no-verify`) or change git config unless asked. If a hook fails, fix the cause and make a new commit.
- **Match the repository.** Read `git log --oneline -15` and follow its message style, language (Indonesian or English), and branch naming before applying any convention below.
- Never commit secrets or files such as `.env`, keystores, or service-account JSON (see Pre-commit).

## Commits

**Atomic**: each commit does one logical thing and leaves the build working. Keep formatting changes, refactors, dependency bumps, and features in separate commits.

**Size**: ~100 lines is easy to review; ~300 is acceptable for one logical change; split anything near 1000. Split by layer, by vertical slice, or refactor-then-feature.

**Messages explain why.** If the repo has no clear convention, use:

```
<type>: <short description>

<optional body: why this change, not what the diff shows>
```

Types: `feat`, `fix`, `refactor`, `test`, `docs`, `chore`. Bad: "update", "fix", "misc".

When the user asks you to commit: review `git status` and `git diff --staged`, stage only the intended files by name, write the message, commit, and report the result.

## Branching

- Default recommendation: short-lived branches off the main branch, merged within a few days; feature flags over long-lived branches. If the project uses gitflow or release branches, follow that; commit discipline matters more than the model.
- Names: `feature/<slug>`, `fix/<slug>`, `chore/<slug>`, `refactor/<slug>` unless the repo does otherwise.
- Delete branches after merge (only when asked).
- Worktrees (`git worktree add ../proj-a feature/a`) let you run separate branches in parallel directories without stashing. Remove with `git worktree remove` when done. Use only when the user wants parallel work.

## Merge conflicts

1. `git status` to list conflicted files; do not guess, read each conflict block.
2. Understand both sides: `git log --merge -p -- <file>` or compare `git diff --ours --theirs`. Know what each side intended.
3. Resolve by combining intent, not by picking a side blindly. Remove all markers (`<<<<<<<`, `=======`, `>>>>>>>`).
4. For lockfiles and generated files, do not hand-merge: take one side then regenerate (`npm install`, `go mod tidy`, Gradle sync, `supabase db diff`).
5. Run build and tests, then `git add` the resolved files. Complete with the merge/rebase continue command.
6. If the conflict is large or the intent unclear, stop and ask the user. `git merge --abort` / `git rebase --abort` return to the pre-merge state safely.

**Merge vs rebase**: follow the team's convention. Rebasing local, unpushed commits is fine; never rebase or amend commits that others have pulled without the user's approval.

## Undoing things safely

| Goal | Command | Notes |
|---|---|---|
| Discard my edits to one file | `git restore <path>` | Check `git diff <path>` first; it is irreversible |
| Unstage | `git restore --staged <path>` | Keeps the changes |
| Undo a pushed commit | `git revert <sha>` | Creates a new commit; safe on shared branches |
| Fix the last unpushed commit | `git commit --amend` | Only if not pushed, and only when asked |
| Set work aside | `git stash push -m "msg"` | Recoverable with `git stash pop` |
| Recover "lost" commits | `git reflog` | Then `git branch rescue <sha>` |

## Pre-commit hygiene

```bash
git status                  # what is included? any unexpected files?
git diff --staged           # review the actual change
```

- **Secrets**: run a scanner such as `gitleaks protect --staged` if installed; a plain `grep` for "token" gives noise and misses real keys. If a secret reaches a remote, rotate it first, then purge history.
- **Run the repo's own checks** (discover them; do not assume npm): `npm|pnpm test && lint && tsc --noEmit`, `go vet ./... && go test ./...`, `./gradlew test lint`, `supabase test db`.
- Hooks: use what the repo already has (husky, lefthook, pre-commit). Do not add a new hook system unasked.

## Ignore rules and generated files

- Commit generated files only if the project expects them (lockfiles, Prisma/Supabase migrations, `go.sum`).
- Do not commit: `node_modules/`, `.next/`, `dist/`, `bin/`, `.env*` (keep `.env.example`), IDE-local config.
- Stack-specific: **Android** `build/`, `.gradle/`, `local.properties`, `*.jks`/`*.keystore`, signing passwords; **Firebase** service-account JSON, `.firebase/`; **Supabase** `.env`, `supabase/.temp/`; **AWS** credentials files, `*.pem`, `*.tfstate`, `.terraform/`; **Go** build outputs and local env files.
- If a `.gitignore` is missing, propose one before the first commit.

## Using git to debug

```bash
git log --oneline -20                 # what changed recently
git diff HEAD~5..HEAD -- src/         # diff a range
git blame -L 40,60 path/to/file       # who last touched these lines
git log --grep="validation" --oneline # search messages
git bisect start; git bisect bad HEAD; git bisect good <sha>   # then test each step
git bisect reset                      # always finish by resetting
```

`git bisect run <script>` automates the search; it leaves the repo on a detached commit until `git bisect reset`.

## Change summaries

After modifying code, give a short summary: files changed and why, things deliberately **not** touched (out of scope), and open concerns or assumptions. The "not touched" list shows scope discipline and surfaces issues without unrequested edits.

## Release and versioning

A version is the contract your consumers rely on.

**Semantic Versioning** `MAJOR.MINOR.PATCH`: major = breaking, minor = backward-compatible feature, patch = backward-compatible fix. If consumers could observe the change breaking, it is major, whatever the diff size. When unsure, assume breaking.

**Tag releases**, and derive the version from the tag so artifact, tag, and changelog agree:

```bash
git tag -a v1.4.0 -m "Release 1.4.0"
git push origin v1.4.0     # only when the user asks
```

Stack notes:
- **Go modules**: tags are the version (`v1.4.0`); a major v2+ requires the module path to end in `/v2`.
- **Android**: `versionName` is the human version; `versionCode` must increase on every Play upload. Keep both in one place (Gradle) and tie to the tag.
- **npm packages**: bump via `npm version` so the tag and `package.json` stay in sync.

**Changelog** is curated for consumers, not `git log`. Group by Added / Changed / Fixed / Deprecated / Removed / Security, newest first, phrased by user impact. Write each entry with the change itself. Breaking changes need a migration note and a deprecation window.

```markdown
## [1.4.0] - 2026-06-12
### Added
- Bulk task import via CSV
### Fixed
- Timezone drift in recurring due dates
### Deprecated
- `GET /v1/tasks/all`, use paginated `GET /v1/tasks` (removal in 2.0)
```

## Common rationalizations

| Rationalization | Reality |
|---|---|
| "I'll commit when the feature is done" | One giant commit is hard to review, debug, or revert. |
| "The message doesn't matter" | Messages are documentation for future you and future agents. |
| "I'll split this later" | Split before submitting, not after. |
| "`reset --hard` is a quick undo" | It deletes uncommitted work, including the user's. Restore specific files or stash. |
| "Force push is fine, it's my branch" | Only if the user asked and nobody else has pulled it. |
| "No .gitignore needed" | Until `.env` or a keystore gets committed. |
| "Small fix, bump the patch" | Check what consumers can observe. |
| "The changelog is the commit log" | Commits are for you; the changelog is for consumers. |

## Red flags

- Commits, pushes, or tags made without being asked
- `reset --hard`, `clean -fd`, `--force`, or `--no-verify` used casually
- Secrets, keystores, service accounts, or `.env` files staged
- Messages like "fix", "update", "misc", or ignoring the repo's message style
- Formatting changes mixed with behavior changes
- Hand-merged lockfiles
- Breaking change under a minor or patch version; release without tag; changelog that is dumped commit messages

## Verification

Per commit (when committing was requested):
- [ ] One logical change; build and tests pass using the repo's own commands
- [ ] Only intended files staged; no secrets; message explains why and matches repo style

Per release:
- [ ] Version bump matches the change; release is tagged; changelog entry is curated
