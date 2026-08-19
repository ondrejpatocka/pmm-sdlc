# PMM Feature / Improvement Triage Workflow

A repeatable workflow for triaging a single PMM Jira **New Feature** or **Improvement**
ticket. Produces a local Markdown report and, on completion, posts an internal-only
(Developers-role) Jira comment summarizing the verdict and additively applies the
labels/components it proposes (never removing or replacing existing ones); never
otherwise mutates Jira, and never touches GitHub. Sibling to
`pmm-triage-bug-workflow.md` (same folder, shared skills and `triage-reports-log/`); for **Bug**
tickets use that workflow instead.

This document is intentionally tool-agnostic. It describes **what** an AI assistant (or a
human) should do at each step and **what** to write to disk — not which CLI, MCP, IDE, or
API to use.

> **Cardinal rule:** A feature/improvement is about *desired future behavior*. Never
> assume PMM lacks (or already has) the capability from the ticket text alone — verify
> against the current code and docs in §5 before judging feasibility or duplication.

## How to invoke this workflow

Point the assistant at:

1. This workflow document (as the instruction set) **plus the shared skills it references**
   under `ai-workflows/skills/` (`pmm-fetch-jira-ticket`, `pmm-arch-code-map`, `pmm-add-jira-comment`, `pmm-edit-jira-labels-components`). The runner **must
   open** them — the workflow references them without restating their content: Claude Code
   reads them on demand; in Cursor/Copilot attach or `@`-mention them; a human opens the files.
2. The Jira ticket URL to triage.
3. The local `pmm` repository checkout (and any sibling repos under the same parent directory).

**Run each ticket in a fresh session (required — see §0).** One ticket per session so a
previous ticket's context, memory, or cached data cannot bias the next:

- **Claude Code:** prefer a one-shot headless run per ticket — e.g. `claude -p "Follow pmm-triage-feature-improvement.md for <URL>"`. Interactively, run `/clear` between tickets. Do **not** use `--continue` / `--resume`.
- **Cursor / Claude Desktop / Copilot Chat:** open a **new chat** per ticket; re-attach the workflow doc + skills.
- **Scripted / scheduled runs:** a new assistant process per ticket; never reuse a session across tickets.

Whichever you use, respect the §0 guardrails and emit the Appendix B report layout.

## 0. Modes & guardrails

This workflow supports two execution modes; the steps are identical, only the uncertainty behavior differs.

- **Assisted mode (default):** A human triager drives; the assistant asks for clarification when stuck.
- **Autonomous mode:** No human in the loop; the assistant writes a `[BLOCKED: needs-human]` section to the report and stops the workflow cleanly instead of asking.

The assistant must:

- Read and act only with the access already configured on the operator's machine (Jira, GitHub, local repos). Do not require a specific tool.
- The only permitted Jira mutations are (1) posting one internal, Developers-role-restricted, triage-summary comment via the `pmm-add-jira-comment` skill (§9.7), and (2) additively applying the labels/components proposed in the TL;DR, plus the constant `ai-triage` marker label, via the `pmm-edit-jira-labels-components` skill (§9.8) — existing labels and components are always preserved, never removed or replaced. Never change priority, status, or fixVersion; never remove a label or component; never open PRs; never push commits — those stay manual, per §9.3. Everything produced (design options, effort, acceptance criteria) is a **proposal**, not an implementation.
- Write only to `pmm-sdlc/ai-workflows/pmm-triage/triage-reports-log/` inside the `pmm-sdlc` repo working tree.
- Record every irreversible recommendation (disposition, verdict, suggested Jira changes, blocked questions) in the report so the human has an audit trail.
- **Start each run stateless.** Base the triage only on this workflow document (and the `../skills/*.md` skills it references), the ticket URL, and the current state of the checked-out repos. Do **not** carry over conversation history, recalled memory, cached fetches, or earlier `triage-reports-log/` reports (those are outputs, not inputs); re-fetch Jira and re-read code **live**. Every verdict must be reproducible from the ticket + code alone.

## 1. Preconditions

Fail fast if any of these is false. Record what failed and stop.

