---
name: prd-review
description: Critically review a Product Requirements Document (PRD) against the hallmarks of a good PRD — problem framing, scope boundaries, testable requirements, non-functional coverage, UX/modern-web design, traceability, readiness to hand off to a planning agent, and testable-behavior/test-pyramid handoff — and report gaps. Advisory only — never edits the PRD. Use when the user says "review this PRD", "review my PRD", "audit this PRD", "does this PRD have gaps", "check my PRD against best practices", "is this PRD ready", "is this PRD ready to plan from", "can an agent build a phased plan from this", "does this PRD cover the key test scenarios", "/prd-review", or shares a PRD and asks what's missing.
---

# PRD Review

Critically review a PRD across the dimensions below and report gaps. This is a gate on the document,
not on the product decisions inside it — don't second-guess product calls the PRD makes
deliberately, only flag where the PRD fails to specify, bound, or justify something a reader would
need. Advisory only: never edit the PRD, never write the missing sections yourself unless asked.

This PRD's downstream reader is often not a human alone — it gets handed to a planning agent
(e.g. `EnterPlanMode`) to produce a phased implementation plan with no further back-and-forth.
Dimensions 9 and 10 exist specifically for that: a PRD can score well on dimensions 1–8
per-requirement and still leave a planning agent unable to sequence the work or know which
behaviors actually matter, because the narrative context (user stories, "why this matters") that
justified each requirement usually doesn't survive past the PRD itself. Judge both as their own
concern, not as covered by general testability (dimension 3) or traceability (dimension 7).

## Step 0 — Find the PRD

Figure out what you're reviewing, in this order:

1. A file path or pasted PRD text the user gave you directly.
2. A doc produced or discussed earlier in this session.
3. The most recently modified PRD-shaped file in the repo (e.g. `docs/prd/**`, `docs/specs/**`,
   `PRD.md`, a spec file the project calls its PRD).

State which one you're using. If none of these resolve, ask the user for the PRD rather than
guessing.

**Mandatory: check for the project's own documented standards before rating anything.** This
skill runs across many repos with no shared layout, so never assume a specific file exists. Two
checks are required every time, and dimensions 6 and 10 below depend on their outcome. These
checks assume the PRD is a file in, or otherwise clearly associated with, the current repo. If the
PRD came from pasted text or an external doc (resolution path 1 or 2) and nothing in the current
repo appears related to it, skip both checks, use the generic fallbacks for dimensions 6 and 10,
and say explicitly that the fallback was used because the PRD isn't tied to this repo — don't
apply another project's conventions just because a repo happens to be open.

- **Spec/PRD format conventions** — look for a documentation index, architecture rules doc, or
  contributing guide that defines how this project's own requirements docs are structured (e.g. a
  `docs/` index, an `ARCHITECTURE.md`, a wiki link in the README). If one exists, judge structure
  against it and don't fault the PRD for not matching a generic template. If none exists, judge
  structure against the generic dimensions below with no project-specific adjustment.
- **Testing strategy / pyramid doc** — look for a documented testing strategy (e.g. `TESTING.md`,
  a testing section in `CONTRIBUTING.md`, a rule file under `docs/`). If one exists, judge
  dimension 10 against its specific pyramid/levels. If none exists, fall back to the generic
  testing pyramid described in dimension 10 — do not skip the check just because no doc exists.

State explicitly in the output which of these two were found (with location) versus which used
the generic fallback, so the reader knows whether the review reflects their own conventions or a
default.

## Step 1 — Decide how to run the review

- **Short PRD** (single feature, fits in one read, roughly fewer than 3 distinct features/epics
  and under ~400 lines): work through the dimensions yourself, inline, in order below.
- **Long or multi-feature PRD** (at or above that threshold, or long enough that holding all of it
  in context at once is unreliable): dispatch subagents via the Agent tool as follows.
  - Every subagent prompt must include the PRD text (or file path), the one numbered dimension
    below verbatim as its job, the rating scale from Step 2 (including the N/A and evidence
    rules), and Step 0's findings — the located conventions/testing-pyramid doc, or an explicit
    note that the generic fallback applies. Without that last piece, a subagent has no way to
    apply the project-specific conventions dimensions 6 and 10 depend on.
  - Dimensions 1–9 have no cross-dependencies: dispatch their subagents together in a single
    message so they run in parallel. Tell each subagent to return one row per requirement/section
    it evaluated plus a one-line rationale — no free-form essays.
  - Dimension 10 requires dimensions 6's and 9's findings (it must cross-check its scenarios
    against dimension 6's edge cases and dimension 9's risk flags) — it cannot run blind in the
    same parallel batch. Wait for dimensions 6 and 9 to return, then dispatch dimension 10's
    subagent with their findings included in its prompt.
  - Synthesize all results yourself in Step 3; a subagent returning malformed or off-topic output
    is a reason to redo that dimension yourself, not to drop it from the report.

