---
name: minion-plan-and-develop
description: Implement a Jira story end-to-end from grooming to shipped PRs. Trigger: 'plan and develop', 'take this to PR', 'implement this ticket', 'ship this ticket'. Requires Functional Grooming section in Jira parent — if missing, use groom-story-functionally first. Not for one-off edits.
---

# Minion: Plan and Develop

Each phase has a fixed goal, tool, and user-approval gate. Phase outputs persist as markdown files — files (not conversation) are the source of truth between phases, so corrections survive session boundaries and gates catch mistakes early.

**Capability exclusions:** Do not invoke `debugiq` or `JIRA shashtra` via `mcp__enggxmcp__`. For JIRA/Confluence, call `mcp__atlassian__*` directly (not `atlassian-ops`). The `mindtickle-help` Compass capability is allowed for feature/terminology lookups.

## Prerequisites

**JIRA key:**
- Bare ID (`ENGG-1234`) → use as-is.
- URL → extract trailing key.
- Empty/ambiguous → ask. Don't guess.

**Functional Grooming section:** the parent story description must already contain a Functional Grooming section bounded by the markers `BEGIN: groom-story-functionally` and `END: groom-story-functionally` (written by the `groom-story-functionally` skill into the parent description itself — there is no separate sub-task). Match on the **marker text**, not a fixed heading level — the markers render as headings (commonly `###`, e.g. `### BEGIN: groom-story-functionally`) and the level can vary. If the markers are missing, stop and tell the user to run `groom-story-functionally` first — the parent description stays the single source of truth for grooming.

**MCPs:** `mcp__atlassian__*` and `mcp__enggxmcp__*` must be available. If missing:

```bash
claude mcp add --transport http enggxmcp https://enggx-mcp.mt-bots.mindtickle.com/sse
claude mcp add --transport http atlassian https://mcp.atlassian.com/v1/mcp
```

On Claude.ai: Settings → Connectors → enable **Atlassian MCP** + **MindTickle Compass**, then reload.

## Workspace

Per-story workspace at `story-pipeline/<KEY>/` in the CWD. Persist each phase to a markdown file before opening the gate; later phases read it.

| File | Phase |
|---|---|
| `phase-1.md` | Parent description's Functional Grooming section |
| `phase-2.md` | Categorized touchpoints |
| `phase-3.md` | FINAL PLAN |
| `phase-4.md` | Confluence URL + subtask keys |
| `phase-5/<SUBTASK-KEY>.md` | Per-subtask PR link + notes |
| `cost.md` | Token + cost ledger across all phases |

`mkdir -p story-pipeline/<KEY>` on first phase. Offer to add `story-pipeline/` to `.gitignore` if the CWD is a git repo and the entry isn't there.

---

## Cost tracking

Two cost sources, different telemetry. Atlassian calls = no LLM cost.

- **Compass [exact]:** every `execute_capability_tool` response carries `usage` +
  `cost`. Print immediately after each call:
  `Compass: <in>↓ / <out>↑ tokens, $<cost>`
- **Claude:** how you get this depends on environment (below). Prefer EXACT; the
  byte estimate is a last resort.

**Per-phase Compass close** (after gate resolves `yes`/`no`/`pause`, not `edit`; skip if
no Compass call — Phases 1 & 4 have none):
`Phase N close — Compass [exact]: <in>↓ / <out>↑ tokens, $<cost>`
Phase 5: print per subtask, then a Phase 5 aggregate after the last.

### Claude cost — EXACT (Claude Code; the default)

In Claude Code the real per-turn usage is on disk in the session transcript —
including the cache tiers — so **do not estimate**. Either tell the user to run
`/cost`, or compute it yourself from the transcript:

```bash
python3 ${CLAUDE_PLUGIN_ROOT}/skills/minion-plan-and-develop/references/cost-from-transcript.py \
  ~/.claude/projects/<project-slug>/<session-id>.jsonl
```
(`project-slug` = CWD path with `/`→`-`; if unsure, pick the most recently modified
`*.jsonl` there.) The script dedups by message id and applies tiered rates:
fresh input ×1, **cache write ×1.25, cache read ×0.10**, output ×1 — the cache split
is the dominant cost driver, which is exactly what a byte estimate cannot see.

