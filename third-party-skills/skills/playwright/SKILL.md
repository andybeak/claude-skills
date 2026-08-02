---
name: playwright
description: >
  Rules and best practices for authoring stable, low-flake Playwright end-to-end tests
  in TypeScript with @playwright/test. Use when writing, reviewing, or debugging E2E tests,
  working with Playwright locators, fixtures, assertions, page objects, or test configuration.
  Triggers include: "write a playwright test", "e2e test", "playwright",
  "browser automation test", "end-to-end test", "fix flaky test",
  or any task involving @playwright/test, test.extend, or page.locator.
---

# Playwright Test Authoring Rules for AI Agents

> **Audience:** AI coding agents (Claude Code, Cursor, Aider, Copilot, etc.) writing **end-to-end tests in TypeScript with `@playwright/test`**.
> **Scope:** Rules for stable, low-flake, low-maintenance tests against modern web apps. Optimized for inclusion in `CLAUDE.md` / `AGENTS.md` / `.cursorrules`.
> **Version target:** Playwright ≥ 1.50. Features used include `test.step.skip`, `toMatchAriaSnapshot`, `locator.describe`, and trace mode `retain-on-failure-and-retries`.

---

## 0. Hard bans (zero tolerance)

| ❌ Never use | ✅ Use instead | Why |
|---|---|---|
| `page.waitForTimeout(ms)` | Web-first assertion (`expect(locator).toBeVisible()` etc.) or `expect.poll` / `expect.toPass` | Playwright API docs (`page.waitForTimeout`): *"Never wait for timeout in production. Tests that wait for time are inherently flaky."* |
| `await page.waitForLoadState('networkidle')` (esp. SPAs) | Assert on a user-visible element (`expect(...).toBeVisible()`) | SPAs with polling, WebSockets, or analytics never become idle — test hangs or flakes (`networkidle` waits for 0 connections for 500 ms) |
| `expect(await locator.isVisible()).toBe(true)` / `textContent()`-then-`toBe` | `await expect(locator).toBeVisible()` / `toHaveText(...)` | Manual checks return immediately, no retry. Best Practices: *"Don't use manual assertions that are not awaiting the expect."* |
| `locator.click({ force: true })` as a "fix" | Fix the real flow (close overlay, wait for enabled) | `force` skips actionability checks and hides real bugs |
| `page.$`, `page.$$`, `page.$eval`, `page.$$eval`, `ElementHandle.*` | `page.locator(...)` + `getBy*` | ElementHandles are deprecated; no auto-wait, references go stale |
| Raw CSS/XPath when an accessible locator exists (`page.locator('.btn-primary > span:nth-child(2)')`) | `getByRole` → `getByLabel` → `getByText` → `getByTestId` | DOM/CSS churn breaks tests; user-facing locators don't |
| `page.waitForNavigation()` | `page.waitForURL(...)` *or* nothing (most actions auto-wait) | Deprecated for `@playwright/test` |
| `page.waitForSelector(sel)` | `await locator.waitFor()` or just use the locator | Legacy API; `Locator` is the modern primitive |
| `test.only`, committed `test.skip`/`test.fixme` | Tag (`@skip`) and gate via `--grep-invert`; require owner + ticket | Enforced by `eslint-plugin-playwright/no-focused-test` |
| Manual retry loops (`for`/`while` until success) | `expect.toPass()` / `expect.poll()` | Built-in retries handle timing, timeouts, and reporting consistently |
| Sharing mutable state across tests (module-level `let user`, etc.) | Worker-scoped fixture keyed by `testInfo.workerIndex` | Workers run in parallel — shared mutable state = race conditions |
| Committing `storageState.json` with real session tokens | `.gitignore` `playwright/.auth/`; load via `setup` project | Secret leakage; tokens expire |
| Hard waits for animations (`waitForTimeout(500)`) | `contextOptions: { reducedMotion: 'reduce' }` + CSS `@media (prefers-reduced-motion)` | Disables animations deterministically |

