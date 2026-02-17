# Triage Fixes Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Apply 5 fixes to the inbox-triage skill and CLAUDE.md based on issues found during the first live triage run.

**Architecture:** All changes are edits to existing markdown files — no new files. Two structural rewrites (Sections 4 and 6 of SKILL.md) and three line edits across SKILL.md and CLAUDE.md.

**Tech Stack:** Markdown skill definitions for Claude Code plugin system. No executable code.

**Design doc:** `docs/plans/2026-02-17-triage-fixes-design.md`

---

### Task 1: Restructure Section 4 — Index-First Conflict Detection

**Files:**
- Modify: `para-flow/skills/inbox-triage/SKILL.md:70-87`

**Step 1: Replace Section 4 content**

Replace the entire Section 4 (lines 70-87) with the following. Keep the `## Section 4: Conflict Detection` heading.

```markdown
## Section 4: Conflict Detection

### Purpose

For each MOVE candidate, check for existing vault notes. Uses a two-phase index-first approach — build a complete file index once, then match all candidates against it.

### Phase 1: Build File Index

Before checking individual files, build a searchable index of all existing vault notes:

| Step | Mechanism | Purpose |
|------|-----------|---------|
| 1a | Read CLAUDE.md People, Terms, and Projects tables | Instant conflict detection for known entities — no filesystem I/O |
| 1b | Run `find PARA/1\ Projects PARA/2\ Areas PARA/3\ Resources -name "*.md" -type f` | Build complete file index of PARA/1-3 in a single filesystem traversal |

Store the combined results as the **vault file index** for Phase 2.

**Fallback:** If `find` fails or returns empty for a directory that should have content, flag it: "Could not index {directory}. Conflict detection incomplete for this location." Continue with partial results.

### Phase 2: Match Against Index

For each MOVE candidate, check the vault file index using deterministic signals:

| Strategy | Match against | Match means |
|----------|---------------|-------------|
| Exact filename | Index paths containing `{filename}` (excluding Inbox and Archive) | Strong conflict |
| Jira key | Index paths matching `{KEY}*` pattern | Strong conflict |
| Person name | Index paths under `Areas/People/` matching `{name}*` | Strong conflict |
| Alias match | Read frontmatter of candidate files identified in index | Moderate conflict |

Each layer narrows the set. Alias matching (requires reading files) only runs on items not already caught by filename, Jira key, or person name matching.

If a conflict is found, annotate the MOVE item with the conflicting file's path and a note for the user to decide.

No content similarity. No fuzzy matching. Deterministic signals only.
```

**Step 2: Verify the edit**

Read `para-flow/skills/inbox-triage/SKILL.md` lines 70-110 and confirm Section 4 has the new two-phase structure with the `find` command and the Phase 1/Phase 2 layout.

**Step 3: Commit**

```bash
git add para-flow/skills/inbox-triage/SKILL.md
git commit -m "Restructure conflict detection to index-first approach

Replace per-file glob with two-phase strategy: build complete file
index via CLAUDE.md tables + find command, then match all candidates
against it. Fixes silent glob failures in sandboxed environments."
```

---

### Task 2: Add Merge Resolution to Section 6

**Files:**
- Modify: `para-flow/skills/inbox-triage/SKILL.md` — Section 5 confirmation prompt and Section 6 CONFLICT execution

**Step 1: Update Section 5 confirmation prompt**

In the confirmation prompt code block (around line 136 after Task 1 edits — find the exact line by searching for `keep inbox copy, keep existing, or skip?`), replace:

```
- Item 6: conflicts with existing note — keep inbox copy, keep existing, or skip?
```

with:

```
- Item 6: conflicts with existing note — keep inbox, keep existing, merge, or skip?
```

Also update the line after the confirmation prompt code block. Find:

```
For conflict items, ask per-item: keep inbox (overwrites existing), keep existing (delete inbox copy), or skip (leave in inbox).
```

Replace with:

```
For conflict items, ask per-item: keep inbox (overwrites existing), keep existing (delete inbox copy), merge (append unique inbox content to existing), or skip (leave in inbox).
```

**Step 2: Add merge option to Section 6 CONFLICT execution**

Find the CONFLICT execution subsection. Current content:

```markdown
### CONFLICT execution

- **Keep inbox:** Enhance inbox copy, overwrite existing note, delete inbox copy.
- **Keep existing:** Delete inbox copy.
- **Skip:** Leave in inbox, no action.
```

Replace with:

```markdown
### CONFLICT execution

- **Keep inbox:** Enhance inbox copy, overwrite existing note, delete inbox copy.
- **Keep existing:** Delete inbox copy.
- **Merge:** Combine unique content from both files into the existing note, then delete inbox copy. See merge rules below.
- **Skip:** Leave in inbox, no action.

### Merge rules

1. Read both files fully.
2. Identify sections/content blocks in the inbox copy not present in the existing note. Use headings, bullet groups, or paragraph breaks as section boundaries.
3. Append unique content under a `## Merged from Inbox` heading in the existing note. If the structure is obvious (e.g., adding a phone number to a contact section), integrate inline instead.
4. Preserve the existing note's frontmatter as authority. Only add new aliases or tags from the inbox copy that the existing note lacks.
5. Delete the inbox copy after successful merge.
6. Log: `Merged: {filename} — {N} sections added to {existing path}`
```

**Step 3: Verify the edit**

Read the CONFLICT execution and Merge rules sections. Confirm four resolution options and six merge steps.

**Step 4: Commit**

```bash
git add para-flow/skills/inbox-triage/SKILL.md
git commit -m "Add merge conflict resolution option to inbox-triage skill

