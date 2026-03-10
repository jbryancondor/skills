# Notion Page Content Template

## Contents
- [Page content template](#page-content-template) — substitute all `[PLACEHOLDER]` values before use
- [Placeholder substitution guide](#placeholder-substitution-guide) — which fields are auto vs manual
- [Notion mention syntax](#notes-on-notion-mention-syntax)

For a real filled example showing the expected quality and formatting, see [EXAMPLE-OUTPUT.md](EXAMPLE-OUTPUT.md).

## Page content template

Use this as the `content` field in `mcp__notion__notion-create-pages`.
Replace all `[PLACEHOLDER]` values. Images cannot be embedded via API — leave image placeholders as instructional text for the engineer.

---

```
# On-Call Hand-off
---

### Squad Report {toggle="true"}

**Owner:** [MENTION_ROTATION_L3_USER]

### Relevant Incidents from the last week

[AUTO: list each incident as a Notion page link, e.g.:]
- [Incident page title](https://www.notion.so/INCIDENT_PAGE_ID)

If no incidents found, write: "No incidents recorded this rotation."

### Relevant remediation actions update

[AUTO: list each incomplete P1 remediation subtask as an Asana URL with status, e.g.:]
- https://app.asana.com/1/.../task/TASK_ID?focus=true (in progress)

### Alert Groups

📸 _[**SCREENSHOT 1 — Alerts overview**: paste the Grafana alerts screenshot here after page is created]_

- Pending cases to resolve (to hand over to the next on-call engineer):
  - [AUTO from Step 7A — group by source category:]
  - Support Cases
    - [ASANA_TASK_URL]
    - [additional context if needed, e.g. "5 additional not started cases re-assigned through Asana."]
  - [Other categories as applicable, e.g. Marketplace Cancellation]
    - [SLACK_OR_ASANA_LINK]
  - If none found, write: "No pending cases to hand over."
- Most frequent alerts triggered:
  - **High Severity**
    📸 _[paste high severity screenshot here after page is created]_
  - **Low Severity**
    📸 _[paste low severity screenshot here after page is created]_
  - **Deep Dive Relevant Alerts**
    [AUTO from Step 6c — GROUP related alerts by theme, not individually:]
    - **[Theme/group name]:**
      - [Alert name 1]
      - [Alert name 2]
      ⏩ Solution: [description, link to run-book if applicable]
      🟢 Enhancement: [optional — run-book updates or improvements made]
    - **[Another theme]:**
      - [Alert name]
      ⏩ Solution: [description]

### Bugs Backlog

📸 _[**SCREENSHOT 2 — Bugs Backlog Merchant Acquisition**: paste the Asana bugs dashboard screenshot for Merchant Acquisition here after page is created]_

📸 _[**SCREENSHOT 3 — Bugs Backlog Merchant Success**: paste the Asana bugs dashboard screenshot for Merchant Success here after page is created]_

[AUTO: list all bugs created or updated this week from both bug boards:]
- [Bug name](ASANA_TASK_URL) — open — [assignee]
- [Bug name](ASANA_TASK_URL) — completed — [assignee]

### Accepted bugs update

[AUTO: list only completed/accepted bugs from the rotation window:]
- [Bug name](ASANA_TASK_URL)

### Outstanding Postmortems

- [OUTSTANDING_POSTMORTEMS — if user provided a Notion link, include the link + auto-extracted brief summary. Otherwise "None"]

### Significant Events

> Please provide a summary of the key insights and events from your on-call experience. Identify relevant patterns and improvement opportunities.

[SIGNIFICANT_EVENTS_NUMBERED_LIST]

### Actionable items

> What are the actionable items you found during the on-call?

[AUTO from Step 7B — numbered list with brief description + Asana link:]
1. [Description of work done] [ASANA_TASK_URL]
2. [Description of work done] [ASANA_TASK_URL]

### Meetings notes {toggle="true"}

> Schedule a meeting for the handover and use this space for your notes.

- Most frequently escalated cases (resolved by on-call engineer):
  - [AUTO from Step 7 — merged list of resolved support cases + completed L3 On Call subtasks:]
  - [Case name](ASANA_TASK_URL)
  - [Case name](ASANA_TASK_URL)
- Alerts created or updated:
  - Manual check.
- Runbooks created or updated:
  - Manual check.
- Bugs, enhancements, and roadmap tasks created and implemented:
  - Related to escalated cases and alerts.
  - Manual check.
```

---

## Placeholder substitution guide

| Placeholder | Source | Auto? |
|-------------|--------|-------|
| `[MENTION_ROTATION_L3_USER]` | Notion user ID resolved in Step 2 — format as mention | Yes |
| Incident page links | Queried from Notion incidents DB in Step 3 | Yes |
| P1 remediation subtask URLs | Fetched from Asana in Step 4 | Yes |
| Pending cases | Auto-fetched — incomplete tasks from Step 7A | Yes |
| High/Low severity alerts | Extracted from screenshots in Step 6a | Yes |
| Deep Dive Relevant Alerts | User-selected alerts, context from Slack in Step 6c | Yes |
| Bugs Backlog list | Auto-fetched from both bug boards in Step 5 | Yes |
| Accepted bugs list | Auto-fetched — completed bugs from Step 5 | Yes |
| Outstanding Postmortems | Provided by user; if Notion link given, summary auto-extracted | Semi |
| `[SIGNIFICANT_EVENTS_NUMBERED_LIST]` | Provided by user | No |
| Actionable items | Auto-fetched — completed tasks from Step 7B | Yes |
| Alerts created or updated | Manual — engineer fills in Notion | No |
| Resolved support cases | Auto-fetched from Support Cases board + L3 On Call subtasks in Step 7 | Yes |

## Notes on Notion mention syntax

To mention a user in Notion content, use:
```
<mention-user id="USER_ID_HERE"/>
```

To link a Notion page inline, use its URL directly as a markdown link:
```
[Page Title](https://www.notion.so/PAGE_ID)
```
