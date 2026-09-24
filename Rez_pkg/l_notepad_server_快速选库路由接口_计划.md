# 计划：l_notepad 快速选库路由接口（`GET /api/search/route`）

> 目标版本：`l_notepad_server` 999.0 ｜ 2026-09-24
> 关联文档：`Rez-Docs/Rez_pkg/l_notepad_搜索接口使用文档.md`
> 代码：`l_notepad_server/src/l_notepad_server/{routers/search.py, search_index.py, search_vec.py, knowledge.py}`

## 1. 目标

输入一段**复杂需求**，**快速**返回"相关**知识库**排序"，让 Agent 据此决定读哪几个库。

三条硬约束（决定性）：

1. **必须快**：毫秒级到几百毫秒；慢就不做，把活退回给 Agent。
2. **深度可调**：一个 `depth` 参数控制投入。
3. **预算内必返回**：超预算或依赖不可用 → **降档返回**并标记 `degraded`，绝不阻塞。

## 2. 非目标

- 不做"需求分解 / 多子查询"（那是 `depth=3`，即**交给 Agent 自己处理**，本接口不实现）。
- 不改动 `/api/search` 现有行为、返回结构与打分。
- 请求快路径**不引入**同步网络调用或模型兜底（这正是现状慢的根源）。

## 3. 现状与依据（已核实）

| 事实 | 位置 |
|---|---|
| 端点注册风格：`APIRouter(prefix="/api/search")`，GET 本机直连 8765 免 token | `routers/search.py:17`、搜索文档 §7.1 |
| 文档级检索入口；默认 `hybrid`，无 ollama 时每次约 6s（反复探测 11434） | `search_index.search()` :1334、搜索文档 §6 |
| 长句召回的陷阱：`parse_query` 是**段间 AND**（仅在同一组 OR）；需求含空格/标点分出的多个子句时容易零命中，混合中英/多分句尤甚 | `parse_query` :159、搜索文档 §3 |
| 库目录：`list_bases()` → `name/title/description/article_count` | `knowledge.py:72`、`GET /api/kb/bases` `routers/kb.py:362` |
| 索引行带 `kb_name`，可按库 `GROUP BY` | `search_docs.kb_name` `db.py:121` |
| 语义复用件：embed / 余弦 / 归一化 / 缓存 | `search_vec.py`（`embed_texts` :403、`_dot` :443） |
| **索引源与文档 §6 不符**：kb 索引来自 **depot 已上传归档**（`_kb_sync_one`），**不是**本机 workspace（`_sources()` 只返回个人笔记） | `search_index.py:496` / `:561`、`db.py:115` 注释 |

> ⚠️ 最后一条是本次顺带发现的**文档与代码不一致**（搜索文档 §6 称索引各知识库 workspace）。实现阶段以代码为准，并修正文档。

## 4. 接口契约

```
GET /api/search/route
  q          str   必填   复杂需求原文
  depth      int   默认 1  0..3（3 = 不处理，提示交 Agent）
  budget_ms  int   默认 300 硬时间预算
  limit      int   默认 10  返回库数上限
  sources    str   默认 kb  逗号分隔来源过滤（note / kb）
```

返回：

```json
{
  "query": "...",
  "depth_req": 2,
  "depth_used": 1,
  "degraded": true,
  "reason": "embed unavailable (ollama down)",
  "took_ms": 12,
  "kbs": [
    {"kb_name": "rez_pkg", "score": 6.76, "doc_hits": 33, "best_bm25": -49.48,
     "meta_hits": 2, "vec": 0.0}
  ]
}
```

字段对齐现有风格（`depth_used`/`degraded`/`reason` ≈ 现有 `vec.used/reason`）。

## 5. depth 分档