## Step 2 — Review across ten dimensions

For every dimension, rate what you find using this scale, and require a concrete quote or
location for anything below Strong — "feels thin" is not a finding:

- ✅ **Strong** — present, specific, and usable by the team that has to act on it.
- ⚠️ **Weak** — present but vague, incomplete, or would produce disagreement between two readers.
- ❌ **Missing** — not addressed at all.
- **N/A** — the dimension doesn't apply to this PRD (e.g. UX for a pure data-pipeline feature).
  Never use N/A silently — always state the one-line reason it doesn't apply.

For dimensions 2, 3, 9, and 10 — the ones this skill treats as gates on build-readiness — require
a one-line location or quote even for a Strong rating (e.g. "out-of-scope list at §4"), not just
"-". These are the dimensions most likely to be rated Strong on a skim rather than a check, and a
Strong rating on them should be as verifiable as a Weak or Missing one. "-" remains fine for a
Strong rating on the other dimensions.

Dimensions 5, 6, 9, and 10 are composite — each bundles several sub-checks under one rating. Rate a
composite dimension at its *worst* sub-check level (any Missing sub-check makes the dimension
Missing; otherwise any Weak sub-check makes it Weak). List each failing sub-check individually in
the finding column, or as a short bullet list beneath the table row if they don't fit in one
cell — don't collapse them into a single vague sentence a second reviewer couldn't reproduce.

1. **Problem & goal framing** — Is the user/business problem stated, with why it matters? Is
   there a measurable goal or success metric, not just a feature description? A PRD that opens
   with "build X" instead of "users can't do Y, so we're building X to enable Z, measured by W" is
   Weak or Missing here.

2. **Scope boundaries & prioritization** — Are in-scope and out-of-scope explicitly listed? Is
   there a priority order (must/should/could, or equivalent) so a team under time pressure knows
   what to cut first? Absence of an explicit out-of-scope list is a common Weak/Missing finding —
   call it out even if scope seems "obvious" from the in-scope list, since that's exactly where
   silent scope creep happens.

3. **Requirement testability** — Is each requirement concrete and verifiable, or does it rely on
   subjective language ("intuitive", "fast", "user-friendly") with no measurable threshold? Are
   requirements atomic (one testable statement each) rather than bundled paragraphs that hide
   multiple asks? Flag any requirement a QA engineer couldn't turn directly into a pass/fail test
   case.

4. **Outcome vs. implementation altitude** — Does the PRD stay at the level of user-facing
   behavior and let engineering/design own the "how"? Flag over-specification (dictating specific
   UI widgets, algorithms, or architecture without a stated reason) and under-specification
   (requirements so vague that two valid-but-different implementations would both satisfy it).

5. **Non-functional requirements** — This dimension is composite (see the aggregation rule in
   Step 2's preamble): check each of the following as a real requirement with thresholds where
   relevant, not omitted or waved at with a single adjective.
   - **Performance, security, reliability, scale**: concrete targets, not just named as concerns.
   - **Privacy/compliance**: for anything handling user data, check it isn't silently assumed.
   - **Observability**: does the PRD say how the team will know the feature is working (and
     failing) in production — logging, metrics, or alerting expectations — or does "done" stop at
     shipping with no way to detect a regression?
   - **Rollout/release safety**: migration strategy, feature-flagging or staged rollout, backwards
     compatibility, and rollback/kill-switch expectations, where the feature changes existing
     behavior or touches data in place.

6. **UX & modern web design** — Treat this as a first-class dimension, not a subset of "non-functional." For any PRD describing a user-facing surface, check explicitly for:
   - **User flows & edge cases**: primary happy-path flow is described, plus key edge cases
     (empty state, first-use/onboarding, error recovery, permission-denied, offline/slow network).
   - **States, not just the ideal case**: loading, empty, error, and success states are each
     addressed for any view/interaction that fetches or mutates data.
   - **Accessibility**: WCAG-level expectations (keyboard navigation, screen-reader/semantic
     structure, color contrast, focus management) are stated as requirements, not left implicit.
   - **Responsive/multi-device behavior**: is behavior on mobile/tablet/narrow viewports specified
     where the surface is web-facing, not just a desktop assumption?
   - **Performance as UX**: are there any perceived-performance expectations (e.g. Core Web
     Vitals-style targets — load time, interaction responsiveness, layout stability) rather than
     leaving "feels fast" unquantified?
   - **Consistency with existing patterns**: first check whether the project has a design
     system, component library, or style guide (e.g. a `packages/ui`-style shared component
     package, a Storybook config, a `tokens`/theme file, a documented component manifest, a
     STYLEGUIDE.md). If one exists, judge whether the PRD reuses its existing components/patterns
     and flag any new one-off pattern the PRD introduces where an existing component already
     covers it. If no such system exists in the project, fall back to generic consistency
     checks instead: does the PRD's described UI stay consistent with itself (same terminology,
     same interaction pattern for the same kind of action, no contradictory navigation models
     across sections) rather than inventing a different pattern per screen?
   - **Usability heuristics spot-check**: sanity-check the flows against basic heuristics (clear
     feedback after actions, easy error recovery/undo, no unnecessary steps, consistent
     terminology) — flag anything that would visibly violate one.

   A PRD for a non-UI/backend-only feature can mark this dimension N/A, but state that
   explicitly rather than skipping it silently.