```python
RATES = {  # $/Mtok — VERIFY against current published pricing
  "claude-opus-4-8":   {"in": 15.0, "out": 75.0},
  "claude-opus-4-7":   {"in": 15.0, "out": 75.0},
  "claude-sonnet-4-6": {"in": 3.0,  "out": 15.0},
  "claude-sonnet-4-5": {"in": 3.0,  "out": 15.0},
  "claude-haiku-4-5":  {"in": 0.8,  "out": 4.0},
}
active_model = active_model.split("[")[0]                  # strip "[1m]" etc.
rate = RATES.get(active_model, RATES["claude-opus-4-8"])   # safe default, never KeyError
```
Caveat: any single request >200K tokens is billed at premium long-context rates —
the script flags such turns; apply the per-model premium multiplier if your run crosses it.

### Claude cost — APPROX (Claude.ai web only; no filesystem, no `/cost`)

Last resort, ~±50% — the cache-hit rate, the dominant variable, is unobservable here. Maintain `bytes_in` (tool-result + user-msg), `bytes_out` (assistant-text + tool-arg), `turn_count`, `active_model` across the run, then:

```python
content_tokens_in = bytes_in  / 3.7        # ~3.7 chars/token
output_tokens     = bytes_out / 3.7
multiplier = (turn_count + 1) / 2 * 0.4    # history compounding × cache discount (0.4 ∈ [0.2,0.6] → ±50%)
effective_input_tokens = content_tokens_in * multiplier
rate = RATES.get(active_model.split("[")[0], RATES["claude-opus-4-8"])
est_claude_cost = effective_input_tokens * rate["in"]/1e6 + output_tokens * rate["out"]/1e6
```

### Ledger

`cost.md`: append every Compass call (exact) + one Claude row at end-of-run.
Columns: phase, subtask, source, in-tokens, out-tokens, cost, kind
(`exact` from transcript/`/cost`, or `approx ±50%` from the byte formula).

### Run-summary comment

Auto-posted on parent after the last Phase 5 subtask ships, via `addCommentToJiraIssue`.
Use whichever Claude path you used:
```
Compass [exact]:  $<X.XX>  (<N> calls)
Claude  [<exact|approx>]: $<Y.YY> (model: <active_model>)
Total:            $<X+Y>
```
- EXACT: `Claude [exact]: $<Y.YY>` — from the session transcript (cache-tiered) or `/cost`.
- APPROX (Claude.ai web): `Claude [approx]: ~$<Y.YY> (±50%)` + note: *approximated from
  bytes × history compounding × cache discount; ±50% reflects the unobservable cache-hit
  rate. No `/cost` equivalent on Claude.ai.*

---

## Phase 1 — Pull Functional Grooming

**Goal:** Capture the parent ticket's Functional Grooming section as `phase-1.md`. Don't re-derive — the parent description is the source of truth.

**Tool:** `mcp__atlassian__*` only — no Compass, no code reads.

### Run

1. `mcp__atlassian__getJiraIssue` for `<KEY>` with `fields: ["*all"]` and `expand: "comment,renderedFields"` → summary, status, full description, AC, attachments, comments, remote links.
2. Locate the Functional Grooming section in the description, bounded by the markers (match on the marker **text**, tolerant of heading level — they render as headings, commonly `###`):
   - `### BEGIN: groom-story-functionally`
   - `### END: groom-story-functionally`

   Extract the content between them — it contains the **Problem framing** and **Solution design (requirements)** subsections (themselves rendered as sub-headings, e.g. `### Problem framing`, with `#### Behavior` / `#### Constraints` / `#### Acceptance criteria` beneath).
3. Optional: `mcp__atlassian__getConfluencePage` for any linked Confluence pages so `phase-1.md` is self-contained.

If the markers are missing, **stop**:

> "I couldn't find a Functional Grooming section in `<KEY>`'s description (markers `BEGIN: groom-story-functionally` … `END: groom-story-functionally`). Run the `groom-story-functionally` skill first — it writes the section into the parent description — then come back."

### Persist

`story-pipeline/<KEY>/phase-1.md`:

