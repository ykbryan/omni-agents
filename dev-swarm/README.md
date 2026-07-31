# dev-swarm

A multi-vendor coding **swarm**. A Claude (Sonnet 5) brain takes your prompt,
enriches it, fans it out into tasks, and drives each through a bounded pipeline of
eight specialists — Claude, Codex, and MiniMax. The brain writes no code and never
merges; the human merges the PRs. **No kiro dependency.**

## Roster

| Role | Worker | Harness | Model | Effort | Purpose |
|---|---|---|---|---|---|
| **brain** | dev-swarm | `claude-sdk` | `claude-sonnet-5` | medium | takes the goal, fans out, orchestrates |
| **research** | researcher | `pi` (minimax) | `minimax/MiniMax-M3` | — | online / local research (`explore`) |
| **plan** | planner | `claude-sdk` | `claude-opus-5` | xhigh | PRD + clarifying questions (`explore`) |
| **implement** | implementer | `claude-native` | `claude-sonnet-5` | high | NORMAL tasks: review PRD, code + tests, open PR; browser + preview (`implement`) |
| **expert** | expert implementer | `claude-native` | `claude-opus-5` | high | DIFFICULT tasks only: hard bugs / failed fixes; browser + preview (`implement`) |
| **review** | code reviewer | `codex-native` | `gpt-5.6-sol` | high | cross-vendor diff review (`review`) |
| **qa** | QA / visual-check | `claude-native` | `claude-opus-5` | high | run & see: browser/visual (`review`) |
| **docs** | document writer | `pi` (minimax) | `minimax/MiniMax-M3` | — | README + LEARNING (`implement`) |
| **host** | host / preview | `pi` (minimax) | `minimax/MiniMax-M2.7` | — | serve latest on an unused port + Tailscale URL (`implement`) |

**Permissions:** the entire swarm runs **`bypassPermissions`** — no per-action
approval prompts, so it works fully headless. For the Claude harnesses Omnigent
translates this to `--dangerously-skip-permissions` (which also clears the
trust-folder dialog); codex bypasses via `yolo: true`. Safety does **not** come
from prompting — it comes from the `blast_radius` guardrail every worker carries:
the catastrophic set (force-push, `rm -rf /`, hard-reset to a remote ref) stays
denied regardless.

## Pipeline (per task; independent tasks may fan out in parallel)

1. **research** *(if needed)* — gather external facts (docs, APIs, prior art,
   error messages) online or locally, and feed them into planning.
2. **plan → PRD gate** — the planner drafts a PRD and asks the **human**
   clarifying questions; the brain relays them and waits. Once answered, the plan
   is finalized into a clear PRD (scope, acceptance criteria, verification).
3. **triage → implement / expert** — both implementers read the PRD and
   self-assess who should own each task; the brain then delegates by **difficulty**:
   **normal → `implement` (Sonnet)**, **difficult → `expert` (Opus)**. The chosen
   worker reviews the PRD (asks blocking questions if needed), writes code + tests,
   drives to green, opens its own PR. A fix a normal implementer can't land
   (repeated review/QA failures, HARD flags) escalates to `expert` with the full
   history.
4. **review** (Codex/GPT, cross-vendor) — judges the diff vs the PRD.
   - fail → back to **implement**, retry.
   - **3 failures → STOP the whole swarm and alert the human** (with the review
     history). No silent looping.
   - pass → QA.
5. **qa** (Claude Opus, live/visual) — runs the app and checks it works and looks
   right.
   - fail → back to **implement**, re-QA.
   - **hard problem, or the same failure more than 3 times → consult `LEARNING.md`
     first** (a past session may have already solved it) and fold what it says
     into the next attempt.
   - **more than 3 failures → back to `plan`** to re-plan with the QA feedback +
     whatever LEARNING had (the plan, not just the code, is likely wrong).
   - pass → docs.
6. **docs** — updates the **README** from all the changes, and appends to
   **LEARNING.md** — a cross-session memory written *for agents in future
   sessions*: what was tried, what failed and why, what worked, and the gotchas.
7. **host** — pulls the latest changes and serves the app on an **unused port**,
   returning a **Tailscale preview URL** for the human.

**New task while a preview is up:** the brain first tells `host` to **kill** the
running app and wait, then runs the new task's pipeline and re-hosts — so the
preview never shows stale code.

## Cross-vendor by design

Implementer = **Claude Sonnet 5**; code reviewer = **Codex / GPT-5.6 Sol** (a
different vendor — real independent review); planning and run-and-see QA use
**Claude Opus 5**. The reviewer gets only the diff + PRD (never the worktree) and
never edits; only the implementer opens a PR.

## LEARNING.md — cross-session memory

`docs` maintains `LEARNING.md` specifically so that **agents in later sessions**
(which start with no memory of this run) can read what already failed and worked.
Point future dev-swarm runs at it. The brain also consults it **within** a run —
whenever QA hits a hard problem or the same failure repeats more than 3 times, it
reads `LEARNING.md` first (a prior session may have already solved it) before
spending more fix attempts.

## Visual QA & browser access

`qa` and `implement` run on **`claude-native`** (Claude Code), which has the
Playwright/browser MCP configured (`~/.claude.json`) and verified working — so
visual/UI checks and browser-based diagnosis work out of the box, no extra wiring.
`research` and `host` (pi) also have the Playwright MCP available. Screenshots are
fine on Claude models; use `browser_snapshot` (text/DOM) when that's enough.

## Host requirements (the runner)

- A **Claude provider** (`omnigent setup`) — for the brain, `plan`, `implement`,
  `qa`.
- **`codex`** on PATH — for `review` (able to serve `gpt-5.6-sol`).
- **`pi`** + the **minimax** provider — for `research`, `docs`, `host`
  (`minimax/MiniMax-M3`, `minimax/MiniMax-M2.7`, provider-qualified).
- **`tailscale`** on PATH + host on the tailnet — for `host` preview URLs.
- A browser MCP (Playwright) for `qa` visual checks and `research` browsing.

## Layout

```
dev-swarm/
  config.yaml                # brain (claude-sdk sonnet-5, medium) + full pipeline
  agents/
    research/config.yaml     # pi minimax/MiniMax-M3 — online/local research
    plan/config.yaml         # claude-sdk opus-5 (xhigh) — PRD + questions
    implement/config.yaml    # claude-native sonnet-5 (high) — normal implementer
    expert/config.yaml       # claude-native opus-5 (high) — expert implementer (hard tasks)
    review/config.yaml       # codex-native gpt-5.6-sol (high) — cross-vendor review
    qa/config.yaml           # claude-native opus-5 (high) — QA / visual
    docs/config.yaml         # pi minimax/MiniMax-M3 — README + LEARNING
    host/config.yaml         # pi minimax/MiniMax-M2.7 — serve (unused port) + Tailscale URL
  README.md
```

## Install (server side) / run

Seeded as a persistent built-in via `OMNIGENT_BUILTIN_AGENT_DIRS` (mounted repo in
the server's docker-compose override), so it appears in the app's **Agents**
picker. Refresh after editing: `git pull` in the mounted checkout, restart the
server container.
