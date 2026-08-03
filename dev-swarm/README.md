# dev-swarm

A multi-vendor coding **swarm**. A Claude (Sonnet 5) brain takes your prompt,
enriches it, fans it out into tasks, and drives each through a bounded pipeline of
eight specialists — Claude (direct SDK + via kiro on pi), GPT, and MiniMax —
plus a ninth, on-demand MiniMax `support` worker for debugging,
troubleshooting, log/data analysis, and boilerplate. The brain writes no code
and never merges; the human merges the PRs.

## Roster

| Role | Worker | Harness | Model | Effort | Purpose |
|---|---|---|---|---|---|
| **brain** | dev-swarm | `claude-sdk` | `claude-sonnet-5` | medium | takes the goal, fans out, orchestrates |
| **research** | researcher | `pi` (minimax) | `minimax/MiniMax-M3` | — | online / local research (`explore`) |
| **plan** | planner | `pi` (kiro) | `kiro/claude-opus-5:xhigh` | xhigh | PRD + clarifying questions (`explore`) |
| **implement** | implementer | `pi` (kiro) | `kiro/claude-sonnet-5:high` | high | NORMAL tasks: review PRD, code + tests, open PR; text/DOM browser (`implement`) |
| **expert** | expert implementer | `pi` (kiro) | `kiro/claude-opus-5:high` | high | DIFFICULT tasks only: hard bugs / failed fixes; text/DOM browser (`implement`) |
| **review** | code reviewer | `pi` (kiro) | `kiro/gpt-5-6-sol:high` | high | cross-model diff review (`review`) |
| **qa** | QA / visual-check | `claude-native` | `claude-opus-5` | high | run & see: browser/visual (`review`) |
| **docs** | document writer | `pi` (minimax) | `minimax/MiniMax-M3` | — | README + LEARNING (`implement`) |
| **host** | host / preview | `pi` (minimax) | `minimax/MiniMax-M2.7` | — | serve latest on an unused port + Tailscale URL (`implement`) |
| **support** | support (on-demand) | `pi` (minimax) | `minimax/MiniMax-M3` | — | debug/troubleshoot, log/data analysis, boilerplate (`explore`/`implement`) |

**Permissions:** the swarm runs headless, no per-action approval prompts. Every
worker except `qa` uses **`bypassPermissions`** (for Claude harnesses, Omnigent
translates this to `--dangerously-skip-permissions`; the kiro-on-pi workers get
uninterrupted headless autonomy directly). `qa` (`claude-native`) uses
**`auto`** instead — `bypassPermissions` does not reliably clear Claude Code's
per-worktree "trust this folder?" dialog for a headless claude-native sub-agent
(confirmed: it hung on boot), while `auto` both auto-approves tool calls and
skips that trust dialog. Safety does **not** come from prompting — it comes
from the `blast_radius` guardrail every worker carries: the catastrophic set
(force-push, `rm -rf /`, hard-reset to a remote ref) stays denied regardless.

## Task vs question (the brain decides first)

The brain classifies your prompt before doing anything. A **question** never
activates the swarm — the brain answers directly, or (if it can't) dispatches
`research`, synthesizes the findings, and answers you with sources. Only a **task**
(build / change / fix / ship / investigate the codebase) runs the pipeline below.

## Pipeline (per task; independent tasks may fan out in parallel)

1. **research** *(if needed)* — gather external facts (docs, APIs, prior art,
   error messages) online or locally, and feed them into planning.
2. **plan → PRD gate** — the planner drafts a PRD and asks the **human**
   clarifying questions; the brain relays them and waits. Once answered, the plan
   is finalized into a clear PRD (scope, acceptance criteria, verification) that
   also **labels each task's difficulty** (normal/difficult) so the brain can route
   one owner per task.
3. **implement / expert — one owner per task** — the PRD labels each task's
   difficulty, and the brain delegates it to **exactly one** implementer:
   **normal → `implement` (Sonnet)**, **difficult → `expert` (Opus)**. The two
   **never work the same task** (no wasted tokens). The chosen worker reviews the
   PRD (asks blocking questions if needed), writes code + tests, drives to green,
   opens its own PR. A fix `implement` can't land (repeated review/QA failures,
   HARD flags) **hands off** to `expert` with the full history — one at a time,
   never both at once.
