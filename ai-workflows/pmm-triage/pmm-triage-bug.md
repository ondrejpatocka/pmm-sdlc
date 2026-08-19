# PMM Bug Triage Workflow

A repeatable workflow for triaging a single PMM Jira bug ticket. Produces a local Markdown report and, on completion, posts an internal-only (Developers-role) Jira comment summarizing the verdict and additively applies the labels/components it proposes (never removing or replacing existing ones); never otherwise mutates Jira, and never touches GitHub.

This document is intentionally tool-agnostic. It describes **what** an AI assistant (or a human) should do at each step and **what** to write to disk — not which CLI, MCP, IDE, or API to use. Pick whichever assistant fits your environment.

> **Cardinal rule:** Never treat Jira text alone as proof the bug still exists on current `main`. Verify against the current code in §5 before drawing conclusions.

## How to invoke this workflow

You can drive this workflow with any general-purpose AI coding assistant. The operator's job is to point the assistant at:

1. This workflow document (as the instruction set) **plus the shared skills it references** under `ai-workflows/skills/` (`pmm-fetch-jira-ticket`, `pmm-arch-code-map`, `pmm-add-jira-comment`, `pmm-edit-jira-labels-components`). The runner **must open** them — the workflow references them without restating their content: Claude Code reads them on demand; in Cursor/Copilot attach or `@`-mention them; a human opens the files.
2. The Jira ticket URL to triage.
3. The local `pmm` repository checkout (and any sibling repos under the same parent directory).

Suggested ways to invoke it with the most common assistants:

- **Cursor IDE:** open this file in your workspace, then prompt: *"Follow `pmm-triage-bug-workflow.md` for ticket `<URL>`."* Optionally save a slimmer copy as a Cursor rule under `.cursor/rules/` to attach it automatically.
- **Claude (Code, Desktop, or API):** attach this file as context, then prompt: *"Use the attached workflow to triage `<URL>`."* In Claude Code, place a copy at the repo root or under `.claude/` so it's picked up by default.
- **GitHub Copilot Chat:** open this file in the editor, then in the chat panel use `#file:pmm-triage-bug-workflow.md` and prompt: *"Triage `<URL>` per this workflow."*
- **CLI / scripted runs:** pipe the workflow plus the ticket URL into your assistant of choice; the workflow's autonomous-mode behavior (see §0 and §10) is designed for unattended runs.

**Run each ticket in a fresh session (required — see the stateless guardrail in §0).** One ticket per session so a previous ticket's context, memory, or cached data cannot bias the next:

- **Claude Code:** prefer a one-shot headless run per ticket — e.g. `claude -p "Follow pmm-triage-bug-workflow.md for <URL>"` — because each `-p` invocation starts with empty conversation history. Interactively, run `/clear` between tickets. Do **not** use `--continue` / `--resume` (they restore the prior conversation). Loading `CLAUDE.md` and this workflow doc is fine — that is instruction context, not prior-run state.
- **Cursor / Claude Desktop / Copilot Chat:** open a **new chat** per ticket instead of continuing an existing thread; re-attach the workflow doc each time.
- **Scripted / scheduled runs:** invoke a new assistant process per ticket and pass only the workflow doc + ticket URL; never reuse a session/handle across tickets.

Whichever you use, respect the §0 guardrails and emit the Appendix B report layout.

## 0. Modes & guardrails

This workflow supports two execution modes; the steps are identical, only the uncertainty behavior differs.

- **Assisted mode (default):** A human triager drives; the assistant asks for clarification when stuck.
- **Autonomous mode:** No human in the loop; the assistant writes a `[BLOCKED: needs-human]` section to the report and stops the workflow cleanly instead of asking.

The assistant must:

