# l_notepad 搜索改造史（时间线）

> 合并自四篇：`l_notepad_搜索功能增强.md`、`l_notepad_知识库检索优化_计划.md`（一期）、
> `l_notepad_知识库检索优化_二期方案.md`（二期），以及 `l_notepad_server_快速选库路由接口.md` 的 §13–§17 迭代记录。
> **现行接口契约以 `l_notepad_搜索接口使用文档.md` 为准**；本篇只回答「改了什么、为什么改、量到什么、还差什么」。
> ⚠️ 这些文档里的多处数字/结论**后来被订正过**：正文一律写**最新值**，被订正的值保留为
> 「（原为 X，后订正为 Y）」，汇总见 §7。
> 代码侧逐项变更另见 `l_notepad_server/999.0/src/l_notepad_server/doc/CHANGELOG.md`。
>
> 📌 **旧文件名映射**：在代码、工具、测试里若仍看到
> `l_notepad_搜索功能增强.md` / `l_notepad_知识库检索优化_计划.md` / `l_notepad_知识库检索优化_二期方案.md` /
> `l_notepad_server_快速选库路由接口.md`，都指**本文件**（2026-10-04 四篇合一）。
> 已知未同步的残留引用（**刻意保留**：CHANGELOG 是历史日志，`eval_cases.json` / `kb_golden.yaml`
> 是测试基线，改名字可能让期望值失效）：`l_notepad_server/doc/CHANGELOG.md`、`tools/quality_report.py`、
> `tools/eval_cases.json`、`tests/kb_golden.yaml`、`data/index_skip.json`。

---

## 1. 时间线总览

| 日期 | 轮次 | 主题 | 关键结果 |
|---|---|---|---|
| 2026-09-24 | 第一轮 | 快速选库 `route` + `mode=auto` + 独立全局搜索页 `/web/search` + 代码库索引 `source=code` | route depth0 ~0.4ms / depth1 2–8ms / depth2 热态 120–190ms；长句 `auto` 18 命中；代码库 44 文件入索引 |
| 2026-09-25 | 第二轮 | 手动建索引（含代码）、代码参与向量语义、嵌入失败隔离、关键单字+同义词、IDF 泛词抑制、体量治理、「要搜哪些包」勾选、判断依据面板 | 原句→代码库：目标文件第 2 名、`vec=0.6316` |
| 2026-09-26~27 | 症状索引 | 口语症状写进 `search_fts.symptom` 独立列 + 代码块上下文头 + 中英改写/RRF 融合 | 症状通道打通；但多轮测量本身被污染（见 §4） |
| 2026-09-28 | 一期 | 体检 + 整改（P0 重排喂错料 / P1 语料 / P2 语料治理 / P3 auto / P4 硬预算 / P5 评测集） | 15 条 golden top-1 20% → 73.3%（**后订正**：15 条偏乐观，40 条真实 75.7%）；docs 17.3k → 2.5k；**重排最终默认关** |
| 2026-09-28 | 二期 | 症状串库 / 未索引可见 / 文件名召回 / mode 可观测 / confidence 校准 / 评测集扩集 | 15 条 top-1 → 80.0%；40 条真实 75.7%；`auto` 回退率 65% → 11% |

---

## 2. 2026-09-24：第一轮

### 2.1 快速选库路由 `GET /api/search/route`（原 route 文档）

**目标**：输入一段**复杂需求**，**快速**返回"相关**知识库**排序"，让调用方决定读哪几个库。
**三条硬约束**：必须快（毫秒级到几百毫秒，慢就不做）／深度可调（`depth`）／预算内必返回（超预算降档标 `degraded`，绝不阻塞）。
**非目标**：不做需求分解（=`depth=3`，交给调用方）；不改 `/api/search` 行为；快路径不引入同步网络调用或模型兜底。
**现状依据（已核实）**：

| 事实 | 位置 |
|---|---|
| 端点注册风格：`APIRouter(prefix="/api/search")`，GET 本机直连 8765 免 token | `routers/search.py:17`、使用文档 §7.1 |
| 文档级检索入口；默认 `hybrid`，无 ollama 时每次约 6s（反复探测 11434） | `search_index.search()` :1334 |
| 长句召回陷阱：`parse_query` 是**段间 AND**（仅在同一组 OR）；需求含空格/标点时容易零命中 | `parse_query` :159 |
| 库目录：`list_bases()` → `name/title/description/article_count` | `knowledge.py:72`、`GET /api/kb/bases` `routers/kb.py:362` |
| 索引行带 `kb_name`，可按库 `GROUP BY` | `search_docs.kb_name` `db.py:121` |
| 语义复用件：embed / 余弦 / 归一化 / 缓存 | `search_vec.py`（`embed_texts` :403、`_dot` :443） |
| **索引源与文档 §6 不符**：kb 索引来自 **depot 已上传归档**（`_kb_sync_one`），**不是**本机 workspace | `search_index.py:496` / `:561` |

> 最后一条是当时顺带发现的**文档与代码不一致** —— 已按代码修正使用文档 §6（见使用文档 §6 索引源）。

**接口契约**（现行参数以使用文档 §1.3 为准，此处记初版）：`q` 必填；`depth` 默认 1（0..3）；`budget_ms` 默认 300；`limit` 默认 10；`sources` 默认 `kb`（**后改为 `kb,code`**）。返回含 `query/depth_req/depth_used/degraded/reason/took_ms/kbs[]`。

**评分（初版）**：`score = 0.5*log1p(doc_hits) + 2.5*meta_hits + 1.5*vec`；关键词抽取为**纯 OR**（剔除低信息 bigram），不做段间 AND，故无"覆盖率"字段；`best_bm25` 只作参考不参与打分。权重为 `search_index._ROUTE_W_*`。排序：`route_score` 降序、`kb_name` 次之。

**实施阶段**：阶段 0 契约+骨架（depth0 元数据）→ 阶段 1（核心）depth1 词法按库聚合（`keyword_terms()` / `kb_aggregate()` 一条 SQL）→ 阶段 2 观测字段 + 文档补写 + 修正搜索文档 §6 → 阶段 3（可选）depth2 库摘要向量。

**关键实现点（文件级）**：

| 文件 | 改动 |
|---|---|
| `routers/search.py` | 新增 `@router.get("/route")`：参数解析 + 预算/降级编排 |
| `search_index.py` | `keyword_terms()`、`kb_aggregate()`（`GROUP BY kb_name`）、复用 `kb_bases()` |
| `knowledge.py` | 复用 `list_bases()`（depth0 元数据） |
| `search_vec.py` + `db.py` | （阶段3）库摘要向量表 + 预建 worker + `_dot` 复用 |

**验证**：`curl` 分别打 `depth=0/1/2` 对照 `took_ms`；长需求 `/api/search` 预计 `total=0` 而 `/route` 仍有库排序；停 ollama 后 `depth=2` → `depth_used=1`；`budget_ms=1` 强制降级；本机直连 8765 免 token 能取 kb；改 `.py` **手动重启**。

**风险与对策**：关键词抽取质量差 →「二元 OR + 停用词」再用真实需求回归调；kb 索引依赖 depot 归档，归档不通时库内零文档 → 元数据参与 depth0/1 打分兜底；摘要向量陈旧/缺失 → 变更触发重嵌 + 惰性兜底，缺失即降级；大库霸榜 → `log1p(doc_hits)` 弱权重 + 元数据命中为主序。

**实施状态（2026-09-24）**：阶段 0、1、3（= depth2 语义）已实现。
- `routers/search.py`：`GET /api/search/route`。
- `search_index.py`：`route_terms()` / `route_match_expr()` / `_meta_bases()` / `_kb_aggregate()` / `_kb_semantic()` / `route()` 及 `_ROUTE_*` 常量。
- `search_vec.py`：`_probe_embed()`（0.3s TCP 快探）+ `kb_semantic_scores()`；depth2 语义不可用/不可达即降级。
- 验证（HTTP e2e，服务由 src 热重载加载新代码）：depth0 ~0.4ms（`meta_hits=2`）；depth1 ~2–8ms → `rez_pkg` `doc_hits=33`；depth2（ollama + bge-m3 在线）`rez_pkg` `vec=0.607`、score 6.76→7.67、`degraded=false`、**热态 120–190ms**、冷启动首次 6.1s；depth3 → 空 `kbs` + "交调用方自行分解"。
- 遗留：depth2 冷启动受 embedding 模型加载影响（可后续加预热）；未做服务端需求分解（= depth3，刻意留给调用方）。

**第二轮优化（2026-09-24）**：

| 项 | 内容 | 位置 |
|---|---|---|
| B/A2 | score 并入 bm25：`+1.5*tanh(-bm25/20)`，自带 IDF，压低全库高频泛词 | `search_index._ROUTE_W_BM25` / `route()` |
| A5 | depth2 改**库摘要向量**：新增 `kb_vec` 表 + `_kb_vec_build/_ensure`（元数据向量 0.5 + 文档质心 0.5），查询只嵌 1 次 + O(库数) 点积 | `search_vec.kb_semantic_scores` |
| A6 | route 快路径跳过 `refresh()`（容忍几秒陈旧，省 stat 全目录） | `_kb_aggregate(refresh_index=False)` |
| A8 | 结果缓存：`(q, depth, budget, limit, sources, user)` 10s TTL，命中 ~0ms | `_route_cache_*` |
| A9 | `budget_ms>0 且 <100` → `depth>=2` 自动退回 `1`（`reason_code=budget_downgrade`） | `route(budget_ms=...)` |
| A10 | 新增 `reason_code`（`ok`/`delegate`/`budget_downgrade`/`semantic_degraded`）+ `cached` 字段 | `route()` |

复验（本地单元，`py_312` 直连库）：`d2 budget=50` → `used=1/budget_downgrade`；`d2 budget=300` → `used=2`（score 9.14，含 vec+bm25）；`d1` → score 8.23（含 bm25）；重复 `d1` → `cached=true, 0.0ms`；`d3` → `delegate`。
**未做**：A1 标注集调权重、A7 embedding 预热、A4 IDF 词块过滤、A3 库描述/别名增强。
**环境备注**：本轮多次 HTTP 复验被 `SrcHotReload` 的重启窗口打断（8765 短时 502/拒连，随后自愈）。

> `_ROUTE_W_BM25` 这项是**第一次订正** route 评分：初版公式里没有 bm25 项。

### 2.2 `mode=auto`（前端搜索参数）

**问题**：顶栏弹窗固定 `mode=lex`（长句/自然语言零命中 → 显示"无命中"，实测复现）；完整列表页 `/web?q=` 只传 `q`、未传 `mode`（走默认 `hybrid`），与弹窗口径不一致。
**方案**：新增 `mode=auto` —— `search_index.search_auto()`：先 `lex`（毫秒级），仅在零命中时回退 `hybrid`；响应多一个 `mode_used`。

| 文件 | 改动 |
|---|---|
| `search_index.py` | 新增 `search_auto()` |
| `routers/search.py` | `/api/search` 支持 `mode=auto` |
| `routers/web.py` | `web_list` 新增 `mode` 参数（默认 `hybrid` 不变；`auto` 走 `search_auto`），上下文回传 `mode`/`mode_used` |
| `templates/base.html` | 顶栏表单加 `<input hidden name=mode value=auto>`；弹窗改 `mode=auto`；「查看完整列表」链接带 `&mode=auto` |
| `templates/web_index.html` | 试搜 `mode` 选择器加「自动」选项 |

验证：长句 `auto` → `mode_used=hybrid`、18 命中（top=`Rez_pkg/l_homepage.md`）；短句 `auto` → `mode_used=lex`、2.4ms；`/web?q=…&mode=auto` → 200 且含命中卡片。
未做（当时范围外）：`sources` 参数、知识库页的 `mode`。

> ⚠️ **auto 的判据后来被改过两次**：① 2026-09-25 晚「命中 <3 或长句」也回退（原「只有零命中才回退」在长句上形同虚设）；② 一期 P3 改为**分数判据** `_auto_confident`，二期再标定为 top-1 ≥ **25** 且 top1/top2 ≥ **1.0**（原为 40 / 1.3），回退率 65% → **11%**。现行判据见使用文档 §1.1 / §1.4。

### 2.3 独立全局搜索页 `/web/search`

**需求**：单独一个搜索路由，能搜所有知识库和笔记（不寄居在笔记列表页 `/web?q=`）。

| 文件 | 改动 |
|---|---|
| `routers/web.py` | 新增 `GET /web/search`（`web_search`）：参数 `q / mode(auto) / sources / kb / rerank / limit / offset`；检索一次取到 `MAX_LIMIT`，服务端切片分页 + 知识库分面；**注册在 `/web/{note_path:path}` 之前**；模式说明常量 `MODE_HELP` |
| `templates/web_search.html` | 新页面：搜索表单（mode/sources/kb/rerank/limit 五项）、当前模式一句话说明、「搜索帮助」`<details>` 面板（逐项解释参数/模式/查询语法，含 `auto` 释义）、命中统计、库分面、结果卡片、分页、空态引导 |
| `templates/base.html` | 顶栏表单 `action` 改为 `{{ web_base }}/search`；弹窗「查看完整列表」指向 `/web/search?…&mode=auto` |

验证（HTTP）：空态 200；长句 `mode=auto` → 18 卡片、模式 `hybrid`、`📚 知识库 13 / 📝 笔记 5`、分面 `rez_pkg 13`、摘要 `<mark>` 高亮正常；`mode=hybrid&sources=kb&kb=rez_pkg&rerank=0&limit=20` 各选择器回显正确；`sources=kb`/`sources=note` 过滤生效；帮助面板正常。

**修复与增强（同日）**：
- 修 `rerank` 报错：表单未选时提交空串，`Optional[int]` 触发 `int_parsing`；改为 `rerank: str` 手动解析（空 = 跟随全局）。
- 搜索历史：`localStorage`（键 `ln_search_history`）记最近 20 条，`window.lnSearchHistory` 供顶栏与搜索页共用。
- 结果卡片改**复用 `LN.renderSearchHits`**（`app.js`）：命中原始数据（含 `open_url`）以 JSON 注入页面，前端渲染，得到与顶栏弹窗/「搜索索引」页一致的丰富参数。

> `/web?q=`（笔记列表页内检索）保留不动；新页面为目标落点。

### 2.4 代码库索引 `source=code`

**需求**：把本机代码库（`l_notepad_client`）纳入索引，用自然语言"ctrl+中键呼出卡很久…"能搜到。

| 文件 | 改动 |
|---|---|
| `search_index.py` | 新增 `CODE_EXTS` / `CODE_SKIP_DIRS` / `SETTING_CODE_ROOTS`；`code_roots()` / `set_code_roots()` / `_scan_code()`（os.walk 剪枝 + 扩展名过滤 + 大小上限）；`_PERM_SQL` 加 `source='code'`；`_refresh()`/`_rebuild()` 纳入代码扫描；`stats()` 增 `code_exts`/`code_roots` |
| `routers/search.py` | `GET/PUT /api/search/code_roots`（PUT 管理员，保存后自动后台重建）；`GET /api/search/code/file` 只读查看；`_open_url` 支持 code |
| `routers/web.py` | `sources` 允许 `code`；`_hit_open_url` 支持 code |
| `templates/web_search.html` / `web_index.html` | 来源下拉加「仅代码库」；新增「代码库索引」卡片（textarea + 保存并重建） |

