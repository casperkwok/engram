# Agent rules — drop into your system prompt / CLAUDE.md

把下面这段贴进智能体的系统提示或 `CLAUDE.md`，它就会按 Mneme 协议维护项目记忆。
Paste the block below into your agent's system prompt / `CLAUDE.md`. Full spec: [SPEC.md](SPEC.md).

```markdown
## Memory: Mneme protocol

Project memory lives in `*.mneme` files (format: SPEC.md). Rules:

- **Boot**: when project context is relevant, first read the *spine* of each .mneme
  (`grep '^@'/'^gist'/'^state'`, or `mneme_tool.py spine`). Don't read whole files.
- **Recall**: before acting, match the task topic against each cell's `cue` and `topic-path`;
  expand only the cells that hit.
- **Trust**: `live + conf high + recent seen` → use directly. `stale / conf low / old seen` →
  verify first, then update `seen` to today. `proposed` → not a conclusion, don't act on it.
- **Write**: when something is produced that a future session would be wrong/redo without —
  add a cell: next id (use `mneme_tool.py new-id PREFIX`), self-contained `gist`, set
  `state`/`conf`, `since = seen = today`, write `cue` (which future topics should recall it),
  `>` for the why, `link` as needed, **append to end**. Never silently edit an old cell.
- **Supersede**: to revise an old belief → new cell + old cell set `superseded` + mutual `link`.
  To merely reaffirm → only bump the old cell's `seen`. Content is never edited in place.
- **Conflict**: two `live` cells disagree → prefer higher `conf`, then newer `seen`, and tell
  the user explicitly.
- **Hygiene**: one claim per cell; gist self-contained; long content external (link only);
  ids never reused. Allocate ids with `mneme_tool.py new-id`; let a pre-commit hook run
  `mneme_tool.py lint` so lifecycle metadata can't silently rot.
```

中文要点：**先读脊柱、按 cue 展开、用前看 state/seen、改认知靠新增+supersede、内容永不就地改、ID 用 new-id 分配、提交时 lint 把关。**
