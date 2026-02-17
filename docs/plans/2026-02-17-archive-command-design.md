# Project Lifecycle & Archive Command — Design

**Date:** 2026-02-17
**Author:** Boris Vulikh
**Status:** Validated
**Target Version:** 1.3.0

## Problem

The PARA/1 Projects folder accumulates completed, merged, or abandoned project notes. Cleanup is manual: the user must remember which projects are stale, check Jira statuses individually, and move folders to the year-based archive. This is error-prone and rarely happens on time.

The CLAUDE.md Projects table also drifts out of sync with reality — no automated mechanism reconciles it against actual folder state and Jira statuses.

## Goals

1. Single command that scans PARA/1 Projects, enriches each project from diary notes and Jira, and recommends which to archive.
2. Move confirmed stale projects to `PARA/4 Archive/Archive {YYYY}/`, following established folder conventions.
3. Update CLAUDE.md Projects table after archiving.
4. Reconcile the Projects table against current PARA/1 Projects state (add missing, remove gone, update statuses).
5. Safe: always show plan and require user confirmation before any file moves.

## Current Archive Convention

```
PARA/4 Archive/
├── Archive 2025/
│   ├── PROJECT-4585/                         ← standalone project
│   ├── Database evaluation PROJECT-5647/     ← standalone project
│   └── PROJECT-6599 - Bug empty fields/      ← standalone project
└── Archive 2026/
    ├── Data Management/                      ← parent umbrella
    │   ├── Calculating Simple... PROJECT-5984/
    │   ├── Cleanup expired/
    │   └── Creating Database Schemas.../
    ├── Onboarding/
    └── PROJECT-6837 - Bug.../                ← standalone project
```

Patterns: standalone Jira tickets are top-level under the year. Sub-projects of a parent are archived under `Archive {YYYY}/{Parent Name}/`. The parent folder in PARA/1 Projects/ is not moved — only completed sub-folders are extracted. Archive year is when archiving happens, not when the project was created.

## Design

### New Files

| File | Purpose |
|------|---------|
| `commands/archive.md` | Thin orchestrator command (~100-120 lines) |
| `skills/project-lifecycle/SKILL.md` | 10 numbered referenceable sections (~250-300 lines) |

### Command Signature

```
/para-flow:archive                  → scan, classify, present plan, ask to proceed
/para-flow:archive --since 90       → override staleness threshold (default: 30 days)
```

No `--apply` flag. The command always shows the plan and asks for confirmation interactively. YAML frontmatter:

```yaml
---
description: Scan projects for staleness, recommend and execute archiving
argument-hint: "[--since N]"
---
```

### Skill Structure (project-lifecycle)

10 numbered sections, referenced by the command as "Use project-lifecycle skill Section N (Name)":

| # | Section | Purpose |
|---|---------|---------|
| 1 | Core Concepts | Definitions: project, umbrella, sub-project, staleness, archive vs resources |
| 2 | Inventory Rules | Scan PARA/1 Projects/, extract Jira keys, detect parent/child, read frontmatter |
| 3 | Diary Enrichment | Grep Work Diary/ for project mentions, extract last-mention dates |
| 4 | Jira Enrichment | Fields to fetch (status, resolution, updated, assignee), MCP-unavailable fallback |
| 5 | Classification Rules | ARCHIVE / REVIEW / ACTIVE thresholds, `--since N` override |
| 6 | Archive Placement | Target path rules for standalone, sub-project, umbrella, resources |
| 7 | Presentation Format | Table templates for each classification group |
| 8 | Post-Move Updates | CLAUDE.md removal, TASKS.md cleanup, move logging |
| 9 | Project Table Sync | Full reconciliation: scan PARA/1, compare to CLAUDE.md, add/remove/update |
| 10 | Important Rules | Safety rules, confirmation requirements, read-only constraints |

### Execution Stages

The command runs 6 sequential stages:

