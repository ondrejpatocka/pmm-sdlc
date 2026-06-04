# PMM Bug Triage Workflow

A repeatable workflow for triaging a single PMM Jira bug ticket. Produces a local Markdown report; never mutates Jira or GitHub.

This document is intentionally tool-agnostic. It describes **what** an AI assistant (or a human) should do at each step and **what** to write to disk — not which CLI, MCP, IDE, or API to use. Pick whichever assistant fits your environment.

> **Cardinal rule:** Never treat Jira text alone as proof the bug still exists on current `v3`. Verify against the current code in Step 5 before drawing conclusions.

## How to invoke this workflow

You can drive this workflow with any general-purpose AI coding assistant. The operator's job is to point the assistant at:

1. This workflow document (as the instruction set).
2. The Jira ticket URL to triage.
3. The local `pmm` repository checkout (and any sibling repos under the same parent directory).

Suggested ways to invoke it with the most common assistants:

- **Cursor IDE:** open this file in your workspace, then prompt: *"Follow `pmm-triage-bug-workflow.md` for ticket `<URL>`."* Optionally save a slimmer copy as a Cursor rule under `.cursor/rules/` to attach it automatically.
- **Claude (Code, Desktop, or API):** attach this file as context, then prompt: *"Use the attached workflow to triage `<URL>`."* In Claude Code, place a copy at the repo root or under `.claude/` so it's picked up by default.
- **GitHub Copilot Chat:** open this file in the editor, then in the chat panel use `#file:pmm-triage-bug-workflow.md` and prompt: *"Triage `<URL>` per this workflow."*
- **CLI / scripted runs:** pipe the workflow plus the ticket URL into your assistant of choice; the workflow's autonomous-mode behavior (see §0 and §9) is designed for unattended runs.

Whichever you use, the assistant must respect the guardrails in §0 (read-only Jira/GitHub, local-only outputs) and emit the same report layout (see Appendix B).

## 0. Modes & guardrails

This workflow supports two execution modes; the steps are identical, only the uncertainty behavior differs.

- **Assisted mode (default):** A human triager drives; the assistant asks for clarification when stuck.
- **Autonomous mode:** No human in the loop; the assistant writes a `[BLOCKED: needs-human]` section to the report and stops the workflow cleanly instead of asking.

The assistant must:

- Read and act only with the access already configured on the operator's machine (Jira, GitHub, local repos). Do not require a specific tool.
- Never modify the Jira ticket, never open PRs, never push commits.
- Write only to `pmm-sdlc/ai-workflows/pmm-triage-bug-ticket/audit-log/` inside the `pmm-sdlc` repo working tree.
- Record every irreversible decision (label/component choices, verdict, blocked questions) in the report so the human has an audit trail.

## 1. Preconditions

Fail fast if any of these is false. Record what failed and stop.

- The Jira ticket URL is provided (example: `https://perconadev.atlassian.net/browse/PMM-15076`).
- Jira is reachable with the operator's access (any method: MCP, REST, browser fetch, manual paste).
- The `pmm` repo is checked out at the current `v3` branch. Record the branch name and HEAD commit in the report header.
- `pmm-sdlc/ai-workflows/pmm-triage-bug-ticket/audit-log/` exists inside the `pmm-sdlc` working tree. If missing, create it.

## 2. Fetch the Jira ticket and initialize the report

This step produces the single report file used for the rest of the workflow. Every later step appends to the same file; nothing else is written elsewhere.

### 2.1 Create the report file

Create:

```
pmm-sdlc/ai-workflows/pmm-triage-bug-ticket/audit-log/Triage-report-<KEY>-<slug>-<YYYYMMDD-HHMMSS>.md
```

File requirements:

- UTF-8 encoding, no BOM.
- LF line endings.
- Human-readable on Windows, macOS, and Linux.
- `<slug>` is derived from the issue summary, lowercased ASCII, characters restricted to `[a-z0-9-]`, capped at 50 chars (truncate at a word boundary).
- `<YYYYMMDD-HHMMSS>` is the operator's **local** time at run start.

