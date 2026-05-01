# PLANS

How Boop tracks multi-PR work without it rotting in someone's head.

## The two kinds of plan

**Lightweight plans.** A bug fix or a single-PR feature doesn't need a plan. The PR description is the plan. Don't over-document.

**Execution plans.** Anything that spans multiple PRs, multiple sessions, or has reversible decisions in it. These get a file in [`exec-plans/active/`](./exec-plans/active/) and stay there until the work lands. When it lands, the file moves to [`completed/`](./exec-plans/completed/) unchanged.

## File shape

See [`exec-plans/active/README.md`](./exec-plans/active/README.md). Three sections that matter: **Goal**, **Steps**, **Decisions log**. The decisions log is the part that survives the work — it's the trail of "why we did it this way" that future you (or a future agent working on the codebase) can read in five minutes.

## Tech debt

Conscious shortcuts go in [`exec-plans/tech-debt-tracker.md`](./exec-plans/tech-debt-tracker.md). Each row has a **what would force a fix** column — that's the trigger for promoting a shortcut into an execution plan.

When you're tempted to add a `// TODO` comment, add a row there instead. TODO comments rot in place; tracked rows get reviewed.

## How agents fit

Boop is a personal-agent template, but the codebase itself is human-written. The reason this `docs/` tree exists in the same shape an agent-generated codebase would use is that the principle is the same either way: future-you (or a Codex-style agent working on a fork) needs the system of record in the repo, not in your head.

If you ever do let an agent drive multi-PR work in this repo, the plan file in `exec-plans/active/` is the handoff format.
