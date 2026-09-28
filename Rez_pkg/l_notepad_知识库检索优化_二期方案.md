# l_notepad 知识库检索二期优化方案

> 日期：2026-09-28
> 范围：`l_notepad_server` 检索服务、`l_agent_chat` 消费方、代码/知识库/笔记三类语料。
> 文档性质：问题体检 + 分阶段实施方案 + 行业实践对照 + 代码示例 + 结果预期。
>
> 一期已经完成：词法/RRF/块长度归一、导航块降权、语料裁剪、golden set、`confidence`、`degraded`、重排默认关闭。
> 二期不再重复调 RRF 参数。重点转向：**语料正确性、未索引可见性、文件名召回、可观测性、评测闭环**。

---

## 0. 执行摘要

### 0.1 当前判断

一期让 Rez-Docs 主场景从约 20% top-1 提升到 **73.3%**，集外抽样约 **75%**。这证明排序修复有效。
但整体知识库仍有四个结构性问题：

1. **症状/中文锚点可能串库**：`_symptom_pick()` 的 `rel` 后缀兜底没有库隔离。同名相对路径出现在多个包时，症状由字典遍历顺序决定。
2. **14,772 篇 `never_index` 语料静默消失**：查询 ChatRoom / postgresql 等库时，接口返回低分无关结果，不告诉调用方「目标库没有索引」。
3. **文件名召回不稳定**：`rel` 不是 FTS 独立索引列；CHANGELOG 等长文档按文件名查询可能进不了前五。
4. **普通接口缺少统一运行信息**：显式 `lex/hybrid/sem` 查询没有 `mode_used/mode_reason`；低分但相对最相关的答案会被固定绝对分数压成低 `confidence`。

### 0.2 优先级

| 优先级 | 项目 | 风险 | 推荐动作 |
|---|---|---|---|
| P0 | 症状按库隔离 | 结果语义错误，agent 可能拿错代码 | 收紧 `rel` fallback；增加同名路径回归测试 |
| P0 | 未索引库显式提示 | 能力边界不可见，agent 把噪声当答案 | 响应增加 `unindexed_packages` |
| P1 | 路径/文件名召回 | 用户点名文件却召不回 | 先加 query-time path bonus，后评估 FTS path 列 |
| P1 | mode 可观测性 | 无法解释为何走 hybrid/lex | 所有响应统一返回 `mode_used/mode_reason` |
| P1 | confidence 校准 | 正确结果被低置信度误弃 | 只改 confidence，不改排序，加入查询内相对分 |
| P2 | holdout 扩展 | 73.3% 可能被小样本放大 | golden 扩到 30~50 条，增加负例/目录页样例 |

### 0.3 总体结果预期

完成 P0/P1 后，预期不是「所有问题 top-1 100%」，而是：

- 不再发生跨库症状错配；
- 查询未索引库时明确提示，不再静默给低分噪声；
- 文件名/CHANGELOG/函数名类问题 Recall@5 提升；
- 每次查询能解释实际模式、降级原因、未索引范围；
- agent 能区分「没有答案」「目标库未索引」「返回了低置信候选」三种状态。

---

## 1. P0：症状锚点串库

### 1.1 问题

代码索引 key 形如：

```text
code:<库标签>:<相对路径>
```

例如：

```text
code:l_frp:999.0/README.md
```

症状选择函数当前支持三层匹配：

```python
if key in table:
    return table[key]
if rel and rel in table:
    return table[rel]
for k, v in table.items():
    if rel and (k.endswith("/" + rel) or k.endswith(":" + rel)):
        return v
```

位置：`search_index.py:705-717`。

最后一层没有库标签约束。多个库存在相同 `rel` 时，结果取决于 `dict` 插入顺序。例如多个包都有：

```text
999.0/README.md
src/utils.py
```

其中一个包的症状可能被另一个包复用。后果：

- FTS `symptom` 列出现错误中文语义；
- 向量 `anchor_for()` 把错误症状嵌入代码块；
- 用户问「搜索卡顿」时，无关包 README 变成高频候选；
- agent 读取错误文件，后续分析方向错误。

这是**结果正确性问题**，不是普通排序问题。

### 1.2 当前链路

代码索引入口：`search_index.py:2062-2064`：

```python
_upsert(
    conn, key, body, st.st_size, st.st_mtime,
    source="code", kb_name=label, rel=rel, rev=0,
    symptom=symptom_for(key, rel),
    title=code_head(label, rel, body), digest=digest,
)
```

`_upsert()` 在 `search_index.py:1577-1580` 写入 FTS：

```python
conn.execute(
    "INSERT INTO search_fts(rowid, title, symptom, body, note_path) VALUES(?,?,?,?,?)",
    (rowid, index_text(title), index_text(symptom), index_text(body), key),
)
```

FTS 结构：

```sql
CREATE VIRTUAL TABLE search_fts USING fts5(
    title,
    symptom,
    body,
    note_path UNINDEXED,
    tokenize='unicode61 remove_diacritics 2'
)
```

`anchor_for()` 同样通过 `_symptom_pick()` 取职责与症状，拼进向量块前缀。因此修复必须同时覆盖词法与向量两侧。

### 1.3 推荐方案

保留症状通道，不删除 `symptom` 列。行业成熟方案通常保留 contextual metadata，但要求 metadata 与文档 ID 严格绑定，禁止模糊跨文档继承。

最小修复规则：

1. 完整 key 精确命中：允许；
2. 代码文档禁止裸 `rel` 命中；
3. 代码文档禁止跨库 `endswith` 匹配；
4. 非代码笔记可保留 `rel` 匹配，因为笔记没有库标签；
5. 同一 `rel` 多库同时存在时，宁可没有症状，也不能猜一个。

### 1.4 实现示例

```python
def _symptom_pick(table, key: str, rel: str = "", *, source: str = "", kb_name: str = ""):
    if not table:
        return None
    if key in table:
        return table[key]

    # code 文档必须库隔离；不允许 rel 后缀跨库猜测
    if source == "code":
        return None

    if rel and rel in table:
        return table[rel]
    return None


def symptom_for(key: str, rel: str = "", *, source: str = "", kb_name: str = "") -> str:
    text = _symptom_pick(symptoms(), key, rel, source=source, kb_name=kb_name)
    return str(text) if text else ""
```

调用处改为显式传递来源：

```python
symptom=symptom_for(key, rel, source="code", kb_name=label)
```

更严格版本：只接受 `code:<label>:<rel>`：

```python
def _code_symptom(table, label: str, rel: str):
    return table.get(f"code:{label}:{rel}")
```

### 1.5 行业先进实例

