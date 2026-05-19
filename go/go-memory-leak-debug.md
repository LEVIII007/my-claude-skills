---
name: go-memory-leak-debug
description: Debug memory leak issues in a Go service. Verifies the repo is a Go module (stops early if not), detects the Git remote URL for MR creation, asks structured intake questions, statically analyses the codebase for leak patterns (goroutine leaks, unbounded caches, unclosed resources, context leaks, timer leaks), generates a ranked report of top N root causes (default 5, configurable via --top N), applies per-confirmed code fixes, and optionally raises a GitLab / GitHub MR/PR with all applied fixes. Use when a Go service shows steadily growing memory usage or OOM crashes.
argument-hint: "[--top N] [package or file path]"
allowed-tools: Bash Read Write Edit Glob Grep AskUserQuestion Skill
---

Debug the Go memory leak. Arguments: `$ARGUMENTS`

---

## Step 0: Parse Arguments (MANDATORY FIRST STEP)

Inspect the **first token** of `$ARGUMENTS`:

- Starts with `--top` → extract `N` from `--top N` (e.g. `--top 3` sets `TOP_N = 3`). Set `TOP_N_FROM_FLAG = true`. Remaining text after the token is `SCOPE`.
- Otherwise → `TOP_N_FROM_FLAG = false`. Full `$ARGUMENTS` is `SCOPE` (may be empty — means entire repo).

`SCOPE` is a Go package path or file path to restrict analysis. Empty means the full repo.

---

## Step 0.5: Verify Go Service + Detect Git Remote (STOP EARLY IF NOT GO)

**This step must run before any intake questions.**

### Go service check

```bash
find . -name "go.mod" -not -path "*/vendor/*" -maxdepth 4 | head -1
```

If the output is **empty** (no `go.mod` found): output the following message and **STOP immediately** — do not ask any intake questions, do not continue:

> "This skill is Go-specific. No `go.mod` was found in this directory or its subdirectories (up to 4 levels deep). Please run this skill from the root of a Go module. If this is a polyglot repo, navigate to the Go service subdirectory first."

If `go.mod` is found: read the module name from it (first line: `module <name>`) and store as `MODULE_NAME`.

### Git remote URL detection

```bash
git remote get-url origin 2>/dev/null
```

- If a URL is returned: set `SERVICE_REPO_URL = <url>`. Tell the user:
  > "Detected Git remote: `<url>`. This is the service that will be analyzed and where fixes will be raised as an MR. If this is wrong, type the correct URL."
  If the user provides a correction, update `SERVICE_REPO_URL`. The URL is recorded in the report header for traceability; MR creation is handled by the `raise-mr` skill in Step 9.

- If no remote is found (not a git repo or no `origin` remote):
  Ask the user (via `AskUserQuestion`, one question):
  > "No Git remote detected. What is the remote URL for this Go service?"
  Options:
  - `Skip — I will handle MR creation manually`
  - `Enter URL via Other field (HTTPS or SSH)`
  The "Other" free-text entry captures the actual URL. Store the typed value as `SERVICE_REPO_URL`.
  If the user chooses "Skip" or enters nothing: set `SERVICE_REPO_URL = ""` — the analysis continues, but MR creation in Step 9 will note that no remote is configured.

---

## Step 1: Intake — Collect Context

### 1a. Severity and observability (call AskUserQuestion once with all four)

Ask the following four questions together:

1. **How long has the memory leak been observed?**
   Options: `< 1 week` | `1–4 weeks` | `1–3 months` | `> 3 months`
   → store as `LEAK_DURATION`

2. **If the service is not restarted, how often does it OOM?**
   Options: `Multiple times a day` | `Once a day` | `Every 2–4 days` | `Weekly or less / Unknown`
   → store as `OOM_FREQUENCY`

3. **Approximate memory growth before OOM (startup → peak)?**
   Options: `< 500 MB growth` | `500 MB – 2 GB growth` | `> 2 GB growth` | `Unknown`
   → store as `MEMORY_GROWTH`

4. **Do you have pprof heap profiles from the leaking service?**
   Options: `Yes — will share the top20 output` | `No — please guide me on collection` | `Not sure what pprof is`
   → store as `PPROF_STATUS`

### 1b. Scope and focus (call AskUserQuestion once; omit Q4 if `TOP_N_FROM_FLAG = true`)

1. **Which components do you suspect are leaking?** (multiSelect: true)
   Options: `HTTP handlers or middleware` | `Background workers or goroutines` | `In-memory caches, maps, or connections` | `Unknown — scan everything`
   → store as `SUSPECTED_COMPONENTS`
   Note: "In-memory caches, maps, or connections" covers unbounded maps, database/SQL pools, gRPC streams, and HTTP transports.

