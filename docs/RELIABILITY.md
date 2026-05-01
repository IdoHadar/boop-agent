# RELIABILITY

What can fail, what we catch, and what we let fail loudly on purpose.

## The failure surfaces

| Surface | Failure mode | Mitigation |
|---|---|---|
| Sendblue inbound | Webhook retries on the same `message_handle` | `sendblueDedup` table claims handles atomically; duplicate deliveries no-op |
| Sendblue outbound | Rate limit, transient 5xx | Single retry on 5xx, log + drop on rate limit (rare for personal-agent volume) |
| Execution agent | Stuck > 15 min | `server/heartbeat.ts` polls every 60s, marks `failed`, fires `AbortController` if still in-process |
| Execution agent | Server restart while running | Heartbeat catches the orphan on next tick (controller is gone but DB still says `running`) |
| Composio session | Toolkit becomes unauthenticated mid-task | The tool call returns an auth-error string; the agent surfaces it back through the draft path or as a final message |
| Convex mutation | Network blip | Convex's client retries with idempotency keys; mutations are written to be idempotent |
| Automation tick | One tick takes longer than the 30s poll interval | Ticks are spawned in parallel; a slow run can't block the next due check |
| Automation scheduler | Multi-instance double-fire | **Not mitigated.** Single-process deploy is required. See `tech-debt-tracker.md` #1. |

## What we fail loud on

- **Unknown integration slug.** The dispatcher logs `[integrations] unknown integration: …` and continues without it. The user sees a degraded reply, not a silent skip.
- **Unauth'd Composio session.** Same. The agent reports it can't reach Gmail/Slack/etc., the user reconnects in the dashboard.
- **Schema drift.** Convex rejects mismatched mutations at the type boundary. We don't catch that — we let it crash and read the stack trace.
- **Memory consolidation parse error.** A bad model output fails the run, which is logged in `consolidationRuns.notes`. We don't auto-retry; we surface it.

## What we monitor

- **The dashboard's Dashboard tab.** Cumulative spend, live agent statuses, the count of `failed`/`stuck` agents in the last 24h.
- **The Agents tab.** Per-agent token usage, duration, and integration logos. Outliers are where to look first.
- **The Drafts tab.** A pending draft older than the conversation it came from is a sign the user got distracted — not a system failure, but worth noticing.
- **The Memory tab.** Sudden cluster collapse after a consolidation run is a hint that the proposer was overconfident.

## What we don't have (and why)

- **No paging.** This is a personal agent. The user is the operator. iMessage is the alert channel.
- **No SLOs.** Latency budgets live in the spec (`product-specs/imessage-as-the-interface.md`); we don't track 99th percentiles.
- **No structured error taxonomy.** Errors flow through Convex `agentLogs` as `logType: "error"` with free-text content. A taxonomy would be premature for the volume.
- **No multi-region.** Single deploy.

## The single largest reliability bet

The dispatcher is the only thing between the user and the world. Its system prompt, its tool surface, and the dispatcher/executor split are the entire safety story. If the dispatcher misbehaves, drafts can't save us — the dispatcher is what calls `send_draft`.

Changes to the dispatcher's prompt or tools should be small, reviewed against the four core beliefs, and accompanied by a recorded-conversation regression check before they merge.