- **Anthropic Contextual Retrieval**：上下文前缀提升检索，但上下文属于当前 chunk，不跨 chunk、跨文档复用。
- **LlamaIndex Parent Document Retriever**：子块回到所属父文档，不用相似路径猜父文档。
- **Elasticsearch metadata filter**：`tenant_id`、`document_id` 与正文分离，过滤条件优先于相关性排序。
- **多租户 RAG 实践**：先做 document scope，再做 lexical/vector/rerank；不能让 reranker 修复错误 metadata。

### 1.6 验收

新增以下回归：

```python
assert _symptom_pick(
    {"code:a:999.0/README.md": "A"},
    "code:b:999.0/README.md", "999.0/README.md",
    source="code", kb_name="b",
) is None

assert _symptom_pick(
    {"code:a:999.0/README.md": "A"},
    "code:a:999.0/README.md", "999.0/README.md",
    source="code", kb_name="a",
) == "A"
```

预期：同名路径跨库症状错配 **0**；症状查询 Recall@5 不应下降超过 2%。

---

## 2. P0：`never_index` 语料静默消失

### 2.1 问题

当前 `index_skip.json` 的 `never_index` 有 6 个库：

- `postgresql`
- `ChatRoom`
- `l_comfyui_frontend`
- `l_comfyui_ai`
- `l_mayaPlug`
- `conemu`

合计约 14,772 篇。`never_index_libs()` 已存在：`search_index.py:355-357`。

当前行为：

```text
查询：ChatRoom 消息 发送
结果：返回其他库低分文件
状态：没有告诉调用方 ChatRoom 未索引
```

这会让人误判为「知识库没有相关内容」，让 agent 更危险地把低分候选当答案。

### 2.2 推荐方案：响应增加能力边界

不要立刻把 14,772 篇全部恢复索引。先让接口诚实表达边界。

新增字段：

```json
{
  "unindexed_packages": [
    {
      "label": "ChatRoom",
      "reason": "never_index",
      "hint": "index_all --libs ChatRoom"
    }
  ]
}
```

只在以下条件触发：

1. `sources` 包含 `code`，或调用方未指定 sources；
2. 查询明确包含库名，或查询结果为空/全部低置信；
3. 目标库属于 `never_index_libs()`；
4. 不在每次正常查询热路径遍历完整库目录。

### 2.3 实现示例

```python
def _mentioned_unindexed_packages(conn, query: str, packages=None):
    never = search_index.never_index_libs()
    if not never:
        return []

    q = str(query or "").lower()
    requested = {str(x) for x in (packages or []) if str(x)}
    out = []
    for label in sorted(never):
        if label.lower() in q or label in requested:
            out.append({
                "label": label,
                "reason": "never_index",
                "hint": f"index_all --libs {label}",
            })
    return out
```

在 `search()` 返回组装处（`search_index.py:3655-3669`）加入：

```python
unindexed = _mentioned_unindexed_packages(conn, query, packages)
return {
    "total": total,
    "hits": page,
    "degraded": bool(dropped),
    "degraded_reason": "; ".join(dropped),
    "unindexed_packages": unindexed,
    "budget_ms": _QUERY_BUDGET_MS,
    "vec": vinfo,
    "rerank": rinfo,
    "zh_en": zh_en_info,
}
```

更准确的做法：在 `local_libs(conn)` / `lib_rows(conn)` 建短 TTL cache，只对 query 中出现的库名做判断。

客户端行为：

```python
if result.get("unindexed_packages"):
    hint = "目标代码库未建立索引：" + ", ".join(
        x["label"] for x in result["unindexed_packages"]
    )
    # agent 不应把低分 hits 当作目标库答案
```

### 2.4 行业先进实例

- **Elasticsearch index existence / alias health**：索引不存在时返回明确错误或健康状态，不伪造空结果。
- **GitHub Code Search**：仓库范围是显式 scope；scope 不可用时提示，而不是返回其他仓库结果。
- **Google Cloud Vertex AI Search**：数据源连接失败与「没有相关文档」分开报告。
- **RAG production observability**：`no_result`、`filtered_all`、`source_unavailable` 分为不同 reason code。

### 2.5 结果预期

- 查询未索引库时：`unindexed_packages` 命中率 100%；
- agent 不再把其他库低分结果当目标库答案；
- 正常文档查询响应耗时增加目标 `<5ms`；
- 明确区分「库不存在」「库未索引」「库已索引但无命中」。

---

## 3. P1：文件名 / 路径召回不稳定

### 3.1 问题

FTS 表只有：

```text
title, symptom, body, note_path UNINDEXED
```

`note_path` 不参与检索。`rel` 只存在 `search_docs.rel`，查询时 SELECT 出来，但不能参与 MATCH。

当前 `title`：

- 代码：`code_head(label, rel, body)`，包含路径与符号；
- note / KB：默认是 `rel`，但只是路径整串的 title，不是独立 path 字段。

因此：

- 查询 `l_notepad_server 变更 历史` 时，`CHANGELOG.md` 可能进不了前五；
- 查询明确文件名时，文件名召回依赖 title 分词与正文是否同时命中；
- 路径、文件名、正文权重无法独立调节。

### 3.2 方案 A：增加 FTS `path` 列

适合长期方案。迁移成本可控，现有已有 `migrate_symptom_column()` 可复用。

表结构：

```sql
CREATE VIRTUAL TABLE search_fts_new USING fts5(
    title,
    symptom,
    path,
    body,
    note_path UNINDEXED,
    tokenize='unicode61 remove_diacritics 2'
)
```

写入：

```python
INSERT INTO search_fts(
    rowid, title, symptom, path, body, note_path
) VALUES(?,?,?,?,?,?)

(index_text(title), index_text(symptom), index_text(rel), index_text(body), key)
```

查询权重：

```python
bm25(search_fts, 6.0, _W_SYM_COL, 3.0, 1.0) AS rank
```

建议：

- `title=6.0`：标题/符号；
- `symptom=4.0`：中文症状；
- `path=3.0`：路径/文件名；
- `body=1.0`：正文。

不要直接把 path 权重调到 6 以上，避免 README/文件名污染正文。

### 3.3 方案 B：无迁移 query-time bonus

短期先做，零数据库迁移：

```python
def _path_bonus(query: str, rel: str) -> tuple[float, str]:
    q = set(_query_tokens(query))
    name = Path(rel).name.lower()
    stem = Path(name).stem.lower()
    matched = [t for t in q if t in name or t in stem]
    if not matched:
        return 1.0, ""
    return 1.0 + min(0.25, 0.08 * len(matched)), "文件名命中"
```

在 `search()` 评分阶段：

```python
path_mult, path_why = _path_bonus(query, str(r["rel"]))
meta["score"] = round(meta["score"] * path_mult, 4)
meta["path_bonus"] = path_mult
```

该方案只能提升已进入 FTS 候选的文档，不能解决「正文/标题完全不命中」的召回。因此先做 B 观测，再做 A。

### 3.4 行业先进实例

