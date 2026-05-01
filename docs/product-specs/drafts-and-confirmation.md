# Spec: drafts and confirmation

## What the user should feel

> I asked Boop to email Sarah. It replied with the draft text. I said "send it." It sent. If I'd said "no, change the subject," it would have rewritten and asked again.

## What the system must do

- Stage every external action (send email, create event, post message, etc.) as a `drafts` row with `status: "pending"` before committing.
- Surface the draft contents in the user's iMessage reply: the kind of action, a one-line summary, and the key fields (recipient, subject, time, body).
- Wait for an explicit user confirmation in the same conversation before committing.
- On confirm, spawn a fresh execution agent with the stored payload as its task — that's the only path that calls the real send tool.
- On reject, mark the draft `rejected` and acknowledge in chat.
- Show every draft (pending, sent, rejected) in the dashboard's Drafts tab with its raw payload for inspection.

## What the system must not do

- Do not auto-send drafts after a delay.
- Do not mark a draft `sent` until the actual send tool returns success.
- Do not let the executor reach a real send tool directly — the dispatcher's `send_draft` is the only commit path.
- Do not accept ambiguous confirmations ("ok") to send something destructive without echoing back what's about to commit.
