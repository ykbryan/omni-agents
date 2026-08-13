# dev-swarm

A multi-vendor coding **swarm**. A Claude (Sonnet 5) brain takes your prompt,
enriches it, fans it out into tasks, and drives each through a bounded pipeline of
nine specialists — Claude (direct SDK and claude-native), GPT (Codex CLI),
and MiniMax —
plus a tenth, on-demand MiniMax `support` worker for debugging,
troubleshooting, log/data analysis, and boilerplate. The brain writes no code
and never merges; the human merges the PRs.

## Roster

| Role | Worker | Harness | Model | Effort | Purpose |
|---|---|---|---|---|---|
| **brain** | dev-swarm | `claude-sdk` | `claude-sonnet-5` | medium | takes the goal, fans out, orchestrates |
| **research** | researcher | `pi` (minimax) | `minimax/MiniMax-M3` | — | online / local research (`explore`) |
| **plan** | planner | `codex` | `gpt-5.6-sol` | xhigh | PRD + clarifying questions (`explore`) |
| **implement** | implementer | `claude-native` | `claude-sonnet-5` | high | NORMAL tasks: review PRD, code + tests, open PR; native browser (`implement`) |
| **expert** | expert implementer | `claude-native` | `claude-opus-5` | high | DIFFICULT tasks only: hard bugs / failed fixes; native browser (`implement`) |
| **review** | code reviewer | `codex` | `gpt-5.6-sol` | high | cross-model diff review (`review`) |
| **qa** | QA / visual-check | `claude-native` | `claude-opus-5` | high | run & see: browser/visual (`review`) |
| **sanitize** | sanitizer / content-integrity | `pi` (minimax) | `minimax/MiniMax-M3` | — | scan diff for invisible/anomalous Unicode; flag possible disclosure signals to the human (`implement`) |
| **docs** | document writer | `pi` (minimax) | `minimax/MiniMax-M3` | — | README + LEARNING (`implement`) |
| **host** | host / preview | `pi` (minimax) | `minimax/MiniMax-M2.7` | — | serve latest on an unused port + Tailscale URL (`implement`) |
| **support** | support (on-demand) | `pi` (minimax) | `minimax/MiniMax-M3` | — | debug/troubleshoot, log/data analysis, boilerplate (`explore`/`implement`) |

**Permissions:** the swarm runs headless, no per-action approval prompts. The
pi/minimax workers (`research`, `docs`, `host`, `support`, `sanitize`) and the
`codex` planner and reviewer use **`bypassPermissions`**, which gives them
uninterrupted headless autonomy directly. The `claude-native` workers (`qa`,
`implement`, `expert`) use **`auto`** instead — `bypassPermissions`
(`--dangerously-skip-permissions`) does
not reliably clear Claude Code's per-worktree "trust this folder?" dialog for a
headless claude-native sub-agent in a fresh git worktree (confirmed: it hung on
boot), while `auto` both auto-approves tool calls and skips that trust dialog.
Safety does **not** come from prompting — it comes from the `blast_radius`
guardrail every worker carries: the catastrophic set (force-push, `rm -rf /`,
hard-reset to a remote ref) stays denied regardless.

**Roster discipline:** the brain has exactly these ten sub-agents and no
others. It must never dispatch to an `agent_id` outside this roster (via
`sys_session_create`, `sys_session_send`, or any other path), and never
improvise a substitute when a roster worker's provider is down — a provider
outage means drop that worker, tell the human exactly what's blocked, and
stop. Any one-off deviation requires the human's explicit, in-the-moment
sign-off, never an inference from something said earlier. (This is a
hardened rule, not a suggestion: a previous session's brain went off-roster
to an ad-hoc Codex session during a `pi` outage to keep a task moving: then
the brain itself got stuck before it could relay a follow-up correction
narrowing that session's role, and the unsupervised session kept going and
self-merged a PR with no review/QA gate.)

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
4. **review** (GPT via Codex CLI, cross-model) — judges the diff vs the PRD.
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
   - pass → sanitize.
6. **sanitize** (MiniMax M3 via pi) — scans the final diff for invisible or
   anomalous Unicode (zero-width characters, bidi control characters — the
   Trojan Source class, CVE-2021-42574, homoglyphs). Cleans unambiguous noise
   directly; anything that could be an intentional signal — an
   attribution/disclosure marker, a watermark, a license notice — gets
   reported to the human, never silently stripped. This is a content-integrity
   and security check, not a tool for hiding that a change is AI-generated:
   `Co-authored-by:` trailers and any AI-authorship attribution stay untouched.
7. **docs** — updates the **README** from all the changes, and appends to
   **LEARNING.md** — a cross-session memory written *for agents in future
   sessions*: what was tried, what failed and why, what worked, and the gotchas.
8. **host** — pulls the latest changes and serves the app on an **unused port**,
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

Implementers run **Claude (Sonnet 5 normal / Opus 5 expert) on claude-native**; the
reviewer runs **GPT-5.6 Sol on the Codex CLI** — a different model line, so review is
an independent cross-check, not the same model grading its own work. The reviewer
gets only the diff + PRD (never the worktree) and never edits; only the implementer
opens a PR. Planning runs on **GPT-5.6 Sol via the Codex CLI**, same as review;
run-and-see QA uses **Claude Opus 5**.

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
screenshots and live visual/UI checks work out of the box. `implement` and
`expert` also run on **`claude-native`**, so they have the same native
Playwright MCP access — TEXT/DOM snapshots or a real screenshot when they
genuinely need to see something, though the systematic pixel/visual pass stays
`qa`'s job. `review` (Codex CLI) and `research`, `host` (minimax on pi) have
the Playwright MCP too, but are **text/DOM only** (`browser_navigate` +
`browser_snapshot`) — a screenshot is a large image and these models
reject it with `context_length_exceeded`.

## Host requirements (the runner)

- A **Claude provider** (`omnigent setup`) — for the brain, `qa`,
  `implement`, `expert`.
- **`codex`** (Codex CLI, authenticated) — for `plan` and `review`. The model
  (`gpt-5.6-sol`) is taken from your `~/.codex/config.toml` default, not
  pinned in either agent config; reasoning effort IS pinned per-agent
  (`plan` xhigh, `review` high).
- **`pi`** + the **minimax** provider — for `research`, `docs`, `host`,
  `support`, `sanitize` (`minimax/MiniMax-M3`, `minimax/MiniMax-M2.7`,
  provider-qualified).
- **`tailscale`** on PATH + host on the tailnet — for `host` preview URLs.
- A browser MCP (Playwright) for `qa` visual checks and `research` browsing.

## Layout

```
dev-swarm/
  config.yaml                # brain (claude-sdk sonnet-5, medium) + full pipeline
  agents/
    research/config.yaml     # pi minimax/MiniMax-M3 — online/local research
    plan/config.yaml         # codex gpt-5.6-sol (xhigh) — PRD + questions
    implement/config.yaml    # claude-native claude-sonnet-5 (high) — normal implementer
    expert/config.yaml       # claude-native claude-opus-5 (high) — expert implementer (hard tasks)
    review/config.yaml       # codex gpt-5.6-sol (high) — cross-model review
    qa/config.yaml           # claude-native opus-5 (high) — QA / visual
    sanitize/config.yaml     # pi minimax/MiniMax-M3 — invisible-Unicode / integrity scan
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
