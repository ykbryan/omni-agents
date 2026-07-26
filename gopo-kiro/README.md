# gopo-kiro

A polly-style coding orchestrator with a fixed three-stage pipeline and a
custom harness/model mix.

## Pipeline

| Stage | Worker | Harness | Model | Purpose |
|---|---|---|---|---|
| **gopo** (brain) | orchestrator | `pi` | `minimax-m3` | plans, delegates, verifies; writes no code |
| **plan** | planner | `kiro-native` | `claude-opus-4.8` | investigate + produce plan/acceptance contracts (`explore`) |
| **execute** | implementer | `claude-native` | `claude-sonnet-5` | make code/test changes, open PR (`implement`) |
| **verify** | reviewer | `kiro-native` | `gpt-5.6-sol` | cross-vendor review of the diff (`review`) |

Flow: **plan → execute (opens its own PR) → verify → (fixes loop) → human merges.**
gopo never writes code and never merges.

## Layout

```
gopo-kiro/
  config.yaml            # orchestrator (gopo) — brain + prompt + roster
  agents/
    plan/config.yaml     # kiro planner
    execute/config.yaml  # claude implementer
    verify/config.yaml   # kiro reviewer
  README.md
```

## What was requested vs. what this ships (read this)

You asked for:

- gopo — hermes, **minimax m3**
- plan — kiro, **claude-opus-5**
- execute & code — claude, claude-sonnet-5
- verify — kiro, gpt-5.6-sol

Three of those needed adjustment to be runnable. Each is a one-line change in
the relevant `config.yaml` if you disagree with the call.

1. **plan model `claude-opus-5` → `claude-opus-4.8`.** kiro's catalog has no
   `claude-opus-5`. Its strongest Opus is `claude-opus-4.8` (it also has
   `claude-sonnet-5`). Confirm with `kiro-cli chat --list-models`. If you want
   real Opus 5, it lives on the **claude** harness, not kiro — you'd move `plan`
   to `claude-native` + `claude-opus-5`.

2. **gopo brain model `minimax-m3`** is not in Omnigent's static catalog — it's
   only resolvable if your configured gateway/provider serves it. If it doesn't,
   the brain's turns will error. Verify with `sys_list_models` / your gateway.

3. **gopo brain harness: `pi` (not `hermes-native`).** The brain was originally
   `hermes-native`, but a native-harness brain (a) can't drive the orchestration
   toolset (`sys_session_send`, spawn) and (b) is hidden from the Web UI **Agents**
   picker (the UI folds `*-native` harnesses into the "Harnesses" section). The
   `pi` harness is headless + non-native, so gopo-kiro now lists in the Agents
   picker alongside polly/debby, and pi can run any gateway model — keeping
   `minimax-m3`. Requirements: the `pi` CLI on the host PATH
   (`curl -fsSL https://pi.dev/install.sh | sh`) and a configured Pi model
   provider that serves `minimax-m3` (`omni setup` → Pi, or the app's Pi
   "needs setup" flow). If your provider doesn't serve `minimax-m3`, drop the
   `model:` line (pi default) or pin a served model. To go fully SDK instead,
   set `harness: claude-sdk` and drop `model:`.

## ⚠️ kiro-native is currently broken (Omnigent #3011)

Two of the three workers (`plan`, `verify`) run on `kiro-native`, which does not
work reliably:

- **Linux host:** kiro-cli's interactive TUI exits 1 on launch → every kiro turn
  fails. **Do not run gopo-kiro on the Linux/VM host.**
- **macOS host:** works intermittently; a turn can still fail with
  `kiro-native TUI input prompt was not ready before injection` — re-dispatch to
  retry.

**Run gopo-kiro on the macOS host only**, and expect occasional planner/verifier
retries until #3011 is fixed. When it is, no change is needed here. If you want a
fully reliable variant now, swap `plan`/`verify` off kiro (e.g. `plan` →
`claude-sdk`, `verify` → `codex` or `pi`).

## Install / run

Place the `gopo-kiro/` directory where Omnigent discovers agents (alongside the
bundled `examples/`, or point at it explicitly), then launch it against your
server, e.g.:

```bash
# on the macOS host (kiro workers are macOS-only, #3011)
omni run ~/omni-agents/gopo-kiro --server http://omni.taile2a4c7.ts.net:8000
```

Or, since gopo-kiro is seeded as a persistent built-in on the server (via the
`OMNIGENT_BUILTIN_AGENT_DIRS` env in the server's compose), just pick
**Gopo-kiro** from the desktop app's Agents picker.

Host PATH requirements: `pi` (brain), `kiro-cli` (plan, verify), and `claude`
(execute). gopo runs a one-shot preflight (`command -v kiro-cli claude`) and
routes only to workers whose CLI resolves. The `pi` brain additionally needs a
configured Pi model provider (see note 3 above) — until then the app shows a
"needs setup" badge on the agent.
