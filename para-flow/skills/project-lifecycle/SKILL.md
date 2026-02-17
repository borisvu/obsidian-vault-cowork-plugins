---
name: project-lifecycle
description: Project staleness detection, classification, archive placement rules, and CLAUDE.md project table sync. Used by the archive command and referenced by update command.
---

# Project Lifecycle Skill

Detect stale projects, classify them for archiving, execute moves to PARA/4 Archive, and keep the CLAUDE.md Projects table in sync. Organized into numbered sections that commands reference directly.

## Section 1: Core Concepts

### Definitions

| Term | Meaning |
|------|---------|
| **Project** | A folder in PARA/1 Projects/ representing active work (may contain sub-folders and notes) |
| **Umbrella project** | A project folder that contains sub-project folders (e.g., Data Management contains Data copy, Related Tables) |
| **Sub-project** | A folder nested inside an umbrella project |
| **Staleness** | A project with no recent diary mentions, no recent Jira activity, and/or a terminal Jira status |
| **Archive** | Moving a project folder to `PARA/4 Archive/Archive {YYYY}/` — the project is preserved but no longer active |
| **Resources** | Moving a project folder to `PARA/3 Resources/` — the content is reference material, not project work |

### Archive Location

```
PARA/4 Archive/
├── Archive 2025/
│   ├── PROJECT-4585/                         ← standalone project
│   └── PROJECT-6599 - Bug empty fields/      ← standalone project
└── Archive 2026/
    ├── Data Management/                      ← parent umbrella preserved
    │   ├── Calculating Simple... PROJECT-5984/
    │   └── Creating Database Schemas.../
    └── PROJECT-6837 - Bug.../                ← standalone project
```

### Work Week

Sunday through Thursday (Israel). Friday and Saturday are non-work days. Use `~~time` MCP for the current date — never guess.

## Section 2: Inventory Rules

### Scanning PARA/1 Projects/

List all top-level folders in `PARA/1 Projects/`. For each folder, build a registry entry:

| Field | How to determine |
|-------|-----------------|
| `folder_name` | Directory name as-is |
| `jira_key` | Extract from folder name if it matches `UPPERCASE-DIGITS` pattern (e.g., PROJECT-7371 from "PROJECT-7371 - Sync fails") |
| `is_sub_project` | True if this folder is nested inside another project folder |
| `parent_project` | Parent folder name, if `is_sub_project` is true |
| `has_sub_projects` | True if this folder contains sub-folders that are themselves projects |
| `frontmatter_status` | Read the main `.md` file inside the folder, extract `Status` from YAML frontmatter |
| `creation_date` | From frontmatter `creation_date` field |
| `last_updated` | From frontmatter `last_updated` field |

### Finding the main .md file

For a project folder, the main file is typically named the same as the folder (e.g., `PROJECT-7371 - Sync fails/PROJECT-7371 - Sync fails.md`). If not found, use the first `.md` file in the folder that has YAML frontmatter.

### Umbrella detection

A folder is an umbrella if it contains sub-folders where at least one sub-folder has its own `.md` file with frontmatter. Files directly in the umbrella folder (meeting notes, README, etc.) do not make it an umbrella — only sub-folders with project-like structure do.

**Rule:** Umbrella projects are never classified as a unit. Only their sub-projects are classified individually.

## Section 3: Diary Enrichment

### Purpose

For each project in the registry, determine when it was last mentioned in the Work Diary.

### Search strategy

1. For each project, build search terms:
   - The folder name (e.g., "Data Management", "PROJECT-7371 - Sync fails")
   - The Jira key if present (e.g., "PROJECT-7371")
   - Any aliases from the main .md file frontmatter
2. Grep all files under `Work Diary/` for these terms
3. Extract the date from each matching filename (`YYYY-MM-DD.md`)
4. Record the most recent match as `last_diary_mention`
5. If no match found in any diary note, set `last_diary_mention` to null — this is a strong staleness signal

### Performance note

Search the most recent 6 months of diary files first (`Work Diary/{current year}/` and `Work Diary/{previous year}/`). Only search older files if no recent mentions are found and you need to confirm total absence.

## Section 4: Jira Enrichment

### Purpose

For projects with a Jira key, fetch current ticket state from Atlassian MCP.

### Fields to retrieve

| Jira Field | Usage |
|-----------|-------|
| `status` | Primary signal: Done/Closed/Merged → archive candidate |
| `resolution` | Confirms completion: Fixed, Done, Won't Do, Duplicate |
| `updated` | Last Jira activity date → staleness input |
| `assignee` | Context for understanding project ownership |

### MCP unavailable fallback

