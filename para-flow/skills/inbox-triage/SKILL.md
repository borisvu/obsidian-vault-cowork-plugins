---
name: inbox-triage
description: Content-aware inbox classification, conflict detection, frontmatter enhancement, and PARA placement rules. Used by the triage command.
---

# Inbox Triage Skill

Classify unsorted inbox files, detect conflicts with existing vault notes, enhance frontmatter, and place files in the correct PARA location. Organized into numbered sections that commands reference directly.

## Section 1: Core Concepts

### Definitions

| Term | Meaning |
|------|---------|
| **Inbox** | `PARA/0 Inbox/` — catch-all landing zone for unsorted notes |
| **Triage** | Reading each inbox file, classifying it, and placing it in the correct PARA location |
| **Enhancement** | Adding/fixing frontmatter, tags, and aliases to conform to vault conventions before placement |
| **Conflict** | An existing vault note that matches an inbox file by filename, Jira key, person name, or alias |

### Work Week

Sunday through Thursday (Israel). Use `~~time` MCP for the current date — never guess.

### Content Types

| Type | Detection signals | Default destination |
|------|-------------------|---------------------|
| Person | Tags contain `person`, filename matches known person, biographical content | `PARA/2 Areas/People/` |
| Jira ticket | Filename matches `UPPERCASE-DIGITS`, Jira URL in frontmatter, `jira` tag | `PARA/1 Projects/{KEY}/` or `PARA/3 Resources/Jira/` |
| Term/concept | Short definition (<20 lines), domain concept, no project context | `PARA/3 Resources/Terms/` |
| Project material | References active projects, implementation details, action items | `PARA/1 Projects/{relevant project}/` |
| Reference material | Essays, guides, drafts — informational, not project-tied | `PARA/3 Resources/` |
| Meeting note | Date in filename, attendee lists, discussion points | `PARA/1 Projects/{relevant}/Meetings/` or `PARA/3 Resources/` |

Ambiguous content type -> default to `PARA/3 Resources/` with note "Content type unclear."

## Section 2: Inventory Rules

### Scanning PARA/0 Inbox/

For each file in `PARA/0 Inbox/`:

1. Read full content (not just frontmatter).
2. Record: `filename`, `has_frontmatter`, `frontmatter_fields`, `content_length` (lines excluding frontmatter), `content_preview` (first 5 non-empty body lines).
3. Non-markdown files (images, PDFs, etc.) — list separately: "Non-markdown files found: {list}. These require manual sorting." Do not classify or move them.
4. Nested folders — treat as a single item. Read the main `.md` file inside for classification. Move the entire folder as a unit.

## Section 3: Classification Rules

### Two Classifications

Two classifications: DELETE and MOVE. Conflicts are detected separately in Section 4 and annotate MOVE items.

### DELETE — empty, temporary, or accidental

A file is DELETE if any of:

- Body is empty (only frontmatter, or frontmatter + blank lines)
- Body is a single URL/link with no explanation
- No tags, no aliases, and body has fewer than 3 meaningful lines
- Auto-generated stub with all fields blank

**Safety rule:** If the file has more than 10 lines of meaningful content, it is never DELETE. Classify as MOVE with note: "Low-quality but has content — review manually."

### MOVE — has value, needs placement

Everything not DELETE. Sub-classify by content type using the table in Section 1. For Jira tickets, determine placement per daily-report skill Section 7 (assignee -> Projects, otherwise -> Resources/Jira).

## Section 4: Conflict Detection

### Purpose

For each MOVE candidate, check for existing vault notes using deterministic signals only.

### Conflict Strategies

| Strategy | Search | Match means |
|----------|--------|-------------|
| Exact filename | `PARA/**/{filename}` excluding Inbox and Archive | Strong conflict |
| Jira key | `PARA/**/{KEY}*` in Projects and Resources/Jira | Strong conflict |
| Person name | `PARA/2 Areas/People/{name}*` | Strong conflict |
| Alias match | Search vault for files with matching frontmatter aliases | Moderate conflict |

If a conflict is found, annotate the MOVE item with the conflicting file's path and a note for the user to decide.

No content similarity. No fuzzy matching.

## Section 5: Presentation Format

### Grouping

