# Para Flow — Second Brain Productivity

A Second Brain productivity plugin for [Cowork](https://claude.com/product/cowork) and Claude Code, built for Obsidian vaults organized with the PARA method.

## What It Does

- **Task management** — `TASKS.md` for the active backlog, readable and writable by Claude
- **Workplace memory** — Two-tier system: `CLAUDE.md` hot cache + deep storage mapped to PARA folders
- **Daily reports** — Generates daily standup reports from Work Diary entries
- **Jira integration** — Resolves Jira ticket links, creates vault entries for referenced tickets
- **Visual dashboard** — HTML board view of tasks and memory

## PARA Mapping

| Plugin concept     | PARA location                      |
| ------------------ | ---------------------------------- |
| People profiles    | `PARA/2 Areas/People/`             |
| Partner companies  | `PARA/2 Areas/Partners/`           |
| Customer companies | `PARA/2 Areas/Customers/`          |
| Active projects    | `PARA/1 Projects/`                 |
| Terms & glossary   | `PARA/3 Resources/Terms/`          |
| Company context    | `PARA/3 Resources/Company/`        |
| Archived projects  | `PARA/4 Archive/`                  |
| Daily notes        | `Work Diary/YYYY/MM/YYYY-MM-DD.md` |
| Weekly notes       | `Work Diary/gggg/MM/gggg-[W]ww.md` |

## Commands

| Command                         | What it does                                                           |
| ------------------------------- | ---------------------------------------------------------------------- |
| `/start`                        | Initialize CLAUDE.md, TASKS.md, dashboard; bootstrap memory from vault |
| `/update`                       | Triage stale tasks, sync Jira, check memory gaps                       |
| `/standup`                      | Generate laconic standup report from Work Diary entries                 |
| `/standup --full`               | Generate comprehensive daily report with all sections                  |
| `/standup since YYYY-MM-DD`     | Standup covering multiple days since given date                        |
| `/archive`                        | Scan projects for staleness, recommend and execute archiving           |
| `/archive --since N`              | Override staleness threshold (default: 30 days)                        |

## Installation

    Add a marketplace from GitHub repo

## Changelog

### 1.3.0

- Added `/archive` command for project lifecycle management
- New project-lifecycle skill with 10 referenceable sections
- Scans PARA/1 Projects/ for stale projects using diary mentions, Jira status, and frontmatter
- Three-tier classification: ARCHIVE, REVIEW, ACTIVE
- Safe interactive workflow: always shows plan, requires confirmation before moving files
- Moves projects to `PARA/4 Archive/Archive {YYYY}/` following established conventions
- Supports umbrella projects with sub-project archiving
- Reconciles CLAUDE.md Projects table after archiving
- Added project health advisory to `/update` command report

### 1.2.0

- Renamed `/daily-report` command to `/standup` with laconic default format
- Added `--full` flag to `/standup` for comprehensive daily reports
- Added morning mode: before 10:00 AM, treats today's work as empty
- Added standup section filtering: excludes "Free notes" and "To do on next working day" from standup output
- Added mode-aware link formatting: brief keys in standup, full descriptions in `--full`
- Restructured daily-report skill into 10 numbered, referenceable sections
- Eliminated ~50% duplication between command and skill

### 1.1.0

- Renamed plugin from `productivity` to `para-flow`
- Removed PII, updated for public release
- Added Firefox support for dashboard

### 1.0.1

- Added Firefox support for dashboard

### 1.0.0

- Initial implementation: task management, two-tier memory, daily reports, Jira integration