- Read and act only with the access already configured on the operator's machine (Jira, GitHub, local repos). Do not require a specific tool.
- The only permitted Jira mutations are (1) posting one internal, Developers-role-restricted, triage-summary comment via the `pmm-add-jira-comment` skill (§9.6), and (2) additively applying the labels/components proposed in the TL;DR, plus the constant `ai-triage` marker label, via the `pmm-edit-jira-labels-components` skill (§9.7) — existing labels and components are always preserved, never removed or replaced. Never change priority, status, or fixVersion; never remove a label or component; never open PRs; never push commits — those stay manual, per §9.2.
- Write only to `pmm-sdlc/ai-workflows/pmm-triage/triage-reports-log/` inside the `pmm-sdlc` repo working tree.
- Record every irreversible decision (label/component choices, verdict, blocked questions) in the report so the human has an audit trail.
- **Start each run stateless.** Base the triage only on this workflow document (and the `../skills/*.md` skills it references), the ticket URL, and the current state of the checked-out repos. Do **not** carry over conversation history, recalled memory, cached fetches, or earlier `triage-reports-log/` reports (those are outputs, not inputs); re-fetch Jira and re-read code **live**. Every verdict must be reproducible from the ticket + code alone. (Per-tool clean-session mechanics: see the invoke section.)

## 1. Preconditions

Fail fast if any of these is false. Record what failed and stop.

- The Jira ticket URL is provided (example: `https://perconadev.atlassian.net/browse/PMM-15076`).
- The Jira ticket is readable by the **`pmm-fetch-jira-ticket`** skill ([`../skills/pmm-fetch-jira-ticket.md`](../skills/pmm-fetch-jira-ticket.md), applied in §2.3).
- The ticket issuetype is **Bug** (otherwise see §3).
- The `pmm` repo is checked out at the current `main` branch. Record the branch name and HEAD commit in the report header.
- `pmm-sdlc/ai-workflows/pmm-triage/triage-reports-log/` exists inside the `pmm-sdlc` working tree. If missing, create it.

## 2. Fetch the Jira ticket and initialize the report

This step produces the single report file used for the rest of the workflow. Every later step appends its own section to this file (Appendix B lists the section order); nothing else is written elsewhere.

### 2.1 Create the report file

Create (naming rules in Appendix A; timestamp leads so reports sort chronologically):

```
pmm-sdlc/ai-workflows/pmm-triage/triage-reports-log/<YYYYMMDD-HHMMSS>-Triage-<issueType>-<KEY>-<slug>.md
```

File requirements: UTF-8 (no BOM), LF line endings, human-readable on Windows/macOS/Linux.

### 2.2 Write the run header

Write the header immediately, before any Jira fetch or analysis, so an aborted run still leaves a useful artifact:

```markdown
# Triage report — <KEY>

- Ticket: <full Jira URL>
- Issue type: Bug
- Session start: <YYYY-MM-DD HH:MM:SS local, UTC±HH:MM>
- Session end: <YYYY-MM-DD HH:MM:SS local, UTC±HH:MM>   <!-- backfilled at completion -->
- Session duration: <e.g. 4m 12s>                       <!-- backfilled at completion -->
- pmm branch: <branch name>
- pmm HEAD: <commit short SHA>
- Mode: assisted | autonomous
- Operator: <user@host or "autonomous">
- Assistant: <tool name + exact LLM model and version, e.g. "Claude Code / claude-opus-4-8", "Cursor / claude-opus-4-7", "Copilot Chat / gpt-4o-2024-11-20", "Gemini CLI / gemini-2.5-pro". Always record the underlying model ID or version string, not just the tool name.>
```

`Session start` is stamped when the header is written. `Session end` and `Session duration` are left as placeholders here and **backfilled as the last write of the run** (in both normal completion and the `[BLOCKED: needs-human]` / skipped exits), so every report on disk records how long the triage took.

### 2.3 Fetch and append the Jira ticket snapshot

Fetch the ticket and build the `## Ticket snapshot` section by applying the **`pmm-fetch-jira-ticket`** skill ([`../skills/pmm-fetch-jira-ticket.md`](../skills/pmm-fetch-jira-ticket.md)) with the ticket URL (from the header) as its argument. This snapshot is the input to every analysis step that follows.

If **no** method in the skill yields usable data: in **assisted mode**, ask the operator to paste the ticket (or authorize the connector) and record the exchange; in **autonomous mode**, finish a header-only report named `<YYYYMMDD-HHMMSS>-Triage-<issueType>-<KEY>-jira-unreachable.md`, list the methods tried and why each failed, and stop. Never delete the partially-written report — the header alone is still useful as an audit trail.

## 3. Type & status gate

This workflow triages only **Bug** tickets that are still open for triage.

