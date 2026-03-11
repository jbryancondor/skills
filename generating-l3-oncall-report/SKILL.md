---
name: generating-l3-oncall-report
description: Creates the weekly L3 on-call handoff report in the Notion MAS L3 Record database. Use when the user asks to create, generate, write, or fill the L3 on-call report, oncall handover.
model: haiku
allowed-tools: Read, Grep, Glob
---

# L3 On-Call Report Generator

Creates a new entry in the [🚒 MAS L3 Record](https://www.notion.so/34bd8cb0135347ebaaaaca863b755f62) Notion database for the **Merchant Platform** team.

See [DATA-SOURCES.md](references/DATA-SOURCES.md) for all fixed IDs and references.
See [EXAMPLE-OUTPUT.md](references/EXAMPLE-OUTPUT.md) for a real filled report — use it as the quality and formatting bar.

## Step 1: Collect Inputs — Start Here

**ALWAYS ask this first, before anything else:**

> "Which Monday did your on-call start? (YYYY-MM-DD)"

The entire report is scoped to the rotation window. Derive all dates from this answer:

| Variable | Value |
|----------|-------|
| `START_DATE` | The Monday provided by the user |
| `START_DATETIME` | `[START_DATE]T12:00:00` (noon on start Monday) |
| `END_DATE` | `START_DATE + 7 days` (next Monday) |
| `END_DATETIME` | `[END_DATE]T12:00:00` (noon on end Monday) |

Then ask for ALL of the following before doing anything else:

| # | Field | Description |
|---|-------|-------------|
| 1 | Rotation L3 | Current on-call engineer's name |
| 2 | Next Rotation L3 | Name of next on-call engineer |
| 3 | Team | `Merchant Platform` , `Merchant SMB`, `Merchant Product` | All of them by default
| 4 | L3 On Call parent task | URL or GID of the "L3 On Call" Asana task for this rotation |
| 5 | 📸 High severity alerts screenshot | Screenshot from Grafana showing the high severity alert list for this week |
| 6 | 📸 Low severity alerts screenshot | Screenshot from Grafana showing the low severity alert list for this week |
| 7 | Outstanding postmortems | Notion link(s) or "None". If a link is provided, auto-extract a brief summary from the page |
| 8 | Significant events | Key insights and patterns (numbered list) |

**Do not proceed to any workflow step until all inputs above are collected**, including both screenshots. The screenshots are required for Step 6 to cross-reference alert names with Slack thread solutions.

## Workflow

```
Task Progress:
- [ ] Step 1: Ask for start Monday and collect all inputs
- [ ] Step 2: Resolve Notion user IDs for both engineers
- [ ] Step 3: Fetch Merchant Platform incidents from Notion (rotation window)
- [ ] Step 4: Fetch Merchant Platform P1 remediation subtasks from Asana
- [ ] Step 5: Fetch bugs created or updated during the rotation window
- [ ] Step 6: Extract alerts from screenshots, user selects which to deep-dive, fetch Slack context
- [ ] Step 7: Fetch pending cases, actionable items, and resolved cases from Asana
- [ ] Step 8: Generate Markdown preview and wait for user confirmation
- [ ] Step 9: Create the Notion page (only after user confirms)
- [ ] Step 10: Report the new page URL to the user
```

### Step 2: Resolve Notion User IDs

Use `mcp__notion__notion-get-users` to search both engineer names and get their Notion user IDs.

### Step 3: Fetch Merchant Platform Incidents

Query the Notion incidents database (see [DATA-SOURCES.md](references/DATA-SOURCES.md)) for records created between `START_DATETIME` and `END_DATETIME`.

**Filter only incidents where the squad is Merchant Platform.** The squad field may be named "Squad", "Team", or "Affected Team" — inspect the schema first with `mcp__notion__notion-fetch` on the incidents DB.

Match squad values: `Merchant Platform` (current name). Legacy names `Merchant Acquisition` and `Merchant Success` refer to the same team — include them too.

Return each matching incident as its Notion page URL and title.

### Step 4: Fetch Merchant Platform P1 Remediation Subtasks

Fetch the P1 Remediation parent task (see [DATA-SOURCES.md](references/DATA-SOURCES.md)) and get all its subtasks.

Filter to subtasks that:
1. Are **not completed**
2. Belong to **Merchant Platform** — match by checking if the subtask name contains any of: `Merchant Platform`, `Merchant Acquisition`, `Merchant Success`

If the subtask name alone is not enough to determine squad, fetch the subtask's full details to check its project name or tags.

Format each match as:
```
- [ASANA_TASK_URL] (in progress / not started)
```

### Step 5: Fetch Bugs from Both Bug Boards

Search both Merchant Platform bug boards (see [DATA-SOURCES.md](references/DATA-SOURCES.md)) for tasks created **or** modified during the rotation window (`START_DATETIME` → `END_DATETIME`).

Use `mcp__plugin_asana_asana__asana_search_tasks` twice (once per project), run in parallel. See [DATA-SOURCES.md](references/DATA-SOURCES.md) — **Asana Bug Boards** for the exact query and project GIDs.

Merge and deduplicate by task GID. Separate into:
- **All bugs** → for the Bugs Backlog section
- **Completed/accepted bugs** → for the Accepted bugs update section

### Step 6: Alerts — Extract, Select, and Deep Dive

**6a) Extract alerts from screenshots.**

