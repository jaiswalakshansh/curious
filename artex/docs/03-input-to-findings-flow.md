# 03 · Input → findings — one engagement, end to end

This is the doc that answers *"what happens when I add an input?"* We follow a single target from the
moment you type it into the console to the moment ARTEX emits a verified, written-up finding. Every
stage cites the real code.

## The one-line version

> **Operator goal → goal decomposition → scope → planner intents → worker claims one → worker runs real
> tools through the capture proxy → writes assets/facts/findings back to the graph → that wakes the
> planner → repeat until a finding *proves* the goal (and the server agrees) → reporter writes it up.**

Now the long version.

---

## Stage 1 — you create a task

In the UI you create a **task**: a free-text objective plus a scope (a domain, an IP, a CIDR). The SPA
POSTs it to the JSON API, and `server.Manager` creates the task row and **spawns the per-task
goroutines** (`original/server/engine.go:763`, `BaaRai-ARTEX/server/engine.go:854`):

- one **planner loop**,
- **N worker loops** (default 3, the `workers` setting — `server/manager.go`),
- a **claim-sweep janitor** (BaaRai).

Nothing is scripted. From here the system is a feedback loop; your input was just the seed.

## Stage 2 — the goal decomposer turns prose into structure

Before any planning, a **goals agent** runs (`original/agent/goals.go`). It reads your objective and
emits structure via tools:

- **`set_goals`** — extracts the *final deliverable outcomes* (not steps). "Find exploitable issues"
  becomes concrete goal nodes.
- **`set_constraints`** — extracts operator rules into `task_constraints(kind IN ('allow','deny'))`
  (`original/db/schema.sql:287`). These become the **highest-priority red lines**, injected into the
  planner/worker system prompt *every round* (`constraintBlock`, `planner.go:395`).
- **minimal scope** — `goalsScopeTail` enforces registering *exactly* the host you named and not
  silently widening a subdomain to its root (`original/agent/goals.go:53`). Scope becomes the
  **coverage denominator** later.

Output: a few `goal` nodes (state `open`), constraints, and a tight scope. The exploration graph now
has a reason to exist.

## Stage 3 — the planner's first round

A graph change (the new goals) wakes the planner. It **debounces 800 ms** to coalesce the burst
(`engine.go:809`), then runs **one LLM turn** (`agent/planner.go:355`). The model is handed a rendered
overview of the whole graph as its *user* message and must respond with tool calls.

The **hard-floor rule** is injected every round (`planner.go:461`): *if the goal is unmet and there are
no open or running intents, this round MUST produce at least one intent — 0 intents = the task stalls.*
So on round one it is forced to generate directions. It calls **`add_intent`** — BaaRai batches up to
**4** highest-value directions in one call, each carrying:

- `summary` — what to explore,
- `asset_ids` — which assets it targets (for coverage + lineage),
- `parent_ids` — lineage back to the goal.

The planner's doctrine shapes these: **depth over coverage** (drive a high-value entry all the way
through), and **2–3 mechanistically different routes in parallel** (e.g. "attack the upload chain" vs
"attack auth bypass") so it doesn't tunnel-vision (`planner.go` prompt, translated in
[`01-architecture-reference.md`](01-architecture-reference.md) §6).

The new `intent` nodes land in the graph with state `open`.

## Stage 4 — a worker claims exactly one intent

Each worker loop polls the **frontier** (`SELECT … kind='intent' AND state='open' ORDER BY priority
DESC`) and tries to **claim** one. The claim is an **atomic compare-and-set**
(`original/db/exploration.go:1173`):

```sql
UPDATE exploration_nodes SET state='running', owner=$1 WHERE id=$2 AND state='open'
```

Only one worker's `UPDATE` matches `state='open'` — that's the entire mutual-exclusion mechanism, no
locks. BaaRai wraps this in a **fencing-token lease with a heartbeat**, so a hung worker's claim is
swept back to `open` and retried (`BaaRai-ARTEX/server/engine.go:1067`).

## Stage 5 — the worker executes against the real target

The claiming worker runs an LLM agent turn with a **mandate pinned in its system prompt**: *you have ONE
intent; complete it, write discoveries back to the graph, then stop* (`agent/worker.go:230`). It runs
with `PermissionMode: ModeBypass` (`worker.go:436`) — **tools auto-execute, no human approval.** Its
toolbox:

- **Bash** — the real offensive surface: `nmap`, `httpx`, `ffuf`, `nuclei`, `sqlmap`, `curl`, custom
  scripts. Every HTTP call is routed through the `:8788` capture proxy, and the proxy CA is trusted by
  every toolchain (`worker.go:250`), so it's all recorded as evidence-grade traffic.
- **Graph tools** — `insert_assets`, `record_fact`, `report_finding`, `list_assets`, `list_findings`.
- **Trace tools** — `search_all_worker_traces` / `get_worker_trace`: it can read what *other* workers
  saw, so a hidden parameter or error message another worker noticed becomes reusable here.
- **Traffic tools** — `traffic_search` / `traffic_get` / `traffic_blob` to query captured exchanges.
- **Skills / MCP / custom tools** — portable methodology packages and external tool servers.

As it discovers things it **writes back immediately** — *"only what's in the graph counts"*:

- a new host/endpoint → `insert_assets` (the DB auto-registers the host's root_domain + subdomain and
  dedups via partial unique indexes — `original/db/assets.go:784`);
