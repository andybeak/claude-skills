---
name: plan-audit
description: Audit a plan or a specific plan phase against what's actually implemented — trace every requirement to source and tests, and report gaps. Read-only: never edits the plan or the code. Use when the user says "audit plan", "audit phase N of plan", "did we actually finish phase X", "check the plan against what's built", or similar.
---

# Plan Audit

Audit a plan (or one phase of it) by tracing every requirement to source code and tests, and
reporting what's missing or partial. This is a check on work already done, not a review of the
plan itself (that's `plan-review`) and not an implementation pass (that's `plan-builder`).
Advisory only: never edit the plan file, never edit source, never fix what you find — report it.

## Step 0 — Find the plan and scope

Figure out what you're auditing, in this order:

1. A file path or pasted plan the user gave you directly.
2. A plan produced by `EnterPlanMode`/`ExitPlanMode` earlier in this session.
3. The most recently modified plan-shaped file in the repo (e.g. `PLAN.md`, `docs/plans/*.md`).

State which one you're using. If none of these resolve, or the phase to audit is ambiguous, ask
the user rather than guessing.

## Step 1 — Extract requirements

Read the plan (or named phase) in full and list its requirements explicitly before checking
anything. If the plan uses requirement IDs (REQ-1, 5b.3, etc.), preserve them; if not, number them
yourself so the report has stable references.

**Decide how to run the audit:**

- **Small plan/phase** — a handful of requirements: trace them yourself, inline.
- **Large plan/phase** — many requirements, or requirements spanning multiple files/modules: dispatch
  one subagent per requirement (or tight cluster of related requirements) via the Agent tool, all
  launched in a single message so they run in parallel. Give each subagent the requirement text
  with its ID, the project root, and the checks from Step 2 below verbatim. Once all return,
  synthesize into the Step 3 report yourself — a subagent returning malformed or off-topic output
  is a reason to redo that one requirement yourself, not to drop it from the report.

## Step 2 — Trace each requirement

For every requirement:

1. **Source**: locate the file(s)/function(s) that implement it. Name exact locations
   (`file:line` or `file:FuncName`) — "implemented across the codebase" is not acceptable.
2. **Implemented**: YES / PARTIAL / MISSING. A requirement is PARTIAL if it's implemented for some
   inputs/paths but not others, or if it diverges from the plan's specified behavior in a way that
   changes the outcome (a different-but-equivalent implementation approach is still YES — flag the
   divergence as a note, not a gap, unless the plan mandated that specific approach for a stated
   reason). Flag any TODO/FIXME that maps to this requirement as PARTIAL, not YES.
3. **Tests**: name the test file/function that exercises this requirement. A test only counts if it
   would fail were the implementation removed or broken — imports, instantiation, or existence
   checks don't count. Missing or shallow tests are NO, with a one-line reason.
4. **Severity**: for anything not fully YES/YES, mark **blocking** (requirement unmet or unverified
   in a way that affects correctness or the plan's stated goal) or **minor** (a small edge case, a
   thin test, cosmetic divergence).

## Step 3 — Output

```
## Requirement traceability
| ID | Requirement (short) | Source        | Implemented | Tests          | Severity |
|----|----------------------|----------------|-------------|----------------|----------|
| 1  | <paraphrase>         | file.go:Func   | YES         | file_test.go:Test | -     |
| 2  | <paraphrase>         | file.go:Func2  | PARTIAL     | NO             | blocking |

## Overall verdict
<one line: phase complete and ready, or not — with the reason if not>

## Blocking gaps
<numbered list, one line each, or "none">

## Minor gaps
<numbered list, one line each, or "none">

## Summary
- Total requirements: N
- Fully implemented + tested: N
- Blocking gaps: N
- Minor gaps: N
```

Keep it proportionate: a five-requirement phase doesn't need a page of prose. If a requirement is
fully satisfied, one row is enough — don't pad it with explanation.
