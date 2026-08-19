# Porting PMM triage to a `pmm-ai` Claude Code plugin — analysis & plan

Target: turn `pmm-triage-bug.md` and `pmm-triage-feature-improvement.md` into an installable
Claude Code **plugin** on `percona/pmm-ai`, so any teammate gets the same triage behavior from
the local CLI, the IDE extension (VS Code/JetBrains), or a Claude Code Web session — without
copy-pasting workflow docs or re-attaching files by hand every time.

Status: **plan / not yet implemented**. Written 2026-08-18, informed by (a) the two source
workflow docs and their four shared skills in this repo, (b) `percona/pmm-ai`'s existing
`pmm-docs` plugin (merged, in production use), (c) an **unmerged** WIP branch
`add-pmm-dev-plugin` on that same repo, and (d) Anthropic's current plugin docs (verified live —
see §3). Sibling to [`routines-port-plan.md`](routines-port-plan.md), which covers a different,
larger, not-yet-started effort (fully unattended scheduled triage on Claude Code Routines) — see
§7 for how the two relate.

Two decisions below were made by the operator up front and are treated as settled, not
re-litigated in this document:

1. **Naming**: ship as an independently-named plugin, not literally `pmm-triage`, to avoid
   confusion with an unrelated skill of that name already in flight elsewhere (§2).
2. **Report persistence**: the internal Jira comment becomes the sole durable audit trail; the
   plugin does not write a Markdown report file to disk (§5).

---

## 1. Recommendation in one paragraph

Ship a new plugin, **`pmm-jira-triage`**, under `plugins/pmm-jira-triage/` in `percona/pmm-ai`,
with **three** skills: a single auto-invocable entry point, `pmm-triage`, that takes just a
ticket key/URL, fetches it, reads its `issuetype`, and dispatches to whichever of the two
faithfully-ported workflow skills applies — `pmm-triage-bug` or `pmm-triage-feature` — which
remain available for direct/manual invocation when the type is already known (§4). The two
workflow docs are adapted in exactly two ways beyond that dispatch layer: (a) the four shared
skills move from free-floating `../skills/*.md` references into the plugin's own `references/`
directory, linked via `${CLAUDE_PLUGIN_ROOT}` instead of relative paths (so the plugin is portable
once installed — see §4), and (b) every step that today writes to `triage-reports-log/` instead
produces its output as the skill's final chat response, with the existing Jira-comment provenance
footer (already an accepted deviation — see `pmm-triage-jira-comment-footer` memory) doing double
duty as the durable audit record. Everything else — the six/seven-way classification, the
de-duplication search, the curated label/component vocabulary, the GitHub-permalink code-citation
convention, the TL;DR structure — ports unchanged, because none of it depends on the filesystem.
This keeps the port small, low-risk, and reviewable as two PRs (§8): one against `pmm-ai`, one
against `percona/pmm` to wire up team-wide auto-enable.

---

## 2. Why not just call it `pmm-triage`

`percona/pmm-ai` already has an **unmerged** branch, `add-pmm-dev-plugin` (theTibi, with Alex
Demidoff also committing; last touched 2026-08-13), that adds a `pmm-dev` plugin with a skill
named `pmm-triage`, invoked as `/pmm-dev:pmm-triage`. It is a genuinely different tool wearing a
confusingly similar name:

| | This port (`pmm-jira-triage`) | `pmm-dev:pmm-triage` (WIP) |
|---|---|---|
| Deliverable | Full structured report (chat) + one Jira comment | One short (~6–12 line) Jira comment only |
| Classification | 6-way (bug) / 7-way (feature) with a justified verdict | bug / feature / not-actionable |
| De-duplication | Explicit search across Jira + GitHub + upstream trackers, ranked, tagged | Not present |
| Marker label | `ai-triage` | `ai-reviewed` |
| Touches the code repo? | Never — read-only on every repo, always | Read-only, **except** it may add and delete one throwaway repro test file |
| Ends with | A verdict + posted comment, then stops | The comment, then stops — but is designed to hand off into two more skills (`pmm-fix`, `pmm-feature`) that branch, implement, and open PRs |
| Report persistence | None (Jira comment is the record) | None |

