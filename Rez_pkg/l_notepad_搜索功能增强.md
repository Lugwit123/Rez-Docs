# l_notepad 搜索功能增强 — 进度与待办

> 2026-09-25 ｜ 关联文档：`l_notepad_搜索接口使用文档.md`、`l_notepad_server_快速选库路由接口_计划.md`
> 本文是**总进度 + todolist**；接口细节以《搜索接口使用文档》为准。

## 1. 第二轮增强（2026-09-25 本次）

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
| 9 | **文档** | — | 《搜索接口使用文档》§1.3/§1.4/§1.5/§2/§6/§9/§10；`CHANGELOG v3.4.0` | ✅ |
| 10 | **「要搜索哪些包」勾选** | `GET /api/search/code_packages`、`/api/search?packages=`、搜索页勾选面板 | 货架（`L_NOTEPAD_PKG_ROOT` = `<trayapp>/rez-package-source`）下 54 个 rez 包，`kind=pkg` 手动建索引；勾选只过滤 `source=code`（笔记/知识库不受影响）；默认不勾选 = 不限；选择存 `localStorage['ln_search_packages']` | ✅ |
| 11 | **「判断依据」对话框** | 每条命中的 `explain` 字段 + 卡片上的「判断依据」按钮（`app.js` / `app.css`） | 分项表（权重×取值=得分：短语/覆盖率/词频/近邻/bm25/语义）、逐词块（IDF 权重 × 出现次数）、泛词（权重 0）、未命中词块、名次与排序主序、**与下一条的分差 + 主因**、一句话结论；纯前端渲染（顶栏弹窗/搜索页/试搜共用） | ✅ |

### 改动文件（本轮）
`search_index.py`、`search_vec.py`、`routers/search.py`、`routers/web.py`、
`templates/{web_search,web_code,web_index,base}.html`、`static/{app.js,app.css}`、
`Rez-Docs/Rez_pkg/l_notepad_搜索接口使用文档.md`、`doc/CHANGELOG.md`。

## 2. 第一轮增强（2026-09-24）

| # | 功能 | 入口 / 接口 | 关键实现 | 状态 |
|---|------|------------|----------|------|
| 1 | **快速选库路由** | `GET /api/search/route?q=&depth=&budget_ms=` | `search_index.route()`：depth 0 元数据 / 1 词法按库聚合 / 2 库摘要向量语义 / 3 交回 Agent；`reason_code`、结果缓存、bm25 并入打分、`budget_ms<100` 自动降档 | ✅ |
| 2 | **`mode=auto`** | `/api/search?mode=auto` | 先词法，零命中回退 hybrid；返回 `mode_used` | ✅ |
| 3 | **独立全局搜索页** | `GET /web/search` | 一次搜「笔记 + 全部知识库 + 代码库」；参数 `mode/sources/kb/rerank/limit/offset`；知识库分面、搜索帮助面板、搜索历史（localStorage）、结果卡片复用 `LN.renderSearchHits` | ✅ |
| 4 | **前端接线** | `base.html` / `web_index.html` | 顶栏表单 → `/web/search`；左上角 ☰ 菜单加「全局搜索」；搜索历史 datalist（顶栏 + 搜索页共用）；修复 `rerank=` 空串 `int_parsing` 报错 | ✅ |
| 5 | **代码库索引** | `source=code`：`GET/PUT /api/search/code_roots`、`GET /api/search/code/file` | 本机目录纳入检索；`CODE_EXTS`/`CODE_SKIP_DIRS`/`_scan_code()`；`_PERM_SQL` 放行 code；状态页「代码库索引」卡片；搜索页来源「仅代码库」；只读查看 | ✅ |
| 6 | **文档** | — | 《搜索接口使用文档》§1.1–§1.5 / §6 / §11 增补；`CHANGELOG v3.3.0`；计划文档 §13–§17 | ✅ |

### 第一轮实测（2026-09-24，保留作对照）

| 场景 | 结果 |
|------|------|
| 长句 `mode=auto` | `mode_used=hybrid`，18 命中 |
| 短句 `mode=auto` | `mode_used=lex`，2.4ms |
| `route` depth 0/1/2/3 | 0.4ms / 2–8ms / 热态 120–190ms / 空 + `delegate` |
| 自然语言原句→代码库 | `lex total=5`，目标文件 **#3**，`vec=0` |
| 控制实验（词含"卡顿/主线程/钩子"） | `total=1`，仅目标文件，覆盖 100% |

**当时的结论**：「词法能定位、但不理解」——原句排序由 repo 泛词驱动，「卡」被停用字过滤丢弃，代码库无语义。
本轮（2026-09-25）的三处修改正对这三条：**单字/同义保留**、**IDF 泛词抑制**、**代码语义**。

## 3. 实测记录（本机，2026-09-25）