- **Issue type.** Allowed: `Bug`.
  - If the type is `New Feature` or `Improvement`, write a one-line redirect report and stop:

    ```markdown
    Redirected: issuetype=<actual>. Use pmm-triage-feature-improvement.md for this ticket.
    ```
  - For any other type (Task, Epic, Story, Sub-task, …), write a one-line skipped report and stop:

    ```markdown
    Skipped: issuetype=<actual>. This workflow triages only Bug tickets.
    ```
- **Status.** Allowed: `New`, `Open`, `To Do`. If not allowed, write a one-line skipped report and stop:

  ```markdown
  Skipped: status=<actual status>. Triage only runs on New / Open / To Do.
  ```

  Skipped/redirect filename: `pmm-sdlc/ai-workflows/pmm-triage/triage-reports-log/<YYYYMMDD-HHMMSS>-Triage-skipped-<issueType>-<KEY>-<slug>.md`

If the type and status are allowed, append a **Gate** section recording:

- **Assignee.** If the ticket is assigned to someone other than the triager / unassigned, flag it (do not stop) — triaging an in-flight ticket can step on toes.
- **Existing proposed solutions.** Search the ticket's comments, `is fixed by` / `is implemented by` links, and any `github.com/percona/pmm/pull/...` URLs for an existing fix attempt. Record findings.

## 4. Completeness check

Evaluate whether the ticket contains enough information to act. The **Completeness** section covers:

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

- `tech/MySQL` — MySQL DB monitoring and tooling.
- `tech/MongoDB` — MongoDB DB monitoring and tooling.
- `tech/PostgreSQL` — PostgreSQL DB monitoring and tooling.
- `tech/Valkey` — Valkey **or** Redis DB monitoring and tooling (one label covers both).

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
- `PMM Client` — PMM client distribution.
- `PMM Agent` — client agent component.
- `PMM Admin` — CLI component.
- `Docker` — Docker installation.
- `K8s` — Kubernetes installation.
- `HA` — High Availability installation.
- `OpenShift` — OpenShift installation.
- `AMI` — AWS installation.
- `Exporter` — `*_exporter` repos (mysqld_exporter, mongodb_exporter, postgres_exporter, etc.); the project also carries per-exporter components (`MySQLd_Exporter`, `MongoDB_Exporter`, `Postgres_Exporter`, `ProxySQL_Exporter`, `RDS_Exporter`, `Azure_Exporter`, `Node_Exporter`, `Valkey/Redis_Exporter`) — pick the specific one when the ticket clearly names a single exporter, else this generic bucket.
- `VictoriaMetrics` — VM time-series store.
- `ClickHouse` — ClickHouse storage for QAN.
- `PMM UI` — PMM frontend.
- `PMM Update Service` — upgrade / update flow.
- `Telemetry` — telemetry collection.

(`API` and `Security` were dropped from this list — no PMM Jira component matches either name exactly; note API-surface or security-relevant findings in the Findings/Rationale text instead of a component pick.)

### 4.4 Light sanity flags

These are **preliminary** gut-checks, not the final call — §7 makes the authoritative classification and §9.1 the outcome. Mark any that clearly apply (none, one, or several), each with a one-sentence justification; don't force a fit.

- **Obsolete / not relevant** — already fixed on `main`, duplicate of another ticket, feature removed, or only affects EOL versions with no supported backport path.
- **Not a bug** — works as designed, user misunderstanding, needs documentation or training instead of code.
- **Not a sensible PMM code change** — wrong product, support / config issue only, no credible change in this repo matches the report.
- **Should not be fixed** — security, compatibility, or product policy implies "Won't Fix" or "Not a bug" is the right Jira outcome.
- **Stale or empty** — insufficient detail to act; guessing would be irresponsible.

## 5. Codebase verification

This is the enforcement step for the cardinal rule, and the analytical core of the whole workflow. A shallow "I grepped the error string and it's still there" is **not acceptable**. The goal of this step is to trace the reported symptom to the **specific code that produces it** and state a **root cause**, backed by real links to the current PMM source. Everything after this step (validity, recommended fix, verdict) is only as good as the investigation here.

### 5.0 Code-reference convention (used in §5, §7, §8)

