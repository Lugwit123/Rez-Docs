# l_notepad 知识库检索优化计划（2026-09-28 体检 + 整改）

> 本文是一次**实测体检报告 + 整改方案**。每条问题都带复现命令与实测数字，方案都给到可落地的代码位置。
> 被检对象：`l_notepad_server`（`127.0.0.1:8765`，SQLite + FTS5 + bge-m3 向量 + bge-reranker-v2-m3 重排）与
> 它的消费方 `l_agent_chat`（`notepad_search` / `notepad_read` 工具）。
>
> 复现脚本：`d:/Temp/kb_audit.py` ~ `kb_audit7.py`（本文每张表都注明了对应脚本）。
>
> **更新（2026-09-28 当天）**：本方案已落地并**复核过一轮**，实测结果、与预期的偏差、计划外发现见
> **§9 实施结果**（完整基线表：top-1 20% → **73.3%**，recall@5 93.3%，docs 17.3k → 2.5k，
> **重排最终默认关**）。§0~§8 保持体检与方案原样，便于对照。
>
> 复核（2026-09-28 12:00）订正了本节此前三处失实：**DB 没变小**（是 GB/GiB 换算，见 §9.2）、
> **有 1 条是真召不回**而非「都排在第 2~3 名」（见 §9.1）、**硬预算不是墙钟上限**（见 §9.5）。

---

## 0. 结论速览

| # | 问题 | 严重度 | 一句话 | 修完预期 |
|---|------|--------|--------|----------|
| P0 | 重排喂错料，把正确答案踩出前三 | **致命** | 送给 cross-encoder 的是 **18 字的标题块**，而重排分是**唯一排序主序** | top-1 命中 1/5 → 5/5；查询 1.2s → 0.35s |
| P1 | 文档语料只有 40 篇，仓库一半知识没入库 | 高 | `AGENTS.md`、`.cursor/rules/*.mdc`、`wuwo/*.md`、`Doc/`、`openspec/` 全在库外；`default` 库 0 篇 | 可检索文档 61 → ~120 篇，覆盖运维/启动器/规则 |
| P2 | 85% 的索引给「默认不搜」的库做嫁妆 | 中 | 14,672 / 17,325 docs 属 `default_off` 库；DB 1.29 GB；每次预热 9s | DB ≈ 0.4 GB，预热 ≤ 3s |
| P3 | `auto` 档位形同虚设 | 中 | 判据是「命中数 ≥ 3」，bigram OR 召回下永远成立 → **从不**回退语义 | 零召回/低分查询真正有语义兜底 |
| P4 | 尾延迟没封死 | 中 | 抽样首查一次 **>180s 无响应**（稳定态 1.2s） | 硬预算，最坏 1.5s 返回 `degraded` |
| P5 | 没有离线评测集，分数也无量纲 | 高（元问题） | `score` 在 11~95 间飘；P0 这种退化上线三周无人发现 | 一条命令出 nDCG@5 / Recall@5，回归可拦 |

**一句话**：管道（增量索引、FTS5、块级向量、重排、来源过滤、包过滤）已经做到同类自研方案里相当完整的程度，
**但没有评测闭环** —— 于是一个「喂料 bug」让最贵的那一环（重排）变成了净负收益，而且没人知道。

---

## 1. 现状画像（实测，`kb_audit.py` / `kb_audit2.py`）

```
docs                17,325        fts_rows 17,325   fts_consistent true
db_bytes            1,292,988,416  (1.29 GB)
vec                 bge-m3 @ :11434   docs 2,595 / chunks 100,507 / dim 1024
  by_source         code 2,476 docs / 99,204 chunks   kb 40 / 1,018   note 19 / 285
rerank              bge-reranker-v2-m3-Q8_0.gguf @ :11435   enabled=true  top_n=8  timeout 8s
sources             note   21 docs   (last_indexed 09-23, newest_mtime 09-03)
                    kb     default  0 docs      ← 空库
                    kb     rez_pkg  40 docs / 1.09 MB
                    code   54 个本机库 / 全部已建索引
history             最近 5 条全是 `warm/startup`，各 9.0~9.9s（热重载重启频繁）
```

本机库 docs 分布（前 6 占 85%，且**全部**标了「默认不搜」）：

| docs | 库 | 默认不搜 |
|---|---|---|
| 5,346 | ChatRoom | ✅ |
| 3,567 | postgresql | ✅ |
| 3,178 | l_comfyui_frontend | ✅ |
| 1,530 | l_comfyui_ai | ✅ |
| 1,051 | l_mayaPlug | ✅ |
| 100 | conemu | ✅ |
| 512 | Lugwit_Module | |
| …余 47 个库合计 ~2,000 | | |

检索档位实测（同一查询 `服务卡片 怎么重启`，`kb_audit3.py`）：

| 档位 | 耗时 | 备注 |
|---|---|---|
| `mode=lex&sources=note,kb&rerank=0` | **199 ms** | 纯词法 |
| `mode=lex&sources=note,kb` | 1,231 ms | 开重排（默认） |
| `mode=auto&sources=note,kb` | 1,201 ms | `mode_used=lex`（未回退） |
| `mode=hybrid&sources=note,kb` | 7,504 ms | 走 ollama bge-m3 |
| `/api/search/route` | 160 ms | 选库快路径 |

---

## 2. P0 — 重排喂错料，把正确答案踩出前三【致命】

### 2.1 症状（A/B 对照，`kb_audit5.py`）

五条真实提问，`rerank=0` vs `rerank=1`，**4 变差、1 持平、0 变好**：

| 提问 | `rerank=0` 第一名 | `rerank=1` 第一名 | 判定 |
|---|---|---|---|
| 变体哈希 空壳包 | `Rez_pkg/变体哈希与wuwo的处理方法.md` (score 62.96) | `README.md` | 正确文档**掉出前三** |
| 单实例守卫 端口被占 | `solo_单实例守卫模式.md` (95.07) | `src_hot_reload_源码热重载与主页常驻.md` | 正确文档**掉出前三** |
| 相册 回收站 数据结构 | `相册功能与数据模型.md` (66.08) | `Rez_pkg/lugwit_baidu_netdisk.md` | 换成无关文档 |
| wuwo 启动报错 找不到包 | `Rez包创建和启动指导文档.md` (53.69) | `Rez_pkg/wuwo_gui.md` | 次优 |
| 热重载 改了源码不生效 | `l_agent_chat使用指南.md` | 同上 | 持平 |

同时**耗时涨 4~6 倍**：200~310 ms → 900~1,250 ms。即：查询 **83% 的时间花在重排上，买到的是更差的排序**。

### 2.2 先排除「模型不行」

直接打 `:11435`：

```
GET  /health      → {"status":"ok"}
GET  /v1/models   → models\bge-reranker-v2-m3-Q8_0.gguf
POST /rerank  {"query":"单实例守卫","documents":["solo 单实例守卫模式说明","无关文本"]}
              → [{index:0, relevance_score: 3.417}, {index:1, relevance_score: -6.309}]
```

模型是正牌 cross-encoder，判别力正常。**问题不在模型。**

> 补充：`/api/search/stats` 里 `rerank.model` 显示为空串 —— 那是因为 `L_NOTEPAD_RERANK_MODEL` 没设、
> 服务端把「模型名交给 :11435 自己决定」。不是故障，但状态页看不出实际模型，建议回填 `/v1/models` 的结果。

### 2.3 根因：送进去的是**标题块**（`kb_audit7.py`）

