# Data Sources Reference

## Contents
- [Team Identity](#team-identity)
- [Rotation Window](#rotation-window)
- [Notion — Incidents DB and L3 Record](#notion)
- [Asana — P1 Remediation Actions](#asana)
- [Slack — Alerts channel](#slack)
- [Asana — Support Cases Board](#asana-support-cases-board)
- [Asana — L3 On Call Parent Task](#asana-l3-on-call-parent-task)
- [Asana — Bug Boards](#asana-bug-boards)

## Team Identity

Our team is **Merchant Platform**. When filtering data from any source, match against:
- `Merchant Platform` (current name)
- `Merchant Acquisition` (legacy)
- `Merchant Success` (legacy)

## Rotation Window

On-call runs **Monday 12:00 → next Monday 12:00** (7 days).

| Variable | Derivation |
|----------|------------|
| `START_DATE` | Monday provided by the user |
| `START_DATETIME` | `[START_DATE]T12:00:00` |
| `END_DATE` | `START_DATE + 7 days` |
| `END_DATETIME` | `[END_DATE]T12:00:00` |

All data fetches must be scoped to this window.

## Notion

| Resource | Value |
|----------|-------|
| MAS L3 Record database | https://www.notion.so/34bd8cb0135347ebaaaaca863b755f62 |
| MAS L3 Record data source ID | `be808cce-e00c-446a-9503-1257c02bab90` |
| Incidents database | https://www.notion.so/3601eaf29e32459e85db55aaae5568a2 |
| L3 Template page | https://www.notion.so/31eeace5de2b8019b0cdce4c4fef9df6 |

### Fetching Merchant Platform incidents

1. Fetch the incidents DB schema first to find the squad/team property name:
   ```
   mcp__notion__notion-fetch(id="https://www.notion.so/3601eaf29e32459e85db55aaae5568a2")
   ```
2. Query with `mcp__notion__notion-query-data-sources` filtering by:
   - `created_time` between `START_DATETIME` and `END_DATETIME`
   - Squad/team field matching `Merchant Platform`, `Merchant Acquisition`, or `Merchant Success`

## Asana

| Resource | Value |
|----------|-------|
| P1 Remediation Actions task GID | `1204184859300684` |
| P1 Remediation Actions URL | https://app.asana.com/0/1201765457329654/1204184859300684 |

### Fetching Merchant Platform P1 remediation subtasks

```
mcp__plugin_asana_asana__asana_get_task(
  task_id="1204184859300684",
  opt_fields="subtasks,subtasks.name,subtasks.completed,subtasks.permalink_url"
)
```

From the returned subtasks:
- Keep only those where `completed = false`
- Keep only those whose name contains `Merchant Platform`, `Merchant Acquisition`, or `Merchant Success`
- If squad cannot be determined from name alone, fetch the subtask's project to verify

Real report format:
```
- https://app.asana.com/1/.../task/TASK_ID?focus=true (in progress)
- https://app.asana.com/1/.../task/TASK_ID?focus=true (not started)
```

## Slack

| Resource | Value |
|----------|-------|
| Alerts channel ID | `C03PYLV2XUK` |
| Alerts channel URL | https://addi.enterprise.slack.com/archives/C03PYLV2XUK |

### Reading alerts during the rotation window

1. Convert `START_DATETIME` and `END_DATETIME` to Unix timestamps (seconds).
2. Read the channel:
   ```
   mcp__plugin_slack_slack__slack_read_channel(
     channel_id="C03PYLV2XUK",
     oldest="[START_UNIX_TS]",
     latest="[END_UNIX_TS]",
     limit=100
   )
   ```
3. For each alert message, read its thread:
   ```
   mcp__plugin_slack_slack__slack_read_thread(
     channel_id="C03PYLV2XUK",
     message_ts="[MESSAGE_TS]"
   )
   ```
4. Group messages by alert name (the alert name is typically in the message title or first line).
5. For each group: count occurrences, extract context from message body, extract solution from thread replies.

## Asana Support Cases Board

| Resource | Value |
|----------|-------|
| Project GID | `1209570512740114` |
| Board URL | https://app.asana.com/1/705028621614854/project/1209570512740114/board/1209576846197481 |

### Completed cases (resolved by the on-call engineer)

```
mcp__plugin_asana_asana__asana_search_tasks(
  projects_any="1209570512740114",
  assignee_any="me",
  completed=true,
  completed_on_after="[START_DATE]",
  completed_on_before="[END_DATE]",
  opt_fields="name,permalink_url,completed_at,assignee.name,custom_fields"
)
```

### Pending cases (incomplete, to hand over)

```
mcp__plugin_asana_asana__asana_search_tasks(
  projects_any="1209570512740114",
  assignee_any="me",
  completed=false,
  modified_on_after="[START_DATE]",
  modified_on_before="[END_DATE]",
  opt_fields="name,permalink_url,assignee.name,custom_fields"
)
```

## Asana L3 On Call Parent Task

The user provides the GID or URL of the current rotation's "L3 On Call" task (changes every week). Extract the task GID from the URL if a full URL is given.

Fetch completed subtasks:
```
mcp__plugin_asana_asana__asana_get_task(
  task_id="[L3_ONCALL_TASK_GID]",
  opt_fields="subtasks,subtasks.name,subtasks.completed,subtasks.permalink_url,subtasks.completed_at"
)
```

Partition subtasks:
- `completed = true` → completed tasks (for Actionable items + Resolved cases in Meetings Notes)
- `completed = false` → pending tasks (for Pending cases to hand over)

Merge each partition with the corresponding Support Cases results (deduplicate by GID).

## Asana Bug Boards

Two boards to search in parallel for bugs created or updated during the rotation window.

| Board | Project GID | URL |
|-------|-------------|-----|
| Bug Board 1 | `1200680916517745` | https://app.asana.com/1/705028621614854/project/1200680916517745/list/1205115104789272 |
| Bug Board 2 | `1202632056546796` | https://app.asana.com/1/705028621614854/project/1202632056546796/list/1205116312449281 |

### Search query (run for each project GID)

```
mcp__plugin_asana_asana__asana_search_tasks(
  projects_any="[PROJECT_GID]",
  modified_on_after="[START_DATE]",
  modified_on_before="[END_DATE]",
  opt_fields="name,permalink_url,completed,created_at,modified_at,assignee.name,custom_fields"
)
```

- Run both searches in parallel
- Merge and deduplicate results by task GID
- Separate into: **all bugs** (for Bugs Backlog) and **completed/accepted bugs** (for Accepted bugs update)
