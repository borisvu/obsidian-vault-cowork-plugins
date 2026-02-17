# Archive Command Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add `/para-flow:archive` command that scans PARA/1 Projects for staleness, recommends archiving, and moves confirmed projects to PARA/4 Archive.

**Architecture:** New project-lifecycle skill (10 numbered sections) provides all classification and archive logic. New archive command is a thin orchestrator referencing skill sections. Update command gets a lightweight advisory line. No changes to daily-report or memory-management skills.

**Tech Stack:** Markdown files (Claude Code plugin system), Atlassian MCP (Jira), ~~time MCP

**Design doc:** `docs/plans/2026-02-17-archive-command-design.md`

---

### Task 1: Create project-lifecycle skill — Sections 1–5

These sections cover data gathering: definitions, inventory, diary enrichment, Jira enrichment, and classification.

**Files:**
- Create: `para-flow/skills/project-lifecycle/SKILL.md`

**Step 1: Create the skill file with frontmatter and sections 1–5**

Write the file with the following content:

```markdown
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
```

**Step 2: Verify the file structure**

Run: `grep -c "^## Section" para-flow/skills/project-lifecycle/SKILL.md`
Expected: `5` (Sections 1 through 5)

Run: `head -3 para-flow/skills/project-lifecycle/SKILL.md`
Expected: YAML frontmatter with `name: project-lifecycle`

**Step 3: Commit**

```bash
git add para-flow/skills/project-lifecycle/SKILL.md
git commit -m "Add project-lifecycle skill sections 1-5 (inventory, enrichment, classification)"
```

---

### Task 2: Complete project-lifecycle skill — Sections 6–10

These sections cover actions: placement, presentation, post-move updates, table sync, and safety rules.

**Files:**
- Modify: `para-flow/skills/project-lifecycle/SKILL.md` (append after Section 5)

**Step 1: Append sections 6–10 to the skill file**

Append the following content after Section 5:

```markdown
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
```

**Step 2: Verify the complete skill file**

Run: `grep -c "^## Section" para-flow/skills/project-lifecycle/SKILL.md`
Expected: `10` (Sections 1 through 10)

Run: `grep "^## Section" para-flow/skills/project-lifecycle/SKILL.md`
Expected: All 10 section headers in order

**Step 3: Commit**

```bash
git add para-flow/skills/project-lifecycle/SKILL.md
git commit -m "Add project-lifecycle skill sections 6-10 (placement, presentation, sync, rules)"
```

---

### Task 3: Create archive command

**Files:**
- Create: `para-flow/commands/archive.md`

**Step 1: Write the command file**

