# Design Document: Triage Command Post-Run Fixes

**Date:** 2026-02-17
**Version:** 1.4.1
**Context:** Fixes from first live triage run. Seven issues identified, grouped into two structural changes and three line edits.

## Issues Addressed

| # | Issue | Severity | Fix type |
|---|-------|----------|----------|
| 1 | Glob unreliable for conflict detection | High | Restructure Section 4 |
| 2 | No sandbox file operation fallback | High | Add to Section 6 |
| 3 | Tag mismatch: `contact` vs `person` | Medium | Line edit (3 locations) |
| 4 | `last_updated` not in vault convention | Low | Line edit (1 location) |
| 5 | No merge conflict resolution option | High | Restructure Section 6 |
| 6 | Merge execution underspecified | Medium | Covered by #5 |
| 7 | Dropped flags still in command | Non-issue | Already clean — no change needed |

## Fix 1: Section 4 — Index-First Conflict Detection

### Problem

The skill instructs `PARA/**/{filename}` glob per file. During the live run, every glob returned "No files found" — including for files that existed. Conflicts were only caught by falling back to `ls -R`.

### Design

Replace per-file glob with a two-phase approach: build a complete file index first, then match all inbox files against it.

**Phase 1 — Build file index (two steps):**

| Step | Mechanism | Purpose |
|------|-----------|---------|
| 1a. CLAUDE.md tables | Read People, Terms, Projects tables from CLAUDE.md | Instant conflict detection for known entities — no filesystem I/O |
| 1b. `find` command | `find PARA/1\ Projects PARA/2\ Areas PARA/3\ Resources -name "*.md" -type f` | Build complete file index of PARA/1-3 in a single filesystem traversal |

**Phase 2 — Match against index:**

For each MOVE candidate, check the index using the four strategies:

| Strategy | Match against |
|----------|---------------|
| Exact filename | File index paths |
| Jira key prefix | File index paths (match `{KEY}*` pattern) |
| Person name prefix | File index paths under `Areas/People/` |
| Alias match | Read frontmatter of candidate files identified in index |

Each layer narrows the set. Alias matching (the expensive one) only runs on items not already caught by filename/key/name matching.

**Fallback rule:** If `find` fails or returns empty for a directory that should have content, flag it: "Could not index {directory}. Conflict detection incomplete for this location." Continue with partial results.

### Rationale

- `find` is a single OS-level filesystem traversal — reliable where Glob tool abstraction failed
- CLAUDE.md check is instant for the ~30 most common people/terms/projects
- Excludes Inbox and Archive by construction (only searches PARA/1-3)
- All subsequent matching is string operations against the index — no per-file I/O

## Fix 2: Section 6 — Merge Resolution + Sandbox Fallback

### Problem (merge)

The skill offers three conflict resolutions: keep inbox, keep existing, skip. The most common real-world case — partial merge where the existing note is better structured but the inbox copy has unique details — has no instruction.

### Design (merge)

Add a fourth conflict resolution option:

```
CONFLICT resolution options:
- Keep inbox: enhance inbox copy, overwrite existing, delete inbox copy
- Keep existing: delete inbox copy
- Merge: read both files, append unique inbox content to existing note, delete inbox copy
- Skip: leave in inbox, no action
```

**Merge execution rules:**

1. Read both files fully.
2. Identify sections/content blocks in the inbox copy not present in the existing note. Use headings, bullet groups, or paragraph breaks as section boundaries.
3. Append unique content under a `## Merged from Inbox` heading in the existing note. If the structure is obvious (e.g., adding a phone number to a contact section), integrate inline instead.
4. Preserve the existing note's frontmatter as authority. Only add new aliases/tags from the inbox copy that the existing note lacks.
5. Delete the inbox copy after successful merge.
6. Log: `Merged: {filename} — {N} sections added to {existing path}`

Merge decisions are left to the LLM's judgment with user confirmation. The skill does not attempt content similarity scoring or automated quality comparison — it shows both files side-by-side (Section 8 rule 4), and the user picks the resolution.

### Problem (sandbox)

The skill assumes `rm` and `mv` work. In Cowork's sandboxed environment, the mounted workspace blocks deletes and moves. Desktop Commander MCP (using host filesystem paths) was needed as a workaround, but the cloud desktop doesn't know the vault's host path.

### Design (sandbox)

Add to Section 6 execution rules:

```
If a file delete or move fails (e.g., "Operation not permitted" in sandboxed environments):
1. Check if Desktop Commander MCP is available.
2. If yes, check CLAUDE.md Preferences for a stored `vault_host_path`.
   - If found, retry using that host path.
   - If not found, ask the user for the vault's host filesystem path.
     Store it in CLAUDE.md Preferences for future use.
3. If Desktop Commander is not available, ask the user for an alternative method.
4. Never silently skip a failed operation — always report the failure and the fallback attempted.
```

### Rationale (sandbox)

The cloud desktop sees the workspace at its sandbox mount path, but Desktop Commander operates on the host filesystem. There's no metadata to map between them — the user must provide the host path once, then it's stored for future runs.

## Fix 3: Tag Convention — `contact` → `person`

### Problem

Three-way tag confusion: CLAUDE.md says `contact`, skill detection uses `person`, vault has `people`. User decision: canonical tag is `person`.

### Changes

| File | Location | Before | After |
|------|----------|--------|-------|
| SKILL.md | Section 6 step 2 (tags) | `contact` for people | `person` for people |
| SKILL.md | Section 5 MOVE table example | `+tags: contact` | `+tags: person` |
| CLAUDE.md | Tag Conventions table | `Person \| contact, {company-lowercase}` | `Person \| person, {company-lowercase}` |

## Fix 4: Remove `last_updated` from Required Frontmatter

### Problem

Section 6 requires `last_updated` in frontmatter, but no existing vault notes use this field. Adding it creates inconsistency.

### Change

| File | Location | Before | After |
|------|----------|--------|-------|
| SKILL.md | Section 6 step 1 | `aliases`, `tags`, `creation_date`, `last_updated` | `aliases`, `tags`, `creation_date` |

## Fix 7: Dropped Flags — No Change Needed

The `--apply` and `--enhance` flags were already removed from `triage.md` during implementation. The command definition shows no flags. The design doc records them as dropped decisions (historical record). No changes required.

## Files Modified

| File | Changes |
|------|---------|
| `para-flow/skills/inbox-triage/SKILL.md` | Section 4 restructured, Section 5 tag example, Section 6 restructured (merge + sandbox + tag + frontmatter) |
| `CLAUDE.md` | Tag Conventions table: `contact` → `person` |
| `para-flow/commands/triage.md` | No changes needed |

## Design Decisions

| Topic | Decision | Rationale |
|-------|----------|-----------|
| Conflict detection mechanism | `find` command + CLAUDE.md tables instead of Glob | Glob failed silently in live run; `find` is OS-level, single traversal, reliable |
| Merge strategy | LLM judgment + user confirmation | Content similarity scoring is fragile; showing both files and letting user decide is more reliable |
| Sandbox fallback | Desktop Commander with stored host path | One-time user prompt, then automatic for future runs |
| Canonical person tag | `person` | User decision; aligns skill detection signal with execution |
| `last_updated` field | Removed from required | No existing vault notes use it; match actual convention |
