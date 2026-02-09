# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Cowork/Claude Code plugin repository (`boris-plugins`) containing a single plugin called `productivity`. It is a customized fork of Anthropic's `knowledge-work-plugins/productivity` plugin, adapted for an existing Obsidian vault that follows the PARA method. The plugin runs when the working directory is the root of an Obsidian vault.

The original implementation spec is in `implementation-spec.md`. The validated design with deltas is in `docs/plans/2026-02-09-productivity-plugin-design.md`.

## Repository Structure

```
work-notes-plugins/
├── README.md                            # Marketplace README
├── LICENSE                              # Apache 2.0
├── CLAUDE.md                            # This file (Claude Code guidance)
├── implementation-spec.md               # Original spec
├── docs/plans/                          # Design documents
├── productivity/
│   ├── .claude-plugin/plugin.json       # Plugin manifest
│   ├── .mcp.json                        # MCP server config (Atlassian/Jira)
│   ├── README.md
│   ├── CONNECTORS.md                    # Tool category mapping (~~category placeholders)
│   ├── CLAUDE.md                        # Template for working memory hot cache
│   ├── commands/
│   │   ├── start.md                     # /productivity:start — init system, bootstrap memory
│   │   ├── update.md                    # /productivity:update — sync Jira, triage tasks
│   │   └── daily-report.md              # /productivity:daily-report — generate standup reports
│   └── skills/
│       ├── memory-management/SKILL.md   # Two-tier memory: CLAUDE.md + PARA folders
│       ├── task-management/SKILL.md     # TASKS.md management
│       ├── daily-report/SKILL.md        # Daily report generation from Work Diary
│       └── dashboard.html               # Visual HTML dashboard (copied from source)
```

## Architecture

### Plugin System

- `.claude-plugin/plugin.json` defines the plugin manifest (name, version, description)
- `commands/*.md` are user-invocable slash commands with YAML frontmatter (`description`, `argument-hint`)
- `skills/*/SKILL.md` are skill definitions with YAML frontmatter (`name`, `description`) that commands reference
- `.mcp.json` declares MCP server connections (currently only Atlassian for Jira)
- `CONNECTORS.md` maps abstract tool categories (`~~issue tracker`, `~~chat`, etc.) to concrete MCP servers

### PARA Integration

The plugin maps its concepts to an existing Obsidian vault's PARA structure instead of creating its own `memory/` directory. Projects use **subdirectories** (matching vault convention), not flat files:

| Plugin concept | Vault location |
|---------------|----------------|
| People profiles | `PARA/2 Areas/People/` |
| Partner companies | `PARA/2 Areas/Partners/` |
| Customer companies | `PARA/2 Areas/Customers/` |
| Active projects | `PARA/1 Projects/{Name}/` (subdirectory) |
| Assigned Jira tickets | `PARA/1 Projects/{KEY} - {Summary}/{KEY} - {Summary}.md` |
| Non-assigned Jira tickets | `PARA/3 Resources/Jira/{KEY} - {Summary}.md` |
| Terms / glossary | `PARA/3 Resources/Terms/{Term}.md` (individual files) |
| Company context | `PARA/3 Resources/Company/` |
| Archived projects | `PARA/4 Archive/{YYYY}/` |
| Unsorted items | `PARA/0 Inbox/` (scanned during --comprehensive) |
| Daily notes | `Work Diary/YYYY/MM/YYYY-MM-DD.md` |
| Weekly notes | `Work Diary/gggg/MM/gggg-[W]ww.md` |

### Two-Tier Memory System

1. **CLAUDE.md (hot cache)** — compact (~50-80 lines) working memory at vault root with top ~30 people, ~30 terms, active projects, preferences
2. **PARA folders (deep storage)** — full profiles and details in the Obsidian vault, growing indefinitely

Lookup flow: CLAUDE.md -> PARA folders (search recursively) -> Jira API -> ask user.

### Three Commands

1. **`/productivity:start`** — Verifies vault structure, creates CLAUDE.md/TASKS.md/dashboard.html if missing, bootstraps memory by scanning PARA content
2. **`/productivity:update`** — Syncs Jira issues, triages stale tasks, resolves unlinked Jira references, fills memory gaps. `--comprehensive` mode adds deep vault scanning and Inbox triage
3. **`/productivity:daily-report`** — Reads Work Diary entries, resolves Obsidian links and Jira tickets, cleans tags, outputs formatted standup report. Supports `since YYYY-MM-DD` for multi-day reports

## Key Conventions

### Obsidian Markdown Rules (enforced across all skills/commands)

- All created `.md` files (except CLAUDE.md and TASKS.md) **must** have YAML frontmatter with at minimum: `aliases`, `tags`, `creation_date`, `last_updated`
- Internal cross-references use `[[wikilinks]]`, not markdown links
- External links (Jira URLs, websites) use standard markdown links
- Filenames use **Title Case with spaces** (e.g., `Michael Zingerman.md`), not kebab-case
- Jira ticket files use `{KEY} - {Summary}` format (e.g., `XTYPE-6837 - Bug - Missing records in Comparison Results.md`)
- Always check if a file exists before creating — never overwrite vault content
- Person links in body text use `[[Full Name|Nickname]]` format
- When searching for Jira tickets in vault, use glob patterns to handle both flat files and subdirectories

### Work Week

Sunday through Thursday (Israel). Friday and Saturday are non-work days. This affects daily report date logic.

### Jira Integration

- Jira instance: `xtypeio.atlassian.net`
- Known prefixes: `XTYPE` (dev tasks), `IDEA` (product ideas)
- Jira ticket patterns detected by `[[UPPERCASE-DIGITS]]` in Obsidian links
- Ticket placement: assigned to user -> `PARA/1 Projects/{KEY} - {Summary}/`, otherwise -> `PARA/3 Resources/Jira/`
- Ticket notes must include the Jira key in `aliases` frontmatter for wikilink resolution

### Tag Conventions

| File type | Required tags |
|-----------|--------------|
| Person | `contact`, `{company-lowercase}` |
| Customer | `customer`, `xtype`, `servicenow` |
| Partner | `partner`, `xtype` |
| Jira ticket | `jira`, `{project-prefix-lowercase}` |
| Project | `project` |
| Term | `reference`, `glossary` |

### Safety Rules

- NEVER create PARA directories — they already exist in the vault
- NEVER modify existing vault files without user confirmation (CLAUDE.md and TASKS.md are exceptions)
- NEVER auto-add tasks or memories without user confirmation
- Daily report is read-only on Work Diary files
- Jira note creation always requires user confirmation
