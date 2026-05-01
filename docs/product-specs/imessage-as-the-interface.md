# Spec: iMessage as the interface

## What the user should feel

> I texted Boop. It replied like a person. I didn't open an app, didn't log in, didn't navigate a menu. When I asked for something hard ("research X for me"), it acknowledged fast and came back with the result a few minutes later. When I asked for something casual, the reply was instant.

## What the system must do

- Accept inbound iMessage via Sendblue's webhook (`POST /sendblue/webhook`).
- Dedup duplicate webhook deliveries by `message_handle` so retries don't double-process.
- Respond to casual turns (greetings, recalls, confirmations) in **2–3 seconds end-to-end**.
- Respond to "real work" turns with an **acknowledgment in 2–3 seconds** and a final result whenever the work completes.
- Strip markdown from outbound text — iMessage doesn't render it.
- Chunk replies longer than 2900 chars into multiple messages so Sendblue accepts them.
- Show typing or a brief acknowledgment for any turn that's going to take longer than ~5 seconds (e.g. "On it — searching now").
- Persist every inbound and outbound message to `messages` in Convex with `role`, `conversationId`, and `turnId`.

## What the system must not do

- Do not require an app, login, or browser.
- Do not send markdown formatting characters (`**`, `_`, `#`, etc.) verbatim into iMessage.
- Do not send a single message longer than the carrier limit.
- Do not commit external actions (send email, post Slack, etc.) without an explicit user confirmation in the conversation. See [drafts-and-confirmation.md](./drafts-and-confirmation.md).
- Do not surface model-internal artifacts (tool names, JSON, error stack traces) in the user-facing reply. Those go to the dashboard.
