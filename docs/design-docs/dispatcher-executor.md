# Dispatcher / executor split

> The interaction agent decides. The execution agent does. Never the same agent.

## The shape

```
iMessage → Sendblue webhook
                  │
                  ▼
        ┌─────────────────────┐
        │  Interaction Agent  │   server/interaction-agent.ts
        │  (dispatcher only)  │   ── always runs, every turn
        │  • recall / write   │   (boop-memory)
        │  • spawn_agent      │   (boop-spawn)
        │  • send_ack         │   (boop-ack — pre-spawn one-liner)
        │  • automation tools │   (boop-automations)
        │  • draft decisions  │   (boop-draft-decisions)
        │  • self-inspection  │   (boop-self: model, timezone, integrations)
        └────────┬────────────┘
                 │  spawn_agent(task, integrations: [...])
                 ▼
        ┌─────────────────────┐
        │  Execution Agent    │   server/execution-agent.ts
        │  (per-task, ephemeral)
        │  • WebSearch        │
        │  • WebFetch         │
        │  • Composio tools   │   (only the ones the dispatcher asked for)
        │  • save_draft       │
        └─────────────────────┘
```

## What each side may do

**Dispatcher (interaction agent).** Reads the user's message + last 10 turns. Has memory, automation, draft-decision, ack, and self-inspection tools, plus one tool to spawn work. *Cannot* search the web, send email, write files, or call any external API — the SDK's built-in `WebSearch`, `WebFetch`, `Bash`, file tools, `Agent`, and `Skill` are explicitly disallowed on this side. Its system prompt drills one rule: you are a DISPATCHER, not a doer.

**Executor (execution agent).** Spawned once per task. Loads only the integrations named in the spawn call. *Cannot* respond to the user directly — its return value goes back to the dispatcher as a tool result and the dispatcher rewrites it in its own voice. *Cannot* commit external actions — those go through `save_draft` and the dispatcher's `send_draft` is the only commit path.

## Why this works

- **Cost.** Casual turns (the majority) complete in one dispatcher call. Spawning is opt-in.
- **Context size.** A Gmail-search agent doesn't need Notion's 30 tools in its context. Per-spawn scoping keeps execution context tiny.
- **Determinism.** The dispatcher's behavior is predictable because its toolset is small. We can reason about when it will and won't spawn. The early failure mode — "the model just tries the email tool" — was eliminated by making it structurally impossible.
- **Audit.** Every tool call, tool result, and text block on the execution side is logged to Convex. The dashboard replays runs deterministically.

## What we ruled out

- **One agent with all tools.** Tried it. Context bloat made models slow on simple turns. Tool-name collisions across toolkits caused the model to pick the wrong sender.
- **Dispatcher with WebSearch.** Tempting on quick lookups, but every shortcut we add to the dispatcher pulls behavior across the boundary. Keeping the dispatcher pure means we can reason about its cost.
- **Sub-sub-agents.** The execution agent does not get `spawn_agent`. Recursive spawning made cost unbounded and audit trails harder to reason about. If an execution task needs decomposition, we update the dispatcher to plan more carefully — not push planning down.

## What would change this

- A real-time use case where a single tool round-trip dominates (e.g. a voice loop) might justify collapsing the split for that surface. The dispatcher/executor model is for batch-style chat where one user turn maps to one logical unit of work.
