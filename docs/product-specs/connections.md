# Spec: connections

## What the user should feel

> I opened the dashboard, clicked Connect on Gmail, signed in on a popup, closed it. From the next text onward, Boop could read my email when I asked.

## What the system must do

- Render a Connections tab listing curated toolkits (`CURATED_TOOLKITS` in `server/composio.ts`).
- For each toolkit, show: connected / disconnected state, the connected account identity (resolved via the toolkit's "who am I" tool), Connect / Disconnect buttons.
- On Connect, call `POST /composio/toolkits/:slug/authorize` and open the returned `redirectUrl` in a popup.
- After the popup closes, call `POST /composio/refresh` to re-register all live toolkits.
- Make the new slug available to `spawn_agent(integrations: [...])` immediately — no restart, no deploy.
- Surface an amber banner on toolkits that need a one-time custom auth-config setup (e.g., Twitter/X) with a link to `platform.composio.dev/auth-configs`.
- On Disconnect, revoke the Composio connection and re-register, dropping the slug from `availableIntegrations()`.

## What the system must not do

- Do not store OAuth tokens locally. Composio holds them.
- Do not expose the full Composio catalog to the dispatcher — only curated toolkits are user-visible.
- Do not let an execution agent see toolkits the dispatcher didn't pass to it. Per-spawn scoping is non-negotiable. See [integrations-as-mcp.md](../design-docs/integrations-as-mcp.md).
- Do not require a code change to add a toolkit that's already in the curated list — connecting is a UI action.
