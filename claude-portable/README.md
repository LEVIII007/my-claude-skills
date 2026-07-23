# Claude portable profile — Shashank Tyagi

Sanitized coding-style profile to carry to a new machine/company. Contains no
Mindtickle-internal information: no project memories, architecture notes,
ticket plans, transcripts, or internal tool references.

## Contents

- `CLAUDE.md` — global working principles: think-before-coding, simplicity
  first, surgical changes, goal-driven execution, verification honesty,
  no code comments, user-keeps-control.
- `memory/` — four starter memory files (the same rules with their
  origin stories, plus your user context) in Claude Code's memory format,
  with a `MEMORY.md` index.

## Setup on the new machine

1. Install Claude Code, then copy the global file:

   ```
   cp CLAUDE.md ~/.claude/CLAUDE.md
   ```

   This alone ports the core style — it loads in every session, every project.

2. Memory directories are created per-project by Claude Code, so they can't be
   pre-seeded for repos that don't exist yet. In your first session inside a
   new project, say:

   > Read the files in ~/claude-portable/memory/ and save them to your memory.

   Claude will re-ingest them into that project's memory store. Repeat per
   project, or skip it — CLAUDE.md already carries the load-bearing rules.

3. Optional: personal skills. If you want `graphify` / `remotion-best-practices`,
   copy them from the old machine's `~/.claude/skills/` into the new one's,
   and re-add the trigger block to `~/.claude/CLAUDE.md` (the old machine's
   copy has it at the top).

4. Update `memory/user-shashank-context.md` once you settle in — it currently
   says "voice agents at Back2YY Combinators, joined mid-2026".

## Do NOT copy from the old machine

- `~/.claude/projects/` (session transcripts + Mindtickle project memories)
- `~/.claude/history.jsonl`, `plans/`, `tasks/`
- `~/.claude/plugins/` (mindtickle-engg / mindtickle-pm are company IP)
- `~/.claude/settings.json` MCP/permission entries pointing at internal systems