把 `变体哈希 空壳包` 的 8 个候选连同「实际送去重排的 chunk」全打出来：

```
1. score=32.66  rerank= 1.129  chunk_no=1   README.md
   chunk[895] = '| 文档 | 一句话摘要 |…| [Rez_pkg/变体哈希与wuwo的处理方法.md](…) | rez-pip 变体目录名含
                 Windows 非法字符导致"空壳包"的问题,以 reflex 为完整案例 + 排查手册 |…'
2. score=11.42  rerank=-0.391  chunk_no=16  Rez_pkg/l_homepage.md
   chunk[30]  = '## 局部刷新（Jinja block + 哈希 diff）'
6. score=62.96  rerank=-3.632  chunk_no=0   Rez_pkg/变体哈希与wuwo的处理方法.md
   chunk[18]  = '# 变体哈希与 wuwo 的处理方法'          ← 只有标题！
```

三层因果，缺一不可：

1. **块选择退化成标题块**。`_best_chunk()`（`search_index.py:3013`）按「短语命中数 > 覆盖率/词频」挑块，
   而一个 18 字的标题块里「变体哈希」的**覆盖率是 100%**，必然压过 900 字的正文块。
   **缺长度归一** —— 这是 BM25 里靠 `b` 参数解决的经典问题，自研打分函数最常漏的一条。
2. **喂给 cross-encoder 的料没有上下文**。18 字标题送进 bge-reranker → `-3.63`；
   而 README 那张「文档索引表」恰好把「变体哈希」和「空壳包」写在同一行 → `+1.13`，**它赢了**。
   对照实验：把**文档路径/标题**送去重排，排序立刻正确 ——
   `变体哈希与wuwo的处理方法.md -1.599` 排第一，`README.md -8.894` 垫底。
   → **模型没错，料错了。**
3. **重排是唯一排序主序**。`_apply_rerank()`（`search_index.py:3126`）的 `order_key` 只用
   `sigmoid(rerank) × prior`，`score` 仅作平手次键。于是一个 62.96 分（第二名 32.66，领先 1.9 倍）的
   词法冠军被一个坏窗口的 -3.63 直接踩到第 6 —— **强词法信号被完全丢弃**。

### 2.4 行业先进做法

| 做法 | 谁在用 | 对应本问题 |
|---|---|---|
| **RRF 融合名次**（`Σ w/(k+rank)`，Cormack 2009） | Elasticsearch 8.x `rrf` retriever（`rank_constant`/`rank_window_size`）、OpenSearch hybrid、Weaviate、Qdrant | 重排**不独占主序**，与词法名次融合。本仓已有 `_rrf_fuse()`（中英双路用），只是没用在重排上 |
| **Phased ranking**（一阶段便宜召回、二阶段只在窗口内重排，且有 `rank-score-drop-limit`） | Vespa | 重排只能在窗口内**微调**，不能把一阶段冠军扫出结果页 |
| **Parent-document / auto-merging retrieval**（小块检索、大块喂模型） | LlamaIndex `AutoMergingRetriever`、LangChain `ParentDocumentRetriever` | 标题块只用于**定位**，喂重排/LLM 时换成父块 |
| **Contextual Retrieval**（给每块前置文档级上下文再嵌入/重排；官方实测检索失败率降 35%，配重排降 49%） | Anthropic 2024 | 正是「18 字标题块无上下文」的标准解药 |
| **Rerank 输入带标题前缀、目标 ~512 token** | Cohere Rerank 最佳实践、BGE-reranker 官方示例 | `_trim_for_rerank` 应前置 `rel`/标题 |
| **nDCG@k / Recall@k 离线评测**（BEIR、Ragas） | 所有检索系统 | 见 P5 |

### 2.5 整改方案（三步，按收益排序）

#### 第 0 步（立即，可逆）：先把重排关掉，止血

```bash
curl -X POST 127.0.0.1:8765/api/search/rerank -H "Content-Type: application/json" -d "{\"enabled\": false}"
```

立即效果：查询 1.2s → 0.2~0.3s，top-1 正确率 1/5 → 5/5（实测 `rerank=0` 那一列）。
修好 2.5 的 (a)(b)(c) 后再打开。

#### (a) 喂料带上下文：标题前缀 + 短块补齐父块

`search_vec.py`，改 `_trim_for_rerank` 的调用点（`rerank_docs` 里那一行 list comprehension）：

```python
# search_vec.py
RERANK_MIN_CHARS = _env_int("L_NOTEPAD_RERANK_MIN_CHARS", 200)   # 低于此视为「没上下文」


def rerank_input(query: str, hit: dict) -> str:
    """组装送给 cross-encoder 的文本：**标题前缀 + 有上下文的正文窗口**。

    为什么必须这么做（2026-09-28 实测）：`变体哈希与wuwo的处理方法.md` 被选中的块是
    `chunk_no=0`、正文只有 18 字的标题行 —— bge-reranker 给 -3.63，而 README 里一张
    「文档索引表」因为同时含「变体哈希」「空壳包」拿了 +1.13，正确文档被踩到第 6。
    对照实验里只把**文档路径**送去重排，正确文档立刻回到第 1（-1.599 vs README -8.894）。
    """
    title = str(hit.get("rel") or hit.get("path") or "")
    body = str(hit.get("chunk") or "")
    if len(body) < RERANK_MIN_CHARS:
        # 标题/小标题块：用同篇相邻块补足上下文（parent-document 的轻量版）
        body = doc_window(hit, min_chars=RERANK_MIN_CHARS * 3)
    return f"{title}\n{_trim_for_rerank(body, query)}"
```

`rerank_docs` 的签名从 `documents: list[str]` 改为接 hits（或由 `_apply_rerank` 先组装好字符串传入，
改动面更小）：

```python
# search_index.py  _apply_rerank 内
scores, sub = search_vec.rerank_docs(
    conn, query, [search_vec.rerank_input(query, h) for h in cand]
)
```

#### (b) 排序改 RRF 融合，别让重排独占主序

```python
# search_index.py
_RRF_K = 60.0            # Elasticsearch rank_constant 默认值，同一量级
_W_LEX, _W_RERANK = 1.0, 1.0


def _fuse_lex_rerank(cand: list[dict[str, Any]]) -> list[dict[str, Any]]:
    """词法名次 + 重排名次做加权 RRF（`Σ w/(k+rank)`），先验乘在融合分上。

    为什么不再用「sigmoid(rerank) × prior」当唯一主序：重排只看**一个 400 字窗口**，
    窗口选差一次就能把词法冠军（62.96，领先第二名 1.9 倍）扫到第 6 —— 实测踩过。
    RRF 只看名次、不要求量纲可比，一路失手时另一路仍能把它拉回前列。
    """
    lex = {id(h): i for i, h in enumerate(sorted(cand, key=lambda x: -float(x.get("score") or 0)))}
    rr = {id(h): i for i, h in enumerate(sorted(cand, key=lambda x: -float(x.get("rerank") or 0)))}

    def key(h: dict[str, Any]) -> tuple[float, float, str]:
        s = _W_LEX / (_RRF_K + 1 + lex[id(h)]) + _W_RERANK / (_RRF_K + 1 + rr[id(h)])
        return (-s * float(h.get("prior") or 1.0), -float(h.get("score") or 0), h["path"])

    return sorted(cand, key=key)
```

把 `_apply_rerank` 末尾的 `ranked = sorted(cand, key=order_key)` 换成 `ranked = _fuse_lex_rerank(cand)`；
`_annotate_ranks(hits, order_by=...)` 相应改成 `"fused"`（并把融合分写进 `explain`，「判断依据」面板才对得上）。