Three grouped tables, most actionable first: DELETE, then MOVE, then CONFLICTS (a subset of MOVE items with vault matches).

### DELETE table

```
## Delete (empty/temporary)

| # | File | Reason |
|---|------|--------|
| 1 | SN Agent.md | Empty — only frontmatter, no content |
| 2 | Stuck copy flow investigation.md | Single URL, no context |
```

### MOVE table

```
## Move to vault

| # | File | Type | Target | Enhancement |
|---|------|------|--------|-------------|
| 3 | Update Set.md | Term | Resources/Terms/ | +tags: servicenow, term |
| 4 | XTYPE-7292 - Audit operation... | Jira | Resources/Jira/ | Already well-structured |
| 5 | Damien Davis.md | Person | Areas/People/ | +tags: contact |
```

### CONFLICT table

```
## Conflicts with existing notes

| # | Inbox file | Existing note | Signal |
|---|------------|---------------|--------|
| 6 | Damien Davis.md | People/Damien Davis.md | Exact filename match |
```

Conflict items also appear in the MOVE table. The conflict table highlights them for user decision.

### Confirmation prompt

```
Proceed with this plan?
- Items 1-2: delete
- Items 3-5: move + enhance
- Item 6: conflicts with existing note — keep inbox copy, keep existing, or skip?

Enter item numbers to modify, "all" to execute, or "none" to cancel:
```

For conflict items, ask per-item: keep inbox (overwrites existing), keep existing (delete inbox copy), or skip (leave in inbox).

## Section 6: Execution Rules

### Enhancement (applied to all MOVE items before moving)

1. **Frontmatter:** Ensure valid YAML with at minimum `aliases`, `tags`, `creation_date`, `last_updated`.
2. **Tags:** Add based on content type (`contact` for people, `jira` + project prefix for tickets, `reference`/`glossary` for terms, `project` for project material).
3. **Aliases:** Jira key for tickets, first name for people, abbreviations for terms.
4. **creation_date:** If missing, use file modification date or extract from filename.
5. **Filename convention:** Person notes → `Firstname Lastname.md`. Jira tickets → `{KEY} - {Summary}.md`.

### DELETE execution

- Remove file from inbox.
- Log: `Deleted: {filename} ({reason})`

### MOVE execution

- Enhance the file in-place.
- Move to target path.
- Log: `Moved: {filename} → {target path}`

### CONFLICT execution

- **Keep inbox:** Enhance inbox copy, overwrite existing note, delete inbox copy.
- **Keep existing:** Delete inbox copy.
- **Skip:** Leave in inbox, no action.

### Error handling

- If target directory doesn't exist within PARA/1, 2, or 3 — create it.
- Never create new top-level PARA folders.
- If a move fails, skip the item and report the error — don't abort the batch.

## Section 7: Post-Triage Updates

### CLAUDE.md sync

If any new people, terms, or projects were placed:

- New person → offer to add to CLAUDE.md People table.
- New term → offer to add to CLAUDE.md Terms table.
- New project material → check if Projects table needs updating.

Present diff and confirm before writing. Uses existing memory-management skill.

### Summary output

```
Inbox triage complete:
  Deleted: N files (empty/temporary)
  Moved: N files (breakdown by destination)
  Conflicts: N resolved (breakdown by resolution)
  CLAUDE.md: changes applied / no changes needed
  Inbox: N files remaining
```

## Section 8: Important Rules

### Safety

1. **No file deletes or moves without explicit user confirmation** — always show the plan first and wait for approval.
2. **Never modify Work Diary files** — they are read-only.
3. **Never create top-level PARA directories** — only create sub-directories within existing PARA/1, 2, or 3.
4. **For conflict items, show both files before user decides** — display content of inbox file and existing note side by side.
5. **Log all actions in summary output** — every delete, move, and conflict resolution must be recorded.

### Behavioral

6. **Always use `~~time` MCP** for the current date — never guess or calculate.
7. **Always read a file before claiming it doesn't exist** — only report missing after a failed read.
8. **Non-markdown files are listed but never classified or moved** — report them for manual sorting.
9. **Nested inbox folders are treated as single items** — classify by the main .md file inside, move the entire folder as a unit.
