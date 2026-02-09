# Implementation Spec: Personal Productivity Plugin

> **For:** Claude Code to implement  
> **Based on:** Anthropic's `knowledge-work-plugins/productivity` plugin  
> **Adapted for:** Boris's PARA-based Obsidian workflow

---

## Overview

Create a Git repository `boris-plugins` containing a single plugin `productivity` — a customized fork of Anthropic's productivity plugin adapted to work with an existing Obsidian vault that follows the PARA method.

The plugin runs in Cowork (Anthropic's agentic desktop app) or Claude Code. It assumes the **working directory is the root of Boris's "Work Notes" Obsidian vault**.

---

## Repository Structure

```
boris-plugins/
├── README.md                          # Marketplace README
├── LICENSE                            # MIT
├── productivity/
│   ├── plugin.json                    # Plugin manifest
│   ├── .mcp.json                      # MCP server connections (Jira)
│   ├── README.md                      # Plugin README
│   ├── CONNECTORS.md                  # Tool category mapping
│   ├── CLAUDE.md                      # Template for working memory (hot cache)
│   ├── commands/
│   │   ├── start.md                   # /productivity:start
│   │   ├── update.md                  # /productivity:update
│   │   └── daily-report.md            # /productivity:daily-report
│   └── skills/
│       ├── memory-management/
│       │   └── SKILL.md               # Memory skill adapted for PARA
│       ├── task-management/
│       │   └── SKILL.md               # Task management skill
│       ├── daily-report/
│       │   └── SKILL.md               # Daily report generation skill
│       └── dashboard.html             # Visual dashboard (from original)
```

---

## Vault Assumptions

The working directory is the Obsidian vault root. The following PARA structure **already exists** and must NOT be created or overwritten:

```
./PARA/
├── 0 Inbox/
├── 1 Projects/              # Active projects with goals and end dates
├── 2 Areas/
│   ├── People/              # Individual people profiles
│   ├── Partners/            # Partner company profiles
│   └── Customers/           # Customer company profiles
├── 3 Resources/             # Reference material by topic
└── 4 Archive/               # Completed projects, by year

./Work Diary/
├── YYYY/
│   └── MM/
│       └── YYYY-MM-DD.md    # Daily notes
└── gggg/
    └── MM/
        └── gggg-[W]ww.md    # Weekly notes (ISO week, weekday-aware)
```

**Files the plugin creates in the vault root (if missing):**
- `CLAUDE.md` — working memory hot cache
- `TASKS.md` — active task list
- `dashboard.html` — visual dashboard

**Files the plugin reads but NEVER creates:**
- Everything under `./PARA/` (reads existing content)
- Everything under `./Work Diary/` (reads existing daily/weekly notes)

**Files the plugin MAY create inside PARA:**
- New Jira ticket notes in `./PARA/1 Projects/` or `./PARA/3 Resources/`
- New people files in `./PARA/2 Areas/People/`

---

## Obsidian Markdown Requirements

**All `.md` files created by this plugin MUST be valid Obsidian markdown.** This applies to every file the plugin creates — people, companies, projects, Jira tickets, glossary entries.

### Frontmatter

Every created `.md` file (except CLAUDE.md and TASKS.md) MUST include YAML frontmatter with at minimum:

```yaml
---
aliases:
  - {short name or alternate reference}
tags:
  - {relevant tags as YAML array}
creation_date: {YYYY-MM-DD}
last_updated: {YYYY-MM-DD}
---
```

**Tag conventions** (based on existing vault):

| File type | Required tags | Optional tags |
|-----------|--------------|---------------|
| Person | `contact`, `{company-lowercase}` | `{department}`, `{role-keyword}` |
| Customer | `customer`, `xtype`, `servicenow` | — |
| Partner | `partner`, `xtype` | `{specialty}` |
| Jira ticket | `jira`, `{project-prefix-lowercase}` (e.g., `xtype`, `idea`) | `bug`, `feature` |
| Project | `project` | `{status}`, `{team}` |
| Glossary | `reference`, `glossary` | — |

**Additional frontmatter fields by type:**

People:
```yaml
company: "[[Company Name]]"
role: {Role title}
```

Customers:
```yaml
Status: {active|churned|onboarding}
customer_health: {healthy|at-risk|critical}
partner: "[[Partner Name]]"
```

Partners:
```yaml
Status: {active|inactive}
```

Jira tickets:
```yaml
aliases:
  - {XTYPE-1234}
Status: {Open|In Progress|Done|etc from Jira}
URL: https://xtypeio.atlassian.net/browse/{KEY}
```

Projects:
```yaml
Status: {active|completed|on-hold}
```

### Internal Links

**Always use Obsidian wikilinks** for cross-references within the vault:

```markdown
# DO — wikilinks
[[Michael Zingerman]]
[[Michael Zingerman|Michael]]
[[Heineken]]
[[XTYPE-7292]]

# DON'T — markdown links for internal vault references
[Michael](./PARA/2%20Areas/People/Michael%20Zingerman.md)
```

**External links** (Jira URLs, websites) use standard markdown:
```markdown
[XTYPE-7292](https://xtypeio.atlassian.net/browse/XTYPE-7292)
```

### File Structure

Files have **no rigid body template**. Use frontmatter for metadata and free-form sections below. Don't force a fixed structure — match the style of existing vault files.

**General pattern observed in vault:**
```markdown
---
{frontmatter}
---
## Overview
{1-2 sentence summary}

{Remaining sections as needed — tables, notes, whatever fits}

## Related
- [[Link 1]] — context
- [[Link 2]] — context
```

### CLAUDE.md and TASKS.md Exceptions

These two files are **plugin working files**, not Obsidian knowledge base entries:
- No frontmatter required
- They are machine-readable by Claude, not primarily browsed in Obsidian
- Wikilinks are fine in CLAUDE.md for vault cross-references

### Global Rules for All Skills and Commands

Every skill and command file must enforce:
- All created `.md` files (except CLAUDE.md and TASKS.md) MUST have Obsidian frontmatter with: `aliases`, `tags`, `creation_date`
- All internal cross-references MUST use `[[wikilinks]]`
- Follow vault filename conventions: **Title Case with spaces** (e.g., `Michael Zingerman.md`, not `michael-zingerman.md`)
- **Check if a file already exists before creating** — never overwrite vault content
- When linking to people in body text, use `[[Full Name|Nickname]]` format

---

## File: `productivity/plugin.json`

```json
{
  "name": "productivity",
  "version": "1.0.0",
  "description": "PARA-integrated productivity system for Obsidian. Task management, workplace memory mapped to PARA folders, daily report generation from Work Diary, and Jira integration.",
  "author": {
    "name": "Boris"
  }
}
```

---

## File: `productivity/.mcp.json`

```json
{
  "mcpServers": {
    "atlassian": {
      "type": "http",
      "url": "https://mcp.atlassian.com/v1/mcp"
    }
  }
}
```

> Only Atlassian (Jira) for now. Slack, Notion, etc. can be added later.

---

## File: `productivity/CONNECTORS.md`

```markdown
# Connectors

## How tool references work

Plugin files use `~~category` as a placeholder for whatever tool the user connects in that category. Plugins are tool-agnostic — they describe workflows in terms of categories rather than specific products.

## Connectors for this plugin

| Category | Placeholder | Included servers | Other options |
|----------|-------------|-----------------|---------------|
| Issue tracker | `~~issue tracker` | Atlassian (Jira) | Linear, GitHub Issues |
| Chat | `~~chat` | — | Slack, Teams |
| Email | `~~email` | — | Microsoft 365, Gmail |
| Calendar | `~~calendar` | — | Microsoft 365, Google Calendar |
| Knowledge base | `~~knowledge base` | — | Notion, Confluence |
```

---

## File: `productivity/README.md`

```markdown
# Productivity Plugin (PARA Edition)

A productivity plugin for [Cowork](https://claude.com/product/cowork) and Claude Code, adapted for Obsidian vaults using the PARA organizational method.

## What It Does

- **Task management** — `TASKS.md` for the active backlog, readable and writable by Claude
- **Workplace memory** — Two-tier system: `CLAUDE.md` hot cache + deep storage mapped to PARA folders
- **Daily reports** — Generates daily standup reports from Work Diary entries
- **Jira integration** — Resolves Jira ticket links, creates vault entries for referenced tickets
- **Visual dashboard** — HTML board view of tasks and memory

## PARA Mapping

| Plugin concept | PARA location |
|---------------|---------------|
| People profiles | `PARA/2 Areas/People/` |
| Partner companies | `PARA/2 Areas/Partners/` |
| Customer companies | `PARA/2 Areas/Customers/` |
| Active projects | `PARA/1 Projects/` |
| Terms & glossary | `PARA/3 Resources/Terms/` |
| Company context | `PARA/3 Resources/Company/` |
| Archived projects | `PARA/4 Archive/` |
| Daily notes | `Work Diary/YYYY/MM/YYYY-MM-DD.md` |
| Weekly notes | `Work Diary/gggg/MM/gggg-[W]ww.md` |

## Commands

| Command | What it does |
|---------|-------------|
| `/start` | Initialize CLAUDE.md, TASKS.md, dashboard; bootstrap memory from vault |
| `/update` | Triage stale tasks, sync Jira, check memory gaps |
| `/daily-report` | Generate daily report from Work Diary entries |
| `/daily-report since YYYY-MM-DD` | Report covering multiple days since given date |

## Installation

    claude plugins add boris-plugins/productivity
```

---

## File: `productivity/commands/start.md`

```markdown
---
description: Initialize the productivity system and open the dashboard
---

# Start Command

> See [CONNECTORS.md](../CONNECTORS.md) for tool references.

Initialize task and memory systems within an existing PARA-based Obsidian vault, then open the dashboard.

## Instructions

### 1. Verify Vault Structure

Confirm the working directory is an Obsidian vault by checking for:
- `./PARA/` directory with expected subdirectories
- `./Work Diary/` directory

If these don't exist, STOP and tell the user: "This doesn't look like your Work Notes vault. Please run this from the vault root directory."

### 2. Check What Exists

Check the vault root for:
- `CLAUDE.md` — working memory hot cache
- `TASKS.md` — task list
- `dashboard.html` — visual dashboard

### 3. Create What's Missing

**If `TASKS.md` doesn't exist:** Create it with the standard template (see task-management skill). Place in vault root.

**If `dashboard.html` doesn't exist:** Copy from `${CLAUDE_PLUGIN_ROOT}/skills/dashboard.html` to vault root.

**If `CLAUDE.md` doesn't exist:** This is a fresh setup — after opening the dashboard, begin the memory bootstrap (step 5).

### 4. Open the Dashboard

Tell the user: "Dashboard is ready at `dashboard.html`. Open it from your file browser to get started."

### 5. Orient the User

If everything was already initialized:
```
Dashboard open. Your tasks and memory are loaded.
- /productivity:update to sync tasks and check memory
- /productivity:update --comprehensive for a deep scan
- /productivity:daily-report for today's daily report
```

If memory hasn't been bootstrapped yet, continue to step 6.

### 6. Bootstrap Memory (First Run Only)

Only if `CLAUDE.md` doesn't exist.

**Scan existing PARA content to populate CLAUDE.md:**

1. **People:** Read filenames and frontmatter from `PARA/2 Areas/People/`
2. **Partners/Customers:** Read from `PARA/2 Areas/Partners/` and `PARA/2 Areas/Customers/`
3. **Projects:** Read from `PARA/1 Projects/`
4. **Terms:** Check if `PARA/3 Resources/Terms/` exists and read contents

**Then scan TASKS.md** (if it has content) for unresolved shorthand — names, acronyms, project references.

**Ask the user** about anything that can't be decoded from existing vault content:
```
I found some terms in your tasks and notes I want to confirm:

1. **PSR** — What does this stand for?
2. **Michael Z** — Is this Michael Zingerman from People/?
```

### 7. Optional: Scan Jira

If Atlassian MCP is available, offer:
```
Want me to scan your Jira assigned issues to populate tasks and learn project context?
```

If yes, fetch open issues assigned to user, add to TASKS.md, and create project entries.

### 8. Write CLAUDE.md

From everything gathered, create CLAUDE.md using the template in the memory-management skill. Reference PARA paths instead of memory/ paths.

### 9. Report

```
Productivity system ready:
- Tasks: TASKS.md (X items)
- Memory: CLAUDE.md populated (X people, X terms, X projects)
- Deep memory: mapped to PARA/2 Areas/, PARA/1 Projects/, PARA/3 Resources/
- Dashboard: dashboard.html

Use /productivity:daily-report for your daily standup summary.
```

## Notes

- NEVER create PARA directories — they already exist
- If memory is already initialized, this just opens the dashboard and reports status
- Nicknames are critical — capture how people are actually referred to
```

---

## File: `productivity/commands/update.md`

```markdown
---
description: Sync tasks and refresh memory from current activity
argument-hint: "[--comprehensive]"
---

# Update Command

> See [CONNECTORS.md](../CONNECTORS.md) for tool references.

Two modes:
- **Default:** Sync Jira tasks, triage stale items, check memory for gaps
- **`--comprehensive`:** Deep scan of vault content, flag missed items, suggest new memories

## Usage

```
/productivity:update
/productivity:update --comprehensive
```

## Default Mode

### 1. Load Current State

Read `TASKS.md` and `CLAUDE.md`. If they don't exist, suggest `/productivity:start` first.

### 2. Sync from Jira

If Atlassian MCP is available, fetch issues assigned to user (open/in-progress).

Compare against TASKS.md:

| Jira issue | TASKS.md match? | Action |
|------------|-----------------|--------|
| Found, not in TASKS.md | No match | Offer to add |
| Found, already in TASKS.md | Match by key or title | Skip |
| In TASKS.md with Jira key, not in Jira results | Possibly completed | Offer to mark done |
| Completed in Jira | In Active section | Offer to mark done |

Present diff and let user decide.

### 3. Resolve Jira Links in Vault

Search for Obsidian-style Jira links (`[[XTYPE-NNNN]]`, `[[IDEA-NNNN]]`, or any `[[PREFIX-NNNN]]` matching Jira patterns) across the vault that don't have corresponding files.

For each unresolved link:
1. Query Jira for the ticket summary and assignee
2. Determine placement:
   - **User is assignee OR user says they're working on it** → `PARA/1 Projects/{TICKET-KEY}.md`
   - **Otherwise** → `PARA/3 Resources/Jira/{TICKET-KEY}.md`
3. Create the file with Jira metadata (see Jira Note Template below)

### 4. Triage Stale Items

Review Active tasks in TASKS.md and flag:
- Tasks with due dates in the past
- Tasks in Active for 30+ days
- Tasks with no context

Present each for triage: Mark done? Reschedule? Move to Someday?

### 5. Decode Tasks for Memory Gaps

For each task, attempt to decode all entities using CLAUDE.md → PARA lookups:
```
Task: "Send PSR to Todd re: Phoenix blockers"

Decode:
- PSR → ✓ Pipeline Status Report (in CLAUDE.md)
- Todd → ✓ Todd Martinez (in People/)
- Phoenix → ? Not in memory
```

### 6. Fill Gaps

Present unknown terms grouped:
```
I found terms I don't have context for:

1. "Phoenix" (from: "Send PSR to Todd re: Phoenix blockers")
   → What's Phoenix? Should I create a project entry?

2. "Maya" (from: "sync with Maya on API design")
   → Who is Maya? Should I add to People?
```

Add answers to PARA locations and update CLAUDE.md if frequently used.

### 7. Report

```
Update complete:
- Tasks: +3 from Jira, 1 completed, 2 triaged
- Jira links: 2 new notes created
- Memory: 2 gaps filled
- All tasks decoded ✓
```

## Comprehensive Mode (`--comprehensive`)

Everything in Default Mode, plus:

### Extra: Scan Recent Vault Activity

Scan recent daily notes in `Work Diary/` for:
- Referenced people not in CLAUDE.md
- Referenced projects not tracked
- Obsidian links to non-existent files

### Extra: Suggest New Memories

```
## New People (mentioned in diary but not in People/)
| Name | Frequency | Context |
|------|-----------|---------|
| Maya Rodriguez | 5 mentions | API design discussions |

## New Terms
| Term | Frequency | Context |
|------|-----------|---------|
| XAPI | 3 mentions | seems related to platform work |

## Suggested Cleanup
- Project "Horizon" — no mentions in 30 days. Archive?
```

## Notes

- Never auto-add tasks or memories without user confirmation
- Jira links are detected by pattern: `[[UPPERCASE-DIGITS]]` (e.g., `[[XTYPE-7292]]`, `[[IDEA-123]]`)
- Safe to run frequently
```

---

## File: `productivity/commands/daily-report.md`

```markdown
---
description: Generate a daily standup report from Work Diary entries
argument-hint: "[since YYYY-MM-DD]"
---

# Daily Report Command

> Generate a summary of recent work from Work Diary entries.

Reads daily notes, resolves Obsidian links, cleans up tags, and produces a ready-to-send daily report in chat. Designed to make daily reporting painless — even when you've missed a few days.

## Usage

```
/productivity:daily-report
/productivity:daily-report since 2026-02-06
```

## Work Week

**Sunday through Thursday** (Israel work week).
- "Last work day" on Sunday = previous Thursday
- "Last work day" on Monday = Sunday
- "Next work day" after Thursday = Sunday

## Instructions

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
2. Read the file
3. If file doesn't exist, note: "No daily note found for YYYY-MM-DD"

### 3. Extract Sections

From each daily note, extract content from these sections (per the template):

| Section | Purpose |
|---------|---------|
| `# Goals` | What was planned (checkbox items) |
| `# Worked on today` | What was actually done |
| `# Achievements` | Notable completions |
| `# To do on the next working day` | Carry-forward items |
| `# Free notes` | Additional context |

### 4. Resolve Obsidian Links

For every `[[...]]` link in extracted content:

**Step 4a: Check if it's a Jira ticket pattern** (`[[XTYPE-NNNN]]`, `[[IDEA-NNNN]]`, or similar `[[UPPERCASE-DIGITS]]`):

1. Check if a file exists at `PARA/1 Projects/{KEY}.md` or `PARA/3 Resources/Jira/{KEY}.md`
2. If file exists → use its title/frontmatter summary
3. If file doesn't exist → query Jira via Atlassian MCP for the ticket summary
4. If Jira returns a result:
   - Use the summary in the report
   - Create a note file (see "Jira Note Creation" below)
5. If Jira is unavailable or no result → use the key as-is: `(XTYPE-7292)`

**Step 4b: Check if it's a person link** (`[[Michael Zingerman|Michael]]` or `[[Michael Zingerman]]`):
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

### Summary
[2-3 sentence high-level summary of work across all days]

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

### Observations
- [patterns noticed: recurring work areas, blocked items, etc.]
- [suggestions based on patterns]
```

### 7. Jira Note Creation (Side Effect)

During link resolution (step 4a), if a Jira ticket was queried and no vault note exists, create one:

**Determine placement:**
- Query Jira for assignee
- If user is the assignee → `PARA/1 Projects/{KEY}.md`
- Otherwise → `PARA/3 Resources/Jira/{KEY}.md` (create `Jira/` subfolder if needed)

**Ask user** before creating: "I found references to these Jira tickets without vault notes. Create them?"

List tickets with proposed locations. User confirms or overrides placement.

### 8. Output

Present the composed report directly in chat. The user can copy/paste it wherever needed.

## Jira Note Template

When creating a new Jira ticket note (must match existing vault conventions — see XTYPE-6860 as reference):

```markdown
---
aliases:
  - {KEY}
tags:
  - jira
  - {project-prefix-lowercase}
creation_date: {YYYY-MM-DD}
Status: {status from Jira}
URL: https://xtypeio.atlassian.net/browse/{KEY}
---
## Overview

{Jira summary/title}. {One line of context from description if available.}

**Jira:** [{KEY}](https://xtypeio.atlassian.net/browse/{KEY})
**Status:** {status}
**Assignee:** [[{assignee full name}]]
**Project:** {Jira project name}

## Related

- [[{any related people or projects if known}]]
```

Note: `aliases` includes the Jira key so `[[XTYPE-7292]]` wikilinks resolve. `URL` field (not `jira-url`) matches existing vault convention. Assignee is a wikilink. No rigid body structure — keeps it light.

## Examples

### Single day

```
User: /productivity:daily-report

[Reads Work Diary/2026/02/2026-02-09.md]
```

Input (from daily note):
```
# Worked on today
- Prepared for a sync with [[Michael Zingerman|Michael]] tomorrow. #planned
- Worked on a production issue - [[XTYPE-7292]]. #unplanned
```

Output:
```
## Daily Report — 2026-02-09 (Sunday)

### What I worked on
- Prepared for a sync with Michael tomorrow.
- Worked on a production issue — Audit operation ended without results (XTYPE-7292).

### Plan for next work day (Monday, 2026-02-10)
- [items from "To do on the next working day" section]
```

### Multiple days

```
User: /productivity:daily-report since 2026-02-05

[Reads daily notes for Feb 5 (Wed), 6 (Thu), 9 (Sun) — skips Fri/Sat]
```

## Notes

- If a daily note file doesn't exist for a date in range, mention it but continue with available days
- Work days: Sunday, Monday, Tuesday, Wednesday, Thursday
- Non-work days: Friday, Saturday
- Never modify the original daily note files — this is read-only
- Jira note creation is the only write operation, and requires confirmation
- If Atlassian MCP is not connected, Jira links resolve to their key text only (e.g., "XTYPE-7292")
- The report is output in chat, not written to a file
```

---

## File: `productivity/skills/memory-management/SKILL.md`

```markdown
---
name: memory-management
description: Two-tier memory system mapped to PARA-organized Obsidian vault. CLAUDE.md for working memory hot cache, PARA folders for deep storage. Decodes shorthand, acronyms, nicknames, and internal language.
---

# Memory Management (PARA Edition)

Memory makes Claude a workplace collaborator who speaks your internal language.

## The Goal

Transform shorthand into understanding:

```
User: "ask michael about the audit issue on xtype-7292"
              ↓ Claude decodes
"Ask Michael Zingerman (CTO) about the audit operation failure
 on XTYPE-7292 (Audit operation ended without results, P1 bug)"
```

## Architecture

```
CLAUDE.md                              ← Hot cache (~30 people, common terms, active projects)
PARA/
  1 Projects/                          ← Active project notes + Jira tickets (assigned to you)
  2 Areas/
    People/                            ← Individual people profiles
    Partners/                          ← Partner company profiles
    Customers/                         ← Customer company profiles
  3 Resources/
    Terms/                             ← Glossary, acronyms, internal language
    Jira/                              ← Jira tickets you're not assigned to
    Company/                           ← Company context, teams, tools, processes
  4 Archive/                           ← Completed projects (by year)
```

**CLAUDE.md (Hot Cache):**
- Top ~30 people you interact with most
- ~30 most common acronyms/terms
- Active projects (5-15)
- Your preferences
- **Goal: Cover 90% of daily decoding needs**
- **All entries point to PARA locations for full detail**

**PARA folders (Deep Storage):**
- Rich detail when needed for execution
- Full profiles, histories, context
- Can grow indefinitely — it's an Obsidian vault

## PARA Location Mapping

| What | PARA Location | Filename convention |
|------|---------------|---------------------|
| Person | `PARA/2 Areas/People/{Full Name}.md` | Title case, spaces |
| Partner company | `PARA/2 Areas/Partners/{Company Name}.md` | Title case |
| Customer company | `PARA/2 Areas/Customers/{Company Name}.md` | Title case |
| Active project | `PARA/1 Projects/{Project Name}.md` | Title case |
| Jira ticket (yours) | `PARA/1 Projects/{XTYPE-1234}.md` | Jira key |
| Jira ticket (other) | `PARA/3 Resources/Jira/{XTYPE-1234}.md` | Jira key |
| Glossary / terms | `PARA/3 Resources/Terms/Glossary.md` | Single file |
| Company context | `PARA/3 Resources/Company/` | By topic |
| Archived project | `PARA/4 Archive/{YYYY}/{Project Name}.md` | By year |

**IMPORTANT:** Obsidian vaults use **Title Case with spaces** for filenames, not lowercase-hyphens. Follow existing vault conventions. Check existing files before creating new ones.

## Lookup Flow

```
User: "ask michael about the audit issue on xtype-7292"

1. Check CLAUDE.md (hot cache)
   → michael? ✓ Michael Zingerman, CTO
   → xtype-7292? Not in hot cache

2. Check PARA (deep storage)
   → Search PARA/1 Projects/ and PARA/3 Resources/Jira/ for XTYPE-7292
   → Found: PARA/1 Projects/XTYPE-7292.md ✓

3. If still not found → check Jira via MCP
   → Query Atlassian MCP for ticket details
   → Create vault note, add to CLAUDE.md if relevant

4. If nothing → ask user
   → "What is XTYPE-7292? I'll remember it."
```

## Working Memory Format (CLAUDE.md)

Target ~50-80 lines. Use tables for compactness.

```markdown
# Memory

## Me
[Name], [Role] on [Team]. [One sentence.]

## People
| Who | Role |
|-----|------|
| **Michael** | Michael Zingerman, CTO |
| **Sarah** | Sarah Chen, Engineering (Platform) |
→ Full profiles: PARA/2 Areas/People/

## Terms
| Term | Meaning |
|------|---------|
| PSR | Pipeline Status Report |
| P0 | Drop everything priority |
→ Full glossary: PARA/3 Resources/Terms/Glossary.md

## Projects
| Name | What |
|------|------|
| **XTYPE-7292** | Audit operation bug, P1 |
| **Phoenix** | DB migration, Q2 launch |
→ Details: PARA/1 Projects/

## Preferences
- Work week: Sun-Thu (Israel)
- Async-first communication
- PARA method for organization
- Obsidian for knowledge management
```

## How to Interact

### Decoding User Input (Tiered Lookup)

**Always** decode shorthand before acting:

```
1. CLAUDE.md (hot cache)                → Check first, covers 90%
2. PARA/3 Resources/Terms/Glossary.md   → Full glossary
3. PARA/2 Areas/People/                 → Full profiles
4. PARA/1 Projects/                     → Project details
5. Ask user                             → Unknown term? Learn it.
```

### Adding Memory

When user says "remember this" or provides new context:

**IMPORTANT: All created files must include Obsidian frontmatter. See "Obsidian Markdown Requirements" section for full rules.**

1. **People:**
   - Create `PARA/2 Areas/People/{Full Name}.md` with frontmatter: `aliases` (nickname), `tags` (`contact`, `{company}`), `creation_date`, `company` (as wikilink), `role`
   - If it's a company → `PARA/2 Areas/Partners/` or `Customers/` with appropriate tags
   - Add to CLAUDE.md People table if frequently referenced
   - **Always capture nicknames in both `aliases` frontmatter and CLAUDE.md**
   - Check if file exists first — update, don't overwrite

2. **Terms/acronyms:**
   - Add to `PARA/3 Resources/Terms/Glossary.md` (create with frontmatter if doesn't exist)
   - If frequently used, also add to CLAUDE.md Terms table

3. **Projects:**
   - Create `PARA/1 Projects/{Project Name}.md` with frontmatter: `aliases`, `tags` (`project`), `creation_date`, `Status`
   - Add to CLAUDE.md Projects table if active

4. **Jira tickets:**
   - Create at `PARA/1 Projects/{KEY}.md` (if assigned to user) or `PARA/3 Resources/Jira/{KEY}.md` (otherwise)
   - Use the Jira Note Template (see daily-report skill)
   - `aliases` must include the Jira key for wikilink resolution

5. **Preferences:** Add to CLAUDE.md Preferences section

### Promotion / Demotion

**Promote to CLAUDE.md when:**
- You use a term/person frequently
- It's part of active work

**Demote from CLAUDE.md when:**
- Project completed → move note to `PARA/4 Archive/{year}/`
- Person no longer frequent → remove from CLAUDE.md (keep in People/)
- Term rarely used → remove from CLAUDE.md (keep in Glossary)

## Conventions

- **Bold** key terms in CLAUDE.md for scannability
- Keep CLAUDE.md under ~100 lines
- Follow existing vault filename conventions (Title Case with spaces)
- Always capture nicknames and alternate names
- Check if a file already exists before creating a new one
- Never modify existing vault files without user confirmation (except CLAUDE.md and TASKS.md)
```

---

## File: `productivity/skills/task-management/SKILL.md`

```markdown
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
- Jira references as plain text with URL: `**XTYPE-7292** — [Audit operation bug](https://xtypeio.atlassian.net/browse/XTYPE-7292), assigned to me`
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
- The `/productivity:daily-report` skill reads from Work Diary, not TASKS.md
- When processing daily reports, offer to update TASKS.md if completed items match

## Conventions

- **Bold** task titles
- Include "for [person]" for commitments to others
- Include "due [date]" for deadlines
- Include Jira key when applicable
- Keep Done section for ~1 week, then clear
```

---

## File: `productivity/skills/daily-report/SKILL.md`

```markdown
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

1. Search vault: `PARA/1 Projects/{KEY}.md` then `PARA/3 Resources/Jira/{KEY}.md`
2. If found → use the note's `# title` or frontmatter
3. If not found → query Atlassian MCP: get summary, status, assignee
4. If Jira returns data:
   - Format: `{Jira summary} ({KEY})`
   - Offer to create vault note (user confirms placement)
5. If Jira unavailable → just use the key: `(XTYPE-7292)`

### Jira Note Placement Decision

| Condition | Location |
|-----------|----------|
| User is Jira assignee | `PARA/1 Projects/{KEY}.md` |
| User says they work on it | `PARA/1 Projects/{KEY}.md` |
| Otherwise | `PARA/3 Resources/Jira/{KEY}.md` |

### Jira Note Template

```markdown
---
aliases:
  - {KEY}
tags:
  - jira
  - {project-prefix-lowercase}
creation_date: {YYYY-MM-DD}
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
```

---

## File: `productivity/skills/dashboard.html`

Copy the original `dashboard.html` from the uploaded files **as-is**. No modifications needed for MVP.

> **Dashboard improvement suggestion for later:** The current dashboard reads `TASKS.md` and a `CLAUDE.md` with `memory/` paths. A future iteration could update the memory panel to show PARA-mapped content and add a "Recent Daily Reports" section. This is not blocking and can be done incrementally.

---

## File: `productivity/CLAUDE.md` (Template)

This is the template — actual content gets populated during `/start`. Ship this as a starting point:

```markdown
# Memory

## Me
[To be filled during /start]

## People
| Who | Role |
|-----|------|
→ Full profiles: PARA/2 Areas/People/

## Terms
| Term | Meaning |
|------|---------|
→ Full glossary: PARA/3 Resources/Terms/Glossary.md

## Projects
| Name | What |
|------|------|
→ Details: PARA/1 Projects/

## Preferences
- Work week: Sun-Thu
- PARA method for organization
- Obsidian for knowledge management
- Jira instance: xtypeio.atlassian.net
- Jira prefixes: XTYPE (dev tasks), IDEA (product ideas)
```

---

## File: `boris-plugins/README.md` (Repository Root)

```markdown
# Boris Plugins

Personal plugin marketplace for [Cowork](https://claude.com/product/cowork) and Claude Code.

## Plugins

| Plugin | Description |
|--------|-------------|
| [productivity](./productivity/) | PARA-integrated task management, workplace memory, daily reports, and Jira integration |

## Installation

    claude plugins add boris-plugins/productivity

## Requirements

- Obsidian vault with PARA structure as working directory
- Work Diary folder with daily notes
- Optional: Atlassian (Jira) MCP connection for ticket resolution
```

---

## File: `boris-plugins/LICENSE`

Use the same MIT license from the original plugin.

---

## Implementation Notes for Claude Code

### Order of operations

1. Create the repository directory structure
2. Copy `dashboard.html` from uploads as-is into `productivity/skills/`
3. Copy `LICENSE` from uploads to repo root
4. Create all `.md` and `.json` files with the content specified above
5. Initialize git repo with initial commit

### Key differences from original plugin

| Original | This fork |
|----------|-----------|
| `memory/` directory | PARA folders (already exist in vault) |
| `memory/people/` | `PARA/2 Areas/People/` |
| `memory/projects/` | `PARA/1 Projects/` |
| `memory/glossary.md` | `PARA/3 Resources/Terms/Glossary.md` |
| `memory/context/` | `PARA/3 Resources/Company/` |
| Generic MCP connections | Atlassian (Jira) only |
| No daily report | `/daily-report` command + skill |
| Filenames: lowercase-hyphens | Filenames: Title Case with spaces (Obsidian convention) |
| Standalone workspace | Integrated with existing Obsidian vault |

### Files NOT included (intentional)

- No separate `memory/` directory — PARA replaces it
- No email/calendar MCP connections — can add later
- No Slack MCP — can add later

### Testing checklist

After implementation, the following should work:

- [ ] `claude plugins add` recognizes the plugin
- [ ] `/productivity:start` detects PARA structure and creates CLAUDE.md + TASKS.md
- [ ] `/productivity:update` triages tasks and offers Jira sync
- [ ] `/productivity:daily-report` reads today's daily note and produces clean output
- [ ] `/productivity:daily-report since 2026-02-05` reads multiple days, skips Fri/Sat
- [ ] Jira links resolve to ticket summaries when Atlassian MCP is connected
- [ ] Unresolved Jira links offer to create vault notes with correct placement
- [ ] CLAUDE.md references PARA paths, not memory/ paths
- [ ] Dashboard opens and shows tasks
```
