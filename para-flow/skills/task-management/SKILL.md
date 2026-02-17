---
name: task-management
description: Task management using a shared TASKS.md file in the Obsidian vault root. Reference this when the user asks about tasks, wants to add/complete items, or needs help tracking commitments.
---

# Task Management

Tasks are tracked in `TASKS.md` at the vault root. Both you and the user can edit it.

## File Location

**Always use `TASKS.md` in the current working directory (vault root).**

- If it exists, read/write to it
- If it doesn't exist, create it with the template below

## Dashboard

On first interaction with tasks, check if `dashboard.html` exists. If not, copy from `${CLAUDE_PLUGIN_ROOT}/skills/dashboard.html` and inform the user.

## Template

```markdown
# Tasks

## Active

## Waiting On

## Someday

## Done
```

## Task Format

- `- [ ] **Task title** — context, for whom, due date`
- Jira references as plain text with URL: `**PROJECT-7292** — [Audit operation bug](https://<company_name>.atlassian.net/browse/PROJECT-7292), assigned to me`
- Sub-bullets for details
- Completed: `- [x] ~~Task~~ (YYYY-MM-DD)`

## Interaction Patterns

**"What's on my plate" / "my tasks":**

- Read TASKS.md
- Summarize Active and Waiting On
- Highlight overdue or urgent items

**"Add a task" / "remind me to":**

- Add to Active with `- [ ] **Task**` format
- Include context (who, due date, Jira link)

**"Done with X" / "finished X":**

- Change `[ ]` to `[x]`, add strikethrough and date
- Move to Done section

**"What am I waiting on":**

- Read Waiting On section
- Note how long each item has been waiting

## Relationship to Work Diary

- TASKS.md is the **backlog** — the full list of commitments
- Work Diary's "To do on the next working day" is a **daily plan** — a subset for tomorrow
- The `/para-flow:standup` command reads from Work Diary, not TASKS.md
- When processing daily reports, offer to update TASKS.md if completed items match

## Conventions

- **Bold** task titles
- Include "for [person]" for commitments to others
- Include "due [date]" for deadlines
- Include Jira key when applicable
- Keep Done section for ~1 week, then clear