Both independently converged on the same good idea (a model-attribution footer on the Jira
comment) and the same rule (additive-only labels/components, Developers-role visibility) — which
is reassuring, but doesn't change the fact that a teammate typing `/pmm-…:pmm-triage PMM-15076`
has a 50/50 shot at the wrong tool. Renaming this plugin — recommended: **`pmm-jira-triage`** —
sidesteps the collision without blocking on someone else's unfinished branch. (It's a cheap
choice to revisit: it's a folder name and a `plugin.json` field, not a public contract, until the
PR merges.) `pmm-fix` and `pmm-feature` are out of scope here regardless of naming — this port
never implements code, matching the two source workflows' cardinal "never touches GitHub"
guardrail.

Worth noting for later, not now (§9): that branch hasn't even added itself to
`.claude-plugin/marketplace.json` yet, so it isn't installable via the documented `/plugin
install` flow today — it's genuinely early. A short cross-reference in this plugin's README
(§6) is enough for now; a real reconciliation conversation (e.g. should the two share the
`references/jira-atlassian.md`-equivalent content, should the label vocabularies converge) is a
follow-up, not a blocker for this PR.

---

## 3. What actually changes vs. what's assumed — verified against current docs

The previous porting analysis (`routines-port-plan.md`) was written for Claude Code **Routines**
(unattended, scheduled, cloud-only). This one targets the **plugin/skill system** used
interactively from three surfaces, which is a different set of mechanics. Three claims below were
checked directly against `code.claude.com/docs` rather than assumed, because they're load-bearing
and the pmm-ai README's own rationale depends on them:

| Claim | Verified | Detail |
|---|---|---|
| A repo can commit `.claude/settings.json` with `extraKnownMarketplaces` + `enabledPlugins` so a plugin is available with no interactive `/plugin` step | **Already proven in production**, not just docs | `pmm-docs` ships exactly this way today (see `pmm-ai/README.md`), which is stronger evidence than the docs page, which doesn't spell out cross-surface behavior explicitly. |
| The interactive `/plugin` command doesn't work in a Claude Code Web session | **Confirmed** | `claude-code-on-the-web.md`: "Commands that only run in the terminal interface, such as `/plugin` or `/resume`, aren't available." This is exactly why `pmm-ai`'s README pushes enablement into committed `settings.json` instead of asking each teammate to run `/plugin install` themselves on the web. |
| A skill's `SKILL.md` can reference another file elsewhere **in the same plugin** and have Claude read it, without copying the file into every skill's own directory | **Confirmed, with one caveat** | `plugins-reference.md`: `${CLAUDE_PLUGIN_ROOT}` resolves "anywhere the placeholder appears" in skill/agent content. Caveat: "Copied plugins cannot reference files outside their directory" — paths that traverse **outside the plugin root** (`../something`) break after a marketplace install, because only the plugin's own tree gets copied into the cache. Referencing a **sibling path inside the same plugin** via `${CLAUDE_PLUGIN_ROOT}/references/foo.md` is exactly the supported case. |

That third point is the one architectural correction worth calling out: the WIP `pmm-dev`
branch assumed skills "must be self-contained — there is no mechanism for one skill to read
another's" and built a `scripts/build.sh` that materializes copies of shared reference docs into
each skill's own directory before every commit, specifically to avoid drift. That workaround is
real and justified for *their* stated secondary goal — also publishing each skill as a standalone
zip for `claude.ai → Settings → Skills → Shared` upload, which genuinely has no plugin context and
thus no `${CLAUDE_PLUGIN_ROOT}`. We don't have that secondary goal here (the ask is IDE + Web via
the `pmm-ai` marketplace, not standalone skill uploads), so we can skip the copy-and-regenerate
step entirely: one shared `references/` directory at the plugin root, real links, no build step,
no drift by construction. If a future need for standalone skill uploads shows up, adopting the
same materialize-on-build pattern is a small, isolated addition — not a rearchitecture.

---

## 4. Plugin layout

```
plugins/pmm-jira-triage/
├── .claude-plugin/
│   └── plugin.json
├── README.md
├── references/                              # the four shared skills — single source of truth
│   ├── pmm-fetch-jira-ticket.md             #   ported verbatim from ai-workflows/skills/
│   ├── pmm-arch-code-map.md                 #   (frontmatter already skill-shaped — see §6)
│   ├── pmm-add-jira-comment.md
│   └── pmm-edit-jira-labels-components.md
└── skills/
    ├── pmm-triage/                           # NEW — auto-detecting entry point, see below
    │   └── SKILL.md
    ├── pmm-triage-bug/
    │   └── SKILL.md                          # ported from pmm-triage-bug.md
    └── pmm-triage-feature/
        └── SKILL.md                          # ported from pmm-triage-feature-improvement.md
