# Task Tracker Skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Claude Code plugin (`task-tracker`) that auto-detects ideas in conversation, records them in per-project `docs/ideas.md`, automatically updates status as work progresses, and supports listing, stale reports, and HTML report generation.

**Architecture:** Pure prompt-driven plugin — all behavior lives in a single `SKILL.md` file with thin slash-command wrappers. No helper scripts, no runtime dependencies. Per-project data in `docs/ideas.md` (markdown sections with HTML-comment metadata). Installed once via the Claude Code plugin system, then available in every project.

**Tech Stack:** Markdown only. Plugin scaffolding via Claude Code plugin format (`.claude-plugin/plugin.json`).

**TDD adaptation:** Because the deliverable is natural-language instructions (not executable code), each task produces a section of the skill, then verifies by (a) reading the file back to confirm structure and (b) — for the final task — running scenario tests against the installed skill.

**Spec reference:** `docs/superpowers/specs/2026-05-21-task-tracker-skill-design.md`

---

## File Structure

```
task-tracker-skill/
├── .claude-plugin/
│   └── plugin.json              # Plugin manifest
├── skills/
│   └── task-tracker/
│       └── SKILL.md             # Single skill file — all behavior
├── commands/
│   ├── ideas-list.md            # /ideas-list
│   ├── ideas-stale.md           # /ideas-stale
│   ├── ideas-report.md          # /ideas-report
│   └── idea-add.md              # /idea-add
├── README.md                    # Install + usage
└── docs/
    ├── superpowers/
    │   ├── specs/
    │   │   └── 2026-05-21-task-tracker-skill-design.md  (already exists)
    │   └── plans/
    │       └── 2026-05-21-task-tracker-skill.md         (this file)
    └── verification-scenarios.md  # Manual test scenarios for the final task
```

**Responsibilities:**
- `plugin.json` — declares the plugin to Claude Code.
- `skills/task-tracker/SKILL.md` — entire behavior of the skill (the one file the LLM actually loads at runtime).
- `commands/*.md` — slash-command wrappers that pre-route to specific skill operations.
- `README.md` — install instructions and usage examples for the user.
- `docs/verification-scenarios.md` — manual scripts the user runs after install to confirm the skill works.

---

## Task 1: Plugin scaffold

**Files:**
- Create: `.claude-plugin/plugin.json`
- Create: `skills/task-tracker/SKILL.md` (empty placeholder)
- Create: `README.md` (skeleton)
- Create: `.gitignore`

- [ ] **Step 1: Create `.gitignore`**

```
# OS / editor
.DS_Store
Thumbs.db
*.swp
.vscode/
.idea/

# Generated reports (per-project, not tracked here)
docs/ideas-report.html
```

- [ ] **Step 2: Create `.claude-plugin/plugin.json`**

```json
{
  "name": "task-tracker",
  "version": "0.1.0",
  "description": "Auto-tracks development ideas across all projects in docs/ideas.md, updates status as work progresses, and generates listing + HTML reports.",
  "author": {
    "name": "vladko13",
    "email": "vladko13@gmail.com"
  }
}
```

> **Note:** Confirm the exact plugin.json schema against current Claude Code plugin docs before proceeding. Common alternative keys: `displayName`, `repository`, `entryPoints`. Adjust if the format has changed.

- [ ] **Step 3: Create `skills/task-tracker/SKILL.md` placeholder**

```markdown
---
name: task-tracker
description: PLACEHOLDER — will be filled in Task 2
---

PLACEHOLDER — content added in subsequent tasks.
```

- [ ] **Step 4: Create `README.md` skeleton**

```markdown
# Task Tracker Skill

A Claude Code plugin that auto-records development ideas to `docs/ideas.md`, updates their status as work progresses, and generates listings + HTML reports.

## Install
TBD — filled in Task 10.

## Usage
TBD — filled in Task 10.
```

- [ ] **Step 5: Verify scaffold**

Run:
```bash
ls -la .claude-plugin/ skills/task-tracker/
```
Expected: `plugin.json` and `SKILL.md` exist, are non-empty, are valid format (JSON parses, YAML frontmatter parses).

```bash
python -c "import json; json.load(open('.claude-plugin/plugin.json'))"
```
Expected: no error.

- [ ] **Step 6: Commit**