- **Elasticsearch**：`title^3`, `path^2`, `body` 多字段 `multi_match`；字段权重独立。
- **OpenSearch**：BM25 初排 + field-level boost + rerank 二阶段。
- **Sourcegraph**：文件名、路径、符号、正文分通道检索，文件名命中有独立排序信号。
- **GitHub Code Search**：`path:`、`repo:`、符号与正文是不同检索范围。

### 3.5 结果预期

- `CHANGELOG.md` 类文件名问题 Recall@5 提升 10~20 个百分点；
- 文件名明确命中的查询 top-1 提升；
- 正文查询 top-1 不下降超过 1 个百分点；
- 迁移后可独立调 path 权重，不再依赖拼接 title。

---

## 4. P1：所有接口统一返回实际搜索模式

### 4.1 问题

`mode_used` / `mode_reason` 当前只在 `search_auto()` 写入（约 `search_index.py:3729-3747`）。

普通 `lex/hybrid/sem` 直接走 `search()`，返回没有这两个字段。结果消费者无法知道：

- 请求是否真的走了指定模式；
- 是否因为预算不足跳过语义；
- 是否发生 hybrid fallback；
- 是否仅返回词法结果。

### 4.2 实现示例

在 `search()` 的统一返回处（`search_index.py:3655-3669`）增加：

```python
return {
    "total": total,
    "hits": page,
    "mode_used": mode,
    "mode_reason": (
        "显式 mode"
        if not dropped else
        "显式 mode，但发生降级：" + "; ".join(dropped)
    ),
    "degraded": bool(dropped),
    "degraded_reason": "; ".join(dropped),
    "budget_ms": _QUERY_BUDGET_MS,
    "vec": vinfo,
    "rerank": rinfo,
}
```

`search_auto()` 继续覆盖这两个字段。这样：

```json
{
  "mode_used": "lex",
  "mode_reason": "显式 mode",
  "degraded": false
}
```

### 4.3 行业先进实例

- **Vespa phased ranking**：响应/trace 能看到 first-phase、second-phase 实际执行阶段。
- **Elasticsearch profile API**：能看到实际使用的 retriever 与阶段耗时。
- **OpenSearch search pipeline**：pipeline 名称、处理器与降级状态可观测。
- **Google SRE**：返回结果与服务状态分离，degraded 不等于 no-result。

### 4.4 结果预期

- 100% `/api/search` 响应拥有 `mode_used`、`mode_reason`；
- agent 能解释为何没有走向量/重排；
- 线上异常可按 `mode_reason` 聚合；
- 不改变排序，只增加可观测性。

---

## 5. P1：confidence 从绝对分数改为查询内校准

### 5.1 问题

当前公式：

```python
def _confidence(score):
    return round(
        1.0 / (1.0 + math.exp(-(float(score) - 40.0) / 12.0)),
        3,
    )
```

位置：`search_index.py:181-186`。

`score` 是加权和，不同查询量纲不同。结果：

```text
卡片 name 和 label 区别
top-1 正确，但 confidence ≈ 0.19
```

agent 可能把正确答案误判成低可信。

### 5.2 推荐方案

只改变 confidence，不改变排序。排序继续使用 `score` / fused score；confidence 不进入 sort key。

查询内相对分更适合表达「当前结果是否明显领先」：

```python
def _confidence(score: float, top_score: float, second_score: float = 0.0) -> float:
    absolute = 1.0 / (1.0 + math.exp(-(score - 40.0) / 12.0))
    if top_score <= 0:
        return round(absolute, 3)
    relative = max(0.0, min(1.0, score / top_score))
    gap = 1.0 if second_score <= 0 else max(0.0, min(1.0, (score - second_score) / max(top_score, 1.0)))
    value = 0.55 * absolute + 0.30 * relative + 0.15 * gap
    return round(max(0.0, min(1.0, value)), 3)
```

应用时先确定排序后的 top1/top2，再给每个 hit 填字段：

```python
top_score = float(page[0].get("score") or 0.0) if page else 0.0
second_score = float(page[1].get("score") or 0.0) if len(page) > 1 else 0.0
for hit in page:
    hit["confidence"] = _confidence(
        float(hit.get("score") or 0.0), top_score, second_score
    )
```

语义模式仍需单独处理余弦分数，不能把余弦分数直接当 0~1 的概率。

### 5.3 行业先进实例

- **Cohere Rerank**：返回 relevance score，生产系统通常结合 top gap、阈值与业务过滤，而不是把单一分数当概率。
- **Vespa**：rank score 与业务 confidence 分离，允许多阶段信号组合。
- **Google Search Quality**：质量/置信度是校准层，不直接替换主排序分。
- **RAGAS / TruLens**：将 retrieval relevance、context precision 与最终答案质量分开评估。

### 5.4 结果预期

- 低绝对分但 top1 明显领先的正确答案不再统一低于 0.5；
- 不改变 top1、Recall@5、nDCG；
- agent 的重查触发更准确；
- 后续可用 50+ golden/holdout 样本做 isotonic/logistic calibration。

---

## 6. P2：评测集从 15 条升级为可持续回归集

### 6.1 当前局限

`tools/kb_eval.py` 当前支持：

- `top1`；
- `recall@k`：`want` 任意一个出现在前 k 即命中；
- 二元相关的 `nDCG@k`；
- `p50/p95`；
- `rerank=0/1` A/B。

当前 15 条用例偏小。它能发现大回归，但不能覆盖：

- 代码/KB/note 三源；
- 未索引库负例；
- 目录页应该赢的导航问题；
- 文件名、函数名、长句、口语症状；
- 多个正确答案的相关性等级。

### 6.2 新用例格式

```yaml
- q: ChatRoom 消息 发送
  sources: code
  packages: [ChatRoom]
  expect:
    status: unindexed
    package: ChatRoom

- q: Rez-Docs 有哪些文档
  sources: note,kb
  expect:
    top1_any:
      - rez_pkg/README.md

- q: l_notepad_server 变更 历史
  sources: note,kb
  expect:
    want:
      - l_notepad_server/999.0/src/l_notepad_server/doc/CHANGELOG.md
    max_rank: 5

- q: 服务卡片 怎么重启
  sources: note,kb
  want:
    - rez_pkg/Rez_pkg/服务托管与统一启动入口_计划.md
  expect:
    mode_used: lex
```

### 6.3 指标扩展

建议至少达到 40 条：

| 类别 | 数量 | 目的 |
|---|---:|---|
| Rez-Docs 主文档 | 12 | 主场景 top-1 |
| 包内 `.md/.mdc` | 6 | P1 文档桶 |
| 代码/函数 | 8 | path/symbol |
| 口语症状 | 6 | symptom 通道 |
| 目录/README | 3 | 导航块不误伤 |
| 未索引库 | 3 | 能力边界提示 |
| 长句/混合中英 | 2 | auto/hybrid |

新增指标：

