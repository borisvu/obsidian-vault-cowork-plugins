# Triage Command Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add a `/para-flow:triage` command that reads every file in `PARA/0 Inbox/`, classifies by content quality and type, detects conflicts with existing vault notes, and executes confirmed deletes/moves with automatic frontmatter enhancement.

**Architecture:** New command (`triage.md`) orchestrates 7 steps, each referencing a section of a new skill (`inbox-triage/SKILL.md`). Follows the same pattern as `/archive` + `project-lifecycle`. The update command's `--comprehensive` inbox scan is simplified to an advisory pointing to `/triage`.

**Tech Stack:** Markdown skill/command definitions for Claude Code plugin system. No executable code.

**Design doc:** `docs/plans/2026-02-17-triage-command-design.md`

---

### Task 1: Create inbox-triage skill (Sections 1-4)

**Files:**
- Create: `para-flow/skills/inbox-triage/SKILL.md`

**Reference patterns:**
- `para-flow/skills/project-lifecycle/SKILL.md` — same structure (numbered sections, YAML frontmatter)
- `para-flow/skills/daily-report/SKILL.md` — same 10-section pattern

**Step 1: Create the skill file with Sections 1-4**

Create `para-flow/skills/inbox-triage/SKILL.md` with YAML frontmatter and Sections 1-4 (Core Concepts, Inventory Rules, Classification Rules, Conflict Detection). Content is specified in the design doc sections "Section 1" through "Section 4".

The YAML frontmatter must be:

```yaml
---
name: inbox-triage
description: Content-aware inbox classification, conflict detection, frontmatter enhancement, and PARA placement rules. Used by the triage command.
---
```

**Step 2: Commit**

```bash
git add para-flow/skills/inbox-triage/SKILL.md
git commit -m "Add inbox-triage skill sections 1-4 (core concepts, inventory, classification, conflicts)"
```

---

### Task 2: Append inbox-triage skill Sections 5-8

**Files:**
- Modify: `para-flow/skills/inbox-triage/SKILL.md` (append after Section 4)

**Step 1: Append Sections 5-8**

Append Sections 5-8 (Presentation Format, Execution Rules, Post-Triage Updates, Important Rules) to the existing skill file. Content is specified in the design doc sections "Section 5" through "Section 8".

**Step 2: Commit**

```bash
git add para-flow/skills/inbox-triage/SKILL.md
git commit -m "Add inbox-triage skill sections 5-8 (presentation, execution, post-triage, rules)"
```

---

### Task 3: Create triage command

**Files:**
- Create: `para-flow/commands/triage.md`

**Reference patterns:**
- `para-flow/commands/archive.md` — same structure (YAML frontmatter, Usage, Instructions with numbered Steps, Notes)

**Step 1: Create the command file**

Create `para-flow/commands/triage.md` with the following complete content:

```markdown
---
description: Triage inbox — classify, detect conflicts, enhance, and place files
---

# Triage Command

> Scan PARA/0 Inbox/, classify each file by content and quality, detect conflicts with existing vault notes, and execute confirmed actions.

Always shows a plan and asks for confirmation. No files are deleted or moved without explicit user approval.

## Usage

\`\`\`
/para-flow:triage                  → scan, classify, present plan, ask to proceed
\`\`\`

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
```

**Step 2: Commit**

```bash
git add para-flow/commands/triage.md
git commit -m "Add triage command as thin orchestrator over inbox-triage skill"
```

---

### Task 4: Simplify update command inbox scan

**Files:**
- Modify: `para-flow/commands/update.md:133-147`

**Step 1: Replace the "Extra: Scan Inbox" section**

Replace lines 133-147 (the current "Extra: Scan Inbox" section with its detailed suggestions table) with a lightweight advisory:

```markdown
### Extra: Inbox Advisory

Count files in `PARA/0 Inbox/`. If any files are found, append to the update report:

```
N file(s) in Inbox. Run /para-flow:triage for full analysis.
```

No classification or move suggestions — advisory only.
```

**Step 2: Commit**

```bash
git add para-flow/commands/update.md
git commit -m "Simplify update --comprehensive inbox scan to advisory for /triage"
```

---

### Task 5: Update metadata and documentation

