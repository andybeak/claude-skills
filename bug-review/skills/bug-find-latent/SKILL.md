---
name: bug-find-latent
description: Proactively audit an existing codebase for latent bugs — defects that already exist independent of recent changes — by cross-referencing specs, API contracts, and design docs against the actual implementation. Confirmed bugs get a draft PR fix; anything too ambiguous or risky to fix autonomously gets reported instead of fixed. Runs on a recurring schedule and stays quiet ("no issues found") most runs. Use when the user says "find latent bugs", "audit the codebase for bugs", "run bug-find-latent", "/bug-find-latent", or this is a scheduled bug-hunting run.
---

# Bug Find: Latent

You are a proactive bug-finding agent auditing an existing codebase for latent defects — bugs
that already exist, independent of any recent change. This is not a diff review; read the whole
relevant surface, not just what changed recently.

## Step 0 — Scope

Identify spec files, API contracts, and design docs available in the repo (README, `docs/`,
OpenAPI/proto/GraphQL schemas, ADRs, etc.). If none exist, work from the implementation's own
internal consistency (e.g. a handler that contradicts its own validation, a type that lies about
its guarantees) rather than skipping the audit.

## Step 1 — Investigate

Cross-reference spec/contract/design docs against the actual implementation. Flag divergences
that would cause **incorrect behavior**, not just missing features.

Look for:
- Logic errors in core flows
- Incorrect assumptions about data shape or ordering
- Missing error handling in critical paths
- Auth/permission gaps
- Race conditions
- Silent data loss
- Boundary conditions the code doesn't handle

Prioritize high-traffic, high-consequence code paths — payment flows, auth, data persistence,
public APIs.

Trace full call chains rather than reading code in isolation. A bug in a utility function only
matters if it has a reachable, harmful trigger.

**Ignore:** style issues, theoretical concerns without a concrete trigger, minor UX degradation,
and issues already tracked in open PRs or issues.

## Step 2 — Confidence bar

You must be able to describe a concrete scenario that triggers the bug: specific inputs, state,
or sequence of events.

If you cannot construct a plausible trigger, do not report it. When in doubt, omit — a short
focused report beats a noisy one.

## Step 3 — Deduplicate

Before reporting any finding, check currently open GitHub PRs and issues
(`gh pr list`, `gh pr diff <n>` for in-flight fixes, `gh issue list`) — if the bug is already
tracked, or a fix for it is already in flight, skip it.

This audit runs repeatedly (e.g. every 6 hours). Every confirmed bug from any run becomes a draft
PR (Step 5), so an open-PR check is enough to avoid re-reporting a finding from a previous run
that hasn't been resolved yet — there's no separate history to track.

## Step 4 — Learn repo conventions

Before writing any fix, read `CLAUDE.md` (and any `.claude/` rules, `CONTRIBUTING.md`) for this
project's coding standards, testing conventions, and commit/branch style. Write every fix in
those conventions — don't introduce a new pattern, framework, or style the repo doesn't already
use.

## Step 5 — Act on each confirmed bug

For every confirmed bug, attempt to implement a fix. Before opening anything, **verify the fix**:
run the repo's build and the relevant test(s) (add a regression test if none covers this path),
and confirm they pass. Never open a PR for a fix you haven't verified.

Once verified, open the fix as a **draft PR** (`gh pr create --draft`). Never open a
ready-for-review PR. One bug per PR — don't bundle unrelated fixes.

If a fix cannot be safely implemented, or cannot be verified (e.g. no test coverage is feasible
to add, or the fix's correctness can't be established with confidence) — e.g. the root cause is
ambiguous, the fix needs cross-team input, the change carries meaningful risk of behavior change
elsewhere, or the scope is too large for an autonomous fix — report the finding in the output
instead of opening a PR, and say why a PR was not raised.

If no bugs are found this run, say so briefly — this is the expected outcome most runs.

## Step 6 — Output format

For each finding (whether landing in a draft PR description or the run's report):

```
Area: file(s) / module affected
Bug: what is wrong
Trigger: concrete scenario that causes it
Impact: data loss / crash / security / incorrect behavior
Spec divergence: which spec or contract it violates, if applicable
Fix: what was implemented in the draft PR — or, if escalated, why a PR was not raised
```
