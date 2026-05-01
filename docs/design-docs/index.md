# Design docs

Decision records for Boop's architecture. One doc per significant design choice. Each one answers three things:

- **What** is the rule or shape?
- **Why** did we land here?
- **What did we rule out**, and what would change it?

These are the "why" of the system. The "what" lives in `ARCHITECTURE.md`. The "how to use it" lives in the README.

| Doc | Subject |
|---|---|
| [core-beliefs.md](./core-beliefs.md) | The four operating principles every other doc inherits from |
| [dispatcher-executor.md](./dispatcher-executor.md) | Why we split the agent in two and what each side may do |
| [memory-system.md](./memory-system.md) | Tiers, decay, reinforcement, and what we deliberately stripped out |
| [consolidation-pipeline.md](./consolidation-pipeline.md) | The proposer / adversary / judge loop and why three roles |
| [draft-safety.md](./draft-safety.md) | The draft staging layer and why it's structural, not advisory |
| [integrations-as-mcp.md](./integrations-as-mcp.md) | Per-spawn toolkit-scoped MCP servers, and why not "give the agent everything" |

## When to add a new design doc

Add one when a change would make a future reader (human or agent) ask "why did they do it this way?" and the answer isn't obvious from the code. Don't add one for routine implementation choices — those belong in commit messages.

A good design doc fits in one screen. If it wants to be longer, it's probably two docs.
