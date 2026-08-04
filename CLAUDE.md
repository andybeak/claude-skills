# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A Claude Code **plugin marketplace**: a `.claude-plugin/marketplace.json` catalog listing plugins,
each of which is a directory of `skills/<skill-name>/SKILL.md` files. There is no build, lint, or
test step — everything is markdown (skill instructions) and JSON (plugin/marketplace manifests).
Changes are validated by reading them, not by running a command.

## Structure

```
.claude-plugin/marketplace.json      # catalog: {name, owner, plugins: [{name, source}]}
<plugin-name>/
  .claude-plugin/plugin.json         # {name, version, description}
  skills/
    <skill-name>/
      SKILL.md                       # frontmatter (name, description, ...) + instructions body
```

- **`marketplace.json`** is the root index. Every plugin directory must have a matching entry
  `{ "name": "<plugin-name>", "source": "./<plugin-name>" }` or it isn't discoverable.
- **`plugin.json`** (`name`, `version`, `description`) identifies one plugin. Bump `version` when a
  plugin's skills change meaningfully.
- **`SKILL.md`** frontmatter must include `name` and `description`. The `description` is the only
  thing Claude sees when deciding whether to invoke the skill unprompted, so it must state *what
  the skill does* and *when to trigger it* (explicit trigger phrases, e.g. `"review this PRD"`,
  `/prd-review`) — not just a title. Some third-party skills add extra frontmatter fields
  (`allowed-tools`, `hidden`, `license`, `compatibility`, `when_to_invoke`, `stance`); these are
  optional extensions, not part of the core schema used by this repo's own plugins.

## Existing plugins

- **`plan-manager`** — full plan lifecycle: `plan-review` (pre-execution review), `plan-builder`
  (implement one phase with scope discipline), `plan-audit` (verify a phase was actually built).
- **`prd-manager`** — `prd-create`: interview-and-draft a new PRD, self-reviewed against
  `prd-review` until it's ready to hand off. `prd-review`: critique a PRD across ten dimensions
  (problem framing, scope, testability, non-functional coverage, UX, traceability, plannability for
  a phased-plan handoff, test-pyramid handoff) before it goes to a planning agent.
- **`bug-review`** — `bug-find-latent`: scheduled audit that cross-references specs/contracts
  against the implementation, drafts PR fixes for confirmed bugs, and escalates ambiguous ones.
- **`third-party-skills`** — vendored/reference skills from external sources plus a few personal
  workflow skills (web-perf, humanizer, systematic-debugging, agent-browser,
  resolve-agent-reviews, playwright, defensive-coding, qa-adverserial-review, bot-cursor). Treat
  these as mostly-external content: preserve their original structure/frontmatter conventions
  rather than normalizing them to match this repo's own plugins.

## Conventions when adding or editing a skill

- Advisory/review skills (`plan-review`, `plan-audit`, `prd-review`, `qa-adverserial-review`) are
  explicitly **read-only** — they report findings and never edit the target artifact. Preserve that
  boundary; it's stated in their own descriptions for a reason.
- Skill bodies in this repo's own plugins (`plan-manager`, `prd-manager`, `bug-review`) follow a
  numbered-steps structure (Step 0 locate input → Step N review/act → final step defines the exact
  output format/template). Match that shape for new skills in these plugins rather than inventing a
  new structure per skill.
- A new plugin needs three things together, or it won't be usable: the `skills/<name>/SKILL.md`
  file, a `.claude-plugin/plugin.json` for the plugin, and an entry in the root
  `.claude-plugin/marketplace.json`.
