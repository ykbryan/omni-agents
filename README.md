# omni-agents

Custom [Omnigent](https://github.com/omnigent-ai/omnigent) agent configurations.

Each subdirectory is a self-contained agent spec (a top-level `config.yaml`, plus
an `agents/` folder for any sub-agents) that Omnigent can discover and launch.

## Agents

- **[`gopo-kiro/`](./gopo-kiro/)** — a polly-style coding orchestrator with a
  fixed three-stage pipeline: a kiro planner, a Claude implementer (opens its own
  PR), and a kiro verifier, driven by a hermes/minimax brain. See its
  [README](./gopo-kiro/README.md) for the harness/model matrix and important
  caveats (kiro-native is macOS-host-only pending omnigent-ai/omnigent#3011).