---

## 1. Locators — strict priority order

Always pick the **highest** locator on this ladder that uniquely identifies the element. Stop at the first that works.

1. **`page.getByRole(role, { name })`** — interactive elements (button, link, textbox, checkbox, heading, dialog, etc.). Default for almost everything.
2. **`page.getByLabel(text)`** — form fields with a `<label>`.
3. **`page.getByPlaceholder(text)`** — inputs without labels (forms only).
4. **`page.getByText(text, { exact: true })`** — non-interactive copy / assertions on visible text.
5. **`page.getByTitle(text)`** / **`page.getByAltText(text)`** — images, icons, tooltips.
6. **`page.getByTestId('foo')`** — only when no semantic locator exists. Requires `data-testid` (configurable via `testIdAttribute`).
7. **`page.locator(css)` / XPath** — **last resort**, e.g. third-party widgets without semantics.

Rationale (verbatim, Playwright Best Practices, `playwright.dev/docs/best-practices`, "Use locators" section): *"To make tests resilient, we recommend prioritizing user-facing attributes and explicit contracts."*

### Locator rules

- ✅ Always pass `{ name }` to `getByRole` (a role alone is too broad).
- ✅ Default `getByText` to `{ exact: true }` to avoid substring strict-mode violations when content grows.
- ✅ **Chain & filter** instead of `nth()` / indices. Scope first, then act:
  ```ts
  const row = page.getByRole('row').filter({ hasText: 'Acme Corp' });
  await row.getByRole('button', { name: 'Delete' }).click();
  ```
- ✅ Use `locator.or(...)` for "either of" (v1.33+) and `locator.and(...)` for "matches both" — not `try`/`catch`.
- ✅ For lists, prefer `filter({ hasText })` / `filter({ has: locator })` over `.nth(i)`.
- ❌ Don't use `first()` / `last()` / `nth(i)` to "make ambiguity go away" — fix the ambiguity by scoping.
- ✅ Add `data-testid` only when the element has no accessible role/name *and* you control the markup.
- ✅ Use `locator.describe('Subscribe button')` (v1.53+) to label opaque locators in trace/reports.

---

## 2. Assertions — web-first only

### Rules

- ✅ **All UI assertions must be `await expect(locator).matcher(...)`** so they auto-retry until the assertion timeout (default 5 s; configure via `expect: { timeout: 10_000 }`).
- ❌ Never `expect(await locator.textContent()).toBe(...)` — one-shot read, no retry, racy.
- ❌ Never `expect(await locator.isVisible()).toBeTruthy()` — `isVisible()` returns immediately.
- ✅ Prefer positive matchers over negated ones: `toBeHidden()` > `not.toBeVisible()`.
- ✅ For "either" states, use `locator.or(...).first()` rather than `try`/`catch`.
- ✅ Use `expect.soft(...)` only when a test has multiple independent checks where a single failure shouldn't abort.
- ✅ Pass a custom message as 2nd arg for context: `expect(loc, 'cart badge updates after add').toHaveText('1')`.

### Reaching for retry APIs

Use **only** when no locator/page assertion fits:

- **`expect(page).toHaveURL(...)` / `toHaveTitle(...)`** — for navigation.
- **`await expect.poll(async () => fetchSomething(), { timeout: 15_000 }).toBe('done')`** — non-DOM values that change over time (API status, queue, file).
- **`await expect(async () => { ... }).toPass({ timeout: 30_000 })`** — group several assertions that must all pass at the same moment. **Keep inner assertion timeouts short** (`{ timeout: 1_000 }`) so the outer block can retry without burning the whole window on one inner assertion.

Default `expect` timeout is 5 s; raise globally via `expect: { timeout: 10_000 }` — don't sprinkle long per-assertion timeouts.

---

## 3. Waiting & timing — let Playwright do it