```markdown
---
description: Scan projects for staleness, recommend and execute archiving
argument-hint: "[--since N]"
---

# Archive Command

> Scan PARA/1 Projects/ for stale projects, recommend archiving, and execute confirmed moves.

Always shows a plan and asks for confirmation. No files are moved without explicit user approval.

## Usage

```
/para-flow:archive                  → scan, classify, present plan, ask to proceed
/para-flow:archive --since 90       → override staleness threshold (default: 30 days)
```

## Instructions

### Step 0: Parse Arguments and Get Time

1. Parse the argument string:
   - `--since N` → override the 30-day staleness threshold with N days (see project-lifecycle skill Section 5)
   - No arguments → use default 30-day threshold
2. Use `~~time` MCP to get the current date, time, and day of week (project-lifecycle skill Section 1).

### Step 1: Inventory

Use project-lifecycle skill Section 2 (Inventory Rules).

Scan `PARA/1 Projects/` and build a registry of all projects, including:
- Folder names, Jira keys, parent/child relationships
- Frontmatter status and dates from main .md files
- Umbrella detection (has sub-project folders)

### Step 2: Enrich from Diary

Use project-lifecycle skill Section 3 (Diary Enrichment).

For each project in the registry, search `Work Diary/` for mentions of the project name and Jira key. Record the most recent mention date.

### Step 3: Enrich from Jira

Use project-lifecycle skill Section 4 (Jira Enrichment).

For projects with Jira keys, query Atlassian MCP for current status, resolution, last update, and assignee. If MCP is unavailable, skip and log a warning.

### Step 4: Classify

Use project-lifecycle skill Section 5 (Classification Rules).

Classify each project as ARCHIVE, REVIEW, or ACTIVE based on:
- Jira terminal statuses (Done/Closed/Merged → always ARCHIVE)
- Diary mention recency
- Frontmatter status
- CLAUDE.md annotations
- `--since N` threshold override if provided

Apply the umbrella rule: never classify umbrellas directly, only sub-projects.

### Step 5: Present Plan

Use project-lifecycle skill Section 7 (Presentation Format).

Display the classification tables grouped by ARCHIVE, REVIEW, and ACTIVE. Ask the user to confirm which items to proceed with. For REVIEW items, ask per-item: archive, move to Resources, or skip.

### Step 6: Execute Moves

Use project-lifecycle skill Section 6 (Archive Placement) for target paths.

For each confirmed item:
1. Determine target path per placement rules
2. Create archive year folder and parent sub-folders if needed
3. Move the project folder to the target path
4. Handle conflicts if target already exists (ask user)

### Step 7: Post-Move Updates

Use project-lifecycle skill Section 8 (Post-Move Updates).

1. Update CLAUDE.md Projects table — remove archived entries
2. Check TASKS.md for references to archived projects — present for user triage
3. Print the move log summary

### Step 8: Project Table Sync

Use project-lifecycle skill Section 9 (Project Table Sync).

Reconcile CLAUDE.md Projects table against current PARA/1 Projects/ state:
- Add rows for new projects not in the table
- Remove rows for projects no longer in PARA/1
- Update status annotations from Jira if available
- Present diff and ask for user confirmation before writing

## Notes

- This command follows project-lifecycle skill Section 10 (Important Rules)
- No files are moved without explicit user confirmation
- Archive means move, never delete
- Work Diary files are read-only — never modified
- Only Archive year folders and sub-folders are created — never PARA/1, 2, or 3 directories
- If PARA/1 Projects/ is empty, print "No projects found. Nothing to do." and exit
```

**Step 2: Verify the command file**

Run: `head -4 para-flow/commands/archive.md`
Expected: YAML frontmatter with `description` and `argument-hint`

Run: `grep -c "^### Step" para-flow/commands/archive.md`
Expected: `9` (Steps 0 through 8)

**Step 3: Commit**

```bash
git add para-flow/commands/archive.md
git commit -m "Add archive command as thin orchestrator over project-lifecycle skill"
```

---

### Task 4: Add advisory line to update command

**Files:**
- Modify: `para-flow/commands/update.md` (after the `### 7. Report` section, before `## Comprehensive Mode`)

**Step 1: Add the advisory check**

Insert the following section between `### 7. Report` and `## Comprehensive Mode`:

```markdown
### 8. Project Health Advisory

Lightweight check — no Jira calls or diary scanning.

1. List folders in `PARA/1 Projects/`
2. For each, read the main `.md` file and check the frontmatter `Status` field
3. Count folders where Status matches: Done, Closed, Merged, Resolved, Cancelled (case-insensitive)
4. If count > 0, append to the report output:

```
N project(s) in PARA/1 Projects/ appear completed. Run /para-flow:archive for full analysis.
```

5. If count is 0, do not add any output — the advisory is silent when all projects look active
```

**Step 2: Verify the change**

Run: `grep "Project Health Advisory" para-flow/commands/update.md`
Expected: One match for the new section header

Run: `grep "para-flow:archive" para-flow/commands/update.md`
Expected: One match referencing the archive command

**Step 3: Commit**

```bash
git add para-flow/commands/update.md
git commit -m "Add project health advisory to update command"
```

---

### Task 5: Update metadata and documentation

**Files:**
- Modify: `para-flow/.claude-plugin/plugin.json` (version bump)
- Modify: `para-flow/README.md` (add command, changelog)
- Modify: `CLAUDE.md` (repo root — structure table, command descriptions)

**Step 1: Bump plugin version**

In `para-flow/.claude-plugin/plugin.json`, change:
- `"version": "1.2.0"` → `"version": "1.3.0"`

**Step 2: Update para-flow/README.md**

Add archive command to the Commands table (after the standup rows):

```markdown
| `/archive`                        | Scan projects for staleness, recommend and execute archiving           |
| `/archive --since N`              | Override staleness threshold (default: 30 days)                        |
```

Add changelog entry at the top of the Changelog section (before 1.2.0):

