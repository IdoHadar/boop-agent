# Tech debt tracker

Known shortcuts and the conditions that would justify paying them off. One row per item. Add new rows when you take a shortcut on purpose; remove rows when the underlying assumption stops holding.

| # | Item | Where | Why we accepted it | What would force a fix |
|---|---|---|---|---|
| 1 | In-process automation scheduler | `server/automations.ts` | Single-process, single-tenant deploy. No race. | Deploying multiple instances; need a Convex lock or external scheduler. |
| 2 | No user auth | repo-wide | Single-tenant personal agent | Multi-user deployment; add Clerk or similar. |
| 3 | Embedding column unused by default | `convex/schema.ts:memoryRecords.embedding` | Default Boop ships without semantic recall | >1k active memories where keyword recall stops cutting it. |
| 4 | No structural enforcement of `save_draft` on executor | `server/execution-agent.ts` system prompt | Cheaper than maintaining a per-toolkit allowlist | A regulated deployment where an accidental send is unacceptable. |
| 5 | Curated toolkit list lives in code | `server/composio.ts:CURATED_TOOLKITS` | Editing an array is fine for a small fork | A multi-tenant deploy where each user picks their own visible set. |
| 6 | Memory consolidation runs over the whole active set | `server/consolidation.ts` | <500 active memories in practice | >1k active rows; chunk + parallelize. |
| 7 | Heartbeat hard-codes 15-min timeout | `server/heartbeat.ts` | Matches our longest known happy-path run | Tasks that legitimately run >15 min (e.g. deep-research toolkits). |
| 8 | Sendblue dedup TTL is implicit (table never trimmed) | `convex/sendblueDedup.ts` | Low row count for a single user | A high-volume deployment; add a TTL cleanup. |

## How to use this file

When you're tempted to add a "TODO" comment in code, instead add a row here. The TODO comment will rot in place; the row here is reviewed every time someone reads the doc tree. Link the row back from the code with a comment that just says `// see exec-plans/tech-debt-tracker.md #N` if the explanation is needed at the call site.
