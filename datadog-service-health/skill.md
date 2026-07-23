---
name: datadog-service-health
description: Per-service health metrics for a calendar month. Sub-skill of health-review-report; use standalone for a single service check.
user-invocable: true
argument-hint: "[Month YYYY] [APM service name] [env e.g. prod or prod-us]"
---

# Datadog — per-service health (one service × one env)

Compute health metrics for **one** APM `service` and **one** `env` (e.g. `prod`, `prod-us`) for the **report month** and comparison windows. v1 does **not** apply per-service resource filters; query the service/env as configured in the team’s dashboard or APM.

## Inputs (standalone)

Collect if not already provided:

1. **`SERVICE`** — exact APM service name  
2. **`ENV`** — e.g. `prod`, `prod-us`  
3. **Month and year** — e.g. “February 2026”

**Compute `START_MS` / `END_MS`** (UTC, report month):

- `START_MS` = first instant of the report month as epoch ms (`Date.UTC(Y, monthIndex, 1, 0, 0, 0, 0)`).  
- `END_MS` = last instant of the report month (`Date.UTC(Y, monthIndex, lastDay, 23, 59, 59, 999)`), with `lastDay` from standard calendar rules including leap-year February.

## Inputs (orchestrated)

When `health-review-report` passes pre-computed values, use them as-is:

- `SERVICE`, `ENV`, `START_MS`, `END_MS`  
- Optional narrative alignment from the parent: **“this month”** = last 30 days, **“last month”** = previous 30 days, **“6-month”** = last 180 days — use these rolling windows **only** if the parent workflow instructs you to match the CX Health dashboard semantics; otherwise use calendar month for all series consistently and state which you used.

## Capability

**Datadog MCP tools** — use the following specific tools:

- **`aggregate_spans`** — primary tool for p90/p99 latency and error counts. Query with `service:<SERVICE> env:<ENV>`; use `group_by.fields: ["resource_name"]` for RPC-level breakdown. Computes: `P90` and `P99` on `duration`, `COUNT` on `*` filtered to `status:error`. Use for **Check A** (prior window) and **Check B** (6-month / 180-day baseline) **per RPC** when the tool accepts the time range at `resource_name` granularity.
- **`get_datadog_metric`** — fetch multi-query timeseries for 6-month latency trend (e.g. `p90:trace.grpc.server.duration{service:<SERVICE>,env:<ENV>}`). Use `from`/`to` for the calendar window. **Fallback for Check B:** if `aggregate_spans` cannot return a reliable **per-RPC** baseline over ~180 days (limits, empty groups, or tool errors), use **service-level** metrics over the same baseline interval as a **proxy** for that RPC’s Check B (document in the working notes only — do **not** put implementation caveats in the published report body).
- **`search_datadog_spans`** — inspect individual error spans to identify top error contributors (error messages, `resource_name`, status codes). Filter with `service:<SERVICE> env:<ENV> status:error`.
- **`search_datadog_services`** — retrieve service metadata (team, description, links) if needed.

If any of these tool names are unavailable, list the tools exposed by the Datadog MCP server and select the closest equivalent before proceeding.

## Windows

- **Current month** (or “this month” rolling per parent): primary numbers.  
- **Previous month** (or “last month” rolling): compare p90/p99 and error volume.  
- **6-month**: trend phrase for latency; approximate **errors per month** average over 6 months.

## Top contributors

- Identify **top 2–3 error contributors** (operation / resource + short context + representative error message or pattern).  
- Prefer the same breakdown the team’s dashboard uses (span attributes, `resource_name`, status codes).

## Top 5 RPCs by usage (AGENTS.md parity)

Align with CX-style health reports: **⚠️** only when **both** latency **and** errors worsen **vs prior month *or* vs a 6‑month baseline** (see flag rule below). If **only** latency **or** only errors worsens on a given check, that check does not qualify.

### Windows (must match the parent report)

- **Report window:** `START_MS` … `END_MS` (computed inline when standalone, or passed by orchestrator).  
- **Prior window (Check A — “last month”):** The **full calendar month immediately before** the report month (UTC), **unless** the parent explicitly uses **rolling** CX semantics — then use the **same duration** as the report window **immediately before** `START_MS` (end at `START_MS − 1` ms). Use the **same** calendar-vs-rolling choice as the service-level “Latency vs last month” / “Error trend vs last month” rows.  
- **Baseline window (Check B — “6‑month”):** **180 days** ending at `START_MS` (**half-open:** `[START_MS − 180d, START_MS)` in UTC), **unless** the parent uses strict calendar semantics — then use the **six full calendar months** immediately before the report month. Stay consistent with the service-level “6‑month” narrative for this run.

