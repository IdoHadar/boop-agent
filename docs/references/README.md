# References

External docs we lean on, captured locally so an agent working in this repo can read them in-context. Files in this directory should be:

- **Static.** Vendor-published `llms.txt` or distilled markdown, not arbitrary HTML.
- **Versioned.** Filename ends in the version when the upstream is fast-moving (e.g. `claude-agent-sdk-llms-0.1.txt`).
- **Annotated.** A short header at the top of each file describing what's in it and when it was pulled.

## What lives here

| File | Source | Pull cadence |
|---|---|---|
| `claude-agent-sdk-llms.txt` | Anthropic's Claude Agent SDK distilled docs | On SDK minor version bumps |
| `composio-llms.txt` | Composio's distilled toolkit docs | Quarterly or on auth-config changes |
| `convex-llms.txt` | Convex's distilled docs (matches `convex/_generated/ai/guidelines.md`) | On `npx convex ai-files install` runs |
| `sendblue-api.md` | Sendblue webhook + send API surface we use | When the webhook shape changes |

## What does not live here

- Internal docs. Those go in `docs/design-docs/` or `docs/product-specs/`.
- Tutorials. We link to them; we don't copy them.
- Anything covered by `convex/_generated/ai/guidelines.md`. That file is the canonical Convex reference for in-repo agents — see `CLAUDE.md`.

## Adding a new reference

1. Drop the file in this directory.
2. Add a row to the table above.
3. Add a one-line pointer to the relevant doc that depends on it (e.g. a design doc that references SDK behavior).
