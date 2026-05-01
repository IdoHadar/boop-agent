# Active execution plans

In-progress, multi-PR work. One file per plan. Each plan tracks its own progress log so we can hand off cleanly.

## Plan file shape

```markdown
# <plan title>

**Status:** active
**Owner:** <name>
**Started:** YYYY-MM-DD

## Goal
What changes when this is done.

## Steps
- [ ] Step 1
- [ ] Step 2

## Decisions log
- YYYY-MM-DD — <decision> — <rationale>

## Open questions
- …
```

When a plan completes, move the file to `../completed/` unchanged. The history stays — that's the point.

## When to write a plan

Write one when the work spans more than a single PR or more than a single session. A bugfix doesn't need a plan; a feature with three PRs and an exec-plan-shaped trail does.
