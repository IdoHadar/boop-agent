# Draft safety layer

> Aggressively capable. Never unilaterally committed.

## The rule

Any action that touches the outside world — sending an email, creating a calendar event, posting to Slack, opening a GitHub issue — stages as a `draft` row in Convex before it commits. The dispatcher is the only path to commit.

## How it's structural, not advisory

The execution agent's tool surface includes `save_draft(kind, summary, payload)` and the integration tools themselves (`mcp__gmail__GMAIL_SEND_EMAIL`, etc.). The execution agent's system prompt routes everything through draft.

But the actual structural enforcement is on the dispatcher side:

- `save_draft` writes a `drafts` row with `status: "pending"`.
- The execution agent returns text describing the staged action.
- The dispatcher receives that text and decides whether to call `send_draft(draftId, integrations)`.
- `send_draft` spawns a *new* execution agent with the stored payload as its task. That second spawn is the only path that actually invokes the send tool.

The user is in the loop because the dispatcher's reply is "I drafted this, want me to send it?" — and the dispatcher only calls `send_draft` after an explicit user confirmation arrives.

## Why not block the send tool entirely on the executor

We tried it. Two failure modes:

1. **Blocking the tool meant the agent couldn't *test* what the send would do.** Dry-run modes don't exist for most third-party APIs. Letting the executor see the tool but route through draft is the closest we get.
2. **Some toolkits expose dozens of "send-shaped" tools** with subtle differences. Maintaining an allowlist per toolkit is the kind of thing that drifts. The system-prompt rule plus the dispatcher gate is sturdier.

## What lands as a draft

Anything the user can *undo* by saying "no" or "cancel that." Reads, searches, list operations don't draft — they return inline. Writes, sends, creates, deletes all draft.

## What we ruled out

- **Auto-send after a delay.** Tempting for low-stakes drafts (calendar events). Rejected — surprise sends erode trust. The user's confirmation is the contract.
- **Draft expiration.** We have an `expired` status in the schema but the cleanup loop is intentionally absent. A draft sitting in the dashboard is more useful than a draft that vanished.

## What would change this

- A hands-free deployment (e.g., voice-only) would justify auto-send for explicit-intent commands, with a separate confirmation surface (verbal "yes"). Today's UX assumes the user can read.
