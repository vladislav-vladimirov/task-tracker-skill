# Task Tracker Skill — Design

**Date:** 2026-05-21
**Status:** Approved, ready for implementation planning
**Owner:** vladko13@gmail.com

## Problem

Development sessions generate ideas chaotically — features, refactors, fixes, experiments — and the user jumps between them. Without a structured capture mechanism, ideas are lost and their execution status becomes hard to track. A global CLAUDE.md rule already requires each project to maintain `docs/ideas.md` with Name/Description/Status, but enforcement is manual and inconsistent.

## Goal

A Claude Code plugin that:
1. Auto-detects ideas as they come up in conversation and records them.
2. Automatically updates idea status as work progresses, announcing each change.
3. Supports explicit listing, filtering, stale-idea reports, and an aesthetically-pleasing HTML report.
4. Works in any project without per-project setup.

## Non-goals

- Cross-project aggregation or central index (per-project storage only).
- A GUI, web service, or external database.
- Replacing existing task/ticket systems (Jira, Linear, GitHub Issues). This is for *in-session* idea capture.

## Architecture

**Plugin layout:**
```
task-tracker-skill/
├── .claude-plugin/
│   └── plugin.json
├── skills/
│   └── task-tracker/
│       └── SKILL.md
├── commands/
│   ├── ideas-list.md
│   ├── ideas-stale.md
│   ├── ideas-report.md
│   └── idea-add.md
└── README.md
```

**Pure prompt-driven** — all behavior (detection, parsing, status inference, HTML generation) lives in `SKILL.md` as instructions to Claude. No helper scripts. No runtime dependencies. The plugin is installed once via the Claude Code plugin system and is then available in every project.

**Per-project data** — each project has its own `docs/ideas.md`. The skill reads this at session start (if present) so Claude has current state in context.

## Storage format (`docs/ideas.md`)

```markdown
# Ideas

<!-- Managed by task-tracker skill. Hand-edit freely; format must stay consistent. -->

## Add OAuth login
<!-- id: add-oauth-login | status: in progress | added: 2026-05-12 | updated: 2026-05-20 | tags: feature, auth | ref: PR#42 -->
Support Google + GitHub sign-in via NextAuth. Replaces the current password-only flow.

## Refactor billing module
<!-- id: refactor-billing-module | status: proposed | added: 2026-05-18 | updated: 2026-05-18 | tags: refactor -->
The billing logic is spread across three files; consolidate into a single service.
```

**Field rules:**
- `id` — kebab-cased slug of the name. Stable reference for HTML reports and duplicate detection.
- `status` — one of `proposed | planned | in progress | done | dropped`. Matches the existing CLAUDE.md global rule.
- `added` — ISO date (YYYY-MM-DD), set on creation, never modified.
- `updated` — ISO date, updated on every status change.
- `tags` — comma-separated list. May be empty.
- `ref` — optional. Populated when status moves to `in progress` (file path or branch) or `done` (commit hash or PR link).
- Description — markdown text from after the comment to the next `##` heading.

**Ordering:** Entries are appended chronologically; newest at the bottom. Status changes update metadata in place — entries are never reordered.

## Auto-detection behavior

**Session start.** The skill loads `docs/ideas.md` if present so current state is in context. Missing file is fine — created on first idea capture.

**Capture threshold: anything mentioned as a possibility.** Trigger phrases include:
- "we could…", "what if we…", "it'd be nice to…"
- "let's also…", "another thing…", "TODO: …"
- "we should fix/refactor/add…"
- explicit: "track this: …", "idea: …"

**Capture flow:**
1. Detect candidate idea.
2. Fuzzy-match against existing entries (similar name or description).
3. Branch:
   - **No match:** add as `proposed`, announce: `Tracked new idea: "<name>" (proposed).`
   - **Single likely match:** ask: `This sounds like the existing idea "<X>" — append to that one or create new?`
   - **Multiple candidates:** show top 2-3 and ask which one (or create new).

