---
name: defensive-coding
description: Load this skill when the user asks the agent to "code defensively," review code for reliability, or work on production-bound TypeScript/React or Go code. It encodes industry best practices for trust boundaries, error handling, concurrency, and system-level reliability as of 2026.
when_to_invoke: User says "code defensively", "make this more reliable", "review for robustness", "audit error handling", "write production-grade", or works on any code path that crosses a trust boundary (network, disk, user input, IPC).
stance: Pragmatic-thorough. Validate aggressively at trust boundaries, prefer total functions and explicit errors internally, do NOT redundantly re-validate trusted internal data. Every rule has a "why" so the agent can apply judgment.
---

# Defensive Coding Skill (TypeScript/React + Go, 2026)

## TL;DR for the agent
- **Validate at boundaries, trust internally.** Parse external data once into a typed value at the edge (Zod v4 for TS; explicit validation funcs for Go); inside the trust boundary, lean on the type system and avoid paranoid re-checks.
- **Make illegal states unrepresentable.** Prefer discriminated unions, branded types, total functions, and zero-value-correct structs over runtime guards.
- **Every I/O has a timeout, every retry has jitter, every goroutine has an exit, every error is handled or explicitly ignored with a reason.** No silent failures.

---

## 1. General Defensive Programming Principles

### 1.1 Fail-fast vs fail-safe
- **Fail fast** at startup and at trust boundaries: invalid config, missing secrets, unparseable input → crash or 4xx immediately. The earlier the failure, the cheaper the diagnosis.
- **Fail safe** at the *edge of the user experience*: a single product card that fails to render should not crash the whole page; a background sync that fails should degrade, not take down the foreground.
- Rule of thumb: **fail fast on programmer/configuration errors; fail safe on expected operational errors.** Google's Go style guide codifies this: "Do not use `panic` for normal error handling. Instead, use `error` and multiple return values," but panics are acceptable as "invariant checks" for "bugs that should always be caught during code review and/or testing." (google.github.io/styleguide/go/decisions#dont-panic and best-practices#when-to-panic)

### 1.2 Trust boundaries
A **trust boundary** is any point where data crosses from a less-trusted zone to a more-trusted one: network → server, server → DB driver, browser → renderer, IPC, file → parser, env → config. **Validate at every trust boundary, exactly once.** Past the boundary, code should rely on types, not re-validate.

### 1.3 Input validation
- **Allow-list, not block-list.** OWASP A05:2025 Injection: "Use positive server-side input validation."
- Parse, don't validate (Alexis King): turn `string` into `Email` once; downstream functions take `Email`, not `string`.
- Validate **structure, type, range, and semantics** in that order.

### 1.4 Invariants and assertions
Assert what *must* be true after construction or at function entry for non-public boundaries. In Go, panic on "impossible" violations within a package; never let those panics cross a package boundary. Google's style guide is explicit: "panics are never allowed to escape across package boundaries and do not form part of the package's API" (best-practices#when-to-panic).