- ✅ Actions (`click`, `fill`, `check`, `selectOption`, `press`) **already auto-wait** for visible + stable + enabled + receives-events. **Don't pre-wait** with `waitFor({ state: 'visible' })` before them.
- ✅ Use `await locator.waitFor()` only for explicit DOM-state waits that no following action covers — e.g., waiting for a spinner to detach (`{ state: 'detached' }`).
- ✅ Use `page.waitForURL(...)` after explicit navigation actions you can't synchronize with a visible assertion.
- ✅ For network-driven flows, **register the wait BEFORE the trigger** (otherwise on fast networks the response arrives before the listener attaches and the promise hangs to timeout):
  ```ts
  const resp = page.waitForResponse(r =>
    r.url().includes('/api/orders') && r.request().method() === 'POST');
  await page.getByRole('button', { name: 'Place order' }).click();
  await resp; // safe now
  ```
  Or wrap in `Promise.all([page.waitForResponse(...), action()])`.
- ❌ Don't use `page.waitForLoadState('load' | 'networkidle')` as a generic "is the page ready" wait. Assert on UI.
- ✅ If you must navigate with a stricter wait, prefer `await page.goto(url, { waitUntil: 'domcontentloaded' })` + an explicit `expect(...).toBeVisible()` on the readiness indicator.

---

## 4. Helpers, fixtures, and POM

**Default helper mechanism for agents: `test.extend` fixtures. Use POM only when the same page has ≥3 reusable interactions.**

### Why fixtures > POM for AI agents
- One declaration site, one import, automatic setup/teardown.
- Composable: fixtures depend on fixtures.
- On-demand: only the fixtures used by a test are constructed.
- No `new PageObject(page)` boilerplate in every test.
- The dependency graph appears in the test signature (`async ({ checkout, seededUser }) => {…}`) — easier for an LLM to reason about than constructor-and-import mazes.

### Pattern: page-object-as-fixture

```ts
// fixtures.ts
import { test as base, expect } from '@playwright/test';
import { CheckoutPage } from './pages/CheckoutPage';

type Fixtures = { checkout: CheckoutPage };

export const test = base.extend<Fixtures>({
  checkout: async ({ page }, use) => {
    await use(new CheckoutPage(page));
  },
});
export { expect };
```

```ts
// checkout.spec.ts
import { test, expect } from './fixtures';

test('checkout works', async ({ checkout }) => {
  await checkout.goto();
  await checkout.addItem('Widget');
  await expect(checkout.total).toHaveText('$10.00');
});
```

### Fixture rules
- ✅ Locators in fixtures/POMs are `readonly` properties (`readonly submit = this.page.getByRole('button', { name: 'Submit' });`).
- ✅ Keep assertions in spec files; keep actions/locators in fixtures/POMs. Don't return new page objects from action methods.
- ✅ Use **worker-scoped fixtures** (`{ scope: 'worker' }`) for expensive resources (DB connection, seeded org, auth token) **only if immutable** during tests.
- ✅ Use **test-scoped** (default) for anything per-test (a logged-in `page`, a seeded user).
- ✅ For unique-per-worker test data, derive from `testInfo.workerIndex`: `` `user-${test.info().workerIndex}` ``.
- ❌ Don't share mutable state via module-level variables.

---

## 5. Network & server lag — mocking vs. real backend

**Decision tree:**

1. **External / third-party APIs you don't control** → **mock with `page.route` + `route.fulfill`**. Best Practices, "Avoid testing third-party dependencies": *"Only test what you control. Don't try to test links to external sites or third party servers that you do not control."*
2. **Your own backend, deterministic, fast** → hit it for real. Use web-first assertions; they retry by default.
3. **Your own backend, slow / eventually-consistent** → use `expect.toPass` or `expect.poll` around the consistency check; **don't bump global timeouts**.
4. **Loading spinner / staged render** → assert on the **end-state element**, not the spinner's absence.