### Procedure

1. From APM, determine the **top 5 resources/operations by request count** (or throughput) for this service × env in the **report window** (`resource_name` / operation name as RPC label).

2. **Check A — prior window (month-over-month):** For each of the 5 RPCs, compare **report window** vs **prior window**:  
   - **Latency:** same headline statistic as the report (default **p90** on duration for that RPC; use **p99** instead only if the team dashboard standardizes on p99 for RPC rows — stay consistent within the report).  
   - **Errors:** error **count** (or error rate if counts are unavailable — state which was used in working notes only).  
   **Check A is true** when **both** latency **increased** and errors **increased** vs the prior window (strictly higher numbers, not “flat”).

3. **Check B — 6‑month baseline:** For each of the 5 RPCs, over the **baseline window**:  
   - **Latency baseline:** **average p90** for that RPC across the baseline window (one aggregate per RPC over the whole baseline interval is acceptable; alternatively average of **monthly** p90 values if you already have monthly buckets — pick one method and use it for all five RPCs in the same section).  
   - **Error baseline:** **average errors per month** for that RPC = *(total error spans in baseline window for that RPC) / 6* when the baseline is 180 days; if using six calendar months, divide by **6** as well.  
   **Check B is true** when **both** `report_window_p90 > baseline_avg_p90` **and** `report_window_errors > baseline_avg_errors_per_month` for that RPC. (This is “higher than the 6‑month typical” for both dimensions at once.)

4. **Anomaly flag (union):** For each RPC, **⚠️** if **Check A *or* Check B** is true. **✅** otherwise (including when only one dimension worsens, or when checks disagree).  
   - **Partial data:** If Check B cannot be computed for a given RPC (empty series, query failure, or RPC missing from baseline breakdown), apply **Check A only** for that RPC’s flag. If **Check A** is also unavailable, output **✅** and treat detailed comparison as `[TBD]` only if the whole Datadog step failed.

5. **Narrative below the list:** Use the phrase **“error anomaly”** only when errors are **elevated vs baseline** (prior or 6‑month, same as the narrative elsewhere in the section). For **very low counts** (e.g. 2–3), write *“N errors (low count; listed for visibility)”* — do **not** call that an **error anomaly**.

6. **Tooling:** Prefer **`aggregate_spans`** with `group_by.fields: ["resource_name"]` for per-RPC p90 and error counts in each window. If a **180‑day** (or six‑month) per-RPC aggregate is **not** returned reliably, use the **`get_datadog_metric`** fallback described under **Capability** for Check B **only**, and prefer Check A for the flag when the fallback would blur per-RPC truth.

## Verdict (one line)

Apply the same rules as the report template:

- **✅ Good** — latency stable or improving (p90/p99), no error spike.  
- **⚠️ Review** — latency up for one or both, or a one-off error spike / single high-error day without sustained degradation.  
- **🔴 Attention** — sustained error spike (e.g. ≫ 6‑month average) or sustained degradation.

Use **emojis + words** exactly: `✅ Good`, `⚠️ Review`, `🔴 Attention`.

## Output structure

Return markdown suitable for pasting into the report’s per-service table:

1. **Verdict line:** `**Verdict:** <emoji> <Status> — <short reason>**`  
2. **Table rows** (exact row keys expected by the template):  
   - Latency vs last month (p90 / p99)  
   - 6-month latency trend (p90 / p99)  
   - Error trend vs last month (counts + one-line trend)  
   - 6-month average errors/month  
   - Top 2–3 error contributors — bullet cell using `<br>•` between items (Confluence-friendly)  
   - Top 5 RPCs — one line per RPC with ✅ or ⚠️ (dual baseline: prior month **or** 6‑month per‑RPC, per **Top 5 RPCs by usage** above), plus legend and short narrative only where needed

## Rules

- Do **not** mention internal filter tweaks in the report body.  
- If Datadog is unreachable, output `[TBD — Datadog access required]` per cell and list what was attempted.