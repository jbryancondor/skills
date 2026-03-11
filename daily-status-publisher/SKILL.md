---
name: daily-status-publisher
description: >
  Creates and publishes a daily async status update to the team's Slack channel thread.
  Use this skill whenever the user asks to write, create, post, send, or generate their
  daily standup, daily update, async daily, or daily status — even if they phrase it
  casually like "do my daily", "write my standup", "post my update today", or "send my
  async". Also trigger when the user says things like "what should I put in my daily"
  or "help me write today's status".
---

# Daily Status Publisher

Generates and posts the user's daily async update to the team Slack thread in
**#merchant-platform** (`C03DP13RJPP`), drawing from Asana task activity across three
boards.

## Fixed references

| Resource | ID / Value |
|---|---|
| Slack channel | `C03DP13RJPP` |
| Asana: Merchant Platform - Execution Board | `1205194975006557` |
| Asana: L3 OnCall Escalations - Merchant Channels & Activation | `1209570512740114` |
| Asana: L3 OnCall Escalations - Merchant Operations | `1213345436349848` |

## Daily status template

The update must answer these three questions:

```
*What did you accomplish on the previous business day?* What went well, what didn't go well.

*What are you working on today?* What is the most critical thing you should work on today?

*Are there any roadblocks or need help?* Ask for help in advance, don't wait until being
fully blocked — if you are in doubt, ask for help preemptively.
```

---

## Workflow

```
Progress:
- [ ] Step 1: Resolve target date and previous business day
- [ ] Step 2: Fetch Asana tasks across all three boards
- [ ] Step 3: Show task list and ask user which to include
- [ ] Step 4: Build answers for Q1 (yesterday) and Q2 (today)
- [ ] Step 5: Infer roadblocks for Q3
- [ ] Step 6: Present draft and confirm with user
- [ ] Step 7: Find or handle missing Slack thread
- [ ] Step 8: Post the update
```

---

### Step 1: Resolve dates

- **Target date** = today (`YYYY-MM-DD`) unless the user explicitly specifies a different date.
- **Previous business day**: subtract one calendar day, skipping back over Saturday/Sunday.
  - Monday → Friday of last week
  - Tuesday–Friday → the day before

Do not ask the user for these dates; compute them automatically.

---

### Step 2: Fetch Asana tasks

**Always resolve the logged-in Asana user first.** Call
`mcp__plugin_asana_asana__asana_get_user` with `user_id: "me"` and capture the
returned `gid`. Never hardcode a user GID — all task filtering must use the GID of
the authenticated account so the skill works correctly regardless of who is running it.

Then run **three parallel searches** using `mcp__plugin_asana_asana__asana_search_tasks`,
one per board, filtering by:
- `assignee_any: <gid_from_logged_in_user>`
- `modified_on_after: <previous_business_day>` — only tasks touched on or after the
  previous business day (avoids surfacing stale unrelated tasks)
- Include both completed and incomplete tasks

For each task returned, also fetch stories (comments) using
`mcp__plugin_asana_asana__asana_get_stories_for_task` to understand recent activity.

---

### Step 3: Show task list and ask user which to include

Do not silently classify or filter tasks on your own. Instead, present **all tasks
found** as a numbered list grouped by board, showing key metadata for each so the user
can decide what's relevant for this day's update:

```
Tasks found for [previous_business_day] → [target_date]:

📋 Merchant Platform - Execution Board
  1. [Task name] — due: [due_on] | status: [complete/incomplete] | last activity: [date]
  2. [Task name] — due: [due_on] | status: [complete/incomplete] | last activity: [date]

📋 L3 OnCall - Merchant Channels & Activation
  3. [Task name] — due: [due_on] | status: [complete/incomplete] | last activity: [date]

📋 L3 OnCall - Merchant Operations
  4. [Task name] — due: [due_on] | status: [complete/incomplete] | last activity: [date]

Which tasks should I include in your daily status? (e.g. "1, 3, 4" or "all")
You can also tell me which belong to yesterday vs. today if the due dates aren't clear.
```

