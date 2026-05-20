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

## Status auto-updates

Status changes are **automatic** and **announced** (one line, no confirmation requested).

**Transition table:**

| Trigger condition | New status | `ref` value | Announcement template |
|---|---|---|---|
| User commits to plan ("we'll do X next", "let's plan to tackle X this week") without starting work | `planned` | unchanged | `Marking "<name>" → planned.` |
| First `Edit` or `Write` tool call against a file that maps to a tracked idea | `in progress` | the file path being edited | `Marking "<name>" → in progress.` |
| A `git commit` is made that clearly relates to the idea | `done` | commit hash (`git log -1 --format=%h`) | `Marking "<name>" → done (commit <hash>).` |
| A PR is opened/merged related to the idea | `done` | PR URL or `PR#<number>` | `Marking "<name>" → done (PR#<n>).` |
| User says "done", "X is done", "finished X", "shipped X" | `done` | last known commit hash if available, else empty | `Marking "<name>" → done.` |
| User says "drop X", "scrap X", "nevermind on X", "kill X" | `dropped` | unchanged | `Marking "<name>" → dropped.` |

**On every status change, also:**
- Update `updated:` to today's date (YYYY-MM-DD).
- Re-read `docs/ideas.md` first; modify only the metadata line of the affected entry; write back.

**Mapping work to an idea:**

Before auto-updating, you must be **confident** the action maps to a specific tracked idea. Apply this test:

1. Does the file being edited (or the commit message, or the spoken phrase) name or clearly reference an existing idea by name, slug, or distinctive keyword?
2. If yes → update.
3. If multiple ideas could match → pick the most specific (longest shared keyword overlap). If still ambiguous → don't auto-update; either ask the user or proceed without updating.
4. If no idea clearly fits → don't auto-update. Just keep working.

**Things that do NOT trigger status updates:**
- Reading files (no edit).
- Editing files that don't map to any tracked idea.
- Running tests, linters, or other read-only tools.
- Conversation about an idea without starting work on it.

**Manual overrides** via natural language are routed through the same logic:
- "mark X as in progress" / "start on Y" → `in progress`
- "X is done" / "mark X done" → `done`
- "drop X" / "scrap X" → `dropped`
- "plan X" / "let's plan X for later" → `planned`

When the user explicitly states a status, do not ask — just apply it and announce.

## Operations: list, filter, stale

These produce terminal output (text tables, not files).

### `/ideas-list [status]` — list ideas

**Natural-language equivalents:** "show ideas", "list ideas", "what ideas do we have", "show in-progress ideas".

**Behavior:**
1. Re-read `docs/ideas.md`.
2. If a status filter is provided, include only matching entries; otherwise include all and group by status.
3. Sort within each status group by `updated:` descending (most recent first).
4. Render as a compact markdown table.

**Output format (no filter):**

```
### in progress (1)
| Name | Age | Updated | Tags |
|---|---|---|---|
| Add OAuth login | 9d | 0d ago | feature, auth |

### proposed (1)
| Name | Age | Updated | Tags |
|---|---|---|---|
| Refactor billing module | 3d | 3d ago | refactor |

### planned (0)
_(none)_
```

Render sections in this order: `in progress`, `planned`, `proposed`, `done`, `dropped`. Always show the first three even if empty (render `_(none)_`). Skip `done` and `dropped` entirely if empty.

**Output format (with filter):** drop the heading hierarchy; just show the single table.

**Age** = days since `added:`. **Updated** = days since `updated:` (`0d ago` for today).

### `/ideas-stale` — surface forgotten ideas

**Natural-language equivalents:** "what's stale", "any stalled ideas", "what have I forgotten".

**Thresholds (constants):**
- `STALE_PROPOSED_DAYS = 30` — flag `proposed` ideas with `updated >= 30` days ago.
- `STALE_IN_PROGRESS_DAYS = 14` — flag `in progress` ideas with `updated >= 14` days ago.

**Behavior:**
1. Re-read `docs/ideas.md`.
2. Compute days since `updated:` for each entry.
3. Filter to entries that exceed their threshold.
4. Sort by days-since-updated descending.
5. Render output; if zero stale entries, output: `No stale ideas. ✓` (no other punctuation; the checkmark is the only non-ASCII).

**Output format:**

```
### Stale ideas (2)
| Name | Status | Days idle | Ref | Suggested action |
|---|---|---|---|---|
| Add OAuth login | in progress | 18d | src/auth/oauth.ts | finish or drop |
| Refactor billing | proposed | 45d | — | revive, drop, or plan |
```