- **Parent:** `<KEY>` — `<summary>` — `<status>`
- **Grooming source:** parent description, between `groom-story-functionally` markers (no separate sub-task).
- **Requirement summary** — Problem framing + Solution design from the marked grooming section verbatim (light markdown cleanup OK; no paraphrasing or trimming).
- **Acceptance criteria** — from the Solution design's AC subsection if present.
- **Linked artifacts** — Confluence/Figma/etc. from the parent (description + remote links).
- **Open questions / caveats** — anything the grooming flagged unresolved. Phase 2 must respect these.

### Gate

> Phase 1 saved to `story-pipeline/<KEY>/phase-1.md`. Map touchpoints next? (yes / edit / no)

- **yes** → Phase 2.
- **edit** → corrections to `phase-1.md`, re-show, re-prompt.
- **no** → stop. Grooming itself wrong/stale? Fix it via `groom-story-functionally` (which rewrites the parent description between markers), then re-run Phase 1. Don't patch `phase-1.md` to override JIRA.

---

## Phase 2 — Find touchpoints

**Goal:** Map *where* in the codebase this story touches — both **direct** touchpoints (code that must change) and the **indirect blast radius** (code that does *not* change but can break or needs coordination) — categorized, breadth-first, across **all** repos. **Map, not deep-dive.**

**Tool:** Compass via `mcp__enggxmcp__` (code-search capability). See `${CLAUDE_PLUGIN_ROOT}/skills/minion-plan-and-develop/references/compass-how-to.md`.

**Hard rule:** No file reads, no deep call-graph tracing, no logic analysis. Identify *locations* grouped by concern — including second-order locations (who consumes or depends on the changed surface), which Compass surfaces well via cross-repo search. Whether each indirect hit *actually* breaks is Phase 3's job to confirm; Phase 2 just makes sure none are missed.

### Run

A **single** `mcp__enggxmcp__execute_capability_tool` call — Compass groups globally in one pass; multiple calls force manual stitching and produce overlap.

