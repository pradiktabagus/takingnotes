---
name: security-and-hardening
description: Hardens code against vulnerabilities. Use when handling user input, authentication/authorization, secrets, file uploads, database access rules (Firebase Rules, Supabase RLS), cloud permissions (AWS IAM/S3), Go backend services, Android apps, or external integrations. Also use for security review, "keamanan", or "is this safe" questions.
---

# Security and Hardening

Treat every external input as hostile, every secret as sacred, every authorization check as mandatory. Security is a constraint on every line that touches user data, auth, or external systems.

Stack covered: Next.js / React / TypeScript, Firebase, Supabase, AWS, Go backends, Kotlin / Android. Load only the sections that match the code at hand.

## Handling secrets while working (agent rules)

- When reading config files (`.env`, `opencode.jsonc`, `settings.json`, `firebase.json`, `*.tfvars`), **never print secret values** in output. Check structure, redact tokens (`figd_***`).
- If you see a secret in a file you did not expect to hold one, tell the user, name the file, and recommend rotation. Do not repeat the value.
- Never commit, log, or paste secrets. Prefer env vars or a secret manager over inline values in tool configs.

## Threat model first (5 minutes)

1. **Trust boundaries**: where untrusted data enters (HTTP, forms, uploads, webhooks, deep links, third-party APIs, queues, LLM output).
2. **Assets**: credentials, PII, payments, admin actions, money movement.
3. **STRIDE per boundary**: Spoofing (authn), Tampering (integrity, parameterized queries), Repudiation (audit logs), Information disclosure (field allowlists, generic errors), DoS (rate limits, size caps, timeouts), Elevation of privilege (authz, least privilege).
4. **Abuse cases**: for each feature ask "how would I misuse this?" and make that the first test.

If you cannot name the trust boundaries, you are not ready to secure the feature.

## Boundary rules

**Always**
- Validate all external input at the boundary (schema validation, size caps).
- Parameterize every query; never concatenate user input into SQL or shell.
- Use framework output escaping; do not bypass it.
- Check **authorization**, not just authentication, on every protected action.
- Hash passwords with argon2id, scrypt, or bcrypt (or delegate to Firebase Auth / Supabase Auth / Cognito).
- Run the native dependency audit before release.

**Ask the user first**
- New or changed auth flows, new roles or permissions
- Storing new categories of sensitive data (PII, payments)
- New external integrations, CORS changes, file upload handlers
- Changing rate limits, IAM policies, security rules, or RLS policies

**Never**
- Commit secrets; log secrets/tokens/PII; expose stack traces to users
- Trust client-side validation or client-supplied roles/ids as authorization
- Use `eval` or raw HTML injection with user data
- Disable TLS verification or security headers "for convenience"

## Next.js / React / TypeScript

- **Server Actions and Route Handlers are public HTTP endpoints.** Authenticate and authorize inside each one, and validate input with zod (or equivalent). A hidden button is not access control.
- **Do not rely on middleware alone for auth.** A past middleware bypass (CVE-2025-29927) showed it is one layer, not the gate. Re-check the session next to data access (a data-access layer function such as `getCurrentUser()`).
- **Env vars**: only `NEXT_PUBLIC_*` reach the browser. Never put secrets there. Mark server-only modules with `import 'server-only'`. Do not pass whole DB records to Client Components; pass a DTO with allowed fields.
- **XSS**: React escapes by default. `dangerouslySetInnerHTML` and `href={userUrl}` (`javascript:` URLs) need sanitization (DOMPurify) and scheme allowlisting.
- **Headers**: set via `headers()` in `next.config` (HSTS, `X-Content-Type-Options`, `frame-ancestors` in CSP, `Referrer-Policy`). Use a nonce-based CSP if inline scripts are needed rather than `unsafe-inline`.
- **Sessions on web**: prefer `httpOnly; secure; sameSite=lax` cookies. Firebase and Supabase client SDKs persist tokens in browser storage by default; if you accept that, XSS defenses (CSP, no untrusted HTML) become critical, or use the SSR cookie flow (`@supabase/ssr`, Firebase session cookies).
- **Open redirects**: validate `next`/`returnTo` params against an allowlist of internal paths.
- **SSRF**: any server-side fetch of a user-influenced URL needs a host allowlist, https-only, rejection of private/loopback/link-local IPs (including `169.254.169.254`), and `redirect: 'error'`. DNS rebinding can still bypass a check-then-fetch; for high-risk surfaces pin the resolved IP.

## Firebase

- **Security Rules are the access control.** Default deny. Never ship `allow read, write: if true` or time-based "test mode" rules.
- Base rules on `request.auth.uid` and on data in the document, never on client-supplied fields. Validate writes: `request.resource.data` keys, types, sizes, and immutable fields.
- **Roles via custom claims**, set only from the Admin SDK on a trusted backend. Never store role in a user-writable document and trust it.
- **Storage rules**: restrict by path owner, `request.resource.size`, and `contentType`.
- **Firebase config / API key in the web app is public by design.** Protect data with Rules and App Check, and restrict the key by referrer/API in Google Cloud. The **Admin SDK service account JSON is a secret**: never in client code or git.
- **Callable / HTTPS Cloud Functions**: verify `context.auth` (callable) or the ID token (HTTP) and check authorization; add App Check enforcement for abuse-prone endpoints.
- **Test rules** with the emulator and `@firebase/rules-unit-testing`: one test that must be denied for every rule that matters.