### 1.5 Principle of least privilege in code
- Functions take the smallest type they need (don't pass the whole `User` to a function that needs `UserId`).
- DB users get the minimum grants. OWASP: "Grant database users for applications only the minimum necessary privileges."
- Goroutines/threads get scoped contexts with deadlines, not the root context.

### 1.6 Immutability and shared mutable state
- TS: prefer `readonly`, `as const`, immutable updates. Don't mutate props or state.
- Go: prefer value types where copy is cheap; document when a struct is concurrency-safe and when ownership transfers. Uber Go Style Guide: "Copy incoming slices/maps if you store them. Copy outgoing ones if they expose internal state."

### 1.7 Total vs partial functions
A total function returns a sensible value for every input in its declared type. Replace partial functions (`head([])` throws) with total ones (`head([]) → undefined | T`, or take `NonEmptyArray<T>`).

### 1.8 Make illegal states unrepresentable
Replace `{loading: boolean, data: T | null, error: Error | null}` (which has 8 states, most invalid) with a discriminated union — see §2.3.

### 1.9 Defensive copying
Copy at API boundaries when:
- A caller passes a mutable container you'll retain (slice/map/array).
- You expose internal state (return a copy).
Skip it for short-lived, single-owner data — it's not free.

### 1.10 Postel's Law — modernized
The original "Be conservative in what you do, be liberal in what you accept" (RFC 760, Jon Postel) is **substantially discredited for new systems**. Martin Thomson's IETF Internet-Draft *The Harmful Consequences of the Robustness Principle* (draft-thomson-postel-was-wrong, 2018) and the related IAB RFC 9413 *Maintaining Robust Protocols* (David Schinazi, 2023) argue that liberal acceptance "actually leads to a lack of robustness, including security": flaws become entrenched as de facto standards. Florentin Rochet and Olivier Pereira's PoPETs paper *Dropping on the Edge: Flexibility and Traffic Confirmation in Onion Routing Protocols* (PoPETs 2018(2): 27–46, DOI 10.1515/popets-2018-0011) showed it could be exploited to discover Tor onion-service guard nodes in one day without injecting a relay. **The 2026 default is "virtuous intolerance"**: emit strictly, accept strictly, fail loudly on malformed input. Be liberal only at legacy interop seams, and document them.

### 1.11 The pragmatic middle ground
- **At trust boundaries:** belt-and-braces. Validate types, ranges, semantics, sizes; bound resource usage.
- **Internally:** trust the types. Don't `if (!user) return` after a function whose signature is `(user: User)` — that's noise and it lies about the API.
- **Across module boundaries within the same trust zone:** validate only what the type system can't (e.g., "list is non-empty," "string is a valid UUID") and lift those into the type.

---

## 2. TypeScript / React

### 2.1 tsconfig.json — the strict baseline (2026)
```jsonc
{
  "compilerOptions": {
    "strict": true,                          // 8 flags in one
    "noUncheckedIndexedAccess": true,        // arr[i] is T | undefined
    "exactOptionalPropertyTypes": true,      // {x?: T} ≠ {x: T | undefined}
    "noImplicitOverride": true,
    "noFallthroughCasesInSwitch": true,
    "noPropertyAccessFromIndexSignature": true,
    "useUnknownInCatchVariables": true,      // implied by strict, keep explicit
    "moduleResolution": "bundler",           // for Next/Vite
    "isolatedModules": true
  }
}
```
**Why `noUncheckedIndexedAccess`?** Matt Pocock's Total TypeScript tip *Make accessing objects safer by enabling "noUncheckedIndexedAccess" in tsconfig* puts it bluntly: "The 'noUncheckedIndexedAccess' is the most awesome config option you've never heard of." It's his first stop beyond `strict: true`. Without it, `users[10]` has type `string`, which is a lie — empty arrays exist.

**Never** disable individual strict flags to suppress errors — fix the errors. `any` is a contagious lie; reach for `unknown` and narrow.

### 2.2 Runtime validation at boundaries — Zod 4 (2026 default)
Zod v4 (stable, released 2025 under the `zod@^3.25.0` tag) is the 2026 default for parsing untrusted input. Per the official release notes at zod.dev/v4 and InfoQ's coverage: ~14× faster string parsing, 7× faster array parsing, 6.5× faster object parsing vs Zod 3, with a `@zod/mini` distribution at ~1.9 KB gzipped for tree-shakable edge/frontend use. Alternatives: **Valibot** (smaller, function-based), **ArkType** (compile-time TS-string syntax). Use one consistently per project.

```ts
import { z } from "zod";

const UserSchema = z.object({
  id: z.uuid(),
  email: z.email(),
  age: z.int().min(13).max(120),
  role: z.enum(["admin", "member"]),
});
type User = z.infer<typeof UserSchema>;

// At the boundary — parse, don't validate.
export async function fetchUser(id: string): Promise<User> {
  const res = await fetch(`/api/users/${id}`);
  if (!res.ok) throw new ApiError(res.status);
  return UserSchema.parse(await res.json()); // throws on bad shape
}
```
**Bad:** `const user = (await res.json()) as User;` — a lie. **Good:** parse with the schema; the *type* is the *runtime contract*.

### 2.3 Discriminated unions for state
```ts
// BAD — 8 states, 6 of them invalid
type Bad = { loading: boolean; data: User | null; error: Error | null };

// GOOD — 4 states, all valid, TS narrows automatically
type RemoteData<T, E = Error> =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: T }
  | { status: "error"; error: E };

function render(s: RemoteData<User>) {
  switch (s.status) {
    case "idle":    return <Empty />;
    case "loading": return <Spinner />;
    case "error":   return <ErrorView error={s.error} />;
    case "success": return <UserCard user={s.data} />;
    default: {
      const _exhaustive: never = s;       // compile error if a case is added
      return _exhaustive;
    }
  }
}
```

### 2.4 Branded types for IDs and validated primitives
```ts
type Brand<K, T> = K & { readonly __brand: T };
type UserId = Brand<string, "UserId">;
type PostId = Brand<string, "PostId">;
type Email  = Brand<string, "Email">;

function makeEmail(s: string): Email {
  if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(s)) throw new Error("bad email");
  return s as Email;
}
function getUser(id: UserId) { /* ... */ }
getUser("u_1" as PostId); // ❌ compile error — won't mix IDs
```

### 2.5 `unknown` over `any`; type guards
```ts
function isUser(x: unknown): x is User {
  return UserSchema.safeParse(x).success;
}
try { /* ... */ } catch (e: unknown) {
  if (e instanceof MyError) { /* handle */ }
  else { logger.error({ err: e }); throw e; }
}
```

### 2.6 Result vs throw
Throw at the *bottom of the call stack* (when error is exceptional) and at the *very top* (uncaught → error boundary). Return `Result` at module/API seams where errors are part of the domain (e.g., `parseDate`, `chargePayment`). Don't mix — pick one per layer.
```ts
type Result<T, E = Error> = { ok: true; value: T } | { ok: false; error: E };
```

### 2.7 React — error boundaries
Wrap **per route** and **per major widget** so one widget's crash doesn't blank the page. Kent C. Dodds, *Use react-error-boundary to handle errors in React* (kentcdodds.com/blog/use-react-error-boundary-to-handle-errors-in-react): "Use `react-error-boundary` ... you can almost think about the `ErrorBoundary` component the same way you do a `try/catch` block. You can wrap it around a bunch of React components to handle lots of errors, or you can scope it down to a specific part of the tree."

Native error boundaries do NOT catch errors in "Event handlers ... Asynchronous code (e.g. setTimeout or requestAnimationFrame callbacks) ... Server side rendering ... Errors thrown in the error boundary itself" — use the `useErrorBoundary` hook (`showBoundary(error)`) to forward those into the same declarative fallback.

### 2.8 React — race conditions in `useEffect`
**Always** assume async responses arrive out of order. Two patterns; prefer AbortController on modern targets:
```tsx
// Pattern A — AbortController (preferred)
useEffect(() => {
  const ac = new AbortController();
  (async () => {
    try {
      const res = await fetch(`/api/user/${id}`, { signal: ac.signal });
      setUser(UserSchema.parse(await res.json()));
    } catch (e) {
      if ((e as Error).name !== "AbortError") setError(e as Error);
    }
  })();
  return () => ac.abort();
}, [id]);

// Pattern B — ignore flag (when no AbortController, e.g., non-fetch promises)
useEffect(() => {
  let ignore = false;
  (async () => {
    const data = await loadData(id);
    if (!ignore) setData(data);
  })();
  return () => { ignore = true; };
}, [id]);
```
For nontrivial data fetching, delegate to TanStack Query / SWR — they solve dedup, caching, cancellation, and stale data uniformly.

### 2.9 React — other hook discipline
- **Stale closures:** use functional setters (`setX(prev => …)`) inside intervals/effects; keep dependency arrays honest.
- **Keys:** never use array index for lists that can reorder; use a stable id.
- **Controlled inputs:** if `value` is set, `onChange` must update it; never flip controlled↔uncontrolled.
- **Effect dependencies:** never lie. Use the eslint-plugin-react-hooks exhaustive-deps rule.
- **Subscriptions & timers:** every `addEventListener`/`setInterval`/socket open returns a cleanup; if you don't return one, you've leaked.
- **Hydration mismatches (SSR):** never branch on `typeof window`, `Date.now()`, or `Math.random()` during render. Move client-only UI behind `useEffect` or `<ClientOnly>`.

### 2.10 Defensive rendering
- Optional chaining + nullish coalescing: `user?.profile?.name ?? "Anonymous"`.
- Treat *every* API response as `unknown` until parsed.
- Empty states are first-class UI: show `<Empty />`, not a blank grid.

### 2.11 Form validation
- Schema-driven (Zod) on both client and server. Client for UX, server for security.
- Validate per-field on blur; submit re-validates the whole schema.
- Distinguish "untouched / dirty / submitted" so you don't yell at users mid-typing.

---

## 3. Go

### 3.1 Error handling — idioms
- **Wrap with `%w` only when callers should programmatically inspect the chain** (i.e., it's part of your API contract). Google: "The `%w` verb is specifically designed for error wrapping. It creates a new error that provides an `Unwrap()` method, allowing callers to programmatically inspect the error chain using `errors.Is` and `errors.As`." Use `%v` at system boundaries to sever the chain. (google.github.io/styleguide/go/best-practices#error-handling)
- Sentinel errors (`var ErrNotFound = errors.New("...")`) only for stable, well-known conditions; error types when callers need structured fields; otherwise opaque.
- Dave Cheney's rule: **handle errors once**. Log *or* return — not both.

```go
// Good
if err != nil {
    return fmt.Errorf("loading user %s: %w", id, err)
}

// Bad — both logs and wraps; double-reports
if err != nil {
    log.Printf("load failed: %v", err)
    return err
}
```

### 3.2 Panic vs error
Per Google Go style: "Do not use `panic` for normal error handling. Instead, use `error` and multiple return values." Panic for true programmer errors / unreachable code only, and "panics are never allowed to escape across package boundaries." Libraries: "Libraries should prefer returning an error to the caller rather than aborting the program, especially for transient errors." Also: "resist the temptation to recover panics to avoid crashes, as doing so can result in propagating a corrupted state. The further you are from the panic, the less you know about the state of the program, which could be holding locks or other resources." (google.github.io/styleguide/go/best-practices)

### 3.3 `recover` only at goroutine boundaries
Spawning a goroutine in a server? Wrap it so a panic doesn't kill the process:
```go
func safeGo(ctx context.Context, fn func(context.Context) error) {
    go func() {
        defer func() {
            if r := recover(); r != nil {
                slog.ErrorContext(ctx, "goroutine panic",
                    "panic", r, "stack", string(debug.Stack()))
            }
        }()
        if err := fn(ctx); err != nil {
            slog.ErrorContext(ctx, "goroutine error", "err", err)
        }
    }()
}
```

### 3.4 Context propagation
Google Go style is absolute: "When passed to a function or method, `context.Context` is always the first parameter. Do not add a context member to a struct type. Instead, add a context parameter to each method on the type that needs to pass it along. Do not create custom context types or use interfaces other than `context.Context` in function signatures. There are no exceptions to this rule." (google.github.io/styleguide/go/decisions#contexts)

Every I/O takes a `ctx`. Every long loop checks `ctx.Done()`. Every server entry derives a context with timeout.

### 3.5 Goroutine lifecycles — no leaks
Google: "When you spawn goroutines, make it clear when or whether they exit ... Goroutines can leak by blocking on channel sends or receives. The garbage collector will not terminate a goroutine blocked on a channel even if no other goroutine has a reference to the channel." (decisions#goroutine-lifetimes)

Use `errgroup` for fan-out with bounded errors and shared cancellation:
```go
g, ctx := errgroup.WithContext(ctx)
g.SetLimit(8) // concurrency cap — bulkhead
for _, item := range items {
    item := item
    g.Go(func() error { return process(ctx, item) })
}
if err := g.Wait(); err != nil { return err }
```
Test for leaks: `go.uber.org/goleak` in `TestMain`.

### 3.6 Channels — ownership, closing, nil
- **The sender closes; the receiver never does.** Closing a channel you don't own is a runtime bug.
- Closing a channel signals "no more values," not "stop." Use `context` for cancellation.
- A `nil` channel blocks forever — useful for disabling a `select` case.

### 3.7 Mutex vs channel
- Mutex for protecting state in one struct (e.g., a cache).
- Channel for communicating ownership and signaling between goroutines.
- "Share memory by communicating" is a guideline, not dogma — a `sync.RWMutex` around a map is often clearer than a channel-based actor.

### 3.8 Race detector in CI
Run `go test -race ./...` in CI on every PR. Per the official Go documentation (go.dev/doc/articles/race_detector): "memory usage may increase by 5-10x and execution time by 2-20x." Budget accordingly — but the cost of a data race in production is higher. Catches data races before they corrupt production.

### 3.9 `defer` and the err-shadowing trap
```go
// Bad — closeErr is silently dropped; err in outer scope unchanged
defer f.Close()

// Good — surface Close errors when they matter (writes!)
defer func() {
    if cerr := f.Close(); cerr != nil && err == nil {
        err = cerr
    }
}()
```
Always use *named* returns when you want a deferred function to mutate the returned error.

### 3.10 Constructors and zero values
- Prefer **zero-value-useful** types (`sync.Mutex{}`, `bytes.Buffer{}`).
- When invariants require setup, use `NewX` returning `(*X, error)`. For optional config, use functional options.
```go
type Server struct { /* … */ }
type Option func(*Server)

func WithTimeout(d time.Duration) Option { return func(s *Server) { s.timeout = d } }

func NewServer(addr string, opts ...Option) (*Server, error) {
    s := &Server{addr: addr, timeout: 5 * time.Second}
    for _, o := range opts { o(s) }
    if s.addr == "" { return nil, errors.New("addr required") }
    return s, nil
}
```

### 3.11 JSON unmarshaling — pitfalls
- Tag every serialized field explicitly. Uber Go Style Guide rationale: "The serialized form of the structure is a contract between different systems ... [tags guard] against accidentally breaking the contract by refactoring or renaming fields."
- `json.Decoder.DisallowUnknownFields()` for strict ingress; reject unexpected fields rather than silently swallow them.
- Pointers (`*int`) when "missing" must differ from "zero."
- Validate semantically after unmarshaling: ranges, enums, non-empty.

### 3.12 Timeouts — no naked HTTP clients
```go
// Bad — http.DefaultClient has no timeout; can hang forever
resp, err := http.Get(url)

// Good — explicit timeouts at every layer
client := &http.Client{Timeout: 10 * time.Second}
req, _ := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
resp, err := client.Do(req)
```
Server side: set `ReadTimeout`, `WriteTimeout`, `IdleTimeout`, and `ReadHeaderTimeout` on every `http.Server`.

### 3.13 Structured logging — `slog` (Go 1.21+, 2026 default)
```go
logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level: slog.LevelInfo,
    AddSource: true,
}))
slog.SetDefault(logger)

slog.InfoContext(ctx, "order placed",
    "order_id", id, "user_id", userID, "amount_cents", amt)
```
Implement `slog.LogValuer` on types holding secrets so they're auto-redacted. Never log PII or tokens. Always include a correlation/trace ID.

### 3.14 Iterators (Go 1.23+)
The `iter` package and range-over-func let you write composable, lazy iteration. Per the Go blog *Range Over Function Types*: iterator methods on a collection are conventionally named `All`. Prefer `iter.Seq[T]` over returning materialized slices when sequences may be large or infinite — saves memory and enables short-circuit termination.

---

## 4. System-level Reliability

### 4.1 Timeouts everywhere
**Every** network call, DB query, cache call, file read on a network mount, and IPC must have an explicit deadline. AWS Builders' Library, Marc Brooker, *Timeouts, retries and backoff with jitter*: "Set timeout on any remote call, and any call across processes on the same server. This includes both the connection timeout and request timeout." (aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter)

### 4.2 Retries with exponential backoff + jitter
Marc Brooker / AWS: "Our solution is jitter. Jitter adds some amount of randomness to the backoff to spread the retries around in time." Without jitter, recovering services get hit by synchronized retry storms. "Full Jitter" formula (Marc Brooker, AWS Architecture Blog, *Exponential Backoff And Jitter*, 04 Mar 2015): `sleep = random_between(0, min(cap, base * 2^attempt))`. Cap attempts; retry only on transient errors (5xx, 429, network); never retry non-idempotent operations without an idempotency key.

### 4.3 Circuit breakers — when warranted
Pattern from Michael Nygard's *Release It!*: wrap a remote call; after N failures in a window, **Open** (fail fast for a sleep window); **Half-Open** allows probes; success → **Closed**. Martin Fowler: "Once the failures reach a certain threshold, the circuit breaker trips, and all further calls to the circuit breaker return with an error, without the protected call being made at all." Use when:
- The dependency is across a network boundary.
- Failures are correlated (the dependency is overloaded, not your call).
- Waiting on timeouts would exhaust your thread/connection pool.

Skip for in-process calls, local resources, and small services where a well-tuned timeout suffices. A circuit breaker without a sensible fallback just changes the error message — bad fallback can be worse than a timeout.

### 4.4 Bulkheads and resource isolation
Nygard's bulkhead pattern: partition resources so a noisy neighbor can't drain everyone's pool. Concrete: separate thread/goroutine pools per downstream, separate DB connection pools per workload class, per-tenant rate limits, separate Kubernetes deployments for critical vs batch.

### 4.5 Idempotency
Any operation that may be retried must be idempotent or carry an idempotency key. POST endpoints that create resources accept `Idempotency-Key` headers and dedupe in a short-lived store. Database writes use `INSERT ... ON CONFLICT` / unique constraints. Webhooks deduplicate on event ID.

### 4.6 Graceful degradation and fallbacks
- Cache-on-failure: if the recommendation service is down, return the last good response from cache and mark it stale.
- Feature-flag the non-essential: ratings, related products, personalization should degrade to defaults without taking down checkout.

### 4.7 Structured logging — three rules
1. **Structured** (JSON/key-value), never `fmt.Sprintf`'d into a single string.
2. **Correlation ID** on every log line in a request — propagate via header (`traceparent` for W3C trace context) and in context.
3. **No secrets, no PII** unredacted. Use a `LogValuer` (Go) or a serializer hook (TS pino/winston).

Levels: `DEBUG` (off in prod), `INFO` (state transitions, request summaries), `WARN` (recoverable anomalies), `ERROR` (action needed). If everything is ERROR, nothing is.

### 4.8 Observability — three pillars + correlation
- **Metrics** (Prometheus / OpenTelemetry): RED for services (Rate, Errors, Duration), USE for resources (Utilization, Saturation, Errors).
- **Traces** (OpenTelemetry): one span per logical operation; attributes for IDs, sizes, statuses.
- **Logs** (structured, correlated): tied to trace via `trace_id` and `span_id`.
The point is *correlation*: from a metric alert → traces of failing requests → logs by `trace_id`.

### 4.9 Health checks vs readiness
- `/livez`: am I alive? (process up, event loop responsive). Failing → restart.
- `/readyz`: can I serve traffic? (deps reachable, warm caches, migrations applied). Failing → drain from LB.
- Never make `/livez` depend on the database — that turns a DB blip into a restart loop.

### 4.10 Graceful shutdown
On SIGTERM:
1. Mark `/readyz` unhealthy. Sleep ~5s for the LB to drain (Kubernetes propagation lag).
2. Stop accepting new connections (`server.Shutdown(ctx)`).
3. Wait for in-flight requests with a deadline less than `terminationGracePeriodSeconds`.
4. Drain workers, close DB pools last.
Go: use `signal.NotifyContext(ctx, syscall.SIGINT, syscall.SIGTERM)` (Go 1.16+).

### 4.11 Rate limiting and backpressure
- Token bucket at the edge for per-client limits.
- Concurrency limits (`errgroup.SetLimit`, semaphores, bounded worker pools) for fan-out.
- Backpressure: bounded channels/queues that *block* or *shed* rather than buffering unbounded.

### 4.12 Input sanitization at API boundaries (OWASP)
- **SQLi:** parameterized queries always. Never string-concatenate SQL. ORMs are safe *only* when used as designed.
- **XSS:** context-aware output encoding; React escapes by default — `dangerouslySetInnerHTML` is dangerous; sanitize with DOMPurify if you must.
- **Command injection:** never pass user input to a shell; use `exec.CommandContext(ctx, prog, args...)` (Go) or `child_process.execFile` (Node), never `exec` with a shell string.
- **SSRF:** allow-list outbound destinations; block link-local/metadata IPs (169.254.169.254).
- **Prompt injection (OWASP LLM Top 10 2025, LLM01):** treat LLM output as untrusted user input; never execute it without re-validation.

### 4.13 Secrets management
- **Never** in code, in env files committed to git, or in logs.
- Read from a secret manager (Vault, AWS Secrets Manager, GCP Secret Manager) or, at minimum, environment variables populated at deploy time.
- Rotate. Scope by service. Audit access.

### 4.14 Config validation at startup
Parse the entire config with a schema (Zod / `envconfig` / `viper` + custom validator) **before** starting servers. Missing required values → crash with a clear message. The 12-factor app's strict env separation makes this trivial.

### 4.15 Feature flags for safe rollouts
- Decouple deploy from release.
- Use percentage rollouts and per-user/per-tenant targeting.
- Every flag has an owner and an expiration date — dead flags rot.

### 4.16 Database transactions and migrations
- Pick the lowest isolation that meets your invariants; default to `READ COMMITTED` and escalate explicitly. Document which operations need `SERIALIZABLE`.
- Lock ordering: always acquire locks in a consistent order across the codebase to avoid deadlocks.
- **Expand/contract (parallel change) for schema changes**, popularized by Pete Hodgson and PlanetScale: "Expand: make non-destructive and backward-compatible changes (add new columns, indexes, constraints in 'NOT VALID' mode). Backfill: fill historical data in the background, in small batches, without blocking. Contract: once everything is ready, apply destructive changes." Every step is rollback-safe; old and new app versions coexist.

### 4.17 Concurrency limits and memory
- Cap concurrency by *resource*, not by feel: connection pool size, file descriptor limit, downstream rate budget.
- Set explicit memory limits on the runtime (`GOMEMLIMIT` for Go 1.19+; `--max-old-space-size` for Node) and on the container. OOM is a *decision*; let it happen at a known threshold, not at the OS's mercy.

### 4.18 No silent failures
Every error is one of: handled, returned, or **explicitly ignored with a comment**.
```go
// Bad
_ = json.Unmarshal(data, &x)

// Good — either return it, or document why ignored
if err := json.Unmarshal(data, &x); err != nil {
    return fmt.Errorf("parse config: %w", err)
}
// or
_ = file.Close() // best-effort close on hot path; data already flushed
```
TS: `void promise` is a bug 99% of the time; lint with `@typescript-eslint/no-floating-promises`.

---

## 5. Review checklist (apply to every PR)

**Trust boundaries**
- [ ] Every external input is parsed against a schema at exactly one place.
- [ ] No `as Foo` / `any` / type assertions on unvalidated data.
- [ ] No `interface{}` / `json.RawMessage` past the parsing layer.

**Errors**
- [ ] Every error is handled, returned, or has a `// ignored because …` comment.
- [ ] Wrap with `%w` only when callers may inspect; otherwise `%v` or domain error.
- [ ] No double-logging (log OR return).
- [ ] TS: catch is typed `unknown`, not `any`.

**Concurrency**
- [ ] Every goroutine has a clear, documented exit path.
- [ ] Every channel has a single, identified closer.
- [ ] `errgroup` with `SetLimit` for fan-out; goleak in tests.
- [ ] `go test -race` passes.
- [ ] React effects clean up (AbortController/ignore flag/clearInterval/unsubscribe).

**I/O**
- [ ] Every HTTP client and server has explicit timeouts.
- [ ] Every retry has capped exponential backoff + full jitter.
- [ ] Idempotency key on retried POSTs.
- [ ] Circuit breaker on cross-service hot paths where overload is plausible.

**State and types**
- [ ] Illegal states unrepresentable (discriminated unions).
- [ ] IDs and validated primitives are branded/typed.
- [ ] Total functions; partial ones returned via `Result` / `(T, error)`.
- [ ] No mutation of inputs you don't own; defensive copy at boundaries.

**Observability**
- [ ] Structured logs with correlation/trace ID.
- [ ] No secrets/PII in logs.
- [ ] Metrics: RED for endpoints, USE for resources.
- [ ] Spans on every external call.

**Security**
- [ ] Parameterized queries; no SQL string concat.
- [ ] Output encoded for context (HTML/JS/URL).
- [ ] No `exec` with a shell string; use argv form.
- [ ] Secrets from a manager, never committed.
- [ ] Least-privilege DB grants and IAM roles.

**Operability**
- [ ] Config validated at startup; fail fast on bad config.
- [ ] `/livez` and `/readyz` distinct; `/livez` has no external deps.
- [ ] SIGTERM handled with graceful drain.
- [ ] DB migrations are expand/backfill/contract; old and new code coexist.
- [ ] Feature flag on risky changes; flag has owner and expiry.

---

## 6. When NOT to apply this skill
- Throwaway scripts and prototypes — the cost of paranoia exceeds the cost of failure.
- Single-developer internal tools with no SLA.
- Performance-critical inner loops where validation cost dominates (validate at the boundary, not the loop body).

The skill exists to be applied *judiciously*. If you cannot articulate the failure mode you're defending against, you're cargo-culting — stop and think about the actual threat model first.