### 2.2 Write the run header

Write the header immediately, before any Jira fetch or analysis, so an aborted run still leaves a useful artifact:

```markdown
# Triage report — <KEY>

- Ticket: <full Jira URL>
- Run started: <YYYY-MM-DD HH:MM:SS local, UTC±HH:MM>
- pmm branch: <branch name>
- pmm HEAD: <commit short SHA>
- Mode: assisted | autonomous
- Operator: <user@host or "autonomous">
- Assistant: <name and version, e.g. "Cursor / Claude Sonnet", "Claude Code", "Copilot Chat">
```

### 2.3 Fetch and append the Jira ticket snapshot

Capture the following fields from the ticket and append them under a `## Ticket snapshot` section in the report. This snapshot is the input to every analysis step that follows.

- `key`, `issuetype`, `summary`, `priority`, `status`
- `components`, `labels`, `fixVersions`, `affectsVersions`, `resolution`
- `reporter`, `assignee`, `created`, `updated`
- `parent` (key only)
- `description` (full text)
- All comments (author, timestamp, body)
- All issue links (relation + target key + target summary)
- Attachments list (filename + URL) — needed for the Evidence check in Step 4

If the ticket cannot be read, append a one-line note to the report explaining why and stop the workflow. Do not delete the partially-written report — the header alone is still useful as an audit trail.

Suggested layout for the snapshot section:

```markdown
## Ticket snapshot

### Fields
| Field | Value |
|---|---|
| key | <...> |
| issuetype | <...> |
| summary | <...> |
| priority | <...> |
| status | <...> |
| components | <...> |
| labels | <...> |
| fixVersions | <...> |
| affectsVersions | <...> |
| resolution | <...> |
| reporter | <...> |
| assignee | <...> |
| created | <...> |
| updated | <...> |
| parent | <key only> |

### Description
<verbatim description>

### Comments
- <author> @ <timestamp>:
  <body>
- ...

### Issue links
- <relation> → <KEY> — <target summary>
- ...

### Attachments
- <filename> — <url>
- ...
```

## 3. Status gate

Allowed statuses: `New`, `Open`, `To Do`.

- If the status is allowed, continue.
- If not, write a one-line skipped report and stop the workflow:

  Filename: `pmm-sdlc/ai-workflows/pmm-triage-bug-ticket/audit-log/Triage-skipped-<KEY>-<slug>-<YYYYMMDD-HHMMSS>.md`

  Content (single line plus header):

  ```markdown
  Skipped: status=<actual status>. Triage workflow only runs on New / Open / To Do.
  ```

Also check, before continuing:

- Assignee. If the ticket is assigned to someone other than the triager / unassigned, flag it in the report (do not stop). Triaging an in-flight ticket can step on toes.
- Existing proposed solutions. Search the ticket's comments, `is fixed by` / `is implemented by` links, and any URLs pointing to `github.com/percona/pmm/pull/...` for an existing fix attempt. Record findings.

## 4. Completeness check

Evaluate whether the ticket contains enough information to act. Append a **Completeness** section to the report covering:

### 4.1 Findings

- **Short summary** of the issue in the assistant's own words (1–3 sentences).
- **Steps to Reproduce (STR) assessment:** Present / Partial / Missing. Quote the STR if present.
- **Expected vs. Actual result:** Present / Partial / Missing.
- **Environment details:** check for and quote:
  - PMM Server version + deployment method (Docker / OVF / AMI / Helm / HA). "PMM 3.x" is not enough — require an exact tag.
  - PMM Client version (only when it differs from the server, e.g. backward-compat scenarios).
  - Monitored DB type and version (e.g. "PSMDB 7.0.14 replica set", "PostgreSQL 16.3 with pg_stat_monitor"). Bugs in QAN, exporters, and Backup are nearly always DB-version-specific.
