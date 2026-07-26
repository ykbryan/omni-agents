# gopo-kiro

A polly-style coding orchestrator with a fixed three-stage pipeline. The
orchestrator and the two kiro-backed workers all run on the headless **`pi`**
harness, so nothing depends on the kiro-native TUI.

## Pipeline

| Stage | Worker | Harness | Model | Purpose |
|---|---|---|---|---|
| **gopo** (brain) | orchestrator | `pi` | `minimax-m3` | plans, delegates, verifies; writes no code |
| **plan** | planner | `pi` (kiro provider) | `kiro/claude-opus-4-8:high` | investigate + produce plan/acceptance contracts (`explore`) |
| **execute** | implementer | `claude-native` | `claude-sonnet-5` · effort `high` | make code/test changes, open PR (`implement`) |
| **verify** | reviewer | `pi` (kiro provider) | `kiro/gpt-5-6-sol:high` | independent review of the diff (`review`) |

**Reasoning effort** is `high` on all three workers. For the pi workers it's the
model `:high` suffix (pi's thinking level: off/minimal/low/medium/high/xhigh/max);
for the claude-native `execute` worker it's `reasoning_effort: high` in the
executor config (mapped to Claude Code `--effort`). The `gopo` brain is left at
its default.

Flow: **plan → execute (opens its own PR) → verify → (fixes loop) → human merges.**
gopo never writes code and never merges.

## How the kiro models are reached (this is the point)

`plan` and `verify` use kiro's models — but through the **`pi` harness + the
[`@arvoretech/pi-kiro-provider`](https://www.npmjs.com/package/@arvoretech/pi-kiro-provider)
extension**, which talks to the **Kiro API** directly (headless). They do **not**
drive the `kiro-native` TUI, so they completely sidestep omnigent
[#3011](https://github.com/omnigent-ai/omnigent/issues/3011) (the TUI-injection
bug). Model ids use the provider's dash-form (`kiro/claude-opus-4-8`,
`kiro/gpt-5-6-sol`).

The `gopo` brain runs on `pi` too (headless, non-native) so gopo-kiro lists in
the Web UI **Agents** picker alongside polly/debby (native-harness brains get
hidden under "Harnesses"), and pi can run any provider — here a direct MiniMax
provider for `minimax-m3`.

## Host setup (macOS host — the runner)

1. **pi CLI** on PATH:
   ```bash
   curl -fsSL https://pi.dev/install.sh | sh
   # ensure it's on the daemon's PATH (e.g. symlink into ~/.local/bin)
   ```
2. **kiro provider extension** (for plan/verify):
   ```bash
   pi install npm:@arvoretech/pi-kiro-provider@0.8.5
   pi   # then run: /login kiro   (reuses your kiro-cli identity)
   ```
3. **MiniMax provider** for the `minimax-m3` brain — configure pi with your
   direct MiniMax API key (the model selector is `minimax-m3`). If a provider
   that serves `minimax-m3` isn't configured, the brain won't boot.
4. **claude** CLI on PATH for the `execute` worker.

`omni config list` / a quick `pi --provider kiro --model claude-sonnet-4-6 -p hi`
confirm auth before launching.

## Model notes

- `claude-opus-4-8` is the top Opus the kiro provider exposes — there is **no**
  `opus-5`. Full provider catalog (dash-form): `claude-opus-4-8/4-7/4-6`,
  `claude-sonnet-5/4-6/4-5/4`, `claude-fable-5`, `claude-haiku-4-5`,
  `deepseek-3-2`, `glm-5`, `qwen3-coder-next`, `gpt-5-6-sol/terra/luna`,
  `minimax-m2/m2-1/m2-5`, `auto`.
- `minimax-m3` is **not** in the kiro provider (its MiniMax tops out at `m2-5`),
  which is why the brain uses a **direct** MiniMax provider instead.

## Install (server side) / run

gopo-kiro is seeded as a persistent built-in on the server via
`OMNIGENT_BUILTIN_AGENT_DIRS` (set in the server's docker-compose override to the
mounted repo), so it appears in the app's **Agents** picker. To refresh after
editing: `git pull` in the mounted checkout, then restart the server container.

Launch on the **macOS host** (kiro auth + pi extension live there):

```bash
omni run ~/omni-agents/gopo-kiro --server http://<your-omnigent-server>:8000
# or pick "Gopo-kiro" from the app's Agents picker, host = <your-macos-host>
```

gopo runs a one-shot preflight (`command -v pi claude`) and routes only to
workers whose CLI resolves.
