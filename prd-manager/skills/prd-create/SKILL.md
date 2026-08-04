---
name: prd-create
description: Create a Product Requirements Document (PRD) through an iterative interview-and-draft process, run as a technical product owner championing the end user of the product (not just transcribing whoever is being interviewed), self-reviewed against the prd-review skill's ten dimensions until it reaches a "ready to hand to a planning agent" verdict. Reads project specs/ADRs/architecture docs and explores the codebase first to fill gaps itself, and only interviews the user (grill-me style, one question at a time with a recommended answer) when documents conflict or a real gap remains that nothing in the repo resolves. Drafts the PRD following the project's own spec conventions, then loops draft → prd-review → revise until it passes or the user stops. Use when the user says "create a PRD", "write a PRD for X", "help me draft a PRD", "draft a PRD for this feature", "/prd-create", or wants a new requirements doc built that will hold up to PRD review.
---

# PRD Create

Write a PRD that scores well against the `prd-review` skill — that skill's ten dimensions **are**
the definition of "good" used here, not a separate opinion. Don't duplicate or reinterpret them;
invoke `prd-review` itself as the grading step (Step 3) so the two skills never drift apart.

Unlike `prd-review`, this skill is not advisory-only: it writes the PRD file. It should still
never fabricate product decisions, business data, or metrics the user hasn't given or the
codebase doesn't support — anything not decided or discoverable gets asked, or recorded as an
explicit open assumption, never invented to make a section look complete.

## Persona: technical product owner, championing the end user

Run this whole process as a technical product owner would, not a transcription service. The PRD
exists to make the feature add value to the **end user of the product** — the person who'll
actually use what gets built — and that's a different person from whoever is being interviewed in
Step 1 (usually a PM, founder, or engineer speccing work on the end user's behalf). Hold that
distinction throughout: the interviewee's convenience, habits, or initial framing are not the
same thing as end-user value, and the two should be told apart out loud when they diverge.

Concretely, this means:
- Every problem statement, requirement, and scope decision should trace to a stated benefit for
  the end user. If the interviewee gives you a feature request with no clear end-user benefit
  ("we want X"), push back and ask what it lets the end user do, before recording it — don't
  accept "build X" as a requirement on its own.
- Use the technical grounding from Step 0 (specs, ADRs, code) to judge feasibility and challenge
  scope, the way a product owner with real system knowledge would — not to rubber-stamp whatever
  is easiest to build, and not to gold-plate beyond what the end user needs either.
- When a requirement or piece of scope looks like it serves the interviewee's convenience (less
  work for them, a easier internal process) rather than the end user, say so and ask whether it
  belongs in this PRD or somewhere else.
- This is a stance to bring to the interview and the draft, not a new PRD section — it should
  show up as sharper problem framing (dimension 1), a tighter scope boundary (dimension 2), and
  requirements that hold up under "how does this help the end user" (dimensions 3 and 4), not as
  extra structure `prd-review` doesn't grade.

## Step 0 — Scope the ask & find project conventions

1. Identify the feature/product area from the user's request. If it's too vague to start an
   interview from (e.g. "write me a PRD"), ask one clarifying question before proceeding rather
   than guessing a topic.
2. Run the same two mandatory repo checks `prd-review` runs, for the same reason — the draft
   should be built inside these conventions from the start, not fixed up after the fact:
   - **Spec/PRD format conventions** — a documentation index, architecture rules doc, or
     contributing guide defining how this project structures requirements docs. If found, draft
     into that structure. If not, use the generic template in Step 2.
   - **Testing strategy / pyramid doc** — if found, use its levels when writing the Testing /
     Key Scenarios section (Step 2, dimension 10). If not, use the generic unit/integration/e2e
     fallback `prd-review` itself falls back to.
3. Also check for a design system, component library, or style guide (the same thing dimension 6
   of `prd-review` checks for) if the feature has a UI surface — the interview should ask about
   reuse against real components, not invent UI in the abstract.
4. Look for an existing PRD in the repo to mimic *format* only — never copy its scope or
   requirements into the new one.
