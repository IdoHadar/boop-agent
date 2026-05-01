# Memory system

> Memory must forget on purpose. The point is not retention — it's relevance.

## Three layers

**Extraction** (`server/memory/extract.ts`) runs *fire-and-forget* after every turn. A small Haiku/Sonnet pass reads `(userMessage, assistantReply)` and emits durable facts as discrete `memoryRecord` rows. The model is told to prefer fewer, higher-quality facts over many trivial ones.

**Decay + cleanup** (`server/memory/clean.ts`) runs every six hours by default. For each active memory:

```
effectiveScore   = importance × decay × reinforcement

adaptiveHalfLife = BASE_HALF_LIFE_DAYS (11.25) × (1 + importance)
lambda           = (ln 2 / adaptiveHalfLife) × DECAY_BETA (0.8) × (1 + decayRate)
decay            = exp(-lambda × daysSinceAccess)
reinforcement    = 1 + log(1 + accessCount) × 0.1
```

The half-life *scales with importance* so identity and correction memories persist much longer than context. Per-segment defaults set the baseline:

| Segment | Tier | importance | decayRate |
|---|---|---|---|
| `identity` | `permanent` | 0.85 | 0.01 |
| `correction` | `long` | 0.80 | 0.015 |
| `relationship` | `long` | 0.75 | 0.02 |
| `preference` | `long` | 0.70 | 0.02 |
| `project` | `long` | 0.65 | 0.025 |
| `knowledge` | `long` | 0.60 | 0.03 |
| `context` | `short` | 0.40 | 0.08 |

Effective score below `0.15` → archived (skipped for `long`-tier rows so they only ever prune, never archive). Below `0.05` → pruned. `permanent` rows are skipped entirely.

**Consolidation** (`server/consolidation.ts`) runs daily or on demand. Three-agent pipeline — see [consolidation-pipeline.md](./consolidation-pipeline.md).

## Segments

Every memory is filed under one segment so retrieval can scope by intent:

| Segment | Use |
|---|---|
| `identity` | Who the user is — name, role, preferences about themselves |
| `preference` | What the user likes / how they want things done |
| `correction` | Something the user said the agent got wrong (carries `corrects` in `metadata`) |
| `relationship` | Other people the user mentions and their context |
| `project` | Active work, deadlines, status |
| `knowledge` | Domain facts the user has taught the agent |
| `context` | Short-lived situational state |

## What we stripped out

The original Boop prototype had:
- Embedding-based clustering with a graph database
- Multi-pass consolidation with knowledge-graph edges
- Implicit memory writes from arbitrary tool calls
- A skills library that wrote memories itself

All of it shipped, none of it earned its complexity for a single-tenant personal agent. The hooks remain in the Convex schema (`vectorIndex("by_embedding")`, `supersedes` array) so a fork can add them back. Default Boop runs without any of it.

## Why explicit writes

Writing is either:
- **Explicit** — the dispatcher calls `write_memory(content, segment, importance, tier)` because the user just told it something durable.
- **Inferred** — `extract.ts` runs after every turn and proposes facts. These are still discrete writes with provenance, not implicit side effects of tool use.

The rule: nothing implicit. A memory should always trace back to either a deliberate model decision or a labeled extraction pass — never to "we noticed the user said this."

## What would change this

- A multi-user deployment would force a per-user keying layer — currently `memoryRecords` are global.
- Heavier retrieval needs (semantic search across thousands of memories) would justify turning the embedding column on by default.