```bash
git add .claude-plugin/ skills/ README.md .gitignore
git commit -m "scaffold: plugin manifest, skill placeholder, readme, gitignore"
```

---

## Task 2: SKILL.md — frontmatter and activation behavior

**Files:**
- Modify: `skills/task-tracker/SKILL.md` (replace placeholder)

This task defines WHEN the skill activates and what it does at session start. It does NOT yet define how it parses or updates ideas — those come in later tasks.

- [ ] **Step 1: Replace the placeholder with the frontmatter + activation section**

Write the full file at `skills/task-tracker/SKILL.md`:

```markdown
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
```

- [ ] **Step 2: Read the file back and verify**

Read `skills/task-tracker/SKILL.md` and confirm:
- YAML frontmatter parses (no syntax errors).
- `description` field reads as a self-contained sentence the LLM can match against (mentions both auto-tracking and the slash commands).
- "When to invoke" lists all four trigger types.
- "Project root detection" specifies the git-root-then-cwd fallback.

- [ ] **Step 3: Commit**

```bash
git add skills/task-tracker/SKILL.md
git commit -m "skill: frontmatter and activation/session-start behavior"
```

---

## Task 3: SKILL.md — storage format and parsing

**Files:**
- Modify: `skills/task-tracker/SKILL.md` (append section)

Define the exact `docs/ideas.md` format so the LLM can read AND write it consistently across sessions and across projects.

- [ ] **Step 1: Append "Storage format" section to SKILL.md**

Append to `skills/task-tracker/SKILL.md`:

````markdown

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
````

- [ ] **Step 2: Read back and verify**

Read the file and confirm:
- Field table is complete (all 6 fields documented).
- Both parsing and writing rules are present.
- The "re-read before every write" rule is included.

- [ ] **Step 3: Commit**

```bash
git add skills/task-tracker/SKILL.md
git commit -m "skill: storage format and parsing/writing rules"
```

---

## Task 4: SKILL.md — auto-detection of new ideas

**Files:**
- Modify: `skills/task-tracker/SKILL.md` (append section)

Define what counts as an idea and how to handle fuzzy duplicates.

- [ ] **Step 1: Append "Capturing new ideas" section**

Append to `skills/task-tracker/SKILL.md`:

````markdown

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
````

- [ ] **Step 2: Read back and verify**

Confirm:
- Trigger phrases section is present and concrete.
- Three-way branching (no match / one match / multiple matches) is documented.
- Announcement format is explicit and consistent with the spec.

- [ ] **Step 3: Commit**

```bash
git add skills/task-tracker/SKILL.md
git commit -m "skill: auto-detection of new ideas with duplicate handling"
```

---

## Task 5: SKILL.md — status auto-updates

**Files:**
- Modify: `skills/task-tracker/SKILL.md` (append section)

Define when and how status moves between values without asking the user.

- [ ] **Step 1: Append "Status auto-updates" section**

Append to `skills/task-tracker/SKILL.md`:

````markdown

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
````

- [ ] **Step 2: Read back and verify**

Confirm:
- Transition table covers all five status values (`proposed` → others).
- The "mapping work to an idea" test is present and gates auto-updates.
- "Things that do NOT trigger" list is included to prevent overzealous updates.

- [ ] **Step 3: Commit**

```bash
git add skills/task-tracker/SKILL.md
git commit -m "skill: status auto-update rules and mapping confidence test"
```

---

## Task 6: SKILL.md — list, filter, and stale operations

**Files:**
- Modify: `skills/task-tracker/SKILL.md` (append section)

Define the three terminal-output operations: full list, status filter, stale report.

- [ ] **Step 1: Append "Operations: list, filter, stale" section**

Append to `skills/task-tracker/SKILL.md`:

````markdown

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

Repeat sections for `done` and `dropped` only if there are entries (skip empty sections except for the ones above the dividing line in this order: in progress, planned, proposed, then done, dropped — skip if zero).

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
- `planned` and idle (>30 days): `start or drop` (only flagged if you choose to extend thresholds; default: planned is never auto-flagged)

After the table, offer: `Want me to update statuses for any of these?`

**Adjusting thresholds:** the user may say "use 60 days for proposed" → use the override for this session only; do not modify SKILL.md.
````

- [ ] **Step 2: Read back and verify**

Confirm:
- Both `/ideas-list` and `/ideas-stale` sections are present.
- Thresholds (30 and 14) are explicit and named as constants.
- Output formats are concrete (table structure shown).

