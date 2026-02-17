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
