# AGENTS.md

> Map, not manual. This file is the table of contents. Deep knowledge lives in `docs/`. If you read only one section, read **Where to look**.

## What Boop is

A personal agent you text. iMessage in, iMessage out, with sub-agents and integrations behind it. See [`README.md`](./README.md) for product context, [`ARCHITECTURE.md`](./ARCHITECTURE.md) for the system shape, and [`docs/DESIGN.md`](./docs/DESIGN.md) for the principles every other doc inherits from.

## Where to look

| When you're working on… | Start here |
|---|---|
| The dispatcher (interaction agent) | [`docs/design-docs/dispatcher-executor.md`](./docs/design-docs/dispatcher-executor.md) → `server/interaction-agent.ts` |
| The executor (execution agent) | [`docs/design-docs/dispatcher-executor.md`](./docs/design-docs/dispatcher-executor.md) → `server/execution-agent.ts` |
| Memory writes, decay, recall | [`docs/design-docs/memory-system.md`](./docs/design-docs/memory-system.md) → `server/memory/` |
| Memory consolidation | [`docs/design-docs/consolidation-pipeline.md`](./docs/design-docs/consolidation-pipeline.md) → `server/consolidation.ts` |
| The draft / confirm flow | [`docs/design-docs/draft-safety.md`](./docs/design-docs/draft-safety.md) → `server/draft-tools.ts` |
| Composio toolkits | [`docs/design-docs/integrations-as-mcp.md`](./docs/design-docs/integrations-as-mcp.md) → `server/composio.ts`, `server/integrations/` |
| Automations / cron | [`docs/product-specs/automations.md`](./docs/product-specs/automations.md) → `server/automations.ts` |
| The debug dashboard | [`docs/FRONTEND.md`](./docs/FRONTEND.md) → `debug/` |
| Convex schema or tables | [`docs/generated/db-schema.md`](./docs/generated/db-schema.md) → `convex/schema.ts` |
| Reliability questions | [`docs/RELIABILITY.md`](./docs/RELIABILITY.md) |
| Security / PII / secrets | [`docs/SECURITY.md`](./docs/SECURITY.md) and [`CLAUDE.md`](./CLAUDE.md) |
| Multi-PR work | [`docs/PLANS.md`](./docs/PLANS.md) → [`docs/exec-plans/active/`](./docs/exec-plans/active/) |
| Known shortcuts | [`docs/exec-plans/tech-debt-tracker.md`](./docs/exec-plans/tech-debt-tracker.md) |
| Why we said no | [`docs/PRODUCT_SENSE.md`](./docs/PRODUCT_SENSE.md) |

## Top-level docs

| File | Purpose |
|---|---|
| [`docs/DESIGN.md`](./docs/DESIGN.md) | The "why" of Boop in one screen |
| [`docs/FRONTEND.md`](./docs/FRONTEND.md) | iMessage surface + the debug dashboard |
| [`docs/PLANS.md`](./docs/PLANS.md) | How multi-PR work is tracked |
| [`docs/PRODUCT_SENSE.md`](./docs/PRODUCT_SENSE.md) | What ships, what doesn't, what we say no to |
| [`docs/QUALITY_SCORE.md`](./docs/QUALITY_SCORE.md) | Per-domain health snapshot |
| [`docs/RELIABILITY.md`](./docs/RELIABILITY.md) | Failure modes and what we catch |
| [`docs/SECURITY.md`](./docs/SECURITY.md) | Trust boundaries, secrets, PII rules |

## Subdirectories

```
docs/
├── design-docs/     ← decision records (the "why")
├── exec-plans/      ← multi-PR work, in-progress and archived
│   ├── active/
│   ├── completed/
│   └── tech-debt-tracker.md
├── generated/       ← auto-generated from code (don't hand-edit)
├── product-specs/   ← user-visible surfaces ("what good looks like")
├── references/      ← vendor-published docs we depend on
└── superpowers/     ← workflow specs (planning skill output)
```

## Operating principles (one line each)

1. **iMessage is the interface.** No app, no login, no UI surface for the end user.
2. **Most turns shouldn't spawn anything.** Default-fast, spawn-when-asked.
3. **Dispatcher decides, executor does.** Two agents, two tool surfaces, one commit path.
4. **Memory forgets on purpose.** Relevance, not retention.

Full text: [`docs/design-docs/core-beliefs.md`](./docs/design-docs/core-beliefs.md). Anything that crosses one of these needs a design doc that names the override.

## Don't put it here

This file is a map. Do not let it grow past one screen. New rules go in the design docs they belong to. New context goes in the relevant spec or principle. If a future change makes this file ambiguous, the fix is to point somewhere new — not to inline the explanation here.

<!-- convex-ai-start -->
This project uses [Convex](https://convex.dev) as its backend.

When working on Convex code, **always read `convex/_generated/ai/guidelines.md` first** for important guidelines on how to correctly use Convex APIs and patterns. The file contains rules that override what you may have learned about Convex from training data.

Convex agent skills for common tasks can be installed by running `npx convex ai-files install`.
<!-- convex-ai-end -->

## Pre-commit checks (this is a public repo)

Before staging or committing **any** file, scan it for personal or sensitive data. Once it lands in git history it's effectively permanent — even after rewriting history, GitHub keeps orphaned commit SHAs reachable for weeks.

What counts as PII / sensitive in this repo:
- Personal email addresses (the maintainer's or anyone else's), phone numbers, postal addresses, full names of non-public individuals.
- Live API keys, tokens, secrets, or anything from `.env.local`.
- Composio connection IDs (`ca_*`), connected-account aliases, OAuth state, refresh tokens.
- Personal search queries / financial details / message content from real conversations (e.g. "rent", landlord names, contacts' names).
- Production URLs, internal hostnames, or anything that maps a public identifier to a private account.

Where this most often slips in:
- One-off debug scripts under `scripts/` and `debug/` written during a real session.
- Test fixtures hand-copied from real responses.
- Comments or commit messages quoting real data.
- README/docs examples that use real values instead of placeholders.

Process:
1. Before `git add`, read each new or modified file end-to-end and substitute any real values with environment variables, CLI args, or generic placeholders (`user@example.com`, `ca_REDACTED`).
2. Prefer not committing ad-hoc debug scripts at all — keep them in your shell history or a gitignored scratch dir.
3. If you realize PII slipped in **before pushing**, amend or reset and re-commit cleanly.
4. If it already pushed, see the recovery steps in [GitHub's sensitive data docs](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/removing-sensitive-data-from-a-repository) and rotate any exposed credentials.