Wait for the user's response before proceeding. The user may:
- Pick specific numbers
- Say "all"
- Exclude some with "all except 2"
- Clarify which bucket a task belongs to (yesterday vs. today)

Carry only the selected tasks into Step 4.

---

### Step 4: Build Q1 and Q2 answers

**Q1 — What did you accomplish yesterday?**

For each completed task, generate a one-line summary. If the task has stories with
meaningful content (progress notes, resolutions), incorporate the key outcome.
Group by board if there are many tasks. Mention if something went particularly well or
had friction (infer from comments if available).

Format example:
```
✅ [Task name] — [one-line outcome or key detail from comments] ([asana task](https://app.asana.com/0/0/<task_gid>))
```

Only append the `([asana task](...))` link when the item is backed by an Asana task.
If the line was written from general context with no associated task, omit the link entirely.

If no tasks were completed, say so honestly and look for in-progress tasks with recent
comments that show work was done even without formal completion.

**Q2 — What are you working on today?**

List tasks due today or currently in progress. If there are multiple, identify which
looks most critical (e.g., due soonest, highest impact based on name/description, or
flagged in comments). Lead with that one.

Format example:
```
🔄 [Task name] — [brief context, why it's a priority if known] ([asana task](https://app.asana.com/0/0/<task_gid>))
```

Same rule: only include the link when there is a real Asana task behind the item.

---

### Step 5: Infer Q3 — Roadblocks

Scan for signals of blockers across all tasks:
- Tasks that are overdue (due_on < target_date) and still incomplete
- Comments that mention words like "waiting", "blocked", "pending", "dependency",
  "need input", "unclear", "help"
- Tasks without any recent story activity that are also overdue

Draft a short summary of what you found. This is speculative — you'll ask the user
to confirm or edit before posting.

If nothing looks blocked, say: *"No blockers inferred from Asana — is there anything
you'd like to flag?"*

---

### Step 6: Present draft and confirm

Show the full draft in the chat using this exact format:

```
📋 *Daily Status Draft — [target_date]*

*What did you accomplish on the previous business day?*
[Q1 content]

*What are you working on today?*
[Q2 content]

*Are there any roadblocks or need help?*
[Q3 content]
```

Then ask:

> "Does this look right? You can confirm, edit any section, or add more context — especially for roadblocks. Once you say yes, I'll post it to the thread."

**Do not proceed to Step 7 until the user confirms.**

Apply any requested edits and show the updated draft. Repeat until confirmed.

---

### Step 7: Find the Slack thread

Search `C03DP13RJPP` (`#merchant-platform`) for a message matching
`"Async Daily - [target_date]"` using `mcp__plugin_slack_slack__slack_search_public`
or read recent channel history with `mcp__plugin_slack_slack__slack_read_channel`.

The parent thread message has the format:
```
Async Daily - YYYY-MM-DD
```

**If the thread is found:** extract its `ts` (timestamp) — this becomes the
`thread_ts` for your reply.

**If the thread is NOT found:** do not guess or fabricate a link. Ask the user:

> "I couldn't find today's Async Daily thread in #merchant-platform. Could you paste
> the Slack message link here so I can reply to it directly?"

Parse the `thread_ts` from the URL the user provides. Slack message URLs follow this
pattern:
```
https://*.slack.com/archives/CHANNEL_ID/pTIMESTAMP?thread_ts=THREAD_TS&...
```
The `thread_ts` query parameter is the parent message timestamp. If it's absent,
derive it from the `p` path segment by inserting a decimal after the 10th digit
(e.g., `p1772719209132939` → `1772719209.132939`).

---

### Step 8: Post the update

If a thread was found, post the confirmed message as a reply using
`mcp__plugin_slack_slack__slack_send_message` with:
- `channel_id: "C03DP13RJPP"`
- `thread_ts: <parent_message_ts>`
- `message: <confirmed_draft>`

Format the Slack message using mrkdwn: use `*bold*` for the question labels and plain
text for the answers. Keep the three-section structure intact.

Confirm to the user with the message link once posted.