- **Supporting evidence:** screenshots, video, logs, stack traces — note what is present, what is missing.
- **Reproducibility verdict:** one of `Reproducible from ticket` / `Plausible — needs verification` / `Not enough info`.
- **Customer impact / occurrences:** single report, multiple customers, theoretical — drives priority.

### 4.2 Labels detected

Pick zero or more of the following Tech labels. State the evidence (quote from ticket) for each pick.

- `Tech/MySQL` — MySQL DB monitoring and tooling.
- `Tech/MongoDB` — MongoDB DB monitoring and tooling.
- `Tech/PostgreSQL` — PostgreSQL DB monitoring and tooling.
- `Tech/Valkey` — Valkey **or** Redis DB monitoring and tooling (one label covers both).

### 4.3 Components detected

Pick one or more components (cap at 3; pick the most likely if more match). State the evidence for each pick.

- `Documentation` — technical documentation.
- `Grafana Dashboards` — dashboards, panels, metrics presentation.
- `QAN` — Query Analytics feature, including stored metrics.
- `RTA` — Real-time query analytics.
- `Alerting` — alerting feature.
- `Advisors` — advisors feature.
- `Backups` — backups feature.
- `Inventory` — inventory management of nodes, services, agents.
- `pmm-client` — PMM client distribution.
- `pmm-agent` — client agent component.
- `pmm-admin` — CLI component.
- `Docker` — Docker installation.
- `K8s` — Kubernetes installation.
- `HA` — High Availability installation.
- `OpenShift` — OpenShift installation.
- `AMI` — AWS installation.
- `Exporters` — `*_exporter` repos (mysqld_exporter, mongodb_exporter, postgres_exporter, etc.).
- `VictoriaMetrics` — VM time-series store.
- `ClickHouse` — ClickHouse storage for QAN.
- `API` — gRPC / REST API surface.
- `UI` — PMM frontend.
- `Updates` — upgrade / update flow.
- `Security` — auth, RBAC, TLS, secrets.
- `Telemetry` — telemetry collection.

### 4.4 Light sanity flags

Mark any that apply (none, one, or several). Each flag must have a one-sentence justification.

- **Obsolete / not relevant** — already fixed on `v3`, duplicate of another ticket, feature removed, or only affects EOL versions with no supported backport path.
- **Not a bug** — works as designed, user misunderstanding, needs documentation or training instead of code.
- **Not a sensible PMM code change** — wrong product, support / config issue only, no credible change in this repo matches the report.
- **Should not be fixed** — security, compatibility, or product policy implies "Won't Fix" or "Not a bug" is the right Jira outcome.
- **Stale or empty** — insufficient detail to act; guessing would be irresponsible.

## 5. Code / runtime verification

This is the enforcement step for the cardinal rule. Append a **Codebase verification** section.

### 5.1 Scope

Search every repo checked out under the parent directory of the `pmm` repo (a common convention is `$GITHUB_ROOT`), plus any repo explicitly named in the ticket. Oversearch is acceptable; missing the relevant repo is not. Typical hits:

- `pmm` (sub-areas: `managed/`, `agent/`, `admin/`, `api/`, `qan-api2/`, `vmproxy/`, `ui/`, `build/`, `api-tests/`, `dashboards/`).
- `pmm-qa`.
- `grafana` (Percona fork).
- `*_exporter` repos when present.
- PBM and similar adjacent repos when present.

### 5.2 What to search for

- Exact error strings or log lines quoted in the ticket.
- Symptom keywords (feature name, endpoint, metric name, panel title).
- API surface in `pmm/api/**/*.proto` if the report touches a gRPC or REST contract.
- Relevant component `AGENTS.md` files for architectural intent.

### 5.3 What to record