- The Jira ticket URL is provided (example: `https://perconadev.atlassian.net/browse/PMM-15207`).
- The Jira ticket is readable by the **`pmm-fetch-jira-ticket`** skill ([`../skills/pmm-fetch-jira-ticket.md`](../skills/pmm-fetch-jira-ticket.md), applied in §2.3).
- The ticket issuetype is **New Feature** or **Improvement** (otherwise see §3).
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
# Feature/Improvement triage report — <KEY>

- Ticket: <full Jira URL>
- Issue type: New Feature | Improvement
- Session start: <YYYY-MM-DD HH:MM:SS local, UTC±HH:MM>
- Session end: <YYYY-MM-DD HH:MM:SS local, UTC±HH:MM>   <!-- backfilled at completion -->
- Session duration: <e.g. 4m 12s>                       <!-- backfilled at completion -->
- pmm branch: <branch name>
- pmm HEAD: <commit short SHA>
- Mode: assisted | autonomous
- Operator: <user@host or "autonomous">
- Assistant: <tool name + exact LLM model and version, e.g. "Claude Code / claude-opus-4-8". Always record the underlying model ID or version string, not just the tool name.>
```

`Session start` is stamped when the header is written. `Session end` and `Session duration` are **backfilled as the last write of the run** (including the `[BLOCKED: needs-human]` / skipped exits).

### 2.3 Fetch and append the Jira ticket snapshot

Fetch the ticket and build the `## Ticket snapshot` section by applying the **`pmm-fetch-jira-ticket`** skill ([`../skills/pmm-fetch-jira-ticket.md`](../skills/pmm-fetch-jira-ticket.md)) with the ticket URL (from the header) as its argument. This snapshot is the input to every analysis step that follows.

If **no** method in the skill yields usable data: in **assisted mode**, ask the operator to paste the ticket (or authorize the connector) and record the exchange; in **autonomous mode**, finish a header-only report named `<YYYYMMDD-HHMMSS>-Triage-<issueType>-<KEY>-jira-unreachable.md`, list the methods tried and why each failed, and stop. Never delete the partially-written report.

## 3. Type & status gate

This workflow triages only **New Feature** and **Improvement** tickets that are still open for triage.

- **Issue type.** Allowed: `New Feature`, `Improvement`.
  - If the type is `Bug`, write a one-line redirect report and stop:

    ```markdown
    Redirected: issuetype=Bug. Use pmm-triage-bug-workflow.md for this ticket.
    ```
  - For any other type (Task, Epic, Story, Sub-task, …), write a one-line skipped report and stop:

    ```markdown
    Skipped: issuetype=<actual>. This workflow triages only New Feature / Improvement.
    ```
- **Status.** Allowed: `New`, `Open`, `To Do`. If not allowed, write a one-line skipped report and stop:

  ```markdown
  Skipped: status=<actual status>. Triage only runs on New / Open / To Do.
  ```

  Skipped/redirect filename: `pmm-sdlc/ai-workflows/pmm-triage/triage-reports-log/<YYYYMMDD-HHMMSS>-Triage-skipped-<issueType>-<KEY>-<slug>.md`

If the type and status are allowed, append a **Gate** section recording:

- **Assignee.** If assigned to someone other than the triager, flag it (do not stop) — an in-flight request may already be in design.
- **Existing design / prior art.** Search the ticket's comments, `is implemented by` / `relates to` links, and any `github.com/percona/pmm/pull/...` or `github.com/percona/pmm/discussions/...` URLs for an existing design doc, POC, or PR. Record findings — they change the disposition and effort.

## 4. Completeness check

Evaluate whether the ticket is shaped well enough to act on. The **Completeness** section assesses it against the fields the team's `jira-feature-request` format expects.

### 4.1 Findings

- **Problem / User Story:** is the underlying *need* clear (ideally `As a <role>, I want <capability>, so that <benefit>`)? Distinguish the **problem** from the **proposed solution** — quote the story if present, or reconstruct it in one sentence and flag that it was inferred.
- **Acceptance criteria:** Present / Partial / Missing. Are they testable/observable (not "the code should…")? Quote if present.
- **Scope & Out-of-scope:** what the request explicitly includes/excludes; note unbounded scope.
- **Design / UX (only if user-facing):** are mockups/flows/affected screens described? `N/A` for backend/API-only.
- **Value & demand (compact):** who is asking (persona), how many (single request / several / many customers / strategic), and any stated business/strategic driver. Drives §9.2 priority.
- **Target persona:** DBA / SRE / developer / operator / platform admin — must be a real PMM persona.
- **Readiness verdict:** one of `Well-specified` / `Needs shaping` / `Not enough info`.

