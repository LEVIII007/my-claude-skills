---
name: maxim-prompt-sync
description: "Create a new prompt or publish a new version of an existing prompt on Maxim from a local spec file. Trigger: 'create a prompt on Maxim', 'publish a new prompt version', 'update Maxim prompt', 'sync this prompt to Maxim'."
---

# Maxim Prompt Sync

You take a local prompt spec (front-matter + `## <role>` message sections) and
either create a brand-new prompt (version 1) or publish a new **immutable**
version of an existing one in a Maxim workspace. Maxim is the source of truth
for what "existing" means — the tool looks the prompt up by name before doing
anything.

## Prerequisites

- A single org-scoped `MAXIM_API_KEY` exported (authenticates all
  workspaces). If missing, the tool fails naming the variable — tell the user
  to export it and stop.

## Workflow

1. **Ask create vs. update, explicitly.** Never infer this from context — a
   typo'd prompt name silently falling into the wrong branch is exactly the
   mistake this guardrail exists to prevent. The CLI also enforces it
   server-side: `prompt create` on a name that already exists fails with
   "already exists — use 'prompt update'"; `prompt update` on a name that
   doesn't exist fails with "not found — did you mean 'prompt create'?".

2. **For create, gather the spec from the user:** model name, provider (must
   be one of `openai, azure, huggingface, anthropic, together, google, groq,
   bedrock, maxim, cohere, ollama, xai, vertex, mistral, fireworks`),
   model parameters (`temperature`, `max_tokens`, `top_p`,
   `frequency_penalty`, `presence_penalty` — all optional), and the message
   sequence (`## System`, `## User`, `## Assistant`, etc.). Write it to
   `./maxim/prompts/<slug>/prompt.md` in the user's working directory (never
   under the plugin root or `/tmp` — this is the user's data) and **confirm
   the path with the user** before proceeding. Example:

   ```markdown
   ---
   name: evidence-extraction
   description: Sales coaching evidence extractor
   model: claude-opus-4-8
   provider: anthropic
   folder: Diagnostics
   temperature: 0.2
   max_tokens: 4096
   version_description: Tighten JSON rules
   ---

   ## System
   You are an evidence extraction specialist.

   ## User
   <signal>{{trait_signal_json}}</signal>
   <traits>{{content}}</traits>
   ```

   `folder` is optional (places the prompt in a named prompt folder, created
   on `create` if it doesn't exist). `version_description` is an optional
   changelog note for the version being published.

   For an update, either edit the same spec file in place (bump the message
   content, model, or parameters) or re-pull it first (step 4) so the diff is
   against the real latest version.

3. **Choose environment.** Ask which environment: `staging` (default),
   `experimentation`, or `production`. Never default to production silently.

4. **Run the CLI via the shim:**
   ```bash
   bash "${CLAUDE_PLUGIN_ROOT}/scripts/run-maxim-evals.sh" prompt create --env <env> ./maxim/prompts/<slug>/prompt.md
   bash "${CLAUDE_PLUGIN_ROOT}/scripts/run-maxim-evals.sh" prompt update --env <env> ./maxim/prompts/<slug>/prompt.md
   bash "${CLAUDE_PLUGIN_ROOT}/scripts/run-maxim-evals.sh" prompt pull   --env <env> "<prompt name>" -o ./maxim/prompts/<slug>/prompt.md
   ```
   `create` accepts `--folder "<name>"` / `--folder-id <id>` to place the
   prompt in a folder (folder placement is create-time only — there is no
   move-on-update). `pull` accepts `--version <n>` to fetch a specific version
   instead of the latest; without it, the highest version number wins.
   **Confirming the write (required — read this).** `create` and `update`
   print what they're about to do and, in a real terminal, ask `[y/N]`. But
   when you (the agent) run the CLI through the Bash tool, stdin is **not a
   TTY**: the `[y/N]` prompt can't be answered and the CLI aborts with a
   message telling you to re-run with `--yes`. `--yes` is therefore the normal
   way you invoke these commands — but it means *you* own the human
   confirmation, so:
   - **Before** running `create`/`update`, show the user exactly what will be
     written (prompt name, env, and what's changing) and get an explicit
     go-ahead in the conversation. Do not treat "the user invoked the skill" as
     that go-ahead.
   - For **`--env production`**, get a distinct, explicit confirmation with
     `AskUserQuestion` first (the CLI's production banner is not shown to you
     when stdin isn't a TTY, so it cannot be your safety net — you are).
   - Only after that confirmation, run the command with `--yes` appended.
   - `pull` is read-only and needs no confirmation or `--yes`.

5. **Surface the result to the user:**
   - `CREATE prompt '<name>' + version 1` → confirmed → `Created prompt
     '<name>' (<id>) with version 1.`
   - `UPDATE prompt '<name>' — publish new version` → confirmed → `Published
     new version of '<name>'.`
   - `UNCHANGED: '<name>' latest version already matches the spec; nothing to
     publish.` — the spec's model/provider/messages/parameters exactly match
     the latest remote version (description-only edits don't count, so they
     won't trigger a new version). This check is best-effort: if Maxim
     returns server-normalized or renamed model parameters, `update` may
     publish a redundant version even though nothing meaningfully changed.
   - `SPEC ERROR: <reason>` — the spec file is malformed (missing
     `name`/`model`/`provider`, unknown provider, no message sections with
     content). Fix the spec and re-run; nothing was sent to Maxim.

## Notes

- **Publish is version-only.** There is no `POST /v1/prompts/deploy` call
  anywhere in this tool — publishing a new version never deploys it to a
  live prompt-config alias or cuts over traffic. Deploying stays a manual,
  deliberate action in the Maxim UI once you're happy with the new version.
- **Versions are immutable.** There is no endpoint to edit an existing
  version — every change is a brand-new version. `update` always publishes a
  new version rather than mutating the latest one; the only way to reduce
  version noise is the `UNCHANGED` short-circuit above.
- **Guardrails are enforced by the CLI, not just this skill**: `create` on an
  existing name and `update` on a missing name both fail loudly rather than
  silently doing the "wrong" operation.
