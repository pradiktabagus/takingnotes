---
name: performance-optimization
description: Optimizes performance with measurement first. Use when something is slow ("lambat", "lag", "jank"), Core Web Vitals or load time need work, Next.js/React pages are heavy, Firestore or Postgres/Supabase queries are slow (N+1, missing indexes, too many reads), Go services need profiling, AWS Lambda has latency, or an Android/Compose app has slow startup, jank, or ANRs.
---

# Performance Optimization

Measure before optimizing. Profile, find the real bottleneck, fix one thing, measure again. Optimize only what measurements prove matters. Do not optimize without evidence of a problem.

Stack covered: Next.js / React / TypeScript, Firestore, Supabase (Postgres), AWS, Go, Kotlin / Android. Load only the sections that match the problem.

## Workflow

```
1. MEASURE   baseline with real data, fixed conditions
2. IDENTIFY  the actual bottleneck, not an assumed one
3. FIX       one change at a time
4. VERIFY    re-measure the same way; keep or revert
5. GUARD     add a budget, test, or monitor so it stays fixed
```

### Step 1: Measure

Use a synthetic measurement (reproducible, good for CI) and a real-user one (validates that users actually benefit).

| Area | Synthetic | Real data |
|---|---|---|
| Web | Lighthouse, DevTools Performance, React Profiler | `web-vitals`, Vercel/Firebase Performance, CrUX |
| Postgres | `EXPLAIN (ANALYZE, BUFFERS)`, `pg_stat_statements` | Supabase Query Performance report |
| Firestore | Emulator + read counters | Firebase usage dashboard (reads per screen) |
| Go | `go test -bench -benchmem`, `pprof` (CPU, heap, goroutine), `go tool trace` | Prometheus/OpenTelemetry, CloudWatch |
| AWS Lambda | CloudWatch duration/init duration, X-Ray | Lambda Insights |
| Android | Android Studio Profiler, Perfetto, Macrobenchmark | Android vitals (Play Console), Firebase Performance |

### Where to look first

```
First load slow        -> bundle size, TTFB, render-blocking, image weight
Interaction sluggish   -> long tasks (>50ms), re-renders, layout thrash
Navigation slow        -> request waterfalls, client fetches, data size
API slow               -> queries/indexes, N+1, payload size, cold start
Intermittent slowness  -> lock contention, GC, pool exhaustion, external deps
```

### Core Web Vitals targets

LCP <= 2.5s, INP <= 200ms, CLS <= 0.1 (measured at the 75th percentile of real users).

## Next.js / React

- **Server Components by default.** Add `'use client'` only on the leaf that needs state or events; keep client boundaries small so less JS ships.
- **Waterfalls**: fetch independent data in parallel (`Promise.all`), start fetches early, stream slow parts with `<Suspense>`. Do not fetch in a client `useEffect` what the server can fetch.
- **Images**: `next/image` with real `width`/`height` (or `fill` plus `sizes`), `priority` only on the LCP image, lazy for the rest. Prefer AVIF/WebP. Fonts via `next/font` to avoid layout shift.
- **Bundle**: analyze with `@next/bundle-analyzer`. Lazy-load heavy, rarely used parts (`next/dynamic`, `React.lazy`). Avoid pulling whole libraries for one function. Load third-party scripts with `next/script` and a strategy.
- **Caching**: caching defaults differ between Next.js versions (for example `fetch` caching changed in 15). Check the installed version's docs, then set `revalidate`/tags deliberately instead of assuming.
- **Re-renders**: fix structure first (colocate state, split components, stable props). Use `React.memo`, `useMemo`, `useCallback` only where the Profiler shows a cost; check whether the React Compiler is enabled before adding manual memoization. Virtualize long lists. Use `useTransition` / `useDeferredValue` for heavy updates hurting INP.
- **CLS**: reserve space for images, embeds, and late content.

## Firestore