| depth | 行为 | 机制 | 量级（本机） | 依赖 |
|---|---|---|---|---|
| 0 | 库元数据匹配：需求关键词 vs `name/title/description` 词重叠 | 读 `knowledge_bases` | < 10ms | 无 |
| 1 | + FTS5 词法**按库聚合**（关键词 OR 召回，`GROUP BY kb_name`） | `search_fts` JOIN `search_docs` | ~5–50ms | 无 |
| 2 | + 语义：需求嵌一次，与各库**预建摘要向量**余弦 | `search_vec` | 在线 ~100–300ms；**离线跳过** | ollama + 预建向量 |
| 3 | 不处理 | — | — | 由 Agent 自行分解/多查询 |

## 6. 评分（先给常量，落地后按真实需求标定）

```
score = 0.5 * log1p(doc_hits)   # 库内词法命中文档量（弱权重，防大库霸榜）
      + 2.5 * meta_hits          # 库名/标题/描述命中的关键词块数
      + 1.5 * vec                # 库摘要语义相似度（depth>=2，暂为 0）
```

关键词抽取为**纯 OR**（剔除低信息 bigram），不做段间 AND，故无"覆盖率"字段；`best_bm25` 仅作参考信息，不参与打分。权重为 `search_index._ROUTE_W_*` 常量，落地后按真实需求标定。

排序：`route_score` 降序，`kb_name` 次之。

## 7. 硬规则（决定"快"）

1. **不碰慢路径**：route 不调用 `search_index.search()` 的 `hybrid` 分支，不做 `_vec_scores` 的 ollama 兜底探测。
2. **不使用 `parse_query` 的段间 AND**：route 自建关键词抽取（中文二元 **OR** + 停用词剔除），对长需求天然有召回。
3. **硬预算**：进入某档前先估时/查依赖；将超 `budget_ms` 或依赖不可用 → 降一档 + `degraded=true` + `reason`。
4. **depth2 依赖预建摘要向量**：未预建 → 自动 `depth_used=1`。
5. **`depth=3`**：直接返回 `depth_used=3`、空 `kbs` 或提示，表示"请 Agent 自行处理"。

## 8. 实施阶段

- **阶段 0**：契约 + 骨架。`routers/search.py` 加 `/route` 端点；实现 `depth=0`（元数据匹配，最快可用）。
- **阶段 1（核心）**：`depth=1` 词法按库聚合。`search_index.py` 新增 `keyword_terms(q)` 与 `kb_aggregate(conn, terms, ...)`（一条 SQL）。
- **阶段 2**：观测字段（`took_ms`/`degraded`/`reason`）+ 文档补写 + **修正搜索文档 §6**（索引源澄清）。
- **阶段 3（可选）**：`depth=2`。新增库摘要向量表 + 后台预建/失效重嵌 + route 内一次 embed 与 N 次点积。

## 9. 关键实现点（文件级）

| 文件 | 改动 |
|---|---|
| `routers/search.py` | 新增 `@router.get("/route")`，参数解析 + 预算/降级编排 |
| `search_index.py` | 新增 `keyword_terms()`、`kb_aggregate()`（`GROUP BY kb_name`），复用 `kb_bases()` |
| `knowledge.py` | 复用 `list_bases()`（depth0 元数据） |
| `search_vec.py` + `db.py` | （阶段3）库摘要向量表 + 预建 worker + `_dot` 复用 |
| `Rez-Docs/.../l_notepad_搜索接口使用文档.md` | 补 `/route` 文档；修正 §6 索引源表述 |

## 10. 验证

- **耗时对比**：`curl` 分别打 `depth=0/1/2`，与 `GET /api/search&mode=lex` 对照 `took_ms`。
- **长需求回归**：同一条复杂需求，`/api/search` 预计 `total=0`，`/route` 应仍有库排序。
- **降级**：停 ollama 后 `depth=2` → `depth_used=1`、`degraded=true`；`budget_ms=1` 强制降级。
- **权限**：本机直连 8765（GET 免 token）能取到 kb（kb 对所有登录用户可见，搜索文档 §7）。
- **维护**：改动后 `wuwor l_notepad_server -- python -c "import l_notepad_server"`；`.py` 改动**手动重启**（热重载不可靠，搜索文档 §9）。

