---
description: Generate a daily standup report from Work Diary entries
argument-hint: "[since YYYY-MM-DD]"
---

# Daily Report Command

> Generate a summary of recent work from Work Diary entries.

Reads daily notes, resolves Obsidian links, cleans up tags, and produces a ready-to-send daily report in chat. Designed to make daily reporting painless — even when you've missed a few days.

## Usage

```
/para-flow:daily-report
/para-flow:daily-report since 2026-02-06
```

## Work Week

**Sunday through Thursday** (Israel work week).

- "Last work day" on Sunday = previous Thursday
- "Last work day" on Monday = Sunday
- "Next work day" after Thursday = Sunday

## Instructions

### 0. Get Current Date and Time

> Use `~~time` MCP to get the current date, time, and day of week. This is the source of truth for all date logic. NEVER guess or calculate dates manually.

### 1. Determine Date Range

**If `since` argument provided:** Use that date as start, today as end.

**If no argument:** Default to last work day only.

- If today is Sunday → last work day is Thursday
- If today is Monday → last work day is Sunday
- Otherwise → yesterday

### 2. Locate Daily Notes

Daily notes live at: `Work Diary/YYYY/MM/YYYY-MM-DD.md`

For each date in the range (work days only — skip Friday and Saturday):

1. Construct the file path
2. **Always attempt to read the file** — never assume a file exists or doesn't exist
3. If the read fails, note: "No daily note found for YYYY-MM-DD"

### 3. Extract Sections

From each daily note, extract content from these sections (per the template):

| Section                           | Purpose                           |
| --------------------------------- | --------------------------------- |
| `# Goals`                         | What was planned (checkbox items) |
| `# Worked on today`               | What was actually done            |
| `# Achievements`                  | Notable completions               |
| `# To do on the next working day` | Carry-forward items               |
| `# Free notes`                    | Additional context                |

### 4. Resolve Obsidian Links

For every `[[...]]` link in extracted content:

**Step 4a: Check if it's a Jira ticket pattern** (`[[PROJECT-NNNN]]`, `[[IDEA-NNNN]]`, or similar `[[UPPERCASE-DIGITS]]`):

1. Search vault for existing note: glob `PARA/1 Projects/*/{KEY}*` and `PARA/3 Resources/Jira/{KEY}*`
2. If file exists → use its title/frontmatter summary
3. If file doesn't exist → query Jira via Atlassian MCP for the ticket summary
4. If Jira returns a result:
   - Use the summary in the report
   - Create a note file (see "Jira Note Creation" below)
5. If Jira is unavailable or no result → use the key as-is: `(PROJECT-7292)`

**Step 4b: Check if it's a person link** (`[[John Smith|John]]` or `[[John Smith]]`):

- Use the display text (after `|`) or the full name
- Strip the link syntax, keep the readable name

**Step 4c: For all other links** (`[[Project Phoenix]]`, `[[Some Document]]`):

- Check if the linked file exists in the vault
- If yes → use the file's title as plain text
- If no → use the link text as plain text

### 5. Clean Tags

Remove these tags from all extracted content:

- `#planned` → remove entirely
- `#unplanned` → remove entirely

Preserve all other tags.

### 6. Compose Report

**For a single day:**

```
## Daily Report — YYYY-MM-DD (Day of week)

### What I worked on
- [cleaned, resolved items from "Worked on today"]

### Achievements
- [items from "Achievements", if any]

### Goals status
- [checkbox items from "Goals" with completion status]

### Plan for next work day ([next work day name, date])
- [items from "To do on the next working day"]

### Notes
- [items from "Free notes", if any non-empty]
```

**For multiple days (since date):**

```
## Daily Report — YYYY-MM-DD to YYYY-MM-DD

### Day-by-day

#### [Day of week], YYYY-MM-DD
**Worked on:**
- [cleaned items]

**Achieved:**
- [if any]

#### [Day of week], YYYY-MM-DD
**Worked on:**
- [cleaned items]

**Achieved:**
- [if any]

[...repeat for each day...]

### Carry-forward for next work day ([day name, date])
- [items from most recent day's "To do on the next working day"]
- [any unchecked goals from the period]
```