### Mocking patterns

```ts
// Block & fulfill
await page.route('**/api/users', route =>
  route.fulfill({ status: 200, json: usersFixture }));

// Patch real response
await page.route('**/api/users', async route => {
  const resp = await route.fetch();
  const data = await resp.json();
  data.role = 'admin';
  await route.fulfill({ response: resp, json: data });
});

// Block analytics & ads (in beforeEach)
await context.route(/(analytics|googletagmanager|hotjar)/, r => r.abort());
```

### Network rules
- ✅ Register `page.route` / `context.route` **before** `page.goto`.
- ✅ Use `context.route` (not `page.route`) in `beforeEach` to cover popups and new pages.
- ✅ Always include an `else route.continue()` branch in conditional handlers — unhandled methods hang.
- ✅ Use HAR replay (`page.routeFromHAR`) for large, stable mock surfaces.
- ❌ Don't mock your own primary user flow (login → checkout → confirm) end-to-end — at least one E2E per critical flow should hit the real backend.

### Handling server lag without flakiness

```ts
// Action triggers async backend work; UI updates eventually.
await test.step('place order', async () => {
  await page.getByRole('button', { name: 'Place order' }).click();
});

// Web-first assertion auto-retries up to expect timeout (raise per-call if needed)
await expect(page.getByRole('status'))
  .toHaveText(/Order #\d+ confirmed/, { timeout: 20_000 });

// For non-UI eventual state (e.g., an outbound job, queue worker)
await expect.poll(
  async () => (await request.get('/api/jobs/123').then(r => r.json())).status,
  { timeout: 30_000, intervals: [500, 1000, 2000, 5000] },
).toBe('completed');
```

---

## 6. Authentication

- ✅ Authenticate **once per role** in a `setup` project; save state with `await page.context().storageState({ path: 'playwright/.auth/user.json' })`.
- ✅ Reference it via `use: { storageState: 'playwright/.auth/user.json' }` on dependent projects, with `dependencies: ['setup']`.
- ✅ Add `playwright/.auth/` to `.gitignore`. Never commit session tokens.
- ✅ For multi-role suites, one auth file per role (`admin.json`, `user.json`); switch per-file with `test.use({ storageState: 'playwright/.auth/admin.json' })`.
- ✅ For per-worker isolation (tests that mutate user state), use a worker-scoped fixture that logs in a unique user per `workerIndex` and **overrides** the built-in `storageState` fixture.
- ✅ Prefer API-based login (`request.post('/login')` + `request.storageState`) over UI login when feasible — faster, less flaky.
- ❌ Don't log in via UI in `beforeEach`.
- ✅ Reset auth in a single file with `test.use({ storageState: { cookies: [], origins: [] } })` for "logged-out" tests.

---

## 7. Test isolation

- ✅ One concern per test, but **don't over-split** — a single test can model a user flow with multiple `test.step(...)` blocks. Best Practices: *"Each test should be completely isolated from another test and should run independently with its own local storage, session storage, data, cookies etc."*
- ✅ Every test starts from a known state: navigate in `beforeEach` or in the first step of the test.
- ✅ Generate unique test data (timestamp, uuid, or `workerIndex`) for anything that hits a shared backend.
- ❌ Don't depend on test order. Don't use `test.describe.serial` unless tests genuinely cannot be re-ordered (rare).
- ✅ Clean up server-side state in `afterEach` / fixture teardown when tests create persistent records.

---

## 8. Animations, transitions, rendering

- ✅ In `playwright.config.ts`, set `use: { contextOptions: { reducedMotion: 'reduce' } }` and respect `@media (prefers-reduced-motion: reduce)` in app CSS.
- ✅ For visual regression (`toHaveScreenshot`), `animations: 'disabled'` is the default — keep it.
- ✅ Mask volatile regions in screenshots: `toHaveScreenshot({ mask: [page.getByTestId('timestamp')] })`.
- ✅ Use `toMatchAriaSnapshot` (v1.49+) for structural assertions instead of pixel screenshots when you only care about semantic structure.