**Files:**
- Modify: `para-flow/.claude-plugin/plugin.json` (version bump)
- Modify: `para-flow/README.md` (commands table + changelog)
- Modify: `CLAUDE.md` (repository structure + commands section + design doc reference)

**Step 1: Bump version in plugin.json**

Change `"version": "1.3.0"` to `"version": "1.4.0"`.

**Step 2: Add triage to README.md commands table**

After the `/archive --since N` row (line 37), add:

```markdown
| `/triage`                         | Triage inbox: classify, detect conflicts, enhance, and place files     |
```

**Step 3: Add 1.4.0 changelog entry to README.md**

Before the `### 1.3.0` line (line 45), add:

```markdown
### 1.4.0

- Added `/triage` command for content-aware inbox triage
- New inbox-triage skill with 8 referenceable sections
- Reads every file in PARA/0 Inbox/, classifies by content quality and type
- Two-tier classification: DELETE (empty/temporary) and MOVE (has value)
- Deterministic conflict detection: filename, Jira key, person name, alias match
- Automatic frontmatter enhancement on all moved files (tags, aliases, creation_date)
- Safe interactive workflow: always shows plan, requires confirmation
- Post-triage CLAUDE.md sync for new people, terms, and projects
- Simplified `/update --comprehensive` inbox scan to advisory pointing to `/triage`

```

**Step 4: Update CLAUDE.md repository structure**

Add `triage.md` to the commands tree (after the `archive.md` line, line 31):

```
│   │   └── triage.md                    # /para-flow:triage — triage inbox files
```

Add `inbox-triage/SKILL.md` to the skills tree (after the `project-lifecycle/SKILL.md` line, line 36):

```
│       ├── inbox-triage/SKILL.md        # Inbox classification, conflicts, enhancement (8 sections)
```

**Step 5: Update CLAUDE.md "Four Commands" section**

Change "Four Commands" heading to "Five Commands" (line 76) and add after the `/archive` description (line 81):

```markdown
5. **`/para-flow:triage`** — Reads every file in `PARA/0 Inbox/`, classifies as DELETE (empty/temporary) or MOVE (has value), detects conflicts with existing vault notes by filename/Jira key/person name/alias, enhances frontmatter before placement, and executes confirmed actions. Post-triage syncs CLAUDE.md tables.
```

**Step 6: Add design doc reference to CLAUDE.md**

After the archive command design doc line (line 10), add:

```markdown
The triage command design is in `docs/plans/2026-02-17-triage-command-design.md`.
```

**Step 7: Update CLAUDE.md "Unsorted items" table row**

Update the Unsorted items row (line 65) from:

```
| Unsorted items            | `PARA/0 Inbox/` (scanned during --comprehensive)         |
```

to:

```
| Unsorted items            | `PARA/0 Inbox/` (triaged by `/triage` command)            |
```

**Step 8: Commit**

```bash
git add para-flow/.claude-plugin/plugin.json para-flow/README.md CLAUDE.md
git commit -m "Bump to v1.4.0, update docs for triage command"
```

---

### Task 6: Final verification

**Step 1: Verify all new files exist**

```bash
ls -la para-flow/commands/triage.md
ls -la para-flow/skills/inbox-triage/SKILL.md
```

**Step 2: Verify skill has all 8 sections**

Read `para-flow/skills/inbox-triage/SKILL.md` and confirm sections 1-8 are present:
1. Core Concepts
2. Inventory Rules
3. Classification Rules
4. Conflict Detection
5. Presentation Format
6. Execution Rules
7. Post-Triage Updates
8. Important Rules

**Step 3: Verify command references all 7 steps**

Read `para-flow/commands/triage.md` and confirm steps 0-6 are present, each referencing the correct skill section.

**Step 4: Verify update command change**

Read `para-flow/commands/update.md` and confirm the "Extra: Scan Inbox" section was replaced with the advisory.

**Step 5: Verify metadata**

- `plugin.json` version is `1.4.0`
- `README.md` has triage in commands table and 1.4.0 changelog
- `CLAUDE.md` has triage in repo structure, Five Commands section, and design doc reference

**Step 6: Verify git log**

```bash
git log --oneline -5
```

Expected: 4 new commits (skill sections 1-4, sections 5-8, command, update change, metadata).