Keep it factual and flat. Do not add summaries, observations, or analysis unless the user asks.

### 7. Jira Note Creation (Side Effect)

During link resolution (step 4a), if a Jira ticket was queried and no vault note exists, create one:

**Determine placement:**

- Query Jira for assignee
- If user is the assignee → `PARA/1 Projects/{KEY} - {Summary}/{KEY} - {Summary}.md` (create subdirectory)
- Otherwise → `PARA/3 Resources/Jira/{KEY} - {Summary}.md` (create `Jira/` subfolder if needed)

**Ask user** before creating: "I found references to these Jira tickets without vault notes. Create them?"

List tickets with proposed locations. User confirms or overrides placement.

### 8. Weekly Alignment Check

After composing the report, check the current week's goals:

1. **Get the ISO week number** from the time MCP (step 0)
2. **Find the weekly note** at `Work Diary/YYYY/MM/YYYY-Www.md` (e.g., `Work Diary/2026/02/2026-W07.md`)
3. **If the file doesn't exist:** Append to the report: "No weekly note found for week Www. Consider creating one."
4. **If the file exists but `# Week Goals` is empty or missing:** Append: "Weekly note exists but has no goals. Consider adding them."
5. **If goals exist:** Read the `# Week Goals`, `# My Quarter Vision`, and `# My Year Vision` sections. Compare today's work items against the week goals. Append:

```
### Week goals (Www)
- [x] Goal that was addressed today — brief note of what matched
- [ ] Goal not yet addressed — not yet addressed
> Consider prioritizing [unaddressed goal] on your next work day.
```

Use the quarter and year vision for context when writing the nudge — reference higher-level goals if the unaddressed item connects to them. Keep the nudge to one line.

The `# Priorities` table (P0/P1/P2) can inform which unaddressed goals to nudge first.

### 9. Output

Present the composed report directly in chat. The user can copy/paste it wherever needed.

## Jira Note Template

When creating a new Jira ticket note (must match existing vault conventions):

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

{Jira summary/title}. {One line of context from description if available.}

**Jira:** [{KEY}](https://<company_name>.atlassian.net/browse/{KEY})
**Status:** {status}
**Assignee:** [[{assignee full name}]]
**Project:** {Jira project name}

## Related

- [[{any related people or projects if known}]]
```

Note: `aliases` includes the Jira key so `[[PROJECT-7292]]` wikilinks resolve. `URL` field (not `jira-url`) matches existing vault convention. Assignee is a wikilink. Filename uses `{KEY} - {Summary}` format matching vault convention.

## Examples

### Single day

```
User: /para-flow:daily-report

[Reads Work Diary/2026/02/2026-02-09.md]
```

Input (from daily note):

```
# Worked on today
- Prepared for a sync with [[John Smith|John]] tomorrow. #planned
- Worked on a production issue - [[PROJECT-7292]]. #unplanned
```

Output:

```
## Daily Report — 2026-02-09 (Sunday)

### What I worked on
- Prepared for a sync with John tomorrow.
- Worked on a production issue — Audit operation ended without results (PROJECT-7292).

### Plan for next work day (Monday, 2026-02-10)
- [items from "To do on the next working day" section]
```

### Multiple days

```
User: /para-flow:daily-report since 2026-02-05

[Reads daily notes for Feb 5 (Wed), 6 (Thu), 9 (Sun) — skips Fri/Sat]
```

## Notes

- If a daily note file doesn't exist for a date in range, mention it but continue with available days
- Work days: Sunday, Monday, Tuesday, Wednesday, Thursday
- Non-work days: Friday, Saturday
- Never modify the original daily note files — this is read-only
- Jira note creation is the only write operation, and requires confirmation
- If Atlassian MCP is not connected, Jira links resolve to their key text only (e.g., "PROJECT-7292")
- The report is output in chat, not written to a file