- [ ] **Step 3: Commit**

```bash
git add skills/task-tracker/SKILL.md
git commit -m "skill: list, filter, and stale-report operations"
```

---

## Task 7: SKILL.md — HTML report generation

**Files:**
- Modify: `skills/task-tracker/SKILL.md` (append section)

Define how to generate the aesthetically-pleasing self-contained HTML report.

- [ ] **Step 1: Append "Operations: HTML report" section**

Append to `skills/task-tracker/SKILL.md`:

````markdown

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
````

- [ ] **Step 2: Read back and verify**

Confirm:
- Full HTML template is present, inline CSS only, no external assets.
- All five statuses are color-coded.
- Substitution rules cover all `{{ ... }}` placeholders.
- Ref rendering rules are explicit.

- [ ] **Step 3: Commit**

```bash
git add skills/task-tracker/SKILL.md
git commit -m "skill: HTML report generation with inline-CSS template"
```

---

## Task 8: SKILL.md — `/idea-add` manual capture + edge cases

**Files:**
- Modify: `skills/task-tracker/SKILL.md` (append section)

- [ ] **Step 1: Append `/idea-add` and "Edge cases" sections**

Append to `skills/task-tracker/SKILL.md`:

````markdown

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
````

- [ ] **Step 2: Read back and verify**

Confirm:
- `/idea-add` section is present with the "too short" fallback.
- Edge-case table covers at least 10 distinct situations.
- Anti-patterns section is present and explicit.

- [ ] **Step 3: Commit**

```bash
git add skills/task-tracker/SKILL.md
git commit -m "skill: idea-add operation, edge cases, anti-patterns"
```

---

## Task 9: Slash-command wrappers

**Files:**
- Create: `commands/ideas-list.md`
- Create: `commands/ideas-stale.md`
- Create: `commands/ideas-report.md`
- Create: `commands/idea-add.md`

Each wrapper is a thin pointer telling Claude to engage the skill's corresponding operation.

- [ ] **Step 1: Create `commands/ideas-list.md`**

```markdown
---
name: ideas-list
description: List tracked ideas in this project, optionally filtered by status. Engages the task-tracker skill.
---

Invoke the task-tracker skill and run the `/ideas-list` operation as documented in `skills/task-tracker/SKILL.md`.

If a status argument is provided (`proposed`, `planned`, `in progress`, `done`, `dropped`), filter to that status. Otherwise show all ideas grouped by status.

Argument: $ARGUMENTS
```

- [ ] **Step 2: Create `commands/ideas-stale.md`**

```markdown
---
name: ideas-stale
description: Surface ideas that have been stalled for too long. Engages the task-tracker skill.
---

Invoke the task-tracker skill and run the `/ideas-stale` operation as documented in `skills/task-tracker/SKILL.md`.

Use the default thresholds (proposed: 30 days, in progress: 14 days) unless overridden in $ARGUMENTS (e.g., "60 days for proposed").

Argument: $ARGUMENTS
```

- [ ] **Step 3: Create `commands/ideas-report.md`**

```markdown
---
name: ideas-report
description: Generate an HTML report of all tracked ideas at docs/ideas-report.html. Engages the task-tracker skill.
---

Invoke the task-tracker skill and run the `/ideas-report` operation as documented in `skills/task-tracker/SKILL.md`.

Write a self-contained HTML report (inline CSS, no external assets) to `<project_root>/docs/ideas-report.html`, overwriting any existing file. Report the path back to the user.
```

- [ ] **Step 4: Create `commands/idea-add.md`**

```markdown
---
name: idea-add
description: Manually record an idea, bypassing auto-detection. Engages the task-tracker skill.
---

Invoke the task-tracker skill and run the `/idea-add` operation as documented in `skills/task-tracker/SKILL.md`.

Use $ARGUMENTS as the source text for the new idea. If $ARGUMENTS is empty or too short (< 4 words), ask the user for the text before proceeding.

Argument: $ARGUMENTS
```

- [ ] **Step 5: Verify all four files**

Run:
```bash
ls commands/
```
Expected: 4 `.md` files.

Read each file; confirm YAML frontmatter parses and each references `skills/task-tracker/SKILL.md`.

- [ ] **Step 6: Commit**

