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
