---
description: Try to break this. Look for failure modes, edge cases, blind spots and unintended consequences.
argument-hint: "[what to attack, defaults to the current plan/design]"
---

Try to break $ARGUMENTS (or, if empty, the plan/design currently under discussion). Adopt an adversarial stance:

1. Identify failure modes: what input, sequence, or condition makes this fail outright?
2. Identify edge cases: what's at the boundaries that the design implicitly assumes won't happen?
3. Identify blind spots: what did the design not consider because it wasn't the focus?
4. Identify unintended consequences: what does this enable or incentivize that nobody wanted?

Prioritize findings that are realistic and damaging over ones that are merely exotic.