Fourth resolution alongside keep-inbox, keep-existing, skip. Merge
reads both files, appends unique inbox content to existing note under
a Merged from Inbox heading, preserves existing frontmatter as
authority, then deletes the inbox copy."
```

---

### Task 3: Add Sandbox Fallback to Section 6

**Files:**
- Modify: `para-flow/skills/inbox-triage/SKILL.md` — Section 6 error handling

**Step 1: Expand the error handling subsection**

Find the error handling subsection in Section 6. Current content:

```markdown
### Error handling

- If target directory doesn't exist within PARA/1, 2, or 3 — create it.
- Never create new top-level PARA folders.
- If a move fails, skip the item and report the error — don't abort the batch.
```

Replace with:

```markdown
### Error handling

- If target directory doesn't exist within PARA/1, 2, or 3 — create it.
- Never create new top-level PARA folders.
- If a move or delete fails, do not silently skip. Follow the sandbox fallback below.

### Sandbox fallback

If a file delete or move fails (e.g., "Operation not permitted" in sandboxed environments):

1. Check if Desktop Commander MCP is available.
2. If yes, check CLAUDE.md Preferences for a stored `vault_host_path`.
   - If found, retry the operation using the host filesystem path via Desktop Commander.
   - If not found, ask the user for the vault's host filesystem path. Store it in CLAUDE.md Preferences for future use, then retry.
3. If Desktop Commander is not available, ask the user for an alternative method to complete the operation.
4. Never silently skip a failed operation — always report the failure and the fallback attempted.
5. If all fallbacks fail for an item, skip it, report the error, and continue the batch.
```

**Step 2: Verify the edit**

Read the error handling and sandbox fallback sections. Confirm the 5-step fallback sequence with Desktop Commander priority.

**Step 3: Commit**

```bash
git add para-flow/skills/inbox-triage/SKILL.md
git commit -m "Add sandbox fallback for file operations in inbox-triage

When delete/move fails in sandboxed environments, try Desktop
Commander MCP with stored host path, ask user for path if not
known, or request alternative method. Stores vault_host_path in
CLAUDE.md Preferences for future runs."
```

---

### Task 4: Line Edits — Tag Convention and Frontmatter

**Files:**
- Modify: `para-flow/skills/inbox-triage/SKILL.md` — Section 5 example, Section 6 step 2
- Modify: `CLAUDE.md:116`

**Step 1: Fix Section 5 MOVE table example tag**

In the MOVE table code block example, find:

```
| 5 | Damien Davis.md | Person | Areas/People/ | +tags: contact |
```

Replace with:

```
| 5 | Damien Davis.md | Person | Areas/People/ | +tags: person |
```

**Step 2: Fix Section 6 step 2 tag**

In Section 6 Enhancement step 2, find:

```
2. **Tags:** Add based on content type (`contact` for people, `jira` + project prefix for tickets, `reference`/`glossary` for terms, `project` for project material).
```

Replace with:

```
2. **Tags:** Add based on content type (`person` for people, `jira` + project prefix for tickets, `reference`/`glossary` for terms, `project` for project material).
```

**Step 3: Remove `last_updated` from Section 6 step 1**

In Section 6 Enhancement step 1, find:

```
1. **Frontmatter:** Ensure valid YAML with at minimum `aliases`, `tags`, `creation_date`, `last_updated`.
```

Replace with:

```
1. **Frontmatter:** Ensure valid YAML with at minimum `aliases`, `tags`, `creation_date`.
```

**Step 4: Fix CLAUDE.md Tag Conventions table**

In `CLAUDE.md` line 116, find:

```
| Person      | `contact`, `{company-lowercase}`     |
```

Replace with:

```
| Person      | `person`, `{company-lowercase}`      |
```

**Step 5: Verify all edits**

Read SKILL.md Section 5 example, Section 6 steps 1-2, and CLAUDE.md line 116. Confirm `person` tag and no `last_updated`.

**Step 6: Commit**

```bash
git add para-flow/skills/inbox-triage/SKILL.md CLAUDE.md
git commit -m "Align tag convention to person, remove last_updated from required frontmatter

Canonical person tag is now 'person' across CLAUDE.md Tag Conventions,
skill execution rules, and presentation examples. Removed last_updated
from required frontmatter fields — no existing vault notes use it."
```

---

### Task 5: Update Section 5 Confirmation Prompt for Merge (Design Doc)

**Files:**
- Modify: `docs/plans/2026-02-17-triage-fixes-design.md` — no changes needed, already documents merge

This task is a verification-only step. Read the design doc and confirm it accurately reflects what was implemented in Tasks 1-4. If any discrepancy, update the design doc.

**Step 1: Read and verify design doc**

Read `docs/plans/2026-02-17-triage-fixes-design.md`. Confirm all 5 fixes match what was implemented.

**Step 2: Final commit (if design doc needed updating)**

```bash
git add docs/plans/2026-02-17-triage-fixes-design.md
git commit -m "Update design doc to match implemented fixes"
```

If no changes were needed, skip this commit.