4. **review** (GPT via kiro, cross-model) — judges the diff vs the PRD.
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

## Support (on-demand, any stage)

`support` isn't a numbered pipeline stage — the brain dispatches it whenever a
worker's report needs deeper diagnostic digging, raw logs/data need making
sense of before planning or fixing anything, or a task needs
boilerplate/scaffolding generated before an implementer should spend tokens on
it. It runs on **MiniMax M3 via pi** — the same model line as `research`/`docs`,
kept free of the fixed pipeline — and returns root-cause findings, log/data
analysis, or scaffold files (written directly into the worktree when asked).
It never opens a PR; the brain routes its output back into the pipeline at
whichever stage needs it (usually `plan` for a root-cause that changes the
plan, or straight to the task's owning implementer for a fix or scaffold).

## Independent review

Implementers run **Claude (Sonnet 5 normal / Opus 5 expert) via kiro on pi**; the
reviewer runs **GPT-5.6 Sol via kiro on pi** — a different model line, so review is
an independent cross-check, not the same model grading its own work. The reviewer
gets only the diff + PRD (never the worktree) and never edits; only the implementer
opens a PR. Planning and run-and-see QA use **Claude Opus 5**.

## LEARNING.md — cross-session memory

`docs` maintains `LEARNING.md` specifically so that **agents in later sessions**
(which start with no memory of this run) can read what already failed and worked.
Point future dev-swarm runs at it. The brain also consults it **within** a run —
whenever QA hits a hard problem or the same failure repeats more than 3 times, it
reads `LEARNING.md` first (a prior session may have already solved it) before
spending more fix attempts.

## Visual QA & browser access

Only `qa` runs on **`claude-native`** (Claude Code), which has the Playwright/
browser MCP configured (`~/.claude.json`) and verified working — so real
screenshots and live visual/UI checks work out of the box. Everyone else with
browser access — `implement`, `expert`, `review` (kiro on pi) and `research`,
`host` (minimax on pi) — has the Playwright MCP too, but is **text/DOM only**
(`browser_navigate` + `browser_snapshot`). They must NOT take screenshots: a
screenshot is a large image and kiro models reject it with
`context_length_exceeded`. Pixel/visual verification is `qa`'s job alone.

## Host requirements (the runner)

- A **Claude provider** (`omnigent setup`) — for the brain and `qa`.
- **`pi`** + the **kiro provider** — for `plan`, `implement`, `expert`, `review`
  (`kiro/claude-opus-5:xhigh`, `kiro/claude-sonnet-5:high`,
  `kiro/claude-opus-5:high`, `kiro/gpt-5-6-sol:high`, provider-qualified).
- **`pi`** + the **minimax** provider — for `research`, `docs`, `host`, `support`
  (`minimax/MiniMax-M3`, `minimax/MiniMax-M2.7`, provider-qualified).
- **`tailscale`** on PATH + host on the tailnet — for `host` preview URLs.
- A browser MCP (Playwright) for `qa` visual checks and `research` browsing.

## Layout

```
dev-swarm/
  config.yaml                # brain (claude-sdk sonnet-5, medium) + full pipeline
  agents/
    research/config.yaml     # pi minimax/MiniMax-M3 — online/local research
    plan/config.yaml         # pi kiro/claude-opus-5:xhigh — PRD + questions
    implement/config.yaml    # pi kiro/claude-sonnet-5:high — normal implementer
    expert/config.yaml       # pi kiro/claude-opus-5:high — expert implementer (hard tasks)
    review/config.yaml       # pi kiro/gpt-5-6-sol:high — cross-model review
    qa/config.yaml           # claude-native opus-5 (high) — QA / visual
    docs/config.yaml         # pi minimax/MiniMax-M3 — README + LEARNING
    host/config.yaml         # pi minimax/MiniMax-M2.7 — serve (unused port) + Tailscale URL
    support/config.yaml      # pi minimax/MiniMax-M3 — on-demand debug/log-data/boilerplate
  README.md
```

## Install (server side) / run

Seeded as a persistent built-in via `OMNIGENT_BUILTIN_AGENT_DIRS` (mounted repo in
the server's docker-compose override), so it appears in the app's **Agents**
picker. Refresh after editing: `git pull` in the mounted checkout, restart the
server container.