```

Both workflow `SKILL.md` files link into `references/` as
`${CLAUDE_PLUGIN_ROOT}/references/<file>.md` — e.g. where `pmm-triage-bug.md` today says `applying
the **pmm-fetch-jira-ticket** skill
([../skills/pmm-fetch-jira-ticket.md](../skills/pmm-fetch-jira-ticket.md))`, the ported skill says
`applying the **pmm-fetch-jira-ticket** skill
(${CLAUDE_PLUGIN_ROOT}/references/pmm-fetch-jira-ticket.md)`. The four reference files themselves
need no content changes — they're already tool-agnostic and already carry skill-shaped
frontmatter (`name`, `description`, `argument-hint`); only their consumers' link syntax changes.

### Auto-detecting the issue type: `pmm-triage` as the single entry point

This is the piece added in this revision. One clarification on invocation syntax first: Claude
Code's slash-command form is `/<plugin>:<skill-name> <arguments>` — the segment after `:` names a
skill or command, it can't be the argument itself, so the literal form is
`/pmm-jira-triage:pmm-triage PMM-15076`, not `/pmm-jira-triage:PMM-15076`.

**Auto-invocation must be gated on explicit triage intent, not on a ticket merely appearing in
the chat.** This matters more than UX polish: reaching a §9.1 outcome ends in a real write —
an internal Jira comment plus additive labels/components (§0's Jira-mutation guardrail, ported
unchanged). A skill's `description` is the *only* lever Claude uses to decide when to
auto-invoke it, so the gating has to be written into that description, not bolted on elsewhere.
`pmm-triage`'s description must say, in effect: fire only when the user expresses clear
triage/investigate intent *together with* a ticket reference (e.g. "triage PMM-15076",
"investigate this ticket", "run triage on `<url>`") — and explicitly **not** just because a
`PMM-####` key or a `perconadev.atlassian.net` URL shows up in the conversation on its own (a
ticket pasted for context, referenced in passing during an unrelated discussion, or quoted from
another tool's output). A stray mention should never silently kick off a workflow that ends by
writing to Jira.

`pmm-triage`'s job is intentionally small — fetch once, read the type, hand off:

```markdown
---
name: pmm-triage
description: Auto-detects whether a PMM Jira ticket (PMM-####) is a Bug or a New Feature/
  Improvement and runs the matching triage workflow, ending in an internal Jira comment plus
  labels. Invoke ONLY when the user expresses clear triage/investigation intent together with a
  ticket reference — e.g. "triage PMM-15076", "investigate this ticket", "run triage on
  <url>", "do a full triage of PMM-15076". Do NOT invoke just because a PMM-#### key or a
  perconadev.atlassian.net URL appears in the conversation on its own — e.g. someone pasting,
  discussing, or referencing a ticket without asking for triage. If you already know the issue
  type, pmm-triage-bug / pmm-triage-feature can be invoked directly instead.
argument-hint: "<Jira ticket URL or key, e.g. PMM-15076>"
---
```

1. Extract the issue key from the argument (URL or bare key).
2. Fetch the ticket **once**, by applying
   `${CLAUDE_PLUGIN_ROOT}/references/pmm-fetch-jira-ticket.md` — this builds the `## Ticket
   snapshot` that both workflows need for their own §2.3.
3. Read the snapshot's `issuetype` and branch:
   - `Bug` → open and follow `${CLAUDE_PLUGIN_ROOT}/skills/pmm-triage-bug/SKILL.md`, starting at
     its §3 (type & status gate) — its §1/§2 are already satisfied by steps 1–2 above; **do not
     fetch the ticket again**.
   - `New Feature` or `Improvement` → same, via
     `${CLAUDE_PLUGIN_ROOT}/skills/pmm-triage-feature/SKILL.md`.
   - Anything else (Task, Epic, Story, Sub-task, unreadable) → skip straight to that workflow's
     one-line "Skipped: issuetype=…" output; there's no reason to open either full file for a type
     neither one accepts.