- File paths and line numbers cited as `repo/path/to/file.go:123`.
- One of: `Still present on v3` / `Appears fixed on v3` / `Not located — symptom area unclear`.
- Short rationale (≤3 sentences) — what the code does today vs. what the ticket reports.

## 6. De-duplication

Append a **De-duplication** section.

### 6.1 Search surfaces (in order)

1. Open and recently-closed `PMM-*` Jira tickets in the same component. The bug template's title prefix makes this fast: `[QAN]`, `[Backup]`, `[Alerting]`, `[Inventory]`, etc.
2. GitHub issues on `percona/pmm` and on the relevant exporter repo if it looks like an exporter bug.
3. Upstream issue trackers: Grafana (Percona fork), VictoriaMetrics, ClickHouse, Exporter.

### 6.2 Search rules

- Time window: open + closed within the last 1 month. Older matches go in a separate "historical" subsection only if they look causally linked.
- Cap the result list at **10 matches total** across all surfaces. Rank by relevance.
- Each match must have a one-line justification (why it might be related).
- Tag each match with one relation:
  - `duplicates` — same bug, same root cause. Recommend closing as Duplicate of that key.
  - `is caused by` — root cause is in an upstream issue. PMM ticket becomes a tracker, not a duplicate.
  - `relates to` — similar but distinct symptom.
  - `unrelated — considered` — found by search but ruled out (still record it, briefly).

## 7. Validity assessment

Append a **Validity assessment** section. Just because a user does not like how something works does not mean it's a bug.

Cross-check sources, in priority order, when deciding "as designed":

1. API contract: `.proto` files under `pmm/api/` (source of truth for behavior).
2. User docs: `pmm/documentation/` (MkDocs).
3. Component `AGENTS.md` files (`managed/`, `agent/`, `admin/`, `ui/`, `api/`, `qan-api2/`, `vmproxy/`, `build/`, `api-tests/`, `dashboards/`) for architectural intent.
4. Original feature ticket or acceptance criteria, when linked.
5. Existing tests under `pmm/api-tests/` — if a test already asserts the current behavior, that's a strong "as designed" signal.

Classify the ticket into exactly one of the following (the 6-way split):

