# polly-kiro

A kiro-powered variant of [polly](https://github.com/omnigent-ai/omnigent/tree/main/examples/polly).
It keeps polly's **dynamic** delegation style — investigate, fan out parallel
work, cross-review — but routes all coding work through kiro's models (via the
`pi` harness + the `@arvoretech/pi-kiro-provider` extension) instead of polly's
six native CLI harnesses. The orchestrator writes no code; every implementer
opens its own PR and the human merges.

## polly-kiro vs gopo-kiro

Same four kiro-backed workers as [gopo-kiro](../gopo-kiro), different
orchestration:

- **gopo-kiro** runs a *fixed* pipeline: plan → execute → verify → qa, every task.
- **polly-kiro** delegates *dynamically* (polly-style): it decides per goal
  whether to investigate first, fan out several `execute` tasks in parallel, and
  when to add `qa` — always cross-reviewing an implementer's diff with `verify`.

## Roster

| Role | Worker | Harness | Model | Purpose |
|---|---|---|---|---|
| **brain** | polly-kiro | `pi` | `minimax-m3` | plans, delegates, cross-reviews; writes no code |
| **plan** | planner / investigator | `pi` (kiro provider) | `kiro/claude-opus-4-8:high` | investigate + plan (`explore` / `search`) |
| **execute** | implementer | `pi` (kiro provider) | `kiro/claude-sonnet-5:high` | make code/test changes, open PR (`implement`) |
| **verify** | code reviewer | `pi` (kiro provider) | `kiro/gpt-5-6-sol:high` | independent cross-model review of the diff (`review`) |
| **qa** | QA / visual-check | `claude-native` | `claude-sonnet-5` (xhigh) | run-and-see: browser/UI/MCP checks (`review`) |

Cross-review is preserved as **cross-model**: the implementer runs Sonnet, the
reviewer runs GPT — an independent second opinion even though both go through
the kiro provider. `qa` adds a genuinely different harness (Claude Code) for
anything that must be run or seen.

## Why pi + kiro (not kiro-native)

The kiro-native TUI harness is affected by omnigent
[#3011](https://github.com/omnigent-ai/omnigent/issues/3011) (exits on Linux,
racy on macOS). Reaching kiro's models through the `pi` harness + the
`@arvoretech/pi-kiro-provider` extension uses the Kiro API **headlessly**, so it
sidesteps #3011 and runs reliably on Linux and macOS.

## Host requirements

- `pi` CLI on PATH (https://pi.dev/install.sh) — for `plan`, `execute`, `verify`.
- kiro extension installed: `pi install npm:@arvoretech/pi-kiro-provider@0.8.5`,
  then a one-time `/login kiro` in pi (reuses your kiro-cli identity).
- A gateway serving `minimax-m3` for the brain (or drop `model:` in
  `config.yaml` to use the pi default / set a served model).
- `claude` CLI on PATH — for the `qa` worker; a browser/Chrome MCP configured
  for Claude Code if you want visual checks.

## Brain caveat

The orchestration layer (dispatch / inbox supervision) is most reliable on the
`claude-sdk` brain. `minimax-m3` can under-orchestrate on long dynamic runs —
looping on continuation-sends instead of dispatch-and-wait. If polly-kiro stalls
or loops, switch the brain in `config.yaml` to:

```yaml
executor:
  config:
    harness: claude-sdk   # drop `model:` to use the provider default
```

Also: restarting the Omnigent server drops every active runner's tunnel
(`runner_disconnected`). Start a **fresh** polly-kiro session after any server
restart rather than continuing one whose brain is already mid-retry.

## Layout

```
polly-kiro/
  config.yaml            # orchestrator (brain + dynamic-delegation prompt + roster)
  agents/
    plan/config.yaml     # pi + kiro planner/investigator
    execute/config.yaml  # pi + kiro implementer
    verify/config.yaml   # pi + kiro cross-model reviewer
    qa/config.yaml        # claude-native QA / visual-check
  README.md
```

## Install / run

Place `polly-kiro/` where Omnigent discovers agents (e.g. point
`OMNIGENT_BUILTIN_AGENT_DIRS` at it, or run it explicitly):

```bash
omni run --agent polly-kiro --server http://<your-omnigent-server>:8000
# (or open it from the desktop app's Agents picker once registered)
```