实测（已配置 `l_notepad_client` 目录）：
- 索引 44 个文件（`stats.sources` 出现 `code l_notepad_client docs=44`）。
- 自然语言查询 `ctrl+中键呼出l_notepad_client笔记程序,整个电脑都会卡很久,点击托盘打开笔记窗口就不会有问题`：`mode=lex` → `total=5`，全部 `source=code`，命中 `folder_favorites_hotkey.py`（#3），另含 `local_main.py` / `folder_favorites_widget.py`；23ms。
- 标识符 `folder_favorites_hotkey` → 4 文件命中，含目标文件。
- 只读查看 `GET /api/search/code/file?root=…&file=…` → 200 返回源码。

**结论**：加代码库索引后这条自然语言查询**能搜到**目标文件（排第 3）；但命中靠词法（中键/呼出/托盘/notepad/client），代码库暂不参与向量语义（`vec=0`），要语义召回代码需后续把 code 纳入嵌入（第二轮已做，见 §3）。

### 2.5 第一轮实测（保留作对照）

| 场景 | 结果 |
|---|---|
| 长句 `mode=auto` | `mode_used=hybrid`，18 命中 |
| 短句 `mode=auto` | `mode_used=lex`，2.4ms |
| `route` depth 0/1/2/3 | 0.4ms / 2–8ms / 热态 120–190ms / 空 + `delegate` |
| 自然语言原句→代码库 | `lex total=5`，目标文件 **#3**，`vec=0` |
| 控制实验（词含"卡顿/主线程/钩子"） | `total=1`，仅目标文件，覆盖 100% |

**当时的结论**：「词法能定位、但不理解」——原句排序由 repo 泛词驱动，「卡」被停用字过滤丢弃，代码库无语义。第二轮的三处修改正对这三点：**单字/同义保留**、**IDF 泛词抑制**、**代码语义**。

---

## 3. 2026-09-25：第二轮功能增强

### 3.1 第二轮增强清单

| # | 功能 | 入口 / 接口 | 关键实现 | 状态 |
|---|------|------------|----------|------|
| 1 | **手动「创建索引」（含代码文件）** | `GET /api/search/index_libs`、`POST /api/search/index_lib`、搜索页「索引管理」面板 | `search_index.local_libs()` 汇总**代码库根(`kind=code`) + 知识库工作区(`kind=kbws`)**；`index_local_lib()` 只扫该库（`CODE_EXTS` 含 `.py/.js/.json` 等），**不上传 depot**；`embed=true` 顺手嵌向量 | ✅ |
| 2 | **代码库参与向量语义（P0-2）** | `search_vec` 嵌入流程 | `_doc_key`/`_pending_docs`/`_doc_text` 支持 `source=code`（读本机文件，键 `code:<label>:<rel>`）；**实测原句命中目标文件 `vec=0.63`** | ✅ |
| 3 | **嵌入失败隔离** | `search_vec.refresh` / `_embed_worker` | 单篇失败**跳过并计数**（`_EMBED_RETRY_MAX=3` 后放弃），不再一篇坏文档卡死整轮；`stats.vec.dropped` 可见 | ✅ |
| 4 | **关键单字 + 同义词** | `route_terms` / `_SYNONYMS` | 单字 `卡/慢/死` 保留（FTS 前缀 `"卡" *`）；bigram 只在**整块都是低信息字**时丢（原来「卡很」会被误杀）；`卡↔卡顿/卡死/阻塞` 同义扩展 | ✅ |
| 5 | **IDF 泛词抑制** | `term_idf()` + `_score` + `route` | FTS5 词表（`fts5vocab`）算词块文档频率，占比 ≥30% 的 repo 泛词不参与覆盖率/词频/近邻/路由打分；`route` 返回 `terms_generic_dropped` | ✅ |
| 6 | **代码库体量治理** | `_scan_code` + `CODE_MAX_*` | 文件数/字节上限（默认 20000 / 512MB）超限即停并标 `capped`；截断时**不删索引行**；`CODE_SKIP_DIRS` 增补 | ✅ |
| 7 | **`route` 支持本机库** | `GET /api/search/route` | `sources` 默认 `kb,code`；depth0 支持本机库标签元数据命中 | ✅ |
| 8 | **代码命中体验** | `GET /web/code` | 只读查看页（行号 + `hl` 高亮，5000 行截断）；结果卡片 🧩 + 库名徽标；`open_url` 统一 | ✅ |
| 9 | **文档** | — | 使用文档 §1.3/§1.4/§1.5/§2/§6/§9/§10；`CHANGELOG v3.4.0` | ✅ |
| 10 | **「要搜索哪些包」勾选** | `GET /api/search/code_packages`、`/api/search?packages=`、搜索页勾选面板 | 货架（`L_NOTEPAD_PKG_ROOT` = `<trayapp>/rez-package-source`）下 54 个 rez 包，`kind=pkg` 手动建索引；勾选只过滤 `source=code`（笔记/知识库不受影响）；**不传/不勾 = 默认范围（全部库 − `index_skip.json` 的 `libs`「默认不搜」），显式勾选则完全按勾的来**；选择存 `localStorage['ln_search_packages']` | ✅ |
| 11 | **「判断依据」对话框** | 每条命中的 `explain` 字段 + 卡片上的「判断依据」按钮（`app.js` / `app.css`） | 分项表（权重×取值=得分：短语/覆盖率/词频/近邻/bm25/语义）、逐词块（IDF 权重 × 出现次数）、泛词（权重 0）、未命中词块、名次与排序主序、**与下一条的分差 + 主因**、一句话结论；纯前端渲染（顶栏弹窗/搜索页/试搜共用） | ✅ |

改动文件（本轮）：`search_index.py`、`search_vec.py`、`routers/search.py`、`routers/web.py`、`templates/{web_search,web_code,web_index,base}.html`、`static/{app.js,app.css}`、`doc/CHANGELOG.md`。

### 3.2 第一轮增强清单（2026-09-24）

| # | 功能 | 入口 / 接口 | 关键实现 | 状态 |
|---|------|------------|----------|------|
| 1 | **快速选库路由** | `GET /api/search/route?q=&depth=&budget_ms=` | `search_index.route()`：depth 0 元数据 / 1 词法按库聚合 / 2 库摘要向量语义 / 3 交回调用方；`reason_code`、结果缓存、bm25 并入打分、`budget_ms<100` 自动降档 | ✅ |
| 2 | **`mode=auto`** | `/api/search?mode=auto` | 先词法，零命中回退 hybrid；返回 `mode_used` | ✅ |
| 3 | **独立全局搜索页** | `GET /web/search` | 一次搜「笔记 + 全部知识库 + 代码库」；参数 `mode/sources/kb/rerank/limit/offset`；知识库分面、搜索帮助面板、搜索历史、结果卡片复用 `LN.renderSearchHits` | ✅ |
| 4 | **前端接线** | `base.html` / `web_index.html` | 顶栏表单 → `/web/search`；左上角 ☰ 菜单加「全局搜索」；搜索历史 datalist；修复 `rerank=` 空串 `int_parsing` 报错 | ✅ |
| 5 | **代码库索引** | `source=code`：`GET/PUT /api/search/code_roots`、`GET /api/search/code/file` | 本机目录纳入检索；`CODE_EXTS`/`CODE_SKIP_DIRS`/`_scan_code()`；`_PERM_SQL` 放行 code；状态页「代码库索引」卡片；搜索页来源「仅代码库」；只读查看 | ✅ |
| 6 | **文档** | — | 使用文档 §1.1–§1.5 / §6 / §11 增补；`CHANGELOG v3.3.0` | ✅ |

### 3.3 实测记录（本机，2026-09-25）

| 场景 | 结果 |
|------|------|
| `route_terms("卡很久 主线程 钩子")` | `['卡很','卡','卡顿','很久','主线','线程','钩子']` —— **「卡」与同义词不再被丢** |
| 同句 `route depth=1` | 4.9ms；`kbs` 含 `rez_pkg`(2.97) 与 `l_notepad_client`(2.42，`kind=code`) |
| **原句 → 代码库**（`程序卡很久，主线程被什么钩子阻塞了`） | `lex total=53`，目标文件 `folder_favorites_hotkey.py` **#2**；`hybrid` 同位置且带 **`vec=0.6316`**（P0-2 达标：口语症状能语义召回代码） |
| `route depth=0 sources=code` | 命中 `l_notepad_client`（`meta_hits=2`，`score=5.0`） |
| IDF 泛词 | `notepad/client/窗口 → 0.0`（`程序/搜索` 保留 2.4）；`route` 结果 `terms_generic_dropped` 回填 |
| 手动建索引 | `POST /api/search/index_lib {"label":"l_notepad_client"}` → `files=44, capped=false, duration_ms=16`（增量无变化） |
| 体量上限（压到 5 文件复测） | `files=5, capped=true`，且**已有 44 行索引没被删**；恢复上限后 `files=44, capped=false` |
| 代码嵌入速度 | 40 个代码文档 23.5s（≈0.6s/文档） |
| **rerank 上线**（本机 llama.cpp + bge-reranker-v2-m3 Q8） | 同一句原话 `mode=hybrid`：目标文件 **rr=+1.004 排第 1**（开 rerank 前是第 3，且 kb 文档以 142 分霸榜）；整轮 **987ms**（rerank 722ms） |
| rerank 调参前后 | 候选不截断 5×900 字符 = **1.55s** → 截到 300 字符 = **0.6s**；llama-server 默认 `-ub 512` 装不下一个候选，必须 `-ub 2048` |
| 纯症状句（去掉 `中键/托盘` 等标识符） | rerank 也救不回来（目标文件第 7）——**语义分（0.58–0.65）本身没区分度** |
| **UI 默认路径暴露的两个问题**（2026-09-25 晚） | ① 长句 `lex` 段间 AND 只剩 **2 条**命中（目标文件不在候选）；② `auto` 只在零命中才回退 hybrid，长句就停在这 2 条上 |
| 修法 + 复测 | 段数 >4 → 全部 OR（`2 → 87` 条候选）；`auto` 在「命中 <3 或长句」时走 hybrid → **`mode_used=hybrid`、目标文件第 1（rr=0.74）**，整轮 ≈1.5s |

**结论**：「口语 → 代码」这条链**打通**（关键单字/同义词保住召回，IDF 抑制泛词，语义分把目标文件稳定在前三）；仍需人工选库/建索引（**有意为之**：不进 depot、不自动上传）。

### 3.4 已知限制

1. **知识库工作区索引是手动的**：改了工作区文件要再点一次「创建索引」（TTL 自动刷新只覆盖代码库根）。
2. **`.py` 不进知识库归档**：`WORKSPACE_EXTS`（也是 depot 自动上传白名单）仍是文档类型；要让知识库归档含代码需另议（体积/配额）。
3. **IDF 是全局统计**：`docs < 20` 时不启用；库很小 + 词很常见时可能误判泛词。
4. **代码块占内存缓存额度**（`L_NOTEPAD_VEC_CODE_CHUNKS=8000`）：代码量极大时会把部分笔记块挤出缓存。
5. **代码根不要指整棵树**：`rez-package-source` 下实测 43 万+ `.py`，超 `CODE_MAX_*` 会截断（截断会少索引，不会误删）。
6. **`route` 默认含本机库**：只要知识库需显式 `sources=kb`。
7. **重排是独立进程**：`D:\Tools\llama.cpp\start_rerank.bat`（llama-server，11435）没起就自动降级回融合排序；改了 `package.py` 的 rerank env 必须**真重启**服务进程——2026-09-25 实测"热重启"只记了一次事件、进程没换（env 仍是旧的 12/400/3）。
8. **纯症状句仍搜不准**：语义分在同 repo 文件间无区分度（0.58–0.65），rerank 只能重排「已经召回的候选」——标识符（`中键`/`托盘`）在候选里才有救。（后由 §4 的症状索引方案部分解决。）

---

## 4. 2026-09-26 ~ 09-27：症状索引方案（口语 → 代码）

**做法**：AI（本地 `qwen2.5:7b`）读每个文件的注释/符号，**只写一段「可能出的毛病」**——用用户口语描述现象（`点了没反应` / `界面卡住不动` / `要等好几秒才动得了` / `功能整个失效了`），落 `data/symptoms.json`，索引时写进 **`search_fts.symptom` 独立列**（`bm25(search_fts, 6, 4, 1)`：标题 6 / 症状 4 / 正文 1）。

**三条踩坑（都很贵）**：
1. **症状塞进 `body` 会稀释**：12 条 intent 拼进正文让部分查询名次**下降**（bm25 长度归一化惩罚）→ 必须独立列。
2. **症状单独建 FTS 小表会把 IDF 打死**：5 份症状彼此雷同 → `df=5/5` → **bm25 恒 `-0.0`** → 加成永远 0。BM25 的命门是**全语料稀有度**，别把它切碎。
3. **Python 侧覆盖率加成权重太小**：`+3×覆盖率` 翻不动 20+ 分的 bm25 项 → 必须让症状**参与召回与 bm25 排分**。

**生成器**：`tools/symptom_build.py`（7B 本地、离线、增量按 mtime+模型名、逐条落盘、`--dry-run/--force/--all-libs`）。

> **§4.1 的实测（8 条 A/B，症状关 → 开）**：seen `0.112 → 0.511`、held_out `0.240 → 0.538`。
> ⚠️ **后订正**：这两组是**开卷上限**（那 5 条症状是"看过目标文件之后手写"的，等于把答案抄进索引），**不是系统能达到的指标**；且随后发现测量方法本身有 bug（见下），**全部作废**。

### 4.1 规模化踩坑：症状同质化会把 IDF 摊平（2026-09-26 晚）

| 症状覆盖 | seen MRR | held_out MRR |
|---|---|---|
| 无 | 0.112 | 0.240 |
| **5 个文件（精选、每条带条目特有对象）** | **0.511** | **0.538** |
| **45 个文件（批量、通用句"点了没反应/界面卡住"）** | **0.080** ⬇️ | **0.220** ⬇️ |

**机制**：批量生成的症状句彼此雷同 → 这些词在 45 篇文档里都出现 → `df` 暴涨 → **IDF 崩** → 症状信号被摊平。**通用症状句等于把"查询语言"变成另一堆泛词**。
**修法（已实施）**：① 提示词强制「专有名词 + 症状」（正例 `按 Ctrl+中键呼出收藏面板时整个电脑卡好几秒`，明确禁止 `点了没反应`）；② **质量闸门**（中文注释 < 3 行的文件跳过）；③ 备用：症状列内二次 IDF；④ 结论：这条路的上限由**症状的特异性**决定，不是条数 —— 宁可少而准。

### 4.2 开卷上限 vs 干净现状；以及"查询侧扩同义词"为什么零效果

**纠正 §4.1 那张表**：`0.511 / 0.538` 是**开卷值**（看过目标文件后手写），不是系统指标。

### 4.3 测量方法本身踩的坑（最贵的一条）

`search()` **只查索引、不刷新**：`_scan_code` 不跑，`search_fts` 的 `symptom`/`title` 列就一直是旧的。之前所有"症状开/关"的对照数字，**都是在陈旧列上量的**——包括 `0.511/0.538` 和 `0.261/0.234`，**两组全部作废**。
**正确姿势**：每次对照前先 `_scan_code(conn, label, root, force=True)` + `conn.commit()`，再查一次 `SELECT title, symptom FROM search_fts WHERE rowid=?` 确认新内容真的落库；多变体对照用**进程内 monkeypatch** 开关，别去改 `data/` 目录。脚本见 `%TEMP%\ab_measure.py`。