```text
MRR
unindexed_notice_rate
mode_reason_coverage
low_confidence_false_negative_rate
p50 / p95 / degraded_rate
```

### 6.4 行业先进实例

- **BEIR**：多数据集、多任务评估，不依赖单一查询集合。
- **MTEB**：区分 retrieval、reranking、分类任务，避免一个总分掩盖局部退化。
- **Ragas**：context precision、context recall 分开看。
- **Google / Netflix SRE**：质量指标与延迟、降级率并列，不只看平均命中。

### 6.5 结果预期

- golden + holdout ≥ 40 条；
- 每次提交自动输出质量、延迟、降级、未索引提示四组指标；
- 任一主场景 top1 下降 >5 个百分点时阻止发布；
- `unindexed_notice_rate` 达到 100%；
- 评测集不再只验证「搜索当前已经擅长的 Rez-Docs」。

---

## 7. 实施顺序

### 第 1 阶段：P0 正确性

1. 收紧 `_symptom_pick()`，代码文档只允许完整 key；
2. 加同名路径跨库回归；
3. 增加 `unindexed_packages`；
4. 查询明确未索引库时不把低分结果冒充目标答案；
5. 线上验证 `ChatRoom` / `postgresql`。

验收：

```text
跨库症状错配 = 0
未索引库提示率 = 100%
正常查询 Recall@5 无明显下降
```

### 第 2 阶段：P1 召回与可观测

1. 先实现 query-time path bonus；
2. 加 `mode_used/mode_reason` 到显式模式响应；
3. confidence 改查询内校准；
4. 对 CHANGELOG、文件名、函数名用例跑 A/B。

验收：

```text
文件名类 Recall@5 +10 个百分点以上
所有响应 mode 字段覆盖率 = 100%
排序指标不因 confidence 改动而变化
```

### 第 3 阶段：P2 长期结构

1. 若 path bonus 有效，再迁移 FTS `path` 列；
2. golden 扩到 40 条；
3. 增加未索引、目录页、症状串库、低分正确答案用例；
4. 将 `kb_eval.py` 接入服务变更验收。

---

## 8. 统一回归命令

```bat
:: 重启源码服务
wuwo svc restart l_notepad_api

:: 当前 A/B
wuwor l_notepad_server -- python -m l_notepad_server.tools.kb_eval --rerank 0 --rerank 1

:: 评测扩展集
wuwor l_notepad_server -- python -m l_notepad_server.tools.kb_eval --golden tests/kb_holdout.yaml
```

每次记录：

```text
git revision
DB docs / vec_docs / vec_chunks
rerank enabled
mode_used distribution
unindexed_notice_rate
Top1 / Recall@5 / nDCG@5 / MRR
p50 / p95 / degraded_rate
```

---

## 9. 失败模式与回滚

| 风险 | 现象 | 回滚 |
|---|---|---|
| 症状过滤过严 | symptom Recall@5 明显下降 | 恢复 exact key + 非 code rel 匹配，保留 code 隔离 |
| 未索引提示误报 | 正常 code 查询频繁出现提示 | 只在查询包含库名或 total=0 触发 |
| path bonus 过强 | README/文件名压过正文 | bonus ≤1.25，加入目录块先验保护 |
| confidence 误导 | agent 重查次数增加 | 只回滚 confidence 映射，不回滚排序 |
| FTS path 迁移失败 | 服务启动失败 | 保留 `search_fts_new`，事务内切换，失败 rollback |
| holdout 过拟合 | 指标上涨但真实查询变差 | 固定 holdout 不参与参数调优 |

---

## 10. 最终目标

知识库检索不是「让某个 top-1 数字变高」，而是建立完整契约：

```text
正确语料 → 正确 scope → 词法/向量召回 → 融合排序 → 置信度 → 可观测响应 → 离线回归
```

本期优先保证：

1. **不串库**；
2. **不隐瞒未索引**；
3. **文件名可找**；
4. **模式可解释**；
5. **低分不等于低相关**；
6. **每次改动有 holdout 证据**。

达到这些条件后，才值得继续投入 cross-encoder、向量模型或更复杂融合参数。

---

## 11. 性能校准（2026-09-28 实测，v3.5.5）

### 11.1 结论

**向量层用 6 倍延迟只换来 1 条召回。** 37 条 golden 逐条对照：

| 模式 | top1 | recall@5 | nDCG@5 | MRR | p50 | p95 |
|---|---|---|---|---|---|---|
| `lex` 纯词法 | 75.7% | 91.9% | 0.818 | 0.823 | **224ms** | **543ms** |
| `auto`（旧判据 40/1.3） | 75.7% | 94.6% | 0.821 | 0.825 | **1397ms** | 1760ms |
| `hybrid` | 75.7% | 94.6% | 0.821 | 0.825 | 1136ms | 1899ms |

`top1` 三档**完全相同**。向量换来的是 `recall@5 +2.7 点 = 37 条里多命中 1 条`，
`nDCG +0.003`、`MRR +0.002`（≈ 噪声），代价 `p50 ×6.2`。

### 11.2 回退的真实收益

```
回退率 65%（24/37）
名次变化： 无变化 21 | 变好 1 | 变差 1 | 从无到有 1
其中「lex 本来就已经排对 top-1」却仍然回退：15/24 = 62.5%
代价：lex 188ms → hybrid 1326ms（每条 +1139ms）
```

逐条只有 3 条有变化：`l_nginx 包 说明`（MISS→5，**唯一真收益**）、
`l_notepad_server 变更 历史`（5→4，仍是 miss@1）、`l_tray 服务 怎么配`（2→**3**，**变差**）。

### 11.3 阈值扫描：质量对阈值完全不敏感

42 组 `(S, R)` 组合（S 40~70 × R 1.3~3.0）扫下来 **top1 / recall / nDCG / MRR 一字不差**，
只有回退率（即延迟）在涨。这说明**阈值只决定花多少时间，不决定答得对不对** ——
所以唯一正确的方向是**少回退**，而不是「调准」判据。

### 11.4 新阈值 25 / 1.0 的推导

要区分的只有三条用例：

| 用例 | lex top1 | top1/top2 | lex→hybrid | 需求 |
|---|---|---|---|---|
| `l_nginx 包 说明` | **19.1** | 1.02 | MISS→5 | **必须回退** |
| `l_tray 服务 怎么配` | 29.9 | 1.18 | 2→**3** | 必须**不**回退 |
| `权限模式 autopilot 实现` | 45.4 | **1.00** | 1→**2** | 必须**不**回退 |

比值项夹不住 `1.00` 与 `1.02`，只能靠分数 → `S ∈ (19.1, 29.9]` 取 **25**；
`R ≤ 1.00` 取 **1.0**（等于**关掉比值项**）。
反直觉但实测如此：**「top1 == top2 并列」恰恰是 hybrid 帮不上忙的场景** ——
并列时前排没有信息可分辨，语义召回把原本第 1 的挤到第 2。

