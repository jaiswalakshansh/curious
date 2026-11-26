# 00 · Overview — the mental model

## What ARTEX is in one breath

ARTEX is an **autonomous penetration tester**. Not a scanner with a chatbot stapled on — an *engine*
that plans, acts, observes, and re-plans on its own, against a real target, until it proves a goal or
exhausts its ideas. You hand it a scope and an objective ("find exploitable issues in `app.example.com`"),
and it drives the whole engagement.

Three things make that claim real instead of marketing:

1. **It thinks in a loop that never dies.** A planner agent reads the current state of the target and
   dispatches work; worker agents do the work and report back; the report changes the state, which
   wakes the planner. No fixed script.
2. **Its memory lives in a database, not a prompt.** The entire engagement — every asset, every fact,
   every finding and the reasoning that links them — is two graphs in PostgreSQL. The LLM context
   window is a scratchpad; the graph is the truth.
3. **A finding is only real when the server can prove it.** ARTEX records every HTTP exchange through
   a man-in-the-middle proxy and refuses to let a goal be marked "done" unless a finding is bound to
   captured traffic whose bytes the server itself re-checks.

That trio — reactive loop + externalized state + machine-verified evidence — is the whole story. The
rest of these docs is *how* each piece is built and *why* it beats the obvious alternative.

## The three forks in this folder

All three share one codebase ancestor (`github.com/Autumn-27/artex`) and one agent runtime SDK
(`github.com/Autumn-27/norma`). They are the same organism at three evolutionary points:

| Fork | `refs/` path | Bet | Scale |
|---|---|---|---|
| **Upstream** | `original/` | The original engine — the loop, the graph, the tools. | the seed |
| **BaaRai mainline** | `BaaRai-ARTEX/` | *Trustworthy, resilient, operable* offense: three server-side evidence gates, provider failover, budgets, a prompt-injection boundary, a browser extension for authenticated targets, 12 skill packages. | 21.4k★ · v0.7.x |
| **artex-ymh** | `artex-ymh/` | *Reach and stealth*: deep internal-network lateral movement, an automatic external→internal handoff, honeypot evasion, a decoy "disguise gate," SHA-pinned tooling. | 112★ · v0.3.x base |

Where they agree, this teardown cites `original/` or `BaaRai-ARTEX/` (the two read in full from source).
Where BaaRai or ymh add something, the doc says so.

## Why study this at all

Because ARTEX is a working, open-source answer to a question the whole industry is circling: **how do
you make an LLM agent reliably good at a long, open-ended, multi-step task without it losing the plot?**

The naive approach is to put the goal in a single agent's prompt and let it ReAct in a loop (this is
roughly what a Claude Code session, or a Pi-style agent, does for coding). That works for bounded tasks
and falls apart on long ones: the context fills with dead ends, the agent tunnel-visions on one path,
it forgets what it already tried, and it happily hallucinates success.

ARTEX's architecture is a direct, legible set of countermeasures to each of those failure modes. That's
what makes it worth reading even if you never touch offensive security: it's a **reference design for a
durable autonomous agent**, and the pentest domain just happens to be a brutally honest test of whether
such a design actually holds up (the target either fell over or it didn't).

[`04-harness-comparison.md`](04-harness-comparison.md) makes that comparison explicit. Start with
[`01-architecture-reference.md`](01-architecture-reference.md) for the full machine, or
[`02-boot-and-launch.md`](02-boot-and-launch.md) if you'd rather watch it start up.

## A standing caveat

ARTEX ships with **high capability and low built-in containment** — tools auto-execute, and the
upstream authorization-scope guard was deliberately removed (see
[`01-architecture-reference.md`](01-architecture-reference.md) §13). Everything here is written to
explain the engineering. Point the real thing only at systems you own or are explicitly authorized to
test.
