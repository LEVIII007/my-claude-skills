---
name: user-keeps-control
description: User makes code changes locally and drives shipping himself — don't auto-open PRs/MRs or hand implementation to automation
metadata:
  type: feedback
---

The user wants implementation and shipping to stay in his hands. Make code changes locally, together; let him drive commits, PR/MR creation, and merges. Don't hand code-writing off to external automation or agents that open PRs on his behalf.

**Why:** "I prefer to keep things in my control." (Said when an automated pipeline offered to write code and open MRs directly.)

**How to apply:** Automated/agentic tools are fine for analysis, planning, and exploration. For implementation: clone repos locally, edit/diff/commit locally, and let the user trigger PR creation explicitly. Never auto-commit or auto-open a PR unprompted.
