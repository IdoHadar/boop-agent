# Spec: memory recall

## What the user should feel

> Boop remembers things I told it weeks ago. It doesn't surface trivia I mentioned once. When I correct it, the correction sticks and the wrong fact is gone, not buried.

## What the system must do

- Run `recall(query)` from the dispatcher's tools to pull the top-relevant active memories for any turn that benefits from context.
- Boost recall scores for memories with `tier: "permanent"` and high `importance`.
- Decay recall weight for memories with low `effectiveScore` so old context fades naturally. See [memory-system.md](../design-docs/memory-system.md).
- Treat user corrections specially: write them with `segment: "correction"`, store the corrected text in `metadata`, and supersede the wrong memory in the same write.
- Log every recall and write to `memoryEvents` so the dashboard's Memory tab can replay them live.

## What the system must not do

- Do not surface memories the user has marked as wrong (corrected memories supersede them).
- Do not include all memories on every turn — recall is scoped, context is scarce.
- Do not write memories that paraphrase the user's last message. Extraction must produce *durable* facts, not transcripts.
- Do not let memory decay alone delete a fact the user explicitly asked the agent to remember — explicit writes can be `permanent`.
