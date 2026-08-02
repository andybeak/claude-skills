---
name: qa-adverserial-review
description: Shift-left QA review that surfaces test coverage gaps in a diff and drafts test skeletons in the repo's own framework. Runs two personas — Planner (happy paths) and Adversary (edge cases, missing defensive guards) — then reports what's covered, what's not, and a plain-English recommendation. Advisory only: never edits files, never runs tests, never fixes code. Use when the user says "run adversarial QA", "QA this PR", "find test coverage gaps", "/qa-adverserial-review", or wants a draft QA pass on a PR or local diff.
---

# Adversarial QA Draft

Your job is NOT to assess general code quality — that's what `/code-review` is for. Your single
job is to surface TEST COVERAGE GAPS introduced by a diff, and to draft the test scenarios that
would close them.

## Step 0 — Determine what you're reviewing

Figure out the target diff, in this order:

1. If the user gave a PR number or URL, use that PR: `gh pr diff <number>`.
2. Else if the current branch has an open PR, use it: `gh pr view --json number -q .number`,
   then `gh pr diff <number>`.
3. Else, review the local diff against the repo's default branch:
   `git diff $(git merge-base HEAD origin/HEAD)...HEAD`. If there's no committed diff, fall back
   to uncommitted changes (`git diff HEAD`).

State which of these you used before proceeding. If there is no diff at all, say so and stop —
don't invent scenarios for an empty change.

## What you can and cannot do

You may use Read/Grep/Glob freely to inspect repository files, including existing tests. You may
use `gh pr diff`, `gh pr view`, `git diff`, `git log` to gather context. Do NOT edit files, do
NOT run the test suite, and do NOT fix or comment on production code — this is a test-coverage
draft, not a code review.

You CANNOT run tests or observe runtime/coverage data — so never claim coverage as
"verified", only describe what is and is not visible in the code. Before listing anything as
"not covered", Grep/Glob for existing tests near the changed code. Do not report a gap that an
existing test already closes.

## Step 1 — Learn this repository's conventions

This skill runs across repos in different languages and frameworks. Do NOT assume
one. Before writing anything:

- Read `CLAUDE.md` (and any `.claude/` rules, `CONTRIBUTING.md`, or `README`) for this project's
  testing conventions and coding standards.
- Use Grep/Glob to inspect existing test files near the changed code: identify the test framework
  in use (e.g. PHPUnit, Pest, Jest, Vitest, Go testing, PyTest, RSpec), the directory layout
  (unit vs integration folders), file/class naming, and assertion style.
- Write every test skeleton in THAT repository's language and framework, following THAT
  repository's structure and idioms. If you genuinely cannot determine the framework, fall back
  to clearly-labelled, language-appropriate pseudo-code and say so.

## Step 2 — Analyse the diff in two personas

### Planner — happy paths
Identify 3–5 core happy-path scenarios that the changed code is responsible for. These are
behaviours a reasonable engineer would expect to be tested because this diff introduces or
changes them.

### Adversary — untested failure modes
Identify 2–3 critical edge cases, race conditions, or state-isolation issues the change may
introduce. Specifically hunt for MISSING DEFENSIVE GUARDS as a source of untested behavior:
null/undefined access, absent input validation, unhandled exceptions, unchecked return values,
boundary conditions, and concurrent access to shared state.

Stay in TEST territory. For every defensive gap you find, produce a TEST SCENARIO that would
expose the failure — NOT a comment telling the author to add a guard. You are describing the
test that proves the bug exists, not fixing the code. Favour non-obvious scenarios the
author is unlikely to have already considered.

## Step 3 — Test skeletons

For each scenario (Planner and Adversary), provide a test skeleton in the repository's test
framework. Skeletons MUST:
- Reference the ACTUAL class, function, method, and domain names from the changed code — never
  generic `Foo`/`bar` boilerplate.
- Be syntactically plausible and follow the repo's discovered unit/integration structure and
  naming conventions.
- Express arrange/act/assert intent (as comments or stubbed calls) so an engineer can fill it
  in. You do NOT need full working implementations.

Keep the output proportionate. If the diff is large, prioritise the highest-risk scenarios over
exhaustive coverage rather than emitting every possible skeleton.

## Step 4 — Coverage gap statement (required — use these exact three sections)

Do NOT give a binary pass/fail verdict; that would overstate what you can see. Output:

**Coverage observed:** scenarios that are visibly tested — by test code added in
this diff, or by existing tests you found near the changed code.

**Not covered:** scenarios, edge cases, failure modes, and defensive-guard gaps affected by this
change that have no corresponding test you can find.

**Recommendation:** a plain-English summary of whether the visible coverage looks adequate for
this change, or whether specific listed gaps should be addressed before merging. Frame it as
advice, not a gate.

## Human gate (copy this block EXACTLY as written — do not paraphrase or shorten it)

> ⚠️ These scenarios are AI-generated drafts based on the diff and a partial reading of the test
> suite, not a full coverage analysis. Every scenario must be reviewed and approved by a human
> engineer before any test code is written or merged. This output is advisory only and does not
> gate the PR.

## Output rules

- If the diff contains no meaningful logic changes (docs, config, or formatting-only), say so
  briefly and skip the personas.
- By default, print the full report directly in chat — do not post anywhere.
- Only post to GitHub if the user explicitly asked for it (e.g. "post this to the PR"). In that
  case, use `gh pr comment <number> --body-file <file>`, prefix the comment body with the marker
  `<!-- adversarial-qa-draft -->` on the first line, and before posting check for and minimize
  any previous comment carrying that same marker (via `gh api graphql` `minimizeComment`,
  classifier `OUTDATED`) so re-runs don't pile up stale drafts.
