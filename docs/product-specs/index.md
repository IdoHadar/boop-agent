# Product specs

Each spec describes one user-visible surface: what it should feel like, what it must not do, and the concrete acceptance criteria. Specs are stable; they change when the *product* changes, not when the implementation does.

| Spec | Surface |
|---|---|
| [imessage-as-the-interface.md](./imessage-as-the-interface.md) | The texting experience — chunking, latency, tone |
| [automations.md](./automations.md) | Recurring jobs scheduled from inside a conversation |
| [connections.md](./connections.md) | Connecting third-party toolkits via the debug dashboard |
| [drafts-and-confirmation.md](./drafts-and-confirmation.md) | Reviewing and confirming staged actions in iMessage |
| [memory-recall.md](./memory-recall.md) | What "the agent remembers me" actually means at the surface |

## Writing a spec

Three sections, in order:

1. **What the user should feel.** A paragraph in their voice. "I texted Boop and …"
2. **What the system must do.** Bullet points with measurable acceptance criteria.
3. **What the system must not do.** Failure modes we've already chosen to prevent.

Specs are not implementation guides. They tell *what* good looks like, not *how* it's built.
