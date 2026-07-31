# omni-agents

Custom [Omnigent](https://github.com/omnigent-ai/omnigent) agent configurations.

Each subdirectory is a self-contained agent spec (a top-level `config.yaml`, plus
an `agents/` folder for any sub-agents) that Omnigent can discover and launch.

## Agents

- **[`dev-swarm/`](./dev-swarm/)** — a multi-vendor coding **swarm**. A Claude
  (Sonnet 5) brain takes your prompt, asks clarifying questions and **enriches**
  it, then fans it out through a **bounded pipeline** of seven specialists:
  a MiniMax **researcher**, a Claude Opus **planner** (writes the PRD), a Claude
  Sonnet **implementer**, a Codex (GPT-5.6 Sol) cross-vendor **reviewer**, a
  Claude Opus **QA/visual** worker, a MiniMax **document writer** (README +
  LEARNING), and a MiniMax **host** that serves the latest code on an unused port
  behind a **Tailscale preview URL**. Bounded loops (review 3 strikes → stop +
  alert; QA more than 3 → replan, consulting `LEARNING.md`), `bypassPermissions`
  swarm-wide, and **no kiro dependency**. See its
  [README](./dev-swarm/README.md).
- **[`polly-kiro/`](./polly-kiro/)** — a kiro-powered **dynamic-delegation**
  orchestrator (pi + kiro-provider workers, pi/minimax brain). See its
  [README](./polly-kiro/README.md).
- **[`gopo-kiro/`](./gopo-kiro/)** — *archival.* A kiro-powered **fixed-pipeline**
  orchestrator (pi + kiro-provider workers: plan → execute → verify → qa) — the
  predecessor to `dev-swarm`, kept for reference. **Not seeded** into the Agents
  picker. See its [README](./gopo-kiro/README.md).