### 4.4 变体的诚实对照（7 条 A/B，每次都强制重灌）

> ⚠️ 原为 8 条，其中 `剪贴板历史是怎么采集的，会不会一直占 CPU 资源` 是**AI 凭空编的用例**——没有真实来源，预期目标也是脑补，**已删**（连带依赖它的结论一起作废）。

| 配置（7 条） | seen MRR | held_out MRR |
|---|---|---|
| 症状关 | 0.107 | 0.293 |
| 症状关 + 关掉上下文头（`title=rel`） | 0.107 | 0.293 |
| ~~症状文本 + 追加现象族词表~~（旧代码，已删） | 0.112 | 0.225 |
| **症状文本原样（现行代码）** | **0.167** | **0.283** |

逐条（症状关 → 症状文本原样）：口语原句 8→**6**、点了没反应 8→**4**、窗口等几秒 14→**12**、搜索时卡住 33→**25**，快捷键 1→1，面板迟钝 7→**16** ⬇️，开机启动 X→35 ⬇️。

**三条结论**：
1. **症状文本本身有用但只在"症状类"提问上**：seen +56%，held_out 反而 -3.4%。变差的是反向词表不匹配——目标文件症状没写"迟钝" / 根本没有对应症状句。
2. **把现象族词表追加进症状列是负收益，已删**（0.169 → 0.112，8 条口径）。机制同 §4.1：族被 30+ 文件标上，`没反应/无反应/失效` 就成了全库泛词 → 族词表**只保留查询侧扩展**（`route_terms`），不写进索引文本。
3. **上下文头（B）对这 7 条零影响**：名次逐条不变。它只在查询提到**包名/符号名**时才起作用，保留但不把它当症状问题的解药。

**查询侧扩同义词仍然零效果**：加 `data/term_families.json`（27 个"现象族"、122 词条）后 `route_terms` 肉眼可见生效，但名次一格没动——**查询侧的族扩展在文档侧没有落点**。
> **结论：瓶颈在文档侧，而"文档侧补词表"这条路被 IDF 封死**——两侧共用泛词 = 两侧都没有信息量。

### 4.5 用例来源纪律（血的教训）

**已发生的两次污染，都是"答案是 AI 给的"**：

| 污染 | 表现 | 处理 |
|---|---|---|
| 开卷手写症状 | 知道测试查询后写症状 = 把答案抄进索引 → `0.511/0.538` | 作废，改盲生成 |
| **AI 编造用例** | 编出 `剪贴板历史是怎么采集的` 这类"像用户会问"的句子，再脑补它该命中哪个文件 → 结论完全建立在虚构上 | **删除该用例，连带依赖它的结论作废** |

**规则**：① 用例**必须有真实来源**（用户原话 / 文档 FAQ / CHANGELOG 问题描述 / 真实报错），AI 想出来的只能标"待验证草稿"，**不得**作判据；② 一条用例必须能说清"**为什么该命中它**"，说不出就是脑补，删；③ **负样本必须有**；④ 条数太少（7 条里 1 条变差 ≈ -4% MRR），结论容易翻转，扩到 30–50 条再谈；⑤ 出处标注在用例文件里（`eval_cases.json` 加 `from` 字段）。

### 4.6 修法：受控词族（只用于生成，不写索引）+ 代码块上下文头

**A. 症状生成改成"受控词表 + 族标注"**（`tools/symptom_build.py`）——**只影响生成，不写进索引**
- prompt 里直接给出 27 个现象族的清单，要求每条输出 `- [族名] <触发/对象> + <现象>`
- 落库格式加 `families`：`{"text","families","mtime","model"}`；**没标族的旧条目自动重生成**
- 标签是**封闭小集合**（27 个），模型只做"选标签"这个简单判断
- 副作用是好的：症状文本会说 `卡顿 / 反应迟钝 / 没有反应`，而不是自由发明
- ⚠️ **一开始还想"把族词表追加进症状列让两侧对齐"——实测负收益，已删**（见 §4.4 第 2 条）。`families` 标签只留作元数据（`stats.symptom.tagged`），不参与索引文本

**B. 代码块上下文头**（`search_index.code_head`）
- `[包名] 相对路径` + `符号: 顶级 def/class 清单`（`.py` 走标准库 `ast`，零新依赖）+ 模块首行说明
- 检索侧写进 `title` 列（bm25 权重 6）；向量侧 `search_vec.embed_doc` 给**每个块**冠上这个头
- 对齐成熟 agent 的 contextual chunk header（Cursor / Kilo 的"自然语言能搜到代码"主要靠它）

**新踩坑（又一次同质化，这次是 prompt 导致的）**：第一轮改动后 34 条里 **8 个文件写出同一句话**「按 Ctrl+中键呼出收藏面板时整个电脑卡好几秒」——模型**照抄 prompt 正例**。两条防线：① 示例改成**另一个项目**（记账系统）的对象，并明确"不许照抄示例里的对象"；② 加确定性兜底 `drop_echoes()` + 每次生成后打**同句撞车诊断**（`echo_report`）。

### 4.7 症状 / 中文锚点覆盖率（2026-09-27 傍晚）

**缺口先量出来**（`quality_report` §4 新增）：

| 范围 | 覆盖 | 说明 |
|---|---|---|
| 默认搜索范围 | **265 / 2,467 = 10.7%** | 默认范围 = 全部库减去 `index_skip.json` 的 `libs` |
| 全部代码 | 279 / 17,178 = **1.6%** | 含 14.7k 篇"默认不搜"的 vendored |

**降成本四招**（原来 17k 篇 × 13s ≈ 60 小时）：① **不换模型**（实测 `qwen2.5:1.5b` 质量更差，速度也没见快 → 保持 7B）；② `--workers 3` 并发；③ `--kind src` + `--default-scope`（只做主战场，≈1,298 篇）；④ `num_predict=400` 收住模型自由发挥。

**过程中挖出并修掉的两个真 bug（都是量出来的）**：

| bug | 现象 | 根因 | 修法 |
|---|---|---|---|
| **全局开关做外科手术** | `vec_docs` 代码文档 2,905 → **247** | 把"症状文件 mtime"塞进**全局**嵌入配方签名 → 任何一次症状重生成都清空全部向量 | `vec_docs.anchor` **按文档**记锚点哈希，逐篇比对（症状重生成 138 篇 → 只重嵌 138 篇） |
| **多进程写锁** | CLI 连篇 `database is locked`（80 轮全失败） | 服务端带 src-watch 热更 + **watchdog 兜底拉起**，一直在重嵌持锁 | 批量工具 `db.connect(timeout=60)`：**等锁**而不是失败 |

**另一个已知残渣**：61 篇**活跃日志/空文件**永不收敛——`vec_rebuild` 现在会"连续两轮无进展就点名退出"。

**实测结果**（`--default-scope --kind src --min-comments 0 --workers 3`）：

| 项 | 结果 |
|---|---|
| 生成 | **1,299 篇 / 失败 0 / 耗时 551s（9.2 分钟）/ 均 0.4s 一篇** |
| 为什么这么快 | src 文件小 + 模型已热 + `num_predict=400` 收住长度（预估 13s/篇是"冷+长输出"的情形） |
| 覆盖率 | 默认范围 **265 → 1,395 / 2,467 = 56.5%** |
| 同句撞车 | 43 条（最严重 **×20**「加载大量数据时界面反应迟钝」）→ `dedupe_store()` 去掉 **139 句** → **撞车 0 条** |

**为什么必须去重**：20 篇共享一句 = 这些词 df 暴涨、IDF 塌掉（§4.1 同一机制）。`dedupe_store()` 按 key 排序决定"谁保留"，可复现。

### 4.8 顺带交付 MCP 服务端

`mcp_server.py`：把检索暴露成 MCP（stdio）工具，任何 agent 都能调，直连同一个 SQLite（不需要服务在跑）。踩坑：`wuwor.bat` 会**吃掉 stdin**（`sys.stdin.read()` 长度为 0），配置必须**直指便携 python + 显式 env**（详见使用文档 §1.7）。

---

## 5. 2026-09-28 一期：体检 + 整改（原《知识库检索优化_计划》）

> 性质：一次**实测体检报告 + 整改方案 +（当天）落地复核**。每条问题带复现命令与实测数字。
> 被检对象：`l_notepad_server`（`127.0.0.1:8765`，SQLite + FTS5 + bge-m3 向量 + bge-reranker-v2-m3 重排）与消费方 `l_agent_chat`（`notepad_search` / `notepad_read` 工具）。

### 5.1 结论速览（体检）

| # | 问题 | 严重度 | 一句话 | 修完预期 |
|---|------|--------|--------|----------|
| P0 | 重排喂错料，把正确答案踩出前三 | **致命** | 送给 cross-encoder 的是 **18 字的标题块**，而重排分是**唯一排序主序** | top-1 命中 1/5 → 5/5；查询 1.2s → 0.35s（**后订正：0.35s 未达成**，见 §5.11） |
| P1 | 文档语料只有 40 篇，仓库一半知识没入库 | 高 | `AGENTS.md`、`.cursor/rules/*.mdc`、`wuwo/*.md`、`Doc/`、`openspec/` 全在库外；`default` 库 0 篇 | 可检索文档 61 → ~120 篇 |
| P2 | 85% 的索引给「默认不搜」的库做嫁妆 | 中 | 14,672 / 17,325 docs 属 `default_off` 库；DB 1.29 GB；每次预热 9s | DB ≈ 0.4 GB，预热 ≤ 3s（**后订正：DB 文件不会自己变小**，见 §5.10） |
| P3 | `auto` 档位形同虚设 | 中 | 判据是「命中数 ≥ 3」，bigram OR 召回下永远成立 → **从不**回退语义 | 零召回/低分查询真正有语义兜底 |
| P4 | 尾延迟没封死 | 中 | 抽样首查一次 **>180s 无响应**（稳定态 1.2s） | 硬预算，最坏 1.5s 返回 `degraded`（**后订正：不是墙钟上限**，见 §5.11） |
| P5 | 没有离线评测集，分数也无量纲 | 高（元问题） | `score` 在 11~95 间飘；P0 这种退化上线三周无人发现 | 一条命令出 nDCG@5 / Recall@5，回归可拦 |

**一句话**：管道（增量索引、FTS5、块级向量、重排、来源过滤、包过滤）已相当完整，**但没有评测闭环** —— 于是一个「喂料 bug」让最贵的那一环（重排）变成了净负收益，而且没人知道。

### 5.2 现状画像（实测）

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

本机库 docs 分布（前 6 占 85%，且**全部**标了「默认不搜」）：ChatRoom 5,346 / postgresql 3,567 / l_comfyui_frontend 3,178 / l_comfyui_ai 1,530 / l_mayaPlug 1,051 / conemu 100（这 6 个合计 **14,772**，占 17,325 的 **85%**），另 Lugwit_Module 512、余 47 库合计 ~2,000。

检索档位实测（同一查询 `服务卡片 怎么重启`）：

| 档位 | 耗时 | 备注 |
|---|---|---|
| `mode=lex&sources=note,kb&rerank=0` | **199 ms** | 纯词法 |
| `mode=lex&sources=note,kb` | 1,231 ms | 开重排（默认） |
| `mode=auto&sources=note,kb` | 1,201 ms | `mode_used=lex`（未回退） |
| `mode=hybrid&sources=note,kb` | 7,504 ms | 走 ollama bge-m3 |
| `/api/search/route` | 160 ms | 选库快路径 |

### 5.3 P0 — 重排喂错料，把正确答案踩出前三【致命】

**症状（A/B，五条真实提问，`rerank=0` vs `rerank=1`，4 变差、1 持平、0 变好）**：

| 提问 | `rerank=0` 第一名 | `rerank=1` 第一名 | 判定 |
|---|---|---|---|
| 变体哈希 空壳包 | `Rez_pkg/变体哈希与wuwo的处理方法.md` (62.96) | `README.md` | 正确文档**掉出前三** |
| 单实例守卫 端口被占 | `solo_单实例守卫模式.md` (95.07) | `src_hot_reload_源码热重载与主页常驻.md` | 正确文档**掉出前三** |
| 相册 回收站 数据结构 | `相册功能与数据模型.md` (66.08) | `Rez_pkg/lugwit_baidu_netdisk.md` | 换成无关文档 |
| wuwo 启动报错 找不到包 | `Rez包创建和启动指导文档.md` (53.69) | `Rez_pkg/wuwo_gui.md` | 次优 |
| 热重载 改了源码不生效 | `l_agent_chat使用指南.md` | 同上 | 持平 |

同时**耗时涨 4~6 倍**：200~310 ms → 900~1,250 ms。即查询 **83% 的时间花在重排上，买到的是更差的排序**。

**先排除「模型不行」**：直接打 `:11435` 的 `/rerank`，模型是正牌 cross-encoder（`单实例守卫` 对正例 3.417、无关 -6.309），判别力正常。**问题不在模型。**
- 补充：`/api/search/stats` 里 `rerank.model` 显示为空串（`L_NOTEPAD_RERANK_MODEL` 没设）—— 不是故障，但状态页看不出实际模型，建议回填 `/v1/models`（一期已做，见 §5.10）。

**根因：送进去的是标题块**。三层因果：
1. **块选择退化成标题块**。`_best_chunk()`（`search_index.py:3013`）按「短语命中数 > 覆盖率/词频」挑块，18 字标题块覆盖率 100% 必然压过 900 字正文块。**缺长度归一**（BM25 里靠 `b` 参数解决的经典问题）。
2. **喂给 cross-encoder 的料没有上下文**。18 字标题 → `-3.63`；README 的「文档索引表」把「变体哈希」「空壳包」写在同一行 → `+1.13` 赢了。对照：只把**文档路径/标题**送去重排，排序立刻正确（`变体哈希与wuwo的处理方法.md -1.599` 第 1，`README.md -8.894` 垫底）。→ **模型没错，料错了。**
3. **重排是唯一排序主序**。`_apply_rerank()`（`search_index.py:3126`）的 `order_key` 只用 `sigmoid(rerank) × prior`，`score` 仅作平手次键 → 62.96 分（第二名 32.66，领先 1.9 倍）的词法冠军被坏窗口 `-3.63` 踩到第 6。

**行业先进做法（设计取舍的依据）**：

| 做法 | 谁在用 | 对应本问题 |
|---|---|---|
| **RRF 融合名次**（`Σ w/(k+rank)`） | Elasticsearch 8.x `rrf` retriever、OpenSearch hybrid、Weaviate、Qdrant | 重排**不独占主序**，与词法名次融合（本仓已有 `_rrf_fuse()`，中英双路用） |
| **Phased ranking**（一阶段便宜召回、二阶段窗口内重排 + `rank-score-drop-limit`） | Vespa | 重排只能窗口内**微调**，不能把一阶段冠军扫出结果页 |
| **Parent-document / auto-merging retrieval** | LlamaIndex `AutoMergingRetriever`、LangChain `ParentDocumentRetriever` | 标题块只用于**定位**，喂重排/LLM 时换父块 |
| **Contextual Retrieval**（每块前置文档级上下文） | Anthropic 2024 | 正是「18 字标题块无上下文」的标准解药 |
| **Rerank 输入带标题前缀、目标 ~512 token** | Cohere Rerank 最佳实践、BGE-reranker 官方示例 | `_trim_for_rerank` 应前置 `rel`/标题 |
| **nDCG@k / Recall@k 离线评测**（BEIR、Ragas） | 所有检索系统 | 见 P5 |