| 场景 | 结果 |
|------|------|
| `route_terms("卡很久 主线程 钩子")` | `['卡很','卡','卡顿','很久','主线','线程','钩子']` —— **「卡」与同义词不再被丢** |
| 同句 `route depth=1` | 4.9ms；`kbs` 含 `rez_pkg`(2.97) 与 `l_notepad_client`(2.42，`kind=code`) |
| **原句 → 代码库**（`程序卡很久，主线程被什么钩子阻塞了`） | `lex total=53`，目标文件 `folder_favorites_hotkey.py` **#2**；`hybrid` 同位置且带 **`vec=0.6316`**（P0-2 达标：口语症状能语义召回代码） |
| `route depth=0 sources=code` | 命中 `l_notepad_client`（`meta_hits=2`，`score=5.0`） |
| IDF 泛词 | `notepad/client/窗口 → 0.0`（`程序/搜索` 保留 2.4）；`route` 结果 `terms_generic_dropped` 回填 |
| 手动建索引 | `POST /api/search/index_lib {"label":"l_notepad_client"}` → `files=44, capped=false, duration_ms=16`（增量无变化） |
| 体量上限（压到 5 文件复测） | `files=5, capped=true`，且**已有 44 行索引没被删**；恢复上限后 `files=44, capped=false` |
| 代码嵌入速度 | 40 个代码文档 23.5s（≈0.6s/文档，含 44 文件 / 1.1MB 的面包屑库） |
| **rerank 上线**（本机 llama.cpp + bge-reranker-v2-m3 Q8，`D:\Tools\llama.cpp\start_rerank.bat`） | 同一句原话 `mode=hybrid`：目标文件 `folder_favorites_hotkey.py` **rr=+1.004 排第 1**（开 rerank 前是第 3，且 kb 文档以 142 分霸榜）；整轮 **987ms**（rerank 722ms） |
| rerank 调参前后 | 候选不截断 5×900 字符 = **1.55s** → 截到 300 字符 = **0.6s**；llama-server 默认 `-ub 512` 装不下一个候选，必须 `-ub 2048` |
| 纯症状句（去掉 `中键/托盘` 等标识符） | rerank 也救不回来（`clipboard_store.py` 第 1，目标文件第 7）——**语义分（0.58–0.65）本身没区分度**，只有整句含标识符时词法候选才对 |
| **UI 默认路径暴露的两个问题**（2026-09-25 晚，用户截图） | ① 长句 `lex` 段间 AND 只剩 **2 条**命中（目标文件不在候选）；② `auto` 只在零命中才回退 hybrid，长句就停在这 2 条上 |
| 修法 + 复测 | 段数 >4 → 全部 OR（`2 → 87` 条候选）；`auto` 在「命中 <3 或长句」时走 hybrid → **`mode_used=hybrid`、目标文件第 1（rr=0.74）**，整轮 ≈1.5s |

### 结论
- 「口语 → 代码」这条链**打通**：关键单字/同义词保住了召回，IDF 抑制住泛词，语义分把目标文件稳定在前三且分数可解释。
- 仍需人工选库/建索引（**有意为之**：不进 depot、不自动上传）。

## 4. 已知限制

1. **知识库工作区索引是手动的**：改了工作区文件要再点一次「创建索引」（TTL 自动刷新只覆盖代码库根）。
2. **`.py` 不进知识库归档**：`WORKSPACE_EXTS`（也是 depot 自动上传白名单）仍是文档类型；要让知识库归档含代码，需另议（体积/配额）。
3. **IDF 是全局统计**：`docs < 20` 时不启用；库很小 + 词很常见时可能误判泛词。
4. **代码块占内存缓存额度**（`L_NOTEPAD_VEC_CODE_CHUNKS=8000`）：代码量极大时会把部分笔记块挤出缓存（该文档本轮无语义分）。
5. **代码根不要指整棵树**：`rez-package-source` 下实测 43 万+ `.py`，超 `CODE_MAX_*` 会截断（截断会少索引，不会误删）。
6. **`route` 默认含本机库**：Agent 只要知识库需显式 `sources=kb`。
7. **重排是独立进程**：`D:\Tools\llama.cpp\start_rerank.bat`（llama-server，11435）没起就自动降级回融合排序；
   改了 `package.py` 的 rerank env 必须**真重启**服务进程——2026-09-25 实测“热重启”只记了一次事件、进程没换（env 仍是旧的 12/400/3）。
8. **纯症状句仍搜不准**：语义分在同 repo 文件间无区分度（0.58–0.65），rerank 只能重排「已经召回的候选」——识别符（`中键`/`托盘`）在候选里才有救。

## 5. TODO（剩余）

| 优先级 | 事项 | 说明 / 验收 |
|--------|------|-------------|
| P2 | **rerank 常驻** | `D:\Tools\llama.cpp\start_rerank.bat` 目前要手开窗口；做成开机自启或托盘托管，否则重启电脑后静默降级（检索仍正常，只是没重排） |
| P2 | 纯症状句召回 | 语义分无区分度 → 试「文件级摘要向量」或查询改写（把口语扩成标识符），目标：无标识符也能进前 3 |
| P2 | **评测集** | 建一批「需求 → 应命中文件」的标注，量化 recall@k，用于继续调权重（IDF 阈值 / 同义词表 / 权重） |
| P2 | 代码命中体验再进一步 | 结果页直接显示命中行号（当前查看页高亮但需自己找行）；文件树定位 |
| P3 | 工作区索引自动化 | 工作区改动后**提示**「该库索引已过期」（`workspace_sync` 已有 (size,mtime) 基线，可复用），仍由用户点按钮重建 |
| P3 | 知识库归档含代码 | 评估 depot 体积/配额后再决定是否放开 `WORKSPACE_EXTS` |
| P2 | 代码树剪枝规则 | 把 `L_NOTEPAD_CODE_SKIP_EXTRA` 之类做成可配置，便于给特定仓库加跳过目录 |

## 6. 下一步建议
先做 P2「评测集」：有了 recall@k，「IDF 阈值 30%」「同义词表」「`_W_VEC` 权重」这些现在靠手感调的参数才有依据；
其次把「工作区索引过期提示」做出来（工作量小，直接复用 `workspace_sync` 已有基线），减少「手动建索引」的漏点。