### 4.2 Labels detected

Pick zero or more Tech labels. State the evidence (quote from ticket) for each pick.

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

These are **preliminary** gut-checks, not the final call — §7 makes the authoritative disposition and §9.1 the outcome. Mark any that clearly apply, each with a one-sentence justification; don't force a fit.

- **Already exists** — the capability appears to ship today or is achievable via configuration (verify in §5).
- **Not a sensible PMM change** — wrong product, belongs to an external tool, or no credible place in this codebase.
- **Needs product decision** — acceptance depends on roadmap/strategy/policy, not engineering.
- **Actually a bug** — the "improvement" is really a defect in existing behavior (route to the bug workflow).
- **Stale or empty** — insufficient detail to act; guessing would be irresponsible.

## 5. Existing-behavior & feasibility

This is the analytical core, and the enforcement step for the cardinal rule. The goal is **not** to reproduce a defect — it is to answer *does this already exist?* and *is it feasible, and where would it be built?*, backed by real links to the current PMM source. Append an **Existing-behavior & feasibility** section.

### 5.0 Code-reference convention (used in §5, §7, §8)

Cite every code location as a **real GitHub link**, never a bare local path. Form the URL as:

```
https://github.com/<owner>/<repo>/blob/<ref>/<path>#L<startLine>-L<endLine>
```

- `<owner>` — **`percona` by default** (PMM, the Percona forks, the Percona-maintained exporters). When a capability would live upstream, link where it actually lives — `grafana/grafana`, `VictoriaMetrics/VictoriaMetrics`, `ClickHouse/ClickHouse`, `prometheus/*`, `mongodb/*`.
- `<repo>` — `pmm`, or the actual adjacent repo (`grafana`, `mongodb_exporter`, `percona-backup-mongodb`, `pmm-qa`, …).
- `<ref>` — the commit SHA from the report header (`pmm HEAD`) for `percona/pmm`, or the SHA/tag you inspected for any other repo. Prefer a SHA (stable permalink); fall back to a branch name only when no SHA is available.
- Keep the local `repo/path/to/file.go:123` in parentheses after the link so the citation stays greppable.
- **The GitHub link is the only part written as a link, not code.** Write `[<path>#L<start>-L<end>](<url>)` so the URL renders as a clickable link; never wrap the URL itself in backticks. **Every other code reference keeps its backtick formatting as before** — symbols, bare file paths, and the parenthetical local path all stay code-styled; only the URL loses its backticks.