## Supabase (Postgres)

- **Enable RLS on every table in an exposed schema** (`public`), and write policies before exposing the table. RLS on with no policy means deny; RLS off means the anon key can read everything.
- **`service_role` key bypasses RLS: server-only, never in client code, never `NEXT_PUBLIC_`.** The anon key is public by design; RLS is what protects data.
- Policies use `auth.uid()` (wrap as `(select auth.uid())` for performance). **Do not base policies on `user_metadata`**: users can edit it. Use `app_metadata` (server-set) or a roles table.
- Views bypass RLS by default; create them `with (security_invoker = true)`. `SECURITY DEFINER` functions need an explicit `set search_path = ''` and should not be exposed via RPC unless intended.
- An `UPDATE` policy usually also needs a matching `SELECT` policy; add `with check` so users cannot change ownership columns.
- **Storage**: policies on `storage.objects` by bucket and owner path; avoid public buckets for private data.
- **Server-side**: use `supabase.auth.getUser()` (validates with the Auth server) for authorization decisions, not `getSession()` alone.
- Test policies by querying as different roles (anon, user A, user B) and confirming cross-user access fails.

## AWS

- **IAM least privilege**: no `*` actions or resources in production policies; scope by resource ARN and condition. Use roles (Lambda, ECS, EC2 instance profiles, OIDC for CI) instead of long-lived access keys. Never commit access keys.
- **S3**: Block Public Access on; serve public files through CloudFront. Presigned URLs: short expiry, fixed key and content-type, size limit (use presigned POST conditions for uploads). Enable default encryption.
- **Secrets**: Secrets Manager or SSM Parameter Store (SecureString), not plain Lambda/ECS environment values for high-value secrets. Rotate.
- **Network**: security groups default closed, no `0.0.0.0/0` on SSH/DB ports; databases not publicly accessible; enforce IMDSv2 on EC2.
- **Detective controls**: CloudTrail on, alarms on root usage and policy changes. Do not log secrets or full request bodies.
- Ask before changing IAM policies, bucket policies, or security groups.

## Go backends

- SQL: `db.QueryContext(ctx, "... WHERE id = $1", id)`, or sqlc/ORM with bound parameters. Never `fmt.Sprintf` into SQL.
- **Server timeouts**: set `ReadHeaderTimeout`, `ReadTimeout`, `WriteTimeout`, `IdleTimeout` on `http.Server`; the default has none (slowloris). Cap bodies with `http.MaxBytesReader`. Use `context` deadlines on outbound calls and DB queries.
- HTML output: `html/template`, never `text/template`. Commands: `exec.Command(name, args...)` with fixed binary and args; never `sh -c` with user input.
- Randomness for tokens: `crypto/rand`, never `math/rand`. Compare secrets with `subtle.ConstantTimeCompare`. Passwords: `golang.org/x/crypto/argon2` or `bcrypt`.
- **JWT**: pin the expected algorithm, verify `exp`, `iss`, `aud`; reject `none`; do not accept the key type from the token.
- File paths: reject traversal (`..`); prefer `os.Root` (Go 1.24+) or `filepath.Rel` checks against a base dir.
- Never set `InsecureSkipVerify: true` outside tests. Return generic errors to clients; log details server-side.
- Tooling: `govulncheck ./...` (reachability-aware), `gosec`, `go vet`, run tests with `-race`.

## Kotlin / Android

- **No secrets in the APK.** API keys in `BuildConfig`, resources, or strings are extractable. Put real secrets behind your backend; use Play Integrity / Firebase App Check to gate access. R8 obfuscation is not protection.
- **Network**: HTTPS only; keep `usesCleartextTraffic` off and define a `network_security_config`. Certificate pinning only with a rotation plan (a bad pin bricks the app).
- **Storage**: tokens and keys in Android Keystore-backed storage (Keystore + DataStore with Tink, or equivalent). Note `androidx.security:security-crypto` is deprecated; check the current recommendation. No sensitive data in plain SharedPreferences, external storage, or logs (`Log.d` with PII).
- **Components**: declare `android:exported` explicitly; export only what must be. Validate every incoming Intent and deep link parameter as untrusted. Use `PendingIntent.FLAG_IMMUTABLE`. Share files via `FileProvider`, not file paths.
- **WebView**: JavaScript off unless needed; no `addJavascriptInterface` with untrusted content; restrict allowed origins; `allowFileAccess` off.
- **Backups and screens**: review `allowBackup` / data extraction rules for sensitive apps; use `FLAG_SECURE` on sensitive screens.
- **Biometrics**: bind to a Keystore key with `BiometricPrompt.CryptoObject`, not a boolean "success" callback.
- Check dependencies with Gradle dependency verification and the Android Studio / Play Console security hints.