- **You pay per document read.** Count reads per screen as the primary metric.
- Always bound queries (`limit`) and paginate with cursors (`startAfter`), never fetch a whole collection to filter client-side.
- Put filters in the query and create the composite index it asks for; do not filter after fetching.
- Realtime listeners: attach only what the screen needs, scope them with queries, and **unsubscribe on unmount/dispose**. A forgotten listener is a read bill and a memory leak.
- Avoid N+1 (a read per list item): denormalize the few fields lists need, or batch with `in` (limited to 30 values) / `getAll`.
- Use aggregation queries (`count()`, `sum()`) instead of reading documents to count.
- Avoid hot documents (counters, global state updated constantly; sustained writes to one document are limited to about 1 per second); shard counters or aggregate server-side.
- Cache on the client (SDK persistence) for screens that reread the same data.

## Supabase / Postgres

- Diagnose with `EXPLAIN (ANALYZE, BUFFERS)`. Look for sequential scans on big tables and row estimates far from actual.
- **Index** foreign keys, columns used in `WHERE`/`JOIN`/`ORDER BY`, and **columns used in RLS policies**.
- **RLS cost**: wrap calls as `(select auth.uid())` so the value is computed once, not per row; avoid subqueries in policies that scan big tables; add the filter in the client query too, so the planner can use indexes.
- Select only needed columns (not `select('*')`). Use embedded selects (`select('*, owner(*)')`) instead of one request per row (N+1).
- Paginate: `.range()` is offset-based and slows down deep into large tables; use keyset pagination (`where created_at < $cursor order by ... limit n`).
- Serverless and edge functions: use the pooled connection string (Supavisor, transaction mode) so you do not exhaust connections.
- Heavy aggregation: materialized views, or `rpc` functions with proper indexes. Check `pg_stat_statements` for the slowest and most frequent queries.

## AWS

- **Lambda**: create SDK/DB clients outside the handler for reuse; tune memory (more memory means more CPU, often cheaper overall; try power tuning); keep packages small; investigate init duration before reaching for provisioned concurrency.
- **Databases**: use RDS Proxy (or pooled connections) with Lambda; check CloudWatch/Performance Insights for slow queries; DynamoDB: design keys to avoid hot partitions, use `Query` not `Scan`.
- **Delivery**: CloudFront in front of static assets and cacheable API responses with correct `Cache-Control`; S3 transfer acceleration only with measured benefit.
- **Tracing**: X-Ray or OpenTelemetry to find the slow hop; do not guess across service boundaries.

## Go

- Benchmark with `go test -bench=. -benchmem -count=10` and compare runs with `benchstat`. This handles the noise requirement in Step 4.
- Profile with `net/http/pprof` or `runtime/pprof`: `go tool pprof` for CPU/heap/allocs; look at goroutine profiles for leaks and block/mutex profiles for contention.
- Reduce allocations on hot paths: preallocate slices/maps (`make([]T, 0, n)`), avoid unnecessary string/byte conversions, reuse buffers (`sync.Pool` only with evidence).
- `database/sql`: set `SetMaxOpenConns`, `SetMaxIdleConns`, `SetConnMaxLifetime`; avoid N+1 (batch queries, joins, or `sqlc`); always pass `context` with timeouts.
- Reuse one `http.Client` (it pools connections); set timeouts. Bound goroutines (worker pools, `errgroup.SetLimit`); make sure each has a way to exit.
- GC tuning (`GOGC`, `GOMEMLIMIT`) is a last resort after reducing allocations.
- Run `-race` in a separate pass; it distorts timings.

## Kotlin / Android (Compose)

