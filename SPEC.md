# Mneme — a memory format for LLM agents

**English** · [中文](SPEC.zh.md)

**Version v0.1 · draft** · © 2026 Casper Kwok · Apache-2.0

**In one line**: Mneme is a plain-text format for the **persistent memory** an agent produces
while it works — decisions, facts, preferences, gotchas, goals — that must survive across
sessions, written so a *probabilistic reader* (a language model) can read and write it cheaply
and trust it.

File extension: `.mneme` (short `.mn`). It is physically plain text with conventions — any
editor, any model opens it directly. Its "format" is not in the symbols but in the structure
forced out by the physics of remembering.

---

## 1. What it is

Mneme is not a document format, nor a database. Documents serve "to be read/rendered by humans";
databases serve "to be precisely queried by programs". Mneme serves a third thing: **long-term
memory that a probabilistic reader (a model) reads repeatedly and acts on**.

Its world has two layers:

- **Cell** — the atom of memory. One decision, one fact, one preference is one cell. Each cell
  carries its own identity, gist, lifecycle state, confidence, freshness, recall cues and body,
  and **is understandable on its own, out of context**.
- **Spine** — a minimal projection of all cells: one line per cell (id + state + gist). The spine
  is **not hand-written; it is generated from the cells**. It is the index the agent keeps
  resident in working memory: cheap, scannable at a glance, always "what do I know".

The agent does not pour the whole store into context. It keeps the spine resident, then expands
the few relevant cells on demand. This is the heart of the design.

---

## 2. Why it's built this way

Every design choice answers a physical constraint of LLM memory. `html` / `md` / `org` / `json`
never faced these — their reader is a human or a deterministic program, not a model.

| Constraint (specific to LLM memory) | What it forces |
|---|---|
| The reader is **probabilistic**, rebuilds meaning from patterns; isolated fragments get misread | Each cell carries identity, state, gist — self-explaining out of context; redundancy here is not waste, it's error correction |
| **Reading is expensive, writing is cheap**: every read burns tokens, a write happens once | Two layers. Keep the cheap spine resident; expand a full cell only on a `cue` hit. However large the store, only the spine + a few cells enter context |
| Memory **expires, gets overturned, decays**; documents are static, memory is not | `state` / `conf` / `seen` are first-class — the format records not just content but **the memory's relation to current reality** |
| Recall is **semantic association**, not paths | The `cue` line: each cell reports "pull me up when the task touches these topics", cooperating with retrieval instead of fighting it |
| **The writer is also a model**: models fear counting brackets and deep indentation, not semantic labels | Line-oriented syntax, one semantic key per line, no nesting, no reliance on exact indentation, self-healing when broken |
| Memory must be **auditable**: why it was decided must not be lost | Append-only content; to change, add a new cell and `supersede` the old — it survives together with its `why` |

The single most important rule, and what most sets it apart from any document format:

> **Content is immutable, state is mutable.**
>
> A cell's `gist` / body / `cue` records "what was known/decided at some moment" — a fact about
> the past, never edited.
> Its `state` / `seen` / `conf` records "this memory's relation to now" — updated as reality moves.
> Want to change content? Create a new cell that supersedes the old. This rule satisfies both
> "auditable" (history never lost) and "fresh" (state can update) — needs that seem contradictory.

---

## 3. Format specification

### 3.1 Anatomy of a cell

```
@ DEC-0042  storage/cache
gist  Local cache uses SQLite instead of JSON files
state live   conf high   since 2026-06-03   seen 2026-06-03
cue   concurrent writes / cache corruption / slow queries / local storage choice
> JSON concurrent writes corrupt; queries need a full load. SQLite: single file, transactions, incremental queries.
> Cost: one more dependency, need a migration script.
link  supersedes DEC-0031
```

Cells are separated by a blank line. A cell starts at an `@` line and runs to the next `@` line
or end of file.

### 3.2 Field reference

| Line head | Meaning | Required | Notes |
|---|---|---|---|
| `@ <ID>  <topic-path>` | cell head: identity + topic path | yes | `ID` is `PREFIX-NNNN`, globally unique, never changed, never reused. `topic-path` is a `/`-separated hierarchy (e.g. `storage/cache`) for human scanning and coarse filtering |
| `gist` | one-line essence | yes | single line, the distilled memory. **Promoted into the spine.** Must be self-contained — readable pulled out alone |
| `state` | lifecycle state | yes | see §3.3 |
| `conf` | confidence | no | `high` / `med` / `low`, default `med`. "How sure am I" |
| `since` | established date | yes | when this memory was first established |
| `seen` | last-confirmed date | yes | last time it was reviewed/reaffirmed as still true. **The decay signal** |
| `cue` | recall cues | recommended | `/`- or `,`-separated topics/keywords. The agent uses these to judge "should the current task pull me up" |
| `>` | body | no | free prose: rationale, detail, context. May span multiple lines |
| `link` | typed relation | no | `link <relation> <ID>`, repeatable. Relations in §3.4 |