### 11.5 落地与实测（v3.5.5）

| | 改前 | 改后 |
|---|---|---|
| `auto` 回退率 | 65% | **11%**（4/37） |
| `auto` p50 | 1397ms | **230ms** |
| `auto` top1 / recall@5 | 75.7% / 94.6% | **75.7% / 94.6%（未丢）** |
| `auto` nDCG@5 / MRR | 0.821 / 0.825 | 0.822 / 0.828 |
| 不传 `mode`（默认档） | 1365ms / `hybrid` | **138ms / `auto`** |

默认档同时从 `hybrid` 改为 `auto`（`routers/search.py` + `routers/kb.py`）：`auto` 在 top1/recall 上
与 `hybrid` 持平、nDCG 略高，而 p50 **230ms vs 1136ms**。

> ⚠️ **测量订正**：本节中途曾用过一个手写脚本，**漏传 golden 用例的 `sources=note,kb` 过滤**，
> 跑出过 `top1 78.4% / nDCG 0.838` 的假提升，并据此写过一段「是同期语料变化所致」的归因 ——
> **那段是错的，已删**。权威口径以 `kb_eval` 为准：**top1/recall 完全不变，只有延迟从 1397ms 降到 230ms**。
> 教训：验证脚本必须**原样复用**既有评测工具的参数拼装（`kb_eval.search()`），
> 自己重写一遍 `params` 就很容易漏掉 `sources` 这类过滤条件，把测量误差当成结论。

### 11.6 顺带查清的

- **`/api/search/route` 没有变慢**：预热后 30 次采样 **min 4ms / 中位 17ms / p95 316ms**。
  此前测到的 498ms 是冷启噪声（max 1524ms）。
- **`hybrid` 比 `auto` 便宜**是个误导：`hybrid` 只跑一遍，`auto` 跑「lex + 按需 hybrid」两遍。
  回退率降到 11% 之后，`auto` 的两遍成本摊薄 → 反而更便宜（220ms < 1136ms）。

### 11.7 遗留

- `Rez-Docs 有哪些文档` 仍失败的根因是 `_nav_prior` 误伤目录页（见 §3 与 v3.5.3 的 `_nav_intent`），
  与本次阈值无关 —— 它现在**会回退 hybrid，但 hybrid 也救不了**（两个 want 都在前 5 之外）。
  真正的修法是独立的低权重 FTS `path` 列，或给「导航意图」更高的分类精度。
- 长句/因果类（`为什么我改了 search_index.py 之后服务没有生效`）仍靠 `long_query` 强制回退，
  是当前 4 次回退之一，也是 recall@5 唯一值得再争的 1 条。

---

## 11. 实施记录（2026-09-28 当天落地）

> 本节只记**实际做了什么、量到什么**。上面 §1~§10 是方案原文，保持原样便于对照。

### 11.1 逐项落地

| 项 | 落地内容 | 位置 | 实测 |
|---|---|---|---|
| P0 症状串库 | `_symptom_pick` 的 rel 兜底**限定同库**（key 前缀 `code:<库>:` 推出作用域）；跨库返回 `None` 而不是猜一个 | `search_index.py _symptom_pick` | **影响面 56 / 2462 篇 code 文档（2.3%）**，全是 `README.md` / `build.bat` / `config.py` / `package.py` / `CHANGELOG.md` 这类**每个包都同名**的路径，横跨 45 个库 |
| P0 存量脏数据 | 新增 `refresh_symptoms(conn)` + `index_all.py --refresh-symptoms`：按当前规则重算并回写 FTS `symptom` 列（不扫文件、不走网络） | `search_index.py` / `tools/index_all.py` | 改写 **1280 / 2462** 篇（含「本该有症状却为空」的补写）；终检 **不一致 0**：1343 篇有精确条目、1119 篇应为空，全对 |
| P0 向量侧锚点 | 无需新代码 —— `vec_docs.anchor` 逐篇比对天生覆盖 | `search_vec._anchor_hash` | 重嵌前 **待重嵌 56 篇**（与串库清单**完全一致**，互相验证）；`vec_rebuild --default-scope` 已在跑 |
| P0 未索引可见 | 响应新增 `unindexed_packages`（`label` / `reason` / `hint`），只在**点名**命中 `never_index` 库时触发；`mode_reason` 里也带上「点了名但库没索引」 | `search_index._mentioned_unindexed` + `search()` 返回 | `ChatRoom 消息 发送` → `[{label: ChatRoom, reason: never_index, hint: wuwor … index_all --libs ChatRoom}]`；普通查询不误报 |
| P1 mode 可观测 | 显式 `lex/hybrid/sem` 也返回 `mode_used` / `mode_reason`（此前只有 `auto` 带） | `search()` 返回 | `mode=lex` → `mode_used=lex`、`reason=显式 mode` |
| P1 confidence 校准 | `_confidence(score, top, second)`：`0.50 绝对 + 0.35 相对 top-1 + 0.15 领先差`；`sem` 模式跳过；**不进排序 key** | `search_index._confidence` / `_recalibrate_confidence` | `卡片 name 和 label 区别` 这类弱榜 top-1：**0.19 → 0.532**；并列 top-1（26/25）仍 **0.474 < 0.5**，弱结果信号没丢 |
| P1 文件名召回 | `_path_bonus(units, rel)`：文件名命中 `+0.15`/词、目录段 `+0.05`/词，**封顶 +0.25** | `search_index._path_bonus` | 单测：`CHANGELOG.md` 命中 `changelog` → ×1.15；目录名命中 → ×1.05；封顶 1.25 |

### 11.2 踩到的坑

1. **改「默认关」会把单次请求一起挡死**（已记入一期文档 §9.3 P0(f)）：
   `_rerank_decision` 先看 `rerank_enabled` 再看请求参数 → `&rerank=1` 失效。
   修法：`rerank_switch()` 返回 `True/False/None` 三分，优先级改成 **单次请求 > 显式配置 > 默认**。
2. **`_symptom_pick` 不该改签名**：作用域完全可以从 key（`code:<库>:<rel>`）推出来，
   改签名会波及 `symptom_for` / `anchor_for` 两个调用链 —— 保持 `(table, key, rel)` 不动。
3. **测试进程必须设 `L_NOTEPAD_ROOT`**：`_family_path()` → `dbmod.default_db_path()` 依赖它，
   否则 `index_skip.json` 找不到 → `never_index_libs()` 为空 → 单测假失败（不是代码 bug）。
4. **块头/锚点污染不止 FTS**：`search_fts.symptom` 改完是 0 行残留，但**向量里还有 201 块**
   脏锚点文本 —— 「改规则」和「洗存量」是两件事，少一步就等于没改。

### 11.3 命令

