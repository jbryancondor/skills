---
name: creating-asana-tasks-from-marketplace-cancellations
description: Converts unresolved Slack cancellation threads into deduplicated Asana subtasks with priority classification. Use when the user asks to create Asana tasks from #cancellations-marketplace threads, sync cancellation requests to Asana, or track marketplace cancellations.
---
# Marketplace Cancellation Task Creator

## Prerequisites

- MCP Asana and Slack servers connected (verify with `/mcp`)
- Read access to #cancellations-marketplace (`C06PV441GLA`)
- Edit permissions on the target Asana parent task
- User must have taken action (reactions/replies) in threads within the last 7 days

## Inputs

1. **Slack User ID** - Auto-populated from logged-in user
2. **Asana Parent Task ID** - Provided by user (e.g., L3 On Call task)

## Fixed Parameters

| Parameter | Value |
|-----------|-------|
| Channel | `C06PV441GLA` (#cancellations-marketplace) |
| Date Range | Last 7 days |
| Filter | Threads with logged-in user's actions only |
| Deduplication | By Loan ID against existing subtasks |

## Workflow

Copy this checklist and track progress:
```
Task Progress:
- [ ] Step 1: Verify MCP connections (run /mcp)
- [ ] Step 2: Fetch logged-in user ID
- [ ] Step 3: Get parent task ID from user
- [ ] Step 4: Load existing subtask names and extract Loan IDs
- [ ] Step 5: Search channel for threads (last 7 days, user-actioned)
- [ ] Step 6: For each thread: deduplicate, validate, classify priority, create task
- [ ] Step 7: Generate summary report
```

**Step 4 is critical** - Always load existing subtasks first to prevent duplicates.

**Step 6 detail per thread:**
1. Extract Loan ID from thread → skip if already exists in parent task
2. Validate required fields → skip if incomplete
3. Classify priority based on age (see below)
4. Create subtask via `asana_create_task`

## Priority Classification

| Priority | Age | Criteria |
|----------|-----|----------|
| **P1** | 7+ days | Customer harmed, escalated |
| **P3** | 3-6 days | Automation failures, double cancellations |
| **P4** | 2-3 days | Complex issues |
| **P5** | 0-2 days | New requests |

## Task Creation

**Naming:** `Marketplace Cancellations - [LOAN-ID] - [PRIORITY] - [CUSTOMER-NAME]`

**API call** - Always use `notes` (plain text), never `html_notes`:
```
asana_create_task(
  name="Marketplace Cancellations - [LOAN-ID] - [P#] - [CUSTOMER]",
  parent="[PARENT-TASK-ID]",
  notes="Posted: [DATE] ([DAYS] days) | Cedula: [CEDULA] | Load: [LOAN-ID]\nBusiness: [SLUG] | Type: [TYPE] | Amount: [AMOUNT]\n\nReason: [TEXT]\n\nSlack Thread: [URL]"
)
```

**Required fields** (skip thread if any missing): customer name, loan ID, cedula, business slug, amount, posted date.

**Slack message fields:** `Nombre:`, `Cedula:`, `Load:` (UUID), `Comercio:`, `Monto:`.

## Output Report Template

```
MARKETPLACE CANCELLATION TASKS

User: @[USERNAME] | Parent: [TASK-NAME] | Range: Last 7 days

Results:
├─ Threads Processed: [N]
├─ New Tasks Created: [N]
├─ Skipped (Already Exist): [N]
├─ Skipped (Invalid Data): [N]
└─ Priority Breakdown: P1:[N] P3:[N] P4:[N] P5:[N]

CREATED TASKS
[For each: Task Name | Asana URL | Slack Thread URL]

SKIPPED THREADS
[For each: Loan ID | Reason (already exists / missing data)]
```

If all threads already have tasks, report status as "ALL THREADS ALREADY TRACKED" with zero new tasks.

## Error Handling

| Error | Action |
|-------|--------|
| MCP disconnected | Run `/mcp` to reconnect, re-authenticate if prompted |
| Invalid parent task ID | Verify ID exists and user has edit permissions |
| No threads found | Confirm user took action in threads within past 7 days |
| Loan ID already exists | Skip thread (intentional deduplication) |
| Missing required fields | Skip thread, note in report |
| XML parsing error | Ensure `notes` param is used, not `html_notes` |

## Limitations

- **Read-only:** Creates tasks only; cannot update existing tasks
- **7-day window:** Fixed; use manual creation for older threads
- **User-scoped:** Only processes threads where the user took action