Cite every code location as a **real GitHub link**, never a bare local path. Form the URL as:

```
https://github.com/<owner>/<repo>/blob/<ref>/<path>#L<startLine>-L<endLine>
```

- `<owner>` — **`percona` by default** (PMM, the Percona forks, the Percona-maintained exporters). When a root cause sits upstream (validity class (c), §7), link where it actually lives — `grafana/grafana`, `VictoriaMetrics/VictoriaMetrics`, `ClickHouse/ClickHouse`, `prometheus/*`, `mongodb/*` — and record both the Percona-side entry point and the upstream location.
- `<repo>` — `pmm`, or the actual adjacent repo (`grafana`, `mongodb_exporter`, `percona-backup-mongodb`, `pmm-qa`, …).
- `<ref>` — the commit SHA from the report header (`pmm HEAD`) for `percona/pmm`, or the SHA/tag you inspected for any other repo. Prefer a SHA (stable permalink); fall back to a branch name only when no SHA is available.
- Keep the local `repo/path/to/file.go:123` in parentheses after the link so the citation stays greppable.
- **The GitHub link is the only part written as a link, not code.** Write `[<path>#L<start>-L<end>](<url>)` so the URL renders as a clickable link; never wrap the URL itself in backticks. **Every other code reference keeps its backtick formatting as before** — symbols, bare file paths, and the parenthetical local path all stay code-styled; only the URL loses its backticks.

