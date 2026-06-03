# Mneme

[English](README.md) · **中文**

**面向 LLM 智能体的记忆格式（纯文本）。**

`.mneme` 用来存放智能体在工作中产生、需要**跨会话保留**的持久记忆——决策、事实、偏好、踩坑、目标——并让概率性的读者（大模型）能低成本、可信任地读写。它是纯文本：任何编辑器、任何模型都能直接打开。

> 版本 **v0.1（草案）** · 规格：[SPEC.md](SPEC.zh.md) · 智能体规则：[AGENT.md](AGENT.md) · 许可：Apache-2.0

## 为什么再造一个格式？
`md` / `json` / `org` 的读者是**人**或**确定性程序**。而智能体的长期记忆有不同的"物理约束"，这些约束逼出一种不同的结构：

| 约束（LLM 记忆特有） | 逼出来的设计 |
|---|---|
| 读者是**概率性的**，孤立片段易误读 | 每个单元自带身份/状态/要旨，脱离上下文也读得懂 |
| **读贵写便宜**（每次读都烧 token） | 两层：常驻的廉价 *脊柱*；按 `cue` 命中才展开完整单元 |
| 记忆会**过期、被推翻、衰减** | `state`/`conf`/`seen` 是一等公民——记录记忆与"当下现实"的关系 |
| 召回是**语义联想**，不靠路径 | 每个单元自带 `cue`：聊到这些话题就把我捞出来 |
| **写它的也是模型**（怕括号缩进，不怕标签） | 一行一个语义键，不嵌套、坏了易自愈 |
| 记忆必须**可审计** | 内容只追加；要改就新增单元并 `supersede`，旧的连同 `why` 一起留存 |

核心原则，区别于一切文档格式：

> **内容不可变，状态可变。**
> 一个 cell 的 `gist`/正文/`cue` 是关于过去的事实，永不更改；`state`/`seen`/`conf` 是它与"当下"的关系，随现实更新。
> 要改内容？新建一个 cell 取代旧的。既可审计、又可保鲜。

## 两层结构
- **Cell（记忆细胞）**：记忆的原子——一条决策/事实/偏好，自包含。
- **Spine（脊柱）**：每个 cell 抽一行（id + 状态 + 要旨）的投影，**派生、不手写**。智能体常驻脊柱，按需展开相关 cell。

## 一个 cell 长这样
```
@ DEC-0042  storage/cache
gist  本地缓存用 SQLite 取代 JSON 文件
state live   conf high   since 2026-06-03   seen 2026-06-03
cue   并发写 / 缓存损坏 / 查询慢 / 本地存储选型
> JSON 并发写会损坏，查询得全量加载。改用 SQLite：单文件、事务、增量查询。
> 代价：多一个依赖，要写迁移脚本。
link  supersedes DEC-0031
```

## 快速开始
```bash
python tools/mneme_tool.py spine  examples/example.mneme   # 派生脊柱
python tools/mneme_tool.py lint   examples/example.mneme   # 体检（查重ID/断链/衰减/坏日期）
python tools/mneme_tool.py new-id DEC examples/example.mneme # 分配下一个唯一 ID
```
把 [AGENT.md](AGENT.md) 的规则贴进你智能体的系统提示 / `CLAUDE.md`；把 `lint` 挂进 pre-commit 钩子，让生命周期元数据不会悄悄腐化。完整规格见 **[SPEC.md](SPEC.zh.md)**。

## 状态
v0.1 草案——为真实使用而生、需被真实使用打磨。字段集、状态机、关系类型都可能随实践调整。

## 许可
**Apache License 2.0**——宽松、含显式专利授权，格式可被自由实现。© 2026 Casper Kwok。见 [LICENSE](LICENSE) 与 [NOTICE](NOTICE)。