**整改方案（三步，按收益排序）**：
- **第 0 步（立即、可逆）**：先关重排止血 `curl -X POST 127.0.0.1:8765/api/search/rerank -d '{"enabled": false}'` → 查询 1.2s → 0.2~0.3s，top-1 正确率 1/5 → 5/5。
- **(a) 喂料带上下文：标题前缀 + 短块补齐父块**：`search_vec.rerank_input(query, hit)` = `标题/rel 前缀 + 有上下文的正文窗口`；`RERANK_MIN_CHARS=200`，块 < 200 字时用同篇相邻块补足（`doc_window`，parent-document 轻量版）。`rerank_docs` 接 hits（或先组好字符串传入）。
- **(b) 排序改 RRF 融合**：`_RRF_K=60`、`_W_LEX=_W_RERANK=1.0`，`_fuse_lex_rerank()` 用词法名次 + 重排名次做加权 RRF，先验乘在融合分上；`_annotate_ranks(..., order_by="fused")` 并把融合分写进 `explain`。
- **(c) 块选择加长度归一**：`_CHUNK_FULL=400`，`damp = min(1.0, len(text)/400)`，`key = (meta["phrase_hits"], meta["score"]*damp)`。
- **(d) 护栏 rank floor**：`_RANK_FLOOR_RATIO=1.5`，词法分领先第二名 ≥1.5 倍者保底进前 3（对齐 Vespa rank-score-drop）。

**预期（后订正见 §5.10/§5.11）**：5 问 top-1 1/5 → 5/5；正确文档进前三 3/5 → 5/5；`sources=note,kb` 耗时 1.0~1.5s → 0.6~0.9s（重排开）/ 0.2~0.3s（关）。

### 5.4 P1 — 文档语料只有 40 篇，仓库一半知识没入库

**问题**：`kb` 只有两个库：`rez_pkg` **40 篇**、`default` **0 篇（空壳）**；`note` 21 篇停更三周 → `sources=note,kb` 可选范围只剩 **61 篇**。仓库里这些高价值文档**全在库外**：`AGENTS.md`、`.cursor/rules/service-lifecycle.mdc`、`wuwo/README.md`、`wuwo/doc/CHANGELOG.md`、`wuwo/重构.md`、`Doc/`、`openspec/`、各包 `rez-package-source/<pkg>/**/*.md`。最后一条尤其别扭：包目录里的 `.md` 被当成 code 命中，「查文档」的正确姿势（`sources=note,kb`）反而**搜不到它们**。

**行业做法**：按 doc-type 分层而不是按物理来源（ES/OpenSearch `_index`+filter、Weaviate class+tenant）；仓库文档自动入库（Cursor/Copilot codebase index 对 `*.md` 与代码分桶、Danswer/Onyx/Glean connector 定时拉文档目录）；知识库以「问题-答案」为单元沉淀。

**整改**：
- **(a)** 立刻把 `.md` 从 code 桶升级成文档桶：写入时按扩展名给 `doc_type`（`DOC_EXTS = {".md",".mdc",".rst",".txt",".adoc"}`），检索侧 `sources` 含 `kb`/`note` 时额外放行 `doc_type='doc'` 的 code 行。收益：`rez-package-source/**/*.md` + `wuwo/**/*.md`（已在 code 索引里）**立刻**可检索，零新增扫描。
- **(b)** 把仓库根文档同步进 `rez_pkg` 知识库：幂等小脚本 `tools/sync_repo_docs.py`（`AGENTS.md` / `wuwo/{README,doc/CHANGELOG,重构}.md` / `.cursor/rules/*.mdc` → `Rez-Docs/_repo`，内容没变不碰 mtime），挂到 `code_watch` 或定时任务。
- **(c)** 清理 `default` 空库（要么删要么明确用途）。

**预期**：`sources=note,kb` 可检索文档 61 → ~120 篇。

### 5.5 P2 — 85% 的索引给「默认不搜」的库做嫁妆

**问题**：6 个 `default_off` 库合计 **14,772 docs = 85%**（不显式传 `packages` 就不搜）。代价：`db_bytes` 1.29 GB（向量 100,507 chunks，`by_source.code` 占 99%）；每次启动 `warm` 预热 **9.0~9.9s**，热重载让这事一小时内发生 5 次；「TTL 全量比对现场 stat 1.7 万文件、首查 51.9s」的老问题根子也是这批文件。

**行业做法**：索引准入白名单 + 分层（Copilot/Cursor 默认排除 `node_modules`/构建产物/vendored；Sourcegraph `zoekt` shard 分层）；向量只嵌「会被搜的东西」（词法全量 + 向量按需）。

**整改**：**(a)** `default_off` 的库不嵌向量（词法索引保留，显式 `packages` 仍可搜）：`_pending_docs` 里 `if source == "code" and kb_name in _off and libs is None: continue`。**(b)** 让「默认不搜」也能不建词法索引（可选）：`index_skip.json` 的 `never_index` 名单（当时为空）放 `postgresql`/`l_comfyui_frontend` 等，17k → ~4k docs。**(c)** 预热按需：先起服务、预热丢后台，或 `L_NOTEPAD_WARM=0` 跳过。

**预期**：`db_bytes` 1.29 GB → ~0.4 GB（(a)）/ ~0.2 GB（(a)+(b)）；启动预热 9.0~9.9 s → ≤ 3 s。（**后订正**：DB 文件不会自己变小，见 §5.7。）

### 5.6 P3 — `auto` 档位形同虚设

**问题**：`search_auto()`（`search_index.py:3362`）的判据是 `total >= _AUTO_MIN_HITS`（**3**）。中文 bigram OR 宽召回下 `total` 动辄 19/42/554 → **判据永远成立**。实测 5 条提问全部 `mode_used=lex`，**一次都没回退**。于是 1.29 GB 存的 100,507 个向量块在默认档位下**从不参与**；向量对 docs 覆盖率仅 **2,595/17,325 = 15%**。

**行业做法**：按「质量」而非「数量」决定加档（Vespa/Elastic phased ranking 看一阶段分数分布、top-1 分数阈值、top1/top2 分差）；零/低召回才升级（cascade retrieval）；query rewriting / HyDE 作词法失手兜底。

**整改**：新增 `_auto_confident(result)`，判据改**分数分布**：`_AUTO_MIN_TOP_SCORE=40.0`（正确文档 top-1 常 50~95，<40 基本是「蹭词」）+ `_AUTO_MIN_GAP=1.3`（top1/top2 分差不足 → 前排没赢家，语义值得一试），并把判据写进 `mode_reason`。配套：hybrid 7.5s 太慢 → 回退档要加**硬预算**（P4）；向量覆盖率靠 P2(a) 腾配额去嵌 `kb`/`note`/常搜包。

**预期**：`auto` 回退率 0% → 10~20%（**后订正：实测 ~100%**，见 §5.10）；近义词提问可命中。

### 5.7 P4 — 尾延迟没封死

**问题**：抽样第一轮第一条 `/api/search` **180s 超时无响应**；随后稳定态一律 1.0~1.5s，无法复现。上游此前修过一个同类问题（TTL 全量比对现场做 → 首查 51.9s，已改丢后台）。**风险点不是「慢」，而是「没有上限」** —— 客户端（`notepad_search` 超时 60s）只能靠超时兜，而超时对 agent 等价于「知识库不可用」。

**行业做法**：软/硬预算 + 优雅降级（Vespa `timeout` + `ranking.softtimeout`、Elasticsearch `timeout`/`terminate_after`、Google「tail at scale」）；返回体自带 `degraded` 标记。

**整改**：`_QUERY_BUDGET_MS = _env_int("L_NOTEPAD_QUERY_BUDGET_MS", 1500)` + `_deadline()`；三个卡点按预算裁掉且**一律先保住词法结果**：① 重排：剩余预算不够就跳过；② 语义/回退：回退前先看预算；③ 返回体标记 `degraded` / `degraded_reason`。`notepad_knowledge.py` 侧同步：`_TIMEOUT` 60s → 20s，`degraded=True` 时给模型 hint。

**预期**：最坏 1.5~2.0s 返回**带 `degraded` 标记的词法结果**。（**后订正**：① 实际 3000ms；② **硬预算不是墙钟上限**，见 §5.10。）

### 5.8 P5 — 没有离线评测集，分数也无量纲【元问题】

**问题**：没有 golden set（P0 那种退化带病运行三周）；`score` 跨查询在 11~95 间飘（`变体哈希` 62.96、`单实例守卫` 95.07、`nginx 反代` 25.16），agent 拿到 25 分无法判断可信度。
**行业做法**：BEIR/MTEB 的 nDCG@10/Recall@k/MRR、Ragas；每次改排序都跑回归；返回校准置信度（Cohere Rerank 返回 0~1 的 `relevance_score`；对 cross-encoder logits 做 sigmoid 标定）。
**整改**：**(a)** 建 golden set（30~50 条，本文已有 13 条现成的，`tests/kb_golden.yaml`）；**(b)** 一条命令出指标 `tools/kb_eval.py`（对 golden set 算 `top1 / recall@k / nDCG@k / P95 延迟`，支持 `--rerank 0 --rerank 1` A/B）；**(c)** 返回校准置信度 `hit["confidence"] = sigmoid((score-40)/12)`，`notepad_search` 返回带 `confidence`，top-1 < 0.5 时给提示。
**预期**：任何排序/分块/重排改动 1 分钟内拿到四个数；P0 类退化合并前被拦下。

### 5.9 实施顺序与验收（原计划）

| 序 | 动作 | 工作量 | 验收 |
|---|---|---|---|
| 1 | 关重排（`POST /api/search/rerank {"enabled":false}`） | 1 分钟 | 5 问 top-1 全对、查询 ≤ 0.35s |
| 2 | 建 golden set + `tools/kb_eval.py`，存基线 | 1 小时 | 一条命令出四个指标 |
| 3 | P0 (a)(b)(c)(d)：喂料带上下文、RRF 融合、块长度归一、rank floor | 半天 | 重排重新打开，`kb_eval` 指标 ≥ 基线 |
| 4 | P1 (a)：`.md` 升级为文档桶（`doc_type`） | 半天 | `sources=note,kb` 能命中包内 `.md` |
| 5 | P1 (b)(c)：仓库根文档同步、清 `default` 空库 | 1 小时 | 「服务怎么重启」命中 `service-lifecycle` |
| 6 | P2 (a)(b)(c)：`default_off` 不嵌向量、`never_index` 名单、预热丢后台 | 半天 | `db_bytes` ≤ 0.5 GB，预热 ≤ 3s |
| 7 | P4：查询硬预算 + `degraded` 标记 | 半天 | 压测下 p99 ≤ 2s，客户端不再超时 |
| 8 | P3：`auto` 改分数判据 + 向量覆盖默认范围 | 半天 | 回退率 10~20%，近义词提问可命中 |
| 9 | P5 (c)：`confidence` + 低置信 hint | 2 小时 | `notepad_search` 返回带 confidence |

**总体预期（计划）**：默认查询 1.2s → 0.6s（重排开）/ 0.25s（关）；`sources=note,kb` top-1 ~20% → ~90%（golden set 口径）；DB 1.29 GB → 0.4 GB；启动预热 9s → 不阻塞。

### 5.10 实施结果（2026-09-28 当天落地，复核过一轮）

> 本节只记**实际做了什么、量到什么、和预期差在哪**。评测资产：`999.0/tests/kb_golden.yaml`（15 条）+ `tools/kb_eval.py`（一条命令出四指标，A/B 自动打 Δ）。体检复现脚本 `d:/Temp/kb_audit.py`~`kb_audit7.py`（`kb_audit.py` 规模 / `kb_audit2.py` 向量 / `kb_audit3.py` 档位耗时 / `kb_audit4.py` 质量抽样 / **`kb_audit5.py` 重排 A/B（§5.3 那张表）** / `kb_audit6.py` 服务身份 / **`kb_audit7.py` chunk 明细（§5.3 根因）**）；落地复核脚本 `d:/Temp/lk_*.py`（`lk_live` 分项探针 / `lk_cost` 档位耗时 / `lk_off` 语料占比 / `lk_warm` 预热拆解 / `lk_check2` 完整性 / `lk_purge`+`lk_vprune` 摘除与清向量）。

**一句话**：**降噪 > 调排序，而「导航页」也是噪声。** golden set top-1：46.7%（修好排序、语料仍脏）→ 66.7%（清掉生成副本 + 自指夹具）→ **73.3%**（给文档目录表降权）→ 80.0%（二期，见 §6）。
> ⚠️ **这串数字全是 15 条量出来的，而 15 条里 12 条是 Rez-Docs 主文档（最强区域）** —— 扩到 40 条后真实 top-1 是 **70.3%**（后订正为 **75.7%**，见 §6.11）。上面的**趋势**可信，**绝对值不可引用**。

三项排序修复（喂料带上下文 / RRF 融合 / 块长度归一）在脏语料上贡献 13.3 点；P2 把语料 17k → 2.5k **不动 top-1**，但把 recall@5 86.7% → **93.3%**。**重排最终默认关**（依据是「不劣且快」，见 §7 订正 3）。15 条口径下剩 3 条 miss@1：`热重载 改了源码不生效`（**多答案用例**，第 2 名就是另一条正确答案）、`l_notepad 搜索接口 怎么调`（同主题近邻抢位）、`CHANGELOG.md`（已升到第 2 名）。

**基线（15 条 golden set，含 ollama:11434 + rerank:11435）**：

| 时点 | 配置 | top1 | recall@5 | nDCG@5 | p50 | p95 |
|---|---|---|---|---|---|---|
| 修好排序（语料仍脏） | `rerank=0` | 33.3% | 86.7% | 0.595 | 1492ms | 1819ms |
| 修好排序（语料仍脏） | `rerank=1` | 46.7% | 86.7% | 0.636 | 2499ms | 2932ms |
| 清镜像 + 自指夹具 | `rerank=0` | 66.7% | 86.7% | 0.767 | 1228ms | 2013ms |
| 清镜像 + 自指夹具 | `rerank=1` | 66.7% | 86.7% | 0.780 | 2557ms | 2852ms |
| +P2(a)(b)（语料 2.5k） | `rerank=1` | 66.7% | 93.3% | 0.796 | 2509ms | 3506ms |
| **+P0(e) 导航块降权（现状）** | **`rerank=0`（新默认）** | **73.3%** | **93.3%** | **0.825** | **1437ms** | **1757ms** |
| +P0(e) 导航块降权 | `rerank=1` | 66.7% | 93.3% | 0.801 | 2711ms | 3106ms |