5. **Read every other project document relevant to the feature area** before the interview
   starts — specs, ADRs, architecture rule docs, prior PRDs, READMEs, changelogs, anything a
   documentation index points at that touches the module(s) this feature will live in or
   integrate with. This is the primary source for Step 1, not a fallback: a decided ADR, a
   normative spec's module boundary, or an existing pattern in a sibling spec should settle a
   question outright, the same way exploring the codebase does. Note what each relevant doc
   already establishes (decided architecture, existing scope boundaries, naming, module
   ownership) so Step 1 can lead with "the spec already says X" instead of asking for X.
6. State which conventions were found (with location) versus which fall back to generic, the same
   way `prd-review` reports this — the user should know upfront what the draft is being built
   against.

## Step 1 — Interview (grill-me style)

The default move for every branch below is **read first, ask last**. For each branch, in order:

1. Check what Step 0's document read already established — does a spec, ADR, architecture rule,
   or prior PRD already answer this, or set a constraint that answers it? If so, state that as the
   answer ("the spec already fixes this as X") rather than asking.
2. If the documents don't cover it, explore the codebase — existing modules, current behavior,
   available design-system components, integration points.
3. **Only ask the user when neither of those resolves it** — a genuine gap where no document or
   code answers the question — or when documents/code **conflict** with each other (a spec says
   one thing, an ADR or the code says another). A conflict is always surfaced to the user, never
   silently resolved by picking the doc that seems more recent or more authoritative-looking —
   state both sides and ask which governs.

When you do ask, ask one question at a time, and give a recommended answer so the user can accept
with "yes" rather than compose one from scratch. The bar for "asked" should be genuinely high: if
a question is answerable by reading one more file, read the file instead of asking. Say what you
found from documents/code as you go, so the user sees the interview is grounded rather than
guessing — this also gives them a natural point to correct you if a doc is stale.

As you go, tag every answer explicitly as **decided** (stated by a doc, the code, or the user) or
**assumption pending validation** (inferred, or a doc conflict resolved provisionally) — capture
this distinction live, don't try to reconstruct it at the end from a flat transcript. Dimension 9
depends on this split surviving into the draft.

Work the branches in roughly this order (skip a branch fast once resolved; don't re-litigate a
settled item):

1. **Problem & goal** — what can't the *end user* do today, why does it matter to them, what's the
   measurable goal or success metric? Reject a framing that starts at "build X" — push back to
   "why", specifically "why does this help the end user", before accepting the feature
   description. If the answer is really about the interviewee's or business's convenience rather
   than end-user value, record that honestly (it can still be a legitimate goal) but don't dress
   it up as user value it isn't.
2. **Scope & priority** — what's explicitly in scope, and — always ask this one, even if it feels
   obvious — what's explicitly *out* of scope. Get a priority order (must/should/could), and when
   proposing what gets cut under time pressure, weight it by end-user impact: what the end user
   would miss most stays in, what's nice-for-the-team-but-invisible-to-the-user is first to cut.
3. **Requirements** — decompose into atomic, individually testable statements (one behavior per
   requirement, no bundled paragraphs). For each, ask enough to state it concretely — reject
   subjective language ("intuitive", "fast") without a threshold. If a proposed requirement has no
   traceable end-user benefit, push back before adding it rather than including it to be thorough.
4. **Altitude check** — for anything that sounds like it's dictating implementation (a specific
   algorithm, widget, or architecture) ask whether that's a deliberate constraint or just habit;
   drop to outcome-level language unless there's a stated reason to pin the "how".
