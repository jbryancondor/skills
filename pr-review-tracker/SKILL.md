---
name: pr-review-tracker
description: >
  Scans a Slack channel for GitHub PR review requests tagged to the user's squad
  and creates Asana subtasks for each pending review. Use this skill whenever the
  user asks to create PR pending to review, list PRs to review, check pending
  reviews, track PR reviews, or manage pull request review tasks — even casual
  phrasing like "what PRs do I need to review", "create my PR review tasks",
  "pending PRs", or "PR reviews for today". By default fetches PRs from today.
model: sonnet
---

# PR Review Tracker

Reads PR review requests from Slack, enriches them with GitHub data, presents a
summary table, and — once confirmed — creates Asana subtasks under a parent task.

## Fixed references

| Resource | ID / Value |
|---|---|
| Slack channel | `C08JA2ANJ06` (review-request channel) |
| Squad group IDs | `S02UNHEKNFJ` (`@merchant-engineering`), `S02U14UUKK6` (`@merchant-platform-frontend`), `S03E7R2912N` (`@merchant-platform-backend`) |
| Current user | Resolve dynamically via `slack_read_user_profile` (no args = logged-in user) |

## Workflow

```
Progress:
- [ ] Step 1: Resolve current user and target date(s)
- [ ] Step 2: Collect PR data (parallel subagents if multi-day)
  - [ ] Step 2a: Read Slack messages and filter relevant ones
  - [ ] Step 2b: Extract GitHub PR URLs from messages
  - [ ] Step 2c: Fetch PR details via GitHub CLI
- [ ] Step 3: Merge results and deduplicate across days
- [ ] Step 4: Build and present the pending PR table
- [ ] Step 5: Ask user to confirm and provide parent Asana task
- [ ] Step 6: Deduplicate against existing Asana subtasks
- [ ] Step 7: Create Asana subtasks
- [ ] Step 8: Show summary with Asana links
```

---

### Step 1: Resolve current user and target date(s)

- Call `mcp__plugin_slack_slack__slack_read_user_profile` with no `user_id` to get
  the logged-in user's `user_id` and `display_name`.
- **Target date(s)**: Determine what the user is asking for:
  - **Single day** (default): today (`YYYY-MM-DD`), or a specific date the user
    mentions.
  - **Multi-day range**: The user may say "last 3 days", "this week",
    "March 5 to March 10", etc. Parse the request into a list of individual dates
    (`YYYY-MM-DD`). For "last N days", generate N dates ending at today. For
    explicit ranges, generate every date from start to end inclusive.
- For each date, compute the Unix timestamps for the start and end of that day —
  these will be the `oldest` and `latest` parameters when reading the channel.

**Squad subteam IDs are hardcoded** — no resolution needed:
- `S02UNHEKNFJ` → `@merchant-engineering`
- `S02U14UUKK6` → `@merchant-platform-frontend`
- `S03E7R2912N` → `@merchant-platform-backend`

---

### Step 2: Collect PR data (parallel subagents if multi-day)

This step gathers Slack messages, extracts PR URLs, and fetches GitHub details.
The strategy depends on whether the request covers one day or multiple days.

#### Single-day path

If there is only **one target date**, execute Steps 2a → 2b → 2c sequentially in
the main agent (no subagent needed).

#### Multi-day path (parallel subagents)

If there are **two or more target dates**, launch one **Agent subagent per day**,
all in parallel. Each subagent independently performs Steps 2a → 2b → 2c for its
assigned date.

Use the `Agent` tool with `subagent_type: "general-purpose"` and
`model: "sonnet"`. Launch **all subagents in a single message** so they run
concurrently. Each subagent prompt must include:

1. The specific date to process (as `YYYY-MM-DD`)
2. The channel ID: `C08JA2ANJ06`
3. The logged-in user's `user_id` (resolved in Step 1)
4. The squad subteam IDs: `S02UNHEKNFJ`, `S02U14UUKK6`, `S03E7R2912N`
5. The full instructions for Steps 2a, 2b, and 2c below
6. Instruction to return a structured summary: a list of PR records with all
   fields from Steps 2b and 2c

**Example subagent prompt template:**

