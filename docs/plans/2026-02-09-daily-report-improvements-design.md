# Daily Report Improvements — Design

**Date:** 2026-02-09
**Status:** Validated

## Context

After real usage of the daily-report command, five recurring problems were identified:

1. **Date confusion** — Claude guessed dates and days of the week incorrectly
2. **Failed to check files** — Claimed files didn't exist without reading them
3. **Over-complicated output** — Added summaries, observations, analysis when user wanted simple bullets
4. **Misunderstood "enhance"** — Interpreted edit requests as "add more" instead of "improve in-place"
5. **Over-engineered edits** — When editing daily notes, added elaborate descriptions instead of matching user's minimal style

Additionally, two new capabilities are needed:

- **Time MCP** — Local MCP server for reliable date/time/day-of-week
- **Weekly alignment** — Compare daily work against weekly goals and quarterly/yearly vision

## Changes

### Section 1: Infrastructure (3 files)

**`productivity/.mcp.json`** — Add time MCP server:
```json
"time": {
  "command": "node",
  "args": ["/Users/borisvulikh/git/context-stack/packages/time-mcp/dist/index.js"]
}
```

**`productivity/CONNECTORS.md`** — Add `~~time` category mapped to time MCP.

**`CLAUDE.md` (repo root)** — Add to Safety Rules:
- Always use time MCP for date/time/day-of-week — never guess
- Always read a file before claiming it doesn't exist
- When asked to edit a daily note, make minimal changes matching user's style
- Default to simple, factual, bulleted output — no summaries or analysis unless asked

### Section 2: Daily Report Command (`commands/daily-report.md`)

- **New Step 0:** Call time MCP to get current date, time, and day of week before any date logic
- **Step 2 strengthened:** "Attempt to read every file. Only report missing after a failed read."
- **Multi-day format simplified:** Remove `### Summary` and `### Observations` sections. Keep Day-by-day + Carry-forward only.
- **New step — Weekly alignment:**
  1. Get ISO week number from time MCP
  2. Find `Work Diary/YYYY/MM/YYYY-Www.md`
  3. If missing: warn user to create one
  4. If goals empty: warn user to add goals
  5. If goals exist: compare work items against week goals, output checkbox status + one-liner nudge

### Section 3: Daily Report Skill (`skills/daily-report/SKILL.md`)

Mirror all command changes:
- Time MCP as date source of truth
- Simplified multi-day format
- Weekly alignment section with path pattern, extraction rules, output format
- Updated Important Rules (read before reporting missing, factual output)
- Priorities table (P0/P1/P2) used for context in nudge, not full comparison

## Design Decisions

- **Daily-report stays read-only** on Work Diary files. Editing daily notes is a separate behavior triggered by explicit user requests, governed by the new CLAUDE.md rule.
- **Weekly alignment runs during daily-report**, not during update. Lightweight checkpoint at report time.
- **Nudge is one-liner** after the goals checkbox list — factual + suggestion, not analysis.
- **Year/quarter vision** provides context for the nudge but isn't compared item-by-item every time.

## Files Changed

| File | Change |
|------|--------|
| `productivity/.mcp.json` | Add time MCP server |
| `productivity/CONNECTORS.md` | Add ~~time category |
| `CLAUDE.md` (repo root) | Add 4 behavioral rules to Safety Rules |
| `productivity/commands/daily-report.md` | Step 0, Step 2, simplified format, weekly alignment |
| `productivity/skills/daily-report/SKILL.md` | Mirror command changes |