2. **Were there notable code or dependency changes before the leak appeared?**
   Options: `Yes — new feature deployed` | `Yes — dependency upgraded` | `No recent changes` | `Not sure`
   → store as `RECENT_CHANGES`

3. **What scope should the analysis cover?**
   Options: `Entire repo` | `Recent git changes only (last 20 commits)` | `Specific package or file`
   If "Specific package or file" and `SCOPE` is already set from arguments, use it; otherwise ask the user to type the path via "Other".
   → store as `ANALYSIS_SCOPE`

4. **(skip if TOP_N_FROM_FLAG = true) How many top findings should be reported and fixed?**
   Options: `3` | `5 (recommended)` | `10`
   → store as `TOP_N`

---

## Step 2: Load Standards

Read the following files — do not skip, do not rely on training knowledge for patterns:

- `${CLAUDE_PLUGIN_ROOT}/standards/go/memory.md` ← primary pattern catalogue and severity guide
- `${CLAUDE_PLUGIN_ROOT}/standards/go/coding.md` ← concurrency and lifecycle rules
- `${CLAUDE_PLUGIN_ROOT}/standards/go/perf.md` ← allocation anti-patterns

Keep the loaded pattern list, severity weights, and scoring multipliers in memory for Steps 4 and 5.

---

## Step 3: Collect Service Context

Go module already verified in Step 0.5 (`MODULE_NAME` is set). Proceed directly to building the file list.

Determine the file list based on `ANALYSIS_SCOPE`:

- **Entire repo:**
  ```bash
  find . -name "*.go" -not -path "*/vendor/*" -not -name "*_test.go" | head -200
  ```
  If the output contains exactly 200 lines, the list was truncated. Add a visible warning to the report header:
  > ⚠️ **Analysis incomplete — file list capped at 200 files.** Large repos may have unscanned files. Re-run with `--top N` and a specific package path to narrow scope.

- **Recent git changes:**
  ```bash
  git diff HEAD~20 --name-only | grep "\.go$" | grep -v "_test.go"
  ```
- **Specific package:**

  Before running `find`, validate `SCOPE`:
  - If `SCOPE` is empty or was not set, treat as "Entire repo" and use the command above.
  - If `SCOPE` starts with `../` or contains `/../`, tell the user: "Invalid scope path — path traversal (`../`) is not allowed." and stop.
  - If `SCOPE` contains characters outside `[a-zA-Z0-9_./-]`, tell the user: "Invalid scope path — only alphanumeric characters, `.`, `/`, `-`, and `_` are allowed." and stop.
  - If the path does not exist (`[ ! -d "./<SCOPE>" ]`), tell the user: "`<SCOPE>` does not exist in this repo. Please re-run with a valid package path." and stop.

  ```bash
  find "./<SCOPE>" -name "*.go" -not -name "*_test.go" 2>/dev/null | head -100
  ```

Quick structural scan to identify high-signal files first:

```bash
grep -rn \
  "go func(\|time\.NewTicker\|time\.NewTimer\|context\.With\|http\.Get\|http\.Post\|\.Do(\|db\.Query\|\.QueryContext\|\.Send(\|\.Recv(\|sync\.Pool\|^var .*= map\[" \
  --include="*.go" \
  --exclude-dir=vendor \
  --exclude="*_test.go" \
  -l . 2>/dev/null | head -30
```

Prioritise files in `SUSPECTED_COMPONENTS` areas and files with the highest hit density from the grep above. Read those files in full before moving to lower-signal files.

If `RECENT_CHANGES` is "Yes — new feature deployed" or "Yes — dependency upgraded":
```bash
git log --oneline -20
git diff HEAD~5 --name-only | grep "\.go$" | grep -v "_test.go"
```
Read changed files first — they are the most likely regression site.

---

## Step 4: Static Analysis — Detect Leak Patterns

For each file read in Step 3, check against every pattern in `standards/go/memory.md`.

For each finding, record:
- **Pattern ID** (e.g. P1-1, P2-3)
- **Pattern name** (human-readable)
- **File path and line number(s)**
- **Severity** (P1 / P2 / P3)
- **Code snippet** (the offending lines, up to 10 lines of context)
- **Why it leaks** (from memory.md description for this pattern)
- **Proposed fix** (from memory.md fix template, adapted to the actual code)