`state conf since seen` are conventionally written on one line as `key value` space-pairs, so a
human reads the status at a glance. They may also be split across lines; parsing is unaffected.

### 3.3 States and lifecycle

| state | Meaning | In default working memory? |
|---|---|---|
| `proposed` | proposed, not yet authoritative | no (unless the task is exactly to evaluate it) |
| `live` | currently true, in effect. The default authoritative state | yes |
| `stale` | maybe expired, long un-reviewed; verify before relying on it | yes (but flagged with caution) |
| `superseded` | replaced by a newer cell, kept for history only | no |
| `retired` | fully dead (closed OKR, abandoned plan), kept for audit only | no |

Allowed transitions: `proposed → live`; `live ⇄ stale`; `live / stale → superseded` (replaced) or
`retired` (abandoned). Content (gist / body / cue) is never changed once written; transitions only
touch the status line.

### 3.4 Relation types (`link`)

`supersedes` / `superseded-by` (paired), `relates`, `depends-on`, `blocks`, `refines`. A
supersession should be marked on both sides: the new cell writes `link supersedes OLD`, the old
cell writes `link superseded-by NEW`.

### 3.5 ID prefix convention (extensible)

| Prefix | Type |
|---|---|
| `DEC` | decision |
| `OKR` | objectives & key results |
| `PREF` | user/project preference |
| `GOTCHA` | pitfall / trap / lesson |
| `FACT` | fact / constraint / environment info |
| `TODO` | todo (note: only long-lived open items belong here, not transient todos) |

Add a new prefix when you need a new type; the convention itself is open.

### 3.6 Definition of the spine

The spine is a **projection** of the cells, not hand-written. Generation rule: for each cell emit
one line `<ID>  <state>  <gist>  [topic-path]`; by default project only cells with
`state ∈ {live, stale}` (`proposed/superseded/retired` do not enter working memory).

```
DEC-0042   live   Local cache uses SQLite instead of JSON files   [storage/cache]
PREF-0007  live   Commit messages in Chinese, first line <= 30 chars   [style/commit]
GOTCHA-003 stale  Migration script fails on Windows paths   [storage/migration]
```

Generate it with one grep (catch `@` / `gist` / `state` lines) or `tools/mneme_tool.py spine`.
**The single source of truth is always the cells; the spine is always derived** — no double bookkeeping.

### 3.7 Informal grammar

```
file      := comment* cell (BLANKLINE cell)*
comment   := "#" TEXT NL                       # lines starting with # are ignored
cell      := header gistline statusline cueline? bodyline* linkline*
header    := "@" SP ID SP+ topicpath NL
gistline  := "gist" SP+ TEXT NL
statusline:= ("state" SP+ STATE | "conf" SP+ CONF | "since" SP+ DATE | "seen" SP+ DATE)+  (one or many lines)
cueline   := "cue" SP+ TEXT NL
bodyline  := ">" SP TEXT NL
linkline  := "link" SP+ REL SP+ ID NL
```

Parsing tolerance: the line-head semantic key is the anchor; unrecognized lines are ignored, not
errors; missing fields take defaults. This lets the imperfect text a model writes still be consumed robustly.

---

## 4. How an agent uses Mneme (core)

These are the protocols an agent follows at runtime — its standard operating procedure when
facing the memory store. A short version is in [AGENT.md](AGENT.md) (paste into a system prompt).

**Protocol A — Boot**: at session start, or when a task starts touching project background, **load
the spine first, do not read the whole file**. Hold the spine as a lightweight "what do I know" index.

**Protocol B — Recall**: before acting, match the current task topic against the spine's `gist`,
each cell's `cue` and `topic-path`. **Expand only the cells that hit.** Don't dump the whole store
into context. Recall generously (favor recall), then judge relevance from the content.

**Protocol C — Trust**: before relying on a recalled cell, check its state:
- `live` + `conf high` + recent `seen` → trust and use directly.
- `stale`, or `conf low`, or old `seen` → **do not take it for granted**. Verify first against the
  user or the current real source; once verified, update `seen` to today (raise `state` back to
  `live`, bump `conf`, as warranted).
- `proposed` → a pending item, not a conclusion; don't act on it unless the task is to advance it.

**Protocol D — Write**: when this work produces a piece of persistent info **a future session
should know**, create a cell:
1. Take the next id (recommended: `mneme_tool.py new-id PREFIX`, prevents collisions).
2. Write a **self-contained** `gist`.
3. Set `state` (just-settled → `live`; under deliberation → `proposed`), `conf`, `since = seen = today`.
4. Write `cue`: think through "under what future task topics should this be pulled up". This
   determines whether it can be found later — don't skip it.