If the Atlassian MCP is not connected:
- Skip this stage entirely
- Log: "Jira enrichment skipped — Atlassian MCP not connected. Classification based on diary mentions and frontmatter only."
- Classification proceeds with Section 5 using available data only
- More projects may land in REVIEW instead of ARCHIVE since terminal Jira statuses cannot be detected

## Section 5: Classification Rules

### Three classifications

Each project is assigned exactly one classification. Evaluate in order: ARCHIVE first, then REVIEW, then default to ACTIVE.

### ARCHIVE — recommend move

A project is classified ARCHIVE if **any** of these are true:
- Jira status is Done, Closed, Merged, or Resolved
- Jira resolution is Fixed, Done, Won't Do, or Duplicate
- Frontmatter `Status` field is one of: Merged, Done, Closed, Resolved, Cancelled
- No diary mention in 60+ days **AND** no Jira activity in 60+ days
- CLAUDE.md Projects table "What" column contains "merged", "done", "closed", or "archived" (case-insensitive)

### REVIEW — flag for user

A project is classified REVIEW if **any** of these are true (and none of the ARCHIVE criteria matched):
- No diary mention in 30–59 days
- Jira status is Backlog with no Jira activity in 30+ days
- Frontmatter `Status` is Backlog with no diary references
- Project folder contains only reference-style content (no tasks, no recent diary mentions) — suggest `PARA/3 Resources/` instead of archive

### ACTIVE — no action

Default when neither ARCHIVE nor REVIEW criteria matched. Positive signals (not required):
- Diary mention within last 30 days
- Jira status is In Progress, In Review, Open, or To Do
- Listed in current week's goals (weekly note)

### `--since N` override

The `--since N` flag replaces the 30-day base threshold:
- REVIEW window: N days to (2N - 1) days
- ARCHIVE window: 2N+ days
- Example: `--since 90` → REVIEW at 90–179 days, ARCHIVE at 180+ days

