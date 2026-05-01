# Architecture

boop-agent is a small distributed system disguised as a single-server app. Four moving parts, each doing one job.

## The four parts

```
┌────────────────────────────────────────────────────────────────┐
│                      EXPRESS + WS SERVER                        │
│                                                                 │
│   POST /sendblue/webhook   ──────►  Interaction Agent           │
│   POST /chat                        (dispatcher, streams)       │
│   WS /ws                                  │                     │
│                                           │ spawn_agent         │
│                                           ▼                     │
│                                    Execution Agent(s)           │
│                                    (one per task)               │
│                                           │                     │
│                                           ▼                     │
│                                    Integrations (MCP)           │
└────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                       ┌────────────┐         ┌────────────────┐
                       │  Convex    │◄───────►│  Debug UI      │
                       │  (truth)   │         │  (read-only)   │
                       └────────────┘         └────────────────┘
```

### 1. Interaction agent — `server/interaction-agent.ts`

The front door. One instance per user turn. Its job is to **decide**, not to do.

- Reads the user's message + last 10 turns from Convex.
- Owns six MCP servers, all in-process (`createSdkMcpServer`):
  - `boop-memory` — `recall(query)`, `write_memory(content, segment, importance, tier?, supersedes?)`.
  - `boop-spawn` — `spawn_agent(task, integrations[], name?)`.
  - `boop-automations` — `create_automation`, `list_automations`, `toggle_automation`, `delete_automation`.
  - `boop-draft-decisions` — `list_drafts`, `send_draft`, `reject_draft`.
  - `boop-ack` — `send_ack` (sends a one-liner to iMessage so the user isn't staring at silence while a sub-agent works).
  - `boop-self` — `get_config`, `set_model`, `set_timezone`, `list_integrations`, `search_composio_catalog`, `inspect_toolkit`.
- The SDK's built-in tools (`WebSearch`, `WebFetch`, `Bash`, file tools, `Agent`, `Skill`) are explicitly disallowed on the dispatcher so it MUST spawn a sub-agent for external work.
- Its system prompt drills the DISPATCHER rule: answer directly for chit-chat, spawn an agent for real work.
- Replies stream through Sendblue back to iMessage (markdown stripped, chunked to 2900 chars).

### 2. Execution agent — `server/execution-agent.ts`

Spawned per task. Ephemeral. One instance, one job, one result.

- Gets the specific `task` the interaction agent wrote (not the raw user message).
- Loads **only** the integrations named in the spawn call.
- System prompt drills: iMessage-friendly output, draft-before-send for any external action.
- Logs every `tool_use`, `tool_result`, and text block to Convex so the debug dashboard can replay it.
- Runs with `permissionMode: bypassPermissions` — the interaction agent is the gatekeeper.
- Returns a string. That string becomes a tool-result back to the interaction agent, which rewrites it in its own voice.

### 3. Memory — `server/memory/`

Four files, four jobs.

**`types.ts`** — shape + defaults.
- Tiers: `short`, `long`, `permanent`. Tier governs lifecycle thresholds, not the decay rate directly.
- Segments: `identity`, `preference`, `correction`, `relationship`, `project`, `knowledge`, `context`.
- Per-segment defaults (`SEGMENT_DEFAULTS`) supply tier, baseline importance, and `decayRate` when a memory is written. Identity is permanent (importance 0.85, decayRate 0.01); context is short (importance 0.40, decayRate 0.08); the rest sit in `long` with rates between 0.015 and 0.03.

**`tools.ts`** — the `boop-memory` MCP server. `recall` and `write_memory`. Recall does vector search via `convex/memoryRecords.vectorSearch` when an embedding is available, then falls back to substring search. Each call emits a `memoryEvents` row so you can watch it live in the dashboard.

**`extract.ts`** — fires post-turn, **fire-and-forget**. Sends `(userMsg, assistantReply)` to a Haiku/Sonnet pass with an extraction prompt, parses JSON facts, writes each one. The model is told to prefer fewer, higher-quality facts over many trivial ones. Skipped on `kind: "proactive"` turns so email-derived content doesn't pollute user-fact memory.

**`clean.ts`** — the memory-cleaning loop. Every 6 hours (configurable):

1. Load active memories.
2. Compute an effective score `importance × decay × reinforcement` using Ping's adaptive exponential decay:
   - `adaptiveHalfLife = BASE_HALF_LIFE_DAYS (11.25) × (1 + importance)` — important memories live longer.
   - `lambda = (ln 2 / adaptiveHalfLife) × DECAY_BETA (0.8) × (1 + decayRate)` — per-segment `decayRate` shortens half-life on top.
   - `decay = exp(-lambda × daysSinceAccess)`.
   - `reinforcement = 1 + log(1 + accessCount) × 0.1` — frequently-recalled rows resist decay.
3. Below threshold `0.15` → archive (skipped for `long`-tier rows). Below `0.05` → prune. Permanent memories are skipped entirely.

This is deliberately simple. Everything sophisticated (consolidation, adversary/judge debates, knowledge graphs, embeddings) was stripped out. Add them back if you need them — the hooks are already in the Convex schema.

### 4. Automations — `server/automations.ts` + `server/automation-tools.ts`

The agent can schedule recurring work from any conversation. When the user says *"every morning at 8 summarize my calendar"*, the interaction agent calls `create_automation(name, cronExpr, task, integrations)`.

How it runs:
- **`server/automations.ts`** starts a 30-second poll (`startAutomationLoop`) when the server boots.
- On each tick it loads enabled automations from Convex, finds ones whose `nextRunAt` is ≤ now, and fires each one in parallel.
- Firing = `spawnExecutionAgent({ task, integrations, conversationId, name: "auto:..." })` — the same sub-agent system the interaction agent uses.
- The result is written as an `automationRun` row, and (if `notifyConversationId` points at an `sms:+...` conversation) pushed back out via Sendblue so the user sees it in iMessage.
- `nextRunAt` is recomputed with `croner` and stored.

The four MCP tools exposed to the interaction agent (`server/automation-tools.ts`):
- `create_automation(name, schedule, task, integrations, notify?)`
- `list_automations(enabledOnly?)`
- `toggle_automation(id, enabled)`
- `delete_automation(id)`

Schedule is a standard 5-field cron expression. Croner also understands extended syntax (timezones, seconds) if you want to upgrade the tool description.

### 5. Drafts — `server/draft-tools.ts`

Any external action (send email, create event, post Slack message) is staged, not committed, by the execution agent.

- Execution agents only have `save_draft(kind, summary, payload)`. The "real" send tools exist in each integration but the system prompt routes agents through `save_draft` first.
- The interaction agent has `list_drafts`, `send_draft(draftId, integrations)`, `reject_draft(draftId)`.
- `send_draft` spawns a new execution agent with the stored payload as its task. This is the only path to actually committing an action.

You can see every draft (pending, sent, rejected) in the Drafts tab of the debug dashboard, including the raw JSON payload.

### 6. Heartbeat + lifecycle — `server/heartbeat.ts`

Every 60 seconds, scan `executionAgents` with status `running`. Any whose `startedAt` is older than 15 minutes gets marked `failed` and the in-process `AbortController` is triggered if it still exists. This handles both server restarts (controller gone, DB still "running") and genuinely stuck agents.

HTTP routes for the debug dashboard:
- `POST /agents/:id/cancel` — abort an in-flight agent
- `POST /agents/:id/retry` — re-spawn an agent with the same task + integrations

### 7. Consolidation — `server/consolidation.ts`

Runs daily (or on-demand via `POST /consolidate`). A three-agent pipeline over the active memory set (skipped if fewer than 6 active memories):

1. **Proposer** (Sonnet by default) receives the full memory list and returns proposals:
   - `merge` — combine several entries into one rewrite
   - `supersede` — newer memory replaces older on a conflicting value
   - `prune` — remove redundant or wrong entries
2. **Adversary** (Haiku by default — cheap skepticism) sees the proposals and emits per-proposal challenges with a `low | medium | high` severity.
3. **Judge** (Sonnet) sees both proposals and challenges and approves or rejects each one with a rationale.
4. Approved proposals are applied via `supersedes` on `memoryRecords` (which archives the superseded memories automatically in the upsert mutation).

Models are overridable via `BOOP_MODEL` and `BOOP_ADVERSARY_MODEL`. The full run — proposals, challenges, decisions, applied actions, and a snapshot of every referenced memory's content — is captured in `consolidationRuns.details` and broadcast progressively (`proposed → challenged → judged → applied`) so the debug UI can stream the pipeline live.

### 8. Integrations — Composio (`server/composio.ts`)

Boop delegates all third-party integrations to [Composio](https://composio.dev/?utm_source=chris&utm_medium=youtube&utm_campaign=collab). One SDK, 1000+ toolkits, hosted auth.

Flow:
1. User clicks **Connect** on a toolkit card in the debug dashboard's Connections tab.
2. Frontend → `POST /composio/toolkits/:slug/authorize` → `authorizeToolkit(slug)` looks up an existing auth config for the toolkit (or creates a Composio-managed one), then calls `composio.connectedAccounts.initiate(boopUserId(), authConfigId, ...)` and returns its hosted `redirectUrl`. Toolkits without a managed OAuth app surface a 409 with `needsAuthConfig: true` so the UI can prompt the user to register their own at `dashboard.composio.dev`.
3. Popup opens the redirect URL. User authenticates. Composio stores the tokens on its side.
4. Popup closes → frontend calls `POST /composio/refresh` → backend re-runs the registry loader which iterates `connectedAccounts.list({ userIds: [boopUserId()] })` and registers each active toolkit as an `IntegrationModule` keyed by its slug.
5. `availableIntegrations()` now includes the new slug, so the dispatcher can spawn a sub-agent with it.

On each spawn, `buildComposioIntegrationModule(slug).createServer()` opens a **fresh toolkit-scoped Composio session**:

```ts
await composio.create(boopUserId(), {
  toolkits: [slug],            // scope — sub-agent only sees this toolkit's tools
  manageConnections: false,    // don't inject auth-management meta-tools
  authConfigs: { [slug]: authConfig.id },             // when present (avoids auto-create 400s for BYO toolkits)
  multiAccount: { enable: true, requireExplicitSelection: true }, // only when ≥2 active connections for this toolkit
});
```

and returns an `McpSdkServerConfigWithInstance` via `createSdkMcpServer`. The sub-agent never sees the full Composio catalog — only the tools for the toolkits the dispatcher asked for.

HTTP routes (`server/composio-routes.ts`, mounted at `/composio`):
- `GET  /status` — `{ enabled }`.
- `GET  /toolkits` — curated list merged with current connection state.
- `GET  /toolkits/:slug/tools` — list of tools the toolkit exposes (cached 10 min).
- `POST /toolkits/:slug/authorize` — returns `{ redirectUrl, connectionId }`. Returns 409 with `needsAuthConfig: true` for toolkits without managed OAuth.
- `POST /toolkits/:slug/disconnect` — revokes + refreshes registry.
- `POST /connections/:id/rename` — set a Composio alias on a connection.
- `POST /refresh` — re-runs the registry loader.
- `POST /webhook` — Composio webhook receiver (HMAC-verified) for proactive triggers like inbound Gmail.

Env:
- `COMPOSIO_API_KEY` — required for integrations. Without it, plain chat + memory + automations still work.
- `COMPOSIO_USER_ID` — optional; defaults to `boop-default` for single-tenant use.

---

## Data model (Convex)

Thirteen tables. Read `convex/schema.ts` for the exact shape.

| Table | Role | Key fields |
|---|---|---|
| `messages` | iMessage + chat transcript | conversationId, role, content, turnId |
| `conversations` | Per-thread metadata | conversationId, messageCount, lastActivityAt |
| `memoryRecords` | The memory store | memoryId, content, tier, segment, importance, decayRate, accessCount, lifecycle, supersedes, embedding |
| `executionAgents` | One row per spawned agent | agentId, task, status, tokens, cost |
| `usageRecords` | Append-only LLM cost log (every model call) | source, model, tokens, costUsd, durationMs |
| `agentLogs` | Per-agent audit trail | agentId, logType, toolName, accounts, content |
| `automations` | Scheduled recurring tasks | automationId, schedule, timezone, task, integrations, enabled, nextRunAt |
| `automationRuns` | One row per automation run | runId, automationId, status, result, agentId |
| `drafts` | Staged external actions | draftId, kind, summary, payload, status |
| `consolidationRuns` | History of consolidation passes | runId, proposalsCount, mergedCount, prunedCount, details |
| `sendblueDedup` | Webhook dedup by `message_handle` | handle, claimedAt |
| `memoryEvents` | Append-only event log for the debug UI | eventType, conversationId, memoryId, data |
| `settings` | Runtime overrides (model, timezone, proactive toggle, webhook secret) read by `server/runtime-config.ts` | key, value, updatedAt |

`memoryRecords` also carries a `vectorIndex("by_embedding")` with 1024-dimension vectors filtered by `lifecycle`.

Indexes are tight — search through the schema to see what's supported.

---

## Message lifecycle

Following a text from iMessage to reply, step by step:

```
1.  Sendblue POST /sendblue/webhook
2.  sendblue.ts:  dedup + spawn handleUserMessage()
3.  interaction-agent:  save user msg, fetch recent history
4.  interaction-agent:  query Claude with memory + spawn tools
     ↳ may call recall / write_memory
     ↳ may call spawn_agent → execution-agent runs, returns text
5.  interaction-agent:  final text → broadcast + return
6.  sendblue.ts:  sendImessage() chunks + sends
7.  interaction-agent:  save assistant msg to Convex
8.  BACKGROUND: extract.ts pulls durable facts, writes memories
9.  LATER: clean.ts decays scores, archives or prunes
```

Steps 6–7 run in parallel where safe. Step 8 is fire-and-forget — the user never waits on extraction.

---

## Why this shape

**Dispatcher / executor split.** The interaction agent has a tiny toolset and a short prompt so it's cheap, fast, and deterministic. The execution agent gets heavy tools (MCPs) but only runs when needed. Most casual turns never spawn an agent — they complete in one interaction-agent call.

**Memory lives next to execution, not in the model.** Claude has no memory across turns. We re-hydrate the relevant slice every turn via `recall()`. Writing is explicit (`write_memory`) or inferred (`extract.ts`). Nothing is implicit.

**Integrations via Composio.** Tool-calling is what the SDK does best. Composio handles the OAuth, token-refresh, and 1000+ service adapters we'd otherwise hand-roll. Each connected toolkit becomes an MCP server on demand, scoped to just that toolkit so the sub-agent's context stays small.

**Convex for state.** Reactive queries power the debug UI without polling. Durable enough for real use, free tier generous enough for a personal agent.

---

## What's intentionally missing

- **No user auth.** This is a single-user tool. Add Clerk or similar if you want multi-tenant.
- **Single-process scheduler.** The automation loop runs in-process. If you deploy multiple instances, you'll double-fire — add a lock in Convex or run a dedicated scheduler pod.
- **Limited proactive surface.** Only inbound Gmail is watched today (`server/proactive-email.ts` + `POST /composio/webhook`). The classifier filters self-sends and low-priority noise, then routes survivors into the dispatcher as a `kind: "proactive"` turn. Other proactive sources (calendar, Slack, etc.) are deliberately not built — add them by registering more Composio triggers.
- **No knowledge graph** — relationships between memories are represented via `supersedes` only, not a full graph.
- **Skills library omitted** — too Boop-specific; write your own prompts/policies in `server/*-agent.ts` system prompts.

All of these are one-file additions. The point of the template is to give you the smallest surface that still actually works.
