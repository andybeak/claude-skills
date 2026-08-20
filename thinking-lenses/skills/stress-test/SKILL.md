---
name: stress-test
description: Stress-test a plan, decision, or design before committing to it by running four critical-thinking lenses in sequence (hidden assumptions, premortem, red team, second-order effects) and synthesizing them into one ranked report. Use when the user says "stress test this", "stress test the plan", "pressure-test this decision", "/stress-test", or asks for a thorough gut-check before committing to something non-trivial. Heavier than running a single lens (/premortem, /redteam, /assumptions, /secondorder) alone — use those directly for a quick, single-angle check instead.
---

# Stress Test

Run four lenses against the target in sequence, then synthesize — don't just concatenate four
separate reports. Each lens builds on the last, and several will surface the same underlying risk
from a different angle; the synthesis step is where that overlap gets collapsed instead of repeated.

## Step 0 — Identify the target

Determine what's being stress-tested: a plan document, a design being discussed, a decision the
user is about to make. If it's not obvious from the conversation or an attached file, ask in one
short question. Don't proceed against a vague or unstated target.

## Step 1 — Assumptions

Surface the hidden assumptions behind the target, including ones treated as too obvious to state.
For each: what has to be true for it to hold, and how confident is that really? Rank by
(likelihood of being wrong) × (damage if wrong). Keep only the ones that are actually load-bearing
— skip anything trivially true.

## Step 2 — Premortem

Assume the target has already failed. Work backwards: state the assumed failure, list the most
likely causes ranked by probability (not just severity), and note the earliest signal that would
have revealed each one in time to act. Where a premortem cause traces back to a Step 1 assumption,
say so explicitly rather than restating it as new.

## Step 3 — Red team

Adopt an adversarial stance. Find failure modes (what input/sequence/condition breaks this
outright), edge cases (what's at the boundaries the design implicitly assumes won't happen), blind
spots (what wasn't considered because it wasn't the focus), and unintended consequences (what this
enables or incentivizes that nobody wanted). Skip anything that's just a restatement of a Step 2
cause — cross-reference instead.

## Step 4 — Second-order effects

State the intended first-order outcome briefly as a baseline. Trace what happens *after* that
outcome lands — how other people, systems, or incentives react to it — following at least one
chain to a third-order effect where it's non-obvious. Flag any second-order effect that undermines
the original goal.

## Step 5 — Synthesize

Produce a single report, not four sections bolted together:

1. **Top risks** — the highest-severity, highest-likelihood findings across all four lenses,
   deduplicated, each tagged with which lens(es) surfaced it.
2. **Assumptions to verify before proceeding** — the load-bearing ones from Step 1 that are still
   unconfirmed.
3. **Effects to watch after launch** — the Step 4 findings that won't show up until after the
   immediate outcome lands.
4. **Recommendation** — one of: proceed as-is / proceed with specific mitigations (name them) /
   rethink before proceeding (name the blocking issue). Don't hedge — pick one and say why.

Be concrete throughout — ground every finding in specifics of the actual target, not generic
risk-management boilerplate.
