# Database schema (generated)

> Generated from `convex/schema.ts`. Do not edit by hand. Re-run the doc-gardening script after schema changes.

Convex tables. Indexes elided where they're not load-bearing for navigation. See `convex/schema.ts` for the canonical shape, including index names and validators.

## `messages`
Per-turn iMessage and chat transcript.
- `conversationId`, `role` (`user` | `assistant` | `system`), `content`
- `agentId?`, `turnId?`, `createdAt`
- Indexes: `by_conversation`, `by_conversation_turn`

## `conversations`
Per-thread metadata.
- `conversationId`, `title?`, `summary?`, `messageCount`, `lastActivityAt`

## `memoryRecords`
The memory store.
- `memoryId`, `content`, `tier` (`short` | `long` | `permanent`)
- `segment` (`identity` | `preference` | `correction` | `relationship` | `project` | `knowledge` | `context`)
- `importance`, `decayRate`, `accessCount`, `lastAccessedAt`
- `lifecycle` (`active` | `archived` | `pruned`), `supersedes?`
- `embedding?` (1024-d float64), `metadata?` (loose JSON sidecar), `createdAt`
- Vector index: `by_embedding`, filtered by `lifecycle`

## `executionAgents`
One row per spawned execution agent.
- `agentId`, `conversationId?`, `name`, `task`
- `status` (`spawned` | `running` | `completed` | `failed` | `cancelled` | `paused`)
- `result?`, `error?`, `mcpServers[]`
- Token + cost fields: `inputTokens`, `outputTokens`, `cacheReadTokens?`, `cacheCreationTokens?`, `costUsd`
- `startedAt`, `completedAt?`

## `usageRecords`
Append-only LLM usage log. Every model call writes here.
- `source` (`dispatcher` | `execution` | `extract` | `consolidation-proposer` | `consolidation-adversary` | `consolidation-judge` | `proactive`)
- `conversationId?`, `turnId?`, `agentId?`, `runId?`
- `model`, token + cost fields, `durationMs`, `createdAt`

## `agentLogs`
Per-agent audit trail.
- `agentId`, `logType` (`thinking` | `tool_use` | `tool_result` | `text` | `error`)
- `toolName?`, `accounts?` (Composio account aliases for multi-account toolkits)
- `content`, `createdAt`

## `memoryEvents`
Append-only event log feeding the dashboard's Memory tab.
- `eventType`, `conversationId?`, `memoryId?`, `agentId?`, `data`, `createdAt`

## `automations`
Scheduled recurring tasks.
- `automationId`, `name`, `task`, `integrations[]`
- `schedule` (5-field cron), `timezone?` (IANA, captured at create time)
- `enabled`, `conversationId?`, `notifyConversationId?`
- `lastRunAt?`, `nextRunAt?`, `createdAt`

## `automationRuns`
One row per automation run.
- `runId`, `automationId`, `status`, `result?`, `error?`, `agentId?`
- `startedAt`, `completedAt?`

## `drafts`
Staged external actions awaiting confirmation.
- `draftId`, `conversationId`, `kind`, `summary`, `payload` (JSON)
- `status` (`pending` | `sent` | `rejected` | `expired`)
- `createdAt`, `decidedAt?`

## `consolidationRuns`
History of memory consolidation passes.
- `runId`, `trigger`, `status`
- `proposalsCount`, `mergedCount`, `prunedCount`
- `notes?`, `details?` (JSON: proposals, decisions, applied)
- `startedAt`, `completedAt?`

## `sendblueDedup`
Webhook dedup by `message_handle`.
- `handle`, `claimedAt`

## `settings`
Runtime overrides (e.g. model selection from inside a conversation).
- `key`, `value`, `updatedAt`
