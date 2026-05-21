# Ideas

<!-- Managed by task-tracker skill. Hand-edit freely; format must stay consistent. -->

## Define empty-result output for filtered ideas-list
<!-- id: define-empty-result-output-for-filtered-ideas-list | status: proposed | added: 2026-05-21 | updated: 2026-05-21 | tags: ux, docs -->
The `/ideas-list <status>` operation doesn't specify what to render when the filter matches zero entries. Define a consistent fallback (e.g. `_(none)_` or an empty table with headers).

## Fix ideas.md commit sequencing on auto-done
<!-- id: fix-ideas-md-commit-sequencing-on-auto-done | status: proposed | added: 2026-05-21 | updated: 2026-05-21 | tags: fix, git -->
When a user code commit triggers auto-status → done, the `ideas.md` update happens after the commit and isn't included in it. Decide whether to amend, pre-stage the ideas.md change, or batch ideas.md updates into a separate follow-up commit.