Example: [managed/services/backup/backup_service.go#L80-L110](https://github.com/percona/pmm/blob/<HEAD-SHA>/managed/services/backup/backup_service.go#L80-L110) (`pmm/managed/services/backup/backup_service.go:80`).

### 5.1 Investigate (existence → location → feasibility)

Orient with the **`pmm-arch-code-map`** skill ([`../skills/pmm-arch-code-map.md`](../skills/pmm-arch-code-map.md)) to decide which component would own the capability, then:

1. **Already possible?** — Does PMM already provide this, in whole or part, today or via configuration? Grep the code and search `documentation/` (<https://github.com/percona/pmm/tree/main/documentation>) and the published docs (<https://docs.percona.com/percona-monitoring-and-management/>). A "yes/partly" here is the highest-value finding — it can close the ticket or reshape it into a smaller extension.
2. **Where would it live?** — Map the capability to the owning component(s)/package(s) via the §5.1 map; name the concrete files/dirs a change would start from, as §5.0 links.
3. **Feasibility & surface** — Is the change tractable on the current architecture? Note what it would touch: an API contract (`.proto`, regenerated stubs), a DB migration, the agent↔server wire protocol, the UI, or an exporter/upstream. Flag anything that makes it large or risky.

### 5.2 What to record

- **Existence status** — one of: `New capability` (nothing like it today) / `Extends existing` (builds on shipped functionality — link it) / `Already possible` (exists or config-doable — link it) / `Not locatable` (can't map to code; say so, don't guess).
- **Where it would live** — the owning component(s) + concrete entry-point files as §5.0 links.
- **Feasibility note** (≤3 sentences) — tractable? what it touches? the single biggest technical constraint.
- **References consulted** — parts of the §5.1 map used, plus `pmm/AGENTS.md` (conventions/tech-stack) and any component `AGENTS.md`, `.proto`, or docs opened.

### 5.3 Scope of search

Search every repo checked out under the parent directory of the `pmm` repo, plus any repo named in the ticket: `pmm` (<https://github.com/percona/pmm>), `pmm-qa`, the Grafana fork (`percona/grafana`), the `*_exporter` repos, and `percona-backup-mongodb`. Oversearch is acceptable; missing the relevant repo is not.

## 6. De-duplication

Append a **De-duplication** section. Features and improvements duplicate and overlap more than bugs (many ways to ask for the same capability), so search by *capability/intent*, not just wording.

### 6.1 Search surfaces (in order)

1. Open and recently-updated `PMM-*` **New Feature / Improvement** Jira tickets in the same component; check any linked **Epic** or roadmap item.
2. GitHub issues and **discussions** on `percona/pmm` (and the relevant exporter repo).
3. Upstream trackers when the capability would live upstream (Grafana, VictoriaMetrics, ClickHouse, an exporter).

### 6.2 Search rules

- Time window: open + updated within the last 6 months (features age slower than bugs). Older matches go in a "historical" subsection only if clearly the same intent.
- Cap the result list at **10 matches total**. Rank by relevance.
- Each match needs a one-line justification (why it might be the same or related intent).
- Tag each match with one relation:
  - `duplicates` — same capability/intent. Recommend closing as Duplicate of that key.
  - `superseded by` — a newer/broader ticket covers it.
  - `part of epic <KEY>` — this is a slice of a larger tracked initiative.
  - `relates to` — adjacent but distinct.
  - `unrelated — considered` — found by search, ruled out (record briefly).

## 7. Disposition

Append a **Disposition** section. Classify the ticket into exactly one of the following (the 7-way split), grounded in §4–§6 evidence and cross-checked against `.proto` contracts, `documentation/`, the §5.1 map, and any linked epic/roadmap:

- **(a) Valid — New Feature** — genuinely new capability, feasible, fits PMM's scope.
- **(b) Valid — Improvement** — enhancement to existing functionality (better UX, more options, performance, scale). Reference the existing feature.
- **(c) Already possible** — ships today or is achievable via configuration. Recommend closing with a how-to / docs pointer.
- **(d) Partially exists / extend** — part of it exists; the delta is a smaller extension. Describe the delta.
- **(e) Duplicate** — same intent as an existing ticket (from §6).
- **(f) Out of scope / Won't do** — wrong product, conflicts with product policy/direction, or not a credible PMM change.
- **(g) Needs product decision** — feasible, but acceptance depends on roadmap/strategy/prioritization, not engineering.

Cross-cutting flags (note if they apply, independent of a–g):

- **Actually a Bug → convert** — it's a defect, not an enhancement; recommend re-typing and routing to `pmm-triage-bug-workflow.md`.
- **Belongs upstream** — the capability would live in Grafana / VictoriaMetrics / ClickHouse / an exporter; PMM ticket becomes a tracker.

Include a one-paragraph justification with evidence references (GitHub links, doc URLs, ticket keys). This disposition drives the §9.1 outcome:

| §7 disposition | Default §9.1 outcome |
|---|---|
| (a) Valid — New Feature / (b) Valid — Improvement | `Ready for Refinement` (or `Ready for Dev` if small and fully specified) |
| (c) Already possible | `Already possible — close with how-to` |
| (d) Partially exists / extend | `Ready for Refinement` (scoped to the delta) |
| (e) Duplicate | `Duplicate of <KEY>` |
| (f) Out of scope / Won't do | `Won't Do — out of scope/policy` |
| (g) Needs product decision | `Needs Product Decision` |
| any, but evidence too thin | `Needs Info` |
| cross-cutting: actually a Bug | `Convert to Bug` |
| cross-cutting: belongs upstream | `Upstream tracker — <link>` |

## 8. Recommended approach

Append a **Recommended approach** section. Act as a **principal/staff engineer** turning the §4 need and §5 feasibility into a concrete, reviewable proposal for how PMM could build it. This is a proposal (see §0) — code suggestion are welcome, but no PRs, or commits. Ground the proposal in PMM's real tech stack and conventions (`pmm/AGENTS.md` plus the relevant component `AGENTS.md`) — e.g. reform migrations, `.proto` + `make gen` regeneration, testify/mockery tests — so the options are realistic to implement.

Applicability by disposition (from §7):

- **(a) / (b) / (d)** — produce the full proposal below.
- **(c) Already possible** — replace with the how-to (config/docs pointer) and stop.
- **(e) / (f) / (g)** — one line stating why no engineering proposal applies (duplicate key / out-of-scope rationale / the product question to resolve), and stop.

For dispositions that warrant a proposal, cover all five, citing code with §5.0 links:

### 8.1 Design options

The viable approach(es), as options **A / B / C** with tradeoffs (this is the engineering
version of the ticket's "Suggested implementation / options"). Give the recommended
option first with a one-line justification; name real alternatives (e.g. "extend the
existing agent" vs. "new exporter"), each with pros/cons.

### 8.2 Affected code areas

The files / packages / components the chosen option touches, each as a §5.0 GitHub link.
Distinguish the **primary** site from **secondary** sites (callers, generated code, tests,
docs). Explicitly call out contract/blast-radius surfaces: API (`.proto` + regenerated
stubs), DB migration, agent↔server wire protocol, UI.

### 8.3 Key unknowns, risks & dependencies

What is unproven and what could go wrong: backward-compatibility of an API/schema change,
client↔server version skew, supported DB versions, performance/cardinality/scale impact,
security implications, upstream dependencies, and any cross-team dependency. State what
spike or decision would retire each unknown.

### 8.4 Effort size

A single T-shirt size with a one-line justification:

- `XS` — one-file, few-line change, no contract impact.
- `S` — localized change in one component, unit-test only.
- `M` — multi-file within a component, or a contract/UI change with tests.
- `L` — cross-component (server + agent, or API + UI + tests), migration, or protocol change.
- `XL` — spans repos (e.g. PMM + exporter/upstream), or needs design sign-off first.

### 8.5 Proposed acceptance criteria

If the ticket lacks testable acceptance criteria (§4.1), author them here — bulleted,
observable, QA-verifiable (Given/When/Then or plain outcomes), including negative/edge
cases. These are a proposal for the refinement discussion, not a commitment.

## 9. Verdict

The final section, and the deliverable.

### 9.1 Outcome

Pick exactly one:

- `Ready for Refinement` — accepted in principle; needs breakdown/sizing/refinement.
- `Ready for Dev` — small and fully specified; can be picked up as-is.
- `Needs Info` — list the specific questions to ask the reporter (§9.5).
- `Needs Product Decision` — feasible; blocked on a roadmap/strategy/priority call.
- `Duplicate of <KEY>`
- `Already possible — close with how-to`
- `Won't Do — out of scope/policy`
- `Convert to Bug` — re-type and route to `pmm-triage-bug-workflow.md`.
- `Upstream tracker — <link>` — keep open as a tracker for an upstream capability.

### 9.2 Priority (Value + Effort + Demand)

A one-line recommended priority built from three compact signals — no false precision:

- **Value** — `High` / `Medium` / `Low` (+ impact on the target persona / product).
- **Effort** — the §8.4 T-shirt size.
- **Demand** — `single` / `several` / `many` customers, or `strategic`.

State the recommended priority and a one-sentence rationale, e.g. *"High value · M effort · several customers → recommend prioritising this cycle."*

### 9.3 Suggested Jira changes

A flat list of changes the triager should apply manually. Label/component **additions**
listed here are auto-applied in §9.8 (from the TL;DR's `Components / Labels:` line, not
re-parsed from this list) — everything else below (removals, classification, fixVersion,
priority, assignee) stays manual; the assistant does not apply those. Each must be actionable:

- Labels to add / remove (with justification).
- Components to add / remove (with justification).
- Classification to record (New feature / Enhancement / Tech-debt-Refactor).
- `fixVersion` / target release candidate, if known.
- Priority delta, if any (with justification).
- Suggested owner or team, if known.

### 9.4 Confidence

- `High` / `Medium` / `Low`.
- One sentence answering: "what would change my mind?"

### 9.5 Open questions for the reporter

Only populated when the outcome is `Needs Info` / `Needs Product Decision`, or confidence is `Low`. Numbered, specific, answerable.

### 9.6 TL;DR

A self-contained brief a busy technical reader (eng lead, PM, senior developer) can act on
without reading the rest. It is written **last**, after every earlier section is settled,
and closes the report as a standalone summary. Keep it to **one screen**. Each labelled line
is a distillation of an earlier finding — no new facts; add a GitHub permalink (per the
code-reference convention) for every code claim, keeping that link format unchanged, but do
not cite report section numbers here — the TL;DR must read as a standalone brief, not a
cross-referenced excerpt. For non-build outcomes (Already possible / Duplicate / Won't Do /
Needs Info / Needs Product Decision), keep the same lines but replace build-specific content
with the routing rationale.

Format: a top-level `TL;DR` line, then the labelled lines below **in this order** — each
label on its own line with its content as nested sub-bullets (one fact per sub-bullet):

- **Verdict:** the outcome and disposition, on one line — e.g. `Ready for Refinement · (d) Partially exists / extend`.
- **Request:** one sub-bullet — the user story / problem: capability, for whom, and why, with the impact tag (e.g. `(HIGH impact)`).
- **Already exists?:** lead with `Partly` / `No` / `Yes` (the existence status), then one sub-bullet per part — what already ships (with the key GitHub permalink) and what is missing.
- **Recommended approach:** sub-bullets for the chosen option — what to build/reuse, the **primary** code site (GitHub permalink), and the biggest risk/unknown. For non-build outcomes, replace with the routing rationale.
- **Priority:** three sub-bullets — **Value** (level + impact on persona/product), **Effort** (T-shirt size + one-line justification), **Demand** (`single` / `several` / `many` / `strategic`).
- **Proposed next steps / owner:** a sub-bullet naming the suggested owner/team, then the concrete next actions as a numbered list.
- **Duplicate tickets / Related tickets:** duplicate/superseded keys and related keys found during de-duplication, plus any `part of epic <KEY>`; omit a category if it has none.
- **Components / Labels:** the detected components and Tech labels, comma-separated.
- **Confidence:** the confidence level (`High` / `Medium` / `Low`) plus the one-line "what would change my mind?".
- **Open questions:** the open questions for the reporter as short numbered one-liners.

### 9.7 Post triage summary as an internal Jira comment

Build the comment text from the §9.6 TL;DR block, verbatim, then apply the
**`pmm-add-jira-comment`** skill ([`../skills/pmm-add-jira-comment.md`](../skills/pmm-add-jira-comment.md))
with the ticket URL (from the header) and that TL;DR text as arguments. Append the
skill's `## Jira comment` output section to the report.

This step runs for every run that reaches a §9.1 outcome, regardless of which outcome —
it does not run for §3 skip/redirect exits or §10 `[BLOCKED: needs-human]` stops, since
those exit before reaching §9. A failure to post (per the skill's fallback chain) does
not fail the whole report: the report still completes, with the `## Jira comment`
section documenting what happened.

### 9.8 Additively apply proposed labels/components

Take the labels and components from the §9.6 TL;DR's `Components / Labels:` line, add
the constant marker label `ai-triage` to that labels list unconditionally — every ticket
that gets the §9.7 triage-summary comment is marked this way, regardless of which Tech
labels were detected — then apply the **`pmm-edit-jira-labels-components`** skill
([`../skills/pmm-edit-jira-labels-components.md`](../skills/pmm-edit-jira-labels-components.md))
with the ticket URL, that labels list, and that components list as arguments. Append the
skill's `## Jira labels & components update` output section to the report.

This step runs under the same applicability rule as §9.7 (every run that reaches a §9.1
outcome; not for §3 skip/redirect or §10 `[BLOCKED: needs-human]` exits) — so `ai-triage`
is added on every run that posts the triage-summary comment. A failure or
partial failure to apply (per the skill's fallback chain) does not fail the whole
report: the report still completes, with the output section documenting what happened.
Existing labels and components on the ticket are never removed or replaced — only added
to.

## 10. Uncertainty & handoff

If the assistant is genuinely unsure at any step — cannot tell whether the capability already exists, cannot map it to code, cannot judge scope/feasibility, finds conflicting evidence — do **not** guess.

- **Assisted mode:** ask the human in chat. Record the question, the human's answer, and the chosen direction in an `## Assistant uncertainty log` section at the end of the report.
- **Autonomous mode:** stop the workflow. Append a `## [BLOCKED: needs-human]` section listing the question(s), what was tried, and what the assistant would need to proceed. Exit non-zero if invoked from a script.

Both modes must leave the report on disk in a readable state.

## 11. Re-running on the same ticket

Re-running is allowed and expected. Because the timestamp leads the filename, each run produces a fresh, chronologically-sorted report; previous reports are preserved for audit — do not overwrite or delete earlier runs. Since each run is stateless (§0), re-running after `main` has moved is how you confirm whether a capability has since shipped or an approach has changed.

## Appendix A — Filename conventions

| Artifact | Pattern |
|---|---|
| Main report (Jira snapshot + all analysis) | `pmm-sdlc/ai-workflows/pmm-triage/triage-reports-log/<YYYYMMDD-HHMMSS>-Triage-<issueType>-<KEY>-<slug>.md` |
| Skipped / redirected (type or status gate) | `pmm-sdlc/ai-workflows/pmm-triage/triage-reports-log/<YYYYMMDD-HHMMSS>-Triage-skipped-<issueType>-<KEY>-<slug>.md` |

Where:

- `<KEY>` is the Jira issue key, e.g. `PMM-15207`.
- `<issueType>` is the ticket's Jira issue type with spaces removed (`NewFeature`, `Improvement`, or `Bug` for a redirected ticket); `Unknown` if it could not be determined (e.g. Jira unreachable).
- `<slug>` is the ticket summary lowercased, ASCII, `[a-z0-9-]` only, truncated to ≤50 chars at a word boundary.
- `<YYYYMMDD-HHMMSS>` is the operator's local time at run start.

Reports for this workflow share the `triage-reports-log/` with the bug workflow; the report header's `Issue type` line distinguishes them, and `<KEY>` is unique per ticket.

## Appendix B — Report skeleton

A run that completes all steps produces a file with the following section order:

```markdown
# Feature/Improvement triage report — <KEY>
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
## Existing-behavior & feasibility
### Existence status
### Where it would live
### Feasibility note
### References consulted
## De-duplication
## Disposition
## Recommended approach
### Design options
### Affected code areas
### Key unknowns, risks & dependencies
### Effort size
### Proposed acceptance criteria
## Verdict
### Outcome
### Priority
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

This is the **feature/improvement** sibling of `pmm-triage-bug-workflow.md`; the two share
guardrails, the `../skills/` skills (`pmm-fetch-jira-ticket`, `pmm-arch-code-map`,
`pmm-add-jira-comment` §9.7, `pmm-edit-jira-labels-components` §9.8), and the
`triage-reports-log/`. When a ticket turns out to be a defect, the `Convert to Bug` outcome hands
off to the bug workflow.

- **Forked per repo / per project.** Edit the allowed issuetypes/statuses, the label /
  component lists, the disposition set, or the search scope to match your team. A fully
  self-contained fork should inline the referenced skills.
- **Composed from shared skills.** Reusable pieces live in `ai-workflows/skills/` and are
  referenced by name + link, not copied; the runner must open them. New PMM workflows
  should reference the same skills rather than duplicate them.
- **Hardened or relaxed.** Posting the internal triage-summary comment (§9.7) and
  additively applying proposed labels/components (§9.8) are already enabled, with the §0
  guardrail explicitly carved out for both — don't let further mutation drift in beside
  them. Removing/replacing a label or component, changing priority or status, or
  accepting more issuetypes, stay manual-only today, per §9.3; extend the §0 guardrail /
  §3 gate explicitly if your team wants to automate those too.

Stable contract (don't break when adapting):

- The report file path and naming pattern (Appendix A).
- The section order in Appendix B.
- The verdict outcomes in §9.1 and the 7-way disposition in §7.

Everything else — search surfaces, label/component lists, time windows, priority model — is meant to be tuned to your team and product.