```
Scan Slack channel C08JA2ANJ06 for PR review requests on DATE.

Context:
- Logged-in user ID: USER_ID
- Squad subteam IDs: S02UNHEKNFJ, S02U14UUKK6, S03E7R2912N
- Unix timestamp range: oldest=OLDEST, latest=LATEST

Instructions:
1. Read messages from channel C08JA2ANJ06 using slack_read_channel with
   oldest=OLDEST, latest=LATEST, limit=100. Paginate if needed.
2. Keep messages that mention any of: <!subteam^S02UNHEKNFJ>,
   <!subteam^S02U14UUKK6>, <!subteam^S03E7R2912N>, <@USER_ID>,
   or literal strings @merchant-engineering, @merchant-platform-backend,
   @merchant-platform-frontend.
3. Extract GitHub PR URLs matching: https://github.com/.../pull/NNN
4. For each PR URL, record: requester (resolve display name via
   slack_read_user_profile), request date, Slack permalink
   (https://app.slack.com/archives/C08JA2ANJ06/p<ts_without_dot>).
5. Deduplicate by PR URL (keep earliest).
6. For each unique PR, run:
   gh pr view <URL> --json title,body,additions,deletions,state,mergedAt,author,url,number
7. Filter out PRs where mergedAt is non-empty or state == "MERGED".
8. Create a short description (~100 chars) from the PR body.

Return a structured list of PR records. For each PR include:
- pr_url, repo, number, title, short_description, author_login
- additions, deletions, state
- requester_name, request_date, slack_permalink
If no relevant PRs found, return an empty list.
```

Wait for all subagents to complete, then collect their results into a single list.

---

#### Step 2a: Read Slack messages and filter relevant ones

Read messages from channel `C08JA2ANJ06` using
`mcp__plugin_slack_slack__slack_read_channel` with:
- `channel_id: "C08JA2ANJ06"`
- `oldest: <start_of_target_date_unix_timestamp>`
- `latest: <end_of_target_date_unix_timestamp>`
- `limit: 100`

If there are more messages, paginate using `cursor` until all messages for the
target date are retrieved.

**Filter criteria** — keep a message only if it matches ANY of these:
1. Message text contains `<!subteam^S02UNHEKNFJ>`, `<!subteam^S02U14UUKK6>`, or
   `<!subteam^S03E7R2912N>`
2. Message text contains `<@USER_ID>` where `USER_ID` matches the logged-in user
3. Message text contains the literal strings `@merchant-engineering`,
   `@merchant-platform-backend`, or `@merchant-platform-frontend`

Also check thread parent messages — if a thread parent matches, include it.

---

#### Step 2b: Extract GitHub PR URLs from messages

For each filtered message, extract GitHub pull request URLs using this regex
pattern:

```
https://github\.com/[^\s>|]+/pull/\d+
```

This captures URLs like:
- `https://github.com/org/repo/pull/123`
- Wrapped in Slack formatting: `<https://github.com/org/repo/pull/123>`
- With link text: `<https://github.com/org/repo/pull/123|PR title>`

For each extracted URL, also record:
- **Requester**: The Slack user who posted the message (from `user` field).
  Resolve their display name via `slack_read_user_profile`.
- **Request date**: The message timestamp, converted to a human-readable date.
- **Slack message link**: The permalink to the original Slack message where the PR
  review was requested. Construct it as
  `https://app.slack.com/archives/C08JA2ANJ06/p<ts_without_dot>` (replace the dot
  in the message `ts` with nothing). This link is shown in both the summary table
  and in each Asana subtask.

Deduplicate by PR URL — if the same PR appears in multiple messages, keep the
earliest occurrence.

---

#### Step 2c: Fetch PR details via GitHub CLI

For each unique PR URL, run:

```bash
gh pr view <URL> --json title,body,additions,deletions,state,mergedAt,author,url,number
```

Capture:
- `title` — PR title
- `body` — PR description (for short summary)
- `additions` + `deletions` — total lines changed (PR size)
- `state` — OPEN / CLOSED / MERGED
- `mergedAt` — non-empty string if merged (the `merged` field does NOT exist in
  `gh pr view`; use `mergedAt` instead)
- `author.login` — GitHub author
- `number` — PR number
- `url` — canonical URL

**Filter out** any PR where `mergedAt` is non-empty or `state == "MERGED"`. These
are already completed and not pending review.

**Create a short description** from the PR body:
- Take the first 1-2 meaningful sentences or the first paragraph
- Strip markdown formatting
- Limit to ~100 characters
- If the body is empty, use the PR title as the description

---

### Step 3: Merge results and deduplicate across days

If the request covered multiple days, the same PR may have been requested on more
than one day (e.g., a re-ping). Merge all PR records from all subagents into a
single list and deduplicate by PR URL — keep the **earliest** occurrence (oldest
request date).

For single-day requests, this step is a no-op (already deduplicated in Step 2b).

---

### Step 4: Build and present the pending PR table

Present a formatted table to the user with all pending PRs:

```
## Pending PR Reviews — [target_date or date_range]

| # | Requester | Short Description | PR Link | Slack Message | Requested | Size |
|---|-----------|-------------------|---------|---------------|-----------|------|
| 1 | @user1 | Fix payment retry logic | [org/repo#123](pr_url) | [View in Slack](slack_permalink) | 10:30 AM | +152 / -23 (175 lines) |
| 2 | @user2 | Add merchant onboarding API | [org/repo#456](pr_url) | [View in Slack](slack_permalink) | 2:15 PM | +890 / -45 (935 lines) |

Total: [N] PRs pending review
```