> **语料**：`docs` 17,325 → **2,515**；向量 `vec_chunks` 100,507 → **55,289**（`vec_docs` 2,488）。`PRAGMA integrity_check` / fts5 `integrity-check` 均 ok，FTS 行数 = 文档行数，孤立向量行 0。
> ⚠️ **DB 文件一个字节都没小**：删行只把页挂到 freelist，文件仍是 `1,292,988,416` 字节 —— 和体检时是**同一个数**（按 10⁹ 写作「1.29 GB」，按 2³⁰ 写就成了「1.20 GiB」，**换算单位不是瘦身**；早先版本把两者摆进「→」里是**本文自己的口径错误，已订正**）。实测 `page_count=315,671 × 4,096`，其中 freelist **192,710 页 ≈ 0.74 GiB** → **`VACUUM` 之后才会掉到约 0.47 GiB**。

**逐项对照（计划 → 实际）**：

| 项 | 计划 | 实际落地 | 偏差备注 |
|---|---|---|---|
| P0 (a) 喂料带上下文 | 标题前缀 + 父块窗口 | `search_vec.rerank_input` + `doc_window`（块 <200 字补到 ~600） | `rerank_docs(trim=False)`：喂料已组好，再截一次会把标题前缀切掉 |
| P0 (b) RRF 融合 | `_RRF_K = 60` | `_fuse_lex_rerank` + **`_RRF_K_FUSE = 60`** | 计划的名字会覆盖 zh_en 双路调好的 `_RRF_K = 10` → 另起常数 |
| P0 (c) 块长度归一 | `_CHUNK_FULL = 400` | 同 | 单测：`变体哈希 空壳包` 选中块 `chunk_no` 0（18 字标题）→ 1（正文） |
| P0 (d) rank floor | 领先 1.5 倍保底进前 3 | 同（直接提到首位） | 比计划略强 |
| P0 补充 | 回填 `/v1/models` | `_rerank_server_model`（60s 缓存，仅状态页用） | 状态页 `model` 不再空串 |
| **P0 (e) 导航块降权**（计划外） | — | `_nav_prior` / `_PRIOR_NAV=0.7`（命中块里 ≥3 行、且过半是 markdown 链接 → 打折） | 长度归一只治「太短的标题块」，没治**900 字的目录表**：`Rez-Docs/README.md` 一行摘要凑齐全部查询词，偷走 2 条 top-1。判据看**命中块内容**不看文件名 |
| **P0 (f) 重排默认关**（计划外） | 计划是「修好后重新打开」 | `rerank_enabled` 默认 `False` + 新增 `rerank_switch()` 区分「显式关 / 没表态」 | ⚠️ **本行结论改过一次，「净负收益」措辞已作废**：二期收尾复测（15 条）`rerank=0` 80.0%/100%/0.888/1133ms 对 `rerank=1` 73.3%/100%/0.881/2493ms —— 质量指标**不劣**、快 2.2 倍。**默认关的依据是「不劣且快」，不是「有害」**（见 §7 订正 3） |
| P1 (a) doc_type | 新增列 + 检索放行 | **不落库**：`_src_clause` 按 `DOC_EXTS` 放行 code 桶里的文档行 | 免迁移、改完立即生效、零重扫 |
| P1 (b) 仓库文档同步 | 平铺复制 | `tools/sync_repo_docs.py`：**保留目录结构**、`.mdc`→`.md`、幂等、`--dry-run/--prune` | 干跑 36 个文件；**未真跑**（会往工作区写 → depot 上传，属共享状态） |
| P1 (c) 清 default 空库 | 删掉或明确用途 | 删不掉（`ensure_default_base` 自动补、`delete_base` 拒绝）→ `list_bases(include_placeholder=False)` | `/api/kb/bases` 只列 `rez_pkg`，`?empty=1` 可取，带 `placeholder` 标志 |
| P2 (a) 不嵌向量 | 同 | 同 + `purge_default_off_vectors` 清存量 | `ChatRoom` 独占 **44,168 / 99,457** 块（44%）→ 摘掉；6 库向量全 0 |
| P2 (b) never_index | 可选 | 6 库全进名单 + `index_all --purge` | 17,286 → **2,515 篇**；顺带补了事件路径漏洞 |
| P2 (c) 预热 | 丢后台 / 可关 | 本来就是后台线程；加 `L_NOTEPAD_WARM=0` | ⚠️ 成本结构与计划假设不符（见下 4） |
| P3 auto 判据 | 分数判据 | `_auto_confident`（top-1 ≥ 40 且 top1/top2 ≥ 1.3）+ `mode_reason`；第一遍纯词法探路且只用一半预算 | 回退从「从不」变成真在回退；实测回退率 ~100% |
| P4 硬预算 | 1500 ms + `degraded` | **3000 ms** + **缩重排窗口** + `degraded` + 嵌入可达性探测 | 两处必须改，否则重排被静默关掉（见下 §5.11） |
| P5 评测集 | 30~50 条 | 15 条（人工确认 `want`）+ `kb_eval.py` | 已立案，后续按需补 |
| P5 (c) confidence | 服务端 + agent hint | 服务端 `confidence` = `sigmoid((score-40)/12)` + 搜索页「判断依据」显示 | agent 侧 hint 未做（`l_agent_chat`，另一个包） |

**计划外发现（都已落成代码/配置）**：
1. **`l_nginx/999.0/runtime/html/docs/**` 是 Rez-Docs 的生成副本** —— 部署时整棵文档树被复制进 nginx 运行时目录，与原件逐字相同却经常排前面（15 条里 6 条 top-1 是它）。`files` 只能按文件名排除 → 新增 **`dirs`** 键（`<库标签>/<相对目录>`，整棵子树不索引）：`_scan_code` 不下钻、`code_watch._rel_ok` 事件侧同拦、`_code_index_one` 兜底、`purge_skip_dirs` 摘存量（实测 30 篇 / 403 块）。
2. **一期文档自己是自指夹具** —— 正文逐字含 golden 查询（4 条 top-1 是它）→ 进 `files`；同时 `_kb_sync_one` 补上 `files` 名单（此前只作用于代码索引），另补 `kb_golden.yaml`。
3. **`never_index` 有事件路径漏洞** —— `_scan_code`/`index_local_lib` 会拒绝整库，但**文件事件绕过扫描**直接走 `_code_index_one`：改一下那个库里的文件就把行写回来。已让 `_code_index_one` 与 `code_watch._rel_ok` 都拦。
4. **预热的成本结构和假设不符**：`_refresh(force=True)` **204 ms** ＋ `sync_kb(default)` 640 ms ＋ `sync_kb(rez_pkg)` **8000 ms** ≈ **8.8 s**。9 秒的大头是 **depot 归档目录列举（网络）**，不是「1.7 万文件 stat」——P2(b) 对预热**没有影响**。杠杆是 `L_NOTEPAD_WARM=0`，或给 `depot_map.list_tree` 加短 TTL 缓存（**建议，未做**）。

### 5.11 与预期的偏差（都是实测校准出来的）

| 计划 | 实际 | 为什么 |
|---|---|---|
| 预算 1500 ms | **3000 ms** | 本机完整管道 = lex 0.15 + 嵌入 1.2 + 重排 1.4 ≈ 2.8s；按 1500 跑，重排会被**系统性**跳过（P0 白修） |
| 卡点用 `RERANK_TIMEOUT_S * 0.6` | 用**每候选实测耗时**（`rerank_cost_per_doc`） | 那是**超时上限**（本机 8s）：`8×0.6` 让「剩余 ≥ 4.8s」永远为假 → 重排被**静默永久关闭**（真踩到了） |
| 预算不够就整档跳过 | 先**缩窗口**（少排几个候选） | Vespa `rank_window_size` 同思路：「排前 3 个」仍优于完全不排 |
| `degraded` 标记所有降级 | 只标**影响结果集**的两档（语义、hybrid 回退） | 重排只影响排序不影响结果集；否则 `auto` 每查都报降级，标记很快失去意义 |
| auto 回退率 10~20% | 实测 **~100%** | 判据是 top1/top2 ≥ 1.3，而当前语料同主题近邻多、分差常在 1.0~1.2 —— 不是判据错，是语料特性 |
| P0 预期「查询 1.2s → 0.35s」 | 重排关 **p50 1437 / p95 1757 ms**；重排开 p50 2711 ms | 计划低估了 ollama 单次嵌入（~1.2s）。0.35s 只在「纯词法 + 不回退」时成立，而回退率 ~100% |
| 硬预算「最坏 1.5s 返回 `degraded`」 | **预算不是墙钟上限** | `_QUERY_BUDGET_MS`(3000) 只在**开下一档之前**判「还够不够」，已进入的阶段（尤其 `embed_texts` 的 HTTP）**没法中断**。实测单查冲到 4.6s。要真封顶得给每档独立超时 + 提前返回，**未做** |
| 重排「恢复正收益」 | **最终仍是负收益 → 默认关** | 降噪 + 导航块降权把词法排序修对之后，重排反而把它踩坏（top1 −6.7 / nDCG −0.023）。这正是 P5 评测闭环的价值：语料一变、结论就反号 |

### 5.12 未做 / 待办（一期）

- **`VACUUM`**：删行只进 freelist（192,710 页 ≈ 0.74 GiB，真空后约 0.47 GiB），需独占写锁 → 建议停服做。**在此之前，任何「DB 变小了」的说法都不成立**。
- **包 README 被注入了别人家的「症状」**（新发现，**未修**）：索引时给每篇代码文档加的块头含一批症状句，但症状**没有按库过滤** —— `l_frp`/`l_nginx`/`l_agent_chat` 的 README 块头里都躺着 l_notepad 的症状（`grep` 原文件确认：文件里没有，是索引时注进去的）。后果：**任何带「搜索/卡顿/没反应」字样的提问，所有包的 README 都是万能候选**。修法是在注入处按 `kb_name` 过滤（影响口语症状通道，需单独评测后再动）。（二期已修，见 §6.2。）
- **`l_nginx/999.0/README.md` 仍压着 `Nginx反向代理机制.md`**：块头把标题重复了两遍 → 词频虚高；正解是块头去重（同一处代码）。
- **`CHANGELOG.md` 召不回**（recall@5 唯一缺口）：P1(a) 已放行 code 桶的 `.md`，但 CHANGELOG 正文极长、查询词密度极低，长度归一后更吃亏。需要**标题/文件名通道**，未做。
- **硬预算不是墙钟上限**：`embed_texts` 进去了就中断不了（单查 4.6s）。要真封顶得给每档独立 socket 超时。
- **`l_agent_chat` 侧**：`notepad_knowledge._TIMEOUT` 60s → 20s；`degraded=True` / 低 `confidence` 时给模型 hint。
- **`tools/sync_repo_docs.py` 未真跑**（会触发 depot 上传）。
- **评测集 15 条偏小**，`want` 只覆盖「top-1 该是谁」；注意 `kb_eval` 的 `recall@k` 是 **hit@k 口径**。
- **一期文档自己在 `index_skip.files` 里**，所以在知识库里搜不到它。

**复现**：

```bash
# 四个指标 + A/B 自动打 Δ（服务需在跑）
wuwor l_notepad_server -- python -m l_notepad_server.tools.kb_eval --rerank 0 --rerank 1
# 改了 search_index.py / search_vec.py 之后必须重启服务才生效（.py 不热重载）
wuwo svc restart l_notepad_api
# 语料治理（幂等）
wuwor l_notepad_server -- python -m l_notepad_server.tools.index_all --purge
wuwor l_notepad_server -- python -m l_notepad_server.tools.vec_rebuild --prune-off
# 仓库根规则/手册同步进 Rez-Docs 工作区（非 dry-run 会触发 depot 上传）
wuwor l_notepad_server -- python -m l_notepad_server.tools.sync_repo_docs --dry-run
```

**附录 B：涉及的代码位置（一期）**

| 位置 | 作用 | 本文对应 |
|---|---|---|
| `search_index.py:3013 _best_chunk` | 挑「命中块」 | P0 (c) 长度归一 |
| `search_index.py _nav_prior` / `_PRIOR_NAV` | 文档目录表降权 | P0 (e) |
| `search_index.py _rerank_decision` | 单次请求 > 显式配置 > 默认 | P0 (f) |
| `search_vec.py rerank_switch` / `rerank_enabled` | 显式开关 vs 默认（默认关） | P0 (f) |
| `search_index.py:3078 _apply_rerank` | 重排并重排序 | P0 (b)(d) |
| `search_index.py:3126 order_key` | **重排独占主序** | P0 (b) |
| `search_index.py:3135 _rrf_fuse` | 已有的 RRF 实现（中英双路用） | P0 (b) 复用思路 |
| `search_index.py:3362 search_auto` / `:68 _AUTO_MIN_HITS` | `auto` 回退判据 | P3 |
| `search_index.py:230 default_off_libs` | 默认不搜名单 | P2 (a) |
| `search_vec.py:1303 _trim_for_rerank` | 400 字窗口 | P0 (a) |
| `search_vec.py:1328 rerank_docs` | 调 `:11435` | P0 (a) |
| `search_vec.py:634 _pending_docs` | 待嵌入文档筛选 | P2 (a) |
| `routers/search.py:41 api_search` | `/api/search` 查询面 | P4 `degraded` |
| `l_agent_chat/notepad_knowledge.py` | agent 侧工具 | P4 超时、P5 hint |

---

## 6. 2026-09-28 二期（原《二期方案》）

> 一期已完成：词法/RRF/块长度归一、导航块降权、语料裁剪、golden set、`confidence`、`degraded`、重排默认关闭。
> 二期不再重复调 RRF 参数，重点转向**语料正确性、未索引可见性、文件名召回、可观测性、评测闭环**。

### 6.1 执行摘要

一期让 Rez-Docs 主场景从约 20% top-1（5 问口径）提升到 **73.3%**（15 条口径），集外抽样约 75%。但整体知识库仍有四个结构性问题：① **症状/中文锚点可能串库**：`_symptom_pick()` 的 rel 后缀兜底没有库隔离——同名相对路径出现在多个包时，症状由字典遍历顺序决定；② **14,772 篇 `never_index` 语料静默消失**（返回噪声而不报「该库未索引」）；③ **文件名召回不稳定**（`rel` 不是 FTS 独立索引列）；④ **普通接口缺统一运行信息**（显式 `lex/hybrid/sem` 没有 `mode_used/mode_reason`；低分但相对最相关的答案被固定绝对分压成低 `confidence`）。

| 优先级 | 项目 | 风险 | 推荐动作 |
|---|---|---|---|
| P0 | 症状按库隔离 | 结果语义错误，agent 可能拿错代码 | 收紧 `rel` fallback；增加同名路径回归测试 |
| P0 | 未索引库显式提示 | 能力边界不可见，agent 把噪声当答案 | 响应增加 `unindexed_packages` |
| P1 | 路径/文件名召回 | 用户点名文件却召不回 | 先加 query-time path bonus，后评估 FTS path 列 |
| P1 | mode 可观测性 | 无法解释为何走 hybrid/lex | 所有响应统一返回 `mode_used/mode_reason` |
| P1 | confidence 校准 | 正确结果被低置信度误弃 | 只改 confidence，不改排序，加入查询内相对分 |
| P2 | holdout 扩展 | 73.3% 可能被小样本放大 | golden 扩到 30~50 条，增加负例/目录页样例 |

### 6.2 P0：症状锚点串库

**问题**：代码索引 key 形如 `code:<库标签>:<相对路径>`（如 `code:l_frp:999.0/README.md`）。症状选择函数支持三层匹配：

