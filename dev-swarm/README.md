# dev-swarm

A multi-vendor coding orchestrator. A Claude (Sonnet 5) brain plans, delegates,
and integrates through six specialists — Claude, Codex, and MiniMax — and the
human merges the PRs. **No kiro dependency.**

## Roster

| Role | Worker | Harness | Model | Effort | Dispatch purpose |
|---|---|---|---|---|---|
| **brain** | dev-swarm | `claude-sdk` | `claude-sonnet-5` | medium | orchestrates; writes no code |
| **plan** | planner | `claude-sdk` | `claude-opus-5` | high | investigate + plan (`explore`) |
| **implement** | implementer | `claude-sdk` | `claude-sonnet-5` | high | code + tests, opens PR (`implement`) |
| **review** | code reviewer | `codex-native` | `gpt-5.6-sol` | high | cross-vendor diff review (`review`) |
| **qa** | QA / visual-check | `claude-sdk` | `claude-opus-5` | high | run & see: browser/visual (`review`) |
| **docs** | document writer | `pi` (minimax) | `minimax/MiniMax-M3` | — | README + LEARNING (`implement`) |
| **host** | host / preview | `pi` (minimax) | `minimax/MiniMax-M2.7` | — | serve latest code + Tailscale URL (`implement`) |

## Flow

**plan → implement (opens its own PR) → review → qa (when it must be run/seen) →
docs → host (preview URL) → (fixes loop) → human merges.** The brain never writes
code and never merges.

## Cross-vendor by design

The implementer is **Claude Sonnet 5**; the code reviewer is **Codex / GPT-5.6
Sol** — a genuinely different vendor, so review is an independent cross-check, not
the same model grading its own work. Planning and run-and-see QA use **Claude
Opus 5** for the hardest judgment. The reviewer gets only the diff + contract
(never the worktree) and never edits; only the implementer opens a PR.

## Host / preview (Tailscale)

The `host` worker checks out the latest code, starts the app bound to `0.0.0.0`,
and returns a **Tailscale URL** (`http://<tailscale-dns-or-100.x-ip>:<port>/`) so
you — or the `qa` worker — can preview the running app. It figures out the run
command from the repo (package.json scripts, Dockerfile, framework default),
launches the server as a long-running process, and reports the URL, port, log
path, and how to stop it. It runs on `minimax/MiniMax-M2.7`.

## Visual QA note (read this)

`qa` runs on `claude-sdk`. Its **visual/browser checks need a browser MCP**
(Playwright) reachable by the worker. The Playwright MCP is configured for **Claude
Code (`claude-native`)** on the host and verified working. If the `claude-sdk`
harness does **not** inherit that MCP, either expose the Playwright MCP to the SDK,
or switch the `qa` worker's harness to `claude-native` (add
`permission_mode: auto`) — one line in `agents/qa/config.yaml`. Text/DOM via
`browser_snapshot`; screenshots are fine on Claude models (unlike kiro).

## Host requirements (the runner)

- A **Claude provider** (subscription / Anthropic API key / gateway) configured
  via `omnigent setup` — for the brain, `plan`, `implement`, and `qa` (claude-sdk).
- **`codex`** CLI on PATH — for `review` (must be able to serve `gpt-5.6-sol`).
- **`pi`** CLI on PATH + the **minimax** provider configured — for `docs` and
  `host` (models `minimax/MiniMax-M3` and `minimax/MiniMax-M2.7`). Use the
  provider-qualified ids so pi uses the local minimax provider, not a gateway.
- **`tailscale`** on PATH and this host joined to your tailnet — for `host` to
  publish a preview URL.
- A browser MCP (Playwright) for `qa`'s visual checks (see note above).

## Model notes

- `claude-opus-5` / `claude-sonnet-5` resolve through the configured Claude
  provider (not kiro). `gpt-5.6-sol` must be a model the `codex` harness can serve
  in your setup. `minimax/MiniMax-M3` and `minimax/MiniMax-M2.7` come from the
  local pi minimax provider.
- Effort ladders: claude-sdk / claude-native use `reasoning_effort`
  (low/medium/high/xhigh/max); codex uses `reasoning_effort` → its
  `model_reasoning_effort`.

## Install (server side) / run

Seeded as a persistent built-in via `OMNIGENT_BUILTIN_AGENT_DIRS` (pointed at the
mounted repo in the server's docker-compose override), so it appears in the app's
**Agents** picker. To refresh after editing: `git pull` in the mounted checkout,
then restart the server container.

```bash
omni run ~/omni-agents/dev-swarm --server http://<your-omnigent-server>:8000
# or pick "Dev-swarm" from the app's Agents picker
```

## Layout

```
dev-swarm/
  config.yaml                # brain (claude-sdk sonnet-5, medium) + prompt + roster
  agents/
    plan/config.yaml         # claude-sdk opus-5 (high) — planner
    implement/config.yaml    # claude-sdk sonnet-5 (high) — implementer, opens PR
    review/config.yaml       # codex-native gpt-5.6-sol (high) — cross-vendor reviewer
    qa/config.yaml           # claude-sdk opus-5 (high) — QA / visual checks
    docs/config.yaml         # pi minimax/MiniMax-M3 — README + LEARNING
    host/config.yaml         # pi minimax/MiniMax-M2.7 — serve latest code + Tailscale URL
  README.md
```