## 11. 风险与对策

| 风险 | 对策 |
|---|---|
| 关键词抽取质量差 → 选库不准 | 先"二元 OR + 停用词"，用真实需求回归再调 |
| kb 索引依赖 depot 归档；归档不通时库内零文档 | 元数据（title/description）参与 depth0/1 打分兜底 |
| 摘要向量陈旧/缺失 | 变更触发重嵌 + 惰性兜底，缺失即降级（不发网络） |
| 大库因文档多霸榜 | `log1p(doc_hits)` 弱权重 + 元数据命中为主序 |

## 13. 实施状态（2026-09-24）

- **阶段 0、1、3 已实现**（阶段 3 = depth2 语义）：
  - `routers/search.py`：`GET /api/search/route`。
  - `search_index.py`：`route_terms()` / `route_match_expr()` / `_meta_bases()` / `_kb_aggregate()` / `_kb_semantic()` / `route()`，及 `_ROUTE_*` 常量。
  - `search_vec.py`：`_probe_embed()`（0.3s TCP 快探）+ `kb_semantic_scores()`（按库语义聚合）；depth2 语义不可用/不可达即降级。
- **验证（HTTP e2e，服务已由 src 热重载加载新代码）**：
  - 长需求：depth0 ~0.4ms（元数据 meta_hits=2）；depth1 ~2–8ms → `rez_pkg` `doc_hits=33`。
  - depth2（ollama + bge-m3 在线的本机）：`rez_pkg` `vec=0.607`，score 6.76→7.67，`degraded=false`；**热态 120–190ms**，冷启动首次 6.1s（模型加载）。
  - depth3 → 空 kbs + "交调用方自行分解"说明。
  - `/api/search` / `/api/search/stats` 正常，既有行为未变。
- **文档**：搜索接口文档新增 §1.3 `/route`（含各档耗时与降级规则），并修正 §6 索引源（kb 来自 depot 归档）。
- **遗留**：depth2 冷启动受 embedding 模型加载影响（可后续加预热）；未做服务端需求分解（=depth3，刻意留给 Agent）。

## 14. 优化（2026-09-24，第二轮）

在原实现上又落地了一批（按前一轮"优化空间"清单）：

| 项 | 内容 | 位置 |
|---|---|---|
| B/A2 | score 并入 bm25：`+1.5*tanh(-bm25/20)`，自带 IDF，压低全库高频泛词 | `search_index._ROUTE_W_BM25` / `route()` |
| A5 | depth2 改**库摘要向量**：新增 `kb_vec` 表 + `_kb_vec_build/_ensure`（元数据向量 0.5 + 文档质心 0.5），查询只嵌 1 次 + O(库数) 点积，不再扫全部块 | `search_vec.kb_semantic_scores` |
| A6 | route 快路径跳过 `refresh()`（容忍几秒陈旧，省 stat 全目录） | `_kb_aggregate(refresh_index=False)` |
| A8 | 结果缓存：`(q, depth, budget, limit, sources, user)` 10s TTL，命中 ~0ms | `_route_cache_*` |
| A9 | `budget_ms>0 且 <100` → `depth>=2` 自动退回 `1`（`reason_code=budget_downgrade`） | `route(budget_ms=...)` |
| A10 | 新增 `reason_code`（`ok`/`delegate`/`budget_downgrade`/`semantic_degraded`）+ `cached` 字段 | `route()` |

**复验（本地单元，`py_312` 直连库）**：`d2 budget=50` → `used=1/budget_downgrade`；`d2 budget=300` → `used=2`（score 9.14，含 vec+bm25）；`d1` → score 8.23（含 bm25）；重复 `d1` → `cached=true, 0.0ms`；`d3` → `delegate`。

