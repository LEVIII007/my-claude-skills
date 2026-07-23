---
name: no-code-comments
description: User dislikes comments in code, especially in small diffs — default to zero comments; why goes in commit/PR message
metadata:
  type: feedback
---

Do not add explanatory comments to code — no doc comments, rationale blocks, or inline explanations — unless explicitly asked. This holds especially for small, targeted changes (e.g. a one-function bug fix), and even when the change encodes a non-obvious root cause.

**Why:** User has said explicitly, more than once, "remove comments... I do not like comments in code, especially when doing little changes." He finds narrative comments noise and expects code to read clearly on its own.

**How to apply:** Default to zero comments in any code I write, across all languages and repos. Put the "why" in the commit message, PR description, or chat response instead. This is stricter than the general "only comment non-obvious WHY" rule — for this user, prefer no comment at all.
