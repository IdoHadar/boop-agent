# Consolidation pipeline

> Three roles, two cheap, one careful. The adversary catches what the proposer wants to ignore.

## The pipeline

```
Proposer (Sonnet)  →  Adversary (Haiku)  →  Judge (Sonnet)
   find merges,         challenge each       approve / reject
   supersedes,          proposal: what       with rationale
   prunes               did we miss?
```

All three roles run over the active memory set. Output is a list of decisions applied in a single transaction.

## Roles

**Proposer.** Sees the active memory list. Emits proposals:

- `merge` — combine N entries into one rewrite
- `supersede` — newer memory replaces older on a conflicting value
- `prune` — remove redundant or wrong entries

**Adversary.** Sees the proposals. Cheap, skeptical. Its job is *not* deep reasoning — it's to surface "these two are *similar* but not *identical*" cases the proposer would otherwise collapse. We use Haiku here on purpose. Cheap skepticism is exactly the right amount.

**Judge.** Sees both proposals and challenges. Emits `approve` or `reject` per proposal with a written rationale.

## Why three

Two would not have caught the failure mode that motivated this pipeline: an early single-pass consolidation made overconfident merges. Two memories that disagreed slightly on a fact got fused into a confident-sounding wrong memory. Adding a separate adversary, structurally biased toward "no, these are different," caught the case. The judge then weighs both with full context.

This is cheaper than it sounds. Adversary is the cheapest model. The judge sees a concise summary, not the full memory list. Proposals are typically <30 per run.

## Application

Approved decisions are applied to `memoryRecords`:

- **merge** — write a new memory with `supersedes: [...oldIds]`. The upsert mutation archives the superseded rows automatically.
- **supersede** — same as merge with one source.
- **prune** — set `lifecycle: "archived"` (or `"pruned"` if scored low enough).

The full run — proposals, challenges, decisions — is captured in `consolidationRuns.details` as JSON so any past run can be inspected later in the dashboard.

## What we ruled out

- **Single-pass consolidation.** Made overconfident merges. See above.
- **Always run on every turn.** Cost adds up; consolidation is a slow signal, not a real-time one. Daily is enough.
- **A fourth "rewriter" role.** The judge produces the final text directly. A separate rewriter doubled cost without measurable improvement.

## What would change this

- A larger memory set (>1000 active rows) might justify chunking the proposer pass and parallelizing adversary calls per chunk.
- A regulated deployment might need the judge's rationale to be human-readable and persisted forever — currently we keep it but archive aggressively.