## Input validation and uploads

Validate at the boundary with a schema (zod in TS; struct tags plus explicit checks in Go; sealed/validated models in Kotlin). Return `422` with field errors, not internals.

Uploads: allowlist types, cap size, do not trust extension or `Content-Type` alone (check magic bytes when it matters), store outside the web root or in object storage with a random name, and serve with a safe `Content-Type`.

## Dependencies and supply chain

- Find the real install root and package manager (lockfile, `packageManager`, CI). Stop if lockfiles compete.
- Run the native audit (`npm|pnpm|yarn audit`, `govulncheck`, Gradle dependency check). Triage by reachability: critical/high and reachable = fix now; unreachable = fix soon with a note; dev-only/low = backlog. Record the reason and a review date for any deferral.
- **Never run forced remediation automatically** (`npm audit fix --force`). Preview, read changelogs, test.
- Install scripts: do not blanket-approve. Review new dependencies (owner, maintenance, release age, typosquats such as `crossenv`) together with lockfile diffs.
- Audits only catch known advisories, not a new malicious package.

## Rate limiting and abuse

Limit auth endpoints strictly (for example 10 attempts per 15 minutes per IP and per account), general APIs more loosely. Use the platform's mechanism: middleware in Go (`x/time/rate` or a gateway), API Gateway / WAF on AWS, Firebase App Check, Supabase Auth rate limits plus edge rate limiting. Cap payload size and pagination size everywhere.

## Secrets management

- `.env.example` committed with placeholders; `.env*`, `*.pem`, `*.key`, service account JSON in `.gitignore`.
- Before committing, inspect staged changes for secrets (a dedicated scanner such as `gitleaks` beats `grep`).
- **If a secret reaches a remote, rotate it first**, then purge history. Deleting the line is not enough.

## AI / LLM features

If the app calls an LLM, treat the model as an untrusted component (OWASP LLM Top 10):
- Model output is untrusted input: never pass it to `eval`, SQL, shell, `innerHTML`, or file paths without validation and encoding.
- Prompts can be hijacked by any text in context (user message, fetched page, PDF). Enforce permissions in code, not in the system prompt.
- Keep secrets and other users' data out of prompts.
- Scope tools to the minimum; require confirmation for destructive actions; validate tool arguments.
- Cap tokens, request rate, and loop depth. In RAG, partition embeddings per tenant.

## Review checklist

- **Authn/Authz**: every endpoint, Server Action, Cloud Function, and RPC checks who is calling and what they may touch; users can reach only their own resources.
- **Data rules**: Firebase Rules / Supabase RLS / IAM scoped and tested for deny cases.
- **Input**: validated at the boundary; queries parameterized; output encoded; URL fetches allowlisted.
- **Secrets**: none in code, git history, client bundles, or APK; service-role / admin keys server-only.
- **Transport and headers**: HTTPS, security headers, CORS restricted to known origins (never `*` with credentials).
- **Errors and logs**: generic errors to clients; no secrets or PII in logs.
- **Dependencies**: audit triaged, lockfile committed, scripts reviewed.
- **Abuse**: rate limits, size caps, timeouts.

## Common rationalizations

| Rationalization | Reality |
|---|---|
| "Internal tool, security doesn't matter" | Internal tools get compromised; attackers pick the weakest link. |
| "We'll add security later" | Retrofitting is far harder than building it in. |
| "The client hides that button" | Every endpoint is callable directly. Enforce on the server or in rules. |
| "The anon / Firebase config key is public, so it's fine" | Only if Rules / RLS are correct. That is where the security lives. |
| "The framework handles it" | Frameworks give tools, not guarantees. |
| "The audit passed" | Audits match known advisories; they don't catch new malicious packages. |
| "It's just LLM output, only text" | It can be SQL, a script tag, or a shell command. |

## Red flags

- User input reaching SQL, shell, HTML, or file paths unvalidated
- `allow ... if true`, tables without RLS, `0.0.0.0/0`, IAM `*:*`
- Service-role / admin / AWS keys in client code, repos, or APKs
- Roles read from user-editable data (`user_metadata`, client body)
- Endpoints or Server Actions without authorization checks
- Go servers without timeouts; cleartext traffic enabled on Android
- Secrets or PII in logs or LLM prompts
- Competing lockfiles or blanket-approved install scripts

## Verification

- [ ] Authorization checked on every protected endpoint, action, function, and RPC
- [ ] Firebase Rules / Supabase RLS / IAM policies reviewed, with deny-path tests (emulator, multi-role queries)
- [ ] No secrets in source, git history, client bundle, or APK; secret values were not echoed in output
- [ ] Input validated at boundaries; queries parameterized
- [ ] Security headers and CORS verified in the real response
- [ ] Rate limits on auth endpoints; server timeouts and size caps set
- [ ] Dependency audit has no unmitigated reachable critical/high findings
- [ ] LLM output validated before use (if applicable)
