---
name: maxim-dataset-sync
description: "Create a new dataset or append rows to an existing one on Maxim from a local spec directory. Trigger: 'create a Maxim dataset', 'update dataset', 'add rows to dataset', 'sync this dataset to Maxim'."
---

# Maxim Dataset Sync

You take a local dataset spec (a JSON column manifest + a CSV of rows) and
either create a brand-new dataset in a Maxim workspace or append new rows to
an existing one. Datasets are **append-only and schema-fixed**: rows can be
added but never edited or removed via the API, and columns can't change after
creation. Maxim is the source of truth for what "existing" means — the tool
looks the dataset up by name before doing anything.

## Prerequisites

- A single org-scoped `MAXIM_API_KEY` exported (authenticates all
  workspaces). If missing, the tool fails naming the variable — tell the user
  to export it and stop.

## Workflow

1. **Ask create vs. update, explicitly.** Never infer this from context. The
   CLI also enforces it server-side: `dataset create` on a name that already
   exists fails with "already exists — use 'dataset update' to append rows";
   `dataset update` on a name that doesn't exist fails with "not found — did
   you mean 'dataset create'?".

2. **For create, gather the spec from the user:** the dataset name, its
   columns (each a `name` plus a `type`, one of `CONVERSATION_HISTORY`,
   `INPUT`, `VARIABLE`, `EXPECTED_OUTPUT`, `EXPECTED_TOOL_CALLS`, `TAGS`,
   `OUTPUT`, `EXPECTED_STEPS`, `SCENARIO`), and the initial rows. Write it to
   `./maxim/datasets/<slug>/` in the user's working directory (never under
   the plugin root or `/tmp`) as two files, and **confirm the path** with the
   user before proceeding:

   `dataset.json`:
   ```json
   {
     "name": "testset",
     "folder": "Diagnostics",
     "columns": [
       { "name": "signal", "type": "INPUT", "dataType": "text" },
       { "name": "expected", "type": "EXPECTED_OUTPUT" }
     ]
   }
   ```
   `folder` is optional (places the dataset in a named dataset folder, created
   on `create` if it doesn't exist). `dataType` defaults to `"text"`.

   `rows.csv` — a plain CSV whose header row is exactly the column names:
   ```csv
   signal,expected
   "rep interrupted twice","coaching flag: interruption"
   ```

   For an update, add new rows to the same `rows.csv` (or a re-pulled one) —
   existing rows should stay as-is; do not edit or reorder columns.

3. **Choose environment.** Ask which environment: `staging` (default),
   `experimentation`, or `production`. Never default to production silently.

4. **Run the CLI via the shim:**
   ```bash
   bash "${CLAUDE_PLUGIN_ROOT}/scripts/run-maxim-evals.sh" dataset create --env <env> ./maxim/datasets/<slug>
   bash "${CLAUDE_PLUGIN_ROOT}/scripts/run-maxim-evals.sh" dataset update --env <env> ./maxim/datasets/<slug>
   bash "${CLAUDE_PLUGIN_ROOT}/scripts/run-maxim-evals.sh" dataset pull   --env <env> "<dataset name>" -o ./maxim/datasets/<slug>
   ```
   `create` accepts `--folder "<name>"` / `--folder-id <id>` to place the
   dataset in a folder.

   **Confirming the write (required — read this).** `create` and `update`
   print what they're about to do and, in a real terminal, ask `[y/N]`. But
   when you (the agent) run the CLI through the Bash tool, stdin is **not a
   TTY**: the `[y/N]` prompt can't be answered and the CLI aborts with a
   message telling you to re-run with `--yes`. `--yes` is therefore the normal
   way you invoke these commands — but it means *you* own the human
   confirmation, so:
   - **Before** running `create`/`update`, show the user exactly what will be
     written (dataset name, env, and the new/changed rows) and get an explicit
     go-ahead in the conversation. Do not treat "the user invoked the skill" as
     that go-ahead.
   - For **`--env production`**, get a distinct, explicit confirmation with
     `AskUserQuestion` first (the CLI's production banner is not shown to you
     when stdin isn't a TTY, so it cannot be your safety net — you are).
   - Only after that confirmation, run the command with `--yes` appended.
   - `pull` is read-only and needs no confirmation or `--yes`.

5. **Surface the result to the user:**
   - `CREATE dataset '<name>' (<n> columns, <n> rows)` → confirmed → the
     dataset is created, then its id/column-ids are resolved via a follow-up
     `GET` (create returns `{}` with no ids — see reference below), and the
     initial rows are appended in the same run.
   - `<n> new row(s) to append, <n> already present, skipping.` → confirmed →
     `Appended <n> row(s) to '<name>'.` Rows already pushed in a previous run
     are detected via a content hash and skipped automatically — re-running
     `update` with the same `rows.csv` (or one that's a superset of a prior
     run) is safe and idempotent.
   - `Nothing to append: all <n> row(s) already present.` — every row in
     `rows.csv` was already pushed; nothing was sent.
   - `column drift for '<name>': spec columns [...] != remote columns [...].
     Columns are fixed at create — recreate the dataset to change them.` —
     **hard error.** There is no "add a column" or "rename a column"
     operation; the only way to change columns is to create a new dataset
     (under a new name) with the desired schema.
   - `SPEC ERROR: <reason>` — `dataset.json` is missing/invalid (missing
     `name`/`columns`, unknown `columnType`, malformed JSON) or `rows.csv` is
     missing. Fix the spec and re-run; nothing was sent to Maxim.

## Notes

- **Append-only, idempotent via a local hash mirror.** The CLI writes a hash
  of each pushed row to `<spec_dir>/.maxim/pushed.json` (keyed by dataset id).
  This is what makes `update` safe to re-run: only rows whose hash isn't in
  that mirror yet get sent. If you delete `.maxim/pushed.json`, the next
  `update` will attempt to re-push every row in `rows.csv` (Maxim itself has
  no de-dup, so this would create real duplicates).
- **Column drift is a hard error by design**, not something this tool
  auto-fixes — `PUT /v1/datasets` is name-only and cannot add, remove, or
  retype columns. If the schema needs to change, create a new dataset.
- `pull` reconstructs `dataset.json` + `rows.csv` from the remote dataset and
  rewrites the local `.maxim/pushed.json` mirror to match, so a subsequent
  `update` only pushes rows genuinely new relative to what's on Maxim.