```bat
:: 改过症状匹配规则后（不回网络，秒级）
wuwor l_notepad_server -- python -m l_notepad_server.tools.index_all --refresh-symptoms

:: 向量侧锚点变化 → 增量重嵌（`--default-scope` 排除默认不搜的库）
wuwor l_notepad_server -- python -m l_notepad_server.tools.vec_rebuild --default-scope

:: 改了 .py 必须重启服务
wuwo svc restart l_notepad_api
```

### 11.4 与预期的偏差

| 计划预期 | 实际 | 原因 |
|---|---|---|
| 「症状串库」影响面按小样本估 | **56 篇 / 2.3%**，但**分布极毒** | 中招的全是 `README.md` 这类每包同名文件，且症状恰好是 l_notepad 的「搜索卡顿」→ 一次污染 45 个包的入口文档 |
| `refresh_symptoms` 预计改写 56 篇 | 实际 **1280 篇** | 除了 56 篇串库，还有大量「本该有症状但 FTS 里是空」的存量行（历史版本写入路径不同）——**回写等价于全量重灌后的状态**，不是破坏 |
| path bonus 预期「文件名类 Recall@5 +10 点」 | **未验证** | 现语料里按文件名提问的用例太少（golden 15 条里只有 1 条），**这个数字没有证据支撑**，需 §6 扩集后再测 |
| `CHANGELOG.md` 召不回 → 加 path bonus 可解 | **解不了** | 查询是中文「变更 历史」，文件名是英文 `CHANGELOG` —— 词面不重叠，bonus 不会触发。真正的解是**查询侧同义扩展**（`变更/历史 → changelog`）或 §3 方案 A 的独立 `path` 列 + 中英对照，**未做** |

### 11.5 最终回归（15 条 golden set，本机）

| 配置 | top1 | recall@5 | nDCG@5 | p50 | p95 |
|---|---|---|---|---|---|
| 二期改动前（一期收尾态） | 73.3% | 93.3% | 0.825 | 1437ms | 1757ms |
| **二期后 `rerank=0`（默认）** | **80.0%** | **100.0%** | **0.888** | **1133ms** | 1602ms |
| 二期后 `rerank=1` | 73.3% | 100.0% | 0.881 | 2493ms | 3538ms |

四项全涨：**top1 +6.7 点、recall@5 +6.7 点、nDCG +0.063、p50 −304ms**。
剩余 3 条「miss@1」的性质完全不同（见 11.7 第 1 条）。

### 11.6 顺带修掉的收敛 bug（顺藤摸瓜发现的，不属于本方案）

追查「症状脏锚点有多少篇」时发现 `vec_rebuild` **永远收敛不了**：每轮都把同一批 ~65 篇当「待嵌」重嵌，
一路空转到 80 轮上限，白烧 CPU。根因是 `vec_docs.anchor` 两侧不同构：

| 位置 | 对「没有中文锚点」的文档（笔记 / 知识库 / 无症状条目的代码）取什么值 |
|---|---|
| 写入（`_upsert_doc` 的两处 `vec_docs` INSERT） | `_anchor_hash("")` = **`da39a3ee5e6b4b0d`**（`sha1("")[:16]`） |
| 比对（`_pending_docs`） | **`""`** 空串 |

→ 永远不相等 → **这 61 篇（40 知识库 + 21 笔记）每轮重嵌一次**。
修法三处：写入侧改成 `_anchor_hash(anchor) if anchor else ""`；比对侧同样按「空文本 → 空值」；
存量行用 `_norm_anchor()` 把 `sha1("")` 归一成 `""`（兼容旧库）。
实测 **pending 61 → 0**。

> 同一个文件里 788-791 行已经因为同类问题栽过一次（「两篇 0 字符的笔记让 refresh 空转 60 轮」），
> 这次是同一个坑换了字段 —— **「不存在」的表示法必须只有一种**。

### 11.7 这次测量本身踩的三个坑（比代码更值得记）
1. **`kb_eval` 只打印 `want` 的第一条 → 读表的人会误判**。
   `热重载 改了源码不生效` 的 `want` 有**两条**（`src_hot_reload_*` 与 `dev_mod_*`），
   正确答案在第 2 名，但输出只打 `sorted(want)[0]`，看着就像召不回。
   已修：打印**全部** `want`，并标注「在第 N 名（多答案用例）」还是「不在前 k，真召不回」。
2. **重排的结论在三次测量里翻了三次**（净负 → 正 → 负 → 正 → 负），根因不是重排本身，是**样本太小**：
   15 条里翻 1 条 = **±6.7 点**，nDCG 只看小数点后第三位。中途还夹着一个更蠢的原因 ——
   **本方案文档自己被索引进去，在 golden 查询里当了第 2 名**，把 recall 从 93.3% 顶到 100%，
   让我一度以为「重排恢复了正收益」。摘掉它之后立刻回落。
   → **结论：15 条不够做这种判断**，§6「扩到 40 条」不是可选项。
   在此之前，**默认关重排**的依据是「质量指标不劣 + 快 2.2 倍」（80/100/0.888/1133ms vs 73.3/100/0.881/2493ms），
   **不是**「重排有害」。
3. **方案类文档必须进 `index_skip.json` 的 `files`**（一期文档早就进了，二期这篇忘了）：
   评测集里的查询词会被方案文档正文逐字命中，它就成了「自指的答案」，
   实测直接改写了 recall@5。已补进名单并摘除存量行。

### 11.8 评测集扩到 40 条后的**真实画像**（本文最该看的一节）

§11.5 的 80.0% 是 **15 条**量出来的，而那 15 条里 **12 条属于 A 组（Rez-Docs 主文档）**——
恰好是检索最强的区域。扩到 40 条（37 排序 + 3 断言）后：

| 配置 | top1 | recall@5 | nDCG@5 | MRR | p50 | p95 | 未索引提示 |
|---|---|---|---|---|---|---|---|
| **`rerank=0`（默认）** | **75.7%** | **94.6%** | **0.822** | **0.825** | **1176ms** | 1486ms | **3/3** |
| `rerank=1` | 73.0% | 94.6% | **0.839** | 0.815 | 2561ms | 3288ms | 3/3 |
| Δ | **−2.7** | ±0 | **+0.017** | −0.010 | +1385ms | | |

> ⚠️ 本节数字**改过四次**：56.8%（我 D 组 `want` 抄错）→ 70.3%（修正 `want`）→ 73.0%（B 组
> `want` + 导航意图门）→ **75.7%**（路径折扣 + 软放宽）。**一个评测集能把结论带偏 19 点**。

**结论一：真实 top-1 是 75.7%。** 15 条那版的「73.3% → 80.0%」是**采样偏差**
（A 组占 80% 权重）。A 组本身的提升是真的（10/12），但**它不代表知识库**。

