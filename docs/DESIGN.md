# DESIGN

> The "why" of Boop, in one screen. For "what," read [`ARCHITECTURE.md`](../ARCHITECTURE.md). For "how to use," read [`README.md`](../README.md).

Boop is a personal agent you text. Every architectural choice serves that single sentence.

## The four operating principles

Each one has its own design doc — these are the headlines.

1. **The interface is iMessage.** No app, no login, no UI. The interface is the constraint.
2. **Most turns shouldn't spawn anything.** A casual reply must be one cheap dispatcher call. Spawning is opt-in.
3. **Dispatcher decides, executor does.** Two agents, two tool surfaces, two lifecycles. Never the same agent.
4. **Memory forgets on purpose.** Decay, archive, prune. Relevance, not retention.

Full text: [`docs/design-docs/core-beliefs.md`](./design-docs/core-beliefs.md).

## What the system is made of

- **Interaction agent** (`server/interaction-agent.ts`) — the dispatcher. Tiny toolset. Always runs.
- **Execution agent** (`server/execution-agent.ts`) — per-task, ephemeral. Loads only the integrations the dispatcher named.
- **Memory** (`server/memory/`) — extract, decay, consolidate. Three layers, each one fire-and-forget.
- **Drafts** (`server/draft-tools.ts`) — every external action stages before it commits.
- **Automations** (`server/automations.ts`) — cron on top of executor spawns.
- **Composio integrations** (`server/composio.ts`) — per-spawn toolkit-scoped MCP servers.
- **Convex** — durable state and the reactive substrate the dashboard reads.
- **Debug dashboard** (`debug/`) — Vite + React UI over Convex queries.

The full prose explanation lives in [`ARCHITECTURE.md`](../ARCHITECTURE.md).

## The decision records

| Decision | Doc |
|---|---|
| Why split dispatcher from executor | [dispatcher-executor.md](./design-docs/dispatcher-executor.md) |
| How memory tiers, decay, and segments work | [memory-system.md](./design-docs/memory-system.md) |
| Why three roles in consolidation | [consolidation-pipeline.md](./design-docs/consolidation-pipeline.md) |
| Why drafts are structural, not advisory | [draft-safety.md](./design-docs/draft-safety.md) |
| Why per-spawn toolkit scoping | [integrations-as-mcp.md](./design-docs/integrations-as-mcp.md) |

## The non-decisions

These are choices we made *by not making them*:

- **No user auth.** Single-tenant template. Add Clerk if you fork for multi-user.
- **No knowledge graph.** `supersedes` edges only.
- **No proactive context gathering** in the default template — too opinionated about what to watch.
- **No skills library** beyond what's in `.claude/skills/` — too Boop-specific to bake in.

Each of these is a one-file change. The point of the template is the smallest surface that still actually works.

## Where the line is

If a change increases the dispatcher's tool surface, slows the casual-turn path, or moves an external send out of the draft layer — it crosses one of the four principles. Crossing the line is fine when warranted, but it needs a design doc that names the override.