**Suggested action heuristic:**
- `in progress` and idle: `finish or drop`
- `proposed` and idle: `revive, drop, or plan`
- `planned` is never auto-flagged by default.

After the table, offer: `Want me to update statuses for any of these?`

**Adjusting thresholds:** the user may say "use 60 days for proposed" → use the override for this session only; do not modify SKILL.md.

## Operations: HTML report

### `/ideas-report` — generate `docs/ideas-report.html`

**Natural-language equivalents:** "generate a report", "make an HTML report", "give me a report I can share".

**Behavior:**
1. Re-read `docs/ideas.md`.
2. Generate a single self-contained HTML file: inline CSS, no external assets, no JavaScript.
3. Write to `<project_root>/docs/ideas-report.html` (overwrite if exists).
4. Output: `Wrote report → docs/ideas-report.html (open it in a browser to view).`

**HTML structure (use this template; substitute values):**

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>Ideas — {{ PROJECT_NAME }}</title>
  <style>
    :root {
      --bg: #fafafa; --fg: #1a1a1a; --muted: #6b7280; --card: #ffffff;
      --border: #e5e7eb; --accent: #2563eb;
      --proposed: #6b7280; --planned: #2563eb; --in-progress: #d97706;
      --done: #059669; --dropped: #9ca3af;
    }
    * { box-sizing: border-box; }
    body { margin: 0; font: 16px/1.5 -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
           background: var(--bg); color: var(--fg); }
    .container { max-width: 1100px; margin: 0 auto; padding: 32px 24px 64px; }
    header { margin-bottom: 32px; }
    h1 { margin: 0 0 8px; font-size: 28px; }
    .meta { color: var(--muted); font-size: 14px; }
    .summary { display: flex; flex-wrap: wrap; gap: 12px; margin-top: 16px; }
    .pill { padding: 4px 10px; border-radius: 999px; font-size: 12px; font-weight: 600;
            background: #fff; border: 1px solid var(--border); }
    .pill.proposed { color: var(--proposed); }
    .pill.planned { color: var(--planned); }
    .pill.in-progress { color: var(--in-progress); }
    .pill.done { color: var(--done); }
    .pill.dropped { color: var(--dropped); }
    section { margin-top: 32px; }
    section h2 { font-size: 18px; margin: 0 0 12px; text-transform: capitalize; }
    .grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(320px, 1fr)); gap: 16px; }
    .card { background: var(--card); border: 1px solid var(--border); border-left: 4px solid var(--proposed);
            border-radius: 8px; padding: 16px; }
    .card.proposed { border-left-color: var(--proposed); }
    .card.planned { border-left-color: var(--planned); }
    .card.in-progress { border-left-color: var(--in-progress); }
    .card.done { border-left-color: var(--done); }
    .card.dropped { border-left-color: var(--dropped); opacity: 0.7; }
    .card h3 { margin: 0 0 8px; font-size: 16px; }
    .card p { margin: 0 0 12px; color: #374151; }
    .card .footer { display: flex; flex-wrap: wrap; gap: 8px; align-items: center;
                    color: var(--muted); font-size: 12px; }
    .card .tag { padding: 2px 8px; border-radius: 4px; background: #f3f4f6; }
    .card a { color: var(--accent); }
    .empty { color: var(--muted); font-style: italic; }
  </style>
</head>
<body>
  <div class="container">
    <header>
      <h1>Ideas — {{ PROJECT_NAME }}</h1>
      <div class="meta">Generated {{ GENERATED_AT }} · {{ TOTAL_COUNT }} ideas total</div>
      <div class="summary">
        <span class="pill in-progress">In progress · {{ COUNT_IN_PROGRESS }}</span>
        <span class="pill planned">Planned · {{ COUNT_PLANNED }}</span>
        <span class="pill proposed">Proposed · {{ COUNT_PROPOSED }}</span>
        <span class="pill done">Done · {{ COUNT_DONE }}</span>
        <span class="pill dropped">Dropped · {{ COUNT_DROPPED }}</span>
      </div>
    </header>

    <!-- Repeat per status, in this order: in-progress, planned, proposed, done, dropped -->
    <section>
      <h2>In progress</h2>
      <div class="grid">
        <!-- one .card per idea -->
        <article class="card in-progress">
          <h3>{{ NAME }}</h3>
          <p>{{ DESCRIPTION_HTML_ESCAPED }}</p>
          <div class="footer">
            <span>added {{ ADDED }}</span>
            <span>· updated {{ UPDATED }}</span>
            <!-- tags -->
            <span class="tag">{{ TAG }}</span>
            <!-- ref: if URL, render as link; if commit hash, plain text; if file path, plain text -->
            <span>· ref: <a href="{{ REF_URL }}">{{ REF_LABEL }}</a></span>
          </div>
        </article>
        <!-- ... -->
      </div>
      <!-- If zero entries for this section, omit the section entirely (do not render an empty grid). -->
    </section>
    <!-- ... -->
  </div>
</body>
</html>
```

**Substitution rules:**
- `{{ PROJECT_NAME }}` — basename of the project root.
- `{{ GENERATED_AT }}` — human-readable date+time (e.g., `2026-05-21 14:32`).
- Counts — integers; if zero, still render the pill (so totals are always visible).
- `{{ DESCRIPTION_HTML_ESCAPED }}` — escape `&`, `<`, `>` from the markdown description; preserve line breaks as `<br>`.
- `ref` rendering:
  - Starts with `http://` or `https://` → render as `<a href>` with the URL as label.
  - Matches `PR#\d+` → render as plain text (no link, since we don't know the repo).
  - Matches a git hash (7-40 hex chars) → render as plain text.
  - Otherwise → render as plain text (likely a file path).
- Sections with zero entries: omit the entire section (no empty grids).
- Order of sections (skip if empty): `in progress`, `planned`, `proposed`, `done`, `dropped`.

**Self-contained constraint:** the file must open with no network access and no external files. Verify by writing then re-reading the file and confirming no `<script src=>`, `<link href=>`, or `<img src="http">` references exist.

## Operations: `/idea-add`

### `/idea-add <text>` — manual capture

**Natural-language equivalents:** "save this as an idea: …", "track this: …", "add idea: …".

**Behavior:**
1. Take the user's text as the source.
2. Distill a concise action-oriented name (kebab-case it for the `id`).
3. Infer 0-2 tags from the text.
4. Use the original text (or a one-sentence trim) as the description.
5. Run fuzzy-match against existing entries.
   - If match → ask same question as auto-capture flow.
   - If no match → add as `proposed` and announce normally.
6. If the text is too short to derive a clear name (e.g., < 4 words), ask: `One-line label for this idea?` Then proceed.

This bypasses the threshold check used in auto-detection — anything passed to `/idea-add` is captured.

## Edge cases

| Situation | Handling |
|---|---|
| `docs/` directory missing | Create it on first write. No prompt. |
| `docs/ideas.md` missing | Create with header (`# Ideas` + managed-by comment) on first write. |
| File malformed (e.g., a section missing its metadata comment) | Parse what's parseable. Leave malformed entries untouched. Emit one-line warning: `Couldn't parse N entries in ideas.md — left as-is. Run /ideas-list to see what I did read.` |
| Fuzzy match ambiguous (multiple candidates) | Show top 2-3 candidates by name; ask which (or create new). |
| Idea raised mid-flow during unrelated work | Still record. One-line announcement. Return to the prior task immediately. |
| Code edit could map to multiple ideas | Pick the most specific (longest shared keyword overlap). If still ambiguous, don't auto-update — keep working. |
| Git not initialized | `ref` falls back to a file path (or empty if neither commit nor file is known). No error. |
| User edits `docs/ideas.md` manually mid-session | Re-read the file before every write. If parsed content has changed since last read, show a one-line diff summary before overwriting. |
| User overrides project root ("the ideas file is at …") | Honor the override for the current session. Do not persist it. |
| User asks to delete an entry | Don't actually delete — set status to `dropped`. Explain: "Entries aren't deleted; marking dropped preserves history. If you want a true delete, edit the file by hand." |
| User asks to rename an entry | Update the `## <name>` heading and re-kebab the `id`. Update `updated:` date. Existing `ref` is preserved. |
| Status word the user uses doesn't match the vocabulary (e.g., "marked as 'wip'") | Map to the nearest valid status (`wip` → `in progress`). Announce the mapping: `Mapping "wip" → "in progress".` |
| Description contains markdown that could break parsing (e.g., starts with `##`) | Indent or escape the offending lines so they don't look like new entries. Verify by re-reading after write. |

## Anti-patterns (do not do)

- **Do not** silently lose ideas or status changes. Every write must be preceded by a read.
- **Do not** create empty announcement messages — every status change has the one-line format from the transition table.
- **Do not** invoke this skill for clearly hypothetical academic discussion ("imagine if a system could…") or for questions about existing behavior.
- **Do not** modify other projects' `docs/ideas.md` — only the current project's file.
- **Do not** edit the `# Ideas` header or the managed-by comment line. These are anchors.
- **Do not** persist any session-only overrides (custom thresholds, custom file paths) back into SKILL.md.
