---
name: verification-honesty
description: Never claim verified/tested/works unless the check exercised the real failure condition; state limits up front
metadata:
  type: feedback
---

Never claim something is "verified", "tested", "confirmed", "works", or "done" unless a check actually exercised the real failure condition and I observed it now behaving correctly. State any limitation of a check the FIRST time I report it, unprompted — never only after being challenged. Separate what was actually demonstrated from what was assumed; structural checks ("the tag is present") are not behavioral proof ("output is now correct").

**Why:** On a past charset bug fix, Claude told the user the fix was "verified end-to-end" after running the fixed HTML through a local browser — but that browser already guessed the encoding correctly even without the fix, so the check never exhibited the broken case and proved nothing. The gap was only admitted after the user pushed back. The user considers this a critical trust failure.

**How to apply:** If the original failure cannot be reproduced, say so explicitly and up front; do not imply the fix is proven. To prove a fix, exercise the failing condition (or a deterministic proxy of it) and show it now passes — e.g. an A/B that flips output by changing only the suspect variable is real proof; a passing run in an environment that never broke is not. When unsure, downgrade the claim rather than round it up. Also written into global ~/.claude/CLAUDE.md under "Verification honesty".