7. **Stakeholder alignment & traceability** — Is it clear who owns this PRD and who needs to sign
   off (design, eng, QA, legal, etc.)? Can each requirement be traced to an acceptance criterion a
   reviewer could check the shipped feature against? A PRD with no way to verify "done" against it
   is a traceability gap.

8. **Document hygiene / living-document status** — Does the PRD have a status (draft/review/
   accepted), an owner, and a last-updated date or changelog? A PRD with no status is a risk:
   readers can't tell if it's still authoritative.

9. **Plannability for phased delivery** — Assume the next reader is a planning agent that must
   produce a phased plan with no chance to ask follow-up questions. Check specifically:
   - **Decomposability**: are requirements already grouped into distinct, independently
     describable units (features, milestones, epics) rather than one flat, unordered list? A PRD
     can pass dimension 3 (each requirement individually testable) and still fail here if there's
     no structure above the requirement level for a plan to hang phases on.
   - **Sequencing signals**: does the PRD state hard dependencies or ordering constraints between
     units ("X must exist before Y", "requires the new schema first", "blocked on the other
     team's API")? Silence here forces the planning agent to guess at phase order, and a wrong
     guess costs a rebuild, not a re-read.
   - **Risk/uncertainty flagged where it exists**: are the units carrying technical or product
     risk (unproven approach, external/third-party dependency, unvalidated user demand, a
     migration) marked as such, rather than written in the same confident tone as settled work?
     Phased plans typically front-load risk with a spike or walking skeleton — an agent can't
     choose to do that if the PRD hides which parts are actually uncertain.
   - **Assumptions vs. decisions, separated**: does the PRD distinguish what's decided from what's
     an open question or an assumption pending validation? A planning agent needs to know what
     it's allowed to treat as fixed input versus what it should surface back to a human rather
     than silently resolve on its own.
   - **Integration/reuse pointers**: does the PRD name what existing systems, modules, or data
     this touches — enough for a planning agent to scope its own investigation — without
     dictating the implementation itself? (This is the boundary from dimension 4: naming "this
     extends the existing X module" is a pointer; specifying X's internal design is overreach.)
   - **Per-unit acceptance criteria**: does each decomposable unit have its own "done" condition,
     so a phase boundary maps to something checkable, rather than the whole PRD being one
     all-or-nothing "done"?

   Rate this the same way (Strong/Weak/Missing), but make the finding actionable for this
   specific use: name which units/requirements would leave a planning agent guessing, and what it
   would have to guess (order, risk, scope of investigation, or done-ness).

10. **Testable behavior handoff (test-pyramid pre-work)** — User stories and their surrounding
    narrative justify *why* a behavior matters, but that narrative typically does not survive past
    the PRD — by build time, a planning/implementing agent usually sees only the extracted
    requirements, not the story context that produced them. Check for an explicit section
    (commonly labeled "Testing," "Key Scenarios," or "Behavioral Acceptance Criteria") that
    distills the user stories into scenarios a plan can act on directly. That section should:
    - State concrete, user-observable scenarios (Given/When/Then or equivalent acceptance-scenario
      prose) — not implementation-level unit-test cases; that decomposition is engineering's job
      at build time, not the PRD's.
    - Map each scenario to the requirement/unit it covers, so a planning agent can assign it to
      the right phase instead of bolting all testing onto the end.
    - Cover the primary happy path *and* the highest-risk/highest-value edge cases named elsewhere
      in the PRD (cross-check against dimension 6's edge cases and dimension 9's risk flags) — a
      section that only restates the happy path is Weak, not Strong.
    - Indicate roughly where each scenario sits on a testing pyramid, so the plan doesn't default
      every scenario to the same test level. Use whichever pyramid the project has documented (per
      the mandatory check in Step 0) if one was found. If none was found, apply this generic
      fallback pyramid: **unit** (single function/component in isolation, no I/O — cheap, most
      numerous), **integration** (a few real collaborators wired together — a service plus its
      real database, two modules talking through their real interface — fewer than unit), **e2e/
      behavioral** (a full user-observable flow through the real system end-to-end — fewest,
      reserved for scenarios where the value is in the flow actually working, not any one piece
      of it). A scenario belongs at the *lowest* level that can actually verify it; mark a
      scenario "e2e" only when the point is the end-to-end flow itself (e.g. "user completes
      checkout"), not because it's convenient to write one large test instead of several small
      ones.

    A PRD missing this section entirely is a common, easy-to-miss gap — nothing looks wrong until
    build time, when the plan has requirements but no signal for which behavior the user story was
    actually about. Flag it explicitly as a plannability risk: "the plan will not know which
    end-to-end behavior matters without this," not just as a generic documentation gap.

## Step 3 — Output

Structure the response as:

```
## PRD under review
<file/source, and which conventions it's being judged against>

## Dimension review
| # | Dimension                        | Rating | Finding (location/quote) |
|---|-----------------------------------|--------|---------------------------|
| 1 | Problem & goal framing            | ⚠️      | <specific gap>            |
| 2 | Scope boundaries & prioritization | ❌      | <specific gap>            |
| 3 | Requirement testability           | ✅      | -                         |
| 4 | Outcome vs. implementation        | ...    | ...                       |
| 5 | Non-functional requirements       | ...    | ...                       |
| 6 | UX & modern web design            | ...    | ...                       |
| 7 | Stakeholder alignment/traceability| ...    | ...                       |
| 8 | Document hygiene                  | ...    | ...                       |
| 9 | Plannability for phased delivery  | ...    | ...                       |
| 10| Testable behavior handoff         | ...    | ...                       |

For a composite dimension (5, 6, 9, 10) rated Weak or Missing, list each failing sub-check as a
bullet directly beneath its table row instead of squeezing them into the Finding cell.

## Overall verdict
Derive this, don't eyeball it — apply in order and stop at the first match:
1. Any ❌ on dimension 2, 3, 9, or 10 → **needs significant rework**.
2. Any ⚠️ on dimension 2, 3, 9, or 10 (and no ❌ there) → **ready with fixes first**.
3. Otherwise → **ready to hand to a planning agent**.

State the verdict in one line, then name which rule/dimension drove it.

## Gaps to close before build
<numbered list, one line each — Missing and Weak findings that block confident implementation>

## Worth tightening
<numbered list, one line each — Weak findings that aren't blocking>

## What a planning agent would have to guess right now
<numbered list, one line each, drawn from dimensions 9 and 10's findings — name the unit and
what's unresolved (order / risk / investigation scope / done-ness / which behavior to test end-to-
end). Empty list is a fine outcome — state "none, this PRD sequences, bounds, and specifies the
test-worthy behavior of its own work" rather than omitting the section.>

## Suggested phase boundaries (if dimension 9 is Strong or Weak, not Missing)
<a short, tentative grouping of the PRD's own units into phases, based only on the dependencies,
risk flags, and units the PRD itself already states — not a new plan, and not implementation
detail. Skip this section entirely if dimension 9 is Missing (there's nothing in the PRD to
group), and say why.>

## Strengths worth keeping
<one or two lines — what the PRD already does well, so a rewrite doesn't accidentally lose it>
```

Keep it proportionate: a one-page PRD doesn't need ten paragraphs of prose — a filled table plus
a short verdict is enough. Never pad a Strong rating with explanation; "-" is a fine finding for a
Strong row on dimensions 1, 4, 5, 6, 7, and 8 — dimensions 2, 3, 9, and 10 still need a one-line
location per Step 2's evidence rule, even when Strong. If a dimension is genuinely N/A (e.g. UX
for a pure data-pipeline PRD), say so in one line rather than forcing a rating.
