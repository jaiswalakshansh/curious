# 05 · The console — what the operator actually sees

ARTEX's frontend is a **Next.js SPA built on shadcn/ui**, embedded in the Go binary via `go:embed` and
served same-origin on `:8787`. Structurally it's a thin layer: a handful of real routes
(`web/src/app`) — `(auth)/login`, `(auth)/setup`, `(main)/dashboard`, `(main)/chat` — with the rich
tooling rendered as **client-side sections inside the dashboard shell**, fed by the JSON API and live
**Server-Sent Events**. This doc maps each section to the machinery underneath it, so the UI reads as a
window onto the graph rather than a separate thing.

## First run — setup & login

`(auth)/setup` bootstraps the first operator account; `(auth)/login` issues the **JWT** every later API
call carries. There's no multi-tenant SaaS here — it's a self-hosted console for one team, which is why
auth is a thin gate rather than an identity platform.

## The sections (all live over SSE)

The sidebar/nav exposes these views (labels pulled from `web/src/navigation` + components):

### Dashboard / Overview
The landing view: campaign totals, **token consumption**, and a live **activity stream** (the
README calls it 仪表盘：总览 / Token 消耗 / 活动流). The activity stream is the human-readable projection of the
engine's `activity` table — the same `round` markers, intent claims, tool calls, and write-backs the
planner reasons over. Token/usage comes from the `llm_usage` ledger (`llmrec/`), which is also what
budget accounting reads after a restart. This is where you watch the loop breathe.

### Tasks
Create and manage engagements. Creating a task here is the **input event** from
[`03-input-to-findings-flow.md`](03-input-to-findings-flow.md): it writes the task row and spawns the
planner loop + N workers + claim-sweep. Each task shows status (`running / paused / done`), its goals,
and controls to pause/resume/kill — which map to the engine's guard checks (`paused/deleting/terminal`)
and to `kill_work` / `steer_work`.

### Sessions
The **execution view** — the transcript of individual agent runs: the worker's conversation, every tool
call and its output (执行过程 · 会话 / 工具调用). This is the norma `transcript` surfaced verbatim, so you can
see exactly what a worker did, step by step, including the Bash it ran and what came back. It's the
operator's audit trail and the raw material the reporter agent cites.

### Findings
Confirmed vulnerabilities. Each finding carries its **Proof-of-Exploit level** (L1–L4), its verification
status, the bound **traffic evidence**, the generated Markdown report, and retest verdicts
(`reproduced / fixed / inconclusive`). This view is the payoff of the whole evidence-binding layer
([`01-architecture-reference.md`](01-architecture-reference.md) §4): what you see here already passed the
server's proof gates, so a finding in this list is one the platform itself stands behind.

### Assets
The **global asset graph** as a browsable inventory: `root_domain / subdomain / ip / service / app /
endpoint`, with tech stack, ports, params, and auth state. Because the asset graph is *global and shared
across tasks*, this inventory accretes across engagements — knowledge from one run is visible in the
next. Rows here are the `assets` table (`db/schema.sql:41`), deduped and hierarchized by the DB itself.

### Companies
The scoping/grouping layer above assets (`companies` + `company_scope`): assets belong to a company, and
scope is expressed per company. This is what keeps multiple engagements' data from bleeding together in
the shared asset graph.

### Coverage (the exploration graph, visualized)
The signature view: a **force-directed graph of the exploration layer** — goals, intents, facts,
findings, hints — with **tested assets highlighted** and nodes you can collapse/expand (资产覆盖图 · 力导向布局 ·
已测高亮 · 节点折叠展开). This is the `exploration_nodes` / `_edges` / `_anchors` tables drawn as what they are: a
lineage graph. The collapse/expand mirrors **cold-digest compaction** — `digest` nodes fold quiet
branches, and expanding one restores the detail. The highlight is the **coverage ledger**: scope is the
denominator, anchored facts the numerator, so you can literally see how much of the authorized surface
has been touched.

### Traffic
Every HTTP exchange captured by the `:8788` recording proxy, searchable (`traffic_search` / `traffic_get`
/ `traffic_blob`), with a detail view that can **replay** a captured request. This is the evidence plane
made visible — the same store the agents query and bind findings against. When a finding says "proven,"
this is where the receipt lives.

### Workspace
A scratch/filesystem area for the agents' artifacts — `cmd-output/` spills, skill working files, uploads —
so large tool output and intermediate files don't bloat the graph or the context.

### Settings
Operator configuration: **LLM providers** (keys, model profiles, failover ranking for `llmpool`), the
**worker pool size** (the `workers` setting that decides N), **noise profile** (`quiet / balanced /
aggressive`), **budgets**, **intercept rules** (the tool-call approval/deny gate), **scope** and RoE
mode. Almost every knob the earlier docs mention as "a DB setting" is edited here — including, notably,
the intercept rules that are the *only* remaining destructive-action gate upstream
([`01-architecture-reference.md`](01-architecture-reference.md) §13).

### Chat (`(main)/chat`)
The human-in-the-loop channel: a `mainagent` / chat role you can talk to about a task, ask questions, or
steer. This is the one place the system is interactive rather than autonomous — messages here can inject
intents or guidance into a running task (`server/intent_intervention.go`).

## The mental model for the whole UI

Every section is a **projection of the same database**, updated live over SSE:

| UI section | Underlying table(s) / system |
|---|---|
| Dashboard | `activity`, `llm_usage` |
| Tasks | task rows + the engine goroutines |
| Sessions | norma `transcript` |
| Findings | `findings`, `finding_traffic_bindings`, `finding_retests` |
| Assets | `assets` (global) |
| Companies | `companies`, `company_scope` |
| Coverage | `exploration_nodes/_edges/_anchors` + `task_scope` ledger |
| Traffic | the capture store (go-mitmproxy) + `evidence/` |
| Workspace | the data dir / `cmd-output/` |
| Settings | provider pool, `workers`, noise, budgets, intercept rules, scope |

There's no hidden state in the frontend. The console doesn't *drive* the engine so much as **watch and
nudge** it — which is exactly right for an autonomous system: the operator's job is to scope it, launch
it, read its receipts, and pull the brake, not to click through each step. That's the UI expression of
the same thesis as [`04-harness-comparison.md`](04-harness-comparison.md): the machine runs the loop; the
human owns the goal and the guardrails.
