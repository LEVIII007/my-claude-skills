# Maxim evaluators REST API (cheatsheet)

- Base URL: `https://api.getmaxim.ai` · Auth header: `x-maxim-api-key`. **Use this host, not the SDK path** (`app.getmaxim.ai/api/sdk/...` can only fetch one evaluator by name, cannot list). The SDK is never used.
- Auth: a single **org-scoped** key authenticates every workspace.
- `GET  /v1/workspaces` — enumerate workspaces `{data:[{id,name,accountId,...}]}`. Needs an org key; a workspace-scoped key returns `403 "not allowed to list workspaces"`.
- `GET  /v1/evaluators?workspaceId=<ws>[&folderId=<f>]` — list. Returns `{data:[{id,name,type,builtin,reversed,config}]}`. Match by name for updates. **Built-ins (`builtin:true`) are ignored** — manage `builtin:false` only.
- `POST /v1/evaluators` — create. Body: `{workspaceId, name, type:"AI", config, [description, tags, folderId]}`
- `PUT  /v1/evaluators` — update. Body: `{workspaceId, id, config, [name, description, tags, folderId]}` (no `type`)
- `DELETE /v1/evaluators` — body `{workspaceId, id}`. **Separately permissioned**: standard org key returns `403 "doesn't have permission to DELETE EVALUATORS"`. `--prune` degrades to report-only on 403.

AI `config`: `{gradingStyle, scale:{min,max}, model, provider, instructions, optionalVariables, passFailCriteria:{entryLevel:{name,operator,value}, runLevel:{name,operator,value}}}`.

## Prompts & Datasets

Same base URL and auth as above. Used by the `prompt` and `dataset` CLI commands
(`maxim_evals.cli_prompts`, `maxim_evals.cli_datasets`).

### Prompts

- `GET  /v1/prompts?workspaceId=<ws>[&name=<n>][&folderId=<f>]` — list. Match by
  `name` for create/update lookups (the API does not filter exactly on `name`,
  so the client always filters the returned list client-side).
- `POST /v1/prompts` — create the prompt shell (name/description/folderId only —
  **no model config**). Body: `{workspaceId, name, [description, folderId]}`.
  Response is double-wrapped: `{"data": {"data": {...}}}` — unwrap until `data`
  is no longer a dict/list to reach `{id, name, ...}`.
- `PUT  /v1/prompts` — update the prompt shell (name/description/folderId).
  **Not used by this CLI**: the CLI never renames a prompt or edits its
  description after creation; only new versions are published.
- `POST /v1/prompts/versions` — publish a new **immutable** version. Body:
  `{promptId, description, modelName, modelProvider, messages:[{role,content}],
  modelParameters}`. There is no version-update endpoint — every change is a
  new version. `modelProvider` must be one of the enum below.
- `GET  /v1/prompts/versions?workspaceId=<ws>&promptId=<id>` — list all versions
  of a prompt. Each has a `version` number (integer, monotonically increasing);
  the highest number is the latest. A version's `config` mirrors the create
  body (`messages` entries come back wrapped as `{payload:{role,content}}`).
- **`POST /v1/prompts/deploy` is intentionally never called.** Publishing a
  version does not deploy it to any environment/prompt-config alias; deploy
  remains a manual, human action in the Maxim UI. This is a deliberate scope
  boundary, not an oversight — the CLI's job is version-safe publish, not
  traffic cutover.
- Provider enum (`modelProvider` / spec front-matter `provider`): `openai`,
  `azure`, `huggingface`, `anthropic`, `together`, `google`, `groq`, `bedrock`,
  `maxim`, `cohere`, `ollama`, `xai`, `vertex`, `mistral`, `fireworks`.

### Datasets

- `GET  /v1/datasets?workspaceId=<ws>[&name=<n>]` — list. Returns
  `{data:[{id,name,columns:[{id,name,columnType,meta},...]}]}`. Match by
  `name`; column `id`s only exist on the remote record, never in the local
  spec.
- `POST /v1/datasets` — create. Body: `{workspaceId, name, columns:[{name,
  columnType, meta:{columnDataType, delimiter}}], [createDefaultColumns,
  folderId]}`. **Response body is `{}`** — no id or column ids come back.
  Callers must immediately `GET /v1/datasets?name=<n>` to resolve the new
  dataset's id and per-column ids before appending entries.
- `PUT  /v1/datasets` — update. Body: `{workspaceId, id, name}`. **Name-only**:
  there is no way to add/remove/rename columns via this endpoint — the schema
  is fixed at creation. This CLI does not call `update_dataset` at all; dataset
  "update" only means appending rows via `/v1/datasets/entries`. Detecting a
  columns mismatch between the local spec and the remote dataset is treated as
  a hard error (recreate the dataset to change columns), not something to fix
  via `PUT`.
- `columnType` enum: `CONVERSATION_HISTORY`, `INPUT`, `VARIABLE`,
  `EXPECTED_OUTPUT`, `EXPECTED_TOOL_CALLS`, `TAGS`, `OUTPUT`, `EXPECTED_STEPS`,
  `SCENARIO`.
- `POST /v1/datasets/entries` — append rows. Body: `{workspaceId, datasetId,
  entries}` where **`entries` is a list of cell-arrays**, one array per row:
  `[[{columnId, value:{type:"text", payload:"<cell text>"}}, ...], ...]`. This
  is purely additive — there is no update/delete-entry endpoint, so the CLI
  keeps a local hash mirror (`.maxim/pushed.json` next to the spec) to skip
  rows already pushed and stay idempotent across repeated `dataset update`
  runs.
- `GET  /v1/datasets/entries?workspaceId=<ws>&datasetId=<id>` — list entries.
  Each entry is `{cells:[{columnId, value:{type, payload}}, ...]}`; map
  `columnId` back to a column name via the dataset's `columns` array to
  reconstruct a row.
