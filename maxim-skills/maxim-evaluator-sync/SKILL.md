---
name: maxim-evaluator-sync
description: "Create or update Maxim LLM-judge (type AI) evaluators from a prompt's evaluator designs; runs after evaluator-prompt-creator. Trigger: 'sync evaluators to Maxim', 'push evaluators to Maxim', 'create Maxim evaluators from this prompt', 'update Maxim evaluators'."
---

# Maxim Evaluator Sync

You take a prompt's local evaluator designs and materialize them as **LLM-judge (type AI)** evaluators in a Maxim workspace, creating new ones and updating existing ones idempotently. Maxim is the source of truth: always pull first, show a diff, and confirm before writing.

## Prerequisites

- A single org-scoped `MAXIM_API_KEY` exported (authenticates all workspaces). If missing, the tool fails naming the variable — tell the user to export it and stop. A `403 "not allowed to list workspaces"` means the key is workspace-scoped, not org-scoped.
- Evaluator designs exist as `eval_<name>.md` files in a directory (output of `evaluator-prompt-creator`). If they do not exist, first run `evaluator-planner` then `evaluator-prompt-creator` (support both a `prompt_prd.md` or a minimal intent derived from the prompt).

## Workflow

1. **Locate designs.** Find the `evaluators/<prompt-name>/` directory holding `eval_*.md` files. Confirm the path with the user.
2. **Choose environment.** Ask which environment: `staging` (default), `experimentation`, or `production`. Never default to production.
3. **Plan (dry run).** Run:
   ```bash
   bash "${CLAUDE_PLUGIN_ROOT}/scripts/run-maxim-evals.sh" plan --env <env> <design_dir>
   ```
   Show the user the CREATE/UPDATE/UNCHANGED/ORPHAN plan and any `SKIPPED (non-AI, out of scope)` notices. Deterministic (non-AI) designs are intentionally not synced.
4. **Review the translation.** If any evaluator's mapping looks wrong (grading style, scale, thresholds, provider/model), edit the `eval_<name>.md` front-matter (`scale_min/scale_max`, `pass_threshold`, `run_threshold`, `provider`) and re-run `plan`. See `references/translation-mapping.md`.
5. **Sync.** Once the plan looks right:
   ```bash
   bash "${CLAUDE_PLUGIN_ROOT}/scripts/run-maxim-evals.sh" sync --env <env> <design_dir>
   ```
   **Confirming the write (required).** `sync` shows the diff and, in a real terminal, asks `[y/N]`. But when you (the agent) run the CLI through the Bash tool, stdin is **not a TTY**: the prompt can't be answered and the CLI aborts telling you to re-run with `--yes`. So `--yes` is the normal way you invoke `sync` — but it means *you* own the human confirmation. Before running it, show the user the plan and get an explicit go-ahead in the conversation; for `--env production`, get a distinct `AskUserQuestion` confirmation first (the production banner isn't shown to you without a TTY, so it can't be your safety net — you are). Only then run `sync` with `--yes`. (`plan` is read-only — no confirmation or `--yes` needed.)
6. **Report.** Relay the created/updated/failed counts to the user. On any failure, surface the Maxim error body.

## Notes

- Update matches remote evaluators **by name** within the workspace and carries the Maxim `id` into the update — no manual id handling. Built-in evaluators are never touched.
- Orphans (in Maxim, not in the designs) are reported by default. `--prune` deletes them only with a delete-capable key; a standard org key gets `403` and `--prune` degrades to report-only.
- `pull` refreshes a local mirror at `<design_dir>/.maxim/remote.json` for reference. `workspaces` lists workspaces the key can see (needs an org key).
- Reconciliation compares evaluator **config only** (grading style, scale, thresholds, model/provider, instructions, optional variables). Editing only `description` or `tags` in a design after creation will show as UNCHANGED and won't re-sync — change the config or recreate the evaluator to update them.
- **Folders:** pass `--folder "<name>"` to `plan`/`sync` to place the evaluators in a named evaluator folder (created on `sync` if it doesn't exist; `--folder-id <id>` targets one by id). Folder placement *is* reconciled — an evaluator in the wrong folder is moved via UPDATE. Without `--folder`/`--folder-id`, folder placement is left untouched (evaluators land in the workspace root, and manually-foldered ones are never moved back).