```bash
git add commands/
git commit -m "commands: slash-command wrappers for list/stale/report/add"
```

---

## Task 10: README + install/usage docs

**Files:**
- Modify: `README.md` (replace skeleton with full content)

- [ ] **Step 1: Replace `README.md` with full content**

```markdown
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
```

- [ ] **Step 2: Verify README renders cleanly**

Read the file; confirm:
- All four slash commands are documented.
- Install + uninstall instructions are present.
- Configuration note about thresholds is present.

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "docs: full README with install, usage, and storage format"
```

---

## Task 11: Verification scenarios + manual test pass

**Files:**
- Create: `docs/verification-scenarios.md`

This task is the substitute for an automated test suite. It produces a script of scenarios for the user to run after install, and the engineer should run through each scenario at least once before declaring the plan complete.

- [ ] **Step 1: Create `docs/verification-scenarios.md`**

````markdown
# Task Tracker — Verification Scenarios

Manual test scenarios to run after installing the plugin. Run them in a scratch project (e.g., create a temp directory, `git init`, install the plugin, then run Claude Code there).

Each scenario lists the input, expected behavior, and what to check.

---

## S1: First idea in fresh project

**Setup:** Fresh directory, no `docs/ideas.md` yet.

**Input:** "We could add a dark mode toggle to the settings page."

**Expected:**
- Claude announces: `Tracked new idea: "Add dark mode toggle to settings page" (proposed).`
- File `docs/ideas.md` is created with header + one entry.
- Metadata: `status: proposed`, `added` = today, `updated` = today, `tags` includes at least `feature` or `ui`, no `ref`.

**Check:** `cat docs/ideas.md` — file exists with correct structure.

---

## S2: Duplicate detection

**Setup:** From S1 state.

**Input:** "It'd be cool to add a dark theme option in settings."

**Expected:**
- Claude detects this is similar to "Add dark mode toggle to settings page".
- Claude asks: `This sounds like the existing idea "Add dark mode toggle to settings page" — append to that one or create new?`

**Check:** No new entry is created until you answer.

---

## S3: Status auto-update on edit

**Setup:** Idea exists from S1. Create a file `src/settings.tsx`.

**Input:** Ask Claude to "add the dark mode toggle to src/settings.tsx".

**Expected:**
- Before/while editing, Claude announces: `Marking "Add dark mode toggle to settings page" → in progress.`
- `docs/ideas.md` entry: `status: in progress`, `updated` = today, `ref: src/settings.tsx`.

**Check:** `cat docs/ideas.md` after — status flipped.

---

## S4: Status auto-update on commit

**Setup:** From S3 state, file edited and staged.

**Input:** "Commit this with message 'feat: dark mode toggle'."

**Expected:**
- Commit is made.
- Claude announces: `Marking "Add dark mode toggle to settings page" → done (commit <hash>).`
- `docs/ideas.md` entry: `status: done`, `ref` = the commit hash.

**Check:** `git log -1 --format=%h` matches the `ref` in `ideas.md`.

---

## S5: `/ideas-list`

**Input:** `/ideas-list`

**Expected:** Table output grouped by status, with the dark-mode idea under `done`. Empty sections rendered as `_(none)_`.

**Input:** `/ideas-list proposed`

**Expected:** Single table, only proposed entries (none if S4 completed).

---

## S6: `/ideas-stale`

**Setup:** Manually edit `docs/ideas.md` to set an entry's `updated` to 40 days ago and status to `proposed`. Or wait 30 days. (For testing, manual edit is fine.)

**Input:** `/ideas-stale`

**Expected:** Table listing the stale entry with `Days idle: 40d` and suggested action `revive, drop, or plan`.

**Input (when nothing stale):** `/ideas-stale`

**Expected:** `No stale ideas. ✓`

---

## S7: `/ideas-report`

**Input:** `/ideas-report`

**Expected:**
- Claude announces: `Wrote report → docs/ideas-report.html (open it in a browser to view).`
- File exists.
- Open in browser: layout shows summary pills, status sections, cards per idea. No broken links, no missing assets.
- View source: no `<script src=>`, no `<link href="http...">`, no `<img src="http...">`.

**Check:** `grep -E 'src="https?://|href="https?://' docs/ideas-report.html` returns only user-supplied `ref` URLs, if any.

---

## S8: `/idea-add` manual capture

**Input:** `/idea-add Migrate logging to structured JSON output`

**Expected:**
- Claude adds a new entry, status `proposed`, today's date, tags inferred (e.g., `refactor` or `cleanup`).
- Announces: `Tracked new idea: "Migrate logging to structured JSON output" (proposed).`

**Input:** `/idea-add fix bug`

**Expected:** Claude asks for a clearer label since the text is too short.

---

## S9: Manual override via natural language

**Input:** "Drop the dark mode idea."

**Expected:**
- Claude flips status to `dropped`.
- Announces: `Marking "Add dark mode toggle to settings page" → dropped.`

---

## S10: Malformed file handling

**Setup:** Manually corrupt one entry in `docs/ideas.md` (remove its metadata comment).

**Input:** `/ideas-list`

**Expected:**
- Claude emits one-line warning: `Couldn't parse 1 entry in ideas.md — left as-is. Run /ideas-list to see what I did read.`
- The list output excludes the broken entry; the file is NOT modified.

