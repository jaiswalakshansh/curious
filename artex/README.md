# ARTEX — a deep teardown

**ARTEX** is an autonomous, multi-agent penetration-testing engine: a Go monolith with an embedded
Next.js UI and a PostgreSQL brain. You give it a target and a goal; a *planner* agent reasons about
the target, dispatches *intents*, and a pool of *worker* agents run real tools, capture real traffic,
and write verified findings back into a shared graph — which wakes the planner again. It runs until
the goal is proven or there's nothing left to try.

This folder takes it apart end to end — and asks the question that matters: **why is a custom
blackboard multi-agent engine better at this than wrapping a goal in a single-loop agent harness like
Claude Code or a Pi-style ReAct agent?**

## Read in this order

| # | Doc | What it answers |
|---|---|---|
| 00 | [`docs/00-overview.md`](docs/00-overview.md) | What ARTEX is, the three forks studied here, and the mental model. |
| 01 | [`docs/01-architecture-reference.md`](docs/01-architecture-reference.md) | The full code-grounded reference: the blackboard loop, the dual graph, evidence binding, tools, resilience, safety. The backbone everything else points at. |
| 02 | [`docs/02-boot-and-launch.md`](docs/02-boot-and-launch.md) | What literally happens from `./artex` to a live UI: flags, config, schema apply, the recording proxy, the HTTP server, the self-update exit-code dance. |
| 03 | [`docs/03-input-to-findings-flow.md`](docs/03-input-to-findings-flow.md) | The end-to-end trace of one engagement: operator input → goals → scope → intents → workers → tools → captured traffic → verified finding → report. |
| 04 | [`docs/04-harness-comparison.md`](docs/04-harness-comparison.md) | ARTEX's engine vs a single-loop agent harness (Claude Code / Pi-style). The eight structural reasons the blackboard design wins, and the two places a harness is actually better. |
| 05 | [`docs/05-ui-walkthrough.md`](docs/05-ui-walkthrough.md) | The operator's view: dashboard, tasks, sessions, the live exploration graph, findings, assets, coverage, traffic. |
| 06 | [`docs/06-three-forks-compared.md`](docs/06-three-forks-compared.md) | Component by component across the three forks — what each does, and the good *and* bad of every design choice. |

## The source under study

[`refs/`](refs/) vendors the three repositories the docs cite (`.git` stripped; exact commits in
[`../NOTICE.md`](../NOTICE.md)):

- [`refs/original/`](refs/original/) — `Autumn-27/ARTEX`, the upstream engine.
- [`refs/BaaRai-ARTEX/`](refs/BaaRai-ARTEX/) — the strong mainline continuation (evidence, resilience, operability).
- [`refs/artex-ymh/`](refs/artex-ymh/) — the fork specialized for internal-network depth and OPSEC.

A `file:line` citation like `BaaRai-ARTEX/server/engine.go:854` resolves to `refs/BaaRai-ARTEX/server/engine.go`.