**Size classification hint** (include as a note below the table):
- Small: < 100 lines changed
- Medium: 100–500 lines changed
- Large: > 500 lines changed

If no pending PRs are found, report:
> "No pending PR review requests found for [target_date/range] in the channel.
> Either no reviews were requested, or all requested PRs have already been merged."

Stop the workflow here if there are no pending PRs.

---

### Step 5: Ask user to confirm and provide parent Asana task

First, ask the user to confirm the information:

> "Does this look correct? You can remove any PRs you don't want to track, or
> let me know if anything needs correction."

**Wait for user confirmation before proceeding.**

Once confirmed, ask for the parent Asana task:

> "What is the parent Asana task for these PR reviews? You can provide:
> - An Asana task URL (e.g., https://app.asana.com/0/project/task)
> - A task ID
> - Or I can search for a task called 'Pull Request Revision'"

If the user says to search or just confirms the default name:
1. Use `mcp__plugin_asana_asana__asana_typeahead_search` with
   `resource_type: "task"` and `query: "Pull Request Revision"`
2. Present the results and ask the user to pick the correct one

Extract the parent task GID from whatever the user provides.

---

### Step 6: Deduplicate against existing Asana subtasks

Users often run this skill daily, so the same PR may appear across multiple days
until it's merged. Creating duplicate Asana tasks wastes time and clutters the
task list, so check for existing tasks before creating new ones.

Call `mcp__plugin_asana_asana__asana_get_task` on the parent task GID with
`opt_fields: "subtasks,subtasks.name"` to fetch all existing subtask names in one
call. If the response does not include subtask names (some Asana plans limit nested
opt_fields), fall back to `mcp__plugin_asana_asana__asana_search_tasks` with
`text: "repo#number"` for each pending PR instead.

1. Collect every existing subtask name from the parent task.
2. For each pending PR, build the `repo#number` identifier (e.g. `org/repo#123`).
3. Check whether **any** existing subtask name contains `repo#number` (the naming
   convention from Step 7). The match should be case-insensitive.
4. If a match is found, mark that PR as **already tracked** and skip it:
   > "Skipped [repo#number] — already exists as Asana task"
5. Only proceed to create tasks for PRs that have no existing match.

If **all** PRs already have Asana tasks, report:
> "All [N] PRs already have Asana subtasks. No new tasks created."

And stop the workflow.

---

### Step 7: Create Asana subtasks

For each confirmed pending PR **that was not skipped in Step 6**, create a subtask using
`mcp__plugin_asana_asana__asana_create_task` with:

```
name: "[Short PR Description] - [repo#number]"
parent: "<parent_task_gid>"
notes: |
  Pull Request Review

  PR: [PR title]
  PR Link: [PR URL]
  Slack Request: [Slack message permalink]
  Author (GitHub): [author.login]
  Requested by (Slack): [requester display name]
  Requested on: [request date and time]
  Size: +[additions] / -[deletions] ([total] lines changed)

  Description:
  [Short description from PR body]
```

Use `notes` (plain text), not `html_notes` — PR descriptions often contain angle
brackets and special characters that break HTML parsing in the Asana API.

Process subtasks sequentially (one at a time) to stay within Asana's rate limits.

---

### Step 8: Show summary with Asana links

After all subtasks are created, present a summary:

```
## PR Review Tasks Created

Parent task: [Parent task name] ([Asana link])

| # | PR | Slack Request | Asana Task |
|---|-----|---------------|------------|
| 1 | [org/repo#123] — Fix payment retry | [View in Slack](slack_permalink) | [View in Asana](https://app.asana.com/0/0/<task_gid>) |
| 2 | [org/repo#456] — Add merchant API | [View in Slack](slack_permalink) | [View in Asana](https://app.asana.com/0/0/<task_gid>) |

Total: [N] tasks created successfully.
```

If any task creation failed, list the failures separately with the error reason.

---

## Error handling

| Error | Action |
|-------|--------|
| Channel not accessible | Verify MCP Slack connection with `/mcp` |
| `gh` CLI not authenticated | Run `gh auth login` first |
| PR URL returns 404 | Skip PR, note in output as "PR not found (may be in private repo)" |
| Asana parent task not found | Ask user to provide a valid task ID or URL |
| Rate limit from GitHub | Add 1-second delay between `gh pr view` calls |
| No messages in date range | Report clearly and stop |

## Limitations

- Only processes messages from `C08JA2ANJ06` — PRs shared in other channels or
  DMs are not captured
- GitHub CLI must be authenticated and have access to the repos
- Only creates subtasks; does not update or close them when PRs are merged
- Date range defaults to today; the user can specify a custom range