**未做（需你数据/后续）**：A1 标注集调权重、A7 embedding 预热、A4 IDF 词块过滤、A3 库描述/别名增强。

**环境备注**：本轮多次 HTTP 复验被 `SrcHotReload` 的重启窗口打断（8765 短时 502/拒连，随后自愈），浏览器/脚本重试即可；生产建议按机制手动重启。

## 15. 前端搜索参数（2026-09-24，1+2）

**问题**：顶栏弹窗固定 `mode=lex`（长句/自然语言零命中 → 弹窗显示"无命中"，实测复现）；完整列表页 `/web?q=` 只传 `q`、未传 `mode`（走默认 `hybrid`），与弹窗口径不一致。

**方案**：新增 `mode=auto` —— `search_index.search_auto()`：先 `lex`（毫秒级），仅在零命中时回退 `hybrid`；响应多一个 `mode_used`。

**改动**
| 文件 | 改动 |
|---|---|
| `search_index.py` | 新增 `search_auto()` |
| `routers/search.py` | `/api/search` 支持 `mode=auto` |
| `routers/web.py` | `web_list` 新增 `mode` 参数（默认 `hybrid` 不变；`auto` 走 `search_auto`），上下文回传 `mode`/`mode_used` |
| `templates/base.html` | 顶栏表单加 `<input hidden name=mode value=auto>`；弹窗改 `mode=auto`；「查看完整列表」链接带 `&mode=auto` |
| `templates/web_index.html` | 试搜 `mode` 选择器加「自动」选项 |
| `搜索接口使用文档.md` | §1.1 `mode` 增补 `auto` |

**验证**：长句 `auto` → `mode_used=hybrid`、18 命中（top=`Rez_pkg/l_homepage.md`）；短句 `auto` → `mode_used=lex`、2.4ms；`/web?q=…&mode=auto` → 200 且含命中卡片。

**未做**：`sources` 参数未加（1+2 范围外）；知识库页搜索的 `mode` 未加（第 3 项，另议）。

## 16. 独立全局搜索页（2026-09-24）

**需求**：单独一个搜索路由，能搜所有知识库和笔记（不寄居在笔记列表页 `/web?q=`）。

**实现**
| 文件 | 改动 |
|---|---|
| `routers/web.py` | 新增 `GET /web/search`（`web_search`）：参数 `q / mode(auto) / sources / kb / rerank / limit / offset`；检索一次取到 `MAX_LIMIT`，服务端切片分页 + 知识库分面；**注册在 `/web/{note_path:path}` 之前**。模式说明常量 `MODE_HELP` |
| `templates/web_search.html` | 新页面：搜索表单（mode/sources/kb/rerank/limit 五项选择）、当前模式一句话说明、**「搜索帮助」<details> 面板**（逐项解释参数/模式/查询语法，含 `auto` 释义）、命中统计、库分面（可点选过滤/清除）、结果卡片、分页、空态引导 |
| `templates/base.html` | 顶栏表单 `action` 改为 `{{ web_base }}/search`；弹窗「查看完整列表」指向 `/web/search?…&mode=auto` |

**验证（HTTP）**：空态 200；长句 `mode=auto` → 18 卡片、模式 `hybrid`、`📚 知识库 13 / 📝 笔记 5`、分面 `rez_pkg 13`、摘要 `<mark>` 高亮正常；`mode=hybrid&sources=kb&kb=rez_pkg&rerank=0&limit=20` 各项选择器回显正确；`sources=kb`、`sources=note` 过滤生效；帮助面板与「当前模式」提示正常；均无模板错误。模板改动即时生效，无需重启。

**`auto` 语义（写进页面帮助）**：先词法（毫秒级），零命中才回退语义兜底；有词法命中就快，长句也不会"无命中"。