**False positive guard:** Before flagging, read the full surrounding function (not just the matched line) and any immediate callers. If the apparent issue is intentionally handled elsewhere (e.g. the goroutine is stopped by an external cancel that is not visible at the match site), mark it as "Likely not a leak — context found" and exclude it from the ranked list. Accuracy matters more than hit count.

### pprof guidance

If `PPROF_STATUS` is `No — please guide me on collection` or `Not sure what pprof is`, output the collection commands from `standards/go/memory.md` (section "pprof Collection Commands") to the user now. Tell the user:

> "Run these commands against your staging or production service to collect heap profiles. You can paste the `top20 -cum` output back here and I will incorporate it into the ranking. You can also continue without pprof — the static analysis will still identify structural issues."

If `PPROF_STATUS` is `Yes — will share the top20 output`, ask the user to paste the output now. Parse the function names from the pprof output and note which findings match those hot functions — these get the `× 2.0` multiplier in Step 5.

---

## Step 5: Rank Findings and Select Top N

Score each finding using the weights and multipliers from `standards/go/memory.md` (section "Severity Scoring for Ranking"):

- Base weight: P1 = 3, P2 = 2, P3 = 1
  - **Exception — P2-3:** use base weight = **1** (P3-equivalent). Its memory impact is bounded; the risk is correctness/performance, not unbounded growth.
- Multipliers (cumulative, multiply together):
  - Finding is in a component matching `SUSPECTED_COMPONENTS`: × 1.5
  - Finding is in a file changed in recent commits (and `RECENT_CHANGES` is "Yes"): × 1.3
  - Finding confirmed by pprof data (function appears in `top20 -cum`): × 2.0

Sort by final score descending. Break ties by severity (P1 > P2 > P3), then by file name alphabetically for determinism.

Select the top `TOP_N` findings. If fewer than `TOP_N` findings were found, report all of them and note that the analysis is complete.

---

## Step 6: Write the Report

Write `memory-leak-report.md` to the repository root (or to `<SCOPE>/` if analysis was scoped to a specific package):

```markdown
# Go Memory Leak Analysis Report

**Service:** <module name from go.mod>
**Repository:** <SERVICE_REPO_URL or "not configured">
**Date:** <today's date>
**Analysis scope:** <ANALYSIS_SCOPE — Entire repo / Recent changes / <path>>
**Leak duration:** <LEAK_DURATION> | **OOM frequency:** <OOM_FREQUENCY> | **Memory growth:** <MEMORY_GROWTH>
**Findings ranked:** top <TOP_N> of <total found>
<If pprof data was provided: "pprof confirmation used in ranking.">

---

## Summary

<2-3 sentences describing the dominant leak category found, e.g. "The primary leak source is goroutine accumulation: 4 goroutines are spawned without a shutdown path. Secondary contributors are 2 unclosed HTTP response bodies in the request handling layer.">

---

## Findings

### Finding 1 — <Pattern Name> [<Severity>] ⭐ Score: <score>

**Pattern:** <Pattern ID from memory.md>
**File:** `<path/to/file.go>:<line>`
**Suspected component:** <matching component from SUSPECTED_COMPONENTS, or "General">

**Why it leaks:**
<explanation from memory.md>

**Evidence (offending code):**
\`\`\`go
<code snippet>
\`\`\`

**Proposed fix:**
<description of the change needed>

\`\`\`go
<fix code, adapted to the actual function>
\`\`\`

**Estimated impact:** <High / Medium / Low based on severity and call frequency>

---

<repeat for each finding>

---

## pprof Collection Guide

<Include the full collection commands from standards/go/memory.md if PPROF_STATUS indicated the user needs guidance. Omit this section if the user already has profiles.>

---

## Next Steps

1. Review findings above and confirm which fixes to apply (Claude will walk through each).
2. After applying fixes: `go build ./...` and `go test -race ./... -count=1` must pass.
3. Deploy to staging and collect two heap profiles 5 minutes apart; compare with:
   `go tool pprof -base heap1.prof heap2.prof` → `top20 -cum`
4. If memory growth persists after all P1 fixes, share the post-fix pprof diff for deeper analysis.
5. Delete `memory-leak-report.md` after fixes are merged (it may contain internal paths).
```

After writing the file, tell the user:
> "`memory-leak-report.md` written to `<path>`. You can delete this file after fixes are merged."

---

## Step 7: Apply Fixes — Per-Finding, Confirmed

For each of the top `TOP_N` findings **in ranked order**:

**7a. Present the fix**

Show the exact before/after diff for this specific finding. Include:
- The function name and file:line
- A minimal diff showing only the lines changing
- A one-sentence explanation of what the change does

**7b. Confirm**