---

## 9. `test.step` for readability

- ✅ Wrap logical phases of a test in `await test.step('do X', async () => { ... })`. Improves trace viewer, HTML report, and failure messages.
- ✅ Use `{ box: true }` (v1.39+) inside helper methods so errors point to the caller, not the helper internals.
- ✅ Use `test.step.skip(...)` (v1.50+) for steps temporarily disabled with a TODO — don't comment-out code.
- ❌ Don't add `test.step` to one-line tests; only when you have ≥3 phases or a long flow.

---

## 10. Parallelism & CI configuration

```ts
// playwright.config.ts (recommended baseline)
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? '50%' : undefined,
  reporter: process.env.CI
    ? [['html', { open: 'never' }], ['github']]
    : 'list',
  use: {
    baseURL: process.env.BASE_URL ?? 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
    actionTimeout: 10_000,
    navigationTimeout: 30_000,
    contextOptions: { reducedMotion: 'reduce' },
  },
  expect: { timeout: 10_000 },
  projects: [
    { name: 'setup', testMatch: /.*\.setup\.ts/ },
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'],
             storageState: 'playwright/.auth/user.json' },
      dependencies: ['setup'],
    },
  ],
});
```

### Rules
- ✅ `fullyParallel: true` on green-field projects.
- ✅ CI: `retries: 2`, `trace: 'on-first-retry'` (or `'retain-on-failure-and-retries'` v1.58+ while hunting flakes), `forbidOnly: true`.
- ✅ Local: `retries: 0` — retries hide flakes during development.
- ✅ Use **sharding** for large suites: `--shard=1/4` across multiple CI runners.
- ✅ Tag tests (`test('login @smoke @auth', ...)`) and run subsets via `--grep @smoke`.

---

## 11. Debugging & flake hunting

- ✅ **Trace viewer first.** Don't read logs — open the `.zip` from `playwright-report/` or via `npx playwright show-trace path.zip` (also drop into `trace.playwright.dev`).
- ✅ Use `npx playwright test --ui` while developing — time-travel + auto-trace.
- ✅ Reproduce intermittent failures locally with `--repeat-each=20` on the suspect file before "fixing" with a retry.
- ✅ When a test passes locally but fails in CI: throttle CPU (`page.emulateCPUThrottling`) or run with `workers: 1` to see if it's resource-contention. Research presented at ACM FSE 2025 analyzing 52 projects across Java, JavaScript, and Python found **46.5% of flaky tests are Resource-Affected (RAFTs)** — failure rates shaped by CPU/memory/I/O at runtime rather than code.
- ✅ Install `eslint-plugin-playwright` (`playwright.configs['flat/recommended']`) — catches `waitForTimeout`, `force: true`, missing `await`, deprecated APIs, `no-focused-test`, etc. **Required.**

---

## 12. Iframes, shadow DOM, popups

- ✅ Iframes:
  ```ts
  const frame = page.frameLocator('iframe[name="payments"]');
  await frame.getByRole('button', { name: 'Pay' }).click();
  ```
- ✅ Shadow DOM: Playwright locators **pierce open shadow roots automatically** — use `getByRole` normally.
- ✅ Popups / new tabs: register listener before trigger:
  ```ts
  const popupPromise = page.waitForEvent('popup');
  await page.getByRole('link', { name: 'Open report' }).click();
  const popup = await popupPromise;
  ```

---

## 13. New / recent features worth using (Playwright ≥ 1.45)

