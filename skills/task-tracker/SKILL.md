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

## Storage format (`docs/ideas.md`)

Each idea is a `##` heading section with a single HTML-comment metadata line immediately below, then a markdown description until the next `##`.

**Canonical example:**

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

| Field | Format | Required | Notes |
|---|---|---|---|
| `id` | kebab-case slug of the name | yes | Stable; set on creation, never modified. |
| `status` | `proposed` \| `planned` \| `in progress` \| `done` \| `dropped` | yes | Exact strings, lowercase, with spaces (not hyphens). |
| `added` | `YYYY-MM-DD` | yes | Set on creation, never modified. |
| `updated` | `YYYY-MM-DD` | yes | Refreshed on every status change. |
| `tags` | comma-separated lowercase strings | yes | May be empty (e.g., `tags: `). |
| `ref` | free text — file path, commit hash, or PR URL | no | Populated when status moves to `in progress` or `done`. Omit the field entirely if empty. |

**Metadata line format:** ` | ` (space-pipe-space) separates fields. Field name and value separated by `: ` (colon-space). The whole line is wrapped in `<!-- ... -->`.

**File-level rules:**
- The file always starts with `# Ideas` then the managed-by comment.
- Entries are appended chronologically; **newest at the bottom**.
- Status changes update only the metadata comment in place. Entries are never reordered or removed (use `dropped` instead).
- When creating the file for the first time, write the header (`# Ideas` + managed-by comment) before the first entry.

**Parsing rules (when reading the file):**
1. Skip the `# Ideas` heading and the managed-by comment.
2. For each subsequent `## <name>` heading:
   - Read the name from the heading text.
   - Read the next line; it must be an HTML comment matching the metadata format. Parse fields.
   - Collect all following lines until the next `## ` or EOF as the description.
3. If a `##` section has no metadata comment, or the comment is unparseable, leave it untouched and emit a one-line warning (see Edge cases). Do NOT silently fix or delete.

**Writing rules (when modifying the file):**
- Re-read the file immediately before every write to avoid clobbering manual edits.
- For status updates, regenerate ONLY the metadata comment line for the affected entry; leave name and description untouched.
- For new entries, append at the end of the file (no separator beyond the standard blank line).
- Preserve existing line endings and trailing newline behavior.

## Capturing new ideas

**Threshold:** capture anything mentioned as a possibility. Better to over-capture (the user can `dropped` later) than to miss.

**Trigger phrases (non-exhaustive):**
- "we could…", "what if we…", "it'd be nice to…", "would be cool to…"
- "let's also…", "another thing…", "TODO: …", "FIXME: …"
- "we should fix/refactor/add/remove…"
- Explicit: "track this: …", "idea: …", "save this as an idea"
- A user describes a problem and a possible solution in the same turn, even if non-committal.

**Capture flow:**

1. Detect a candidate idea in the conversation.
2. **Fuzzy-match against existing entries.** Compare normalized name and description against the loaded ideas list. Match criteria:
   - Same `id` slug after kebab-casing the new name → strong match.
   - High word overlap in name (>60%) → likely match.
   - High word overlap in description (>50%) AND topical overlap in name → likely match.
3. **Branch based on match result:**

| Result | Action |
|---|---|
| No match | Add as new `proposed` entry. Announce: `Tracked new idea: "<name>" (proposed).` |
| One likely match | Ask: `This sounds like the existing idea "<X>" — append to that one or create new?` Wait for answer. |
| Multiple likely matches | Show top 2-3 candidates by name. Ask: `Possible matches: 1) <A>, 2) <B>, 3) <C>. Append to one, or create new?` |

**Naming a new entry:**
- Distill a concise, action-oriented heading (e.g., "Add rate limiting to /api/login", not "rate limiting stuff").
- Kebab-case it for the `id` field.
- If the name is ambiguous, ask the user for a one-line label before saving.

**Tags on capture:**
- Infer 0-2 tags from context (e.g., `feature`, `refactor`, `fix`, `experiment`, `cleanup`, `docs`, `perf`, plus any obvious domain like `auth`, `ui`, `db`).
- Don't invent tags when you can't infer naturally — leave empty.

**Description on capture:**
- Use the exact phrasing from the user where possible.
- One or two sentences. Capture WHAT, not HOW.

**Announcement style:**
- One line. No emojis. No follow-up question unless duplicate detection requires one.
- Example: `Tracked new idea: "Add rate limiting to /api/login" (proposed).`

**When you're uncertain** whether something is a real idea or just conversational filler, lean toward capturing — but skip if it's clearly hypothetical academic ("imagine if a system could…") or a question about existing behavior ("how does X work?").