Ask the user one question:
> "Apply this fix to `<file>:<function>`? (Yes / No / Skip all remaining)"

- **Yes** → apply the edit and continue to 7c
- **No** → note this finding as "Skipped by user" in the summary table; continue to the next finding
- **Skip all remaining** → stop the fix loop, proceed to Step 8

**7c. Validate compilation**

```bash
go build ./...
```

- **Passes** → mark finding as "Fixed ✅"
- **Fails** → revert the edit immediately using the exact original content; mark finding as "Fix reverted — compilation error"; print the compiler error; continue to the next finding

**7d. Validate concurrency (for P1-1, P1-2, or P2-3 findings)**

```bash
go test -race ./... -count=1 -timeout 120s
```

Run this for any goroutine-related finding (P1-1, P1-2) **or** any P2-3 finding — P2-3 (sync.Pool misuse) is explicitly a race-safety concern and must be verified with the race detector.

- **Passes** → continue
- **Fails** → report the race or test failure; do not revert (the compilation already passed); ask the user how to proceed

---

## Step 8: Final Summary

```
## Memory Leak Debug — Complete

Report file: memory-leak-report.md
Total patterns scanned: <count from memory.md>
Findings detected: <total raw findings before ranking>
Reported (top N): <TOP_N>

| # | Pattern | ID | Severity | File | Score | Status |
|---|---------|-----|----------|------|-------|--------|
| 1 | <name> | P1-1 | P1 | file.go:42 | 6.0 | Fixed ✅ |
| 2 | <name> | P2-1 | P2 | file.go:88 | 2.6 | Skipped |
...

Next steps:
- Delete memory-leak-report.md after fixes are merged
- Collect pprof heap profiles in staging before and after deploying
- Monitor goroutine count at /debug/pprof/goroutine?debug=1 for 30 minutes under load
- <If any P1 findings remain unfixed>: these are critical — address before next production deploy
```

---

## Step 9: Commit and Raise MR (Optional)

Only proceed with this step if **at least one fix was applied** in Step 7 (status "Fixed ✅").

### 9a. Ask whether to commit and raise an MR

Ask via `AskUserQuestion` (one question):

> "Would you like to commit the applied fixes and raise a merge request?"
> Options: `Yes — commit and raise MR` | `No — I will handle it manually`

If "No": tell the user which files were modified and stop.

If `SERVICE_REPO_URL = ""` and the user chose "Yes": warn that no git remote was detected — the `raise-mr` skill may ask for one or fail at the push step.

### 9b. Exclude the report from git

`memory-leak-report.md` is a local analysis artefact — it must never be committed. Before staging anything, ensure it is excluded:

```bash
# Add to .gitignore if not already present
grep -qxF 'memory-leak-report.md' .gitignore 2>/dev/null || echo 'memory-leak-report.md' >> .gitignore

# Untrack if it was previously committed
git rm --cached memory-leak-report.md 2>/dev/null || true
```

### 9c. Stage the modified files

Stage the files that were actually modified by accepted fixes (use the exact paths recorded in Step 7) plus `.gitignore`. Always include `.gitignore` — staging it is idempotent when already clean, and ensures the entry is committed if it was just added. Do not stage `memory-leak-report.md`.

```bash
git add <file1> <file2> ... .gitignore
```

### 9d. Commit via smart-commit

Delegate to the `smart-commit` skill using `--fast` mode (compilation was already verified after each fix in Step 7c — running full tests again here would duplicate that work):

```
Skill(skill="smart-commit", args="--fast fix(memory): resolve <FIXED_COUNT> memory leak pattern(s) in <MODULE_NAME> — <comma-separated pattern IDs e.g. P1-1, P2-1>")
```

`smart-commit` will:
- Guard against committing directly on `main` (prompts for a branch name if needed)
- Run the linter and secret scan (fast-mode gates)
- Use the supplied message and append the `Smart-Commit: fast-mode` trailer required by `raise-mr`

If `smart-commit` stops with an error (linter failure, secret detected, etc.): **do not continue to 9e**. Report the error and let the user resolve it.

### 9e. Raise the MR via raise-mr

Once the commit succeeds, delegate to the `raise-mr` skill:

```
Skill(skill="raise-mr")
```

`raise-mr` will:
- Run the full-diff code review
- Ask for a Jira issue ID
- Push the branch and create the GitLab MR via `glab`
- Apply the standard AI-assist labels (`ai-raised`, `ai-assist::yes`, etc.)
- Post a quality summary comment on the MR

No additional arguments are needed — `raise-mr` reads the branch state and diff itself.