**Exception:** Jira terminal statuses (Done, Closed, Merged, Resolved) and terminal resolutions (Fixed, Done, Won't Do, Duplicate) always trigger ARCHIVE regardless of `--since` value. Terminal means terminal.

### Umbrella rule

Umbrella projects (has_sub_projects = true) are never classified on their own. Only sub-projects are classified individually. If **all** sub-projects are classified as ARCHIVE, the umbrella itself becomes ARCHIVE (move the whole folder). If any sub-project is ACTIVE or REVIEW, the umbrella stays.

### No-data projects

Projects with no Jira key, no diary mentions, and no frontmatter status → classify as REVIEW with note: "No activity data available — check manually."

## Section 6: Archive Placement

### Target path rules

| Scenario | Target Path |
|----------|-------------|
| Standalone project (top-level folder in PARA/1 Projects/) | `PARA/4 Archive/Archive {YYYY}/{folder_name}/` |
| Sub-project of parent that stays active | `PARA/4 Archive/Archive {YYYY}/{parent_name}/{sub_folder_name}/` |
| All sub-projects done → entire umbrella | `PARA/4 Archive/Archive {YYYY}/{parent_name}/` (move whole folder) |
| User picks Resources for a REVIEW item | `PARA/3 Resources/{folder_name}/` |

### Year determination

Always use the current year (when archiving happens), not the project creation year. Use `~~time` MCP for the current date.

### Folder creation

Create `PARA/4 Archive/Archive {YYYY}/` if it does not exist. Create parent sub-folders (e.g., `Archive 2026/Data Management/`) if needed for sub-project archiving. These are the **only** directories the command is allowed to create. Never create PARA/1, PARA/2, or PARA/3 directories.

### Conflict handling

If the target path already exists in the archive (e.g., a project was partially archived before):
- Ask user: "**{folder_name}** already exists in Archive {YYYY}/. Merge contents into existing folder, or rename to {folder_name} (2)?"
- Wait for user response before proceeding

### Umbrella with non-project files

If an umbrella project folder contains files that are not sub-project folders (e.g., meeting notes, README.md), warn the user: "Data Management/ contains files outside sub-projects: meetings.md, README.md. Include these in the archive, or leave them behind?"

## Section 7: Presentation Format

### Grouping

Present projects grouped by classification. Show ARCHIVE first (most actionable), then REVIEW, then ACTIVE (informational).

### ARCHIVE table

```
## Ready to Archive

| #  | Project                          | Reason                          | Target                                    |
|----|----------------------------------|---------------------------------|-------------------------------------------|
| 1  | PROJECT-7371 - Sync fails        | Jira: Merged                    | Archive 2026/PROJECT-7371 - Sync fails/   |
| 2  | PROJECT-7457 - Stuck definitions | Jira: Done, no diary 45d        | Archive 2026/PROJECT-7457 - Stuck defs/   |
| 3  | Data Management/Data copy        | Jira: Done, last diary Jan 5    | Archive 2026/Data Management/Data copy/   |
```

### REVIEW table

```
## Needs Review

| #  | Project                          | Reason                          | Suggestion                                |
|----|----------------------------------|---------------------------------|-------------------------------------------|
| 4  | PROJECT-6860 - No query results  | No diary mention 65d            | Archive or re-activate?                   |
| 5  | Snow Ingestion Pipeline          | Reference docs only, no tasks   | Move to Resources?                        |
```

### ACTIVE table (compact)

```
## Active (no action)

| Project              | Last Activity                    |
|----------------------|----------------------------------|
| Data Management      | Today (umbrella, 2 active subs)  |
| PROJECT-7406         | Feb 10 (Jira: In Progress)       |
| PROJECT-7451         | Today (in weekly goals)          |
```

### Confirmation prompt

After the tables, ask the user to confirm:

```
Archive items 1-3? Review items 4-5?
Enter item numbers to proceed (e.g., "1,2,3,4"), "all", or "none":
```

For REVIEW items the user confirms, ask per-item: archive, move to Resources, or skip.

## Section 8: Post-Move Updates

### CLAUDE.md Projects table

After all confirmed moves complete:
1. Read the current CLAUDE.md Projects table
2. Remove rows for archived projects
3. Do not add archive notes to the table — keep it clean for active projects only
4. Write the updated table (CLAUDE.md is an exception to the "no modification without confirmation" rule for this operation, since the user already confirmed the archive)

### TASKS.md

1. Read TASKS.md
2. Find tasks referencing archived project names or Jira keys
3. Present each to the user: "These tasks reference archived projects:"
   - Mark as done?
   - Remove from list?
   - Keep as-is?
4. Never auto-remove tasks — always ask

### Move log

Print a summary after all operations:

```
Archived 3 projects:
  PROJECT-7371 - Sync fails       → Archive 2026/PROJECT-7371 - Sync fails/
  PROJECT-7457 - Stuck definitions → Archive 2026/PROJECT-7457 - Stuck definitions/
  Data Management/Data copy        → Archive 2026/Data Management/Data copy/

CLAUDE.md: removed 2 entries
TASKS.md: 1 task flagged for review
```

## Section 9: Project Table Sync

### Purpose

Reconcile the CLAUDE.md Projects table against the current state of PARA/1 Projects/. Runs after archive moves are complete, but can also be referenced independently.

### Step 1: Scan current state

List all folders in `PARA/1 Projects/` (post-archiving). For each, extract: folder name, Jira key (if present), whether it is a sub-project or umbrella.

### Step 2: Compare against CLAUDE.md

| Situation | Action |
|-----------|--------|
| Folder exists in PARA/1 but not in Projects table | Add row: **Name** from folder name (bold), **What** from frontmatter description or Jira summary |
| Row exists in table but folder is gone from PARA/1 | Remove row (it was archived or manually deleted) |
| Both exist | Update What column with current Jira status if available (e.g., append "In Progress" or "In Review") |

### Step 3: Present diff

```
## CLAUDE.md Projects Table Update

Added:
  + PROJECT-7502 — New reporting endpoint

Removed:
  - PROJECT-7371 — Sync fails (archived)
  - PROJECT-7457 — Stuck definitions (archived)

Updated:
  ~ PROJECT-7406 — added "In Review"

Apply these changes?
```

Wait for user confirmation before writing to CLAUDE.md.

### Constraints

- Table stays at ~5-15 entries (hot cache principle)
- If PARA/1 has more than 15 active projects, include only those with recent diary mentions or active Jira status. Note overflow: "> +N more in PARA/1 Projects/"
- Umbrella projects get one row for the parent, not individual rows per sub-project
- If a project's description cannot be determined (no frontmatter, no Jira), use "TODO: describe" as the What column value

## Section 10: Important Rules

### Safety

1. **No file moves without explicit user confirmation** — always show the plan first and wait for approval
2. **Never delete files** — archive means move, not delete
3. **Never modify Work Diary files** — they are read-only
4. **Never create PARA/1, 2, or 3 directories** — only create Archive year folders and sub-folders within them
5. **Never auto-remove tasks** — always present TASKS.md changes for user approval
6. **Log all moves** — print a summary after execution

### Behavioral

7. **Always use `~~time` MCP** for the current date — never guess or calculate
8. **Always read a file before claiming it doesn't exist** — only report missing after a failed read
9. **Check for conflicts** before moving — if target exists in archive, ask user
10. **Respect umbrella structure** — never archive an umbrella with active sub-projects