---
name: pmm-fetch-jira
description: Fetch a single PMM Jira ticket (by key or URL) into a `## Ticket snapshot` — unauthenticated REST API first, Atlassian MCP fallback, manual paste last. Use when a workflow needs one ticket's data.
argument-hint: "[Jira issue key or URL, e.g. PMM-15076]"
---

# Skill: pmm-fetch-jira

Fetch one PMM Jira ticket into a `## Ticket snapshot`. Referenced (not copied) by PMM
workflows such as `pmm-triage-bug-ticket/pmm-triage-bug-workflow.md`. Tool-agnostic:
describe **what** to fetch and **what** to write, not which tool.

- **Input (`$ARGUMENTS`):** a Jira issue key (e.g. `PMM-15076`) or a ticket URL. If a URL
  is given, extract the issue key (the `PMM-…` path segment) before building the calls
  below.
- **Output:** a `## Ticket snapshot` markdown section (layout at the end) for the caller
  to append to its report.

## Getting the ticket data (try in order; use the first that yields data)

1. **Unauthenticated REST API — primary, no setup.** Fetch `https://perconadev.atlassian.net/rest/api/3/issue/$ARGUMENTS?fields=summary,status,issuetype,priority,description,components,labels,fixVersions,versions,resolution,reporter,assignee,created,updated,parent,issuelinks,comment,attachment`. For public PMM tickets this returns JSON with (almost) every field below — no token, no connector. Field-name notes: `key` is in the response root; `versions` = affectsVersions, `comment` = comments, `attachment` = attachments. The v3 `description` is ADF (JSON) — add `&expand=renderedFields` for HTML, or use `/rest/api/2/issue/$ARGUMENTS` for wiki/plain-text fields if ADF is awkward.
2. **Authenticated Atlassian MCP — escalation.** If REST returns 401/403 (restricted ticket) or omits a field you need (e.g. comments not visible anonymously), use the Atlassian connector: find a tool whose name ends in `searchJiraIssuesUsingJql` (or run `ToolSearch` with `select:searchJiraIssuesUsingJql`) and call it with `cloudId: perconadev.atlassian.net`, `jql: key = $ARGUMENTS`, `responseContentFormat: markdown`. Requires the connector authorized (claude.ai → Settings → Connectors, or `/mcp` in Claude Code).
3. **Manual paste — last resort.** Ask the operator to paste the ticket; build the snapshot from that.

**Never fetch the `/browse/<key>` web page for data** — it returns an empty JavaScript SPA shell. Record the method + endpoint used in the snapshot. If only some fields came back, stamp the snapshot `⚠ Partial — <fields missing>` and continue; do not stop.

**On total failure** (no method yields usable data): do not treat a login / 401 / 403 / empty page as proof the ticket is missing. Hand control back to the **calling workflow's** failure handling (e.g. ask the operator / paste in assisted runs; write an "unreachable" artifact and stop in autonomous runs). Never delete a partially-written report.

## Fields to capture

- `key`, `issuetype`, `summary`, `priority`, `status`
- `components`, `labels`, `fixVersions`, `affectsVersions`, `resolution`
- `reporter`, `assignee`, `created`, `updated`
- `parent` (key only)
- `description` (full text)
- All comments (author, timestamp, body)
- All issue links (relation + target key + target summary)
- Attachments list (filename + URL)

## Snapshot layout

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
