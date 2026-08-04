---
name: pmm-add-jira-comment
description: Post a Jira comment restricted to the Developers role (never public) on a single PMM ticket — Atlassian MCP first, authenticated REST fallback, manual last resort. Use when a workflow needs to leave an internal-only note on a ticket.
argument-hint: "[Jira issue key or URL] [comment text]"
---

# Skill: pmm-add-jira-comment

Post one internal-only comment on a PMM Jira ticket. Referenced (not copied) by PMM
workflows such as `pmm-triage/pmm-triage-bug.md` and
`pmm-triage/pmm-triage-feature-improvement.md`. Tool-agnostic: describes **what** to
post and **what** to record, not which tool.

- **Input (`$ARGUMENTS`):** a Jira issue key (e.g. `PMM-15076`) or a ticket URL, plus the
  comment text to post. If a URL is given, extract the issue key (the `PMM-…` path
  segment) before building the calls below.
- **Output:** a `## Jira comment` markdown section (layout at the end) for the caller to
  append to its report.

## Visibility — mandatory, not optional

**Every comment this skill posts MUST be restricted, never public.** Percona Jira
(`perconadev.atlassian.net`) is Jira Cloud; a comment posted here without a `visibility`
restriction is visible to the reporter and anyone with browse access to the ticket.

Confirmed by reading a live PMM ticket's existing comments (`PMM-15076`, project `PMM`):
restricted comments there carry `"visibility": {"type": "role", "value": "Developers",
"identifier": "Developers"}`. Use this as the default for the PMM project. If this skill
is ever pointed at a different Jira project where `Developers` is not a valid role, that
is a failure of the primary posting attempt (see below) — **never retry without
`visibility` set** just to get the comment to post. Fall back per the escalation chain
instead, and record what happened.

## Posting the comment (try in order; use the first that succeeds)

Unlike a read, a write always requires authentication — there is no anonymous tier here.

1. **Authenticated Atlassian MCP — primary.** Use the same Atlassian connector
   `pmm-fetch-jira-ticket` uses. Find a tool whose name ends in `addCommentToJiraIssue`
   (or run `ToolSearch` with `select:addCommentToJiraIssue`) and call it with:
   - `cloudId: perconadev.atlassian.net`
   - `issueIdOrKey`: the extracted key
   - `commentBody`: the comment text
   - `commentVisibility: {"type": "role", "value": "Developers"}`

   Requires the connector authorized (claude.ai → Settings → Connectors, or `/mcp` in
   Claude Code). If the connector is **not authorized / unavailable**, the tool isn't
   found, or the call errors (including an invalid-role/group error), escalate to REST —
   do not retry this call without `commentVisibility`.
2. **Authenticated REST API — escalation.** Only if the operator has already configured
   Jira write credentials accessible to the assistant (e.g. an API token) — this skill
   does not prompt for or set up credentials. `POST
   https://perconadev.atlassian.net/rest/api/3/issue/<key>/comment` with a JSON body
   carrying both the comment content and `"visibility": {"type": "role", "value":
   "Developers"}`. No credentials available, or the request errors → escalate to manual.
3. **Manual — last resort.** Cannot auto-post. In **assisted mode**, show the drafted
   comment text and target visibility to the operator and ask them to paste it into Jira
   by hand; record that manual posting was needed. In **autonomous mode**, do not block
   the caller's report — record the `## Jira comment` section as "not posted," including
   the drafted text and intended visibility, so a human can post it later.

**On any failure of step 1 or 2, do not silently drop the visibility restriction to make
the post succeed.** Move to the next step instead.

## Output layout

```markdown
## Jira comment

- Status: Posted | Not posted — manual action needed
- Method: Atlassian MCP | Authenticated REST | Manual (pending)
- Visibility: role:Developers
- Comment text:
  <the text that was posted, or the drafted text if not posted>
```