#### (c) 块选择加长度归一，别让 18 字标题块赢

```python
# search_index.py  _best_chunk
_CHUNK_FULL = 400        # 达到这个长度才算「有完整上下文」，更短的按比例打折


def _best_chunk(chunks, units):
    best_key = None
    ...
    for no, start, text in chunks:
        meta = _score(text.lower(), units, 0.0)
        # 长度归一（BM25 的 b 参数同理）：标题块覆盖率必然 100%，不打折就永远赢
        damp = min(1.0, len(text) / _CHUNK_FULL)
        key = (meta["phrase_hits"], float(meta["score"]) * damp)
```

#### (d) 护栏：一阶段的压倒性冠军不许被扫出结果页

```python
# search_index.py  _apply_rerank 末尾
_RANK_FLOOR_RATIO = 1.5      # 词法分领先第二名 1.5 倍以上者，保底进前 3

top = max(cand, key=lambda h: float(h.get("score") or 0))
rest = [float(h.get("score") or 0) for h in cand if h is not top] or [0.0]
if float(top.get("score") or 0) >= _RANK_FLOOR_RATIO * max(rest) and ranked.index(top) > 2:
    ranked.remove(top)
    ranked.insert(0, top)    # 对齐 Vespa 的 rank-score-drop 思路：二阶段只微调，不否决一阶段
```

### 2.6 预期结果

| 指标 | 现在 | 修完（预期） |
|---|---|---|
| 5 问 top-1 正确 | 1/5 | 5/5 |
| 正确文档进前三 | 3/5 | 5/5 |
| `sources=note,kb` 查询耗时 | 1.0~1.5 s | 0.6~0.9 s（重排开）/ 0.2~0.3 s（关） |
| 重排的边际收益 | **负** | 平手或微正（近义词、无字面重叠的查询） |

---

## 3. P1 — 文档语料只有 40 篇，仓库一半知识没入库

### 3.1 问题

- `kb` 只有两个库：`rez_pkg` **40 篇**（= `Rez-Docs/**/*.md` 全量，同步是好的）、`default` **0 篇（空壳）**。
- `note` 21 篇，`last_indexed 09-23`、`newest_mtime 09-03` —— 停更三周。
- agent 查「怎么做 / 以前怎么解决的」时，`sources=note,kb` 的可选范围**只有 61 篇**。
- 而仓库里这些高价值文档**全在库外**：

| 路径 | 内容 | 为什么该入库 |
|---|---|---|
| `AGENTS.md` | 仓库入口规则、建包硬规则、`wuwo svc` 命令表 | agent 最常需要的那一页 |
| `.cursor/rules/service-lifecycle.mdc` | 服务启停完整手册 | `AGENTS.md` 明确指向它，但它不可检索 |
| `wuwo/README.md`、`wuwo/doc/CHANGELOG.md`、`wuwo/重构.md` | 启动器本体与变更史 | 「wuwo 启动报错」类问题的一手答案 |
| `Doc/`、`openspec/` | 现有文档与规格 | 设计意图 |
| 各包 `rez-package-source/<pkg>/**/*.md` | 包级说明 | 现在只当**代码**索引（`source=code`），`sources=note,kb` 过滤时被排除 |

最后一条尤其别扭：包目录里的 `.md` 被当成 code 命中，于是「查文档」的正确姿势（`sources=note,kb`）
反而**搜不到它们**。

### 3.2 行业做法

- **按 doc-type 分层而不是按物理来源分层**：Elasticsearch/OpenSearch 用 `_index` + filter，
  Weaviate 用 class + tenant。本仓的 `source∈{note,kb,code}` 混淆了「物理来源」与「内容类型」。
- **仓库文档自动入库**：Cursor / Copilot 的 codebase index 对 `*.md` 与代码分桶；
  Danswer / Onyx、Glean 这类企业检索则用 connector 定时拉取各仓的文档目录。
- **知识库以「问题-答案」为单元沉淀**（踩坑记录、Runbook），而不是只放设计稿 —— 本仓 `Rez-Docs` 已经是这个路子，只是覆盖面窄。

### 3.3 整改

**(a) 立刻：把 `.md` 从 code 桶里升级成文档桶。** 在 `search_index` 写索引时按扩展名给 `doc_type`，
检索侧 `sources=note,kb` 顺带带上 `doc_type='doc'` 的 code 行：

```python
# search_index.py  写入 search_docs 时
DOC_EXTS = {".md", ".mdc", ".rst", ".txt", ".adoc"}

doc_type = "doc" if Path(rel).suffix.lower() in DOC_EXTS else "code"
# …写进 search_docs 新增列 doc_type（带 migration：ALTER TABLE ADD COLUMN doc_type TEXT DEFAULT 'code'）
```

```python
# 检索侧：sources 含 'kb' 或 'note' 时，额外放行 doc_type='doc' 的 code 行
if {"note", "kb"} & set(sources or []):
    where.append("(source IN (?,?) OR (source='code' AND doc_type='doc'))")
```

收益：`rez-package-source/**/*.md` + `wuwo/**/*.md`（已在 code 索引里）**立刻**变成可检索文档，
零新增扫描成本。

**(b) 把仓库根文档同步进 `rez_pkg` 知识库。** 一个幂等小脚本，挂到 `l_notepad_server` 的
`code_watch` 或 `wuwo svc` 的定时任务上：

