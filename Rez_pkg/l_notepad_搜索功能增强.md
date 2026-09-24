# l_notepad 搜索功能增强 — 进度与待办

> 2026-09-24 ｜ 关联文档：`l_notepad_搜索接口使用文档.md`、`l_notepad_server_快速选库路由接口_计划.md`
> 本文是**总进度 + todolist**；接口细节以《搜索接口使用文档》为准。

## 1. 已完成（本次增强）

| # | 功能 | 入口 / 接口 | 关键实现 | 状态 |
|---|------|------------|----------|------|
| 1 | **快速选库路由** | `GET /api/search/route?q=&depth=&budget_ms=` | `search_index.route()`：depth 0 元数据 / 1 词法按库聚合 / 2 库摘要向量语义 / 3 交回 Agent；`reason_code`、结果缓存、bm25 并入打分、`budget_ms<100` 自动降档 | ✅ |
| 2 | **`mode=auto`** | `/api/search?mode=auto` | 先词法，零命中回退 hybrid；返回 `mode_used` | ✅ |
| 3 | **独立全局搜索页** | `GET /web/search` | 一次搜「笔记 + 全部知识库 + 代码库」；参数 `mode/sources/kb/rerank/limit/offset`；知识库分面、搜索帮助面板、搜索历史（localStorage）、结果卡片复用 `LN.renderSearchHits` | ✅ |
| 4 | **前端接线** | `base.html` / `web_index.html` | 顶栏表单 → `/web/search`；左上角 ☰ 菜单加「全局搜索」；搜索历史 datalist（顶栏 + 搜索页共用）；修复 `rerank=` 空串 `int_parsing` 报错 | ✅ |
| 5 | **代码库索引** | `source=code`：`GET/PUT /api/search/code_roots`、`GET /api/search/code/file` | 本机目录纳入检索；`CODE_EXTS`/`CODE_SKIP_DIRS`/`_scan_code()`；`_PERM_SQL` 放行 code；状态页「代码库索引」卡片；搜索页来源「仅代码库」；只读查看 | ✅ |
| 6 | **文档** | — | 《搜索接口使用文档》§1.1–§1.5 / §6 / §11 增补；`CHANGELOG v3.3.0`；计划文档 §13–§17 | ✅ |

### 改动文件
`search_index.py`、`search_vec.py`、`routers/search.py`、`routers/web.py`、
`templates/{base,web_search,web_index,web_kb}.html`、
`Rez-Docs/Rez_pkg/{l_notepad_搜索接口使用文档.md, l_notepad_server_快速选库路由接口_计划.md}`、
`doc/CHANGELOG.md`。

## 2. 实测记录（本机）

| 场景 | 结果 |
|------|------|
| 长句 `mode=auto` | `mode_used=hybrid`，18 命中（顶层含 `Rez_pkg/l_homepage.md`） |
| 短句 `mode=auto` | `mode_used=lex`，2.4ms（有词法命中不走语义） |
| `route` depth 0/1/2/3 | 0.4ms / 2–8ms / 热态 120–190ms（冷启动首次 ~6s）/ 空 + `delegate` |
| `route` `budget_ms=50`+depth2 | 降为 `depth_used=1`、`reason_code=budget_downgrade` |
| **代码库索引**（配置 `l_notepad_client`） | 索引 44 文件；`stats.sources` 出现 `code l_notepad_client docs=44` |
| 自然语言原句→代码库 | `lex` `total=5`，**`folder_favorites_hotkey.py` 排 #3**，24ms；`vec=0` |
| 标识符 `folder_favorites_hotkey` | 4 文件命中（含目标文件） |
| **控制实验**：词含"卡顿/主线程/钩子" | `total=1`，**仅** `folder_favorites_hotkey.py`，覆盖 100% |

### 结论（结果是否有意义）
- **词法能"定位"**：目标文件进前三；#1/#2（`local_main.py`/`folder_favorites_widget.py`）也在热键路径上。
- **但不"理解"**：原句排序由 repo 泛词（`notepad/client/ctrl/窗口/程序`）驱动；**关键词"卡"被停用字过滤丢弃**（"卡很久"→`卡很/很久`含 `很`），噪声项（`settings_widget.py`）仍出现。
- **只有用文档自己的词**（`卡顿/主线程/钩子`）才是**唯一且 100%** 命中 → "知道词才搜得到"。
- 根因：代码库**未嵌入**（无语义），且泛词未按 IDF 抑制。

## 3. 已知限制

1. **代码库不参与向量语义**（命中项 `vec=0`）——"卡很久/卡顿"这类语义无法召回代码。
2. **知识库/归档索引白名单只有文档扩展名**（`.md/.markdown/.txt/.rst/.log`）→ 把 rez 包当知识库上传，**`.py` 不进索引**。
3. **关键词抽取丢弃单字**（`route_terms`/停用字）→ "卡""慢"等关键单字丢失。
4. **代码根不要指整棵树**：`rez-package-source` 下实测有 **43 万+ `.py`**（含三方/缓存/vendored），整树索引会极大拖慢并污染结果；应指向**单个包目录**，并依赖 `CODE_SKIP_DIRS` 剪枝。

## 4. TODO（待办）

| 优先级 | 事项 | 说明 / 验收 |
|--------|------|-------------|
| **P0** | **把 rez 包代码纳入知识库索引** | 放开知识库扩展名：`search_index.WORKSPACE_EXTS` 加入 `CODE_EXTS`；`routers/kb.py::_WORKSPACE_EXTS` 改为**单一来源**（`= search_index.WORKSPACE_EXTS`，影响上传校验 `:126/:551`、浏览预览）；`workspace_sync` 自动跟随。**待定**：全局放开 vs 加开关（`L_NOTEPAD_KB_CODE`）；需评估 depot 上传体积/配额 |
| **P0** | **代码库支持向量语义** | 把 `source=code` 纳入嵌入流程（`search_vec` 的来源/分块），使"卡很久/卡顿"能语义召回代码；验收：原句 + `vec>0` 命中目标文件 |
| P1 | 关键词抽取保留关键单字 + 同义词 | 单字"卡/慢/死"保留或加同义（卡↔卡顿/卡死）；避免 `很/都` 把整词带没 |
| P1 | 泛词抑制（IDF） | `notepad/client/窗口/程序` 等 repo 泛词降权/过滤 |
| P1 | 代码库体量治理 | 单库文件数/字节上限 + 进度显示 + 更严 skip（`site-packages/.venv` 等）；避免整树索引 |
| P2 | `route` 支持 `sources=code` | 当前 route 默认 `sources=kb`，代码库不出现在选库结果里 |
| P2 | 代码命中体验 | 卡片区分图标/徽章；查看页加行号/高亮；文件树定位 |
| P2 | 评测集 | 建一批"需求→应命中文件"的标注，量化 recall@k，用于调权重 |
| P3 | 文档补写 | 代码库索引使用说明并入 Rez-Docs；索引体积/耗时观测 |

## 5. 下一步建议
先做 **P0-1（知识库放开代码扩展名）** 与 **P0-2（代码语义）**：前者打通"网页上传 rez 包 → 连代码一起可搜"，后者让"口语症状"也能命中。两项都改完，再用本次那条原句回归，目标是把 `folder_favorites_hotkey.py` 稳定送到 **#1** 且带语义分。