```markdown
### 1.3.0

- Added `/archive` command for project lifecycle management
- New project-lifecycle skill with 10 referenceable sections
- Scans PARA/1 Projects/ for stale projects using diary mentions, Jira status, and frontmatter
- Three-tier classification: ARCHIVE, REVIEW, ACTIVE
- Safe interactive workflow: always shows plan, requires confirmation before moving files
- Moves projects to `PARA/4 Archive/Archive {YYYY}/` following established conventions
- Supports umbrella projects with sub-project archiving
- Reconciles CLAUDE.md Projects table after archiving
- Added project health advisory to `/update` command report
```

**Step 3: Update root CLAUDE.md**

In the Repository Structure section, add the new files:

```
│   ├── commands/
│   │   ├── start.md                     # /para-flow:start — init system, bootstrap memory
│   │   ├── update.md                    # /para-flow:update — sync Jira, triage tasks
│   │   ├── standup.md                   # /para-flow:standup — generate standup/daily reports
│   │   └── archive.md                   # /para-flow:archive — scan projects, recommend archiving
│   └── skills/
│       ├── memory-management/SKILL.md   # Two-tier memory: CLAUDE.md + PARA folders
│       ├── task-management/SKILL.md     # TASKS.md management
│       ├── daily-report/SKILL.md        # Daily report generation from Work Diary (10 sections)
│       ├── project-lifecycle/SKILL.md   # Project staleness, classification, archiving (10 sections)
│       └── dashboard.html               # Visual HTML dashboard (copied from source)
```

In the "Three Commands" section, rename it to "Four Commands" and add:

```markdown
4. **`/para-flow:archive`** — Scans PARA/1 Projects for staleness using diary mentions, Jira status, and frontmatter. Classifies as ARCHIVE/REVIEW/ACTIVE. Moves confirmed projects to `PARA/4 Archive/Archive {YYYY}/` and reconciles CLAUDE.md Projects table. Supports `--since N` for threshold override
```

In the Project Overview paragraph, add a reference to the archive design doc:

```
The archive command design is in `docs/plans/2026-02-17-archive-command-design.md`.
```

In the PARA Integration table, update the Archived projects row to be more descriptive:

```markdown
| Archived projects         | `PARA/4 Archive/Archive {YYYY}/`                        |
```

**Step 4: Verify all changes**

Run: `grep '"1.3.0"' para-flow/.claude-plugin/plugin.json`
Expected: One match

Run: `grep "archive" para-flow/README.md`
Expected: Multiple matches (command table + changelog)

Run: `grep "archive" CLAUDE.md`
Expected: Multiple matches (structure + command description)

Run: `grep "project-lifecycle" CLAUDE.md`
Expected: One match in the structure table

**Step 5: Commit**

```bash
git add para-flow/.claude-plugin/plugin.json para-flow/README.md CLAUDE.md
git commit -m "Bump to v1.3.0, update docs for archive command"
```

---

### Task 6: Final verification

**Files:** All modified/created files

**Step 1: Verify complete file list**

Run: `git log --oneline HEAD~5..HEAD`
Expected: 4 commits from Tasks 1-5

Run: `ls para-flow/commands/`
Expected: `archive.md  start.md  standup.md  update.md`

Run: `ls para-flow/skills/`
Expected: `daily-report  dashboard.html  memory-management  project-lifecycle  task-management`

Run: `ls para-flow/skills/project-lifecycle/`
Expected: `SKILL.md`

**Step 2: Verify skill has all 10 sections**

Run: `grep "^## Section" para-flow/skills/project-lifecycle/SKILL.md`
Expected: 10 lines, Sections 1 through 10

**Step 3: Verify command references skill sections correctly**

Run: `grep "skill Section" para-flow/commands/archive.md`
Expected: References to Sections 1-10 of project-lifecycle skill

**Step 4: Verify no PII**

Run: `grep -ri "XTYPE\|xtype" para-flow/`
Expected: No matches (all sanitized to PROJECT)

Run: `grep -ri "atlassian.net" para-flow/commands/archive.md para-flow/skills/project-lifecycle/SKILL.md`
Expected: No matches (archive command and skill don't hardcode Jira URLs)

**Step 5: Verify version consistency**

Run: `grep "version" para-flow/.claude-plugin/plugin.json`
Expected: `"version": "1.3.0"`

Run: `grep "### 1.3.0" para-flow/README.md`
Expected: One match
