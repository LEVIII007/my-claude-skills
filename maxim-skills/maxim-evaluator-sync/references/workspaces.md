# Maxim workspaces (environments)

| Env | workspaceId | Notes |
|---|---|---|
| staging | cm8pw71p202z4f96z26aab75k | default target |
| experimentation | clyzhb2lk00096keq04xxrvov | safe for smoke tests |
| production | cm8h3xegw06lspa8599dvebue | never default; loud confirm before writes |

A single org-scoped `MAXIM_API_KEY` authenticates all three workspaces; `--env` only changes the target `workspaceId`. With an org key the tool can also discover workspaces at runtime via `GET /v1/workspaces` (the `workspaces` subcommand). Override the workspace with `--workspace-id` for non-mindtickle use.