- **(a) Defect** — broken in the current release, never worked. Flag as a Bug.
- **(b) Regression** — worked in a previous GA, broken now. Flag as a Bug; record the last known working version in the report (Jira's `Affects version` semantics vary by project — do not change Jira here, just record).
- **(c) Upstream defect** — root cause is in Grafana / VictoriaMetrics / ClickHouse / PBM / an exporter. Recommend the `Upstream` label and a link to the upstream issue.
- **(d) Documentation gap** — system behaves as designed, but the docs disagree or are silent. Recommend re-routing to the `Documentation` component.
- **(e) Configuration / environment issue** — user ran client newer than server, unsupported DB version, exposed a DB the docs say isn't supported, hit a known limit (cardinality, retention). Recommend "Not a bug" with a link to the docs page.
- **(f) Feature request / enhancement** — works as designed; the user wants different behavior. Recommend re-typing the ticket as a feature request.

Include a one-paragraph justification with evidence references (file paths, doc URLs, proto symbols).

## 8. Verdict & recommended actions

Append the final **Verdict** section. This is the deliverable.

### 8.1 Outcome

Pick exactly one:

- `Ready for Dev`
- `Needs Info` — list the specific questions to ask the reporter.
- `Duplicate of <KEY>`
- `Won't Fix — works as designed`
- `Won't Fix — out of scope`
- `Documentation` — route to the docs team.
- `Upstream tracker — <link>` — keep open as a tracker for an upstream fix.
- `Insufficient info — escalate to human triager`

### 8.2 Suggested Jira changes

A flat list of changes the triager should apply manually. The assistant does not apply them. Each item must be actionable:

- Labels to add / remove (with justification).
- Components to add / remove (with justification).
- Priority delta, if any (with justification).
- `fixVersion` candidate, if known.
- Suggested assignee or team, if known.

### 8.3 Confidence

- `High` / `Medium` / `Low`.
- One sentence answering: "what would change my mind?"

### 8.4 Open questions for the reporter

Only populated when the outcome is `Needs Info` or confidence is `Low`. Numbered, specific, answerable.

## 9. Uncertainty & handoff

If the assistant is genuinely unsure at any step — cannot disambiguate between two components, cannot reproduce from the description, cannot locate the symptom area in code, finds conflicting evidence — do **not** guess.

- **Assisted mode:** ask the human in chat. Record the question, the human's answer, and the chosen direction in an `## Assistant uncertainty log` section at the end of the report.
- **Autonomous mode:** stop the workflow. Append a `## [BLOCKED: needs-human]` section listing the question(s), what was tried, and what the assistant would need to proceed. Exit non-zero if invoked from a script.

Both modes must leave the report on disk in a readable state.

## 10. Re-running on the same ticket

Re-running is allowed and expected. Because the timestamp is part of the filename, each run produces a fresh report; previous reports are preserved for audit. Do not overwrite or delete earlier runs.

## Appendix A — Filename conventions

| Artifact | Pattern |
|---|---|
| Main report (Jira snapshot + all analysis) | `pmm-sdlc/ai-workflows/pmm-triage-bug-ticket/audit-log/Triage-report-<KEY>-<slug>-<YYYYMMDD-HHMMSS>.md` |
| Skipped (status gate) | `pmm-sdlc/ai-workflows/pmm-triage-bug-ticket/audit-log/Triage-skipped-<KEY>-<slug>-<YYYYMMDD-HHMMSS>.md` |

Where:

- `<KEY>` is the Jira issue key, e.g. `PMM-15076`.
- `<slug>` is the ticket summary lowercased, ASCII, `[a-z0-9-]` only, truncated to ≤50 chars at a word boundary.
- `<YYYYMMDD-HHMMSS>` is the operator's local time at run start.

## Appendix B — Report skeleton

A run that completes all steps produces a file with the following section order:

```markdown
# Triage report — <KEY>
<run header>

## Ticket snapshot
### Fields
### Description
### Comments
### Issue links
### Attachments
## Status gate
## Completeness
### Findings
### Labels detected
### Components detected
### Light sanity flags
## Codebase verification
## De-duplication
## Validity assessment
## Verdict
### Outcome
### Suggested Jira changes
### Confidence
### Open questions for the reporter
## Assistant uncertainty log   (only if any)
## [BLOCKED: needs-human]      (only if autonomous mode stopped early)
```

## Appendix C — Adapting this workflow

This document is intentionally a plain `.md` file so it can be:

- **Forked per repo / per project.** Pin a copy at the project root or in `docs/` and edit the allowed statuses, label list, component list, or repo scope to match your team.
- **Mirrored as a tool-specific rule.** Cursor (`.cursor/rules/*.mdc`), Claude Code (`.claude/` or `CLAUDE.md`), Continue (`.continuerules`), Aider (`CONVENTIONS.md`), etc. — copy the body, add whatever frontmatter that tool expects, keep the section numbering stable so cross-references survive.
- **Run unattended.** The autonomous-mode behavior in §0 and §9 is designed for scripted / scheduled runs; pair it with a small wrapper that resolves the ticket URL, invokes your assistant, and archives the resulting report file.
- **Hardened or relaxed.** If your team wants the assistant to also write Jira comments or change labels, lift the guardrail in §0 explicitly — do not let it drift. If your team wants stricter behavior (e.g. mandatory STR), promote a "Light sanity flag" in §4.4 into a hard gate.

Stable contract (don't break when adapting):

- The single report file path and naming pattern (Appendix A).
- The section order in Appendix B.
- The verdict outcomes in §8.1.
- The 6-way split labels in §7.

Everything else — sources to grep, label/component lists, search surfaces, time windows — is meant to be tuned to your team and product.
