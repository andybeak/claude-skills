---
name: plan-create
description: Create an implementation plan by sequencing an existing PRD (or, if none is supplied, the current chat context) into buildable phases, self-reviewed against the plan-review skill's seven dimensions until it reaches a "ready to execute" verdict. Optionally asks for a PRD as input; falls back to the conversation's chat context when none is given or available. Reads project architecture/testing docs and explores the codebase first for reuse and boundary constraints, and only asks the user when a real architecture/product decision is unresolved. Drafts the plan following the project's own plan-format conventions, then loops draft → plan-review → revise until it passes or the user stops. Use when the user says "create a plan", "write an implementation plan for X", "plan out this feature", "draft a plan from this PRD", "make me a plan", "/plan-create", or wants a new implementation plan built that will hold up to plan review.
---

# Plan Create

Write a plan that scores well against the `plan-review` skill — that skill's seven dimensions
**are** the definition of "good" used here, not a separate opinion. Don't duplicate or
reinterpret them; invoke `plan-review` itself as the grading step (Step 3) so the two skills
never drift apart.

Unlike `plan-review`, this skill is not advisory-only: it writes the plan file. It should still
never invent an architecture decision, a product scope call, or a testing standard the project
hasn't actually made — anything not decided or discoverable gets asked, or recorded as an
explicit open question, never silently resolved to keep the draft moving.

## Principle: a plan sequences, it doesn't decide

Hold this throughout, even in repos that don't say it explicitly (some do — e.g. a
`docs/plans/README.md` stating "a plan decides nothing"; treat that as the general default even
when unstated). A plan orders and phases work that's already been decided by a PRD, a spec, an
ADR, or an explicit product/architecture call the user makes when asked. If drafting surfaces a
real decision nothing has made yet — a new port shape, a data-storage tradeoff, a security
posture — that's a blocker to flag, not a gap to paper over with an implementer's best guess. The
smaller, routine calls a competent engineer would normally just make (which stdlib function,
what to name a file, which of two equivalent test helpers to reuse) don't need this treatment —
only calls that would be expensive to reverse or that change scope/architecture.

## Step 0 — Find the requirements input and project conventions

**Requirements source**, in this order:

1. The user's request already supplies a PRD — a file path, pasted text, or an explicit
   reference ("plan from the PRD we just wrote", "plan this out" right after a PRD was drafted
   earlier in this session). Use it directly, no need to ask.
2. Not supplied, and not unambiguous from session history: **ask** — "Do you have a PRD I should
   plan from (a file path is fine), or should I build the plan from our conversation so far?"
   Give the chat-context fallback as the recommended default so the user can accept it with
   "yes" rather than go find or write a PRD just to unblock this.
3. If the user has no PRD / says to use chat context: treat the conversation itself as the
   requirements source. State explicitly that no PRD was found or given, then **paraphrase back
   what's being treated as the plan's scope and goal** before drafting starts — there's no
   written doc to check that paraphrase against later, so getting it confirmed now is the only
   guard against building the wrong thing.
4. Don't silently default to "the most recently modified PRD-shaped file in the repo" the way
   `plan-review`/`plan-audit` do for *finding a plan to review* — that's fine when auditing
   something that already exists, but guessing which PRD to *build from* risks planning the
   wrong feature. Only auto-use a file if exactly one plausible candidate exists and the user's
   own request already named the feature it covers; otherwise ask per step 2.

**Project conventions** — run the same two mandatory checks `plan-review` runs (its Step 1), for
the same reason: the draft should be built inside these conventions from the start, not fixed up
after the first review pass.

- **Architecture docs**: `ARCHITECTURE.md`, `docs/architecture/**`, an architecture section in
  `CLAUDE.md`/`README.md`, module-boundary/layering rules, a documented "swap test" or port
  pattern. Note anything that constrains how new components must be structured.
- **Testing strategy docs**: `TESTING.md`, `docs/testing/**`, a testing section in `CLAUDE.md`/
  `CONTRIBUTING.md` — the unit/integration/e2e split and any pyramid-shape rule the plan's test
  sections must match.

Also check, specifically for drafting (not just reviewing):

- **Plan format conventions**: a `docs/plans/README.md`-style doc defining how this project
  writes plans (naming, required sections, status field, per-phase acceptance-criteria shape),
  or an existing plan file to mimic the *format* of (never its scope/content). If found, draft
  into that structure. If not found, use the generic template in Step 2.
- **Code reuse and boundaries**: explore the codebase for existing helpers/services/modules that
  already cover pieces of what's being planned — list concrete file/symbol matches now, so the
  draft leads with reuse instead of getting flagged for reinventing something on the first
  `plan-review` pass (its dimension 4). Note existing module/layering boundaries the plan must
  respect (its dimension 1). For a large or unfamiliar codebase, delegate this research to the
  `Explore` agent rather than skipping it.

State which conventions were found (with location) versus which fall back to generic — the same
way `plan-review`/`prd-create` report this — so the user knows upfront what the draft is being
built against.

## Step 1 — Resolve open decisions before drafting

