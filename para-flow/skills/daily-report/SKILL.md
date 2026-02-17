---
name: daily-report
description: Generate daily standup reports from Work Diary entries. Resolves Obsidian links, cleans tags, summarizes work across multiple days. Use when user needs to compile their daily report or review recent work.
---

# Daily Report Skill

Generate clean daily reports from Work Diary entries. This skill is organized into numbered sections that commands reference directly.

## Section 1: Core Concepts

### Work Diary Location

```
Work Diary/
├── YYYY/
│   └── MM/
│       └── YYYY-MM-DD.md       # Daily notes
└── gggg/
    └── MM/
        └── gggg-[W]ww.md       # Weekly notes (ISO week)
```

### Daily Note Template

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

### Work Week

**Sunday through Thursday** (Israel).

| Today     | Last work day | Next work day |
| --------- | ------------- | ------------- |
| Sunday    | Thursday      | Monday        |
| Monday    | Sunday        | Tuesday       |
| Tuesday   | Monday        | Wednesday     |
| Wednesday | Tuesday       | Thursday      |
| Thursday  | Wednesday     | Sunday        |
| Friday    | Thursday      | Sunday        |
| Saturday  | Thursday      | Sunday        |

## Section 2: Time Awareness

> Before any date operations, use `~~time` MCP to get the current date, time, and day of week. This is the source of truth. NEVER guess or calculate dates manually.

## Section 3: Date Range Resolution

- **No argument:** Last work day → today
- **`since YYYY-MM-DD`:** Given date → today
- Skip Friday and Saturday when iterating

## Section 4: Section Extraction

From each daily note, extract content from these sections (per the template in Section 1):

| Section                           | Purpose                           |
| --------------------------------- | --------------------------------- |
| `# Goals`                         | What was planned (checkbox items) |
| `# Worked on today`               | What was actually done            |
| `# Achievements`                  | Notable completions               |
| `# To do on the next working day` | Carry-forward items               |
| `# Free notes`                    | Additional context                |

**Always attempt to read each file** — never assume a file exists or doesn't exist. If the read fails, note: "No daily note found for YYYY-MM-DD".

Empty sections (just `- ` with no content) should be omitted from the report.

## Section 5: Link Resolution

For every `[[...]]` link in extracted content:

### Step 5a: Jira ticket pattern

Matches `[[UPPERCASE-DIGITS]]` (e.g., `[[PROJECT-7292]]`, `[[IDEA-123]]`):

1. Search vault using globs: `PARA/1 Projects/*/{KEY}*` and `PARA/3 Resources/Jira/{KEY}*`
2. If file exists → use its title/frontmatter summary
3. If not found → query Atlassian MCP for ticket summary
4. If Jira returns data → use the summary, offer to create vault note (Section 7)
5. If Jira unavailable → use the key as-is

**Mode-aware formatting:**

| Mode    | Format                                                 | Example                           |
| ------- | ------------------------------------------------------ | --------------------------------- |
| standup | Key inline with context                                | `Merged PROJECT-7371`             |
| full    | Full description with key in parentheses               | `Audit operation ended (PROJECT-7292)` |

### Step 5b: Person link

`[[Full Name|Display]]` or `[[Full Name]]`:

- Use display text (after `|`) or the full name
- Strip link syntax, keep readable name

### Step 5c: Other links

`[[Project Phoenix]]`, `[[Some Document]]`, etc.:

- Check if the linked file exists in the vault
- If yes → use the file's title as plain text
- If no → use the link text as plain text

## Section 6: Tag Cleaning

| Tag            | Action          |
| -------------- | --------------- |
| `#planned`     | Remove entirely |
| `#unplanned`   | Remove entirely |
| All other tags | Preserve        |

## Section 7: Jira Note Management

### Placement Decision

| Condition                 | Location                                                 |
| ------------------------- | -------------------------------------------------------- |
| User is Jira assignee     | `PARA/1 Projects/{KEY} - {Summary}/{KEY} - {Summary}.md` |
| User says they work on it | `PARA/1 Projects/{KEY} - {Summary}/{KEY} - {Summary}.md` |
| Otherwise                 | `PARA/3 Resources/Jira/{KEY} - {Summary}.md`             |

**Ask user** before creating: "I found references to these Jira tickets without vault notes. Create them?"

### Jira Note Template

```markdown
---
aliases:
  - { KEY }
tags:
  - jira
  - { project-prefix-lowercase }
creation_date: { YYYY-MM-DD }
last_updated: { YYYY-MM-DD }
Status: { status from Jira }
URL: https://<company_name>.atlassian.net/browse/{KEY}
---

## Overview

{Jira summary}. {One line of context if available.}

**Jira:** [{KEY}](https://<company_name>.atlassian.net/browse/{KEY})
**Status:** {status}
**Assignee:** [[{assignee full name}]]
**Project:** {Jira project name}

## Related

- [[{related people or projects if known}]]
```

Note: `aliases` includes the Jira key so `[[PROJECT-7292]]` wikilinks resolve. `URL` field (not `jira-url`) matches existing vault convention. Assignee is a wikilink. Filename uses `{KEY} - {Summary}` format.

## Section 8a: Standup Format

Laconic format for daily standups. Two sections only: what was done, what's planned.

### Section Filtering

Only include content from:
- `# Worked on today` — previous work day(s)
- `# Goals` — today's planned work
- `# Achievements` — notable completions (append to "completed" section)

**Exclude** from standup output:
- `# Free notes` — private/informal content
- `# To do on the next working day` — internal planning, not team-facing

### Morning Mode

If the current time is before 10:00 AM on a work day:
- Treat today's `# Worked on today` as empty (work hasn't happened yet)
- Treat all of today's `# Goals` as unchecked (nothing started yet)

### Output Template

```
Tasks completed since last report:

[Day name]:
• [Brief item — use standup link formatting from Section 5]

Will do today:
• [Unchecked goals from today's daily note]
```

For multi-day ranges, group completed items by day name.

If there are achievements, append them as bullets under the relevant day's completed items.

### Example

```
Tasks completed since last report:

Thursday:
• Merged PROJECT-7371
• Sync with John on API design
• Reviewed pull request for auth module

Will do today:
• Continue PROJECT-7292 investigation
• Prepare demo for sprint review
```

## Section 8b: Full Report Format

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
```

Keep output factual and flat. No summaries or analysis unless the user asks.

## Section 9: Weekly Alignment

Appended to all reports (both standup and full).

1. Get the ISO week number from `~~time` MCP
2. Find the weekly note at `Work Diary/YYYY/MM/YYYY-Www.md`
3. If file missing: "No weekly note found for week Www. Consider creating one."
4. If `# Week Goals` empty/missing: "Weekly note exists but has no goals. Consider adding them."
5. If goals exist: Compare work items against week goals. Output:

```
### Week goals (Www)
- [x] Goal addressed today — brief match note
- [ ] Goal not yet addressed — not yet addressed
> Consider prioritizing [unaddressed goal] on your next work day.
```

Context sections from the weekly note:

- `# My Year Vision` and `# My Quarter Vision` — inform the nudge, not compared item-by-item
- `# Priorities` (P0/P1/P2) — inform which unaddressed goal to nudge first

## Section 10: Important Rules

- **Read-only** on Work Diary files — never modify daily/weekly notes
- Jira note creation requires user confirmation
- **Always read a file** before reporting it as missing — never assume
- Empty sections (just `- ` with no content) should be omitted from the report
- The report is output in **chat**, not written to a file
- Keep output simple, factual, and bulleted. No summaries or analysis unless requested.
