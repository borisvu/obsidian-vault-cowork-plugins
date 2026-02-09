---
description: Sync tasks and refresh memory from current activity
argument-hint: "[--comprehensive]"
---

# Update Command

> See [CONNECTORS.md](../CONNECTORS.md) for tool references.

Two modes:
- **Default:** Sync Jira tasks, triage stale items, check memory for gaps
- **`--comprehensive`:** Deep scan of vault content, flag missed items, suggest new memories

## Usage

```
/productivity:update
/productivity:update --comprehensive
```

## Default Mode

### 1. Load Current State

Read `TASKS.md` and `CLAUDE.md`. If they don't exist, suggest `/productivity:start` first.

### 2. Sync from Jira

If Atlassian MCP is available, fetch issues assigned to user (open/in-progress).

Compare against TASKS.md:

| Jira issue | TASKS.md match? | Action |
|------------|-----------------|--------|
| Found, not in TASKS.md | No match | Offer to add |
| Found, already in TASKS.md | Match by key or title | Skip |
| In TASKS.md with Jira key, not in Jira results | Possibly completed | Offer to mark done |
| Completed in Jira | In Active section | Offer to mark done |

Present diff and let user decide.

### 3. Resolve Jira Links in Vault

Search for Obsidian-style Jira links (`[[XTYPE-NNNN]]`, `[[IDEA-NNNN]]`, or any `[[PREFIX-NNNN]]` matching Jira patterns) across the vault that don't have corresponding files.

For each unresolved link:
1. Query Jira for the ticket summary, status, and assignee
2. Determine placement:
   - **User is assignee OR user says they're working on it** → `PARA/1 Projects/{KEY} - {Summary}/{KEY} - {Summary}.md`
   - **Otherwise** → `PARA/3 Resources/Jira/{KEY} - {Summary}.md`
3. Create the file with Jira metadata (see Jira Note Template in daily-report skill)

When searching for existing notes, use glob patterns like `PARA/1 Projects/*/{KEY}*` and `PARA/3 Resources/Jira/{KEY}*` to handle both flat files and subdirectories.

### 4. Triage Stale Items

Review Active tasks in TASKS.md and flag:
- Tasks with due dates in the past
- Tasks in Active for 30+ days
- Tasks with no context

Present each for triage: Mark done? Reschedule? Move to Someday?

### 5. Decode Tasks for Memory Gaps

For each task, attempt to decode all entities using CLAUDE.md → PARA lookups:
```
Task: "Send PSR to Todd re: Phoenix blockers"

Decode:
- PSR → ✓ Pipeline Status Report (in CLAUDE.md)
- Todd → ✓ Todd Martinez (in People/)
- Phoenix → ? Not in memory
```

### 6. Fill Gaps

Present unknown terms grouped:
```
I found terms I don't have context for:

1. "Phoenix" (from: "Send PSR to Todd re: Phoenix blockers")
   → What's Phoenix? Should I create a project entry?

2. "Maya" (from: "sync with Maya on API design")
   → Who is Maya? Should I add to People?
```

Add answers to PARA locations and update CLAUDE.md if frequently used.

### 7. Report

```
Update complete:
- Tasks: +3 from Jira, 1 completed, 2 triaged
- Jira links: 2 new notes created
- Memory: 2 gaps filled
- All tasks decoded ✓
```

## Comprehensive Mode (`--comprehensive`)

Everything in Default Mode, plus:

### Extra: Scan Recent Vault Activity

Scan recent daily notes in `Work Diary/` for:
- Referenced people not in CLAUDE.md
- Referenced projects not tracked
- Obsidian links to non-existent files

### Extra: Scan Inbox

Check `PARA/0 Inbox/` for items that could be sorted:
- Jira tickets that should move to Projects or Resources
- People notes that should move to People/
- Anything that's been in Inbox for 7+ days

```
## Inbox items to sort
| Item | Suggestion |
|------|-----------|
| XTYPE-7292 - Audit operation... | → Move to PARA/1 Projects/ (assigned to you) |
| Damien Davis.md | → Move to PARA/2 Areas/People/ |
```

### Extra: Suggest New Memories

```
## New People (mentioned in diary but not in People/)
| Name | Frequency | Context |
|------|-----------|---------|
| Maya Rodriguez | 5 mentions | API design discussions |

## New Terms
| Term | Frequency | Context |
|------|-----------|---------|
| XAPI | 3 mentions | seems related to platform work |

## Suggested Cleanup
- Project "Horizon" — no mentions in 30 days. Archive?
```

## Notes

- Never auto-add tasks or memories without user confirmation
- Jira links are detected by pattern: `[[UPPERCASE-DIGITS]]` (e.g., `[[XTYPE-7292]]`, `[[IDEA-123]]`)
- Safe to run frequently