- **Clock API** — `page.clock.install`, `fastForward`, `runFor`, `pauseAt` (v1.45). Deterministic time control instead of ad-hoc `Date` mocks.
- **`expect(locator).toMatchAriaSnapshot(yaml)`** (v1.49) — structural snapshot of accessibility tree. Better than pixel diffs for layout/semantics; codegen can produce these.
- **`test.step.skip` + step `timeout`** (v1.50).
- **`filter({ visible: true })`** (v1.51).
- **Copy prompt / Fix with AI** (v1.51/1.52) — surfaces in HTML report, UI mode, Trace Viewer, and VS Code Copilot; use to bootstrap fixes, do not trust blindly.
- **Aria snapshots in separate `.yml` files** + `updateSnapshots: 'changed'` (v1.52).
- **`locator.describe('label')`** (v1.53) — names locators in trace/reports for readability.
- **`expect(locator).toContainClass(...)`** (v1.54).
- **`trace: 'retain-on-failure-and-retries'`** (v1.58+) — keep traces from every retry while flake-hunting.
- **`expect(page).toMatchAriaSnapshot()` directly on Page** (v1.60).

---

## 14. Quick reference checklist

Before submitting a Playwright test, an agent must verify:

- [ ] No `waitForTimeout`, no `networkidle`, no `force: true`, no `waitForNavigation`/`waitForSelector`.
- [ ] No `page.$` / `$$` / `$eval` / element handles.
- [ ] Locators follow the `getByRole` → `getByLabel` → … ladder; CSS/XPath only with a comment justifying it.
- [ ] Every `getByRole` has `{ name }`; every `getByText` that asserts equality uses `{ exact: true }`.
- [ ] Every UI check is `await expect(locator).matcher()` (web-first); no `expect(await ...).toBe(...)` on UI state.
- [ ] Real-backend lag handled with `expect.toPass` / `expect.poll`, not `waitForTimeout` or inflated global timeouts.
- [ ] External / third-party APIs mocked via `page.route` (registered before `goto`); first-party APIs hit real unless deliberately mocked.
- [ ] Auth done via `setup` project + `storageState`; no UI login in `beforeEach`.
- [ ] Helpers expressed as `test.extend` fixtures; POM only where it earns its keep.
- [ ] Tests are independent: no shared mutable state, unique per-worker data where the backend persists.
- [ ] Logical phases wrapped in `test.step`.
- [ ] `playwright.config.ts` sets `fullyParallel: true`, `retries: 2` in CI, `trace: 'on-first-retry'`, `reducedMotion: 'reduce'`.
- [ ] `eslint-plugin-playwright` `flat/recommended` enabled and passes.
- [ ] No `test.only`, no `test.fixme` without ticket+owner, no committed `auth.json`.

---

## 15. Rationale digest (so an agent can reason about edge cases)

- **Auto-wait is the contract.** Locators + web-first assertions retry until the configured timeout. Bypassing this with hard waits or unawaited state reads is the dominant cause of flake. Luo, Hariri, Eloussi & Marinov, *"An Empirical Analysis of Flaky Tests"* (ACM FSE 2014), analyzing 201 flakiness-fixing commits across 51 open-source projects, found async-wait was the single largest root cause (~45% of cases). A 2025 FSE study across 52 projects further found ~46.5% are *resource-affected* — they pass or fail based on CPU/memory at runtime, not code — which is why CI-only flakes happen on green-locally code.
- **User-facing locators outlive DOM churn.** CSS classes and DOM structure change every sprint; ARIA roles and visible labels change rarely and are part of the product contract.
- **Fixtures > POM for agents** because they reduce ceremony and surface the dependency graph in test signatures.
- **Real backend lag ≠ flakiness.** `expect.toPass` / `expect.poll` express *eventual* consistency without inflating global timeouts; they fail fast on logic errors and slow only when the system genuinely is slow.
- **Mock what you don't own; trust what you do** (with retries). Mocking everything turns E2E into integration tests; mocking nothing makes the suite hostage to staging uptime.
- **One trace beats a hundred reruns.** Configure `trace: 'on-first-retry'` and read the trace before changing code.

— end of rules —