- an observation → `record_fact` (BaaRai ties the fact's evidence to recorded activity; see Stage 7);
- a confirmed vulnerability → `report_finding` with `traffic_refs` pointing at the proving exchange.

The worker is told a first obstruction (a filtered payload, a 404, a blind injection with no echo) is
**not** exhaustion — *run every bypass for this intent before concluding* (`worker.go:240`).

## Stage 6 — the worker finishes; the loop closes

When the worker returns, the engine classifies the outcome into a **semantically rich terminal state**
(`engine.go:1130` / `BaaRai-ARTEX/server/engine.go:1390`):

- `done` — direction covered;
- `exhausted` — hit a turn/time budget; *tried-but-unfinished*, so the planner should retry it;
- `blocked` — external error.

Then `t.NotifyDone(intentID)` **wakes the planner** with the finished intent's id (`engine.go:1155`).
The planner re-reads the graph — now richer by whatever the worker wrote — and plans again, reacting to
what was just found. A credential the worker dumped reshapes the next intents. *This is the heartbeat.*

```
 you ──objective──▶ [goals agent] ──goals+scope──▶ exploration graph
                                                      │ (wake)
                                                      ▼
                            ┌────────── [PLANNER] add_intent ≤4 ──────────┐
                            │                                             │
   NotifyDone(intent) ◀─────┤                                             ▼
        (wake)              │                                    intent(open)
                            │                                             │ CAS claim
                            │                                             ▼
                            └──── write assets/facts/findings ◀── [WORKER] runs tools
                                        back to graph               curl/nmap/sqlmap
                                                                    └▶ :8788 capture proxy
                                                                         └▶ immutable evidence
```

## Stage 7 — proving the goal (the part that makes it honest)

Eventually a worker files a `report_finding` strong enough that the planner tries to close a goal with
**`prove_goal`**. This is where ARTEX refuses to take the model's word. BaaRai runs **three independent
server-side checks** (detailed in [`01-architecture-reference.md`](01-architecture-reference.md) §4):

1. **Host-matched live capture** — the finding's bound traffic must still exist in the capture store
   *and* its URL host must match an in-scope asset (`db/finding_traffic.go:600`). Fail-closed.
2. **Verbatim-quote receipt** — Proof-of-Exploit level **≥ L3**, and an evidence quote (≥ 8 runes) must
   appear **verbatim** in the proof response body — which the server reads and **gzip/deflate-decodes**
   itself before checking (`server/finding_verification.go:140`). L3 means *data was actually extracted*,
   not "looks injectable."
3. **Fact provenance** — if the quoted bytes were seen coming *back from the target* through the proxy,
   the source is force-downgraded to `target_response` and **cannot prove a goal** (`agent/tools.go:1878`).
   `prove_goal` refuses `unverified` facts and `target_response` sources.

Only if all three pass does the goal flip to `met`. Marking the last open goal **auto-completes the
task** (`BaaRai-ARTEX/agent/tools.go:1476`), and the engine cancels in-flight workers with
`AbortGoalMet`.

## Stage 8 — negative results and breadth

Not every intent finds a bug, and ARTEX treats that as information, not silence. BaaRai's
`record_coverage` ledger logs per-surface outcomes (`reported / no_issue_found / ruled_out / …`) *"so an
absent finding is never read as a clean target"* (`agent/coverage_ledger.go`). The planner can query
`list_untested_assets` and push **outward** toward unexplored in-scope surface. A precise clean negative
counts as real work — which is why the loop fans out instead of grinding one path forever.

## Stage 9 — staying alive on a long run

If the engagement is long, three mechanisms keep it from collapsing:

- **Cold-digest compaction** folds quiet branches of the graph into `digest` nodes so the planner's
  context stays bounded, with `expand_digest` to restore detail (`agent/compaction.go`).
- **Budgets** (if enabled) cap task/intent/round token+turn spend and force a *fact-preserving* wrap-up
  instead of a hard kill (`agent/budget.go`).
- **Provider failover** rotates across LLM providers with a circuit breaker so an outage doesn't end the
  run (`llmpool/pool.go`).

## Stage 10 — the report

`report_finding` triggers a separate **reporter agent** (`original/agent/promptcatalog.go`). It pulls
the full evidence, the worker trace of *how* it was found, and the bound traffic, then writes a Markdown
report — under a hard rule: *every line must be traceable to evidence or the execution trace; never
fabricate requests, responses, CVEs or conclusions.* A **retester** agent can later re-run the finding
and mark it `reproduced / fixed / inconclusive`.

---

## The whole thing, as a sentence per actor

- **You:** give a target and a goal.
- **Goals agent:** turns that into goal nodes, constraints, and a tight scope.
- **Planner:** every time the graph changes, dispatches up to 4 target-anchored intents; never scripts.
- **Workers (×N):** each claim one intent, run real tools through a recording proxy, write back, stop.
- **The graph (PostgreSQL):** the shared, durable truth that both separates and connects all of them.
- **The server (not the LLM):** decides when a finding is real, by re-checking captured bytes.
- **Reporter / retester:** write it up and re-verify it.

That division of labor — *plan vs do vs remember vs verify*, each owned by a different component — is the
reason ARTEX can run an open-ended engagement autonomously without losing the thread. The next doc,
[`04-harness-comparison.md`](04-harness-comparison.md), shows what you lose when you collapse all four of
those jobs into one agent in a loop.