> Using this Phase 1 understanding of `<KEY>`:
>
> `<phase-1 summary, pasted inline — Compass has no filesystem access>`
>
> Find touchpoints in the MindTickle codebase, across **all repos** (not just the obvious one). Cover two layers:
>
> **1. Direct touchpoints** — code that must change. Group into categories: Backend API, Service/business logic, Data model/persistence/migrations, Frontend views/components (note DL — Design Library — component usage at call sites if identifiable, for blast-radius awareness only; do not read the DL repo or pick components here — that's Phase 3), Integrations/events/workers, Tests/fixtures, Config/feature flags, **Build/runtime/deploy** (Dockerfile, CI/build config, dependency manifests, startup scripts, image config, **Helm `values-*`** in `devops-helm-chart`). New external service/client → include `devops-helm-chart` and note the transport (**gRPC vs HTTP**); the two use different host/port keys.
>
> **2. Indirect blast radius** — code that does NOT change but could break or needs coordination because of this change. Explicitly hunt for each of:
> - **Downstream consumers** — services, jobs, schedulers, events, or UIs that read/consume the changed API, data, event, or contract.
> - **Cross-repo dependencies** — other repos that import or depend on the changed module, shared library, proto/IDL, or published contract.
> - **Shared configs** — config, feature flags, or env shared across services that this change reads or mutates.
> - **Permissions / access control** — RBAC, ACL, scopes, or auth policies that gate the changed surface.
> - **Read-only paths** — call sites that only read the changed entity/field and may break on a shape or semantic change.
> - **Derived flows** — aggregations, reports, caches, or derived computations downstream of the changed data.
>
> For each touchpoint: repo/service, file(s) or module(s), a `direct` / `indirect` tag, and a one-line "why this is in scope". For `indirect` hits, the "why" must name the coupling (e.g. "consumes the `foo` event", "imports the shared `auth` lib"). Do NOT open files for implementation details.

### Persist

`story-pipeline/<KEY>/phase-2.md` — two sections, *Direct touchpoints* and *Indirect blast radius*, each touchpoint tagged `direct` / `indirect`. Phase 3 fans out from this; every `indirect` hit becomes an explicit "confirm impact" item, never a deploy-time surprise.

### Gate

> Touchpoints at `story-pipeline/<KEY>/phase-2.md`. Build the implementation plan next? (yes / edit / no)

- **yes** → Phase 3.
- **edit** → rephrase the Compass call (better query, tighter scope), rewrite `phase-2.md`, re-prompt.
- **no** → stop.

---

## Phase 3 — Implementation plan

**Goal:** A merged **FINAL PLAN** organized by **deliverable** (a slice that becomes one PR/subtask).

**Tool:** Compass via `mcp__enggxmcp__` — `code-analyst` primary, `code-search` if needed. No other agents.

**Key move:** parallel fan-out, then merge. Dispatch one `mcp__enggxmcp__execute_capability_tool` per touchpoint category from `phase-2.md`. Each goes deep on its category. Then merge.

**Grouping heuristic:** if total touchpoints ≤ 4 files, collapse into one call. Larger changes → one call per category. Err toward more parallelism for bigger changes.

Per-category prompt:

> Story `<KEY>`: `<one-sentence goal from phase-1.md>`.
> Category: `<name>`.
> Touchpoints from Phase 2: `<file/module list, copied from phase-2.md>`.
>
> Deep-analyze. For each **direct** touchpoint: current behavior, what changes for this story, smallest viable change, risks/cross-cutting concerns other categories must account for.
>
> For each **indirect / blast-radius** touchpoint (tagged in phase-2.md): confirm whether this change actually breaks it. Does the consumer rely on the changed shape/semantics? Is a backward-compatible contract, coordinated change, or merge order required? If it's a false positive, say "no impact" with the reason — don't silently drop it.
>
> Then run the **Build & runtime checklist** (below) for this category — Dockerfile, runtime deps, build/boot compatibility, image build, env/runtime config, `devops-helm-chart` external-service config, protocol-correct host/port (gRPC vs HTTP keys), and `values-*` parity across all four env files. Answer each "no impact (reason)" or "change required (exact files + change)". The bar is the service's **image build and boot**, not local compilation.
>
> Anchor "current behavior" to the repo's **default branch (`main`)** — the branch the work branch is cut from.

### Merge

Reconcile cross-cutting concerns (Backend API shape change → Frontend needs to know; changed event → downstream consumers in other repos need to know). Organize by **deliverable**, not category. Each deliverable:

- Title (action-oriented — becomes subtask title)
- Scope (files/modules)
- What changes and why (tied to business goal)
- Dependencies on other deliverables
- **Cross-repo / downstream impact** — repos, consumers, or derived flows (from *Indirect blast radius*) needing lockstep changes, backward-compat, or notice; note merge order. "none" only after the indirect hits were cleared.
- **Build & runtime impact** — result of the checklist below: Dockerfile/image, manifest, build-config, startup, or env/runtime-config changes this deliverable needs, with exact files (they ship *with* the deliverable). "none" only after the checklist was actively run.
- **Frontend conventions** (frontend deliverables) — decide here, not at code time: which **DL components** replace hand-rolled elements, which **`tokens.*` / `textStyles.*`**, new **`defineMessages` keys**, and **Mixpanel events** to emit. Look components up in the design-library repo (**https://gitlab.com/mindtickle/design-library**); see *MindTickle Frontend Code Review Guidelines*.
- Testing notes (unit/integration coverage — include verifying the confirmed downstream consumers)
- Risks / flags
- Branching — cut a work branch from the repo's default branch, named with one of the standard prefixes: `feature/` (new functionality), `bugfix/` (bug fix), or `hotfix/` (urgent production fix). **Target branch by repo type: backend repos → `dev`; frontend repos → `track/integration`.** Record the repo type, prefix, source, and target in `phase-3.md` so they carry forward to Phase 4 subtasks and Phase 5 MR creation.

### Build & runtime validation (mandatory checklist)

Run every code-touching deliverable against this checklist in the per-category analysis; record results in its *Build & runtime impact* field. Each item: "no impact (reason)" or "change required (exact files + change)" — never blank. The bar is the service's **image build and boot**, not local compilation — late-stage build/boot failures are expensive.

- **Dockerfile impact** — new/updated base image, build stage, system package, `COPY`/`ADD` path, `WORKDIR`, build arg, exposed port, or asset copied into the image?
- **Runtime dependencies** — new/upgraded library in the manifest (`go.mod`, `package.json`, `requirements.txt`, `pom.xml`, …)? Actually installed/vendored in the image, not just locally?
- **Startup / build compatibility** — still compiles and builds on the **target** runtime? Any language/runtime bump, breaking flag, or codegen step the build must run?
- **Service boot validation** — still starts? New required env var, config, migration, or external connection (DB/cache/queue/secret) needed at boot, else crash-loop?
- **Image build requirements** — multi-stage changes, asset compilation, CI stages, cache keys, or build-time secrets/args the pipeline must supply?
- **Env / runtime config dependencies** — new env vars, secrets, configmaps, flags, or service-discovery entries needed at runtime; where set per env and who provisions them?
- **External service/client config in `devops-helm-chart` (CRITICAL)** — integrating a new external service/client? Verify its client/host/endpoint config exists in `devops-helm-chart`. If missing: name the keys and plan a `devops-helm-chart` deliverable (own repo → own subtask/MR) so Phase 5 raises it — never assume it's there or defer it.
- **Protocol-correct host/port** — gRPC client → gRPC host/port keys; non-gRPC/HTTP client → standard host/port keys. A mismatch hits the wrong listener and fails at **runtime**. State each client's protocol and the key it wires to.
- **Helm `values-*` parity (MANDATORY)** — any `values-*` change must appear in **all four** env files (`values-integration.yaml`, `values-staging.yaml`, `values-prod.yaml`, `values-prod-us.yaml`), or the missing env crash-loops. Values may differ per env; **the key must exist in all four**. Hard gate.

Any "change required" ships **with** the code change in the same MR — not a follow-up. Cross-repo changes (e.g. `devops-helm-chart`) go under *Cross-repo / downstream impact* with merge order.

### Persist

`story-pipeline/<KEY>/phase-3.md`. Phase 4 turns this into Confluence + subtasks.

### Gate

> FINAL PLAN at `story-pipeline/<KEY>/phase-3.md`, organized by deliverable. Create Confluence + subtasks next? (yes / edit / no)

- **yes** → Phase 4.
- **edit** → corrections to `phase-3.md`, re-show, re-prompt. Wrong touchpoints → loop back to Phase 2; wrong business intent → loop back to Phase 1.
- **no** → stop.

---

## Phase 4 — Confluence page + JIRA subtasks

**Goal:** Persist the approved plan as a Confluence page and create one JIRA subtask per deliverable.

**Tool:** `mcp__atlassian__*`.

**Confluence target:** Space `DevEx`, parent folder `5568659474` (https://mindtickle.atlassian.net/wiki/spaces/DevEx/folder/5568659474). Pass folder ID as `parent` to `mcp__atlassian__createConfluencePage`.

**Page content:** mirror `phase-3.md` — business goal, touchpoints by category, deliverables in order, risks, testing strategy. Link back to the parent JIRA.

**Subtasks:** `mcp__atlassian__createJiraIssue` with `issuetype: Sub-task`, `parent: <parent key>`. Rules:

- **One feature per subtask.** Group by feature slice, not file type/layer.
- **One repo per subtask.** Multi-repo deliverables split by repo, with cross-repo dependency captured.
- **Same repo + same feature collapses to one subtask/PR.** Don't fan out same-repo work — review churn and merge-order pain.

Each subtask description: scope + what/why, target repo, Confluence link, sibling-subtask dependencies, testable acceptance criteria.

**Label parent `AI_Planned`.** Once the Confluence page exists and subtasks are created, add the `AI_Planned` label to the parent `<KEY>` via `mcp__atlassian__editJiraIssue` (preserve existing labels — read first, append, write back). This marks the story as having a Claude-authored plan attached to JIRA. Idempotent: skip if already present.

### Persist

`story-pipeline/<KEY>/phase-4.md` — Confluence URL, subtask keys + titles, dependency graph, parent label state (`AI_Planned` added / already present).

### Gate

> Confluence + subtasks created, recorded in `story-pipeline/<KEY>/phase-4.md`. Start development next? (yes / edit / no)

- **yes** → Phase 5.
- **edit** → `mcp__atlassian__updateConfluencePage` / `mcp__atlassian__editJiraIssue`, update `phase-4.md`, re-prompt.
- **no** → stop.

---

## Phase 5 — Development

**Goal:** Ship each subtask as a PR linked back to its JIRA subtask.

**Tool:** Compass via `mcp__enggxmcp__` (`mcp__claude_ai_mindtickle_engineering__` is the same server on Claude.ai). **Compass writes the code and opens the MR** — don't code locally, don't open MRs via `git`/`glab`/`gh`.

**Compass invocation:** call `execute_capability_tool` with `compass.query` directly — skip `explore_capabilities_tool` (it describes `compass.query` as Q&A only). A directive prompt routes internally to `code-analyst` + `gitlab-ops`; verify `experts_used` includes `"gitlab-ops"`. Reuse `session_id` from Phase 2/3 if available — Compass already has the file context. **This step is long-running** (Compass writes code + opens an MR) — follow the submit→poll→wakeup protocol in `references/compass-how-to.md`; a `still_processing` response is not a failure.

### Per-subtask loop

For each subtask in dependency order (`phase-4.md`):

1. **Invoke `compass.query` as a directive** (not a question). Include:
   - Business goal (from `phase-1.md`)
   - Every file to change with the exact change (from `phase-3.md`) — include exact code where the plan has it
   - **Build & runtime changes** from *Build & runtime impact* in `phase-3.md` (Dockerfile, manifests, build/CI config, startup, env config) — apply in the **same MR** as the code; shipping without them breaks the image build or boot. Don't defer. The Phase 3 hard gates carry over verbatim: `values-*` changes apply to **all four** env files (`values-integration`, `values-staging`, `values-prod`, `values-prod-us`) — per-env values may differ, the key must exist in all four; external config flagged **missing** in Phase 3 → its own `devops-helm-chart` MR, ordered ahead of the consuming service's deploy; wire each client with **protocol-correct host/port** keys (gRPC → gRPC keys, HTTP → standard keys), or it fails at runtime.
   - **Frontend conventions** (frontend repos) — instruct Compass to **use its `gitlab-ops` expert to read the DL component definitions from the design-library repo (https://gitlab.com/mindtickle/design-library) and use those DL components** rather than hand-rolling, anchored to this deliverable's Phase 3 *Frontend conventions* choices (the DL repo is the single source of what exists and how it is imported — see the *MindTickle Frontend Code Review Guidelines* section). **Audit the *exact code* you pass from `phase-3.md` first** — it must already use DL components (no `styled.button`, no hardcoded box-shadow/radius/px). Handing Compass non-compliant example code to copy — not Compass improvising — is the usual source of drift. Then instruct Compass to **self-verify the frontend code against the DL repo and fix it in place *before* opening the MR** — catching it pre-open is far cheaper than the post-open backstop (step 2).
   - MR metadata: branch `<prefix>/<SUBTASK-KEY>-<slug>` where `<prefix>` is `feature`, `bugfix`, or `hotfix` (taken from the deliverable's branching in `phase-3.md`), cut from the repo's `main` branch. **MR target branch by repo type: backend repos → `dev`; frontend repos → `track/integration`.** Description links subtask + Confluence.
     - **MR title format (mandatory):** `[<SUBTASK-KEY>]: type(scope): <description>` — the Jira key MUST be wrapped in **square brackets**, never parentheses. `[PCUTS-124]: feat(api): add foo` is correct; `feat(PCUTS-124): add foo` is **wrong** (Jira Cloud smart-commit key detection requires square brackets and will reject the parenthesised form). This matches the `raise-mr` title convention.
     - **Commit message format (mandatory):** first line `type(scope): <description>`, then a standalone line containing `[<SUBTASK-KEY>]` as a bare token so Jira parses it as a smart-commit reference. Do NOT put the Jira key in the conventional-commit scope position (`feat(PCUTS-124):` is wrong — the `(scope)` slot is for the package/module, not the ticket).
     - **Fallback:** if the commit cannot carry a standalone `[<SUBTASK-KEY>]` line, include `[<SUBTASK-KEY>]` as a bare token in the MR description body — Jira also scans the MR description for smart-commit references.
   - Closing instruction: *"Please write the code changes and open the MR."*
   - `session_id` from prior phases if available
   - Verify: `experts_used` includes `"gitlab-ops"`. If only `"code-analyst"`/`"code-search"`, re-prompt more directively.
2. **Frontend backstop check** (frontend MRs) — defence-in-depth for residual drift after step 1's prevention, and the expensive path (Compass has already written and opened). Run only as a safety net: *if* you can fetch the diff (GitLab MCP `get_merge_request_diffs`), scan for —
   - a **`styled.button` / `styled(<native interactive>)`** or a raw `<button>` / `<div role="button">` where a DL component exists;
   - a **hand-rolled `<svg>`/glyph** instead of the DL icon component;
   - hand-rolled **box-shadow / border-radius / z-index** that a DL component owns;
   - hardcoded hex/px, or a `.scss` file;
   - a user-facing string without a `defineMessages` descriptor (or an icon baked into one);
   - `document.getElementById` / `querySelector` instead of a ref;
   - a planned Mixpanel event missing.

   On any hit, re-prompt Compass with the **specific DL replacement** (e.g. "replace the `styled.button` with `@mindtickle/button/lib/PrimaryButton`") and do not hand off until it's fixed. If these keep firing, the directive in step 1 isn't airtight — fix the prevention, not just the symptom.
3. **Link PR → subtask.** Remote-issue-link API or `mcp__atlassian__addCommentToJiraIssue`. PR description should reference the subtask key for auto-linking.
4. **Transition** subtask to "In Review" if available: `mcp__atlassian__getTransitionsForJiraIssue` → `mcp__atlassian__transitionJiraIssue`.
5. **Persist** `story-pipeline/<KEY>/phase-5/<SUBTASK-KEY>.md` — PR URL, one-paragraph summary, follow-ups/caveats, transition outcome.

### Ordering

Independent deliverables → parallel dispatch in one turn. Dependent deliverables wait for predecessors to merge (or commit against a shared branch).

### Gate

After each PR:

> Subtask `<KEY>` shipped at `<PR-URL>`, notes in `phase-5/<KEY>.md`. Next: `<NEXT-KEY>`. (yes / pause)

- **yes** → next subtask.
- **pause** → stop here. Re-invoke the skill on `<KEY>` to resume. Cheap — useful if you want to merge earlier PRs first.

---

## MindTickle Frontend Code Review Guidelines

**Canonical source — the DL component catalogue lives at https://gitlab.com/mindtickle/design-library.** Every `@mindtickle/*` component, variant, prop, token path, and import path is defined there. **Always use the correct DL component over a hand-rolled element** — look it up in the DL repo before writing markup; don't improvise. Phase 3 records the chosen DL components / tokens / `defineMessages` keys / Mixpanel events; the Phase 5 directive has Compass read those definitions via its `gitlab-ops` expert and use them.

The non-negotiable principle Compass and reviewers enforce — **Design library over hand-rolled:** never a raw `<button>`, `<div role="button">`, `<Button type="primary">`, `styled.button` / `styled(<native interactive>)`, a hand-rolled `<svg>`/glyph, hardcoded hex / px / z-index / box-shadow / border-radius, or a `.scss` file; user-facing strings go through `defineMessages` (`react-intl`); API calls through `@mindtickle/api-helpers`; DOM access via refs (not `document.getElementById`/`querySelector`); tracking via the shell's `trackEvents` Mixpanel saga.

> Per-rule detail (exact component/variant names, token paths, import patterns) lives in the DL repo and in `standards/javascript/design-library/` + `mfe/`. Maintainers: keep those in sync with the DL repo.

---

## Loop-back rules

| User's concern | Loop back to |
|---|---|
| Story intent wrong / Grooming wrong or stale | Stop. Fix grooming via `groom-story-functionally` in JIRA, then re-run Phase 1 |
| Missing or out-of-scope touchpoint | Phase 2 (re-run, rewrite `phase-2.md`) |
| Plan detail wrong, touchpoints right | Phase 3 (re-run, rewrite `phase-3.md`) |
| Confluence page needs edits | Phase 4 (edit in place, update `phase-4.md`) |
| PR misses acceptance criteria | Phase 5 for that subtask |

---

## Reference

- `${CLAUDE_PLUGIN_ROOT}/skills/minion-plan-and-develop/references/compass-how-to.md` — `mcp__enggxmcp__` usage (discover-then-execute, intent phrasing, common capabilities including `atlassian-ops` and `mindtickle-help`). Read on first Compass call.
- `${CLAUDE_PLUGIN_ROOT}/skills/minion-plan-and-develop/references/cost-from-transcript.py` — computes EXACT Claude cost from a session transcript (dedups by message id, applies cache-tiered rates). Used in the **Cost tracking** section's EXACT path.