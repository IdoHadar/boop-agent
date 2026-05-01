# Spec: automations

## What the user should feel

> I texted "every morning at 8 summarize my calendar." Boop confirmed once. From then on, every weekday at 8 I got a summary in iMessage. When I texted "stop the morning summary," it stopped. I never opened a settings page.

## What the system must do

- Accept natural-language scheduling requests in any conversation.
- Convert the request into `create_automation(name, schedule, task, integrations, notify?)` where `schedule` is a 5-field cron expression.
- Store the automation in `automations` with `enabled: true`, the IANA timezone captured at create time, and the next run computed by `croner`.
- Poll for due automations every 30 seconds (`server/automations.ts`).
- Fire each due automation by spawning an execution agent with the same shape as an inline dispatch.
- Write each run's outcome to `automationRuns`.
- If the automation has a `notifyConversationId`, push the result back to the user's iMessage as a fresh assistant message.
- Support `list_automations`, `toggle_automation(id, enabled)`, and `delete_automation(id)` from inside any conversation.

## What the system must not do

- Do not silently fire if the user hasn't confirmed the automation request at least once.
- Do not double-fire when multiple server processes are running. (Today this is enforced by single-process deployment — see [reliability](../RELIABILITY.md).)
- Do not freeze the schedule's timezone to the server's clock — store the IANA zone with the automation.
- Do not drop the result if the notification channel is offline. The run is durable in `automationRuns` regardless of delivery.
