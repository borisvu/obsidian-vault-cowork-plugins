---
description: Generate a daily standup report from Work Diary entries
argument-hint: "[--full] [since YYYY-MM-DD]"
---

# Standup Command

> Generate a standup summary or full daily report from Work Diary entries.

Default mode produces a laconic standup. Use `--full` for the comprehensive report format.

## Usage

```
/para-flow:standup                          → laconic standup (default)
/para-flow:standup --full                   → comprehensive daily report
/para-flow:standup since 2026-02-06         → standup format, date range
/para-flow:standup --full since 2026-02-06  → full format, date range
```

## Instructions

### Step 0: Parse Arguments and Get Time

1. Parse the argument string:
   - `--full` flag → use full report mode (skill Section 8b). Without it → standup mode (skill Section 8a).
   - `since YYYY-MM-DD` → pass date to skill Section 3.
2. Use `~~time` MCP to get the current date, time, and day of week (skill Section 2).
3. **Morning mode detection:** If the current time is before 10:00 AM on a work day and no `--full` flag, activate morning mode per skill Section 8a.

### Step 1: Determine Date Range

Use daily-report skill Section 3 (Date Range Resolution) with the work week from skill Section 1.

- If `since` argument provided → that date through today
- If no argument → previous work day through today

### Step 2: Locate and Read Daily Notes

Use daily-report skill Section 1 (Core Concepts) for file paths: `Work Diary/YYYY/MM/YYYY-MM-DD.md`

For each work day in the range, attempt to read the file. Note any missing files.

### Step 3: Extract Sections

Use daily-report skill Section 4 (Section Extraction) to read daily note sections.

**Standup mode** — apply section filter from skill Section 8a:
- Include: `# Worked on today`, `# Goals`, `# Achievements`
- Exclude: `# Free notes`, `# To do on the next working day`

**Full mode** — extract all five sections per skill Section 4.

If morning mode is active, apply morning mode rules from skill Section 8a.

### Step 4: Resolve Links

Use daily-report skill Section 5 (Link Resolution) with the current mode:
- Standup mode → brief key-inline format
- Full mode → full description with key in parentheses

### Step 5: Clean Tags

Use daily-report skill Section 6 (Tag Cleaning).

### Step 6: Compose Report

- **Standup mode** → use skill Section 8a (Standup Format)
- **Full mode** → use skill Section 8b (Full Report Format)

### Step 7: Weekly Alignment

Use daily-report skill Section 9 (Weekly Alignment). Append to both modes.

### Step 8: Jira Note Creation

If any Jira tickets were resolved from API (not vault) during Step 4, offer to create vault notes per skill Section 7 (Jira Note Management).

### Step 9: Output

Present the composed report directly in chat. The user can copy/paste it wherever needed. Follow skill Section 10 (Important Rules).

## Examples

### Standup (default)

```
User: /para-flow:standup

Tasks completed since last report:

Thursday:
• Merged PROJECT-7371
• Sync with John on API design
• Reviewed pull request for auth module

Will do today:
• Continue PROJECT-7292 investigation
• Prepare demo for sprint review
```

### Full report

```
User: /para-flow:standup --full

## Daily Report — 2026-02-09 (Sunday)

### What I worked on
- Prepared for a sync with John tomorrow.
- Worked on a production issue — Audit operation ended without results (PROJECT-7292).

### Plan for next work day (Monday, 2026-02-10)
- [items from "To do on the next working day" section]
```

## Notes

- Work days: Sunday, Monday, Tuesday, Wednesday, Thursday (skill Section 1)
- Never modify Work Diary files — this is read-only (skill Section 10)
- Jira note creation is the only write operation, and requires confirmation (skill Section 7)
- If Atlassian MCP is not connected, Jira links resolve to their key text only
- The report is output in chat, not written to a file
