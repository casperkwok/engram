# Mneme

**English** · [中文](README.zh.md)

**A plain-text memory format for LLM agents.**

**Not a memory *service* — a *format*.** No database, no embeddings, no daemon: it's plain text
you can grep, diff, and hand to any model or human. If you want semantic search over a memory
server, that's a different tool — this is the file format underneath.

`.mneme` stores the things an agent must remember **across sessions** — decisions, facts,
preferences, gotchas, goals — in a way a *probabilistic reader* (a language model) can read
and write cheaply and trust. It is plain text: any editor, any model opens it directly.

> Version **v0.1 (draft)** · Spec: [SPEC.md](SPEC.md) · Agent rules: [AGENT.md](AGENT.md) · License: Apache-2.0

## Why another format?
`md` / `json` / `org` were designed for **human readers** or **deterministic programs**.
An agent's long-term memory has different physics, and those constraints force a different shape:

| Constraint (specific to LLM memory) | What it forces |
|---|---|
| The reader is **probabilistic**; isolated fragments get misread | Every unit self-describes (id, state, gist) and is understandable out of context |
| **Reading is expensive, writing is cheap** (every read burns tokens) | Two layers: a tiny always-resident *spine*; full units expanded only on a cue hit |
| Memory **expires, gets overturned, decays** | Lifecycle (`state`/`conf`/`seen`) is first-class — it records the memory's *relation to current reality* |
| Recall is **semantic association**, not paths | Each unit carries `cue` keywords: "pull me up when the task is about these" |
| The **writer is also a model** (fears brackets/indentation, not labels) | Line-oriented, one semantic key per line, no nesting, self-healing |
| Memory must be **auditable** | Append-only content; to change, add a new unit and `supersede` the old — its `why` survives |

The central rule, unlike any document format:

> **Content is immutable; state is mutable.**
> A unit's `gist`/body/`cue` is a fact about the past — never edited.
> Its `state`/`seen`/`conf` is its relation to *now* — updated as reality moves.
> Want to change content? Create a new unit that supersedes the old. Auditable *and* fresh.

## Two layers
- **Cell** — the atom of memory: one decision / fact / preference, self-contained.
- **Spine** — a one-line-per-cell projection (id + state + gist), **derived, never hand-written**.
  The agent keeps the cheap spine resident, then expands only the few relevant cells.

## A cell at a glance
```
@ DEC-0042  storage/cache
gist  Local cache uses SQLite instead of JSON files
state live   conf high   since 2026-06-03   seen 2026-06-03
cue   concurrent writes / cache corruption / slow queries / local storage choice
> JSON concurrent writes corrupt; queries need a full load. SQLite: single file, transactions.
> Cost: one more dependency, need a migration script.
link  supersedes DEC-0031
```

## Quick start
```bash
# derive the spine (what the agent keeps resident)
python tools/mneme_tool.py spine examples/example.mneme

# health-check a file (dup ids, broken supersede links, stale cells, bad dates)
python tools/mneme_tool.py lint examples/example.mneme

# allocate the next collision-free id
python tools/mneme_tool.py new-id DEC examples/example.mneme
```
Drop the rules in [AGENT.md](AGENT.md) into your agent's system prompt / `CLAUDE.md`.
Wire `tools/mneme_tool.py lint` into a pre-commit hook so lifecycle metadata can't silently rot.

## Status
v0.1 draft — born from real use, meant to be shaped by real use. Field set, state machine and
relation types may evolve.

## License
Licensed under the **Apache License 2.0** — permissive, with an explicit patent grant so the
format can be freely implemented. © 2026 Casper Kwok. See [LICENSE](LICENSE) and [NOTICE](NOTICE).
