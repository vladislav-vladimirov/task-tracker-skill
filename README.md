# Task Tracker Skill

A Claude Code plugin that auto-records development ideas to `docs/ideas.md` per project, updates their status as work progresses, and generates listings + an aesthetic HTML report.

## What it does

- **Captures ideas automatically** as you talk about them ("we could add OAuth"). Records to `docs/ideas.md`.
- **Updates status as you work.** Starts editing a file mapped to an idea → `in progress`. Commits → `done`. Drops → `dropped`.
- **Slash commands** for explicit operations:
  - `/ideas-list [status]` — show all ideas (or filter by status)
  - `/ideas-stale` — surface ideas that have stalled
  - `/ideas-report` — generate `docs/ideas-report.html`
  - `/idea-add <text>` — manually record an idea
- **Per-project storage.** Each project has its own `docs/ideas.md`. Nothing is centralized.

## Install

This plugin is distributed via the Claude Code plugin system.

1. Clone or download this repo.
2. Install the plugin via Claude Code:
   ```
   /plugin install <path-to-task-tracker-skill>
   ```
   (Or follow current Claude Code plugin installation docs for the exact command — verify against your CLI version.)
3. Restart your session if Claude Code prompts you to.

Verify install:
```
/plugin list
```
You should see `task-tracker` listed.

## Usage

**Automatic capture.** Just talk normally:
> "We could add rate limiting to the login endpoint."

Claude will reply:
> `Tracked new idea: "Add rate limiting to login endpoint" (proposed).`

**Automatic status updates.** When you start editing the relevant code:
> `Marking "Add rate limiting to login endpoint" → in progress.`

When you commit:
> `Marking "Add rate limiting to login endpoint" → done (commit a1b2c3d).`

**Explicit commands.** See list above.

**Manual overrides.** Just say what you want:
> "Drop the OAuth idea." → `Marking "Add OAuth login" → dropped.`

## Storage format

`docs/ideas.md` is plain markdown — hand-edit freely as long as you keep the format consistent. See `skills/task-tracker/SKILL.md` § "Storage format" for the spec.

## Configuration

There is no config file. Two thresholds are defined in `SKILL.md` and can be overridden per-session by asking Claude (e.g., "use 60 days for proposed" before running `/ideas-stale`).

## Files in this plugin

| File | Purpose |
|---|---|
| `.claude-plugin/plugin.json` | Plugin manifest |
| `skills/task-tracker/SKILL.md` | All skill behavior |
| `commands/*.md` | Slash-command wrappers |
| `docs/verification-scenarios.md` | Manual test scenarios |

## Uninstall

```
/plugin remove task-tracker
```

Your `docs/ideas.md` files are NOT removed — they're per-project data.