**Skill naming and auto-invocation.** Only `pmm-triage` is left auto-invocable. `pmm-triage-bug`
and `pmm-triage-feature` get `disable-model-invocation: true` — they become manual/delegate-only,
invoked explicitly (`/pmm-jira-triage:pmm-triage-bug PMM-15076`) by someone who's already looked
at the ticket and wants to skip detection, or reached via step 3 above. This is a deliberate
change from treating all skills as equally auto-invocable (which is the risk-based split the WIP
`pmm-dev` branch uses for its own read-only-vs-mutating skills): with three skills that all
plausibly match "triage this PMM ticket," leaving all three auto-invocable would make Claude's
choice among them a coin flip. Routing that ambiguity through one lightweight, always-correct
dispatcher instead of hoping the model picks well removes the guesswork entirely, and it's cheap —
`pmm-triage` only ever does one extra fetch (already needed downstream anyway) before handing off.
The existing §3 type/status gate in each workflow skill stays as a safety net for the direct/manual
path — if someone forces `pmm-triage-bug` on a ticket that turns out to be a Feature, it still
redirects cheaply, now naming `pmm-triage` as the easier route next time (§5).

---

## 5. Content adaptation — what changes section by section

The only structural change is: **every "write to `triage-reports-log/`" instruction becomes
"this is part of the final chat response."** Everything analytical is untouched. Concretely, per
section of `pmm-triage-bug.md` (identical treatment applies to
`pmm-triage-feature-improvement.md`, substituting its own section numbers):

| Section | Change |
|---|---|
| Frontmatter (new) | Add `name`, `description` (with explicit trigger phrases, e.g. "triage this PMM bug", a pasted `PMM-####` bug URL), `argument-hint: "<Jira Bug ticket URL or key>"`. |
| "How to invoke" | Replace tool-specific instructions (Cursor/Copilot/headless-CLI attach dance) with: install the plugin once, then say "triage PMM-15076" (or `/pmm-jira-triage:pmm-triage PMM-15076`) — `pmm-triage` auto-detects the type and routes for you (§4). Be explicit that *just pasting* a ticket URL/key with no triage/investigate verb is intentionally **not** enough to auto-fire it (§4) — say what you want done, not just what ticket you mean. Mention the direct form (`/pmm-jira-triage:pmm-triage-bug PMM-15076`) as the explicit-type shortcut, not the default. The "fresh session per ticket" guardrail (§0) is unchanged and still the operator's job — a plugin doesn't enforce that automatically in an interactive session, just as the doc doesn't today. |
| §0 Guardrails | Drop "write only to `triage-reports-log/`". Add "never write any file into the target code repo working tree" (the plugin is stricter here than before — no report file at all now, not even a gitignored one). Jira-mutation guardrail (comment + additive labels only) is unchanged. |
| §1 Preconditions | Drop the `triage-reports-log/` existence check. Ticket URL / issuetype / `pmm` checkout-at-`main` checks are unchanged. |
| §2 "Fetch ticket and initialize the report" | Rename to "Fetch the ticket". Drop §2.1 (create report file) and its Appendix A filename convention entirely. §2.2's header fields (session start, `pmm` branch/HEAD, mode, operator, assistant/model) stop being *written* anywhere and become facts the skill **carries through to the end** of the run, surfacing in the provenance footer (see next row) — nothing is lost, it just moves from a file header to the Jira comment. Add one line to §2.3: if the ticket snapshot is already in context because `pmm-triage` fetched it before delegating here, reuse it and skip straight to §3 — don't fetch twice. |
| §3 Type & status gate (redirect line) | Reword the "Redirected: issuetype=…" line to name the sibling skill by its plugin identity, not a filename — e.g. "Redirected: issuetype=New Feature. Use pmm-triage-feature instead (or just invoke pmm-triage next time and it'll route automatically)." Same tweak in the feature workflow's redirect-to-bug line. |
| §9.6 Jira comment | Unchanged mechanism (`pmm-add-jira-comment` skill, Developers-role visibility, built from the TL;DR verbatim). Formalize the provenance footer — today an undocumented-but-approved deviation (`pmm-triage-jira-comment-footer` memory) — as first-class, documented behavior of the ported skill: `Automated triage (<tool>/<model>), verified against pmm@<sha>[, <repo>@<sha> …]. Proposal only — no code changes made.` This is what recovers the §2.2 header's audit value without a file. |
| §9.7 Labels/components | Unchanged. Keep the `ai-triage` marker (this project's own convention — not renamed to match the unrelated `pmm-dev:pmm-triage`'s `ai-reviewed`, since we are not merging the two — see §2). |
| §10 Uncertainty & handoff | Unchanged in spirit. Drop "exit non-zero if invoked from a script" (not meaningful in an interactive skill invocation); autonomous mode still exists for anyone driving the skill headlessly via `claude -p`, unchanged. |
| §11 Re-running | Shrink to one line — re-running is just invoking the skill again; there's no file to preserve or overwrite. |
| Appendix A (filenames) | **Delete.** No files are written. |
| Appendix B (report skeleton) | **Repurpose, don't delete** — same section order, now describing the shape of the skill's final chat response instead of a Markdown file on disk. One UX improvement worth making here: **lead the response with the TL;DR**, then the supporting sections, rather than TL;DR-last as in the file (the file ordering existed so the TL;DR could be *authored* last, after everything else settled — that's still the right authoring order internally, it's just not the right *reading* order for a chat reply someone will skim first). |
| Appendix C (adapting) | Update to describe the plugin composition model (`${CLAUDE_PLUGIN_ROOT}/references/`, not `../skills/`) and cross-reference `pmm-dev:pmm-triage` as a related-but-distinct tool (§2). |

