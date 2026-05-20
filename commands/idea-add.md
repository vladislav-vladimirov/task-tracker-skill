---
name: idea-add
description: Manually record an idea, bypassing auto-detection. Engages the task-tracker skill.
---

Invoke the task-tracker skill and run the `/idea-add` operation as documented in `skills/task-tracker/SKILL.md`.

Use $ARGUMENTS as the source text for the new idea. If $ARGUMENTS is empty or too short (< 4 words), ask the user for the text before proceeding.

Argument: $ARGUMENTS
