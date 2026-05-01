# Integrations as scoped MCP servers

> One toolkit per spawn. Never the full catalog.

## The rule

When the dispatcher calls `spawn_agent(task, integrations: ["gmail", "google-calendar"])`, the executor opens a fresh Composio session **scoped to those toolkits only**:

```ts
await composio.create(boopUserId(), {
  toolkits: [slug],            // scope — sub-agent only sees this toolkit's tools
  manageConnections: false,    // don't inject auth-management meta-tools
});
```

The session's tools are wrapped as an MCP server via `createSdkMcpServer({ name: slug, tools })`. The executor sees ~15 Gmail tools. It does not see Slack's 40, Notion's 30, or any other connected toolkit.

## Why scope per spawn

- **Context size.** A 200-tool catalog crushes the model's attention. A 15-tool catalog focuses it.
- **Tool-name collisions.** Multiple toolkits expose `SEND_MESSAGE`-shaped tools. The model picks wrong when they're side-by-side. Scoping eliminates the conflict.
- **Audit clarity.** Every tool call logs to Convex with the toolkit it came from. The dashboard shows logos and humanized names per toolkit, not a flat soup of slugs.
- **Cost.** Smaller context = fewer input tokens per execution turn. On long agent runs (hours of background work for an automation), this compounds.

## How a new toolkit arrives

1. User clicks Connect in the dashboard's Connections tab.
2. Backend calls `session.authorize(slug)` and returns Composio's hosted `redirectUrl`.
3. User completes OAuth on Composio's side. Tokens stay there.
4. Frontend posts `/composio/refresh` → `registerComposioToolkits()` re-runs and adds the toolkit's `IntegrationModule` to the registry keyed by slug.
5. `availableIntegrations()` now includes the slug. The dispatcher can spawn against it on the next turn.

No code change. No deploy.

## What we ruled out

- **Always-loaded full catalog.** Tested early, shipped fast, broke immediately on tool-name collisions.
- **Hand-rolled per-service integrations.** Two weeks of work per integration that never compounds. Composio's existing OAuth + adapter coverage is the entire point.
- **Tools-as-skills library.** A separate repo of "skills" (curated tool-use playbooks) sat between the dispatcher and the executor in an early version. It added a layer of indirection that was harder to debug than the direct system-prompt approach.

## What would change this

- A self-hosted deployment that can't depend on Composio would need a per-service adapter layer. The `IntegrationModule` interface in `server/integrations/registry.ts` was designed so that adapter could plug in without touching the dispatcher.
