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
