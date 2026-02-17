---
description: Triage inbox — classify, detect conflicts, enhance, and place files
---

# Triage Command

> Scan PARA/0 Inbox/, classify each file by content and quality, detect conflicts with existing vault notes, and execute confirmed actions.

Always shows a plan and asks for confirmation. No files are deleted or moved without explicit user approval.

## Usage

```
/para-flow:triage                  → scan, classify, present plan, ask to proceed
```

## Instructions

### Step 0: Get Time

Use `~~time` MCP to get the current date, time, and day of week (inbox-triage skill Section 1).

### Step 1: Inventory

Use inbox-triage skill Section 2 (Inventory Rules).

List all files in `PARA/0 Inbox/`. For each file:
- Read full content (not just frontmatter)
- Record: filename, has_frontmatter, frontmatter_fields, content_length, content_preview

If the inbox is empty, print "Inbox is empty. Nothing to triage." and exit.

List non-markdown files separately if any are found.
Treat nested folders as single items.

### Step 2: Classify

Use inbox-triage skill Section 3 (Classification Rules).

Classify each file as DELETE or MOVE:
- DELETE: empty, single URL, stub, fewer than 3 meaningful lines
- MOVE: everything with value — sub-classify by content type (Person, Jira, Term, Project material, Reference, Meeting note)

Apply the safety rule: files with 10+ lines of meaningful content are never DELETE.

### Step 3: Conflict Check

Use inbox-triage skill Section 4 (Conflict Detection).

For each MOVE candidate, search for existing vault notes using deterministic signals:
- Exact filename match in PARA (excluding Inbox and Archive)
- Jira key match in Projects and Resources/Jira
- Person name match in Areas/People
- Alias match in vault frontmatter

Annotate conflicts on the MOVE items.

### Step 4: Present Plan

Use inbox-triage skill Section 5 (Presentation Format).

Display three grouped tables:
1. DELETE table (empty/temporary files with reasons)
2. MOVE table (files with type, target, and planned enhancement)
3. CONFLICT table (MOVE items that match existing vault notes)

Ask user for confirmation. For conflict items, ask per-item: keep inbox copy, keep existing, or skip.

### Step 5: Execute

Use inbox-triage skill Section 6 (Execution Rules).

For each confirmed action:
1. DELETE: remove file from inbox, log action
2. MOVE: enhance frontmatter/tags/aliases per vault conventions, move to target path, log action
3. CONFLICT — keep inbox: enhance and overwrite existing note, delete inbox copy
4. CONFLICT — keep existing: delete inbox copy
5. CONFLICT — skip: leave in inbox

If a move fails, skip and report — don't abort the batch.

### Step 6: Post-Triage

Use inbox-triage skill Section 7 (Post-Triage Updates).

1. If new people, terms, or projects were placed, offer to update CLAUDE.md tables (present diff, confirm before writing)
2. Print summary: deleted count, moved count, conflicts resolved, CLAUDE.md changes, remaining inbox count

## Notes

- This command follows inbox-triage skill Section 8 (Important Rules)
- No files are deleted or moved without explicit user confirmation
- Enhancement (frontmatter, tags, aliases) is applied automatically to all moved files
- Work Diary files are never modified
- Non-markdown files are listed but not classified or moved
- If PARA/0 Inbox/ is empty, print "Inbox is empty. Nothing to triage." and exit
