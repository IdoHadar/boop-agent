# FRONTEND

Boop has two "frontends" with very different jobs.

## 1. iMessage (the actual product)

The user-facing surface. There is no app. The frontend is whatever Sendblue puts on the user's phone via Apple's iMessage protocol.

The contract on our side is the spec in [`product-specs/imessage-as-the-interface.md`](./product-specs/imessage-as-the-interface.md). The implementation is the chunker and markdown stripper in `server/sendblue.ts`.

This is the only "frontend" the user ever interacts with. Everything else in this doc is for developers working on Boop itself.

## 2. The debug dashboard (`debug/`)

A read-mostly developer tool. Vite + React + Tailwind 4 + Convex's reactive React hooks.

```
debug/
├── index.html
├── public/
├── src/
│   ├── components/
│   │   └── MemoryGraphView.tsx   ← force-directed graph of clustered memories
│   └── …
└── vite.config.ts
```

Tabs:

| Tab | What it shows | Source of truth |
|---|---|---|
| Dashboard | Cumulative spend, tokens, model breakdown, live agent status | `usageRecords`, `executionAgents` |
| Agents | Every spawned execution agent: status, cost, duration, turn count, integration logos | `executionAgents`, `agentLogs` |
| Memory | Tiered memory table + force-directed cluster graph | `memoryRecords`, `memoryEvents` |
| Automations | Cron schedule, last run, result, next run | `automations`, `automationRuns` |
| Drafts | Pending / sent / rejected drafts with raw payloads | `drafts` |
| Connections | Composio toolkit cards with account identity | `/composio/toolkits` (live) |

## Why so much UI surface for a "read-only" tool

Two reasons:

- **Memory drift is hard to feel without a picture.** The graph view shows clusters appearing, decaying, and consolidating in real time. A flat table doesn't.
- **Tool-call replay is the primary debugging surface.** When something went wrong, you scroll the Agents tab and watch the same `tool_use` / `tool_result` log the agent saw. Without it, debugging is guessing.

## Conventions

- **Convex's `useQuery` is the data layer.** No fetching state, no caching layer, no Redux. Reactive queries handle invalidation.
- **No global app store.** State is per-component or per-tab. The Convex query result is the source of truth.
- **Tailwind 4 + Hugeicons for everything visual.** No second design system.
- **One file per visualization** when a component grows past ~150 lines or owns its own data shape (e.g. `MemoryGraphView.tsx`).

## What's intentionally not here

- No user-facing app. iMessage is the app.
- No mobile dashboard. Use the desktop one.
- No auth on the dashboard. It's a single-user dev tool — bind it to localhost or a tunnel.
- No write surfaces beyond cancel/retry/disconnect. The dashboard observes; it doesn't drive the agent.
