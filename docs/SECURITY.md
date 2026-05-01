# SECURITY

Boop is a single-tenant personal agent. The threat model is "the user's data lives in this deployment, and the agent shouldn't leak it or commit unauthorized actions on the user's behalf."

## The two trust boundaries

**Inbound: Sendblue → Boop.** Anyone who can text the configured Sendblue number can talk to the agent. There is no per-message authentication beyond the phone-number routing Sendblue itself does. If the deployment serves more than one user's phone number, the conversation routing is the auth.

**Outbound: Boop → third-party services.** Composio holds OAuth tokens; Boop never sees them. Every external action flows through the draft layer, so the user is in the loop on every commit. See [`design-docs/draft-safety.md`](./design-docs/draft-safety.md).

## Secrets

| Secret | Where | Rotation |
|---|---|---|
| `ANTHROPIC_API_KEY` (or Claude Code subscription credential) | `.env.local` | When leaked or compromised |
| `COMPOSIO_API_KEY` | `.env.local` | On Composio rotation events |
| `SENDBLUE_API_KEY`, `SENDBLUE_API_SECRET` | `.env.local` | On Sendblue rotation events |
| Convex deploy key | `.env.local` (managed by `npx convex` CLI) | Per Convex's policy |
| OAuth tokens for connected toolkits | **Composio's storage**, not Boop's | Per-toolkit, via Composio dashboard |

`.env.local` is gitignored. **Never** commit it. **Never** paste its contents into a chat or a debug script that lands in the repo.

## What lands in git is public

This repo is public. The PII checklist in [`CLAUDE.md`](../CLAUDE.md) is the canonical pre-commit policy. The short version:

- Real email addresses, phone numbers, postal addresses → placeholders.
- Live API keys, tokens, secrets → never in commits.
- Composio connection IDs (`ca_*`), connected-account aliases → redact.
- Personal queries, financial details, message content from real conversations → don't commit.
- Production URLs, internal hostnames → don't commit.

Once a secret lands in git, GitHub keeps the orphaned commit reachable for weeks even after a force-push. Treat any leak as "rotate immediately, don't try to scrub."

## Drafts are the safety surface

Every external action stages as a draft. The execution agent does not commit; the dispatcher's `send_draft` does, and only after explicit user confirmation in the conversation.

This is enforced **structurally**, not by prompt:
- The execution agent's tool surface includes `save_draft`. The system prompt routes through it.
- The actual commit path is `dispatcher → send_draft → new execution-agent spawn → real tool call`.
- A misbehaving execution agent that calls a real send tool directly will succeed (the prompt didn't gate it), but the dispatcher won't have triggered it. This is the one place where the prompt-only enforcement is acceptable — see `tech-debt-tracker.md` #4.

## Audit trail

Every model call, tool call, and tool result is logged to Convex (`agentLogs`, `usageRecords`, `memoryEvents`, `automationRuns`, `consolidationRuns`, `drafts`). The dashboard replays it.

If you suspect Boop did something it shouldn't have, the dashboard's Agents tab is the canonical source of "what actually happened, in order."

## What's deliberately out of scope

- **Multi-user isolation.** Single-tenant.
- **Per-user encryption at rest.** Convex's at-rest encryption is what we have.
- **Data retention policies.** No automatic purge. The `prune` lifecycle on memories is for relevance, not compliance.
- **Compliance frameworks.** SOC2/HIPAA/etc. are out of scope. If your fork needs them, the audit trail is a starting point, not a finished story.

## If you find a security issue

Don't open a public issue. Email the maintainer directly. The repo's public nature means an issue is a disclosure.
