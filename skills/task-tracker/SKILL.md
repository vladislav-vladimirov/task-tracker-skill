---
name: task-tracker
description: Use proactively at session start and whenever ideas, features, refactors, fixes, or experiments are mentioned. Auto-records ideas to docs/ideas.md, updates status as work progresses, supports /ideas-list, /ideas-stale, /ideas-report, /idea-add commands.
---

# Task Tracker

Auto-records development ideas in `docs/ideas.md` and keeps their status current as work progresses.

## When to invoke

Invoke this skill:
1. **At session start** — read `docs/ideas.md` (if it exists) into context so the current state of all tracked ideas is known.
2. **Whenever a possibility-level idea is mentioned** in conversation (see "Capture threshold" below).
3. **Whenever code editing, committing, or finishing work** could map to a tracked idea (see "Status auto-updates" below).
4. **When the user invokes a slash command** (`/ideas-list`, `/ideas-stale`, `/ideas-report`, `/idea-add`) or uses natural-language equivalents.

## Project root detection

Determine the project root:
- If a `.git` directory exists in the current working directory or an ancestor, use that directory.
- Otherwise use the current working directory.
- If the user has overridden the location ("the ideas file is at X"), honor that for the session.

The ideas file is always at `<project_root>/docs/ideas.md`.

## Session-start behavior

On the first turn of any session in a project:
1. Check whether `<project_root>/docs/ideas.md` exists.
2. If yes, read it and parse all entries (see "Storage format" section) into working context. Do not announce anything to the user — silent read.
3. If no, take no action. The file is created the first time an idea is recorded.

(Subsequent sections in this file define storage format, detection rules, status updates, operations, and edge cases.)
