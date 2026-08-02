---
name: plan-review
description: Critically review an implementation plan against the project's documented architecture and testing strategy, hunt for abuse vectors, and verify the plan reuses existing code instead of reinventing it. Advisory only — never edits the plan or the code. Use when the user says "review this plan", "review my plan before I start", "is this implementation plan complete", "check if this plan aligns with our patterns", "validate this plan before we execute it", or "/plan-review".
---

# Plan Review

Critically review an implementation plan across seven dimensions before it gets executed. This
is a gate on the plan, not on the code — there may be no diff yet. Advisory only: never edit the
plan file, never edit source, never start implementing.

## Step 0 — Find the plan

Figure out what you're reviewing, in this order:

1. A file path or pasted plan the user gave you directly.
2. A plan produced by `EnterPlanMode`/`ExitPlanMode` earlier in this session.
3. The most recently modified plan-shaped file in the repo (e.g. `PLAN.md`, `docs/plans/*.md`,
   whatever this project actually uses).

State which one you're using. If none of these resolve, ask the user for the plan rather than
guessing.

## Step 1 — Load the project's documented standards

This skill runs across different repos — don't assume a layout. Search for:

- **Architecture docs**: `ARCHITECTURE.md`, `docs/architecture/**`, an architecture section in
  `CLAUDE.md` or `README.md`.
- **Testing strategy docs**: `TESTING.md`, `docs/testing/**`, a testing section in `CLAUDE.md` or
  `CONTRIBUTING.md`.

If a category has no documented source, say so explicitly in that section of the output and fall
back to the conventions actually visible in the codebase (existing module boundaries, existing
test patterns) — don't invent a standard that isn't written down or observed.

## Step 2 — Review across seven dimensions

For each dimension, look for concrete evidence in the plan and the codebase — don't rate on
vibes. Every finding needs a location (plan section, or file/line in the codebase) and a
severity: **blocking** (should not ship as-is) or **minor** (worth a look, not a gate).

**Decide how to run this step first:**

- **Small plan** — confined to one file/module, no new or changed API surface, no schema/data
  migration, no new permission or exposed input: work through the seven dimensions yourself,
  inline, in order below.
- **Anything larger** — multiple files or systems touched, a new/changed API surface, a
  schema/data migration, or a plan too big to hold fully in context at once: dispatch one
  subagent per dimension via the Agent tool, all launched in a single message so they run in
  parallel (do not send them one at a time). Give each subagent: the plan text (or its file
  path), the project root, the one numbered dimension below verbatim as its job, and the
  relevant excerpt(s) from the docs loaded in Step 1 (paste the excerpt if short, pass the file
  path if long — don't make every subagent re-discover the docs from scratch). Tell each
  subagent to return findings in the same format Step 3 uses: one line per finding with location
  + severity, or "no issues found." Once all seven return, synthesize them yourself in Step 3 —
  normalize format, don't paste raw subagent output verbatim.

1. **Architecture alignment** — does each new/changed component fit the documented architecture
   (layering, module boundaries, ownership)? Flag anything that crosses a boundary the docs say
   not to cross, or introduces a pattern the docs don't sanction. Also check the design against
   the load/scale it will actually see: no obvious bottleneck at expected volume, and no
   premature optimization for volume that doesn't exist.

2. **Testing strategy alignment** — does the plan's testing approach match what's documented
   (unit vs integration split, required coverage for the affected layer, fixtures/mocking
   policy)? Flag gaps: no test plan for a risky path, wrong test level for the change, missing
   negative/edge cases for anything with a branch or a trust boundary.

3. **Security review** — think like an attacker, not a linter. For every new input, endpoint,
   permission, or trust-boundary crossing in the plan: who can trigger it, with what input, and
   what's the worst they can do? Cover authn/authz gaps, injection classes relevant to this
   stack, secrets/credential handling, privilege escalation via new roles, resource exhaustion on
   anything newly exposed, and cross-tenant/PII data exposure.

4. **Code reuse** — for every piece of "new" functionality the plan proposes, check it against
   this order and stop at the first rung that covers it: (a) does this codebase already have a
   helper, service, or module that does it — grep/search and list exact file/symbol matches; (b)
   does the language's stdlib do it; (c) does a native platform/runtime feature do it; (d) does
   an already-installed dependency do it. Only functionality that falls through all four rungs is
   legitimately new. A plan that reimplements something a few files over, or hand-rolls what
   stdlib or the platform already provides, is a finding, not a nitpick. Same lens for
   infrastructure: flag a new dependency, managed service, or piece of infra the plan introduces
   when something already in use covers it — new spend/footprint needs a reason, not just
   convenience.

5. **Scope discipline** — compare the plan against the actual request/ticket. Flag anything
   included that wasn't asked for: unrelated refactors, "while I'm in here" cleanup. Then, for
   each piece of functionality that *was* asked for, ask two more questions: does this specific
   piece need to exist as its own thing (flag speculative generality — an interface with one
   implementation, a factory for one product, a config knob for a value that never changes), and
   is what's proposed the smallest form that works (a one-liner where the plan writes a function,
   a function where it writes a class)?

6. **Observability & operational readiness** — for behavior the plan adds or changes, does it
   include logging/metrics/alerting proportionate to its blast radius? Flag silent failure modes
   — a step that can fail with no visible signal. Also check the plan covers how it gets
   rolled back or operated if something goes wrong post-deploy (a revert path, a flag, a
   documented manual step) — not just how it ships.

7. **Compatibility & reliability** — does the plan break an existing API contract, DB schema,
   config format, or consumer that isn't part of the plan's own scope? For schema/contract
   changes, check the plan sequences the rollout safely (e.g. dual-write/expand-contract) rather
   than flipping it in one step. Also check how the new/changed component behaves when a
   dependency it relies on fails or is slow — does it degrade gracefully or take down more than
   its own blast radius, and can the change itself be undone if it turns out wrong.

## Step 3 — Output

If Step 2 used subagents, fold their results in here — same structure as if you'd reviewed
inline. A subagent returning malformed or off-topic output is a reason to redo that one
dimension yourself, not to drop it from the report.

Structure the response as one short section per dimension (Pass, or numbered findings with
location + severity), followed by:

**Overall verdict** — one line: ready to execute, or needs revision.

**Blocking items** — findings that should be resolved before execution (numbered, one line
each).

**Worth a look** — minor findings, not gating.

Keep it proportionate to the plan's size. A five-line plan doesn't need seven paragraphs; a
migration plan touching shared infra does. If a dimension turns up nothing, say "no issues
found" in one line and move on — don't pad it.