Read the requirements source in full and list what it actually decides. For anything the plan
would need in order to sequence work but that isn't fixed by the PRD, an existing spec, an ADR,
or observed code — per the principle above, that's a blocker, not an implementer's call to make
silently:

- If the project has an ADR convention (check `docs/adr/` or equivalent) and the gap is
  architecturally significant (hard to reverse, security- or data-shape-relevant), say so and
  recommend resolving it via an ADR (or a spec amendment) before the plan is drafted — offer to
  do that first rather than drafting around the hole. This mirrors how a real architectural gap
  gets closed before sequencing, not folded into phase 3's implementation notes.
- If the gap is a smaller product/scope call (which of two reasonable behaviors, an explicit
  priority order under time pressure), ask the user directly — one question at a time, grill-me
  style, with a recommended answer — rather than drafting a guess into the plan.
- If nothing is actually missing, say so and move on; don't manufacture a question to seem
  thorough.

Tag each resolved gap as **decided** (from a doc or the user, just now) versus **assumption
flagged for override** (a small sequencing-level choice this plan is explicitly allowed to make
per its own convention — e.g. "which adapter ships first," "which stdlib feature covers this" —
called out so it's easy to challenge, not buried). This distinction needs to survive into the
draft itself, the same way `prd-create` carries decided-vs-assumption through to its output.

## Step 2 — Draft the plan

Write the file into the location Step 0's conventions imply (or `docs/plans/plan-<slug>.md` if
a `docs/plans/`-style convention exists with no fixed template, or ask where plans live if
nothing in the repo suggests a location). Structure it so each `plan-review` dimension has an
obvious place to land — adapt section names/format to match a discovered project convention, but
keep this mapping intact either way:

```
# Plan: <Feature>

**Status: Proposed.** Sequences <PRD/spec/ADR references> into N phases.
<name any project-specific pattern this plan is built against, e.g. a swap-test/port
convention from the architecture doc found in Step 0>

## Scope
**In scope** — the minimum slice(s) needed, drawn from the PRD/context, not expanded.
**Out of scope** — explicitly named, with where it's tracked instead (follow-up plan,
backlog item) if it's real deferred work rather than a rejection.        (dimension 5)

## Assumptions this plan makes (flagged, not decided)
<sequencing-level calls from Step 1's second bucket — implementer discretion, not
architecture>

## Phase 1 — <name>
**Why first:** <dependency ordering — what later phases need this to exist>  (dimension 1, 7)
- <deliverable, citing the Step 0 reuse findings: "extends existing X" not "new Y" where
  something already covers it>                                              (dimension 4)
- <deliverable>

**Tests:** <unit vs integration vs e2e split per the project's own testing doc or the
generic pyramid — what's covered at which level, and why that level>        (dimension 2)

**Observability & rollback:** <logging/metrics for this phase's blast radius; how this
phase is rolled back or operated if it fails post-deploy>                   (dimension 6)

**Acceptance:** <one concrete, checkable "done" condition>

## Phase 2 — <name>
...

## Security review
<one pass across every new input/endpoint/permission/trust-boundary the plan as a whole
introduces: who can trigger it, with what input, worst case — authn/authz, injection
classes relevant to this stack, secrets handling, resource exhaustion, cross-tenant/PII
exposure. Called out once, here, so it's checkable as its own artifact rather than
scattered thinly across phases and easy to miss.>                           (dimension 3)

## Compatibility & rollout
<any existing API/schema/config contract this plan changes, and how the rollout is
sequenced safely (expand-contract, dual-write, flag) rather than a single breaking step;
how a dependency failure/slowness during rollout is contained>              (dimension 7)
```

Fill every section from what Steps 0–1 actually produced — an empty or boilerplate section is
worse than an explicit "N/A — `<reason>`", which `plan-review` accepts the same way `prd-review`
does.

## Step 3 — Self-review loop

1. Invoke the `plan-review` skill (via the Skill tool) against the drafted file.
2. If its overall verdict is **"ready to execute"**: stop here, go to Step 4.
3. Otherwise, work through the blocking (and, time permitting, minor) findings it returned:
   - If a finding is fixable from context already gathered or further codebase exploration, fix
     it directly in the draft.
   - If it surfaces a real architecture/product decision Step 1 didn't catch, treat it exactly
     like a Step 1 gap: recommend an ADR/spec amendment, or ask the user — never patch it with a
     silent guess just to clear the finding.
   - Never close a finding by deleting or vaguing-up a phase/requirement just to dodge a
     rating — only close gaps by adding real content (a real test plan, a real rollback step, a
     real reuse citation).
4. Re-run `plan-review` on the revised draft and repeat.
5. Cap this loop at 3 full review passes. If it still hasn't reached "ready to execute" after 3
   passes, stop, report the remaining blocking items and the current verdict, and ask the user
   whether to keep iterating rather than looping silently.

## Step 4 — Handoff

Report:
- The plan's file path.
- The latest `plan-review` verdict, with its blocking/minor findings (or a pointer back to
  Step 3's last run).
- Any items still open that need the user's decision — either because the loop was capped, or
  because Step 1 flagged them as needing an ADR/spec amendment that hasn't happened yet.
