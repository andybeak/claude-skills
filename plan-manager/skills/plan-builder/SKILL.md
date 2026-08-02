---
name: plan-builder
description: Use this skill when the user asks you to implement a single phase of a multi-phase delivery plan with strict scope discipline, requirement traceability, and an auditable report of what was done. Use this skill whenever the user asks to implement, build, or execute a phase, stage, milestone, or step of an existing plan, spec, design doc, or roadmap — even when they don't explicitly say "phase." Trigger on prompts like "implement phase 3 of the plan," "do step 2," "work on milestone B," "build out the next stage," "read the plan and start phase X," or any request that points at a numbered/named chunk of work in a larger document. Also use it when the user attaches a plan and says "do the next bit" or similar.
---

# Phased Plan Implementer

Implement one phase of a delivery plan rigorously: every requirement traced, scope respected, ambiguities surfaced before code is written rather than guessed past.

The user produces detailed phased plans and hands them to coding agents one phase at a time. The failure modes they want to prevent are: scope drift into adjacent phases, "traceability theater" (claims of coverage without evidence), tests that import but don't exercise behavior, and agents guessing past ambiguity instead of asking.

## When this triggers

The user will name a plan (often a file path, sometimes pasted inline) and a phase identifier. Examples:

- "Read plan.md and implement phase 3"
- "Do phase 5b of the crafting plan"
- "Implement the next phase from the design doc"
- "Build stage 2 from @roadmap.md"

If the phase identifier is missing or ambiguous, ask before starting. Don't guess which phase they meant.

## The contract

Treat this as a contract with the user. Every run produces:

1. **Code** that implements only the named phase
2. **Tests** that exercise each requirement's behaviour (not just import smoke checks)
3. **A final report** (inline in chat — see "Report format" below)

If any of these three can't be produced honestly, stop and say so rather than fabricating.

## Workflow

### 1. Load and parse the plan

Read the plan file in full. Find the named phase and extract its requirements as an explicit list before writing any code. If the plan uses requirement IDs (REQ-1, 5b.3, etc.), preserve them; if not, number them yourself so the traceability table has stable references.

If the phase section is short or vague, re-read surrounding context (prior phases, overall goals, glossary) — sometimes a requirement's meaning lives in an earlier definition.

**Decide how to run the phase before starting:**

- **Small phase** — a handful of requirements confined to one file/module: work through it yourself, inline.
- **Large phase** — many requirements, or requirements spanning multiple files/modules/services: dispatch one subagent per requirement (or per tight cluster of related requirements) via the Agent tool, all launched in a single message so they run in parallel. Give each subagent: the requirement text with its ID, the relevant plan/codebase context, and the same contract this skill follows (implement + test + report file:function). Once all return, verify and reconcile their combined work yourself — check for conflicting edits, duplicate helpers, and gaps between what each subagent assumed — before building the traceability table in Step 6. A phase too big to hold fully in context risks losing track of earlier requirements by the time it reaches the last one; fan-out avoids that.

### 2. Surface ambiguities before coding

For each requirement, ask: "Could a reasonable developer implement this two different ways and both be defensible?" If yes, that's an ambiguity. Also flag:

- Requirements that contradict existing code in the repo
- Requirements that reference things not in the plan or codebase
- Requirements that depend on decisions deferred to earlier phases but not actually made there

**Stop and ask before implementing if any material ambiguity exists.** Don't bundle questions into the final report after the fact — that defeats the point. Trivial wording questions can be noted and resolved with a stated assumption; anything that affects design or interface needs an answer first.

### 3. Implement, scoped strictly to this phase

Implement only what the named phase requires. Do not:

- Modify code outside the phase's scope, even to "improve" it
- Refactor adjacent modules unless the phase explicitly requires it
- Start work on later phases because it "seems efficient"
- Edit the plan itself

If you find a real problem in earlier code or in the plan, note it for the report and keep going on the current phase. Don't fix it silently.

### 4. Write tests that actually exercise the requirements

The default test bar is **unit tests for each requirement, plus integration tests wherever a requirement crosses a module, service, or process boundary**. A test "exercises" a requirement when, if the implementation were removed or broken, the test would fail. Tests that only import the module, instantiate a class, or assert that a function exists do not count.

If the plan specifies a different test standard for a requirement, follow the plan.

### 5. Run all tests

Run the new tests and the existing test suite, plus any lint/typecheck the project runs in CI. All must pass before reporting done. If something is genuinely flaky or pre-existing-broken, say so explicitly in the gaps section — don't quietly skip it.

If the phase changes UI-facing or otherwise runtime behavior, use the `/verify` skill to exercise it end-to-end rather than relying on tests alone — passing tests confirm code correctness, not that the feature actually works as a user would hit it.

### 6. Verify

Trace all requirements to source code.

Verify that all requirements are fully implemented and tested.

Report inline in the final chat message using the format below.

## Report format

Use this structure exactly. It's designed to be scannable by the user and auditable later.

```
## Phase <X> implementation report

### Summary
<2-4 sentences: what was built, anything notable>

### Requirements traceability
| ID  | Requirement (short)       | Files / functions                | Tests                          | Status |
|-----|---------------------------|----------------------------------|--------------------------------|--------|
| 1   | <short paraphrase>        | path/to/file.go:FuncName         | path/to/file_test.go:TestName  | Done   |
| 2   | <short paraphrase>        | ...                              | ...                            | Done   |
| 3   | <short paraphrase>        | ...                              | ...                            | Partial — see gaps |

### Test results
- New tests added: <N> unit, <M> integration
- Full suite: <pass/fail counts>
- Anything skipped or flaky: <list, or "none">

### Gaps
<Anything partial, deferred, or skipped, with rationale. "None" is a valid answer.>

### Questions / assumptions
<Anything that was ambiguous and how it was resolved. Ideally these were asked before coding; only put items here if they were trivial enough to resolve with a stated assumption. "None" is a valid answer.>

### Out-of-scope observations
<Real issues noticed in earlier phases or the plan itself, not acted on. "None" is a valid answer.>
```

The traceability table is the heart of the report. Every requirement in the phase appears as a row; every row points at concrete files/functions and concrete tests by name. "Implemented across the codebase" is not an acceptable cell value — name the files.

## What to avoid

- **Traceability theater**: claiming coverage with vague references like "see auth module" instead of named files and functions. If you can't name the file:function that implements requirement N, you haven't implemented requirement N.
- **Scope creep**: touching code outside the phase. The user explicitly wants this prevented.
- **Silent guessing**: resolving ambiguity in your head and proceeding. Ask first.
- **Test theatre**: writing tests that pass without exercising the behaviour. A test that wouldn't fail if the implementation were deleted is not a test of that requirement.
- **Editing the plan**: if the plan needs to change, say so in "Out-of-scope observations" and let the user decide.

## Notes on the report being inline

The report goes in the final chat message, not in a file in the repo. This is deliberate: it keeps the repo clean and puts the audit trail where the user reviews work (in the conversation). If the user later wants a persisted version, they can ask.
