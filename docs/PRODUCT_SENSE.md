# PRODUCT SENSE

Opinions about what Boop is *for*, recorded in one place so they stop being argued over in chat.

## What Boop is

A personal agent **you text**. The texting part is non-negotiable. Switching apps to use Boop is a defeat. Asking the user to log in is a defeat. Showing the user JSON is a defeat.

## Who Boop is for

A single user — *you*, the person who forked it. Boop is a template for personal agents, not a multi-tenant product. Multi-user is a fork concern, not a default concern.

## What "good" feels like

> *Like texting a friend who happens to also be very competent at email, calendars, research, and remembering what you said last Tuesday.*

Three properties of that experience, in priority order:

1. **Fast on casual turns.** A two-second reply is the bar. Any turn that *can* finish without spawning *must*.
2. **Honest on slow turns.** "On it — searching now" is acceptable. Silent multi-minute pauses are not.
3. **Cautious on commits.** Anything that touches the world drafts first. Confirmation before send is the contract.

If a feature degrades any of those three, it's a feature we don't ship.

## What Boop is not

- **Not a chatbot.** It does work, not just talks.
- **Not a workflow engine.** It runs scoped tasks; it doesn't try to become a low-code platform.
- **Not a knowledge base.** Memory is for the user's life, not for Wikipedia.
- **Not multi-modal.** Text in, text out. Sendblue handles attachments at the protocol level; Boop doesn't make a UX out of them.
- **Not enterprise.** No team mode, no admin panel, no audit log beyond the dashboard.

## What ships, what doesn't

A good Boop PR removes a special case, makes a casual turn cheaper, makes a confirmation clearer, or unlocks an integration. A questionable Boop PR adds a configuration option, adds a tab to the dashboard, or makes the dispatcher larger.

When in doubt, the smaller-surface choice wins.

## What we'd say no to

- "Add a settings page in the dashboard for X." → If the user can't tell Boop to do X by texting, X is the wrong shape.
- "Add a slash-command syntax." → iMessage is conversational. Slash commands are a sign we couldn't get the dispatcher to interpret intent.
- "Cache the dispatcher's last reply for X seconds." → If the dispatcher is slow enough that caching matters, we fix the dispatcher.
- "Spawn an executor on every turn just in case." → That's the failure mode the dispatcher/executor split exists to prevent.