5. **Non-functional requirements** — ask through each sub-check, don't skip silently:
   performance/security/reliability/scale targets; privacy/compliance if user data is touched;
   observability (how will the team know it's working/failing in production); rollout safety
   (migration, flagging, backwards compatibility, rollback) if it changes existing behavior.
6. **UX & modern web design** (skip this branch entirely and mark N/A in the draft if there's no
   user-facing surface) — primary flow plus edge cases (empty, first-use, error recovery,
   permission-denied, offline/slow network); loading/empty/error/success states; accessibility
   expectations; responsive/multi-device behavior; perceived-performance targets; and — checked
   against whatever design system Step 0 found — which existing components this reuses versus
   where it deliberately introduces something new.
7. **Stakeholders & traceability** — who owns this PRD, who signs off (design/eng/QA/legal/etc),
   and what acceptance criterion makes each requirement checkably "done".
8. **Document hygiene** — status (draft/review/accepted), owner, date. Set this immediately in the
   draft rather than asking about it last; it's cheap and easy to forget.
9. **Decomposition & phasing** — group requirements into distinct, independently describable units
   (features/milestones/epics), not one flat list. Ask about hard dependencies/ordering between
   units, and which units carry technical or product risk (unproven approach, third-party
   dependency, unvalidated demand, migration) versus which are settled work — risk gets marked
   explicitly, not written in the same confident tone as everything else.
10. **Testable scenarios** — for each unit, get the primary happy-path scenario plus the
    highest-risk/highest-value edge cases (cross-check against what came out of branches 6 and 9).
    State each as a concrete, user-observable scenario (Given/When/Then or equivalent) — not an
    implementation-level unit-test case, that's engineering's job at build time. Assign each a
    rough pyramid level (per Step 0's testing-doc finding, or the generic unit/integration/e2e
    fallback) at the *lowest* level that can actually verify it.

## Step 2 — Draft the PRD

Write the file into the location Step 0's conventions imply (or `docs/prd/<slug>.md` if none was
found). Structure it so every `prd-review` dimension has an obvious place to land — this mapping
is deliberate, keep it 1:1:

```
# <Feature> PRD
Status: <draft/review/accepted> · Owner: <name> · Last updated: <date>   (dimension 8)

## Problem & Goal                                                        (dimension 1)
## Scope
### In scope / Out of scope / Priority                                   (dimension 2)
## Requirements
<atomic, testable, one behavior each>                                    (dimensions 3, 4)
## Non-Functional Requirements
### Performance, Security, Reliability, Scale
### Privacy / Compliance
### Observability
### Rollout & Release Safety                                             (dimension 5)
## UX & Design                                                           (dimension 6, or "N/A — <reason>")
### Flows & edge cases / States / Accessibility / Responsive /
### Performance-as-UX / Consistency with <design system>
## Stakeholders & Sign-off / Acceptance Criteria                         (dimension 7)
## Phases / Units
<unit>: depends on <...>, risk: <...>, assumption vs decision: <...>,
per-unit acceptance criterion                                            (dimension 9)
## Testing / Key Scenarios
<Given/When/Then, mapped to unit, tagged with pyramid level>             (dimension 10)
```

Fill every section from what the interview and codebase exploration actually produced — an empty
or boilerplate section is worse than an explicit "N/A — <reason>", which `prd-review` accepts.

## Step 3 — Self-review loop

1. Invoke the `prd-review` skill (via the Skill tool) against the drafted file.
2. If its verdict is **"ready to hand to a planning agent"**: stop here, go to Step 4.
3. Otherwise, work through the Missing/Weak findings it returned:
   - If a finding is fixable from context already gathered or from further codebase exploration,
     fix it directly in the draft.
   - If it requires a product/business call that wasn't covered in Step 1, ask the user — one
     question at a time, grill-me style, with a recommended answer — then fold the answer in.
   - Never close a gap by softening or deleting a requirement just to avoid a Weak/Missing rating;
     only close gaps by adding real content.
   - Stay in the technical-product-owner persona while patching: a gap-filling requirement or
     scope note still needs a real end-user benefit, not text added purely to satisfy the rubric.
4. Re-run `prd-review` on the revised draft and repeat.
5. Cap this loop at 3 full review passes. If it still hasn't reached "ready to hand to a planning
   agent" after 3 passes, stop, report the remaining gaps and the current verdict, and ask the
   user whether to keep iterating rather than looping silently.

## Step 4 — Handoff

Report:
- The PRD's file path.
- The latest `prd-review` verdict and its dimension table (or a link back to Step 3's last run).
- Any items still open that need the user's decision — either because the loop was capped, or
  because they were deliberately recorded as "assumption pending validation" during the interview
  and still need validating before build.