- **Never block the main thread.** Move I/O and heavy work to coroutines (`Dispatchers.IO` / `Default`); enable `StrictMode` in debug to catch violations. Main-thread stalls surface as jank and ANRs.
- **Compose recomposition**: pass stable/immutable types (Compose compiler reports show unstable params), `remember` expensive values, use `derivedStateOf` for derived state, provide `key` in `LazyColumn`/`LazyRow` items, read state as late as possible (lambda modifiers). Do not do work in the composition body. Check recomposition counts with Layout Inspector before changing code.
- **Startup**: use **Baseline Profiles**, the App Startup library, and lazy initialization of SDKs; measure with Macrobenchmark (cold start, scroll jank).
- **Lists and images**: Paging 3 for big lists, Coil/Glide sized to the view (no full-size bitmaps), avoid nested scrolling and deep layout trees.
- **Database**: Room queries off the main thread, indexes on filtered columns, return `Flow` instead of polling.
- **Memory**: LeakCanary in debug; watch Context/Activity leaks, unreleased listeners, large bitmaps.
- **Build size**: enable R8 and resource shrinking in release; use App Bundles.
- Profile on a mid-range physical device in a release (or benchmark) build, not an emulator in debug.

## Common anti-patterns (any stack)

- **N+1**: one query/read per item in a list. Fix with joins, embedded selects, batching, or denormalized list fields.
- **Unbounded fetching**: every list endpoint and query needs a limit and pagination.
- **Waterfalls**: sequential awaits on independent work.
- **Missing caching or cache headers** on static assets (hashed filenames can be `immutable`) and rarely changing data.
- **Unreleased resources**: listeners, subscriptions, goroutines, connections, bitmaps.

## Step 4: Verify (keep or revert)

A fix is a hypothesis until you re-measure.

- Re-measure **the same way** as the baseline: same command, same conditions, same data, warm vs cold cache matched.
- **Change one thing at a time**, or you cannot attribute the result.
- **Beat the noise**: repeat runs and compare against variance (for example `benchstat`, multiple Lighthouse runs). A 3% gain inside 5% noise is not a gain.

| Result vs baseline | Action |
|---|---|
| Meaningful improvement, tests green | **Keep.** Put the before/after numbers in the commit/PR text. |
| Within noise, or worse | **Revert.** Neutral changes are a revert; you maintain them forever for nothing. |
| Improved but a test failed | **Revert.** Correctness gates the metric. |

An optimization that wins by dropping needed work (skipping validation, caching something that must be fresh) is a regression.

**Log attempts, including reverted ones**, in the PR description or a `PERF.md`: idea, baseline to result, verdict, why. This stops dead ideas from being retried.

## Budgets (adapt to the product)

Pick numbers from real user data and enforce them: initial JS size, image weight above the fold, API p95 latency, Lambda cold start, Android cold start time and frame-jank rate. Enforce in CI where possible (Lighthouse CI, `size-limit` or bundle-analyzer thresholds, Macrobenchmark, `go test -bench` regression checks). Treat numbers in docs or examples as starting points, not universal rules.

## Common rationalizations

| Rationalization | Reality |
|---|---|
| "We'll optimize later" | Fix obvious anti-patterns (N+1, unbounded queries) now; defer micro-optimizations. |
| "It's fast on my machine" | Profile on representative devices, networks, and data volume. |
| "This optimization is obvious" | If you did not measure, you do not know. |
| "It didn't help much but doesn't hurt" | Neutral is a revert. |
| "We already wrote it, may as well keep it" | Sunk cost; the measurement does not care. |

## Red flags

- Optimization with no profile or measurement behind it
- N+1 patterns, list queries without limits, `select *` on wide tables
- Firestore listeners never removed; missing indexes ignored
- Blocking work on the Android main thread; recomposition-heavy screens with no profiling
- Go services without timeouts, pool limits, or a way to profile in production
- `React.memo`/`useMemo` sprinkled everywhere
- Several optimizations bundled into one measurement
- A "win" that required changing, skipping, or deleting a test

## Verification

- [ ] Before and after numbers exist, measured the same way
- [ ] The improvement exceeds run-to-run variance
- [ ] The specific bottleneck was identified (profile, `EXPLAIN`, trace, read count)
- [ ] Changes that did not beat the baseline were reverted; attempts are logged
- [ ] No new N+1 or unbounded fetch; indexes exist for new query and RLS patterns
- [ ] Existing tests still pass
- [ ] A budget, benchmark, or monitor guards the result