```python
if key in table:            return table[key]          # ① 完整 key 精确
if rel and rel in table:    return table[rel]          # ② 裸 rel
for k, v in table.items():                             # ③ 跨库 endswith 猜测
    if rel and (k.endswith("/" + rel) or k.endswith(":" + rel)): return v
```

位置：`search_index.py:705-717`。**第三层没有库标签约束**，多个库存在相同 `rel`（如 `999.0/README.md`、`src/utils.py`）时，症状由 `dict` 插入顺序决定，可能被另一包复用。后果：FTS `symptom` 列出现错误中文语义 → 向量 `anchor_for()` 把错误症状嵌入代码块 → 拒绝相关包被当高频候选 → agent 读错文件。这是**结果正确性问题**。

**当前链路**：`search_index.py:2062-2064` 写代码索引时 `symptom=symptom_for(key, rel)`；`_upsert()`（`:1577-1580`）写入 FTS：

```python
conn.execute(
    "INSERT INTO search_fts(rowid, title, symptom, body, note_path) VALUES(?,?,?,?,?)",
    (rowid, index_text(title), index_text(symptom), index_text(body), key),
)
```

FTS 结构：`search_fts(title, symptom, body, note_path UNINDEXED, tokenize='unicode61 remove_diacritics 2')`。`anchor_for()` 同样通过 `_symptom_pick()` 取职责与症状拼进向量块前缀 → 修复必须覆盖词法与向量两侧。

**推荐方案**：保留症状通道与 `symptom` 列；最小修复规则：① 完整 key 精确命中（元→允许）；② 代码文档禁止裸 `rel` 命中；③ 代码文档禁止跨库 `endswith` 匹配；④ 非代码笔记可保留 `rel` 匹配；⑤ 同一 `rel` 多库同时存在时，**宁可没有症状，也不猜**。

```python
def _symptom_pick(table, key: str, rel: str = "", *, source: str = "", kb_name: str = ""):
    if not table:                return None
    if key in table:             return table[key]
    if source == "code":         return None   # code 文档必须库隔离，不允许 rel 后缀跨库猜测
    if rel and rel in table:     return table[rel]
    return None
```

调用处显式传来源：`symptom=symptom_for(key, rel, source="code", kb_name=label)`。更严格版本只接受 `code:<label>:<rel>`：`table.get(f"code:{label}:{rel}")`。

**行业先进实例**：Anthropic Contextual Retrieval（上下文属于当前 chunk，不跨 chunk/文档复用）、LlamaIndex Parent Document Retriever（子块回所属父文档）、Elasticsearch metadata filter（`tenant_id`/`document_id` 与正文分离，过滤优先于相关性）、多租户 RAG（先 document scope，再 lexical/vector/rerank）。

**验收**：

```python
assert _symptom_pick({"code:a:999.0/README.md": "A"},
    "code:b:999.0/README.md", "999.0/README.md", source="code", kb_name="b") is None
assert _symptom_pick({"code:a:999.0/README.md": "A"},
    "code:a:999.0/README.md", "999.0/README.md", source="code", kb_name="a") == "A"
```

预期：同名路径跨库症状错配 **0**；症状查询 Recall@5 不应下降超过 2%。

### 6.3 P0：`never_index` 语料静默消失

**问题**：`index_skip.json` 的 `never_index` 有 6 个库（`postgresql` / `ChatRoom` / `l_comfyui_frontend` / `l_comfyui_ai` / `l_mayaPlug` / `conemu`），合计约 14,772 篇。查询这些库时返回其他库低分文件，**不告诉调用方「该库没有索引」** → 误判为「知识库没有相关内容」，让 agent 把低分候选当答案。

**推荐方案**：不要立刻恢复 14,772 篇，先让接口诚实表达边界。新增字段：

```json
{"unindexed_packages": [{"label":"ChatRoom","reason":"never_index","hint":"index_all --libs ChatRoom"}]}
```

触发条件：① `sources` 含 `code` 或未指定；② 查询明确包含库名，或结果为空/全部低置信；③ 目标库属 `never_index_libs()`；④ 不在每次查询热路径遍历完整库目录。实现 `_mentioned_unindexed_packages(conn, query, packages)`，在 `search()` 返回组装处（`search_index.py:3655-3669`）加入；更准确的做法是给 `local_libs()`/`lib_rows()` 建短 TTL cache。客户端：`if result.get("unindexed_packages")` → 提示「目标代码库未建立索引」，**不要把低分 hits 当目标库答案**。

**行业先进实例**：Elasticsearch index existence/alias health（索引不存在返回明确错误，不伪造空结果）、GitHub Code Search（scope 显式）、Vertex AI Search（数据源连接失败与「没有相关文档」分开报告）、RAG observability（`no_result`/`filtered_all`/`source_unavailable` 分不同 reason code）。

**结果预期**：未索引库查询 `unindexed_packages` 命中率 100%；正常查询耗时增加目标 `<5ms`；区分「库不存在」「库未索引」「库已索引但无命中」。

### 6.4 P1：文件名 / 路径召回不稳定

**问题**：FTS 只有 `title, symptom, body, note_path UNINDEXED` —— `note_path` 不参与检索，`rel` 只存在 `search_docs.rel`（查询时 SELECT 出来，不能参与 MATCH）。`title`：代码是 `code_head(label, rel, body)`（含路径与符号）；note/KB 默认是 `rel` 整串。因此查 `l_notepad_server 变更 历史` 时 `CHANGELOG.md` 可能进不了前五；路径/文件名/正文权重无法独立调节。

**方案 A（长期）：增加 FTS `path` 列**（迁移成本可控，复用 `migrate_symptom_column()`）：

```sql
CREATE VIRTUAL TABLE search_fts_new USING fts5(
    title, symptom, path, body, note_path UNINDEXED,
    tokenize='unicode61 remove_diacritics 2');
```

写入 `index_text(rel)`；查询权重 `bm25(search_fts, 6.0, _W_SYM_COL, 3.0, 1.0)`（title 6 / symptom 4 / path 3 / body 1）。**不要**把 path 调到 6 以上，避免 README/文件名污染正文。

**方案 B（短期，零迁移）：query-time path bonus**：

```python
def _path_bonus(query, rel):
    q = set(_query_tokens(query)); name = Path(rel).name.lower(); stem = Path(name).stem.lower()
    matched = [t for t in q if t in name or t in stem]
    return (1.0, "") if not matched else (1.0 + min(0.25, 0.08 * len(matched)), "文件名命中")
# search() 评分阶段：meta["score"] = round(meta["score"] * path_mult, 4); meta["path_bonus"] = path_mult
```

B 只能提升已进入 FTS 候选的文档，不能解决「正文/标题完全不命中」的召回 → 先做 B 观测，再做 A。

**行业先进实例**：Elasticsearch `multi_match`（`title^3, path^2, body`）、OpenSearch（BM25 + field boost + rerank）、Sourcegraph（文件名/路径/符号/正文分通道）、GitHub Code Search（`path:`/`repo:`/符号与正文不同检索范围）。

**结果预期**：`CHANGELOG.md` 类文件名问题 Recall@5 +10~20 点；文件名明确命中的查询 top-1 提升；正文查询 top-1 不降超 1 点。

> ⚠️ **后订正**：这条「+10~20 点」**未经验证**（golden 15 条里只有 1 条文件名用例）；且 `CHANGELOG.md` 召不回**解不了**（中文查询 vs 英文文件名无字面重叠，bonus 不触发）。见 §6.11 / §7 订正 14。

### 6.5 P1：所有接口统一返回实际搜索模式

**问题**：`mode_used`/`mode_reason` 当时只在 `search_auto()` 写入（约 `search_index.py:3729-3747`），普通 `lex/hybrid/sem` 直接走 `search()` 没有这两个字段 → 消费者无法知道走没走指定模式、是否因预算跳过语义、是否 hybrid fallback。

**实现**：在 `search()` 统一返回处（`search_index.py:3655-3669`）增加 `mode_used=mode` 与 `mode_reason`（"显式 mode" / "显式 mode，但发生降级：…"），`search_auto()` 继续覆盖。行业对照：Vespa phased ranking（trace 可见实际执行阶段）、Elasticsearch profile API、OpenSearch search pipeline、Google SRE（返回结果与服务状态分离）。

**结果预期**：100% `/api/search` 响应有 `mode_used`/`mode_reason`；不改变排序，只增加可观测性。

### 6.6 P1：confidence 从绝对分数改为查询内校准

**问题**：`_confidence(score)` = `sigmoid((score-40)/12)`（`search_index.py:181-186`）是加权和，不同查询量纲不同 → 例：`卡片 name 和 label 区别` top-1 正确但 `confidence ≈ 0.19`，agent 会把正确答案误判成低可信。

**推荐方案**：**只改 confidence，不改变排序**（confidence 不进 sort key）。查询内相对分更适合表达「是否明显领先」：

```python
def _confidence(score, top_score, second_score=0.0):
    absolute = 1.0 / (1.0 + math.exp(-(score - 40.0) / 12.0))
    if top_score <= 0: return round(absolute, 3)
    relative = max(0.0, min(1.0, score / top_score))
    gap = 1.0 if second_score <= 0 else max(0.0, min(1.0, (score - second_score) / max(top_score, 1.0)))
    return round(max(0.0, min(1.0, 0.55 * absolute + 0.30 * relative + 0.15 * gap)), 3)
```

应用时先确定排序后的 top1/top2，再给每个 hit 填字段。语义模式仍需单独处理余弦分数（不能直接当 0~1 概率）。行业对照：Cohere Rerank（返回 relevance score）、Vespa（rank score 与业务 confidence 分离）、Google Search Quality（置信度是校准层不替换主排序分）、RAGAS/TruLens（retrieval relevance 与最终答案质量分开评估）。

**结果预期**：低绝对分但 top1 明显领先的正确答案不再统一 < 0.5；**不改变 top1/Recall@5/nDCG**；后续可用 50+ golden/holdout 做 isotonic/logistic calibration。

### 6.7 P2：评测集从 15 条升级为可持续回归集

**当前局限**：`tools/kb_eval.py` 支持 `top1` / `recall@k`（hit@k）/ 二元相关 `nDCG@k` / `p50/p95` / `rerank=0/1` A/B；15 条偏小，不能覆盖代码/KB/note 三源、未索引库负例、目录页导航问题、文件名/函数名/长句/口语症状、多正确答案的相关性等级。

**新用例格式**（YAML，含 `expect.status: unindexed` / `top1_any` / `max_rank` / `mode_used` 等断言）：

```yaml
- q: ChatRoom 消息 发送
  sources: code
  packages: [ChatRoom]
  expect: {status: unindexed, package: ChatRoom}
- q: l_notepad_server 变更 历史
  sources: note,kb
  expect:
    want: [l_notepad_server/999.0/src/l_notepad_server/doc/CHANGELOG.md]
    max_rank: 5
```

**指标扩展**（建议至少 40 条：Rez-Docs 主文档 12 / 包内 `.md/.mdc` 6 / 代码函数 8 / 口语症状 6 / 目录 README 3 / 未索引库 3 / 长句 2）：新增 `MRR`、`unindexed_notice_rate`、`mode_reason_coverage`、`low_confidence_false_negative_rate`、`p50/p95/degraded_rate`。行业对照：BEIR、MTEB、Ragas、Google/Netflix SRE。

**结果预期**：golden + holdout ≥ 40 条；每次提交自动输出质量/延迟/降级/未索引四组指标；任一主场景 top1 下降 >5 点阻止发布；`unindexed_notice_rate` 100%。

### 6.8 实施顺序 / 回归命令 / 失败模式 / 最终目标（原二期 §7–§10）

**实施顺序**：
- 第 1 阶段（P0 正确性）：收紧 `_symptom_pick()`（代码文档只允许完整 key）→ 加同名路径跨库回归 → 增加 `unindexed_packages` → 查询明确未索引库时不冒充目标答案 → 线上验证 `ChatRoom`/`postgresql`。验收：跨库症状错配 = 0、未索引库提示率 = 100%、正常查询 Recall@5 无明显下降。
- 第 2 阶段（P1 召回与可观测）：先实现 query-time path bonus → `mode_used/mode_reason` 到显式模式响应 → confidence 改查询内校准 → 对 CHANGELOG/文件名/函数名用例跑 A/B。验收：文件名类 Recall@5 +10 点以上、mode 字段覆盖率 100%、排序指标不因 confidence 改动而变化。
- 第 3 阶段（P2 长期结构）：若 path bonus 有效再迁移 FTS `path` 列 → golden 扩到 40 条 → 增加未索引/目录页/症状串库/低分正确答案用例 → `kb_eval.py` 接入验收。

**统一回归命令**：

```bat
wuwo svc restart l_notepad_api
wuwor l_notepad_server -- python -m l_notepad_server.tools.kb_eval --rerank 0 --rerank 1
wuwor l_notepad_server -- python -m l_notepad_server.tools.kb_eval --golden tests/kb_holdout.yaml
```

每次记录：git revision / DB docs·vec_docs·vec_chunks / rerank enabled / mode_used 分布 / unindexed_notice_rate / Top1·Recall@5·nDCG@5·MRR / p50·p95·degraded_rate。

**失败模式与回滚**：

| 风险 | 现象 | 回滚 |
|---|---|---|
| 症状过滤过严 | symptom Recall@5 明显下降 | 恢复 exact key + 非 code rel 匹配，保留 code 隔离 |
| 未索引提示误报 | 正常 code 查询频繁出现提示 | 只在查询包含库名或 total=0 触发 |
| path bonus 过强 | README/文件名压过正文 | bonus ≤1.25，加入目录块先验保护 |
| confidence 误导 | agent 重查次数增加 | 只回滚 confidence 映射，不回滚排序 |
| FTS path 迁移失败 | 服务启动失败 | 保留 `search_fts_new`，事务内切换，失败 rollback |
| holdout 过拟合 | 指标上涨但真实查询变差 | 固定 holdout 不参与参数调优 |

**最终目标（完整契约）**：正确语料 → 正确 scope → 词法/向量召回 → 融合排序 → 置信度 → 可观测响应 → 离线回归。本期优先：不串库 / 不隐瞒未索引 / 文件名可找 / 模式可解释 / 低分不等于低相关 / 每次改动有 holdout 证据。

### 6.9 性能校准（实测，v3.5.5）

**结论：向量层用 6 倍延迟只换来 1 条召回。** 37 条 golden 逐条对照：

| 模式 | top1 | recall@5 | nDCG@5 | MRR | p50 | p95 |
|---|---|---|---|---|---|---|
| `lex` 纯词法 | 75.7% | 91.9% | 0.818 | 0.823 | **224ms** | **543ms** |
| `auto`（旧判据 40/1.3） | 75.7% | 94.6% | 0.821 | 0.825 | **1397ms** | 1760ms |
| `hybrid` | 75.7% | 94.6% | 0.821 | 0.825 | 1136ms | 1899ms |

`top1` 三档**完全相同**。向量换来的是 `recall@5 +2.7 点 = 37 条里多命中 1 条`、`nDCG +0.003`、`MRR +0.002`（≈ 噪声），代价 `p50 ×6.2`。

