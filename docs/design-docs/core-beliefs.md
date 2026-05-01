# Core beliefs

The four principles every other design choice in Boop inherits from. If a change violates one of these, it needs an explicit override in its own design doc — not a quiet exception.

## 1. The interface is iMessage. The interface is the constraint.

Boop is not "an agent app." It's an agent you text. That single decision rules out almost every other interface and forces every other architectural decision: replies must feel instant on casual turns, must chunk to 2900 chars, must strip markdown, must work without an app open. Anything that requires the user to switch context out of iMessage is a defeat.

## 2. Most turns shouldn't spawn anything.

A casual "what did we talk about yesterday?" must complete in one cheap dispatcher call. Spawning an execution agent, opening a Composio session, loading toolkit context — those are real cost. The dispatcher's job is to *not* spawn unless the user asked for real work. Default-spawn is the failure mode we structurally eliminated; see [dispatcher-executor.md](./dispatcher-executor.md).

## 3. The dispatcher decides. The executor commits. Never the same agent.

The interaction agent has tiny tools and cannot reach the outside world. The execution agent has heavy tools but cannot decide *what* to do — it gets a task from the dispatcher and runs it. Any external action stages as a draft and only commits when the dispatcher confirms. This is enforced by tool surface, not by prompt. See [draft-safety.md](./draft-safety.md).

## 4. Memory must forget on purpose.

Conversations accumulate noise. An agent that remembers everything equally is worse than one with no memory at all — it surfaces stale facts and buries fresh ones. Every memory has a tier, a decay rate, and a score. Below threshold it's archived; further below it's pruned. The point is not retention, it's relevance. See [memory-system.md](./memory-system.md).

---

## What follows from these

- **Boring tech where it costs nothing.** Convex for state, Express for HTTP, Composio for the 1000-toolkit problem. We don't reinvent infrastructure that doesn't differentiate Boop.
- **Single-tenant by default.** Multi-tenancy is a one-file change but a different threat model — we don't pre-pay for it.
- **Repository as the system of record.** If it isn't in the repo (code, design docs, prompts), an agent working on Boop can't see it. Slack threads and meeting notes don't count.
- **Small surface, sharp edges.** The codebase should fit in one head. We strip features rather than add abstractions to manage them.