Example: [managed/services/agents/channel/channel.go#L120-L138](https://github.com/percona/pmm/blob/<HEAD-SHA>/managed/services/agents/channel/channel.go#L120-L138) (`pmm/managed/services/agents/channel/channel.go:120`).

### 5.1 PMM architecture & code map (orient here first)

Orient with the **`pmm-arch-code-map`** skill ([`../skills/pmm-arch-code-map.md`](../skills/pmm-arch-code-map.md)) to decide which component owns the symptom, then cite the code you land on per §5.0.

### 5.2 Scope

Search every repo checked out under the parent directory of the `pmm` repo (a common convention is `$GITHUB_ROOT`), plus any repo explicitly named in the ticket. Oversearch is acceptable; missing the relevant repo is not. Typical hits, with their GitHub homes:

- `pmm` — <https://github.com/percona/pmm> (sub-areas map to the §5.1 component map: `managed/`, `agent/`, `admin/`, `api/`, `qan-api2/`, `vmproxy/`, `ui/`, `build/`, `api-tests/`, `dashboards/`).
- `pmm-qa` — <https://github.com/percona/pmm-qa>.
- `grafana` (Percona fork) — <https://github.com/percona/grafana>.
- `*_exporter` repos when present — e.g. <https://github.com/percona/mongodb_exporter>, <https://github.com/percona/mysqld_exporter>, <https://github.com/percona/postgres_exporter>.
- `percona-backup-mongodb` (PBM) and similar adjacent repos when present — <https://github.com/percona/percona-backup-mongodb>.

### 5.3 How to investigate (drive to root cause, do not stop at the symptom)

Do not stop at the first line that mentions the symptom. Follow the call path until you can explain *why* the reported behavior happens:

1. **Locate the symptom surface** — the exact error string / log line / metric name / panel title / endpoint quoted in the ticket. Grep the repos for it.
2. **Trace inward** — from that surface, follow the call chain (handler → service → store → agent RPC, or UI component → API client → gRPC method) using the §5.1 component map. Note each hop as a GitHub link.
3. **Cross the boundary when needed** — if the symptom originates in the API contract, read the relevant `.proto` under `api/` (<https://github.com/percona/pmm/tree/main/api>); if it originates in data collection, cross into the exporter or agent repo; if in presentation, into `dashboards/` or `ui/`.
4. **Confirm the mechanism** — identify the precise construct responsible (a wrong condition, a missing nil-check, an incorrect query, a version guard, a default value, a race). State it in one sentence.
5. **Check the reproduction reality** — does the current code path still allow the reported inputs to reach the faulty construct? Cross-check version guards and any tests under `api-tests/` (<https://github.com/percona/pmm/tree/main/api-tests>) that pin current behavior.

### 5.4 What to record

- **Root cause** — one to three sentences naming the responsible code construct and mechanism (not just "the bug is in QAN"). If a root cause genuinely cannot be pinned, say so explicitly and mark the symptom area as unclear rather than guessing.
- **Evidence trail** — the ordered list of GitHub links (per §5.0) walked from symptom surface to root cause, each with a one-line note on what that code does.
- **References consulted** — which parts of the §5.1 map you used to orient, plus any component `AGENTS.md` or docs you opened for deeper detail.
- **Status on `main`** — one of: `Still present on main` / `Appears fixed on main` / `Not located — symptom area unclear`.
- **Rationale** (≤3 sentences) — what the code does today vs. what the ticket reports.

## 6. De-duplication

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

The **Validity assessment**: just because a user does not like how something works does not mean it's a bug.

Cross-check sources, in priority order, when deciding "as designed". Cite each as a GitHub link per the §5.0 convention:

1. API contract: `.proto` files under `api/` — <https://github.com/percona/pmm/tree/main/api> (source of truth for behavior).
2. User docs: `documentation/` (MkDocs) — <https://github.com/percona/pmm/tree/main/documentation>, published at <https://docs.percona.com/percona-monitoring-and-management/>.
3. The §5.1 component map (and, for deeper detail, each component's in-repo `AGENTS.md`) for architectural intent.
4. Original feature ticket or acceptance criteria, when linked.
5. Existing tests under `api-tests/` — <https://github.com/percona/pmm/tree/main/api-tests>. If a test already asserts the current behavior, that's a strong "as designed" signal.

Classify the ticket into exactly one of the following (the 6-way split):

- **(a) Defect** — broken in the current release, never worked. Flag as a Bug.
- **(b) Regression** — worked in a previous GA, broken now. Flag as a Bug; record the last known working version in the report (Jira's `Affects version` semantics vary by project — do not change Jira here, just record).
- **(c) Upstream defect** — root cause is in Grafana / VictoriaMetrics / ClickHouse / PBM / an exporter. Recommend the `Upstream` label and a link to the upstream issue.
- **(d) Documentation gap** — system behaves as designed, but the docs disagree or are silent. Recommend re-routing to the `Documentation` component.
- **(e) Configuration / environment issue** — user ran client newer than server, unsupported DB version, exposed a DB the docs say isn't supported, hit a known limit (cardinality, retention). Recommend "Not a bug" with a link to the docs page.
- **(f) Feature request / enhancement** — works as designed; the user wants different behavior. Recommend re-typing the ticket as a feature request.

Include a one-paragraph justification with evidence references (GitHub links to proto symbols / code / tests, doc URLs).

This class drives the §9.1 outcome (so it isn't re-reasoned there):

| §7 class | Default §9.1 outcome |
|---|---|
| (a) Defect / (b) Regression | `Ready for Dev` (or `Needs Info` if under-specified) |
| (c) Upstream defect | `Upstream tracker — <link>` |
| (d) Documentation gap | `Documentation` |
| (e) Configuration / environment | `Won't Fix — works as designed` (or `Needs Info`) |
| (f) Feature request | `Won't Fix — out of scope` (re-type as feature) |
| any class, but evidence too thin to act | `Needs Info` / `Insufficient info — escalate to human triager` |

Independently, if §6 found a same-root-cause match, the outcome is `Duplicate of <KEY>` regardless of class.

## 8. Recommended fix

Here you switch hats: act as a **principal engineer** turning the §5 root cause into a concrete, reviewable fix proposal. This is a proposal (see §0) — code suggestion are welcome, but no PRs, or commits.

Applicability by validity class (from §7):

- **(a) Defect / (b) Regression** — produce the full proposal below.
- **(c) Upstream defect** — propose the PMM-side change *if* a mitigation, version guard, or workaround belongs in `percona/*`; otherwise state that the fix lives upstream, link the upstream location, and note any PMM-side tracking change.
- **(d) Documentation gap** — the "fix" is a docs change; point at the `documentation/` page and section, keep the rest light.
- **(e) Configuration / environment / (f) Feature request** — no code fix; write one line stating why and stop.

For classes that warrant a proposal, cover all five of the following. Ground every claim in the §5 evidence and cite code with §5.0 GitHub links:

### 8.1 Relevant code areas

The specific files / packages / functions the fix touches, each as a GitHub link (repo + `<ref>` + `#L` range). Distinguish the **primary** site (where the root cause lives) from **secondary** sites (callers, tests, generated code, docs) that must change alongside it.

### 8.2 Fix surface

A concise description of *what* changes and the shape of the change — the approach, not a full patch. Note whether it touches an API contract (`.proto`, regenerated stubs), a DB migration, the agent↔server wire protocol, or the UI, since those widen the blast radius. If more than one viable approach exists, give the recommended one first and name the alternative in a sentence.

### 8.3 Key unknowns & risks

What is still unproven and what could go wrong: behavior not yet reproduced, version/compat concerns (client↔server skew, supported DB versions), backward-compatibility of an API or schema change, performance/cardinality impact, and any upstream dependency. State what test or check would retire each unknown.

### 8.4 Relative effort size

A single T-shirt size with a one-line justification:

- `XS` — one-file, few-line change, no contract impact.
- `S` — localized change in one component, unit-test only.
- `M` — multi-file within a component, or a contract/UI change with tests.
- `L` — cross-component (server + agent, or API + UI + tests), migration, or protocol change.
- `XL` — spans repos (e.g. PMM + exporter/upstream), or needs design sign-off first.

### 8.5 Suggested verification

The test(s) or manual reproduction that would prove the fix works — ideally a new/updated test under `api-tests/` or the component's own test suite, named concretely.

## 9. Verdict

The final section, and the deliverable.

### 9.1 Outcome

Pick exactly one:

- `Ready for Dev`
- `Needs Info` — list the specific questions to ask the reporter.
- `Duplicate of <KEY>`
- `Won't Fix — works as designed`
- `Won't Fix — out of scope`
- `Documentation` — route to the docs team.
- `Upstream tracker — <link>` — keep open as a tracker for an upstream fix.
- `Insufficient info — escalate to human triager`

### 9.2 Suggested Jira changes

A flat list of changes the triager should apply manually. Label/component **additions**
listed here are auto-applied in §9.7 (from the TL;DR's `Components / Labels:` line, not
re-parsed from this list) — everything else below (removals, priority, fixVersion,
assignee) stays manual; the assistant does not apply those. Each item must be actionable:

- Labels to add / remove (with justification).
- Components to add / remove (with justification).
- Priority delta, if any (with justification).
- `fixVersion` candidate, if known.
- Suggested assignee or team, if known.

### 9.3 Confidence

- `High` / `Medium` / `Low`.
- One sentence answering: "what would change my mind?"

### 9.4 Open questions for the reporter

Only populated when the outcome is `Needs Info` or confidence is `Low`. Numbered, specific, answerable.

### 9.5 TL;DR

The most-read part of the report: a self-contained brief a busy technical reader (EM, PM, Senior developer) can act on without reading the rest. Write it **last**, after every earlier section is settled; it closes the report as a standalone summary. Keep it to **one screen** — each bullet is an up to four-sentence distillation of an earlier finding, no new facts; add a GitHub permalink (per the code-reference convention) for every code claim, but do not cite report section numbers here — the TL;DR must read as a standalone brief, not a cross-referenced excerpt.

Format: a top-level `TL;DR` line, then the labelled lines below **in this order** — each
label on its own line with its content as nested sub-bullets (one fact per sub-bullet):

- **Verdict:** the outcome and classification, on one line — e.g. `Ready for Dev · (a) Defect`.
- **Problem summary:** one sub-bullet — what breaks, for whom, and under what trigger — the observable symptom in the user's terms.
- **Impact / scope:** multiple sub bullets — priority and customer impact (single report vs. multiple customers vs. theoretical), the affected component(s) and Tech label, affected version(s), and the reproducibility verdict.
- **Root cause:** the responsible code construct and mechanism, with the key GitHub permalink and status on `main` (`still present` / `fixed`). For non-code outcomes, state the equivalent finding (e.g. "duplicate of PMM-XXXXX", "behaves as designed per <doc>").
- **Recommended fix:** sub-bullets for the recommended approach — the primary code area, the effort size, and risks/unknowns. For non-code outcomes, replace with the routing rationale.
- **Proposed next steps / owner:** a sub-bullet naming the suggested owner/team, then the concrete next actions as a numbered list.
- **Duplicate tickets / Related tickets:** duplicate/superseded keys and related keys found during de-duplication.
- **Components / Labels:** the detected components and Tech labels, comma-separated.
- **Confidence:** the confidence level (`High` / `Medium` / `Low`) plus the one-line "what would change my mind?".
- **Open questions:** the open questions for the reporter as short numbered one-liners.

### 9.6 Post triage summary as an internal Jira comment

Build the comment text from the §9.5 TL;DR block, then append a one-line provenance
footer:

```
Automated triage (pmm-triage-bug.md, <tool>/<exact model ID>), verified against
pmm@<short SHA>[, <adjacent-repo>@<short SHA> …]. Proposal only — no code changes made.
```

using the `Assistant` value and repo SHA(s) established in §2.2. The leading
`pmm-triage-bug.md` tag names the workflow document that produced this comment,
whichever tool ran it — this doc has no "plugin" concept of its own and is run by
directly attaching the file, so the document filename is the stable identifier, letting
a reader tell this apart from a comment posted by the `pmm-jira-triage` or `pmm-dev`
Claude Code plugins.

Then apply the **`pmm-add-jira-comment`** skill ([`../skills/pmm-add-jira-comment.md`](../skills/pmm-add-jira-comment.md))
with the ticket URL (from the header) and the TL;DR-plus-footer text as arguments. Append
the skill's `## Jira comment` output section to the report.

This step runs for every run that reaches a §9.1 outcome, regardless of which outcome —
it does not run for §3 skip/redirect exits or §10 `[BLOCKED: needs-human]` stops, since
those exit before reaching §9. A failure to post (per the skill's fallback chain) does
not fail the whole report: the report still completes, with the `## Jira comment`
section documenting what happened.

### 9.7 Additively apply proposed labels/components

Take the labels and components from the §9.5 TL;DR's `Components / Labels:` line, add
the constant marker label `ai-triage` to that labels list unconditionally — every ticket
that gets the §9.6 triage-summary comment is marked this way, regardless of which Tech
labels were detected — then apply the **`pmm-edit-jira-labels-components`** skill
([`../skills/pmm-edit-jira-labels-components.md`](../skills/pmm-edit-jira-labels-components.md))
with the ticket URL, that labels list, and that components list as arguments. Append the
skill's `## Jira labels & components update` output section to the report.

This step runs under the same applicability rule as §9.6 (every run that reaches a §9.1
outcome; not for §3 skip/redirect or §10 `[BLOCKED: needs-human]` exits) — so `ai-triage`
is added on every run that posts the triage-summary comment. A failure or
partial failure to apply (per the skill's fallback chain) does not fail the whole
report: the report still completes, with the output section documenting what happened.
Existing labels and components on the ticket are never removed or replaced — only added
to.

## 10. Uncertainty & handoff

If the assistant is genuinely unsure at any step — cannot disambiguate between two components, cannot reproduce from the description, cannot locate the symptom area in code, finds conflicting evidence — do **not** guess.

- **Assisted mode:** ask the human in chat. Record the question, the human's answer, and the chosen direction in an `## Assistant uncertainty log` section at the end of the report.
- **Autonomous mode:** stop the workflow. Append a `## [BLOCKED: needs-human]` section listing the question(s), what was tried, and what the assistant would need to proceed. Exit non-zero if invoked from a script.

Both modes must leave the report on disk in a readable state.

## 11. Re-running on the same ticket

Re-running is allowed and expected. Because the timestamp leads the filename, each run produces a fresh, chronologically-sorted report; previous reports are preserved for audit — do not overwrite or delete earlier runs. Since each run is stateless (§0), re-running the same ticket after `main` has moved is how you confirm whether a bug is still present or now fixed.

## Appendix A — Filename conventions

| Artifact | Pattern |
|---|---|
| Main report (Jira snapshot + all analysis) | `pmm-sdlc/ai-workflows/pmm-triage/triage-reports-log/<YYYYMMDD-HHMMSS>-Triage-<issueType>-<KEY>-<slug>.md` |
| Skipped / redirected (type or status gate) | `pmm-sdlc/ai-workflows/pmm-triage/triage-reports-log/<YYYYMMDD-HHMMSS>-Triage-skipped-<issueType>-<KEY>-<slug>.md` |

Where:

- `<KEY>` is the Jira issue key, e.g. `PMM-15076`.
- `<issueType>` is the ticket's Jira issue type with spaces removed (`Bug`, `NewFeature`, `Improvement`); `Unknown` if it could not be determined (e.g. Jira unreachable).
- `<slug>` is the ticket summary lowercased, ASCII, `[a-z0-9-]` only, truncated to ≤50 chars at a word boundary.
- `<YYYYMMDD-HHMMSS>` is the operator's local time at run start.

## Appendix B — Report skeleton

A run that completes all steps produces a file with the following section order:

```markdown
# Triage report — <KEY> - <slug>
<run header>

## Ticket snapshot
### Fields
### Description
### Comments
### Issue links
### Attachments
## Gate
## Completeness
### Findings
### Labels detected
### Components detected
### Light sanity flags
## Codebase verification
### Root cause
### Evidence trail
### References consulted
## De-duplication
## Validity assessment
## Recommended fix
### Relevant code areas
### Fix surface
### Key unknowns & risks
### Relative effort size
### Suggested verification
## Verdict
### Outcome
### Suggested Jira changes
### Confidence
### Open questions for the reporter
### TL;DR
### Jira comment
### Jira labels & components update
## Assistant uncertainty log   (only if any)
## [BLOCKED: needs-human]      (only if autonomous mode stopped early)
```

## Appendix C — Adapting this workflow

This document is intentionally a plain `.md` file so it can be:

- **Forked per repo / per project.** Pin a copy at the project root or in `docs/` and edit the allowed statuses, label list, component list, or repo scope to match your team. This workflow references shared skills (see below), so a fully self-contained fork should inline them.
- **Composed from shared skills.** Reusable, workflow-agnostic pieces live in `ai-workflows/skills/` as skill-formatted `.md` files (YAML frontmatter + `argument-hint`) and are referenced by name + relative link, not copied — currently [`pmm-fetch-jira-ticket`](../skills/pmm-fetch-jira-ticket.md) (§2.3), [`pmm-arch-code-map`](../skills/pmm-arch-code-map.md) (§5.1), [`pmm-add-jira-comment`](../skills/pmm-add-jira-comment.md) (§9.6), and [`pmm-edit-jira-labels-components`](../skills/pmm-edit-jira-labels-components.md) (§9.7). Referencing sections point to the skill rather than restate it (single source of truth), so **the runner must open the linked skill** (see the invoke section). They are *not* auto-invoked — they live outside `.claude/skills/`; promote a copy there if you want Claude to auto-load it. New PMM workflows (e.g. a feature/enhancement triage) should reference the same skills. Edit shared behavior in the skill, workflow-specific behavior in the section; keep skills free of any one workflow's specifics (report layout, verdict labels, modes).
- **Mirrored as a tool-specific rule.** Cursor (`.cursor/rules/*.mdc`), Claude Code (`.claude/` or `CLAUDE.md`), Continue (`.continuerules`), Aider (`CONVENTIONS.md`), etc. — copy the body (inlining the skills), add whatever frontmatter that tool expects, keep the section numbering stable so cross-references survive.
- **Run unattended.** The autonomous-mode behavior in §0 and §10 is designed for scripted / scheduled runs; pair it with a small wrapper that resolves the ticket URL, invokes your assistant, and archives the resulting report file.
- **Hardened or relaxed.** Posting the internal triage-summary comment (§9.6) and additively applying proposed labels/components (§9.7) are already enabled, with the §0 guardrail explicitly carved out for both — don't let further mutation drift in beside them. Removing/replacing a label or component, and changing priority, status, or fixVersion, stay manual-only today, per §9.2; extend the §0 exception explicitly if your team wants to automate those too. If your team wants stricter behavior (e.g. mandatory STR), promote a "Light sanity flag" in §4.4 into a hard gate.

Stable contract (don't break when adapting):

- The single report file path and naming pattern (Appendix A).
- The section order in Appendix B.
- The verdict outcomes in §9.1.
- The 6-way split labels in §7.

Everything else — sources to grep, label/component lists, search surfaces, time windows — is meant to be tuned to your team and product.
