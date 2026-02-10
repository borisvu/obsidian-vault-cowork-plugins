---
name: memory-management
description: Two-tier memory system mapped to PARA-organized Obsidian vault. CLAUDE.md for working memory hot cache, PARA folders for deep storage. Decodes shorthand, acronyms, nicknames, and internal language.
---

# Memory Management (PARA Edition)

Memory makes Claude a workplace collaborator who speaks your internal language.

## The Goal

Transform shorthand into understanding:

```
User: "ask john about the audit issue on project-7292"
              ↓ Claude decodes
"Ask John Smith (CTO) about the audit operation failure
 on PROJECT-7292 (Audit operation ended without results, P1 bug)"
```

## Architecture

```
CLAUDE.md                              ← Hot cache (~30 people, common terms, active projects)
PARA/
  0 Inbox/                             ← Unsorted items (scanned during --comprehensive update)
  1 Projects/                          ← Active project folders + Jira tickets (assigned to you)
    {Project Name}/                    ← Subdirectory per project with notes inside
    {KEY} - {Summary}/                 ← Jira ticket subdirectory (when assigned to you)
  2 Areas/
    People/                            ← Individual people profiles
    Partners/                          ← Partner company profiles
    Customers/                         ← Customer company profiles
  3 Resources/
    Terms/                             ← Individual term/acronym files (one per term)
    Jira/                              ← Jira tickets you're not assigned to (flat files)
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

| What                | PARA Location                                            | Filename convention                |
| ------------------- | -------------------------------------------------------- | ---------------------------------- |
| Person              | `PARA/2 Areas/People/{Full Name}.md`                     | Title case, spaces                 |
| Partner company     | `PARA/2 Areas/Partners/{Company Name}.md`                | Title case                         |
| Customer company    | `PARA/2 Areas/Customers/{Company Name}.md`               | Title case                         |
| Active project      | `PARA/1 Projects/{Project Name}/`                        | Subdirectory, Title case           |
| Jira ticket (yours) | `PARA/1 Projects/{KEY} - {Summary}/{KEY} - {Summary}.md` | Subdirectory with descriptive name |
| Jira ticket (other) | `PARA/3 Resources/Jira/{KEY} - {Summary}.md`             | Flat file with descriptive name    |
| Term / acronym      | `PARA/3 Resources/Terms/{Term}.md`                       | One file per term                  |
| Company context     | `PARA/3 Resources/Company/`                              | By topic                           |
| Archived project    | `PARA/4 Archive/{YYYY}/{Project Name}/`                  | By year, subdirectory              |

**IMPORTANT:** Obsidian vaults use **Title Case with spaces** for filenames, not lowercase-hyphens. Follow existing vault conventions. Check existing files before creating new ones.

**NOTE:** Projects may be subdirectories containing multiple notes (meetings, sub-tasks, etc.). When scanning Projects, search recursively — don't just look at the top level.

## Lookup Flow

```
User: "ask john about the audit issue on project-7292"

1. Check CLAUDE.md (hot cache)
   → john? ✓ John Smith, CTO
   → project-7292? Not in hot cache

2. Check PARA (deep storage)
   → Search PARA/1 Projects/*/ and PARA/3 Resources/Jira/ for PROJECT-7292
   → Use glob: PARA/1 Projects/*/PROJECT-7292* or PARA/1 Projects/PROJECT-7292*
   → Found: PARA/1 Projects/PROJECT-7292 - Audit operation ended without results/ ✓

3. If still not found → check Jira via MCP
   → Query Atlassian MCP for ticket details
   → Create vault note, add to CLAUDE.md if relevant

4. If nothing → ask user
   → "What is PROJECT-7292? I'll remember it."
```

## Working Memory Format (CLAUDE.md)

Target ~50-80 lines. Use tables for compactness.

```markdown
# Memory

## Me

[Name], [Role] on [Team]. [One sentence.]

## People

| Who       | Role                               |
| --------- | ---------------------------------- |
| **John**  | John Smith, CTO                    |
| **Sarah** | Sarah Chen, Engineering (Platform) |

> Full profiles: PARA/2 Areas/People/

## Terms

| Term | Meaning                  |
| ---- | ------------------------ |
| PSR  | Pipeline Status Report   |
| P0   | Drop everything priority |