Everything not listed above — the §5.0 code-reference convention, the §5.1 architecture map
usage, the §6 de-duplication search and its relation tags, the §7 6-way (bug) / 7-way (feature)
classification and its outcome table, the §4.2/§4.3 curated Tech-label and component
vocabularies, the §8 principal-engineer fix proposal — ports **verbatim**. This is the load-bearing
value of the two source docs and none of it is filesystem-dependent.

---

## 6. Marketplace + repo wiring

**In `percona/pmm-ai`:**

1. Add `plugins/pmm-jira-triage/.claude-plugin/plugin.json`:
   ```json
   {
     "name": "pmm-jira-triage",
     "description": "Deep triage for a single PMM Jira Bug or New Feature/Improvement ticket — auto-detects the issue type, then does root-cause analysis, de-duplication, and posts a curated-label/component verdict as an internal Jira comment.",
     "version": "0.1.0",
     "author": { "name": "Percona PMM team" }
   }
   ```
2. Add the four `references/*.md` files (ported verbatim from `ai-workflows/skills/`) and the
   three `skills/*/SKILL.md` files — the new `pmm-triage` dispatcher plus the two ported workflow
   skills (§4/§5).
3. Register it in the root `.claude-plugin/marketplace.json`'s `plugins` array (same shape as the
   existing `pmm-docs` entry) — the WIP `pmm-dev` branch has **not** done this step yet, so this PR
   would actually be the second plugin registered, not the third.
4. Add a plugin `README.md` (install snippet, the three skills — lead with `pmm-triage` as the
   default entry point — one line distinguishing this from
   `pmm-dev:pmm-triage` per §2 for anyone who finds both), following the existing `pmm-docs`/
   `pmm-dev` README shape.
5. Run `claude plugin validate .` and `claude plugin validate plugins/pmm-jira-triage` before
   opening the PR (this repo's own documented contribution check).

**In `percona/pmm`** (separate PR — this is the step that makes it available to the *whole team*
with zero per-person setup, and it doesn't exist yet: `pmm/.claude/settings.json` isn't currently
committed, only a personal, gitignored `.claude/settings.local.json`):

```json
{
  "extraKnownMarketplaces": {
    "pmm-ai": { "source": { "source": "github", "repo": "percona/pmm-ai" } }
  },
  "enabledPlugins": {
    "pmm-jira-triage@pmm-ai": true
  }
}
```

If `pmm-docs` already committed an `extraKnownMarketplaces["pmm-ai"]` entry to `pmm/.claude/
settings.json` by the time this ships, only the `enabledPlugins` line needs adding.

**Per-teammate prerequisites** (not code, but worth stating explicitly since this plugin is
useless without them): an authorized Atlassian MCP connector on their claude.ai account (Jira
read/write goes through it first, per `pmm-fetch-jira-ticket`/`pmm-add-jira-comment`), and GitHub
read access to `percona/pmm-ai` while it's private (their connected GitHub account/App must see
it, or marketplace resolution fails in a Web sandbox the same way it would fail for `pmm-docs`
today).

---

## 7. Relationship to `routines-port-plan.md`

That document is about a **different, larger, not-yet-started** effort: fully unattended,
scheduled, cloud-dispatched triage with no human in the loop, running on Claude Code Routines.
This plan does not depend on it, block it, or get blocked by it. Two points of positive overlap
worth noting:

- §5.4 of that plan lists report persistence as an **open, unresolved** question specifically
  because Routines' cloud VMs are ephemeral and `triage-reports-log/` is gitignored. This port's
  §5 decision (Jira comment as the sole record) **answers that question for free** if the two
  efforts are ever unified — a Routines-based worker built on this same ported skill would already
  have no file-persistence problem to solve.