```python
# tools/sync_repo_docs.py
"""把仓库根的规则/手册类文档同步进 Rez-Docs（知识库工作区）→ kb_worker 自动入索引。

为什么复制而不是加 code root：`AGENTS.md` / `.cursor/rules/*.mdc` 是**规则**，要被
`sources=note,kb` 查到；而 code root 里的东西属于 `source=code`，默认被文档类查询过滤掉。
"""
import shutil
from pathlib import Path

ROOT = Path(r"D:\TD_Depot\Software\Lugwit_syncPlug\lugwit_insapp\trayapp")
DEST = ROOT / "rez-package-source" / "Rez-Docs" / "_repo"
SRC = [
    ROOT / "AGENTS.md",
    ROOT / "wuwo" / "README.md",
    ROOT / "wuwo" / "doc" / "CHANGELOG.md",
    ROOT / "wuwo" / "重构.md",
    *(ROOT / ".cursor" / "rules").glob("*.mdc"),
]

DEST.mkdir(parents=True, exist_ok=True)
for src in SRC:
    if not src.is_file():
        continue
    dst = DEST / (src.name if src.suffix == ".md" else src.stem + ".md")
    if dst.exists() and dst.read_bytes() == src.read_bytes():
        continue                      # 幂等：内容没变不碰 mtime，免得触发无谓重嵌
    shutil.copyfile(src, dst)
    print("synced", dst.name)
```

**(c) 清理 `default` 空库**（要么删，要么明确它的用途），别让 agent 在 `/api/kb` 列表里看见一个 0 篇的库。

### 3.4 预期结果

| 指标 | 现在 | 修完 |
|---|---|---|
| `sources=note,kb` 可检索文档 | 61 篇 | ~120 篇（+ 仓库全部包级 `.md`） |
| 「wuwo 启动报错」类问题 | 只能命中二手转述 | 命中 `wuwo/README.md` / CHANGELOG 一手答案 |
| 「服务怎么重启」 | 命中计划文档 | 命中 `service-lifecycle.mdc` 手册 |

---

## 4. P2 — 85% 的索引给「默认不搜」的库做嫁妆

### 4.1 问题

`ChatRoom`(5,346) + `postgresql`(3,567) + `l_comfyui_frontend`(3,178) + `l_comfyui_ai`(1,530) +
`l_mayaPlug`(1,051) + `conemu`(100) = **14,772 docs**，占 17,325 的 **85%**，而它们**全部** `default_off=true`
（不显式传 `packages` 就不搜）。代价：

- `db_bytes` 1.29 GB（其中向量 100,507 chunks，`by_source.code` 占 99%）；
- 每次启动 `warm` 预热 **9.0~9.9s**，而热重载让这事一小时内发生了 5 次；
- 之前那个「TTL 全量比对现场 stat 1.7 万文件、首查 51.9s」的老问题，根子也是这批文件。

### 4.2 行业做法

- **索引准入白名单 + 分层**：Copilot / Cursor 的 codebase index 默认排除 `node_modules`、构建产物、
  vendored 依赖；Sourcegraph 用 `zoekt` 的 shard 分层，冷 repo 不进热 shard。
- **向量只嵌「会被搜的东西」**：向量是最贵的一层（存储 + 嵌入 + 查询时 ANN），主流做法是
  词法全量 + 向量按需（Vespa 的 streaming mode、Elastic 的 `index_options` 分层）。

### 4.3 整改

**(a) `default_off` 的库不嵌向量**（词法索引保留，显式 `packages` 仍可搜）：

```python
# search_vec.py  _pending_docs 内，紧跟现有的 libs 分批判断
_off = search_index.default_off_libs()
if source == "code" and kb_name in _off and libs is None:
    continue        # 默认不搜的库不占向量：它们是 85% 的文档量，却只在显式 packages 时才被检索
```

**(b) 让「默认不搜」也能不建词法索引**（可选，按库开关）：`index_skip.json` 已有 `never_index` 名单
（当前为空）。把 `postgresql` / `l_comfyui_frontend` 这类第三方/构建产物放进去，17k → ~4k docs。

**(c) 预热按需**：`warm/startup` 9s 对一个会被热重载反复重启的服务是纯浪费。改成
「先起服务、预热丢后台」（和已经修过的 `_spawn_full_scan` 同一思路），或 `L_NOTEPAD_WARM=0` 时跳过。

### 4.4 预期结果

| 指标 | 现在 | 修完 |
|---|---|---|
| `db_bytes` | 1.29 GB | ~0.4 GB（(a)）/ ~0.2 GB（(a)+(b)） |
| 启动预热 | 9.0~9.9 s | ≤ 3 s（(b)）/ 不阻塞（(c)） |
| 默认检索质量 | 不变 | 不变（这些库本来就不在默认范围） |

---

## 5. P3 — `auto` 档位形同虚设

### 5.1 问题

`search_auto()`（`search_index.py:3362`）的回退判据是 `total >= _AUTO_MIN_HITS`（**3 条**）。
但中文按 bigram OR 宽召回，`total` 动辄 19/42/554 —— **判据永远成立**。
实测 5 条提问（含 4 段长句 `wuwo 启动报错 找不到包`）全部 `mode_used=lex`，**一次都没回退**。

于是：花了 1.29 GB 存的 100,507 个向量块，在默认档位下**从不参与**；真走 `hybrid` 要 7.5s。
向量对 docs 的覆盖率也只有 **2,595 / 17,325 = 15%** —— 即便回退了，85% 的文档也没有语义表示。

### 5.2 行业做法

- **按「质量」而非「数量」决定要不要加档**：Vespa/Elastic 的 phased ranking 看一阶段的分数分布；
  RAG 工程里常用 top-1 分数阈值、或 top1/top2 分差（score gap）判断「是否有把握」。
- **零/低召回才升级**（cascade retrieval）：Bing / Google 的多级检索都是廉价档不够格才上贵档。
- **query rewriting / HyDE** 作为词法失手时的兜底，比全量 hybrid 便宜。

### 5.3 整改

```python
# search_index.py
_AUTO_MIN_HITS = 3
_AUTO_MIN_TOP_SCORE = 40.0     # 实测：正确文档命中时 top-1 常在 50~95；<40 基本是「蹭词」
_AUTO_MIN_GAP = 1.3            # top1/top2 分差不足 → 前排没有明显赢家，语义值得一试


def _auto_confident(result: dict) -> bool:
    """词法这一档「有把握」吗？——数量判据没用（bigram OR 宽召回下 total 永远 >= 3，
    实测 5 条提问一次都没回退过 hybrid，向量层等于白存）。改看**分数分布**。"""
    hits = result.get("hits") or []
    if int(result.get("total") or 0) < _AUTO_MIN_HITS or not hits:
        return False
    top = float(hits[0].get("score") or 0)
    second = float(hits[1].get("score") or 0) if len(hits) > 1 else 0.0
    if top < _AUTO_MIN_TOP_SCORE:
        return False
    return second <= 0 or top / second >= _AUTO_MIN_GAP
```

`search_auto` 里把 `enough = int(result.get("total") or 0) >= _AUTO_MIN_HITS` 换成
`enough = _auto_confident(result)`，并把判据写进返回的 `mode_reason`（可观测，别让「为什么没回退」再次成谜）。

配套：`hybrid` 7.5s 太慢，回退档要加**硬预算**（见 P4）；向量覆盖率靠 P2(a) 腾出的配额去嵌
`kb` / `note` / 常搜包，把 15% 拉到「默认搜索范围内 100%」。

### 5.4 预期结果

| 指标 | 现在 | 修完 |
|---|---|---|
| `auto` 回退率 | 0%（从不） | 词法低分/零召回时回退（预估 10~20% 查询） |
| 近义词提问（无字面重叠） | 基本查不到 | 语义兜底可命中 |
| 默认档位耗时 | 1.2 s | 有把握时不变；回退时 ≤ 预算上限 |

---

## 6. P4 — 尾延迟没封死

### 6.1 问题

抽样第一轮（`kb_audit2.py`）的**第一条** `/api/search`，**180s 超时无响应**；随后稳定态一律 1.0~1.5s，
无法复现。可疑叠加项：后台全量比对 + 重排服务冷启动（模型加载）+ 首次向量查询。
上游此前已修过一个同类问题（TTL 全量比对现场做 → 首查 51.9s，已改为丢后台线程）。

**风险点不是「慢」，而是「没有上限」** —— 客户端（`notepad_search` 超时 60s）只能靠超时兜，
而超时对 agent 等价于「知识库不可用」，它会转头去 `read_file` 硬翻源码（又慢又费 token）。

### 6.2 行业做法

- **软/硬预算 + 优雅降级**：Vespa `timeout` + `ranking.softtimeout`（超时返回部分结果并标记）、
  Elasticsearch `timeout` / `terminate_after`、Google 的「tail at scale」（hedged request、分级降级）。
- **返回体自带 `degraded` 标记**，调用方知道这次是降级结果 —— 本仓 `/api/search/route` 已有
  `degraded` / `reason_code`，把它推广到 `/api/search`。

### 6.3 整改

```python
# search_index.py  search() 开头
_QUERY_BUDGET_MS = _env_int("L_NOTEPAD_QUERY_BUDGET_MS", 1500)


def _deadline(budget_ms: int | None = None) -> float:
    return time.monotonic() + (budget_ms or _QUERY_BUDGET_MS) / 1000.0
```

三个卡点按预算裁掉，且**一律先保住词法结果**：

```python
# 1) 重排：剩余预算不够就跳过（cross-encoder 是最贵的一档）
if deadline - time.monotonic() < search_vec.RERANK_TIMEOUT_S * 0.6:
    allow_rerank, off_reason = False, "查询预算不足，跳过重排"

# 2) 语义/回退：同理，auto 回退前先看预算
if deadline - time.monotonic() < 2.0:
    result["mode_reason"] = "预算不足，不回退 hybrid"

# 3) 返回体标记
result["degraded"] = bool(dropped)
result["degraded_reason"] = "; ".join(dropped)
```

`notepad_knowledge.py` 侧同步：`_TIMEOUT` 可以从 60s 收回 20s（服务端有硬预算后 60s 没意义了），
并在 `degraded=True` 时给模型一句 hint（「本次结果降级，必要时换关键词重查」）。

### 6.4 预期结果

最坏情况 1.5~2.0s 返回**带 `degraded` 标记的词法结果**，客户端超时从此不该再触发。

---

## 7. P5 — 没有离线评测集，分数也无量纲【元问题】

### 7.1 问题

- **没有 golden set**：P0 那种「重排净负收益」的退化，靠人肉抽样才发现，已经带病运行三周。
- **`score` 没有量纲**：跨查询在 11~95 间飘（`变体哈希` 的冠军 62.96、`单实例守卫` 的 95.07、
  `nginx 反代` 的 25.16）。agent 拿到 25 分的结果无法判断「这条够不够可信」，
  也就不会在结果差时改写查询 —— 只会照着一条弱命中往下走。

### 7.2 行业做法

- **BEIR / MTEB** 的标准指标：nDCG@10、Recall@k、MRR；RAG 侧 **Ragas**（context precision/recall）。
- **每次改排序都跑回归**：Elastic / Vespa 的 ranking 变更都带 A/B + 离线评测；
  LlamaIndex/LangChain 都内置 evaluation 模块。
- **返回校准后的置信度**：Cohere Rerank 返回 0~1 的 `relevance_score`；
  向量检索常用 min-max / softmax 归一，或对 cross-encoder logits 做 sigmoid 标定。

### 7.3 整改

**(a) 建 golden set（30~50 条，半小时的事，本文已有 13 条现成的）**：

```yaml
# rez-package-source/l_notepad_server/999.0/tests/kb_golden.yaml
- q: 变体哈希 空壳包
  sources: note,kb
  want: [rez_pkg/Rez_pkg/变体哈希与wuwo的处理方法.md]
- q: 单实例守卫 端口被占
  sources: note,kb
  want: [rez_pkg/solo_单实例守卫模式.md]
- q: 相册 回收站 数据结构
  sources: note,kb
  want: [rez_pkg/相册功能与数据模型.md, rez_pkg/相册真源与本地缓存方案.md]
- q: 服务卡片 怎么重启
  sources: note,kb
  want: [rez_pkg/Rez_pkg/服务托管与统一启动入口_计划.md]
- q: 相册 批量 多选
  sources: code
  packages: l_WChat
  want: [l_WChat/999.0/src/l_WChat/templates/album.html]
```

**(b) 一条命令出指标**：

```python
# tools/kb_eval.py
"""知识库检索回归：对 golden set 算 Recall@k / nDCG@k / P95 延迟，并支持档位对照。

用法：
    wuwor l_notepad_server -- python tools/kb_eval.py                    # 当前默认配置
    wuwor l_notepad_server -- python tools/kb_eval.py --rerank 0 --rerank 1   # A/B
改排序、改分块、改重排**之前先跑一遍存基线**，之后再跑一遍对比——P0 那种退化就是这么被漏掉的。
"""
import argparse, json, math, statistics, time, urllib.parse, urllib.request
from pathlib import Path

import yaml

BASE = "http://127.0.0.1:8765"
K = 5


def search(case: dict, **override) -> tuple[list[str], float]:
    p = {"q": case["q"], "limit": K, "mode": case.get("mode", "auto")}
    for k in ("sources", "packages", "kb"):
        if case.get(k):
            p[k] = case[k]
    p.update({k: v for k, v in override.items() if v is not None})
    t0 = time.time()
    with urllib.request.urlopen(f"{BASE}/api/search?{urllib.parse.urlencode(p)}", timeout=120) as r:
        d = json.loads(r.read().decode("utf-8", "replace"))
    got = [f"{h.get('kb_name') or ''}/{h.get('rel')}".lstrip("/") for h in (d.get("hits") or [])]
    return got, (time.time() - t0) * 1000


def ndcg(got: list[str], want: set[str]) -> float:
    dcg = sum(1 / math.log2(i + 2) for i, g in enumerate(got) if g in want)
    ideal = sum(1 / math.log2(i + 2) for i in range(min(len(want), K)))
    return dcg / ideal if ideal else 0.0


def run(cases: list[dict], label: str, **override) -> None:
    rec1 = rec = nd = 0.0
    lat: list[float] = []
    for c in cases:
        want = {w.lstrip("/") for w in c["want"]}
        got, ms = search(c, **override)
        lat.append(ms)
        rec1 += 1.0 if got[:1] and got[0] in want else 0.0
        rec += 1.0 if want & set(got) else 0.0
        nd += ndcg(got, want)
        if not (want & set(got[:1])):
            print(f"  [miss@1] {c['q']}\n           want {sorted(want)[0]}\n           got  {got[:3]}")
    n = len(cases)
    print(f"{label:<14} top1={rec1/n:.2%}  recall@{K}={rec/n:.2%}  nDCG@{K}={nd/n:.3f}  "
          f"p50={statistics.median(lat):.0f}ms  p95={sorted(lat)[int(n*0.95)-1]:.0f}ms")


if __name__ == "__main__":
    ap = argparse.ArgumentParser()
    ap.add_argument("--golden", default=str(Path(__file__).parent.parent / "tests" / "kb_golden.yaml"))
    ap.add_argument("--rerank", action="append", type=int)
    args = ap.parse_args()
    cases = yaml.safe_load(Path(args.golden).read_text(encoding="utf-8"))
    for rr in (args.rerank or [None]):
        run(cases, f"rerank={rr}" if rr is not None else "默认", rerank=rr)
```

**(c) 返回校准置信度**，让 agent 会「自我怀疑」：

```python
# search_index.py  组装 hit 时
hit["confidence"] = round(1.0 / (1.0 + math.exp(-(hit["score"] - 40.0) / 12.0)), 3)
# 阈值 40 / 斜率 12 来自 golden set 拟合：正确文档 top-1 分布在 50~95，蹭词命中多在 10~30
```

`notepad_search` 的返回里带上 `confidence`，并在 top-1 < 0.5 时给 hint：
「本次命中置信度低，建议换关键词（去掉动词、只留名词）或改用 `sources='code'` 查实现」。

### 7.4 预期结果

- 任何排序/分块/重排改动都能在 1 分钟内拿到 `top1 / recall@5 / nDCG@5 / p95` 四个数；
- P0 这类退化在合并前就被拦下；
- agent 拿到低置信度结果会主动改写查询，而不是抱着弱命中往下跑。

---

## 8. 实施顺序与验收

| 序 | 动作 | 工作量 | 验收 |
|---|---|---|---|
| 1 | **关重排**（`POST /api/search/rerank {"enabled":false}`） | 1 分钟 | 5 问 top-1 全对、查询 ≤ 0.35s |
| 2 | 建 golden set + `tools/kb_eval.py`，存基线 | 1 小时 | 一条命令出四个指标 |
| 3 | P0 (a)(b)(c)(d)：喂料带上下文、RRF 融合、块长度归一、rank floor | 半天 | 重排重新打开，`kb_eval` 指标 ≥ 基线 |
| 4 | P1 (a)：`.md` 升级为文档桶（`doc_type`） | 半天 | `sources=note,kb` 能命中包内 `.md` |
| 5 | P1 (b)(c)：仓库根文档同步、清 `default` 空库 | 1 小时 | 「服务怎么重启」命中 `service-lifecycle` |
| 6 | P2 (a)(b)(c)：`default_off` 不嵌向量、`never_index` 名单、预热丢后台 | 半天 | `db_bytes` ≤ 0.5 GB，预热 ≤ 3s |
| 7 | P4：查询硬预算 + `degraded` 标记 | 半天 | 压测下 p99 ≤ 2s，客户端不再超时 |
| 8 | P3：`auto` 改分数判据 + 向量覆盖默认范围 | 半天 | 回退率 10~20%，近义词提问可命中 |
| 9 | P5 (c)：`confidence` + 低置信 hint | 2 小时 | `notepad_search` 返回带 confidence |

**总体预期**：默认查询 1.2s → 0.6s（重排开）/ 0.25s（关），
`sources=note,kb` 的 top-1 正确率 ~20% → ~90%（golden set 口径），
DB 1.29 GB → 0.4 GB，启动预热 9s → 不阻塞，
且从此**每次改动都有可回归的数字**。

---

## 9. 实施结果（2026-09-28 当天落地）

> §0~§8 是**体检与方案**（原样保留，便于对照）；本节只记**实际做了什么、量到什么、和预期差在哪**。
> 逐项代码改动见 `l_notepad_server/999.0/src/l_notepad_server/doc/CHANGELOG.md`（v3.5.0）；
> 评测资产：`999.0/tests/kb_golden.yaml`（15 条）+ `tools/kb_eval.py`（一条命令出四个指标，A/B 自动打 Δ）。
> 本节数字的复现脚本在 `d:/Temp/lk_*.py`（`lk_live` 分项探针 / `lk_cost` 档位耗时 / `lk_off` 语料占比 /
> `lk_warm` 预热拆解 / `lk_check2` 完整性 / `lk_purge`+`lk_vprune` 摘除与清向量）。

### 9.1 一句话

**降噪 > 调排序，而「导航页」也是噪声。** golden set top-1：46.7%（修好排序、语料仍脏）
→ 66.7%（清掉生成副本 + 自指夹具）→ **73.3%**（给文档目录表降权，见 9.3 P0(e)）
→ **80.0%**（二期：症状串库修复 + 文件名召回，见二期方案 §11）。
> ⚠️ **这串数字全是 15 条量出来的，而 15 条里 12 条是 Rez-Docs 主文档（最强区域）** ——
> 扩到 40 条后真实 top-1 是 **70.3%**（见二期方案 §11.8）。上面的**趋势**可信，**绝对值不可引用**。

三项排序修复（喂料带上下文 / RRF 融合 / 块长度归一）在脏语料上贡献 13.3 点；
P2 把语料从 17k 压到 2.5k 不动 top-1，但把 **recall@5 86.7% → 93.3%**（15 条口径；40 条口径见 §11.9）。
**重排最终默认关**：40 条复测是**真·权衡**（top1 −5.4 / MRR −0.009 对 recall +8.1 / nDCG +0.025），
依据是「agent 先读 top-1」＋「快 2 倍」，**不是「有害」**。
15 条口径下剩 3 条 miss@1：`热重载 改了源码不生效` 是**多答案用例**（第 2 名就是另一条正确答案），
`l_notepad 搜索接口 怎么调` 同主题近邻抢位，`CHANGELOG.md` 已升到第 2 名。

### 9.2 基线（15 条 golden set，本机，含 ollama:11434 + rerank:11435）

| 时点 | 配置 | top1 | recall@5 | nDCG@5 | p50 | p95 |
|---|---|---|---|---|---|---|
| 修好排序（语料仍脏） | `rerank=0` | 33.3% | 86.7% | 0.595 | 1492ms | 1819ms |
| 修好排序（语料仍脏） | `rerank=1` | 46.7% | 86.7% | 0.636 | 2499ms | 2932ms |
| 清镜像 + 自指夹具 | `rerank=0` | 66.7% | 86.7% | 0.767 | 1228ms | 2013ms |
| 清镜像 + 自指夹具 | `rerank=1` | 66.7% | 86.7% | 0.780 | 2557ms | 2852ms |
| +P2(a)(b)（语料 2.5k） | `rerank=1` | 66.7% | 93.3% | 0.796 | 2509ms | 3506ms |
| **+P0(e) 导航块降权（现状）** | **`rerank=0`（新默认）** | **73.3%** | **93.3%** | **0.825** | **1437ms** | **1757ms** |
| +P0(e) 导航块降权 | `rerank=1` | 66.7% | 93.3% | 0.801 | 2711ms | 3106ms |

> **语料**：`docs` 17,325 → **2,515**；向量 `vec_chunks` 100,507 → **55,289**（`vec_docs` 2,488）。
> `PRAGMA integrity_check` / fts5 `integrity-check` 均 ok，FTS 行数 = 文档行数，孤立向量行 0。
>
> ⚠️ **DB 文件一个字节都没小**：删行只把页挂到 freelist，文件仍是 `1,292,988,416` 字节
> —— 和体检时 §1 记的是**同一个数**，那里按 10⁹ 写作「1.29 GB」，这里若按 2³⁰ 写就成了「1.20 GiB」，
> **换算单位不是瘦身**（早先版本把两者摆进「→」里，是本文自己的口径错误，已订正）。
> 实测 `page_count=315,671 × 4,096`，其中 freelist **192,710 页 ≈ 0.74 GiB** →
> **`VACUUM` 之后才会掉到约 0.47 GiB**，见 9.6。

### 9.3 逐项对照

| 项 | 计划 | 实际落地 | 备注 / 偏差 |
|---|---|---|---|
| P0 (a) 喂料带上下文 | 标题前缀 + 父块窗口 | `search_vec.rerank_input` + `doc_window`（块 <200 字补到 ~600） | `rerank_docs(trim=False)`：喂料已组好，再截一次会把标题前缀切掉 |
| P0 (b) RRF 融合 | `_RRF_K = 60` | `_fuse_lex_rerank` + **`_RRF_K_FUSE = 60`** | 计划的名字会覆盖 zh_en 双路调好的 `_RRF_K = 10` → 另起常数，两处互不影响 |
| P0 (c) 块长度归一 | `_CHUNK_FULL = 400` | 同 | 单测：`变体哈希 空壳包` 选中块 `chunk_no` 0（18 字标题）→ 1（正文） |
| P0 (d) rank floor | 领先 1.5 倍保底进前 3 | 同（直接提到首位） | 比计划略强（计划是「跌出前 3 才提」） |
| P0 补充 | 回填 `/v1/models` | `_rerank_server_model`（60s 缓存，仅状态页用） | 状态页 `model` 不再空串 |
| **P0 (e) 导航块降权**（计划外） | — | `_nav_prior` / `_PRIOR_NAV=0.7`（命中块里 ≥3 行、且过半行是 markdown 链接 → 打折） | 长度归一只治了「太短的标题块」，没治**900 字的目录表**：`Rez-Docs/README.md` 一行摘要凑齐全部查询词，实测偷走 2 条 top-1。判据看**命中块内容**不看文件名，README 里讲内容的块照常参赛 |
| **P0 (f) 重排默认关**（计划外） | 计划是「修好后重新打开」 | `rerank_enabled` 默认 `False` + 新增 `rerank_switch()` 区分「显式关 / 没表态」 | ⚠️ **本行结论改过一次，且「净负收益」的措辞已作废**：二期收尾复测（15 条）`rerank=0` 80.0%/100%/0.888/1133ms 对 `rerank=1` 73.3%/100%/0.881/2493ms —— 质量指标**不劣**、快 2.2 倍。**默认关的依据是「不劣且快」，不是「有害」**；期间量到过 −6.7 / +0.048 等相反结论，根因是 15 条太小（翻 1 条 = ±6.7 点）＋ 本文档自己进过索引当了答案。详见二期方案 §11.6 |
| P1 (a) doc_type | 新增列 + 检索放行 | **不落库**：`_src_clause` 按 `DOC_EXTS` 放行 code 桶里的文档行 | 免迁移、改完立即生效、零重扫；实测 `sources=note,kb` 命中 `l_notepad_server/.../doc/help.md` |
| P1 (b) 仓库文档同步 | 平铺复制 | `tools/sync_repo_docs.py`：**保留目录结构**、`.mdc`→`.md`、幂等（不动 mtime）、`--dry-run/--prune` | 干跑 36 个文件；**未真跑**（会往工作区写 → depot 上传，属共享状态） |
| P1 (c) 清 default 空库 | 删掉或明确用途 | 删不掉（`ensure_default_base` 自动补、`delete_base` 拒绝）→ `list_bases(include_placeholder=False)` | `/api/kb/bases` 只列 `rez_pkg`，`?empty=1` 可取，带 `placeholder` 标志；`/web/kb/default` 仍可直达 |
| P2 (a) 不嵌向量 | 同 | 同 + `purge_default_off_vectors` 清存量 | `ChatRoom` 独占 **44,168 / 99,457** 块（44%）→ 摘掉；6 库向量全 0 |
| P2 (b) never_index | 可选 | 6 库全进名单 + `index_all --purge` | 17,286 → **2,515 篇**；顺带补了事件路径漏洞（见 9.4-3） |
| P2 (c) 预热 | 丢后台 / 可关 | 本来就是后台线程（不阻塞启动）；加 `L_NOTEPAD_WARM=0` | ⚠️ 成本结构与计划假设不符，见 9.4-4 |
| P3 auto 判据 | 分数判据 | `_auto_confident`（top-1 ≥ 40 且 top1/top2 ≥ 1.3）+ `mode_reason`；第一遍纯词法探路且只用一半预算 | 回退从「从不」变成真在回退；实测回退率 ~100%（见 9.5） |
| P4 硬预算 | 1500 ms + `degraded` | **3000 ms** + **缩重排窗口** + `degraded` + 嵌入可达性探测 | 两处必须改，否则重排被静默关掉（见 9.5） |
| P5 评测集 | 30~50 条 | 15 条（人工确认 `want`）+ `kb_eval.py` | 已立案，后续按需补 |
| P5 (c) confidence | 服务端 + agent hint | 服务端 `confidence` = `sigmoid((score-40)/12)` + 搜索页「判断依据」显示 | agent 侧 hint 未做（`l_agent_chat`，另一个包） |

### 9.4 计划外发现（都已落成代码/配置）

1. **`l_nginx/999.0/runtime/html/docs/**` 是 Rez-Docs 的生成副本** —— 部署时整棵文档树被复制进 nginx 运行时目录，
   与原件逐字相同却经常排前面（15 条里 6 条 top-1 是它，原件反而第 2）。`files` 只能按文件名排除 → 新增
   **`dirs`** 键（`<库标签>/<相对目录>`，整棵子树不索引）：`_scan_code` 连下钻都不下钻、`code_watch._rel_ok`
   事件侧同拦、`_code_index_one` 兜底、`purge_skip_dirs` 摘存量（实测 30 篇 / 403 块）。
2. **本文档自己是自指夹具** —— 正文逐字含 golden 查询（4 条 top-1 是它），与 `jev_queries.json` 当初
   被排除是同一个坑 → 进 `files`；同时 `_kb_sync_one` 补上 `files` 名单（此前只作用于代码索引，
   知识库侧照收），另补 `kb_golden.yaml`（评测集本身就是全部查询的清单，会被自己的语料搜出来）。
3. **`never_index` 有事件路径漏洞** —— `_scan_code` / `index_local_lib` 会拒绝整库，但**文件事件绕过扫描**
   直接走 `_code_index_one`：改一下那个库里的文件就把行写回来（`drop_lib` 白摘）。已让 `_code_index_one`
   与 `code_watch._rel_ok` 都拦（与 `dirs` 名单同一处兜底）。
4. **预热的成本结构和本文 §4.1 的假设不符**（本次实测拆解）：
   `_refresh(force=True)` **204 ms** ＋ `sync_kb(default)` 640 ms ＋ `sync_kb(rez_pkg)` **8000 ms** ≈ **8.8 s**。
   9 秒的大头是 **depot 归档目录列举**（网络），不是「1.7 万文件 stat」——P2(b) 对预热**没有影响**
   （它只改了词法规模），也没有影响首查（请求路径本来就不走网络）。杠杆是 `L_NOTEPAD_WARM=0`，
   或给 `depot_map.list_tree` 加短 TTL 缓存（**建议，未做**）。

### 9.5 与预期的偏差（都是实测校准出来的）

| 计划 | 实际 | 为什么 |
|---|---|---|
| 预算 1500 ms | **3000 ms**（`L_NOTEPAD_QUERY_BUDGET_MS`） | 本机完整管道 = lex 0.15 + 嵌入 1.2 + 重排 1.4 ≈ 2.8s；按 1500 跑，重排会被**系统性**跳过（P0 白修） |
| 卡点用 `RERANK_TIMEOUT_S * 0.6` | 用**每候选实测耗时**（`rerank_cost_per_doc`） | 那是**超时上限**（本机 8s）：`8×0.6` 让「剩余 ≥ 4.8s」永远为假 → 重排被**静默永久关闭**（真踩到了） |
| 预算不够就整档跳过 | 先**缩窗口**（少排几个候选） | Vespa `rank_window_size` 同思路：「排前 3 个」仍优于完全不排 |
| `degraded` 标记所有降级 | 只标**影响结果集**的两档（语义、hybrid 回退） | 重排只影响排序不影响结果集；否则 `auto` 每查都报降级，标记很快失去意义 |
| auto 回退率 10~20% | 实测 **~100%** | 判据是 top1/top2 ≥ 1.3，而当前语料同主题近邻多、分差常在 1.0~1.2 —— 不是判据错，是语料特性。降噪后确有查询回到「词法够用」快路径（`变体哈希 空壳包`：`mode_reason=词法够用（top-1 44.0、top1/top2=1.94）`），但**那一档省的只是嵌入那 1.2s，不是端到端 121 ms** |
| P0 预期「查询 1.2s → 0.35s」 | 重排关 **p50 1437 / p95 1757 ms**；重排开 p50 2711 ms | 计划低估了 ollama 单次嵌入（~1.2s）。0.35s 只在「纯词法 + 不回退」时成立，而回退率 ~100% |
| 硬预算「最坏 1.5s 返回 `degraded`」 | **预算不是墙钟上限** | `_QUERY_BUDGET_MS`(3000) 只在**开下一档之前**判「还够不够」，已经进去的阶段（尤其 `embed_texts` 的 HTTP 请求）**没法中断**。实测单查冲到 4.6s。要真封顶得给每档独立超时 + 提前返回，**未做**（见 9.6） |
| 重排「恢复正收益」 | **最终仍是负收益 → 默认关** | 降噪 + 导航块降权把词法排序修对之后，重排反而把它踩坏（top1 −6.7 / nDCG −0.023）。这正是 P5 评测闭环的价值：同一个阶段，语料一变、结论就反号，靠感觉判断必错 |

### 9.6 未做 / 待办

- **`VACUUM`**：删行只进 freelist（192,710 页 ≈ 0.74 GiB，真空后约 0.47 GiB），需独占写锁 → 建议停服做。
  **在此之前，任何「DB 变小了」的说法都不成立**（见 9.2 的口径警告）。
- **包 README 被注入了别人家的「症状」**（2026-09-28 新发现，**未修**）：索引时给每篇代码文档加的块头是
  `[<库>] <路径>` + `符号: …` + **一批症状句**，但症状**没有按库过滤** —— `l_frp` / `l_nginx` /
  `l_agent_chat` 的 README 块头里都躺着「搜索大量笔记时界面反应迟钝」「点击搜索按钮后无任何反应」
  这几句 l_notepad 的症状（`grep` 原文件确认：**文件里没有，是索引时注进去的**）。
  后果：**任何带「搜索 / 卡顿 / 没反应」字样的提问，所有包的 README 都是万能候选**。
  修法是在注入处按 `kb_name` 过滤症状来源；影响面涉及口语症状通道，需单独评测后再动。
- **`l_nginx/999.0/README.md` 仍压着 `Nginx反向代理机制.md`**：它的块头把标题
  「l_nginx — Lugwit 反向代理」**重复了两遍**（块头一次、正文 `#` 标题一次）→ 词频虚高。
  这条**不打算用先验硬压**（包 README 对「nginx 反向代理」本来就是合理命中），
  正解是块头去重，和上一条同一处代码。
- **`CHANGELOG.md` 召不回**（recall@5 唯一的那个缺口）：P1(a) 已经把 code 桶的 `.md` 放行进
  `sources=note,kb`，但 CHANGELOG 正文极长、查询词（「变更」「历史」）在其中密度极低，
  长度归一之后更吃亏。需要**标题/文件名通道**（`rel` 命中给独立加分），未做。
- **硬预算不是墙钟上限**（见 9.5）：`embed_texts` 进去了就中断不了，实测单查冲到 4.6s。
  要真封顶得给每档独立 socket 超时 + 到点直接返回已有结果。
- **`l_agent_chat` 侧**：`notepad_knowledge._TIMEOUT` 60s → 20s（服务端有硬预算后 60s 没意义）；
  `degraded=True` / 低 `confidence` 时给模型 hint（「本次降级，必要时换关键词重查」）。
- **`tools/sync_repo_docs.py` 未真跑**（会触发 depot 上传，属共享状态）。
- **评测集 15 条偏小**（本文 §7.3(a) 建议 30~50 条），且 `want` 只覆盖「top-1 该是谁」，
  未覆盖跨库/跨语言/口语症状通道（后者另有 `symptoms.json` 与 `tools/quality_report.py` 的 MRR 口径）。
  另注意 `kb_eval` 的 `recall@k` 是 **hit@k 口径**（命中 `want` 任意一条即算 1），
  单答案用例下它和 top-1 只差名次信息 —— 别把它当标准召回率读。
- **本文档自己在 `index_skip.files` 里**（避免评测查询被自己的正文命中），
  所以**在知识库里搜不到它**；靠 `Rez-Docs/README.md` 的索引行发现。

### 9.7 复现

```bash
# 四个指标 + A/B 自动打 Δ（服务需在跑）。重排默认关，`--rerank 1` 仍能单次打开
wuwor l_notepad_server -- python -m l_notepad_server.tools.kb_eval --rerank 0 --rerank 1
# 改了 search_index.py / search_vec.py 之后必须重启服务才生效（.py 不热重载）
wuwo svc restart l_notepad_api
# DB 真实体积与 freelist（确认「变小」到底有没有发生）
wuwo\py_312\python.exe -c "import sqlite3;c=sqlite3.connect('file:<pkg>/999.0/data/notepad.sqlite3?mode=ro',uri=True);print([c.execute('PRAGMA '+p).fetchone()[0] for p in ('page_size','page_count','freelist_count')])"
# 语料治理（幂等，可重复执行）
wuwor l_notepad_server -- python -m l_notepad_server.tools.index_all --purge      # never_index + dirs 存量
wuwor l_notepad_server -- python -m l_notepad_server.tools.vec_rebuild --prune-off # 默认不搜库的向量
# 仓库根规则/手册同步进 Rez-Docs 工作区（非 dry-run 会触发 depot 上传）
wuwor l_notepad_server -- python -m l_notepad_server.tools.sync_repo_docs --dry-run
```

---

## 附录 A：复现命令

```bash
# 索引规模 / 向量 / 重排 / 历史 / 本机库清单
wuwo\py_312\python.exe d:\Temp\kb_audit.py
wuwo\py_312\python.exe d:\Temp\kb_audit2.py
# 档位耗时对照（lex / auto / hybrid / rerank=0 / route）
wuwo\py_312\python.exe d:\Temp\kb_audit3.py
# 8 条真实提问的质量抽样
wuwo\py_312\python.exe d:\Temp\kb_audit4.py
# 重排 A/B（本文 2.1 的表）
wuwo\py_312\python.exe d:\Temp\kb_audit5.py
# 重排服务身份 + 向量分布
wuwo\py_312\python.exe d:\Temp\kb_audit6.py
# 重排「喂错料」根因（本文 2.3 的 chunk 明细）
wuwo\py_312\python.exe d:\Temp\kb_audit7.py
```

## 附录 B：涉及的代码位置

| 位置 | 作用 | 本文对应 |
|---|---|---|
| `search_index.py:3013 _best_chunk` | 挑「命中块」 | P0 (c) 长度归一 |
| `search_index.py _nav_prior` / `_PRIOR_NAV` | 文档目录表降权 | P0 (e) |
| `search_index.py _rerank_decision` | 单次请求 > 显式配置 > 默认 | P0 (f) |
| `search_vec.py rerank_switch` / `rerank_enabled` | 显式开关 vs 默认（默认关） | P0 (f) |
| `search_index.py:3078 _apply_rerank` | 重排并重排序 | P0 (b)(d) |
| `search_index.py:3126 order_key` | **重排独占主序** | P0 (b) |
| `search_index.py:3135 _rrf_fuse` | 已有的 RRF 实现（中英双路用） | P0 (b) 直接复用思路 |
| `search_index.py:3362 search_auto` / `:68 _AUTO_MIN_HITS` | `auto` 回退判据 | P3 |
| `search_index.py:230 default_off_libs` | 默认不搜名单 | P2 (a) |
| `search_vec.py:1303 _trim_for_rerank` | 400 字窗口 | P0 (a) |
| `search_vec.py:1328 rerank_docs` | 调 `:11435` | P0 (a) |
| `search_vec.py:634 _pending_docs` | 待嵌入文档筛选 | P2 (a) |
| `routers/search.py:41 api_search` | `/api/search` 查询面 | P4 `degraded` |
| `l_agent_chat/notepad_knowledge.py` | agent 侧工具 | P4 超时、P5 hint |