5. Use `>` for the body: rationale, cost, context.
6. Add `link`s if related.
7. **Append to the end of the file.** Never casually edit an unrelated old cell.

The test: **next time I open a fresh session, would I get it wrong or redo work without this?**
Yes → write it.

**Protocol E — Supersede**: when correcting an existing belief — **do not edit the old cell's
content**. Create a new cell with the new content; set the old cell to `state superseded` and add
`link superseded-by NEW-ID`; the new cell adds `link supersedes OLD-ID`. If it's merely a
reaffirmation that it still holds → update `seen` to today (the only allowed change to an existing
cell, and it changes state, not content).

**Protocol F — Decay sweep**: `live` cells whose `seen` is past a threshold and un-reaffirmed →
set `stale`; clearly no-longer-relevant → `retired`. Best handed to a tool/hook, not left to
discretion (see §6).

**Protocol G — Conflict**: when two `live` cells contradict, prefer higher `conf`, then newer
`seen`; **and tell the user explicitly** rather than silently picking one. Then `supersede` the loser.

**Protocol H — Hygiene**: one claim per cell; keep `gist` self-contained; long content (full
docs, long code) goes external, the cell `link`s or references a path; an id, once assigned, never
changes and is never reused.

---

## 5. Full example

See [examples/example.mneme](examples/example.mneme). It demonstrates a supersession chain
(`DEC-0031` replaced by `DEC-0042`, history preserved), a preference, an OKR. The derived spine
contains only live/stale by default.

---

## 6. Tooling and hooks (don't rely on agent discretion)

`.mneme` is plain text, but two kinds of bookkeeping **will rot if left to a probabilistic reader**
and should be handed to deterministic tools:

- **ID allocation**: hand-counting "the next id" collides across multiple writers. Use
  `tools/mneme_tool.py new-id PREFIX` for deterministic allocation.
- **Lifecycle freshness (`seen`/`state`)**: a model won't proactively run decay sweeps. Use
  `tools/mneme_tool.py lint` to flag `live` cells whose `seen` is past the threshold (default >45
  days), and to validate supersede-link symmetry, dangling links, duplicate ids, bad dates.

`mneme_tool.py` (zero deps, stdlib):

```bash
python tools/mneme_tool.py spine  <file>        # derive the spine (live/stale)
python tools/mneme_tool.py lint   <file>        # health-check; exit code 1 on errors (for hooks)
python tools/mneme_tool.py new-id DEC <file>     # allocate the next unique id
```

**pre-commit hook**: wire `lint` into `.git/hooks/pre-commit` so it runs whenever a `.mneme` file
is committed — structural errors block the commit, decay/asymmetry only warn. This makes rot
**visible on every commit rather than silently accumulating**. The whole point: a probabilistic
reader can't be trusted to keep the books — let a machine guard them.

---

## 7. How humans write it

A human can write or edit cells directly — the syntax is designed to be easy for people too. Scan
the spine for the big picture; when you look closely at a cell, the body is plain prose under `>`.
The one discipline, same as the agent's: **revise beliefs by adding + superseding, not
overwriting**; reaffirm by touching only `seen`.

---

## 8. Non-goals and boundaries

**Good fit**: long-lived, multi-session agent memory; the accumulating memory of a cross-project
or always-on agent; teams that need to audit "why we decided this"; long-term memory shared by
multiple people/agents (solve id allocation first).

**Not a fit**:
- **Not a database**: not for high-frequency precise queries, transactions, large joins.
- **Not for large blobs**: long docs, full code live elsewhere; the cell keeps only a gist + link.
- **Not a scratchpad**: only the distilled things "worth remembering across sessions".
- **Not a replacement for code or docs**: it holds the *why* (decisions, constraints, lessons,
  preferences), not every *what*.
- Small short-lived projects: a markdown list with a status field is often enough; you may not
  need this format.

---

## 9. Known limitations (v0.1)

- **P3 · in-place status edits**: `seen`/`state` still need in-place editing, and a model may edit
  the wrong line. `lint` catches structural damage as a backstop; a sturdier solution (status
  changes as appended events, folded on read) is left to a later version.
- **Truly concurrent multi-writer**: sequential ids (`DEC-0043`) can still race under concurrent
  writes; for that, switch to random-suffix ids (`PREFIX-<rand>`) or per-writer prefix namespaces.
- These limitations are themselves best recorded as a `FACT` cell in the project's meta-memory —
  eat your own dog food.

---

## 10. Versioning of the spec itself

This manual describes Mneme **v0.1 (draft)**. It is a format born from real use and meant to be
shaped by real use. The field set, state machine and relation types may evolve.

## License

Apache License 2.0. © 2026 Casper Kwok. See [LICENSE](LICENSE) and [NOTICE](NOTICE).
