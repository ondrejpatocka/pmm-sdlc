---
name: pmm-edit-jira-labels-components
description: Additively apply labels and/or components to a single PMM Jira ticket, always preserving what's already there — Atlassian MCP first, authenticated REST fallback, manual last resort. Use when a workflow needs to tag a ticket without touching any existing label or component.
argument-hint: "[Jira issue key or URL] [labels to add, comma-separated] [components to add, comma-separated]"
---

# Skill: pmm-edit-jira-labels-components

Additively apply labels and/or components to one PMM Jira ticket. Referenced (not
copied) by PMM workflows such as `pmm-triage/pmm-triage-bug.md` and
`pmm-triage/pmm-triage-feature-improvement.md`. Tool-agnostic: describes **what** to
apply and **what** to record, not which tool.

- **Input (`$ARGUMENTS`):** a Jira issue key (e.g. `PMM-15076`) or a ticket URL, a
  comma-separated list of labels to add (may be empty), and a comma-separated list of
  components to add (may be empty). If a URL is given, extract the issue key first.
- **Output:** a `## Jira labels & components update` markdown section (layout at the
  end) for the caller to append to its report.

## Mandatory rule — additive only, never destructive

**This skill only adds. It must never remove or replace an existing label or
component.** Every write is preceded by a fresh read of the ticket's *current*
`labels`/`components` (read as close to the write as possible, to minimize the race
window — see the tiered mechanics below), merged with the proposed additions
(case-sensitive dedupe), and the full merged set is what gets written. If a write
mechanism can't guarantee this, don't use it.

## Component validation — mandatory before any write attempt

A component must match an existing project component name **exactly, case-sensitively**
— Jira rejects an edit that names an unknown component. Fetch the ticket's project's
valid component list via the lightweight, unauthenticated
`GET https://perconadev.atlassian.net/rest/api/3/project/<projectKey>/components`
(same "no token needed for a public project read" convention `pmm-fetch-jira-ticket`
uses for ticket data) and check each proposed component against the returned names.

The alternative `getJiraIssueTypeMetaWithFields` MCP call also carries this data (as
each field's `allowedValues`), but returns a very large payload — confirmed live, over
200K characters for a single issue type — so prefer the lightweight REST list above;
only fall back to the field-metadata call if that lookup itself fails, and if you do,
extract just the `components` field's `allowedValues` names rather than passing the raw
response through.

If the whole component-list lookup fails, treat every proposed component as
unverifiable: skip all of them (log why) and still proceed with the labels.

## Label validation

No live corpus check — labels have no `allowedValues` restriction in Jira, and this
skill's callers only ever propose from a small, curated vocabulary. Apply one local,
static rule: reject any proposed label containing whitespace (Jira's native rule for
labels) — skip and log it the same way as an unmatched component. Otherwise accept the
label as given, then dedupe case-sensitively against the ticket's own current labels
during the merge step above.

## Applying the update (try in order; use the first that succeeds)

A write always requires authentication — there is no anonymous tier here.

1. **Authenticated Atlassian MCP — primary.** Use the same Atlassian connector
   `pmm-fetch-jira-ticket` uses.
   - Read the ticket's current `labels` and `components` via `getJiraIssue`
     (`fields: ["labels", "components"]`).
   - Validate proposed components against the live project component list (above);
     validate proposed labels for whitespace.
   - Merge: union of current + valid proposed, deduped.
   - Call the tool whose name ends in `editJiraIssue` (or `ToolSearch`
     `select:editJiraIssue`) with `cloudId: perconadev.atlassian.net`, `issueIdOrKey`
     (extracted key), and `fields: {"labels": [...merged labels], "components":
     [...merged components, each as {"name": "<exact Jira name>"}]}`. This tool only
     exposes a full-value replace, not an additive verb — that's why the read-merge step
     above is mandatory, not optional.

   If unauthorized/unavailable or the call errors, escalate.
2. **Authenticated REST API — escalation.** Only if the operator has already configured
   Jira write credentials accessible to the assistant (e.g. an API token) — this skill
   does not prompt for or set up credentials. Here, prefer Jira's real additive `update`
   verb, which avoids the read-merge-write race entirely: `PUT
   https://perconadev.atlassian.net/rest/api/3/issue/<key>` with a body of the form
   `"update": {"labels": [{"add": "<label>"}, ...], "components": [{"add": {"name":
   "<component>"}}, ...]}` — one `add` entry per validated label/component. No
   credentials available, or the request errors → escalate to manual.
3. **Manual — last resort.** Cannot auto-apply. In **assisted mode**, show the operator
   the proposed labels/components (and which components failed validation and why) and
   ask them to apply the valid ones by hand. In **autonomous mode**, do not block the
   caller's report — record the `## Jira labels & components update` section as "not
   applied," including the full proposed lists, so a human can apply them later.

## Output layout

```markdown
## Jira labels & components update

- Status: Applied | Partially applied | Not applied — manual action needed
- Method: Atlassian MCP | Authenticated REST | Manual (pending)
- Labels added: <list, or "none">
- Components added: <list, or "none">
- Skipped — no matching project component: <list, or "none">
- Skipped — invalid label format: <list, or "none">
```