**回退的真实收益**：回退率 65%（24/37）；名次变化 无变化 21 / 变好 1 / 变差 1 / 从无到有 1；其中「lex 本来就已经排对 top-1」却仍回退 15/24 = 62.5%；代价 lex 188ms → hybrid 1326ms（每条 +1139ms）。逐条只有 3 条有变化：`l_nginx 包 说明`（MISS→5，**唯一真收益**）、`l_notepad_server 变更 历史`（5→4，仍 miss@1）、`l_tray 服务 怎么配`（2→**3**，**变差**）。

**阈值扫描：质量对阈值完全不敏感。** 42 组 `(S, R)`（S 40~70 × R 1.3~3.0）扫下来 **top1/recall/nDCG/MRR 一字不差**，只有回退率（即延迟）在涨 → **阈值只决定花多少时间，不决定答得对不对**；唯一正确方向是**少回退**，不是「调准」判据。

**新阈值 25 / 1.0 的推导**（要区分的只有三条用例）：

| 用例 | lex top1 | top1/top2 | lex→hybrid | 需求 |
|---|---|---|---|---|
| `l_nginx 包 说明` | **19.1** | 1.02 | MISS→5 | **必须回退** |
| `l_tray 服务 怎么配` | 29.9 | 1.18 | 2→**3** | 必须**不**回退 |
| `权限模式 autopilot 实现` | 45.4 | **1.00** | 1→**2** | 必须**不**回退 |

比值项夹不住 `1.00` 与 `1.02`，只能靠分数 → `S ∈ (19.1, 29.9]` 取 **25**；`R ≤ 1.00` 取 **1.0**（等于**关掉比值项**）。反直觉但实测如此：**「top1 == top2 并列」恰恰是 hybrid 帮不上忙的场景**。

**落地与实测（v3.5.5）**：

| | 改前 | 改后 |
|---|---|---|
| `auto` 回退率 | 65% | **11%**（4/37） |
| `auto` p50 | 1397ms | **230ms** |
| `auto` top1 / recall@5 | 75.7% / 94.6% | **75.7% / 94.6%（未丢）** |
| `auto` nDCG@5 / MRR | 0.821 / 0.825 | 0.822 / 0.828 |
| 不传 `mode`（默认档） | 1365ms / `hybrid` | **138ms / `auto`** |

默认档同时从 `hybrid` 改为 `auto`（`routers/search.py` + `routers/kb.py`）：`auto` 在 top1/recall 上与 `hybrid` 持平、nDCG 略高，而 p50 **230ms vs 1136ms**。

> ⚠️ **测量订正**：本节中途曾用过一个手写脚本，**漏传 golden 用例的 `sources=note,kb` 过滤**，跑出过 `top1 78.4% / nDCG 0.838` 的假提升，并据此写过一段「是同期语料变化所致」的归因 —— **那段是错的，已删**。权威口径以 `kb_eval` 为准：**top1/recall 完全不变，只有延迟从 1397ms 降到 230ms**。
> 教训：验证脚本必须**原样复用**既有评测工具的参数拼装（`kb_eval.search()`），自己重写一遍 `params` 就很容易漏掉 `sources` 这类过滤条件，把测量误差当成结论。

**顺带查清的**：① `/api/search/route` 没有变慢（预热后 30 次采样 min 4ms / 中位 17ms / p95 316ms；此前测到的 498ms 是冷启噪声）；② 「`hybrid` 比 `auto` 便宜」是误导——`hybrid` 只跑一遍，`auto` 跑「lex + 按需 hybrid」两遍，回退率降到 11% 后 `auto` 反而更便宜。

**遗留**：`Rez-Docs 有哪些文档` 仍失败（`_nav_prior` 误伤目录页，与阈值无关；会回退 hybrid 但 hybrid 也救不了，两个 want 都在前 5 之外；真解法是独立低权重 FTS `path` 列或更高精度的「导航意图」分类）；长句/因果类（`为什么我改了 search_index.py 之后服务没有生效`）仍靠 `long_query` 强制回退。

### 6.10 实施记录（二期当天落地）

**逐项落地**：

| 项 | 落地内容 | 位置 | 实测 |
|---|---|---|---|
| P0 症状串库 | `_symptom_pick` 的 rel 兜底**限定同库**（key 前缀 `code:<库>:` 推出作用域）；跨库返回 `None` 而不是猜 | `search_index.py _symptom_pick` | **影响面 56 / 2462 篇 code 文档（2.3%）**，全是 `README.md`/`build.bat`/`config.py`/`package.py`/`CHANGELOG.md` 这类**每包同名**路径，横跨 45 个库 |
| P0 存量脏数据 | 新增 `refresh_symptoms(conn)` + `index_all.py --refresh-symptoms`：按当前规则重算并回写 FTS `symptom` 列 | `search_index.py` / `tools/index_all.py` | 改写 **1280 / 2462** 篇；终检**不一致 0**（1343 篇有精确条目、1119 篇应为空，全对） |
| P0 向量侧锚点 | 无需新代码 —— `vec_docs.anchor` 逐篇比对天生覆盖 | `search_vec._anchor_hash` | 重嵌前**待重嵌 56 篇**（与串库清单**完全一致**） |
| P0 未索引可见 | 响应新增 `unindexed_packages`（`label`/`reason`/`hint`），只在**点名**命中 `never_index` 库时触发；`mode_reason` 也带上 | `search_index._mentioned_unindexed` + `search()` | `ChatRoom 消息 发送` → `[{label: ChatRoom, reason: never_index, hint: …index_all --libs ChatRoom}]`；普通查询不误报 |
| P1 mode 可观测 | 显式 `lex/hybrid/sem` 也返回 `mode_used`/`mode_reason` | `search()` 返回 | `mode=lex` → `mode_used=lex`、`reason=显式 mode` |
| P1 confidence 校准 | `_confidence(score, top, second)`：`0.50 绝对 + 0.35 相对 top-1 + 0.15 领先差`；`sem` 模式跳过；**不进排序 key** | `search_index._confidence` / `_recalibrate_confidence` | `卡片 name 和 label 区别` 弱榜 top-1：**0.19 → 0.532**；并列 top-1（26/25）仍 **0.474 < 0.5** |
| P1 文件名召回 | `_path_bonus(units, rel)`：文件名命中 `+0.15`/词、目录段 `+0.05`/词，**封顶 +0.25** | `search_index._path_bonus` | 单测：`CHANGELOG.md` 命中 `changelog` → ×1.15；目录名命中 → ×1.05；封顶 1.25 |

**踩到的坑**：
1. **改「默认关」会把单次请求一起挡死**：`_rerank_decision` 先看 `rerank_enabled` 再看请求参数 → `&rerank=1` 失效。修法：`rerank_switch()` 返回 `True/False/None` 三分，优先级改成 **单次请求 > 显式配置 > 默认**。
2. **`_symptom_pick` 不该改签名**：作用域完全可从 key（`code:<库>:<rel>`）推出，改签名会波及 `symptom_for`/`anchor_for` 两个调用链 —— 保持 `(table, key, rel)` 不动。
3. **测试进程必须设 `L_NOTEPAD_ROOT`**：`_family_path()` → `dbmod.default_db_path()` 依赖它，否则 `index_skip.json` 找不到 → `never_index_libs()` 为空 → 单测假失败。
4. **块头/锚点污染不止 FTS**：`search_fts.symptom` 改完是 0 行残留，但**向量里还有 201 块**脏锚点文本 —— 「改规则」和「洗存量」是两件事。

**命令**：

```bat
wuwor l_notepad_server -- python -m l_notepad_server.tools.index_all --refresh-symptoms   :: 改症状匹配规则后（不回网络，秒级）
wuwor l_notepad_server -- python -m l_notepad_server.tools.vec_rebuild --default-scope     :: 向量侧锚点变化 → 增量重嵌
wuwo svc restart l_notepad_api                                                            :: 改了 .py 必须重启
```

**与预期的偏差**：

| 计划预期 | 实际 | 原因 |
|---|---|---|
| 「症状串库」影响面按小样本估 | **56 篇 / 2.3%**，但**分布极毒** | 中招的全是每包同名文件，且症状恰好是 l_notepad 的「搜索卡顿」→ 一次污染 45 个包的入口文档 |
| `refresh_symptoms` 预计改写 56 篇 | 实际 **1280 篇** | 除 56 篇串库，还有大量「本该有症状但 FTS 里是空」的存量行 —— 回写等价于全量重灌后的状态，不是破坏 |
| path bonus 预期「文件名类 Recall@5 +10 点」 | **未验证** | 现语料里按文件名提问的用例太少（golden 15 条里只有 1 条），**这个数字没有证据支撑** |
| `CHANGELOG.md` 召不回 → 加 path bonus 可解 | **解不了** | 查询是中文「变更 历史」，文件名是英文 `CHANGELOG` —— 词面不重叠，bonus 不触发。真正的解是**查询侧同义扩展**或 §6.4 方案 A 的独立 `path` 列 + 中英对照，**未做** |

**最终回归（15 条 golden set）**：

| 配置 | top1 | recall@5 | nDCG@5 | p50 | p95 |
|---|---|---|---|---|---|
| 二期改动前（一期收尾态） | 73.3% | 93.3% | 0.825 | 1437ms | 1757ms |
| **二期后 `rerank=0`（默认）** | **80.0%** | **100.0%** | **0.888** | **1133ms** | 1602ms |
| 二期后 `rerank=1` | 73.3% | 100.0% | 0.881 | 2493ms | 3538ms |

四项全涨：**top1 +6.7 点、recall@5 +6.7 点、nDCG +0.063、p50 −304ms**。

**顺带修掉的收敛 bug（顺藤摸瓜发现）**：追查「症状脏锚点有多少篇」时发现 `vec_rebuild` **永远收敛不了**（每轮把同一批 ~65 篇当「待嵌」重嵌，一路空转到 80 轮上限）。根因是 `vec_docs.anchor` 两侧不同构：写入侧对「没有中文锚点」的文档取 `_anchor_hash("")` = **`da39a3ee5e6b4b0d`**，比对侧取 **`""`** 空串 → 永远不相等 → **61 篇（40 知识库 + 21 笔记）每轮重嵌一次**。修法三处：写入侧 `_anchor_hash(anchor) if anchor else ""`；比对侧同样按「空文本 → 空值」；存量行用 `_norm_anchor()` 把 `sha1("")` 归一成 `""`。实测 **pending 61 → 0**。
> 同一个文件里 788-791 行已经因为同类问题栽过一次 —— **「不存在」的表示法必须只有一种**。

**这次测量本身踩的三个坑（比代码更值得记）**：
1. **`kb_eval` 只打印 `want` 的第一条 → 读表的人会误判**（`热重载 改了源码不生效` 的 `want` 有两条，正确答案在第 2 名，却只打 `sorted(want)[0]`）。已修：打印**全部** `want`，并标注「在第 N 名（多答案用例）」还是「不在前 k，真召不回」。
2. **重排的结论在三次测量里翻了三次**（净负 → 正 → 负 → 正 → 负），根因不是重排本身，是**样本太小**：15 条里翻 1 条 = **±6.7 点**；中途还夹着一个更蠢的原因 —— **本方案文档自己被索引进去，在 golden 查询里当了第 2 名**，把 recall 从 93.3% 顶到 100%。→ **结论：15 条不够做这种判断**；在此之前**默认关重排**的依据是「质量指标不劣 + 快 2.2 倍」，**不是**「重排有害」。
3. **方案类文档必须进 `index_skip.json` 的 `files`**：评测集里的查询词会被方案文档正文逐字命中，它就成了「自指的答案」。已补进名单并摘除存量行。

### 6.11 评测集扩到 40 条后的真实画像（最该看的一节）

§6.10 的 80.0% 是 **15 条**量出来的，而那 15 条里 **12 条属 A 组（Rez-Docs 主文档）** —— 检索最强的区域。扩到 40 条（37 排序 + 3 断言）后：

| 配置 | top1 | recall@5 | nDCG@5 | MRR | p50 | p95 | 未索引提示 |
|---|---|---|---|---|---|---|---|
| **`rerank=0`（默认）** | **75.7%** | **94.6%** | **0.822** | **0.825** | **1176ms** | 1486ms | **3/3** |
| `rerank=1` | 73.0% | 94.6% | **0.839** | 0.815 | 2561ms | 3288ms | 3/3 |
| Δ | **−2.7** | ±0 | **+0.017** | −0.010 | +1385ms | | |

> ⚠️ 本节数字**改过四次**：56.8%（D 组 `want` 抄错）→ 70.3%（修正 `want`）→ 73.0%（B 组 `want` + 导航意图门）→ **75.7%**（路径折扣 + 软放宽）。**一个评测集能把结论带偏 19 点。**

**结论一：真实 top-1 是 75.7%。** 15 条那版的「73.3% → 80.0%」是**采样偏差**（A 组占 80% 权重）；A 组本身提升是真的（10/12），但**它不代表知识库**。

**结论二：重排在 40 条下是「真·权衡」，不是「有害」也不是「无害」**（top1 −5.4、MRR −0.009 对 recall +8.1、nDCG +0.025）。默认仍关 —— 依据是 **agent 先读 top-1**（top1/MRR 更重要）＋ **快 2 倍**。

**结论三：分档强弱差距极大，后续不该再拿总分说事**（`rerank=0`，逐条人工核过）：

| 组 | 用例 | top-1 命中 | 画像 |
|---|---:|---:|---|
| A Rez-Docs 主文档 | 12 | **10（83%）** | 强项。miss 的 2 条都在第 2 名 |
| B 包内 `.md` | 6 | 3（50%） | **弱**：同名的 `rez_pkg/l_tray.md` 主文档总是赢 `l_tray/docs/*.md` |
| C 代码 / 实现文件 | 8 | 5（63%） | 中等；miss 是**同包内**功能近邻（`chat_modes.py` 抢 `permissions.py`） |
| D 口语症状 | 6 | **6（100%）** | **最强**：症状列命中即第 1 |
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
| Rez-Docs 有哪些文档 | `rez_pkg/README.md` | `l_frp`/`pyfory`/`l_thread_safe` 的 `README.md` | 唯一真·跨库同名（README 层） |
| 为什么我改了 search_index.py 之后服务没有生效 | 热重载两篇 | 完全无关的 3 篇 | 长句没召回 |

> ⚠️ **本表推翻了我上一版的结论**。上一版写的是「头号失败模式 = `config.py`/`CHANGELOG.md`/`test_*.py` 跨库同名抢位」，并给了 4 个例子 —— 那些例子**全部来自写错 `want` 的 D 组用例**（正确答案其实都排第 1，我拿「被污染的映射」当了 ground truth）。跨库同名只在 **README/目录页**这一层成立（E 组）。
> **教训：失败分析必须先确认 `want` 是对的。**

→ 下一轮真优先级：**① B 组（包内文档排不上来）② E 组（目录页被 `_nav_prior` 压）③ G 组（长句）**。C 组的同包近邻需要**函数级/符号级**信号（`code_head` 已有符号清单，可做符号命中加权）。

**路径折扣 + 软放宽（75.7% 怎么来的）**：

| # | 改动 | 结果 | 判定 |
|---|---|---|---|
| ① | **路径折扣** `_path_prior` / `_PRIOR_PATH=0.8`：命中词**只出现在路径串里**时打折 | top1 73.0 → **75.7%**、recall 89.2 → **91.9%**、nDCG +0.023、MRR +0.021，**零延迟代价** | **保留** |
| ② | **软放宽** `_RELAX_SOFT_HITS=10`：段间 AND 命中 < 10 也放宽成 OR | recall 91.9 → **94.6%**、nDCG 0.805 → **0.822**、MRR 0.812 → **0.825**，top1 不变 | **保留** |
| ③ | 长句召回（G 组） | **未实施** | 记录待做 |

