---
name: source-driven-development
description: "Grounds version-sensitive code in official documentation instead of memory. Use when writing or changing code against a versioned framework, SDK, or cloud API where signatures or best practices change (Next.js, React, Firebase, Supabase, AWS SDK, Go standard library or modules, Kotlin/Android/Jetpack Compose), when unsure of an API, when a deprecation or migration may apply, or when the user asks for verified or documented code. Not for pure logic, renames, or quick fixes. Indonesian triggers: cek dokumentasi resmi, API terbaru, versi library, apakah ini deprecated, sesuai dokumentasi."
---

# Source-Driven Development

For version-sensitive code, verify against official documentation instead of implementing from memory. Training data goes stale, APIs get deprecated, and best practices change. The goal is code whose important decisions trace to a source the user can check.

**Scope it to what matters.** Look up version-sensitive or non-obvious APIs and patterns (data fetching, auth, caching, routing, security rules, SDK initialization, lifecycle, Compose APIs). Do not fetch docs for every line, for pure logic that is the same across versions, for renames and typo fixes, or when the user explicitly wants speed.

## Process

```
DETECT -> FETCH -> IMPLEMENT -> CITE
```

### 1. Detect stack and versions
Read the dependency files and prefer the **lockfile** for exact versions; use the manifest only if there is no lockfile.

```
package.json + lockfile (package-lock.json, pnpm-lock.yaml, yarn.lock, bun.lock)  -> Node, Next.js, React
go.mod / go.sum                                   -> Go
build.gradle(.kts), libs.versions.toml            -> Kotlin, Android, AGP, Compose BOM
firebase.json, functions/package.json             -> Firebase
supabase/config.toml, @supabase/* versions        -> Supabase
pubspec.yaml / pubspec.lock                       -> Dart/Flutter
requirements.txt, pyproject.toml, Cargo.toml, Gemfile, composer.json  -> others
```

State what you found, for example `STACK: Next.js 15.x, React 19, @supabase/ssr x.y (from package-lock.json)`. Ask the user only if the version cannot be determined from the repo.

### 2. Fetch the relevant official page
Fetch the specific page for the feature, not a homepage. Use whatever access the tool provides: a web fetch tool, or a documentation MCP (for example Context7) if configured. Documentation served by an official docs MCP counts as an official source; unofficial summaries do not.

**Source hierarchy:**

| Priority | Source | Examples |
|---|---|---|
| 1 | Official docs | nextjs.org/docs, react.dev, firebase.google.com/docs, supabase.com/docs, docs.aws.amazon.com, go.dev/doc and pkg.go.dev, developer.android.com, kotlinlang.org/docs |
| 2 | Official blog, changelog, release notes, migration guides | |
| 3 | Standards references | MDN, web.dev |
| 4 | Compatibility data | caniuse.com |

Not authoritative as primary sources: Stack Overflow, tutorials and blog posts, AI-generated summaries, and your own memory.

Extract the key pattern and any deprecation or migration notes. If two official sources conflict, tell the user and check which works against the detected version.

**No web or docs access?** Say so, implement from memory, and mark each version-sensitive piece `UNVERIFIED` (see below). Do not pretend to have checked.

### 3. Implement what the docs show
- Use signatures from the docs, not memory; use the new pattern if the docs replaced the old one; avoid deprecated APIs; flag anything the docs do not cover as unverified.
- **Docs conflict with existing project code**: surface it, do not silently pick one.

```
CONFLICT: the codebase uses useState for form pending state;
React 19 docs recommend useActionState (react.dev/reference/react/useActionState).
A) modern pattern  B) match existing code  -> which do you prefer?
```

### 4. Cite
Give the user the sources for non-obvious, version-sensitive decisions.

- **Put citations in your reply**, with full URLs (deep links with anchors when possible) and a short quote if it supports a non-obvious decision. This is the default.
- **Code comments only** when a future reader needs the source to understand a version-specific workaround or a surprising choice. Do not add a citation comment to every call; it clutters code (and conflicts with the `antislop-code` skill).
- Mention runtime or platform support data when recommending platform features.
- If you could not find documentation:

```
UNVERIFIED: I could not find official documentation for this pattern.
It is based on training data and may be outdated. Verify before production use.
```

Honesty about what you could not verify beats false confidence.

## Common rationalizations

| Rationalization | Reality |
|---|---|
| "I'm confident about this API" | Confidence is not evidence; memory contains outdated patterns that look right. |
| "Fetching docs costs tokens" | A hallucinated signature costs more. One targeted fetch is cheap. |
| "The docs won't cover it" | Then the pattern may not be officially recommended; that is useful to know. |
| "I'll just say it might be outdated" | Verify and cite, or flag `UNVERIFIED`. A vague hedge helps nobody. |
| "Simple task, no need" | Simple patterns get copied into ten places. Check version-sensitive ones. |

## Red flags

- Writing version-sensitive code from memory without checking the installed version
- "I believe" or "I think" about an API instead of a source
- Citing Stack Overflow or blog posts as primary sources
- Using deprecated APIs because they appear in training data
- Fetching a whole docs site instead of the one relevant page
- Claiming a doc check you did not do, or citing without being able to fetch
- Citation comments on every line

## Verification

- [ ] Versions were identified from the lockfile or dependency files
- [ ] Version-sensitive patterns were checked against official docs for that version (or flagged `UNVERIFIED` when docs were unavailable)
- [ ] Sources are official; non-obvious decisions have full-URL citations in the reply
- [ ] No deprecated APIs were used (checked against migration guides where relevant)
- [ ] Conflicts between docs and existing code were surfaced to the user
