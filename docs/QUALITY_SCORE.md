# QUALITY SCORE

A snapshot of where the codebase is healthy and where it's leaning. Updated whenever a domain changes shape — not on a fixed cadence.

Scores are coarse on purpose: a single letter per domain. **A** = clean and bounded. **B** = working but with known gaps. **C** = correct but bloated, brittle, or hard to extend. **D** = active liability.

## Scores

| Domain | Score | Reason |
|---|---|---|
| Dispatcher (interaction agent) | A | Tight toolset, stable system prompt, predictable cost. The principle is enforced structurally. |
| Executor (execution agent) | A | Ephemeral by design, audited end-to-end, scoped tools. Drafts route correctly. |
| Memory — extract | B | Fire-and-forget works, but extraction quality varies by model and by turn shape. No regression eval yet. |
| Memory — decay/clean | A | Simple, deterministic, easy to reason about. Tunables live in `types.ts`. |
| Memory — consolidation | B | Three-role pipeline works, but a >1k active-memory deployment would need chunking the proposer pass. |
| Drafts | A | Structural. The two-spawn commit path is the cleanest part of the codebase. |
| Automations | C | Single-process scheduler, no distributed lock, hard-coded poll interval, mixed concerns (scheduling + result routing + notification). The lowest-cohesion module by [graphify](../graphify-out/). |
| Composio integrations | B | Per-spawn scoping is correct. The curated list lives in code (acceptable for a fork-template). |
| Heartbeat / lifecycle | B | Catches stuck agents but timeout is hard-coded at 15 min. Long-running tasks (deep-research toolkits) hit the cap. |
| Convex schema | A | Tight, indexed, validators present. `metadata` JSON sidecars are intentional, not slop. |
| Debug dashboard | B | Read paths are great. The few write surfaces (`cancel`, `retry`, `disconnect`) lack a confirmation step. |
| Sendblue webhook | A | Dedup works, chunker works, markdown stripped. Boring in the good way. |
| Setup script | B | Long but linear. Survives a clean install on macOS; less battle-tested elsewhere. |
| Tests | D | None to speak of. The audit trail is the de facto test surface. This is the single largest gap in the codebase. |

## What "moving the score" looks like

- **Tests D → C.** Even one harness that replays a recorded conversation through the dispatcher and asserts on the spawn shape would lift the score. Coverage isn't the goal — confidence under refactors is.
- **Automations C → B.** Pull scheduling into one file, result routing into another, notification into a third. Three small files, one concern each.
- **Memory — extract B → A.** A regression eval over a fixed transcript set with snapshot diffs of extracted facts.
- **Heartbeat B → A.** Make the timeout configurable per-spawn, defaulted from the integration set.

## What we deliberately leave at B

Boop is a template. A `B` for "single-process scheduler" is correct for a single-user fork. Promoting it to `A` would mean infrastructure that doesn't matter for the default user.

The point of the score is not to climb to all-A — it's to keep us honest about where the seams are.