Visually inspect both screenshots provided in Step 1. Extract every alert name and trigger count visible in:
- The **high severity screenshot**
- The **low severity screenshot**

**6b) Present a ordered list and WAIT for user selection.**

Show the user all extracted alerts grouped by severity, formatted as a ordered list:

```
High Severity:
- 1. Alert Name A — fired N times
- 2. Alert Name B — fired N times

Low Severity:
- 3. Alert Name C — fired N times
- 4. Alert Name D — fired N times
```

Ask: **"Which alerts should I deep-dive? Select the ones to include in the Deep Dive Relevant Alerts section."**

**Do NOT proceed until the user selects.** The user will check the alerts they want investigated.

**6c) Fetch Slack context ONLY for selected alerts.**

Read the Slack alerts channel for messages posted between `START_DATETIME` and `END_DATETIME`. See [DATA-SOURCES.md](references/DATA-SOURCES.md) — **Slack** for the channel ID and query patterns.

For each **user-selected** alert, read its Slack thread(s) inner messages and extract:
- Context (what caused it)
- Solution(s) applied
- Enhancements (run-book updates, if mentioned)

The same alert may fire multiple times — consolidate across all occurrences.

Produce three outputs for the template:

**1. High severity list** — all alerts from the high severity screenshot, with trigger count.

**2. Low severity list** — all alerts from the low severity screenshot, with trigger count.

**3. Deep Dive Relevant Alerts** — **group related alerts by theme** (e.g., "Corbeta Intermittence errors", "Ledger Errors"), not individually. Each group has a bold name, the child alert names, and shared Solution/Enhancement:
```
- **[Theme/group name]:**
  - [Alert name 1]
  - [Alert name 2]
  ⏩ Solution: [description, link to run-book if applicable]
  🟢 Enhancement: [optional — run-book updates or improvements made]
```

See [EXAMPLE-OUTPUT.md](references/EXAMPLE-OUTPUT.md) — **Deep Dive Relevant Alerts** section for the exact format.

If a selected alert has no matching Slack thread, mark it as `No solution documented in Slack`.

### Step 7: Fetch Tasks from Support Cases Board and L3 On Call Parent Task

Fetch tasks from both sources in parallel. See [DATA-SOURCES.md](references/DATA-SOURCES.md) — **Asana Support Cases Board** and **Asana L3 On Call Parent Task** for project GIDs and queries.

From both sources, collect **all tasks assigned to the logged-in user** during the rotation window. Merge and deduplicate by GID. Then partition into:

**A) Pending cases** (incomplete tasks assigned to me) → populates the "Pending cases to resolve" section under Alert Groups. These are cases to hand over to the next on-call engineer.

**B) Completed tasks** (completed by me during the rotation) → used for two sections:
- **Actionable items** — numbered list, each entry: brief description of work done + Asana task URL
- **Most frequently escalated cases** — in the Meetings Notes section

For pending cases and actionable items, fetch the task name/description from Asana and format as:
```
1. [Description of work done] ASANA_TASK_URL
2. [Description of work done] ASANA_TASK_URL
```

For escalated cases:
```
- [Case name](ASANA_TASK_URL)
```

### Step 8: Generate Markdown Preview — WAIT FOR CONFIRMATION

Before creating anything in Notion, render the full report as a **Markdown document** in the chat and explicitly ask:

> "Does this look correct? Reply **yes** to create the Notion page, or tell me what to fix."

**Do NOT proceed to Step 9 until the user explicitly confirms.**

The Markdown preview must include every section fully populated:
- All auto-fetched data substituted in place (incidents, P1 remediations, bugs, alert summaries, resolved cases)
- All user-provided inputs in their correct sections
- Screenshot placeholders shown as labeled comments (e.g. `<!-- SCREENSHOT 2: paste Bugs Backlog Merchant Acquisition here -->`)

If the user requests changes, apply them to the preview and show the updated version again. Repeat until confirmed.

### Step 9: Create the Notion Page (only after user confirms Step 8)

Use `mcp__notion__notion-create-pages` with:
- **Parent:** `data_source_id: be808cce-e00c-446a-9503-1257c02bab90`
- **Content:** See [TEMPLATE.md](assets/TEMPLATE.md) — substitute all `[PLACEHOLDER]` values

**Property mapping:**

| Property | Value |
|----------|-------|
| Title | `Engineering L3 On-Call @[START_DATE]` |
| date:Start rotation:start | `[START_DATE]` |
| date:Start rotation:is_datetime | `0` |
| Rotation L3 | `["[USER_ID_CURRENT]"]` |
| Next Rotation L3 | `["[USER_ID_NEXT]"]` |
| Team | JSON array — one or both of `"Merchant SMB"`, `"Merchant Product"` |

Sections that cannot be auto-populated MUST keep the original placeholder text so the engineer knows what to fill manually.

### Step 10: Report Results (after Notion page is created)

Provide:
- Direct Notion page URL
- Which sections were auto-populated vs. need manual completion
- Reminder: paste screenshots (Grafana alerts, Asana bugs backlog) directly into the page after creation