**修复与增强（同日）**
- 修 `rerank` 报错：表单未选时提交空串，`Optional[int]` 触发 `int_parsing`；改为 `rerank: str` 手动解析（空 = 跟随全局）。
- 搜索历史：`localStorage`（键 `ln_search_history`）记最近 20 条，`window.lnSearchHistory` 供顶栏与搜索页共用；搜索页自带 `<datalist id="ln-search-history-page">`（该页无顶栏），顶栏用 `ln-search-history`，初始填充延到 `DOMContentLoaded`。
- 结果卡片改**复用 `LN.renderSearchHits`**（app.js）：命中原始数据（含 `open_url`）以 JSON 注入页面，前端渲染，得到与顶栏弹窗/「搜索索引」页一致的**丰富参数**（覆盖/近邻/语义/重排/块/词频/bm25/时间/短语）。

**说明**：`/web?q=`（笔记列表页内检索）保留不动；新页面为目标落点。

## 17. 代码库索引功能（source=code，2026-09-24）

**需求**：把本机代码库（`l_notepad_client`）纳入索引，用自然语言"ctrl+中键呼出卡很久…"能搜到。

**实现**
| 文件 | 改动 |
|---|---|
| `search_index.py` | 新增 `CODE_EXTS` / `CODE_SKIP_DIRS` / `SETTING_CODE_ROOTS`；`code_roots()` / `set_code_roots()` / `_scan_code()`（os.walk 剪枝 + 扩展名过滤 + 大小上限）；`_PERM_SQL` 加 `source='code'`；`_refresh()`/`_rebuild()` 纳入代码扫描；`stats()` 增 `code_exts`/`code_roots` 与 code 来源行 |
| `routers/search.py` | `GET/PUT /api/search/code_roots`（PUT 管理员，保存后自动后台重建）；`GET /api/search/code/file` 只读查看；`_open_url` 支持 code |
| `routers/web.py` | `sources` 允许 `code`；`_hit_open_url` 支持 code |
| `templates/web_search.html` | 来源下拉加「仅代码库」 |
| `templates/web_index.html` | 新增「代码库索引」卡片（textarea + 保存并重建） |
| 搜索接口文档 | §1.1/§1.3/§1.4/§6/§11 增补 `code`；新增 §1.5 |

**实测（已配置 `l_notepad_client` 目录）**
- 索引 44 个文件（`stats.sources` 出现 `code l_notepad_client docs=44`）。
- 自然语言查询 `ctrl+中键呼出l_notepad_client笔记程序,整个电脑都会卡很久,点击托盘打开笔记窗口就不会有问题`：
  - `mode=lex` → `total=5`，全部 `source=code`，**命中 `folder_favorites_hotkey.py`（#3）**，另含 `local_main.py` / `folder_favorites_widget.py`；23ms。
  - `mode=auto` → `mode_used=lex` 同上；`hybrid` 同结果但 ~4.5s（语义冷启动）。
- 标识符 `folder_favorites_hotkey` → 4 文件命中，含目标文件。
- 只读查看 `GET /api/search/code/file?root=l_notepad_client&file=999.0/src/l_notepad_client/folder_favorites_hotkey.py` → 200 返回源码。
- 配置：本机无登录态直连 8765 是 guest，PUT 需管理员，故本次实验用便携 Python 直接写 `app_settings.code_roots` 并扫描；管理员登录后可直接用状态页卡片或 API。

**结论**：加代码库索引后，这条自然语言查询**能搜到**目标文件（`folder_favorites_hotkey.py` 排第 3）；但命中靠词法（中键/呼出/托盘/notepad/client），代码库暂不参与向量语义（`vec=0`），要语义召回代码需后续把 code 纳入嵌入。

## 12. 验收标准

1. 本机 `depth0 ≤ 10ms`、`depth1 ≤ 50ms`。
2. 复杂长需求 `route` 不返回空（返回库排序）。
3. 任意 `budget_ms` 下都在预算内返回，标记降级原因，绝不阻塞。
4. 不影响 `/api/search` 既有行为；文档更新（含 §6 修正）。
