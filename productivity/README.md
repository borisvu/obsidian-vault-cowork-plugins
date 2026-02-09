# Productivity Plugin (PARA Edition)

A productivity plugin for [Cowork](https://claude.com/product/cowork) and Claude Code, adapted for Obsidian vaults using the PARA organizational method.

## What It Does

- **Task management** — `TASKS.md` for the active backlog, readable and writable by Claude
- **Workplace memory** — Two-tier system: `CLAUDE.md` hot cache + deep storage mapped to PARA folders
- **Daily reports** — Generates daily standup reports from Work Diary entries
- **Jira integration** — Resolves Jira ticket links, creates vault entries for referenced tickets
- **Visual dashboard** — HTML board view of tasks and memory

## PARA Mapping

| Plugin concept | PARA location |
|---------------|---------------|
| People profiles | `PARA/2 Areas/People/` |
| Partner companies | `PARA/2 Areas/Partners/` |
| Customer companies | `PARA/2 Areas/Customers/` |
| Active projects | `PARA/1 Projects/` |
| Terms & glossary | `PARA/3 Resources/Terms/` |
| Company context | `PARA/3 Resources/Company/` |
| Archived projects | `PARA/4 Archive/` |
| Daily notes | `Work Diary/YYYY/MM/YYYY-MM-DD.md` |
| Weekly notes | `Work Diary/gggg/MM/gggg-[W]ww.md` |

## Commands

| Command | What it does |
|---------|-------------|
| `/start` | Initialize CLAUDE.md, TASKS.md, dashboard; bootstrap memory from vault |
| `/update` | Triage stale tasks, sync Jira, check memory gaps |
| `/daily-report` | Generate daily report from Work Diary entries |
| `/daily-report since YYYY-MM-DD` | Report covering multiple days since given date |

## Installation

    claude plugins add boris-plugins/productivity