**结论二：重排在 40 条下是「真·权衡」，不是「有害」也不是「无害」**
（top1 −5.4、MRR −0.009 对 recall +8.1、nDCG +0.025）。默认仍关 —— 依据是
**agent 先读 top-1**（top1/MRR 更重要）＋ **快 2 倍**；要召回优先的场景可以单次 `&rerank=1`。

**结论三：分档强弱差距极大，后续不该再拿总分说事**（`rerank=0`，逐条人工核过）：

| 组 | 用例 | top-1 命中 | 画像 |
|---|---:|---:|---|
| A Rez-Docs 主文档 | 12 | **10（83%）** | 强项。miss 的 2 条都在第 2 名 |
| B 包内 `.md` | 6 | 3（50%） | **弱**：同名的 `rez_pkg/l_tray.md` 主文档总是赢 `l_tray/docs/*.md` |
| C 代码 / 实现文件 | 8 | 5（63%） | 中等；miss 是**同包内**功能近邻（`chat_modes.py` 抢 `permissions.py`） |
| D 口语症状 | 6 | **6（100%）** | **最强**：症状列命中即第 1（前提是 `want` 写对，见 11.9 第 0 条） |
| E 目录 / README | 3 | 1（33%） | `_nav_prior` 的已知代价 + README 互抢 |
| F 未索引库（断言） | 3 | **3（100%）** | 二期新能力，已生效 |
| G 长句 / 中英混 | 2 | 1（50%） | 长句仍不稳 |

**结论四：真实的失败模式是「同一区域内的近邻抢位」，不是跨库同名文件**：

| 查询 | 该赢的 | 实际赢的 | 性质 |
|---|---|---|---|
| 权限模式 autopilot 实现 | `l_agent_chat/permissions.py` | `l_agent_chat/chat_modes.py` | **同包**近邻 |
| jwt 主密钥 轮换 | `lugwit_auth/jwt_keys.py` | `lugwit_auth/secret_store.py` | **同包**近邻 |
| l_tray 服务 怎么配 | `l_tray/src/l_tray/doc/services_help.md` | `rez_pkg/Rez_pkg/服务发现与IPC.md` | 包内文档 vs 知识库主文档 |
| l_tray 小工具 启动 修复 | `l_tray/docs/small_program_launch_fix.md` | `rez_pkg/Rez_pkg/l_tray.md` | 同上 |
| Rez-Docs 有哪些文档 | `rez_pkg/README.md` | `l_frp` / `pyfory` / `l_thread_safe` 的 `README.md` | 唯一真·跨库同名（README 层） |
| 为什么我改了 search_index.py 之后服务没有生效 | 热重载两篇 | 完全无关的 3 篇 | 长句没召回 |

> ⚠️ **本表推翻了我上一版的结论**。上一版写的是「头号失败模式 = `config.py`/`CHANGELOG.md`/
> `test_*.py` 跨库同名抢位」，并给了 4 个例子 —— 那些例子**全部来自写错 `want` 的 D 组用例**：
> 正确答案其实都排第 1，我拿「被污染的映射」当了 ground truth（见 11.9 第 0 条）。
> 跨库同名只在 **README/目录页**这一层成立（E 组），不是全局病根。
> **教训：失败分析必须先确认 `want` 是对的**，否则会拿着假失败去设计不该做的改动。

→ 下一轮的真优先级：**① B 组（包内文档排不上来）② E 组（目录页被 `_nav_prior` 压）③ G 组（长句）**。
C 组的同包近邻需要**函数级/符号级**信号（`code_head` 已有符号清单，可做符号命中加权），
和「查询→库归属」是两件事。

#### 11.8.1 路径折扣 + 软放宽（最近一轮，75.7% 怎么来的）

| # | 改动 | 结果 | 判定 |
|---|---|---|---|
| ① | **路径折扣** `_path_prior` / `_PRIOR_PATH=0.8`：命中词**只出现在路径串里**时打折 | top1 73.0 → **75.7%**、recall 89.2 → **91.9%**、nDCG +0.023、MRR +0.021，**零延迟代价** | **保留** |
| ② | **软放宽** `_RELAX_SOFT_HITS=10`：段间 AND 命中 < 10 也放宽成 OR | recall 91.9 → **94.6%**、nDCG 0.805 → **0.822**、MRR 0.812 → **0.825**，top1 不变、延迟不变 | **保留** |
| ③ | 长句召回（G 组） | **未实施**（见下） | 记录待做 |

**① 的收益不是来自它的目标用例** —— 这点必须写清楚：E-1（`Rez-Docs 有哪些文档`）**仍然失败**。
折扣确实生效了（那些包 README 的 `prior` 从 1.0 变成 0.8），但 45 条并列候选只降 20% 不够看，
真索引页 `rez_pkg/README.md` 仍在 40 名开外（它在 68 条候选里排不上，一点）。
**真正升上来的是另一条用例**：`l_notepad 搜索接口 怎么调`（从第 2 名到第 1 名）。
→ 结论：**路径折扣本身是站得住的（净正、零成本），但「路径 token 无区分度」这个问题没被它解决**，
它是治了同一病因的另一个症状。E-1 需要的是**给路径来源的匹配一个独立、低权重的通道**
（FTS 单独 `path` 列，与 §3 方案 A 合并），而不是继续叠乘法系数。

**② 的机制**（已记进 `_RELAX_SOFT_HITS` 注释）：查询里带一个**泛动词**时它会被当成必需段 ——
实测 `权限模式 autopilot 实现` 只召回 **5 条**（> 原来的阈值 3，所以不放宽），
而正确答案 `permissions.py`（含 `MODES = ("default","allow_all","autopilot")`、
`effective(ask, "autopilot")` 与三条断言）**因为不字面含「实现」被整个排除在候选之外**。
放宽后它进了候选（recall 命中），但 top-1 仍是 `chat_modes.py`（它字面含全部三段、coverage=1.0）。
→ **`chat_modes.py` 也答对了这个问题**（它决定 default/allow_all/autopilot 三档下「直接执行还是问一次」），
按多答案原则应一并进 `want`；但那会把 C 组变成「测不出问题」，所以**先留着当已知的近邻歧义**。

**③ 长句（G 组）：能修，但机制不对，所以没动。**
诊断：`为什么我改了 search_index.py 之后服务没有生效` 的目标文档 `src_hot_reload_*` 在**第 10 名**，
候选 只有 0.45 —— 一堆泛词（为什么/我/之后/没有）在拉平排序。
**已验证概念注入有效**：

| 查询 | want 名次 |
|---|---|
| `为什么我改了 search_index.py 之后服务没有生效`（原始） | 10 |
| `改了源码不生效 热重载` | **2** |
| `为什么改了 search_index.py 服务没生效 热重载` | **1** |