**Status auto-updates (announced, not asked):**
| Trigger | New status | `ref` populated with | Announcement |
|---|---|---|---|
| User commits to plan ("we'll do X next") | `planned` | — | `Marking "X" → planned.` |
| Begin editing code mapping to a tracked idea | `in progress` | file path(s) being edited | `Marking "X" → in progress.` |
| Commit made / PR opened / user says "done" | `done` | commit hash or PR link | `Marking "X" → done (commit a1b2c3d).` |
| "drop X" / "scrap X" / "nevermind on X" | `dropped` | — | `Marking "X" → dropped.` |

`updated:` is refreshed on every status change.

**Uncertainty rule:** When it's unclear which tracked idea a piece of work maps to, ask rather than guess. When editing code that could map to multiple ideas, pick the most specific match; if none clearly fits, don't auto-update.

## Explicit operations

All available as both slash commands and natural language.

**`/ideas-list [status]`** — Show ideas, optionally filtered by status. Output: compact terminal table with name, status, age in days, tags. Grouped by status if no filter given.

**`/ideas-stale`** — Surface forgotten ideas. Flagging thresholds:
- `proposed` with no `updated` change for >30 days
- `in progress` with no `updated` change for >14 days

Output: name, status, days since updated, ref. Per-item suggestion (revive / drop / finish). Thresholds are defined as constants in `SKILL.md` so they're easy to tune.

**`/ideas-report`** — Generate `docs/ideas-report.html`.
- Self-contained single HTML file with inline CSS (no external assets).
- Header: project name, generation date, summary counts per status.
- Sections ordered: in-progress → planned → proposed → done → dropped.
- Each idea rendered as a card: name, description, dates, tags, ref link (clickable if URL).
- Visual style: clean modern light theme, status indicated by colored left-border or pill, responsive grid.
- After generation, output the file path so the user can open it.

**`/idea-add <text>`** — Manual capture. Bypasses auto-detection. Parse text → propose name/tags → ask only if ambiguous → add.

**Natural-language status overrides:** "mark X done", "X is done", "scrap X", "start on Y" — all routed through the same auto-update logic. No slash command required.

## Edge cases

- **`docs/` missing:** create on first capture, no prompt.
- **`docs/ideas.md` malformed:** parse what's parseable; for unparseable entries, leave untouched and warn in one line. Never silently lose data.
- **Ambiguous fuzzy match:** show top candidates, ask.
- **Idea raised mid-flow during unrelated work:** record with one-line announcement, return to the task.
- **Code edit maps to multiple ideas:** pick most specific; if none clearly fits, don't auto-update.
- **No git / no commits:** `ref` uses file path or stays empty. No error.
- **Concurrent edits to `ideas.md`:** re-read before every write; show diff before overwriting if a conflict is detected.
- **Project root:** prefer git root (presence of `.git`); otherwise current working directory. User can override.

## Relationship to existing CLAUDE.md rule

The user's global CLAUDE.md already requires `docs/ideas.md` with Name/Description/Status. This skill **implements** that rule rather than replacing it:
- Same file path.
- Same status vocabulary.
- Adds: HTML-comment metadata block, dates, tags, ref, automated capture/update, reporting operations.

No conflict; the skill formalizes and automates what was previously manual.

## Out of scope / future ideas

- Cross-project aggregation or a global index.
- HTML report theming / dark mode toggle.
- Export to JSON/CSV.
- Per-project configuration overrides (custom thresholds, custom file path).
- Integration with Jira / Linear / GitHub Issues.

## Success criteria

1. User can start a session in any project and have ideas captured automatically from natural conversation.
2. Status of every tracked idea reflects current reality without manual maintenance.
3. `/ideas-stale` reliably surfaces forgotten work.
4. `/ideas-report` produces an HTML file the user is happy to share or revisit.
5. Skill never silently loses an idea or status change.
