---
name: bot-cursor
description: "Sweep all projects for open automated fix/test PRs (bot-authored or Claude-Code-authored, even under a human account), review diffs, and merge the ones you agree with."
trigger: /bot-cursor
---

# /bot-cursor

Sweep all projects for open automated fix/test PRs and merge the ones you agree with (CI is ignored).

Automation in this org opens PRs two ways:
- Under a real bot account (e.g. `app/cursor`).
- Under **andybeak's own GitHub account**, via a local Claude Code automation that commits and pushes with his identity. These are NOT distinguishable by PR author login — they must be detected from commit metadata (see step 1b).

**Scope rule:** this skill handles **fixes and tests only**. Feature work (`feat:`/`feature/`) is always out of scope, even if automation-authored — never touch it. Genuine hand-written fixes by andybeak (no automation signal at all) are also out of scope — leave those for him to manage himself.

## Setup

- **Projects:** epic_whispers (`~/projects/epic_whispers`), chaptered (`~/projects/chaptered`), seaworthy (`~/projects/seaworthy`), martialcrm (`~/projects/martialcrm`)

## Steps

### 1. Sweep all projects for open automated fix/test PRs

For each project (`epic_whispers`, `chaptered`, `seaworthy`, `martialcrm`):

```bash
gh pr list --repo andybeak/<project> --state open --json number,title,headRefName,author --limit 50
```

#### 1a. Shape filter — title and branch

Filter to PRs that meet **both** criteria:
1. **Title matches** (case-insensitive): starts with `fix:`, `fix(`, `test:`, or `test(` — or contains the word "fix" or "test"
2. **Branch matches** (head ref): starts with `cursor/`, `fix/`, `test/`, `bot/`, `autofix/`, or `app/cursor`

Immediately discard anything that looks like a feature, regardless of the above: title starts with `feat:`/`feat(`, or branch starts with `feature/`/`feat/`. Feature branches are never in scope for this skill, even if automated.

#### 1b. Automation filter — who/what actually authored it

A PR only proceeds to review if it shows a genuine automation signal:

- **Bot account:** `gh pr list`'s `author.is_bot` is `true` (e.g. `app/cursor`). Proceed straight to review.
- **Human account, automated commit:** `author.is_bot` is `false` (e.g. `andybeak`). Do NOT assume this is manual work — check the actual commit authorship:
  ```bash
  gh pr view <number> --repo andybeak/<project> --json commits \
    --jq '.commits[].authors[].email'
  ```
  If **any** commit lists `noreply@anthropic.com` as an author/co-author, this is a Claude Code automation run under andybeak's identity — treat it exactly like a bot PR and proceed to review.
  If no commit shows `noreply@anthropic.com` (or any other known automation signature) — this is genuine hand-written work. **Do not review or touch it.** Leave it for andybeak.

When in doubt (new automation identity, ambiguous signal), skim `gh pr view <number> --json body` — automated "recurring audit" PRs have a distinctive generated-report structure (numbered findings, "Trigger:"/"Fix:" pairs, a "Test plan" checklist). A PR that reads like a first-person debugging narrative (e.g. references to what "I" tried, offhand asides about what went wrong while working) is a human writing, not automation — skip it even if it also happens to carry a Claude co-author trailer from an assisted commit.

For each PR that passes both filters:
1. Review the diff: `gh pr diff <number> --repo andybeak/<project>`
2. **If you agree with the fix/test:**
   - Mark ready if draft: `gh pr ready <number> --repo andybeak/<project>`
   - Merge: `gh pr merge <number> --repo andybeak/<project> --squash --delete-branch`
   - Record: "Merged PR #N (<project>) — <one-line summary>"
3. **If you disagree or the change is risky:**
   - Do NOT merge.
   - Record: "Skipped PR #N (<project>) — <reason>"

### 2. Report

Print a concise summary of every PR reviewed: merged and skipped (with reason). Separately note any PRs that were filtered out at step 1b as "genuine manual work, left alone" so andybeak knows the sweep saw them and deliberately didn't touch them.