但**没有可用的机制**：查表扩展（`data/term_families.json`）的契约写的是「现象词族，**同族词互为同义**」，
而 `热重载` 是**原因**不是同义词 —— 塞进 `不生效` 族会让**每个**含「无效 / 不生效 / 没用」的查询
都注入「热重载」，而这个精度风险**当前 40 条评测集测不出来**（没有对抗性用例）。
→ 正确做法是新增一张**症状→主题**映射（或给知识库文档也建症状条目），不是改词族表。
**这也是「先用评测集能测到的东西说话」的一次自觉放弃。**

#### 11.8.2 更早一轮：导航意图门 + B 组 `want` 修正（73.0% 怎么来的）

1. **导航意图门**（`_nav_intent`，已留）：`_nav_prior` 原来无差别压目录块，于是「问有哪些文档」
   反而把索引页压掉。现在「块像目录表」和「用户在找目录」分开判（`_nav_intent` 命中就不打折）。
   边界单测 5/5 触发、5/5 不触发。**在这 40 条上没量出收益**（E-1 仍失败），但**也没造成任何回归**，
   作为「不发荒谬结果」的兜底留着。
2. **两条 B 组 `want` 修正**（+1 top1 / +2 recall）：`l_tray 服务 怎么配` / `托盘 小工具 启动 修复`
   的「竞争对手」`rez_pkg/l_tray.md` 经核对**确实答对了问题**（有「服务管理」小节、
   记了小工具启动与 `load_sibling_module` 约束）→ 按 §11.9.0 的原则一并列进 `want`。
3. **被否决的方案：把「自动生成样板块」纳入降权**（已回退，留档在 `search_index.py` 常量区）。
   理由看着很硬 —— `wuwo doc_pkg` 给每个包 README 插的生成块**逐字写死** `D:/…/Rez-Docs/<文档>.md`
   绝对路径，让 45 个包的 README 变成「Rez-Docs 文档」查询的**等价候选**
   （实测 5 个包并列 35.5 分、命中块完全相同），把真索引页 `rez_pkg/README.md` 挤到 **40 名开外**（共 68 条）。
   但**实测净负**：`右键点击文件夹时，右键菜单未弹出`（D 组）被踩掉，而目标 E-1 依旧失败 ——
   top1 73.0% → **70.3%**、nDCG 0.782 → 0.772、MRR 0.791 → 0.778。**已回退。**
   → 说明「样板块不该被索引」这个判断**不成立**（它确实回答了「这个包怎么跑」类提问）。
   E-1 的真问题在别处：**FTS 对 `rez-docs` 这类「路径 token」没有区分度** ——
   68 条里 45 条都能命中它，靠文件名/路径里的词面就并列了。

### 11.9 仍未做

- **评测集自身的可靠性**（比代码更该守的底线，见下第 0 条）。
- **E 组：路径 token 没有区分度**（E-1 的真因，**路径折扣没能解决**）—— 查询里的 `Rez-Docs` 会命中
  68 条候选里的 45 条（45 个包 README 的生成块都写死了 `D:/…/Rez-Docs/<文档>.md`），
  真索引页 `rez_pkg/README.md` 排到 40 名开外。已试过并保留 `_path_prior`（命中只在路径里 → ×0.8），
  净正但**只治了别的症状**：45 条并列候选各降 20% 不足以让索引页冒头。
  **下一步是给路径来源的匹配一个独立低权重 FTS `path` 列**（与 §3 方案 A 合并做），
  不是继续叠乘法系数。也**不要**再试「整块含生成样板就降权」——已实测否决（§11.8.1 第 3 条）。
- **C 组：同包近邻仍是「多答案歧义」**—— `权限模式 autopilot 实现` 放宽后 `permissions.py` 进了候选
  （recall 命中），top-1 仍是 `chat_modes.py`；两者**都答对了问题**（前者是判定实现、后者是档位语义）。
  这类用例要么按多答案收进 `want`，要么承认它测不出问题 —— **别为了让它『能测出问题』去调权重**。
  `会话 检查点 checkpoint`（want 第 2 名，差 0.3 分）另有真信号：**`checkpoint` 匹配不上 `checkpoints`**
  （FTS5 无词干/前缀），值得单独给**前缀匹配**做一次 A/B。
- **G 组：长句召回**——机制缺位，见 11.8.2 第 ③ 条（概念注入有效，但词族表不能塞「原因词」；
  需要新增**症状→主题**映射，或给知识库文档也建症状条目）。
- **`_path_bonus` 的效果未验证**：「文件名类 Recall 提升」目前只有 1 条用例（E 组 nginx）支撑，扩集后再看。
- **`vec_rebuild` 报的 `pending=1`**：那篇文档的库根已不存在（被 `_pending_docs` 的路径校验跳过），
  属幻影计数，不影响嵌入。
- `l_agent_chat` 侧消费 `unindexed_packages` / 低 `confidence` 时给模型 hint。
- `tools/sync_repo_docs.py` 未真跑（会往知识库工作区写文件 → depot 上传，属共享状态）。

#### 11.9.0 头号教训：**失败分析前先确认 `want` 是对的**

我第一版的 D 组（口语症状）6 条里有 **5 条 `want` 是错的**，来源是「修复串库后**失去**症状的文档清单」——
那份清单里的映射**本身就是被污染的**（正是二期要修的那个 bug 的产物）。后果：

- D 组被量成 **1/6 = 17%**（看着像「症状通道坏了」），真实是 **6/6 = 100%**；
- 总分被低估 **13.5 点**（56.8% vs 70.3%）；
- 我还据此写了一个**完全错误的结论**：「头号失败模式 = 跨库同名文件抢位」，
  并准备去做「查询→库软归属」—— 而那个改动**解决的是不存在的问题**。

正确做法（已固化进 `kb_golden.yaml` 的文件头注释）：

1. `want` 只能来自**权威映射**（如 `symptoms.json` 里 `code:<库>:<rel>` 的精确 key），
   **不能**从「搜索结果」或「某个 bug 的产物清单」里抄；
2. 加完用例先跑一道校验：**`want` 路径必须真在索引里**（已写成 `lk_checkgold` 口径，本次 0 缺失）；
3. **每条 miss 都要先人工确认「正确答案确实是这个」**，再把它当失败去归因 ——
   否则会拿着假失败设计改动。

> 这跟 §11.6（重排结论三次反转）是同一类错误的两面：**那次是样本量不够，这次是标注不可靠**。
> 评测集的两条命根子是**独立性**（upstream 别偷看）和**标注正确性**（ground truth 别来自被测系统）。

- §3 方案 A（FTS 独立 `path` 列）—— 先跑 path bonus 观测，再决定是否迁移；
- §6 评测集扩到 40 条 + `unindexed_notice_rate` / `mode_reason_coverage` / `low_confidence_false_negative_rate` 三个新指标；
- `CHANGELOG` 类「文件名与查询词不同语言」的召回（需要同义扩展，不是加权重能解决的）；
- `l_agent_chat` 侧消费 `unindexed_packages` / 低 `confidence` 时给模型 hint。