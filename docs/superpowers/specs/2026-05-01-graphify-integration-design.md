---
title: Graphify Integration Design
date: 2026-05-01
status: approved
---

# Graphify Integration

## Goal

Add [Graphify](https://graphify.net/) as a knowledge graph layer on the Boop repo so that Claude Code sessions have structured, queryable understanding of the codebase. This improves the quality of assistance during development, debugging, and upgrades.

## Scope

This is a developer tooling change — it affects the repo and the local development workflow, not the Boop agent itself (execution agent, interaction agent, Composio integrations, etc.).

## What Changes

### `package.json`

Add one script:

```json
"graph": "graphify . --watch"
```

### `.gitignore`

Add one entry:

```
graphify-out/cache/
```

The cache directory is an incremental build artifact (large, noisy, not useful to track). The graph outputs — `graph.json`, `GRAPH_REPORT.md`, `graph.html` — are committed alongside code changes.

## One-Time Setup (per developer machine)

```bash
pip install graphifyy        # installs the graphify CLI (package name is graphifyy)
graphify install             # installs the Claude Code skill (/graphify, /graphify query, etc.)
npm run graph                # builds initial graph + starts watch mode
```

The initial build runs a Tree-sitter AST pass and an LLM semantic extraction pass. On a repo the size of Boop (~30 TypeScript files + Convex schema + docs), this takes roughly 1–2 minutes. Subsequent starts are fast due to the cache.

> **Note:** `graphify install` may write a skill file to `.claude/skills/graphify/` in the project directory. If it does, that file should be committed. Verify the install destination and commit accordingly.

## Development Workflow

1. Run `npm run graph` in a terminal tab while working.
2. Graphify watches the repo:
   - Code saves → instant AST-only rebuild (no LLM, no cost).
   - Doc/markdown changes → Graphify notifies you to run `--update` for an LLM re-pass.
3. When you commit, include the updated `graphify-out/` outputs (`graph.json`, `GRAPH_REPORT.md`, `graph.html`) alongside your code changes.

## Claude Code Usage

With the skill installed, these commands are available in any Claude Code session on this repo:

| Command | Purpose |
|---|---|
| `/graphify .` | Rebuild the full graph |
| `/graphify query "<question>"` | Ask a natural-language question about the codebase |
| `/graphify path "A" "B"` | Find the connection between two nodes |
| `/graphify explain "<node>"` | Deep-dive on a specific function, module, or concept |

`GRAPH_REPORT.md` is also directly readable — it contains god nodes, surprise edges, and suggested questions, and is a useful starting point for any session.

## Committed Outputs

| File | Committed? | Why |
|---|---|---|
| `graphify-out/graph.json` | Yes | Queryable graph, used by `/graphify query` |
| `graphify-out/GRAPH_REPORT.md` | Yes | Human/agent-readable overview, persists across sessions |
| `graphify-out/graph.html` | Yes | Interactive visualization |
| `graphify-out/cache/` | No | Incremental build cache, large and machine-specific |

## Future: Integrate into `npm run dev`

When ready, wire Graphify into `scripts/dev.mjs` as a 5th parallel process alongside server/convex/vite/ngrok. Add a `which graphify` guard so it's a no-op for developers who haven't installed it. No design changes needed — just plumbing.

## Out of Scope

- Integrating Graphify into the Boop execution agent or interaction agent.
- Exposing Graphify as a Composio toolkit or custom MCP integration.
- Any changes to `server/`, `convex/`, or the debug dashboard.