**Check:** `cat docs/ideas.md` — corrupted entry is still corrupted.

---

## Pass criteria

All 10 scenarios produce the expected behavior. If any fail, file an idea about it (eat the dog food) and fix before declaring the plan complete.
````

- [ ] **Step 2: Install the plugin locally and run through scenarios S1–S10**

```bash
# in a scratch directory
mkdir /tmp/tracker-test && cd /tmp/tracker-test
git init
# follow the install instructions from README.md to install task-tracker-skill plugin
# launch claude code in this directory
# run each scenario in order, recording pass/fail
```

For each scenario, mark pass or fail in your notes. If any fail, fix the relevant section of `SKILL.md` and re-run that scenario.

- [ ] **Step 3: Commit verification doc**

```bash
git add docs/verification-scenarios.md
git commit -m "docs: verification scenarios for manual test pass"
```

- [ ] **Step 4: Final commit if any fixes were needed**

If steps in scenarios S1–S10 surfaced bugs you fixed in `SKILL.md`, commit those fixes with descriptive messages (e.g., `fix(skill): clarify ref field for git-less projects`).

---

## Self-Review

### Spec coverage check
- ✅ Plugin structure (Task 1) — matches spec § Architecture.
- ✅ Storage format with all 6 fields (Task 3) — matches spec § Storage format.
- ✅ Session-start read (Task 2) — matches spec § Auto-detection behavior.
- ✅ Threshold = "anything mentioned as a possibility" (Task 4) — matches spec.
- ✅ Status auto-updates with announcement, no confirmation (Task 5) — matches spec.
- ✅ Status transition table (Task 5) — covers all 5 statuses.
- ✅ Fuzzy duplicate detection w/ ask (Task 4, Task 8) — matches spec.
- ✅ `/ideas-list [status]` (Task 6) — matches spec.
- ✅ `/ideas-stale` with thresholds 30 / 14 (Task 6) — matches spec.
- ✅ `/ideas-report` HTML output (Task 7) — matches spec, includes inline-CSS constraint.
- ✅ `/idea-add` (Task 8) — matches spec.
- ✅ Natural-language overrides (Task 5) — matches spec.
- ✅ Edge cases all 8 from spec (Task 8) — covered + extras (rename, delete, status mapping).
- ✅ Project root detection (Task 2) — matches spec.
- ✅ Relationship to existing CLAUDE.md rule (Task 8 via storage format) — implicit; not explicitly mentioned in skill, but the format and statuses match. Acceptable.
- ✅ Success criteria 1-5 (verification scenarios S1-S10).

### Placeholder scan
- The single `PLACEHOLDER` text in Task 1 Step 3 is intentional (it gets replaced in Task 2). Otherwise no TBD/TODO/vague language in implementation steps.

### Type / name consistency
- File paths consistent across tasks: `skills/task-tracker/SKILL.md`, `commands/*.md`, `docs/ideas.md`, `docs/ideas-report.html`.
- Status values consistently spelled: `proposed | planned | in progress | done | dropped` (spaces not hyphens in storage, hyphens in CSS class names — explicitly handled).
- Thresholds consistently named: `STALE_PROPOSED_DAYS = 30`, `STALE_IN_PROGRESS_DAYS = 14`.
- Field names consistent: `id`, `status`, `added`, `updated`, `tags`, `ref`.

No fixes needed.
