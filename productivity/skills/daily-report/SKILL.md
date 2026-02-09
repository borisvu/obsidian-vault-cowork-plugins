---
name: daily-report
description: Generate daily standup reports from Work Diary entries. Resolves Obsidian links, cleans tags, summarizes work across multiple days. Use when user needs to compile their daily report or review recent work.
---

# Daily Report Skill

Generate clean daily reports from Work Diary entries.

## Work Diary Location

```
Work Diary/
├── YYYY/
│   └── MM/
│       └── YYYY-MM-DD.md       # Daily notes
└── gggg/
    └── MM/
        └── gggg-[W]ww.md       # Weekly notes (ISO week)
```

## Daily Note Template

```markdown
---
tags:
  - Daily_Report
---
# Goals
- [ ]

# Worked on today
-

# Achievements
-

# To do on the next working day
-

# Free notes
```

## Work Week

**Sunday through Thursday** (Israel).

| Today | Last work day | Next work day |
|-------|---------------|---------------|
| Sunday | Thursday | Monday |
| Monday | Sunday | Tuesday |
| Tuesday | Monday | Wednesday |
| Wednesday | Tuesday | Thursday |
| Thursday | Wednesday | Sunday |
| Friday | Thursday | Sunday |
| Saturday | Thursday | Sunday |

## Core Logic

### Date Range Resolution

- **No argument:** Last work day → today
- **`since YYYY-MM-DD`:** Given date → today
- Skip Friday and Saturday when iterating

### Link Resolution Rules

| Pattern | Resolution |
|---------|-----------|
| `[[XTYPE-1234]]` or `[[IDEA-567]]` | Jira ticket: check vault → check Jira API → use key as fallback |
| `[[Full Name\|Display]]` | Use display text after `\|` |
| `[[Full Name]]` | Use the full name as plain text |
| `[[Any Other Link]]` | Check vault for file, use title if found, link text if not |

### Tag Cleaning

| Tag | Action |
|-----|--------|
| `#planned` | Remove entirely |
| `#unplanned` | Remove entirely |
| All other tags | Preserve |

### Jira Resolution Details

For patterns matching `[[UPPERCASE-DIGITS]]` (e.g., `[[XTYPE-7292]]`, `[[IDEA-123]]`):

1. Search vault using globs: `PARA/1 Projects/*/{KEY}*` and `PARA/3 Resources/Jira/{KEY}*`
2. If found → use the note's `# title` or frontmatter
3. If not found → query Atlassian MCP: get summary, status, assignee
4. If Jira returns data:
   - Format: `{Jira summary} ({KEY})`
   - Offer to create vault note (user confirms placement)
5. If Jira unavailable → just use the key: `(XTYPE-7292)`

### Jira Note Placement Decision

| Condition | Location |
|-----------|----------|
| User is Jira assignee | `PARA/1 Projects/{KEY} - {Summary}/{KEY} - {Summary}.md` |
| User says they work on it | `PARA/1 Projects/{KEY} - {Summary}/{KEY} - {Summary}.md` |
| Otherwise | `PARA/3 Resources/Jira/{KEY} - {Summary}.md` |

### Jira Note Template

```markdown
---
aliases:
  - {KEY}
tags:
  - jira
  - {project-prefix-lowercase}
creation_date: {YYYY-MM-DD}
last_updated: {YYYY-MM-DD}
Status: {status from Jira}
URL: https://xtypeio.atlassian.net/browse/{KEY}
---
## Overview

{Jira summary}. {One line of context if available.}

**Jira:** [{KEY}](https://xtypeio.atlassian.net/browse/{KEY})
**Status:** {status}
**Assignee:** [[{assignee full name}]]
**Project:** {Jira project name}

## Related

- [[{related people or projects if known}]]
```

## Output Formats

### Single Day Report

```
## Daily Report — YYYY-MM-DD (Day of week)

### What I worked on
- [resolved, cleaned items from "Worked on today"]

### Achievements
- [from "Achievements" section, if non-empty]

### Goals status
- [checkbox items from "Goals" with ✓/✗ status]

### Plan for next work day ([Day], YYYY-MM-DD)
- [from "To do on the next working day"]

### Notes
- [from "Free notes", if non-empty]
```

### Multi-Day Report

```
## Daily Report — YYYY-MM-DD to YYYY-MM-DD

### Summary
[2-3 sentence overview of work across all days]

### Day-by-day

#### [Day], YYYY-MM-DD
**Worked on:**
- [cleaned items]

**Achieved:**
- [if any]

[...for each day with a note...]

### Carry-forward for [Next work day name], YYYY-MM-DD
- [from most recent "To do on the next working day"]
- [any unchecked goals from the period]

### Observations
- [patterns: recurring themes, blocked items]
- [suggestions for prioritization]
```

## Important Rules

- **Read-only** on Work Diary files — never modify daily/weekly notes
- Jira note creation requires user confirmation
- If a daily note doesn't exist for a date, mention it and continue
- Empty sections (just `- ` with no content) should be omitted from the report
- The report is output in **chat**, not written to a file
