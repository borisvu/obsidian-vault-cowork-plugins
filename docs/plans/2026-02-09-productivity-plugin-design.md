# Productivity Plugin — Implementation Design

> **Date:** 2026-02-09
> **Status:** Approved
> **Based on:** `implementation-spec.md` with deltas from vault structure analysis

## Overview

Implement the `productivity` plugin — a PARA-integrated fork of Anthropic's productivity plugin for Cowork/Claude Code, adapted for Boris's Obsidian vault.

## Deltas from Original Spec

Based on `tree` analysis of the actual `PARA/` vault structure, these changes were agreed:

| Spec says | Actual vault | Change |
|-----------|-------------|--------|
| Jira tickets as flat files: `1 Projects/XTYPE-1234.md` | Subdirectories: `1 Projects/XTYPE-6837 - Summary/XTYPE-6837 - Summary.md` | Match vault: `{KEY} - {Summary}/` subdir with `.md` inside |
| Single glossary: `3 Resources/Terms/Glossary.md` | Individual term files: `3 Resources/Terms/QBR.md` | Individual `Terms/{TERM}.md` files |
| No mention of `0 Inbox/` | Inbox exists with unsorted items | Scan Inbox during `--comprehensive` update |
| Projects are flat | Projects have subfolders | Recursive scanning for project context |
| `plugin.json` at productivity root | Source uses `.claude-plugin/plugin.json` | Use `.claude-plugin/plugin.json` |

## File Manifest

```
productivity/
├── .claude-plugin/plugin.json
├── .mcp.json
├── CONNECTORS.md
├── README.md
├── CLAUDE.md                    # Working memory template
├── commands/
│   ├── start.md
│   ├── update.md
│   └── daily-report.md
└── skills/
    ├── memory-management/SKILL.md
    ├── task-management/SKILL.md
    ├── daily-report/SKILL.md
    └── dashboard.html           # Copied as-is from source
```

Root:
```
README.md
LICENSE                          # Apache 2.0 from source
```

## Affected Spec Files

- **memory-management/SKILL.md** — PARA Location Mapping table updated for subdirs and individual term files; recursive project scanning note added
- **daily-report/SKILL.md** and **commands/daily-report.md** — Jira Note Template uses subdirectory convention; vault search uses glob patterns
- **commands/update.md** — Jira link resolution creates subdirs; comprehensive mode scans Inbox
- **productivity/CLAUDE.md** — Terms pointer updated to `PARA/3 Resources/Terms/` (directory, not single file)

## Implementation Order

1. Save source assets (dashboard.html, LICENSE)
2. Remove source `productivity/` folder
3. Create directory structure
4. Write all plugin files with spec content + deltas
5. Copy dashboard.html and LICENSE into place
6. Create root README.md and LICENSE
7. Update root CLAUDE.md for actual structure
8. Git init + initial commit