- The four shared reference skills, once living at `pmm-ai/plugins/pmm-jira-triage/references/`,
  are a strictly better home for them than today's `pmm-sdlc/ai-workflows/skills/` if the Routines
  effort proceeds — `routines-port-plan.md` §5.2 already recommended promoting them to
  `.claude/skills/` for exactly this reason; a plugin's `references/` is the same idea, one repo
  earlier.

No action needed now; this section exists so a future reader doesn't have to re-derive the
connection.

---

## 8. Rollout — two PRs, verified independently

**PR 1 — `percona/pmm-ai`: add the plugin.**

- Port the four shared skills and the two workflow docs, plus the new `pmm-triage` dispatcher, per
  §4/§5.
- `claude plugin validate` both the marketplace root and the new plugin dir.
- *Verify*: `claude --plugin-dir plugins/pmm-jira-triage` in a local `pmm` checkout, then:
  - Re-run 3–5 tickets that already have a historical report in `triage-reports-log/` (78
    available) by saying "triage `<ticket URL>`" for each, and confirm `pmm-triage` auto-detects
    the same issue type the historical filename records, delegates to the matching workflow, and
    reaches the same verdict/classification/code citations as the old file. This is the same
    regression-by-comparison technique `routines-port-plan.md` Phase 1 used, and it's free here —
    the historical reports already exist and already span both issue types.
  - **Negative check (the actual point of this revision):** paste one of those same ticket
    URLs/keys into the chat with no triage/investigate verb — e.g. just the bare link, or
    mentioning it in an unrelated sentence — and confirm `pmm-triage` does **not** fire. This is
    the test that would have caught the original draft's over-eager description; don't skip it.
  - Specifically confirm the dispatcher only fetches the ticket once — watch for a duplicate
    `pmm-fetch-jira-ticket` application in the transcript when a run is delegated from
    `pmm-triage`.
  - Force a mismatch on purpose (invoke `pmm-triage-bug` directly on a ticket you know is a
    Feature) and confirm the §3 gate still redirects cleanly, now naming `pmm-triage` as the
    easier route.
- Merge, then confirm install works cold: `/plugin marketplace add percona/pmm-ai` +
  `/plugin install pmm-jira-triage@pmm-ai` on a machine that has never touched this repo.

**PR 2 — `percona/pmm`: wire up team-wide auto-enable.**

- Add/extend `.claude/settings.json` per §6.
- *Verify on all three surfaces* (this is the actual ask — "other team members... from Claude IDE
  or Claude Code Web" — so all three need a real check, not just the one you're already using):
  - **Local CLI**: fresh `git clone` of `pmm`, open Claude Code, say "triage `<ticket URL>`",
    confirm the skill auto-fires with no `/plugin install` step — then, in the same session,
    paste a *different* ticket URL with no triage intent and confirm it does **not** fire.
  - **IDE extension**: same two checks (positive + negative) from VS Code or JetBrains.
  - **Claude Code Web**: start a session against `percona/pmm` from claude.ai/code on an account
    that has never installed the plugin locally; confirm the marketplace resolves (needs GitHub
    access to `pmm-ai`) and the skill is available via "triage `<ticket URL>`". This is the one
    surface where a committed-settings failure would be easy to miss until a teammate actually
    hits it.

No phased rollout cap or JQL-polling fan-out is needed here (unlike `routines-port-plan.md`'s
Phase 3/4, whose "dispatcher" is a different thing entirely — a scheduled routine that queries
Jira and fires per-ticket cloud runs, not `pmm-triage`'s per-ticket issue-type routing) — this is
interactive, human-invoked, one ticket at a time, same usage shape as today, just installable
instead of hand-attached.

---

## 9. Explicit non-goals for this PR

- **Not** merging or reconciling with the WIP `pmm-dev:pmm-triage` skill. Flagged in §2 and the
  plugin's README; a real conversation with theTibi is a separate, follow-up thread.
- **Not** deprecating `pmm-triage-bug.md` / `pmm-triage-feature-improvement.md` or
  `triage-reports-log/`. They keep working exactly as today for local, file-audited use; this
  plugin is an additional, lighter-weight, team-portable surface, not a replacement. Revisit once
  the plugin has a burn-in period and the operator decides whether the local file-based workflow
  is still worth maintaining in parallel.
- **Not** implementing the Claude Code Routines / unattended-scheduling effort — see §7.
