# Graphify Integration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire Graphify into the Boop repo so Claude Code sessions have a live, queryable knowledge graph of the codebase via `npm run graph`.

**Architecture:** Two small repo changes (npm script + gitignore), followed by a one-time machine setup (install graphify, build initial graph). The graph outputs (`graph.json`, `GRAPH_REPORT.md`, `graph.html`) are committed alongside code; only the incremental cache is gitignored.

**Tech Stack:** Python 3.10+, `graphifyy` (PyPI), Claude Code skill system, Node.js npm scripts.

---

## File Map

| File | Change |
|---|---|
| `package.json` | Add `"graph"` script |
| `.gitignore` | Add `graphify-out/cache/` |
| `graphify-out/` | Created by `npm run graph` — commit outputs, not cache |
| `.claude/skills/graphify/` | May be created by `graphify install` — commit if present |

---

### Task 1: Add npm script and gitignore entry

**Files:**
- Modify: `package.json`
- Modify: `.gitignore`

- [ ] **Step 1: Add the `graph` script to `package.json`**

In `package.json`, add `"graph"` to the `scripts` block (after `"typecheck"`):

```json
"scripts": {
  "setup": "tsx scripts/setup.ts",
  "sendblue:sync": "node scripts/sendblue-sync.mjs",
  "sendblue:webhook": "node scripts/sendblue-webhook.mjs",
  "preflight": "node scripts/preflight.mjs",
  "dev": "node scripts/dev.mjs",
  "dev:parallel": "npm-run-all --parallel dev:server dev:convex dev:debug",
  "dev:server": "npm run preflight && tsx watch server/index.ts",
  "dev:convex": "convex dev",
  "dev:debug": "vite --config debug/vite.config.ts",
  "build:debug": "vite build --config debug/vite.config.ts",
  "start": "npm run preflight && tsx server/index.ts",
  "deploy:convex": "convex deploy",
  "typecheck": "tsc --noEmit",
  "graph": "graphify . --watch"
}
```

- [ ] **Step 2: Add `graphify-out/cache/` to `.gitignore`**

Append to `.gitignore`:

```
graphify-out/cache/
```

- [ ] **Step 3: Verify the script resolves**

```bash
npm run graph -- --help
```

Expected: graphify prints its help text (or fails with "graphify not found" if not yet installed — that's fine, confirms the script wiring is correct before setup).

- [ ] **Step 4: Commit**

```bash
git add package.json .gitignore
git commit -m "chore: add graphify watch script and gitignore cache"
```

---

### Task 2: One-time machine setup

These steps are not repo changes — they configure the developer's machine.

- [ ] **Step 1: Install the graphify CLI**

```bash
pip install graphifyy
```

Expected: installs `graphify` CLI. Confirm with:

```bash
graphify --version
```

- [ ] **Step 2: Install the Claude Code skill**

```bash
graphify install
```

Expected: installs the `/graphify` slash commands for Claude Code. Check where the skill landed:

```bash
ls .claude/skills/graphify/ 2>/dev/null && echo "project-level" || echo "not in project"
```

- [ ] **Step 3: If the skill installed project-level, commit it**

If the previous step printed `project-level`, the skill file needs to be committed:

```bash
git add .claude/skills/graphify/
git commit -m "chore: add graphify Claude Code skill"
```

If it printed `not in project`, it installed globally — no commit needed.

---

### Task 3: Build the initial graph and commit outputs

- [ ] **Step 1: Run the initial graph build**

```bash
npm run graph
```

Expected: Graphify runs a Tree-sitter AST pass and LLM semantic extraction pass. Takes roughly 1–2 minutes on first run. You'll see progress output, then it enters watch mode. Once you see the watch mode prompt, outputs are ready in `graphify-out/`.

- [ ] **Step 2: Verify outputs exist**

In a second terminal (don't kill the watch process):

```bash
ls graphify-out/
```

Expected:

```
GRAPH_REPORT.md
graph.html
graph.json
cache/
```

- [ ] **Step 3: Verify `cache/` is gitignored**

```bash
git status graphify-out/
```

Expected: `cache/` does NOT appear as an untracked or modified path. `graph.json`, `GRAPH_REPORT.md`, and `graph.html` DO appear as untracked.

If `cache/` appears in `git status`, check that `.gitignore` was saved correctly in Task 1 Step 2.

- [ ] **Step 4: Commit the graph outputs**

```bash
git add graphify-out/graph.json graphify-out/GRAPH_REPORT.md graphify-out/graph.html
git commit -m "chore: add initial graphify knowledge graph"
```

---

### Task 4: Smoke test Claude Code integration

- [ ] **Step 1: Open a Claude Code session in this repo**

```bash
claude
```

- [ ] **Step 2: Run a graphify query**

```
/graphify query "how does the memory system work?"
```

Expected: Graphify returns a structured answer referencing nodes and edges from the graph (e.g. `server/memory/`, `write_memory`, `recall`, Convex tables). If it returns an error about the graph not being found, verify `graphify-out/graph.json` exists at the repo root.

- [ ] **Step 3: Run a path query**

```
/graphify path "interaction-agent" "execution-agent"
```

Expected: a chain of nodes showing how the dispatcher connects to the executor.

- [ ] **Step 4: Confirm watch mode is live**

Edit any `.ts` file (add a blank line, save, revert). In the terminal running `npm run graph`, you should see Graphify detect the change and trigger a fast AST-only rebuild (no LLM call).

---

## Day-to-day workflow after setup

1. `npm run graph` in a terminal tab while working.
2. Graph updates automatically on code saves.
3. When committing, include updated `graphify-out/` files alongside your code changes.
4. In Claude Code: use `/graphify query`, `/graphify explain`, `/graphify path` for codebase questions.

## Future: integrate into `npm run dev`

Add Graphify as a 5th process in `scripts/dev.mjs`, with a guard:

```js
// In the parallel spawn block:
const graphifyInstalled = spawnSync("which", ["graphify"]).status === 0;
if (graphifyInstalled) {
  spawn("graphify", [".", "--watch"], { prefix: "graph" });
}
```

No other design changes needed.