> Full glossary: PARA/3 Resources/Terms/

## Projects

| Name             | What                    |
| ---------------- | ----------------------- |
| **PROJECT-7292** | Audit operation bug, P1 |
| **Phoenix**      | DB migration, Q2 launch |

> Details: PARA/1 Projects/

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
2. PARA/3 Resources/Terms/             → Individual term files
3. PARA/2 Areas/People/                 → Full profiles
4. PARA/1 Projects/                     → Project details (search recursively)
5. Ask user                             → Unknown term? Learn it.
```

### Adding Memory

When user says "remember this" or provides new context:

**IMPORTANT: All created files must include Obsidian frontmatter. See "Obsidian Markdown Requirements" below.**

1. **People:**
   - Create `PARA/2 Areas/People/{Full Name}.md` with frontmatter: `aliases` (nickname), `tags` (`contact`, `{company}`), `creation_date`, `last_updated`, `company` (as wikilink), `role`
   - If it's a company → `PARA/2 Areas/Partners/` or `Customers/` with appropriate tags
   - Add to CLAUDE.md People table if frequently referenced
   - **Always capture nicknames in both `aliases` frontmatter and CLAUDE.md**
   - Check if file exists first — update, don't overwrite

2. **Terms/acronyms:**
   - Create `PARA/3 Resources/Terms/{Term}.md` with frontmatter: `aliases`, `tags` (`reference`, `glossary`), `creation_date`, `last_updated`
   - If frequently used, also add to CLAUDE.md Terms table

3. **Projects:**
   - Create `PARA/1 Projects/{Project Name}/` subdirectory with `{Project Name}.md` inside
   - Frontmatter: `aliases`, `tags` (`project`), `creation_date`, `last_updated`, `Status`
   - Add to CLAUDE.md Projects table if active

4. **Jira tickets:**
   - If assigned to user: create `PARA/1 Projects/{KEY} - {Summary}/{KEY} - {Summary}.md`
   - Otherwise: create `PARA/3 Resources/Jira/{KEY} - {Summary}.md`
   - Use the Jira Note Template (see daily-report skill)
   - `aliases` must include the Jira key for wikilink resolution

5. **Preferences:** Add to CLAUDE.md Preferences section

### Obsidian Markdown Requirements

Every created `.md` file (except CLAUDE.md and TASKS.md) MUST include YAML frontmatter:

```yaml
---
aliases:
  - { short name or alternate reference }
tags:
  - { relevant tags as YAML array }
creation_date: { YYYY-MM-DD }
last_updated: { YYYY-MM-DD }
---
```

**Tag conventions:**

| File type   | Required tags                              | Optional tags                    |
| ----------- | ------------------------------------------ | -------------------------------- |
| Person      | `contact`, `{company-lowercase}`           | `{department}`, `{role-keyword}` |
| Customer    | `customer`, `<company_name>`, `servicenow` | —                                |
| Partner     | `partner`, `<company_name>`                | `{specialty}`                    |
| Jira ticket | `jira`, `{project-prefix-lowercase}`       | `bug`, `feature`                 |
| Project     | `project`                                  | `{status}`, `{team}`             |
| Term        | `reference`, `glossary`                    | —                                |

**Internal links:** Always use `[[wikilinks]]` for vault cross-references. Use `[[Full Name|Nickname]]` for people in body text.

**Filenames:** Title Case with spaces (e.g., `John Smith.md`). Check if file exists before creating.

### Promotion / Demotion

**Promote to CLAUDE.md when:**

- You use a term/person frequently
- It's part of active work

**Demote from CLAUDE.md when:**

- Project completed → move note to `PARA/4 Archive/{year}/`
- Person no longer frequent → remove from CLAUDE.md (keep in People/)
- Term rarely used → remove from CLAUDE.md (keep in Terms/)

## Conventions

- **Bold** key terms in CLAUDE.md for scannability
- Keep CLAUDE.md under ~100 lines
- Follow existing vault filename conventions (Title Case with spaces)
- Always capture nicknames and alternate names
- Check if a file already exists before creating a new one
- Never modify existing vault files without user confirmation (except CLAUDE.md and TASKS.md)