**① 的收益不是来自它的目标用例** —— 这点必须写清楚：E-1（`Rez-Docs 有哪些文档`）**仍然失败**。折扣确实生效了（那些包 README 的 `prior` 从 1.0 变成 0.8），但 45 条并列候选只降 20% 不够看，真索引页 `rez_pkg/README.md` 仍在 40 名开外（68 条候选里排不上）。**真正升上来的是另一条用例**：`l_notepad 搜索接口 怎么调`（从第 2 名到第 1 名）。
→ 结论：**路径折扣本身站得住（净正、零成本），但「路径 token 无区分度」这个问题它没解决**，只是治了同一病因的另一个症状。E-1 需要的是**给路径来源的匹配一个独立、低权重的通道**（FTS 单独 `path` 列，与 §6.4 方案 A 合并），而不是继续叠乘法系数。

**② 的机制**（已记进 `_RELAX_SOFT_HITS` 注释）：查询里带一个**泛动词**时它会被当成必需段 —— 实测 `权限模式 autopilot 实现` 只召回 **5 条**（> 原阈值 3，所以不放宽），而正确答案 `permissions.py`（含 `MODES = ("default","allow_all","autopilot")`、`effective(ask, "autopilot")` 与三条断言）**因为不字面含「实现」被整个排除在候选之外**。放宽后它进了候选（recall 命中），但 top-1 仍是 `chat_modes.py`（字面含全部三段、coverage=1.0）。→ **`chat_modes.py` 也答对了这个问题**（它决定三档下「直接执行还是问一次」），按多答案原则应一并进 `want`；但那会把 C 组变成「测不出问题」，所以**先留着当已知的近邻歧义**。

**③ 长句（G 组）：能修，但机制不对，所以没动。** 诊断：`为什么我改了 search_index.py 之后服务没有生效` 的目标文档 `src_hot_reload_*` 在**第 10 名**（候选只有 0.45，一堆泛词在拉平排序）。**已验证概念注入有效**：

| 查询 | want 名次 |
|---|---|
| `为什么我改了 search_index.py 之后服务没有生效`（原始） | 10 |
| `改了源码不生效 热重载` | **2** |
| `为什么改了 search_index.py 服务没生效 热重载` | **1** |

但没有可用的机制：词族表契约是「现象词族，**同族词互为同义**」，而 `热重载` 是**原因**不是同义词 —— 塞进 `不生效` 族会让**每个**含「无效/不生效/没用」的查询都注入「热重载」，而这个精度风险**当前 40 条评测集测不出来**（没有对抗性用例）。→ 正确做法是新增一张**症状→主题**映射（或给知识库文档也建症状条目），不是改词族表。**这是「先用评测集能测到的东西说话」的一次自觉放弃。**

#### 6.11.1 更早一轮：导航意图门 + B 组 `want` 修正（73.0% 怎么来的）

1. **导航意图门**（`_nav_intent`，已留）：`_nav_prior` 原来无差别压目录块，于是「问有哪些文档」反而把索引页压掉。现在「块像目录表」和「用户在找目录」分开判（命中就不打折）。边界单测 5/5 触发、5/5 不触发。**在这 40 条上没量出收益**（E-1 仍失败），但**没造成任何回归**，作为「不发荒谬结果」的兜底留着。
2. **两条 B 组 `want` 修正**（+1 top1 / +2 recall）：`l_tray 服务 怎么配` / `托盘 小工具 启动 修复` 的「竞争对手」`rez_pkg/l_tray.md` 经核对**确实答对了问题**（有「服务管理」小节，记了小工具启动与 `load_sibling_module` 约束）→ 一并列进 `want`。
3. **被否决的方案：把「自动生成样板块」纳入降权**（已回退，留档在 `search_index.py` 常量区）。理由看着很硬 —— `wuwo doc_pkg` 给每个包 README 插的生成块**逐字写死** `D:/…/Rez-Docs/<文档>.md` 绝对路径，让 45 个包的 README 变成「Rez-Docs 文档」查询的**等价候选**（实测 5 个包并列 35.5 分、命中块完全相同），把真索引页挤到 40 名开外（共 68 条）。但**实测净负**：`右键点击文件夹时，右键菜单未弹出`（D 组）被踩掉，而目标 E-1 依旧失败 —— top1 73.0% → **70.3%**、nDCG 0.782 → 0.772、MRR 0.791 → 0.778。**已回退。** → 「样板块不该被索引」这个判断**不成立**（它确实回答了「这个包怎么跑」类提问）。E-1 的真问题在别处：**FTS 对 `rez-docs` 这类「路径 token」没有区分度**（68 条里 45 条都能命中它）。

### 6.12 仍未做（二期收尾）

- **E 组：路径 token 没有区分度**（E-1 的真因，路径折扣没解决）—— 下一步是给路径来源的匹配一个**独立低权重 FTS `path` 列**（与 §6.4 方案 A 合并做），不是继续叠乘法系数；也**不要**再试「整块含生成样板就降权」（已实测否决）。
- **C 组：同包近邻仍是「多答案歧义」** —— `权限模式 autopilot 实现` 放宽后 `permissions.py` 进了候选，top-1 仍是 `chat_modes.py`；两者**都答对了问题**。这类用例要么按多答案收进 `want`，要么承认测不出问题 —— **别为了让它『能测出问题』去调权重**。`会话 检查点 checkpoint` 另有真信号：**`checkpoint` 匹配不上 `checkpoints`**（FTS5 无词干/前缀），值得单独给**前缀匹配**做一次 A/B。
- **G 组：长句召回** —— 机制缺位（概念注入有效，但词族表不能塞「原因词」；需要新增**症状→主题**映射）。
- **`_path_bonus` 的效果未验证**：目前只有 1 条用例（E 组 nginx）支撑，扩集后再看。
- **`vec_rebuild` 报的 `pending=1`**：那篇文档的库根已不存在（被 `_pending_docs` 的路径校验跳过），属幻影计数，不影响嵌入。
- `l_agent_chat` 侧消费 `unindexed_packages` / 低 `confidence` 时给模型 hint。
- `tools/sync_repo_docs.py` 未真跑（会往知识库工作区写文件 → depot 上传）。
- §6.4 方案 A（FTS 独立 `path` 列）、评测集扩到 40 条 + 三个新指标、`CHANGELOG` 类「文件名与查询词不同语言」的召回。

### 6.13 头号教训：失败分析前先确认 `want` 是对的

第一版 D 组（口语症状）6 条里有 **5 条 `want` 是错的**，来源是「修复串库后**失去**症状的文档清单」—— 那份清单的映射**本身就是被污染的**（正是二期要修的那个 bug 的产物）。后果：D 组被量成 **1/6 = 17%**（看着像「症状通道坏了」），真实是 **6/6 = 100%**；总分被低估 **13.5 点**（56.8% vs 70.3%）；并据此写了一个**完全错误的结论**：「头号失败模式 = 跨库同名文件抢位」，还准备去做「查询→库软归属」—— 而那个改动**解决的是不存在的问题**。

正确做法（已固化进 `kb_golden.yaml` 文件头注释）：① `want` 只能来自**权威映射**（如 `symptoms.json` 里 `code:<库>:<rel>` 的精确 key），**不能**从「搜索结果」或「某个 bug 的产物清单」里抄；② 加完用例先跑校验：**`want` 路径必须真在索引里**；③ **每条 miss 都要先人工确认「正确答案确实是这个」**，再当失败去归因。
> 这与「重排结论三次反转」（§6.10）是同一类错误的两面：**那次是样本量不够，这次是标注不可靠**。评测集的两条命根子是**独立性**（upstream 别偷看）和**标注正确性**（ground truth 别来自被测系统）。

---

## 7. 订正口径清单（旧值 → 新值）

> 全部按「正文用最新值、旧值保留」处理；接口参考（使用文档）里只写最新值。

| # | 项 | 旧值 / 旧结论 | 新值 / 新结论 | 标注位置 |
|---|---|---|---|---|
| 1 | **DB 体积** | 计划预期「DB 1.29 GB → 0.4 GB」/「1.29 GB → 1.20 GB 瘦身」 | **DB 文件一个字节没小**（删行只挂 freelist，192,710 页 ≈ 0.74 GiB）；`VACUUM` 后才约 0.47 GiB。「1.29 GB → 1.20 GB」是 GB/GiB 换算错误 | §5.1 / §5.5 / §5.10 / §5.12 |
| 2 | **查询硬预算** | 「最坏 1.5s 返回 `degraded`」（`1500ms`） | 实际 **3000 ms**（`L_NOTEPAD_QUERY_BUDGET_MS`）；且**不是墙钟上限**（`embed_texts` 已进入就中断不了，实测单查 4.6s） | §5.1 / §5.7 / §5.11 |
| 3 | **重排收益定性** | 「净负收益 / 有害」（一期 P0(f) 原措辞） | **不劣且快（快 2.2 倍）**，40 条口径是**真·权衡**（top1 −5.4 / MRR −0.009 对 recall +8.1 / nDCG +0.025）。**默认关的依据是「不劣且快」，不是「有害」** | §5.10 P0(f) / §5.11 / §6.10 / §6.11 |
| 4 | **15 条 golden top-1** | 「73.3% → 80.0%」 | **采样偏差**（12/15 是 Rez-Docs 主文档）；40 条真实 top-1 = **75.7%** | §5.10 / §6.10 / §6.11 |
| 5 | **症状 A/B 数字** | `0.511 / 0.538`（症状开/关） | **开卷上限**（看过目标文件后手写症状）→ 作废；随后发现「`search()` 不刷新」→ 连 `0.261 / 0.234`（盲生成）也是在**陈旧列**上量的 → **两组全部作废** | §4.1 / §4.3 |
| 6 | **auto 阈值扫描假提升** | 「top1 78.4% / nDCG 0.838，是同期语料变化所致」 | 手写脚本**漏传 `sources=note,kb`** → 该段归因**错误已删**；权威口径 **top1/recall 完全不变**，只有 p50 1397ms → 230ms | §6.9 |
| 7 | **头号失败模式** | 「跨库同名 `config.py`/`CHANGELOG.md`/`test_*.py` 抢位」 | **推翻**：那些例子来自**写错 `want`** 的 D 组用例（正确答案其实排第 1）；跨库同名只在 **README/目录页**层成立 | §6.11 |
| 8 | **40 条 top-1 数字** | 56.8% → 70.3% → 73.0% | **75.7%**（路径折扣 + 软放宽）；「一个评测集能把结论带偏 19 点」 | §6.11 |
| 9 | **recall@5 缺口性质** | 「都排在第 2~3 名」（非真召回失败） | **有 1 条是真召不回**（`CHANGELOG.md`） | §5.10 落地表 / §5.12 |
| 10 | **P0 查询耗时预期** | 「1.2s → 0.35s」 | 重排关 **p50 1437 / p95 1757 ms**；0.35s 只在「纯词法 + 不回退」时成立，而回退率 ~100% | §5.1 / §5.3 / §5.11 |
| 11 | **Jev 蒸馏同义词** | 预期有召回增益 | 工具链跑通但**无增益**（A/B 6→6 / 1→1 / 8→8），**被症状索引方案取代**，产物留档未启用 | §3 / 使用文档 §5.3 |
| 12 | **`_symptom_pick` 签名** | 计划「调用处显式传 `source`/`kb_name`」 | 实际**不改签名**（作用域从 key `code:<库>:<rel>` 推出） | §6.10 踩坑 2 |
| 13 | **`refresh_symptoms` 改写量** | 预期 56 篇 | 实际 **1280 篇**（含「本该有症状但为空」的存量行） | §6.10 |
| 14 | **path bonus 收益** | 「文件名类 Recall@5 +10 点」 | **未验证**（golden 15 条仅 1 条文件名用例）；`CHANGELOG.md` 召不回**解不了**（中文查询 vs 英文文件名无字面重叠） | §6.4 / §6.10 |
| 15 | **预热成本结构** | 假设大头是「1.7 万文件 stat」 | 大头是 **depot 归档目录列举（网络，8s）**；P2(b) 对预热**没有影响** | §5.10 计划外发现 4 |
| 16 | **段间 AND 长句** | 长句 `total=2`（目标文件不在候选） | 段数 >4 → **全部 OR**，`total=2 → 87` | §3.3 |
| 17 | **`auto` 回退率** | 预期 10~20% | 一期实测 **~100%**（判据 40/1.3）；二期标定 **25/1.0** 后降到 **11%** | §5.6 / §5.11 / §6.9 |
| 18 | **`route` 评分公式** | 初版 `0.5*log1p + 2.5*meta + 1.5*vec`（无 bm25 项） | 加 `+1.5*tanh(-bm25/20)`（`_ROUTE_W_BM25`，自带 IDF） | §2.1 第二轮优化 |
| 19 | **`route` 的 `sources` 默认** | `kb` | **`kb,code`**（2026-09-25 起） | 使用文档 §1.3 |
| 20 | **`/api/search` 的 `mode` 默认** | `hybrid` | **`auto`**（2026-09-28 起，`routers/search.py` + `routers/kb.py`） | 使用文档 §1.1 / §1.4 / §4 |
| 21 | **`auto` 语义** | 「只有当一条都没命中时才回退」 | 现为**分数判据**（top-1 ≥ 25 且 top1/top2 ≥ 1.0 才算「够用」） | 使用文档 §1.1 / §1.4 / §4 |
| 22 | **索引源（kb）** | 文档曾称「各知识库 workspace」 | **depot 已上传归档**（`_kb_sync_one`），本机工作区不参与归档索引 | §2.1 / 使用文档 §6 |
| 23 | **症状"手写天花板 0.833"** | 「手写天花板」 | 是**我（强模型）开卷写的**，不是人类水平；真实人类盲写基准未测 | §4.4 相关 |
| 24 | **`l_agent_chat` 的 `notepad_search` hint** | 计划做低置信/degraded hint | **未做**（另一包） | §5.10 / §5.12 |

---

## 8. 未做 / 待办汇总

**一期未做**：`VACUUM`；包 README 症状注入按库过滤（**二期已修**）；`l_nginx/999.0/README.md` 块头去重；`CHANGELOG.md` 标题/文件名通道；硬预算真封顶（每档独立 socket 超时）；`l_agent_chat` 侧超时与 hint；`sync_repo_docs.py` 真跑；评测集扩集。

**二期未做 / 待办**：
- FTS `path` 独立列（方案 A）—— B/E/G 组的根治手段。
- 查询侧同义扩展解决中英文件名（`变更/历史 → changelog`）。
- 长句/因果类召回（G 组）。
- 符号级信号（C 组同包近邻）。
- 评测集 holdout 固定与持续化（§6.7）。
- 向量层校准（`isotonic/logistic`，需 50+ 样本）。

**功能层仍保留的限制**：知识库工作区索引手动；`.py` 不进知识库归档；IDF 全局统计（`docs < 20` 不启用）；代码块占内存缓存额度；`route` 默认含本机库；重排是独立进程（没起就降级）。
