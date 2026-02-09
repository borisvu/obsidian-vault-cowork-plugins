---
description: Initialize the productivity system and open the dashboard
---

# Start Command

> See [CONNECTORS.md](../CONNECTORS.md) for tool references.

Initialize task and memory systems within an existing PARA-based Obsidian vault, then open the dashboard.

## Instructions

### 1. Verify Vault Structure

Confirm the working directory is an Obsidian vault by checking for:
- `./PARA/` directory with expected subdirectories
- `./Work Diary/` directory

If these don't exist, STOP and tell the user: "This doesn't look like your Work Notes vault. Please run this from the vault root directory."

### 2. Check What Exists

Check the vault root for:
- `CLAUDE.md` — working memory hot cache
- `TASKS.md` — task list
- `dashboard.html` — visual dashboard

### 3. Create What's Missing

**If `TASKS.md` doesn't exist:** Create it with the standard template (see task-management skill). Place in vault root.

**If `dashboard.html` doesn't exist:** Copy from `${CLAUDE_PLUGIN_ROOT}/skills/dashboard.html` to vault root.

**If `CLAUDE.md` doesn't exist:** This is a fresh setup — after opening the dashboard, begin the memory bootstrap (step 5).

### 4. Open the Dashboard

Tell the user: "Dashboard is ready at `dashboard.html`. Open it from your file browser to get started."

### 5. Orient the User

If everything was already initialized:
```
Dashboard open. Your tasks and memory are loaded.
- /productivity:update to sync tasks and check memory
- /productivity:update --comprehensive for a deep scan
- /productivity:daily-report for today's daily report
```

If memory hasn't been bootstrapped yet, continue to step 6.

### 6. Bootstrap Memory (First Run Only)

Only if `CLAUDE.md` doesn't exist.

**Scan existing PARA content to populate CLAUDE.md:**

1. **People:** Read filenames and frontmatter from `PARA/2 Areas/People/`
2. **Partners/Customers:** Read from `PARA/2 Areas/Partners/` and `PARA/2 Areas/Customers/`
3. **Projects:** Recursively read from `PARA/1 Projects/` — projects may be subdirectories containing multiple notes
4. **Terms:** Read individual files from `PARA/3 Resources/Terms/` (each file is one term)

**Then scan TASKS.md** (if it has content) for unresolved shorthand — names, acronyms, project references.

**Ask the user** about anything that can't be decoded from existing vault content:
```
I found some terms in your tasks and notes I want to confirm:

1. **PSR** — What does this stand for?
2. **Michael Z** — Is this Michael Zingerman from People/?
```

### 7. Optional: Scan Jira

If Atlassian MCP is available, offer:
```
Want me to scan your Jira assigned issues to populate tasks and learn project context?
```

If yes, fetch open issues assigned to user, add to TASKS.md, and create project entries.

### 8. Write CLAUDE.md

From everything gathered, create CLAUDE.md using the template in the memory-management skill. Reference PARA paths instead of memory/ paths.

### 9. Report

```
Productivity system ready:
- Tasks: TASKS.md (X items)
- Memory: CLAUDE.md populated (X people, X terms, X projects)
- Deep memory: mapped to PARA/2 Areas/, PARA/1 Projects/, PARA/3 Resources/
- Dashboard: dashboard.html

Use /productivity:daily-report for your daily standup summary.
```

## Notes

- NEVER create PARA directories — they already exist
- If memory is already initialized, this just opens the dashboard and reports status
- Nicknames are critical — capture how people are actually referred to
