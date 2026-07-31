# omni-agents

Custom [Omnigent](https://github.com/omnigent-ai/omnigent) agent configurations.

Each subdirectory is a self-contained agent spec (a top-level `config.yaml`, plus
an `agents/` folder for any sub-agents) that Omnigent can discover and launch.

## Agents

- **[`dev-swarm/`](./dev-swarm/)** — a multi-vendor coding orchestrator. A Claude
  (Sonnet 5) brain delegates to six specialists: a Claude Opus planner, a Claude
  Sonnet implementer, a Codex (GPT-5.6 Sol) cross-vendor reviewer, a Claude Opus
  QA/visual worker, a MiniMax (M3) document writer, and a MiniMax (M2.7) host that
  serves the latest code over a Tailscale preview URL. No kiro dependency. See its
  [README](./dev-swarm/README.md).
- **[`polly-kiro/`](./polly-kiro/)** — a kiro-powered dynamic-delegation
  orchestrator (pi + kiro-provider workers, pi/minimax brain). See its
  [README](./polly-kiro/README.md).
