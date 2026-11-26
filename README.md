# Curious

> Reverse-engineering and deep technical breakdowns of systems I find interesting — how they
> *actually* work, read from the source, not the README.

Each subfolder takes one system apart end to end: how it boots, how data flows through it, the
architecture decisions that make it good (or bad), and how it compares to the alternatives.

## Index

| System | Folder | What it is |
|---|---|---|
| **ARTEX** | [`artex/`](artex/) | An autonomous, multi-agent penetration-testing engine (Go + Next.js + PostgreSQL). A deep look at *why* a custom blackboard multi-agent engine beats a single-loop agent harness, how a target flows from input to a verified finding, the UI, and how three forks of the project diverge. |

## How to read a breakdown

Every breakdown folder contains:

- **`docs/`** — the written teardown, numbered in reading order. Grounded in the real source with
  `file:line` citations. High-level mental model down to low-level mechanics, with the flows in between.
- **`refs/`** — the actual source of the system(s) under study, vendored so the citations are
  checkable without hunting down the right commit.

## Licensing

The writing in `docs/` is mine — see [`NOTICE.md`](NOTICE.md). Vendored
third-party source under any `refs/` folder keeps its **own upstream license** (ARTEX and its forks
are AGPL-3.0); their `LICENSE` files travel with them. See [`NOTICE.md`](NOTICE.md) for exact
provenance (repos and commit hashes).

## A note on intent

These are **defensive and educational** teardowns of public, open-source code — systems design,
orchestration, and verification mechanics. They describe *how the machine is built and why the
design is effective*, not operational attack tradecraft. The systems documented here restrict their
own use to targets you own or are explicitly authorized to test; that restriction applies to anyone
acting on anything learned here.
