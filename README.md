# Custom Skills

This repository is a local library of reusable custom skills for your IDE agent.

## Link Skills to Your Local Folder (Symlink)

To use these skills from your personal setup, create symlinks from this repo into your local skills directory (`~/.claude/skills`).

### Quick steps

1. Ensure your local skills folder exists:

```bash
mkdir -p ~/.claude/skills
```

2. Link each skill using this pattern:

```bash
ln -s ~/bcd/project/custom-skills/asana-task-creator-for-marketplace-cancellation ~/.claude/skills/asana-task-creator-for-marketplace-cancellation
ln -s ~/bcd/project/custom-skills/generating-l3-oncall-report ~/.claude/skills/generating-l3-oncall-report
```

3. Reload your IDE/agent so it re-indexes local skills.

### Symlink pattern

```bash
ln -s <SOURCE_SKILL_FOLDER> <LOCAL_SKILLS_FOLDER>/<SKILL_NAME>
```

- `<SOURCE_SKILL_FOLDER>`: folder inside this repository.
- `<LOCAL_SKILLS_FOLDER>`: usually `~/.claude/skills`.
- `<SKILL_NAME>`: final skill folder name shown by your agent.

## Skills

### 1) `creating-asana-tasks-from-marketplace-cancellations`

Path: `asana-task-creator-for-marketplace-cancellation/SKILL.md`

**Purpose:** Turn unresolved marketplace cancellation Slack threads into structured Asana subtasks.

**Includes:**
- Slack thread scan (`#cancellations-marketplace`) for the last 7 days.
- Deduplication against existing subtasks by Loan ID.
- Priority classification (`P1`, `P3`, `P4`, `P5`).
- Summary report of created and skipped tasks.

### 2) `generating-l3-oncall-report`

Path: `generating-l3-oncall-report/SKILL.md`

**Purpose:** Build the weekly L3 on-call handoff report and publish it to Notion after approval.

**Includes:**
- Input collection for rotation dates, engineers, screenshots, and events.
- Data aggregation from Notion incidents, Asana boards, and Slack alerts.
- Markdown preview flow requiring explicit user confirmation.
- Final Notion page creation with structured sections.

Supporting docs:
- `generating-l3-oncall-report/DATA-SOURCES.md`
- `generating-l3-oncall-report/TEMPLATE.md`
- `generating-l3-oncall-report/EXAMPLE-OUTPUT.md`