| # | Stage | Description | Data Sources |
|---|-------|-------------|--------------|
| 1 | Inventory | List all folders in PARA/1 Projects/ and sub-folders | File system (glob) |
| 2 | Enrich from diary | For each project, find last mention in Work Diary | Work Diary/YYYY/MM/*.md |
| 3 | Enrich from Jira | For Jira-linked projects, fetch current status | Atlassian MCP |
| 4 | Classify | Apply staleness rules | Output of stages 1-3 |
| 5 | Present plan | Show recommendation tables, ask for confirmation | Output of stage 4 |
| 6 | Execute | Move confirmed projects, update CLAUDE.md and TASKS.md | User confirmation |

### Classification Rules (Section 5)

**ARCHIVE** — any of:
- Jira status is Done, Closed, Merged, or Resolved
- Jira resolution is Fixed, Done, Won't Do, or Duplicate
- Frontmatter `status` is Merged, Done, Closed, Resolved, or Cancelled
- No diary mention in 60+ days AND no Jira activity in 60+ days
- CLAUDE.md "What" column contains "merged", "done", "closed", or "archived"

**REVIEW** — any of (and no ARCHIVE criteria matched):
- No diary mention in 30–59 days
- Jira status is Backlog with no activity in 30+ days
- Frontmatter `status` is Backlog with no diary references
- Project looks reference-like (no tasks, no diary mentions) — suggest Resources

**ACTIVE** — default when neither matched. Positive signals:
- Diary mention within 30 days
- Jira status is In Progress, In Review, Open, or To Do
- Listed in current week's goals

**`--since N` override:** Replaces the 30-day base threshold. REVIEW becomes N to 2N-1 days, ARCHIVE becomes 2N+ days. Jira terminal statuses (Done/Closed/Merged) always trigger ARCHIVE regardless of `--since`.

**Umbrella rule:** Umbrellas are never classified on their own. Only sub-projects are classified. If all sub-projects reach ARCHIVE, the umbrella itself becomes ARCHIVE.

### Archive Placement (Section 6)

| Scenario | Target Path |
|----------|-------------|
| Standalone project | `PARA/4 Archive/Archive {YYYY}/{folder_name}/` |
| Sub-project of active parent | `PARA/4 Archive/Archive {YYYY}/{parent_name}/{sub_folder_name}/` |
| All sub-projects done → entire umbrella | `PARA/4 Archive/Archive {YYYY}/{parent_name}/` |
| User picks Resources for a REVIEW item | `PARA/3 Resources/{folder_name}/` |

Year is always current year. Create `Archive {YYYY}/` and parent sub-folders if needed — these are the only directories the command creates. If target already exists in archive, ask user: merge or rename.

### Presentation Format (Section 7)

Output groups projects by classification:

```
## Ready to Archive

| #  | Project                          | Reason                          | Target                                    |
|----|----------------------------------|---------------------------------|-------------------------------------------|
| 1  | PROJECT-7371 - Sync fails        | Jira: Merged                    | Archive 2026/PROJECT-7371 - Sync fails/   |
| 2  | PROJECT-7457 - Stuck definitions | Jira: Done, no diary 45d        | Archive 2026/PROJECT-7457 - Stuck defs/   |
| 3  | Data Management/Data copy        | Jira: Done, last diary Jan 5    | Archive 2026/Data Management/Data copy/   |

## Needs Review

| #  | Project                          | Reason                          | Suggestion                                |
|----|----------------------------------|---------------------------------|-------------------------------------------|
| 4  | PROJECT-6860 - No query results  | No diary mention 65d            | Archive or re-activate?                   |
| 5  | Snow Ingestion Pipeline          | Reference docs only, no tasks   | Move to Resources?                        |

## Active (no action)

| Project              | Last Activity                    |
|----------------------|----------------------------------|
| Data Management      | Today (umbrella, 2 active subs)  |
| PROJECT-7406         | Feb 10 (Jira: In Progress)       |
| PROJECT-7451         | Today (in weekly goals)          |
```

Confirmation prompt lets user cherry-pick items by number. For REVIEW items, asks per-item: archive, move to Resources, or skip.

### Post-Move Updates (Section 8)

1. **CLAUDE.md** — Remove archived projects from Projects table.
2. **TASKS.md** — Find tasks referencing archived projects. Present to user: mark done, remove, or keep. Never auto-remove.
3. **Move log** — Print summary of all moves, CLAUDE.md changes, and TASKS.md flags.

### Project Table Sync (Section 9)

Runs after archive moves. Reconciles CLAUDE.md Projects table against current PARA/1 Projects/:

| Situation | Action |
|-----------|--------|
| Folder in PARA/1 but not in table | Add row (What from frontmatter or Jira summary) |
| Row in table but folder gone | Remove row |
| Both exist | Update What with current Jira status if available |

Presents diff to user before writing. Table capped at ~15 entries (hot cache principle). Overflow noted as "> +N more in PARA/1 Projects/". Umbrellas get one row, not per sub-project. Unknown descriptions use "TODO: describe".

### Edge Cases

**Umbrella with non-project files:** If umbrella folder contains files that aren't sub-project folders (e.g., meeting notes), warn user and ask whether to include or leave behind.

**Non-Jira projects:** Classification relies on diary mentions + frontmatter only. No data available → REVIEW with "check manually" suggestion. Reference-like content → suggest Resources.

**Jira MCP unavailable:** Stage 3 skipped. Log warning. More projects may land in REVIEW instead of ARCHIVE since terminal Jira statuses can't be detected.

**Empty PARA/1 Projects/:** Print "No projects found. Nothing to do." and exit.

**20+ projects:** Process all, paginate presentation if needed. Table Sync caps at ~15 entries.

## Changes to Existing Files

| File | Change |
|------|--------|
| `para-flow/.claude-plugin/plugin.json` | Version 1.2.0 → 1.3.0 |
| `para-flow/commands/update.md` | Add advisory line after Stage 7: count Done/Closed frontmatter statuses in PARA/1, suggest `/para-flow:archive` if any found (~5 lines) |
| `para-flow/README.md` | Add archive command, changelog entry for 1.3.0 |
| `CLAUDE.md` (repo root) | Add archive command to structure table and command descriptions |

## Files NOT Changed

| File | Why |
|------|-----|
| `skills/daily-report/SKILL.md` | Archive reads diary files directly, no skill changes needed |
| `skills/memory-management/SKILL.md` | Project Table Sync lives in project-lifecycle skill |
| `commands/start.md` | No changes needed |
| `commands/standup.md` | No changes needed |
| `para-flow/CLAUDE.md` (template) | Keep 2-column table, no structural change |

## Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| No `--apply` flag | Always interactive | Simpler, safer, consistent with plugin's confirm-first philosophy |
| Numbered skill sections | 10 sections | Consistent with daily-report v1.2.0 pattern |
| Project Table Sync in project-lifecycle skill | Not in memory-management | Keeps all project logic co-located |
| Advisory-only update integration | Frontmatter check, no classification | Avoids logic duplication, fast |
| No link scanning | Log moves only | Trust Obsidian's link updating |
| Keep 2-column Projects table | Inline status in What | Backward compatible, less structural change |
| `--since N` scales both thresholds | REVIEW=N to 2N-1, ARCHIVE=2N+ | Predictable, single knob |
| Jira terminal statuses always ARCHIVE | Regardless of --since | Done means done |

## Safety Rules

1. No file moves without explicit user confirmation.
2. Default mode is always scan-and-recommend.
3. Never delete files — archive means move.
4. Never modify Work Diary files (read-only).
5. Log all moves in a summary after execution.
6. Only create Archive year folders and parent sub-folders — never create PARA/1, 2, or 3 directories.

## Future Work (Not In Scope)

- `/para-flow:update --comprehensive` calling Project Table Sync independently
- Vault-wide broken link detection and repair
- Automated scheduling / periodic archive reminders
- Undo/restore from archive
