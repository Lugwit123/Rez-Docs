# l_notepad 搜索接口使用文档

> 适用版本：`l_notepad_server` 999.0（2026-09-17 更新，含倒排索引 + 向量语义检索 + 知识库归档映射）
> 代码位置：`l_notepad_server/src/l_notepad_server/{search_index.py, search_vec.py, routers/search.py, routers/kb.py, depot_map.py}`

## 0. 快速开始

```bash
# 开发机（nginx 8080）
BASE=http://127.0.0.1:8080/note
# 部署机
# BASE=https://121.196.144.88/note

# 登录态：cookie（浏览器会话）或 Authorization: Bearer <l_notepad_token>
TOKEN=<从浏览器 DevTools 复制 l_notepad_token，或走 /api/auth/login 拿>

# ★ 本机直连免 token（仅 GET/HEAD，且必须直连 8765 不经 nginx）——见 §7.1
curl -s "http://127.0.0.1:8765/api/search?q=%E5%88%9B%E5%BB%BA%E5%8C%85&limit=3" | jq

# 全局搜索（笔记 + 知识库，默认混合模式）
curl -s -H "Authorization: Bearer $TOKEN" \
  "$BASE/api/search?q=%E5%88%9B%E5%BB%BA%E5%8C%85&limit=5" | jq

# 只在某个知识库里搜
curl -s -H "Authorization: Bearer $TOKEN" \
  "$BASE/api/kb/rez_pkg/search?q=%E5%A6%82%E4%BD%95%E6%96%B0%E5%BB%BA%E4%B8%80%E4%B8%AA%20Rez%20%E5%8C%85" | jq

# 索引状态（deep=1 做磁盘校对 + FTS 完整性检查）
curl -s -H "Authorization: Bearer $TOKEN" "$BASE/api/search/stats?deep=1" | jq

# 重建索引 / 重嵌向量（管理员）
curl -s -X POST -H "Authorization: Bearer $TOKEN" "$BASE/api/search/reindex_async"
curl -s -X POST -H "Authorization: Bearer $TOKEN" "$BASE/api/search/embed_async?force=1"
```

交互式文档（public，无需登录）：`$BASE/docs`（Swagger UI）、`$BASE/redoc`、`$BASE/openapi.json`。
可视化入口：浏览器打开 `$BASE/web/index`（导航「🔎 搜索索引」）—— 试搜框 + 索引状态 + 模型切换。

### 0.1 最小可用四条（本机直连 8765，免 token，实测 2026-09-17）

给 agent / 脚本用的最短路径：**搜索拿路径 → 再读正文**。搜索结果只有 180 字 `snippet`，正文必须另外调 `workspace/file`。

```bash
B=http://127.0.0.1:8765

# 1) 全局搜索（笔记 + 全部知识库）
curl -s "$B/api/search?q=%E5%88%9B%E5%BB%BA%E5%8C%85&limit=5"

# 2) 单个知识库内搜索（q 要 URL 编码；空格用 %20）
curl -s "$B/api/kb/rez_pkg/search?q=l_qt_wgt_lib&limit=10"

# 3) 列这个知识库的全部文件（拿 rel 用于下一步）
curl -s "$B/api/kb/rez_pkg/workspace"

# 4) 读正文 —— 参数名是 path，值 = 上面拿到的 rel
curl -s "$B/api/kb/rez_pkg/workspace/file?path=Rez_pkg/l_script_editor.md"
```

搜索响应里够用的就 4 个字段：

| 字段 | 用途 |
|---|---|
| `total` | 命中数；`0` 说明关键词切分对不上，换词而不是翻页 |
| `hits[].rel` | 直接喂给 `workspace/file?path=` 读正文 |
| `hits[].snippet` / `matches` | 快速判断这篇要不要读全文 |
| `hits[].score` | `< 1.0` 基本是向量噪声（只有 `vec` 分、`coverage=0`），别当命中 |

Windows `cmd` 里没有 `$B`，直接写全 URL；带 `&` 的 URL 必须整体加双引号，否则 `&` 被 cmd 当命令分隔符。

踩过的坑（省得再试一遍）：

| 试法 | 结果 |
|---|---|
| `GET /api/kb`、`/api/kb/{kb}/stats` | **405** —— 没有「列知识库」的 GET；知识库名要么已知，要么从 `/api/search` 结果的 `kb_name` 里看 |
| `GET /api/kbs` | 404 |
| `workspace/file?rel=...` | **422 `Field required: path`** —— 这个端点用 `path`，只有 `depot/*` 系列用 `rel` |
| `GET /api/kb/{kb}/depot/file?rel=...` | 本机实测 **500**（depot 版本库未连通时如此）；只要读当前文本，走 `workspace/file` 就够 |
| `raw` / `content` / `read` / `doc` 等猜出来的端点名 | 全部不存在，别猜，`$BASE/openapi.json` 里有全表 |

---

## 1. 接口总览

### 1.1 搜索（`routers/search.py`，前缀 `/api/search`）

| 方法 | 路径 | 权限 | 说明 |
|------|------|------|------|
| GET | `/api/search` | 登录 | 检索用户可见的**笔记 + 知识库归档**文档 |
| GET | `/api/search/route` | 登录 | **快速选库**：一段需求 → 相关知识库排序（毫秒级，见 §1.3） |
| GET | `/api/search/stats` | 登录 | 索引状态（文档数/分源明细/待处理队列/重建与嵌入进度/向量模型） |
| POST | `/api/search/reindex` | 管理员 | **同步**清空并重建全部索引，返回写入行数 |
| POST | `/api/search/reindex_async` | 管理员 | **后台**重建，立即返回；进度见 `stats.reindex` |
| GET | `/api/search/models` | 登录 | embedding 模型目录（是否已安装）+ 当前模型 + 下载进度 |
| POST | `/api/search/model` | 管理员 | 切换模型；**未安装只回 `need_download=true`，不下载** |
| POST | `/api/search/model/download` | 管理员 | 显式下载模型（Ollama `/api/pull` 流式，进度见 `models.download`） |
| POST | `/api/search/embed_async` | 管理员 | 后台增量嵌入（`?force=1` 全量重嵌）；进度见 `stats.vec.embed` |

`GET /api/search` 参数：

| 参数 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `q` | str | `""` | 查询串（见 §3 语法） |
| `limit` | int | 20 | 返回条数，上限 **500** |
| `offset` | int | 0 | 偏移（分页） |
| `sources` | str | 全部 | 逗号分隔来源过滤：`note`（个人笔记）/ `kb`（知识库工作区）/ `code`（本机代码库，见 §1.5） |
| `mode` | str | `hybrid` | `lex` 纯词法 / `hybrid` 词法+语义 / `sem` 纯语义 / `auto` 先 `lex`、零命中回退 `hybrid`（返回多一个 `mode_used`） |

### 1.2 知识库（`routers/kb.py`）

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/kb/{kb}/search?q=&mode=&limit=&offset=` | **单知识库范围检索**（作用域下沉到 SQL：`search_docs.kb_name = ?`） |
| GET | `/api/kb/{kb}/depot` | 归档映射（`library` / `subpath` / `base_path` / `ws_name` / `ws_id` / `local_root` / `remote_url`） |
| PUT | `/api/kb/{kb}/depot` | 改映射（body `{"library"?, "subpath"?, "ws_name"?}`；`null`=不改，`""`=回默认），改完自动重建工作区 maps |
| GET | `/api/kb/{kb}/depot/list?rel=` | 列归档目录（返回项带 `rel`，相对 `base_path`） |
| GET | `/api/kb/{kb}/depot/file?rel=&rev=0` | 读归档文件内容（`rev=0` = 最新） |
| POST | `/api/kb/{kb}/depot/submit?rel=&description=` | 把 body 原始字节提交为新版本 |
| GET/PUT | `/api/kb/{kb}/workspace[/file?path=]` | 工作区本地目录（列表带 `rel/name/size/mtime`）与单文件读写；**参数名 `path`，不是 `rel`**（服务端模式；托盘模式见《知识库本机模式》） |
| GET | `/api/kb/{kb}/workspace/reveal` | 在**服务器**资源管理器打开目录 |

### 1.3 快速选库（`GET /api/search/route`，2026-09-24 新增）

输入一段**复杂需求**，返回**知识库级**排序，供 Agent 决定读哪几个库。与 `/api/search` 的区别：
搜索接口只做**文档级**召回（`kb_name` 只是命中项的附属字段），不输出库级判断；route 专门做库级聚合。

```bash
curl -s "http://127.0.0.1:8765/api/search/route?q=%E4%B8%80%E6%AE%B5%E5%A4%8D%E6%9D%82%E9%9C%80%E6%B1%82&depth=1"
```

| 参数 | 默认 | 说明 |
|------|------|------|
| `q` | `""` | 需求原文（可长；内部做关键词抽取，**不走段间 AND**） |
| `depth` | `1` | `0` 仅库元数据匹配 / `1` 元数据 + 词法按库聚合 / `2` 语义加分（需求嵌一次与**各库摘要向量**比对；语义不可用自动降为 `1`）/ `3` 不处理（分解多查询请调用方自行完成） |
| `budget_ms` | `300` | 预算；`>0 且 <100` 时 `depth>=2` 自动退回 `1`（`reason_code=budget_downgrade`），并回填 `over_budget` |
| `limit` | `10` | 返回库数上限（≤50） |
| `sources` | `kb,code` | 逗号分隔来源过滤（`note` / `kb` / `code`）。`code` = 本机库（代码库根 + 知识库工作区，见 §1.5）；只要知识库请显式传 `kb` |

关键词处理（2026-09-25 起）：
1. **同义扩展**：`_SYNONYMS` 把口语症状映射到代码/文档用词（`卡 ↔ 卡顿/卡死/阻塞/无响应`、`慢 ↔ 缓慢/性能/耗时`、`死/崩 ↔ 崩溃/闪退`），命中其一即一并召回；
2. **关键单字保留**：`卡/慢/死` 这类单字保留（FTS 前缀匹配 `"卡" *`），低信息单字（`的/了/很/都`）仍丢弃；bigram 只在**整块都是低信息字**时才丢（「卡很」保留、「需要」丢弃）；
3. **IDF 泛词剔除**：`term_idf()` 按词块文档频率算权重，占比 ≥ `_GENERIC_DF_RATIO`(30%) 的 repo 泛词（`notepad/client/窗口/程序`）不参与路由与打分，被剔除的词见 `terms_generic_dropped`（元数据匹配仍用剔除前的词表）。

返回（`kbs` 按 `score` 降序）：

```json
{"query":"...","depth_req":1,"depth_used":1,"degraded":false,"reason":"","reason_code":"ok",
 "took_ms":4.4,"cached":false,"terms":["选库","路由"],"terms_generic_dropped":["程序"],
 "kbs":[{"kb_name":"rez_pkg","score":8.23,"doc_hits":33,"best_bm25":-45.44,"meta_hits":2,"vec":0.0}]}
```

- `score = 0.5*log1p(doc_hits) + 2.5*meta_hits + 1.5*vec + 1.5*tanh(-bm25/20)`（权重见 `search_index._ROUTE_W_*`；bm25 项自带 IDF，压低"接口"这类全库高频泛词）。
- `doc_hits` = 该库词法命中文档数；`meta_hits` = 库名/标题/描述命中的关键词块数；`best_bm25` = 库内最优 FTS 原始分（负值越负越相关）。
- `reason_code`：`ok` / `delegate`（depth≥3）/ `budget_downgrade`（预算不足跳过语义）/ `semantic_degraded`（语义不可用）。
- `cached`：同一 `(q, depth, budget_ms, limit, sources, user)` 结果缓存 10s（`search_index._ROUTE_CACHE_TTL`）。
- **不触发慢路径**：depth 0/1 只查索引表与元数据表，不访问网络；depth 2 也只做**一次** embed（先 0.3s TCP 探测，不可达立即降级），不触发 `hybrid` 无 ollama 时约 6s 的兜底。depth2 冷启动会构建一次库摘要向量（元数据 + 文档质心，存 `kb_vec` 表）后复用。
- **耗时（本机实测 2026-09-24，42 文档 / 701 块）**：depth0 ~0.4ms、depth1 ~2–8ms、depth2 **热态 ~150–400ms**（embedding 模型冷启动首次可能数秒）；命中缓存 ~0ms。
- `degraded` / `reason`：请求档位与实际档位不一致时给出原因；`reason_code` 为可编程的机器码。

### 1.4 网页搜索页（`GET /web/search`，2026-09-24 新增）

独立搜索页：**一次搜「个人笔记 + 所有知识库归档」**。顶栏搜索框回车即到此页（表单 `action=/web/search`，带 `mode=auto`），弹窗的「查看完整列表」也指向此页。

| 参数 | 默认 | 说明 |
|------|------|------|
| `q` | `""` | 关键词；空则显示引导页 |
| `mode` | `auto` | `auto`（先词法、零命中回退 hybrid）/ `lex` / `hybrid` / `sem`（同 `/api/search`） |
| `sources` | `""` | `note` / `kb` / `code`（逗号分隔），空 = 全部 |
| `kb` | `""` | 限定单个知识库（隐含 `sources=kb`）；结果上方的库标签可直接点选/清除 |
| `rerank` | 空 | `0` / `1`（受全局配置约束），空 = 跟随全局 |
| `limit` | `100` | 每页条数（20/50/100/200，≤200） |
| `offset` | `0` | 分页偏移 |

页面内容：命中数 / 耗时 / **实际模式** / 笔记与知识库计数、**知识库分面**（各库命中数，可点选过滤）、结果卡片（**复用 `static/app.js` 的 `LN.renderSearchHits`**，与顶栏弹窗、「搜索索引」页同款：综合相关度徽章 + 覆盖/近邻/语义/重排/块/词频/bm25/时间等全部打分明细 + `<mark>` 高亮摘要）、分页；页面内嵌**「搜索帮助」面板**（逐项解释 mode/sources/kb/rerank/分页/查询语法，并注明 `auto` = 先词法、零命中再语义兜底），且选择模式后在其下方显示当前模式的一句话说明。模板 `templates/web_search.html`；检索一次取到上限（`MAX_LIMIT=500`）后在服务端切片（命中原始数据以 JSON 注入，前端渲染），分面基于全量命中而非仅当前页。模式说明常量见 `routers/web.py` 的 `MODE_HELP`。

**`mode=auto` 含义**：先用倒排索引词法检索（毫秒级）；**只有当一条都没命中时**才自动改用语义检索兜底。既有词法命中就快，长句/自然语言也不会"无命中"——所以是默认推荐值。

**搜索历史**：搜索框保存最近 20 次查询（浏览器 `localStorage`，键 `ln_search_history`），聚焦时以下拉候选提示；顶栏搜索框与搜索页共用同一份历史（`window.lnSearchHistory`）。

**索引管理面板**（2026-09-25 起）：搜索页搜索框下方多了「索引管理」折叠面板，列出全部本机库（代码库根 / 知识库工作区）及其已索引文档数与最近扫描状态，管理员按库点「创建索引」即可扫该库（含代码文件）并顺带嵌入向量；数据来自 `GET /api/search/index_libs`。

> 注意：`rerank` 参数在页面侧以**字符串**接收（表单未选时提交空串，用 `Optional[int]` 会触发 `int_parsing` 报错），空串 = 跟随全局配置。

### 1.5 本机库索引（`source=code`：代码库根 + 知识库工作区）

把**本机目录**纳入检索——与笔记 / 知识库并列的第三类索引源。2026-09-25 起有两类登记方式：

| `kind` | 是什么 | 何时扫 | 怎么配 |
|--------|--------|--------|--------|
| `code` | **代码库根** | `SCAN_TTL_S=5s` 自动增量 + 手动重建 | 状态页「本机库索引」卡片 / `PUT /api/search/code_roots` |
| `kbws` | **知识库工作区** | **只在手动「创建索引」时**（不自动、不上传 depot） | 知识库页设工作区目录 |

| 端点 | 权限 | 说明 |
|------|------|------|
| `GET /api/search/code_roots` | 登录 | 读取已配置代码库根（`label/root/exists`）与 `code_exts` |
| `PUT /api/search/code_roots` | 管理员 | 保存根目录（`{"roots":["D:\\path\\repo"]}`）并**自动后台重建**；传 `[]` 清空 |
| `GET /api/search/index_libs` | 登录 | 可建索引的本机库：`label/kind/name/root/exists/docs/scan`，附 `exts` / `max_files` / `max_bytes` |
| `POST /api/search/index_lib` | 管理员 | **手动建索引**：`{"label":"l_notepad_client","embed":true}` → 只扫该库，返回 `files/touched/duration_ms/capped`，`embed=true` 时随后台嵌入向量 |
| `GET /api/search/code/file?root=&file=` | 登录 | 只读查看（`root` = 库标签，代码库根与知识库工作区都可）；路径限定在库根内 |
| `GET /web/code?root=&file=&hl=` | 登录 | 网页只读查看页：行号 + `hl`（逗号分隔词）高亮，超 `5000` 行只渲染前 5000 行 |

```bash
# 看有哪些本机库、各自索引了多少
curl -s "http://127.0.0.1:8765/api/search/index_libs"

# 手动给某个库建索引（含 .py 等代码文件），并顺带嵌入向量
curl -s -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"label":"l_notepad_client","embed":true}' "http://127.0.0.1:8765/api/search/index_lib"
# → {"ok":true,"label":"l_notepad_client","kind":"code","files":44,"touched":44,"capped":false,"duration_ms":210,"embed_started":true}
```

- **扫描规则**：扩展名按 `CODE_EXTS`（`.py/.pyi/.ui/.qss/.bat/.cmd/.ps1/.sh/.toml/.yaml/.yml/.json/.js/.ts/.html/.css/.c/.h/.cpp/.cs/.java/.rs/.go` + 文档类）；跳过 `CODE_SKIP_DIRS`（`.git/__pycache__/node_modules/.venv/site-packages/dist/build/target/.vs/obj/...`）；单文件 ≤ `MAX_INDEX_BYTES`(2MB)；增量比对 `mtime/size`。
- **体量治理**：单库超 `L_NOTEPAD_CODE_MAX_FILES`（默认 20000 文件）或 `L_NOTEPAD_CODE_MAX_BYTES`（默认 512MB）即停止扫描并标 `capped=true`（**截断时不会删索引行**，避免误删）；页面/接口都会提示「已截断」。**不要把代码根指到整棵树**（`rez-package-source` 全树有 43 万+ `.py`）。
- **命中**：`source=code`、`kb_name`=库标签（可 `sources=code` 过滤、分面、按库建索引），`rel`=相对库根的路径；`open_url` 指向 `/web/code?root=&file=&hl=`。
- **参与向量语义（2026-09-25 起）**：代码库文件按本机文件内容嵌入（键 `code:<label>:<rel>`，与笔记/知识库共用分块与模型），所以"口语症状"（"程序卡很久"）也能语义召回代码（实测 `vec≈0.63`）；单篇嵌入失败只跳过并计数（`_EMBED_RETRY_MAX=3` 后放弃），不阻塞其余文档。
- **知识库工作区为什么不走归档**：`.py` 进 depot 会连带体积/配额问题，且自动上传与"手动创建索引"的预期相反。工作区走本机索引后，`workspace_sync` 的上传白名单（`WORKSPACE_EXTS`）仍是文档类型，**`.py` 只进本地索引、不会被动上传**；代价是工作区索引需手动重建（手动全量重建也会带上它们）。

### 1.6 「要搜索哪些包」（rez 源码包，2026-09-25 新增）

搜索页可勾选「要搜索哪些包」（`rez-package-source` 下带 `package.py` 的目录，本机 54 个），
只在这些包里搜代码；笔记与知识库不受勾选影响。

| 端点 | 权限 | 说明 |
|------|------|------|
| `GET /api/search/code_packages` | 登录 | `root`（货架目录）+ `packages[]`（`label/root/exists/docs/indexed/scan`） |
| `GET /api/search?packages=a,b` | 登录 | 只保留 `source=code` 且 `kb_name ∈ {a,b}` 的命中 |

- **货架位置**：`L_NOTEPAD_PKG_ROOT`（`package.py` 里按 `{root}/../../../rez-package-source` 给，即 `<trayapp>/rez-package-source`）；
  页面设置 `code_pkg_root` 可覆盖。
- **默认不勾选 = 不限**（搜全部已建索引的库）；勾选后只搜勾选的包。
- **每个包要先建索引**：`POST /api/search/index_lib {"label":"l_agent_chat"}`（`kind=pkg`，手动，不参与 TTL 自动刷新）；
  没建过的包在列表里显示「未建索引」，搜不到。实测 `l_agent_chat`：133 文件 / 0.7s。
- **不要指望一次建全部**：货架全量是 3 万+ 文件 / ≈1.9GB（单 `ChatRoom` 就 1.3 万文件 / 656MB），
  逐个建、按需建；体量上限（`CODE_MAX_FILES` / `CODE_MAX_BYTES`）仍逐包生效。
- **记住选择**：浏览器 `localStorage['ln_search_packages']`（换设备/换浏览器要重选）；
  从顶栏进搜索页时会自动把记住的包补进 URL（`pkgs_saved=1` 防循环）。
- **包名过滤框**（2026-09-25 新增）：54 个包的列表上方有「过滤包名…」输入框，只隐藏不匹配的项，
  **不影响已勾选状态**（勾了再过滤，提交仍是全部勾选项）；过滤时提示「匹配 N 个包」。

---

## 2. 返回结构与打分

`GET /api/search` 顶层：

```json
{
  "query": "创建包", "limit": 20, "offset": 0,
  "total": 22,                 // 命中总数（分页用；hybrid 下为词法命中数，sem 下为语义命中数）
  "took_ms": 3.05,             // 服务端检索耗时
  "fallback": false,           // true = 引号短语无结果，已自动回退模糊匹配
  "vec": {"used": true, "model": "bge-m3", "hits": 24, "reason": ""},
  "hits": [ /* 见下 */ ]
}
```

`hits[]` 每项：

| 字段 | 说明 |
|------|------|
| `path` / `rel` | 文档相对路径（笔记 = 相对 `notepad_list`；知识库 = 相对工作区目录） |
| `source` | `note` / `kb` / `code` |
| `kb_name` | 知识库名（`source=kb`）/ 本机库标签（`source=code`：代码库根目录名或知识库工作区目录名） |
| `snippet` | **命中位置附近** 180 字符摘要（优先短语命中位置 → 否则命中词块最密集的窗口；找不到命中才回退开头） |
| `matches` | 高亮词列表（相邻 bigram 会合并，如 `创建`+`建包` → `创建包`） |
| `coverage` | 覆盖率 = 命中词块的 **IDF 加权和** / 总权重（单字不计入；`term_idf` 判定的泛词权重为 0，既不算命中也不算分母） |
| `tf` | 词频分 = Σ (IDF 权重 × min(单元出现次数, **5**)) / (5 × 总权重) |
| `proximity` | 近邻度 = `1/(1+首现跨度/200)`，命中短语直接记 1.0（泛词不参与定位） |
| `phrase_hits` | 引号短语精确命中次数 |
| `bm25` | FTS5 原始分（负值，越负越相关），列权重 标题 6 / 正文 1 |
| `vec` | 语义相似度（余弦，0~1；未参与则为 0） |
| `score` | 综合分（见下） |
| `updated_at` | 文档最后修改时间 |
| `explain` | **判断依据**（2026-09-25 新增，供页面「判断依据」对话框 / Agent 解释排序）：`parts`（各分项权重×取值=得分）、`terms`（命中词块 + IDF 权重 + 出现次数）、`terms_generic`（被判泛词、未参与打分）、`terms_missed`、`rank`/`of`/`order_by`（名次与排序主序）、`vs_next`（与下一条的分差与主因）、`summary`（一句话结论） |
| `open_url` | 前端打开地址：笔记 → `/web/{rel}`；知识库 → `/web/kb/{kb}?file={rel}`；本机库 → `/web/code?root={label}&file={rel}&hl={matches}` |

**打分公式**（`search_index._score()`，权重为模块常量）：

```
score = 3.0×短语命中 + 2.0×覆盖率 + 1.0×词频 + 1.0×近邻度 + 1.5×(-bm25)     // 词法部分
score += 1.2×vec                                                            // 混合模式叠加语义分
```

**判断依据（`explain` / 页面「判断依据」按钮）**：每条命中都带 `explain`，点结果卡片上的
「判断依据」按钮弹出对话框，逐项拆开：

- **分项表**：`短语命中/覆盖率/词频/近邻度/bm25/语义` 各自的 **权重、取值、得分**（权重×取值）；
- **逐词块**：命中的词块（标签里给 IDF 权重 `w` 与出现次数 `×N`）、未命中的词块、以及
  **被判为 repo 泛词**（覆盖 ≥30% 文档）而权重记 0 的词块 —— 这就是"为什么 `notepad`/`client`
  没能把某篇顶上去"的答案；
- **名次与主序**：`第 N / M 名`、`排序主序：重排分 | 综合分`；
- **为什么排在它前面**：与**下一条**的分差，以及差异最大的三个分项（如 `bm25 +9.04`）；
- **一句话结论**（`explain.summary`），可直接给 Agent 当"为什么召回它"的依据。

> 实测示例（口语整句 `ctrl+中键呼出…整个电脑都卡很久`，`mode=auto`）：第 1 名
> `folder_favorites_hotkey.py` 得分 36.44，其中 bm25 贡献 35.22、语义 0.75、覆盖率 0.35，
> 泛词 `l/notepad/client` 权重 0；重排分 0.70 作主序，比第 2 名高 1.77。

排序：
- `mode=sem` → 只保留有语义分的文档，**按 `score`（= 语义相似度）降序**；
- 其它模式 → 按 `(短语命中数, score, path)` 降序。候选集先按 bm25 取 `(offset+limit)×6` 条再重排（保证相关度优先）。

实测排序示例（`创建包`）：

| 文档 | 短语 | 覆盖 | 词频 | 近邻 | 总分 |
|---|---|---|---|---|---|
| `a.md`（"创建包"连写） | 0 | 1.0 | 0.2 | **0.995** | **3.195** |
| `b.md`（"创建…建包"挨着） | 0 | 1.0 | 0.2 | 0.952 | 3.152 |
| `c.md`（出现多次但离得远） | 0 | 1.0 | **0.3** | 0.099 | 2.399 |
| `d.md`（只有"创建"） | 0 | 0.5 | 0.1 | 0.0 | 1.100 |

---

## 3. 查询语法

| 输入 | 行为 |
|------|------|
| `创建包` | 中文切二元组（`创建`、`建包`），**段内 OR 召回**、段间 AND —— 宽召回，再按覆盖率/近邻排序 |
| `创建 rez 包` | 三段：`创建` AND `rez` AND `包`（英文整词、单字前缀） |
| `"创建包"` | 引号 = **FTS5 短语**，要求 bigram 相邻（精确匹配）；**无结果时自动回退模糊**，响应 `fallback=true` |
| `search` | 英文/数字按整词（大小写不敏感） |
| `笔` | 单字 → 前缀查询（匹配以该字开头的所有词块） |
| 长句（**段数 > 4**） | **段间不再 AND，改成全部单元 OR + 覆盖率排序**：自然语言原句会被切成十几段（`ctrl+中键呼出…整个电脑都卡很久` → 6–8 段），段间 AND 会把命中压到 1–2 篇，目标文件根本进不了候选（重排也救不了）。实测同一句：AND 时 `total=2`，OR 后 `total=87`，目标文件可被 rerank 顶到第 1 |

> 注意：单字查询不参与覆盖率/词频计算（信息量太低），所以搜单字时 `coverage` 恒为 1。

---

## 4. 检索模式

| 模式 | 行为 |
|------|------|
| `lex` | 纯词法（FTS5 倒排）。最快（本机 3-5ms） |
| `hybrid`（默认） | 词法为主 + 语义**加分**（`+1.2×vec`）；**词法零命中时**用语义结果兜底（纯语义命中的文档才补进结果，避免弱模型带来噪声） |
| `sem` | 纯语义：只保留语义分 ≥ 阈值的文档，按相似度排序；语义零命中时返回 `total=0`（不回退词法，避免"总数 15 但列表空"的误导） |

---

## 5. 语义检索（向量）

流程（`search_vec.py`）：文档按段分块 → 本机 Ollama embedding → 归一化 float32 存 `vec_chunks`（`vec_docs` 记 `mtime/size/model` 做增量）→ 查询嵌一次 → 与内存缓存的所有块向量点积（=余弦）→ **每篇取最高分块** 作为该文档的语义相似度。

| 项 | 值 / 规则 |
|---|---|
| 分块大小 | `min(900, 模型 ctx × 0.8)` 字符：`bge-m3` → **900**；`bge-*-zh-v1.5`（ctx 512）→ **409**（避免尾部被截断） |
| 块重叠 | 块大小 / 8 |
| 批量嵌入 | 每次 16 块（`/api/embed`，失败回退 `/api/embeddings`） |
| 查询前缀 | `bge-*-zh-v1.5` 自动加「为这个句子生成表示以用于检索相关文章：」，`bge-m3`/`nomic` 不加 |
| 内存缓存 | 最多 20000 块（超限只缓存最近的） |
| 阈值（按模型标定） | `bge-m3` **0.55**；`zh-v1.5` **0.35**；其它 **0.45**；另加相对窗口 `最高分-0.15` |
| 单文件上限 | 词法索引 2MB；语义按块截断 |
| 环境变量 | `L_NOTEPAD_EMBED_URL`（默认 `http://127.0.0.1:11434`）、`L_NOTEPAD_EMBED_MODEL`（覆盖页面设置）、`L_NOTEPAD_VEC_ENABLED=0`（整体关闭语义） |
| 页面设置 | `app_settings.embed_model`（优先级：环境变量 > 页面设置 > 自动挑 `bge-m3→bge-base-zh→bge-small-zh→nomic`） |

**模型目录**（`GET /api/search/models`）：

| 模型 | 维度 | ctx | 体积 | 说明 |
|---|---|---|---|---|
| `bge-m3` | 1024 | 8192 | 1.2GB | 质量最好、多语言；CPU 内存约 1.3-1.6GB |
| `quentinz/bge-base-zh-v1.5` | 768 | 512 | 205MB | 中文；省内存首选（2C4G 服务器推荐） |
| `qllama/bge-small-zh-v1.5` | 512 | 512 | 26MB | 超轻，2C4G 可用 |
| `nomic-embed-text` | 768 | 2048 | 274MB | 英文为主，**中文质量差**（仅兼容保留） |

**切换模型**（不会自动下载）：

```bash
curl -s -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"model":"quentinz/bge-base-zh-v1.5"}' "$BASE/api/search/model"
# → 已安装：{"ok":true,"model":"...","need_embed":true}   （切换后需重嵌）
# → 未安装：{"ok":false,"need_download":true,"error":"模型未安装：..."}
curl -s -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"model":"quentinz/bge-base-zh-v1.5"}' "$BASE/api/search/model/download"   # 用户确认后才下载
```

切换时会清理**旧模型**的 `vec_docs`/`vec_chunks` 并清内存缓存；向量与当前模型不匹配时语义自动降级并在 `vec.reason` 里说明（"向量需要重新嵌入"）。

---

### 5.1 本地重排（rerank，可选，2026-09-25 本机实测可用）

**为什么需要**：向量点积在同 repo 的文件之间几乎没有区分度（实测全落在 0.58–0.65），"哪个文件才是元凶"分不出来；交叉编码器逐对打分才能分得开（实测目标文件从第 3 顶到第 1）。

**本机部署**（CPU-only）：

| 项 | 值 |
|---|---|
| 二进制 | `D:\Tools\llama.cpp\bin\llama-server.exe`（llama.cpp release `b11179` win-cpu-x64） |
| 模型 | `D:\Tools\llama.cpp\models\bge-reranker-v2-m3-Q8_0.gguf`（606MB，来自 `gpustack/bge-reranker-v2-m3-GGUF`） |
| 启动 | `D:\Tools\llama.cpp\start_rerank.bat` → `llama-server --reranking --host 127.0.0.1 --port 11435 --ctx-size 4096 -b 2048 -ub 2048 -t 16 -np 4` |
| 服务端 env | `l_notepad_server/package.py`（`Lugwit_deploy` 为假时注入）：`L_NOTEPAD_RERANK_URL=http://127.0.0.1:11435`、`RERANK_TOP_N=8`、`RERANK_MAX_CHARS=300`、`RERANK_TIMEOUT_S=8` |

**为什么这几个参数**（CPU 实测 bge-reranker-v2-m3 Q8 ≈ **1.5ms/token**，耗时 ≈ 候选数 × 候选 token 数）：

- `L_NOTEPAD_RERANK_TOP_N`：候选数（代码默认 40，那在 CPU 上是十几秒；本机 8 个 ≈ 0.6s）。
- `L_NOTEPAD_RERANK_MAX_CHARS`：**每个候选只送「查询词附近」的一窗**（代码默认 400）。块按 900 字符切分 ≈ 350 token，不截断的话每个候选就要 0.5s。
- llama-server 的 `-ub` 别用默认 512：512 装不下一个 300–400 字符的候选 → 退化成一次只打一个候选；`-ub 2048` 才能一批打完。
- 实测（16C/32T）：5 候选 `rerank≈0.6s`、整轮 `hybrid`（含 ollama 嵌入）≈**0.85s**；首轮含 embedding 冷启动约 5s。

> 连不上 / 超时一律降级回融合排序（`rerank.used=false` + `reason`），不报错、不返回空；失败 3 次进入 60s 冷却。

---

## 6. 索引维护与状态

- **索引源**：① 个人笔记（`notepad_list`，`source=note`）；② 知识库**已上传到 depot 的归档版本**（`source=kb`，按 `.md/.markdown/.txt/.rst/.log` 过滤，见 `search_index._sources()` / `_kb_sync_one()`）。**本机工作区目录不参与归档索引**（2026-09-24 按代码修正），但可作为**本机库**手动建索引（`kind=kbws`，2026-09-25 起，见 §1.5）；③ **本机库**（`source=code`：代码库根 `kind=code` + 知识库工作区 `kind=kbws`，见 §1.5）。
- **建索引**：中文按二元切分后写入 FTS5（`search_fts`），原文与 `size/mtime` 存 `search_docs`
- **增量更新**：
  - 服务内增删改 → `file_store` 变更通知即时标脏，下次查询补索引；
  - 外部改动（桌面端落盘、托盘写入）→ **5 秒 TTL** 全量比对 `mtime/size`，内容没变不重读；
  - 后台重建期间 `refresh()` 直接跳过（搜索读旧索引，不阻塞）
- **刷新机制（2026-09-17 明确）**：**没有定时重建**；每次搜索前调用 `search_index.refresh()`：
  ① 进程内被标脏的文件（增删改钩子）立即增量索引；
  ② 距上次全量比对 ≥ **`SCAN_TTL_S = 5` 秒**时，遍历**个人笔记**目录按 `size/mtime` 增量比对（知识库归档另由后台线程按 `KB_SCAN_TTL_S = 300` 秒事件/TTL 同步，比对 `size/rev`，见 `search_index._kb_sync_one()`）。
  **全量重建仅手动**：`POST /api/search/reindex`（同步）/ `POST /api/search/reindex_async`（后台），
  时间点见 `stats.reindex.started_at` / `finished_at`（重建会带上全部本机库，含知识库工作区）
- **手动建单个库**（2026-09-25 起）：`POST /api/search/index_lib`（见 §1.5）——只扫一个库、含代码文件、
  不上传 depot；`embed=true` 时随后台嵌入向量。搜索页「索引管理」面板就是这个入口
- **慢查询先看这条**：**无 ollama 时默认 `hybrid` 每次约 6 秒**（反复探测 `127.0.0.1:11434` 超时），
  而 `mode=lex` 约 10ms
- **`GET /api/search/stats` 字段**：`docs` / `fts_rows` / `fts_consistent` / `db_bytes` / `sources[]`（每源 `root/docs/bytes/last_indexed_at`，本机库行另有 `kind/editable/scan`，`deep=1` 加 `disk_files/missing/changed/extra`）/ `code_libs` / `code_max_files` / `code_max_bytes` / `pending` / `scan_ttl_s` / `last_scan_ago` / `max_index_bytes` / `workspace_exts` / `reindex`（后台重建进度）/ `vec`（模型、块数、维度、按来源 `by_source`、嵌入进度、失败放弃 `dropped`、下载进度、目录）/ `deep` / `fts_integrity`
- **相关环境变量（本机库）**：`L_NOTEPAD_CODE_MAX_FILES`（20000）/ `L_NOTEPAD_CODE_MAX_BYTES`（512MB）/ `L_NOTEPAD_VEC_CODE_CHUNKS`（8000，代码块的内存缓存额度）

```bash
# 后台重建（管理员）→ 轮询进度
curl -s -X POST -H "Authorization: Bearer $TOKEN" "$BASE/api/search/reindex_async"
curl -s -H "Authorization: Bearer $TOKEN" "$BASE/api/search/stats" | jq '.reindex'
```

---

## 7. 权限模型

| 来源 | 可见条件 |
|------|---------|
| 笔记（`source=note`） | 拥有者（`note_registry`）/ 被共享（`note_shares.shared_with` = 用户或 `*`）/ **管理员全部可见**（含未登记文件） |
| 知识库工作区（`source=kb`） | 该知识库存在即对**所有登录用户**可见（知识库当前无 owner 概念） |
| 代码库（`source=code`） | 对所有登录用户可见（本机管理员配置的目录） |

权限过滤在 SQL 内完成（`_PERM_SQL`），语义召回的补充文档同样过一遍权限，不会越权。

### 7.1 本机直连免 token（2026-09-16 起）

**直连后端**（`http://127.0.0.1:8765/api/...`）的 **GET/HEAD** 请求**不需要登录态**，
方便本机脚本 / curl / 托盘调试：

```bash
# 无需 token（仅限直连 8765 且是 GET）
curl -s "http://127.0.0.1:8765/api/search?q=创建包&limit=3"
curl -s "http://127.0.0.1:8765/api/kb/rez_pkg/depot/list"
curl -s "http://127.0.0.1:8765/api/search/stats"
```

| 条件 | 说明 |
|---|---|
| **直连后端** | 请求里**没有** `X-Real-IP` / `X-Forwarded-For`。经 nginx（`/note/...`）的请求一定带这两个头 → **从不解禁**（否则同机 nginx 会把所有远程请求伪装成 127.0.0.1） |
| **对端回环** | `request.client.host ∈ {127.0.0.1, ::1, localhost}`；即使后端被绑到 `0.0.0.0`，远程直连也不会免鉴权 |
| **只读** | 仅 `GET`/`HEAD`；`POST/PUT/DELETE`（发布、提交归档、删除、重建索引、切模型…）仍需登录 + 管理员 |
| **身份** | 按 **guest** 处理 → 权限过滤照旧：看不到他人笔记；知识库对所有"登录用户"可见，所以 guest 能看到知识库内容 |
| **关闭开关** | 环境变量 `L_NOTEPAD_LOCAL_NO_AUTH=0` |

所以：
- 本机脚本想免 token → 打 `127.0.0.1:8765`
- 本机**浏览器**访问 `http://127.0.0.1:8080/note/...` → 仍要登录（经 nginx 反代）
- 部署机上远程用户 → 一律要登录 ✓

---

## 8. 前端接入示例

```js
// 全局搜索（混合模式）
const r = await fetch(root + "/api/search?q=" + encodeURIComponent(q) + "&limit=20&mode=hybrid");
const { total, took_ms, fallback, vec, hits } = await r.json();
// hits[i].open_url 可直接给 <a href>；hits[i].matches 用于在 snippet 里做 <mark> 高亮

// 知识库页搜索（作用域 = 本知识库，后端已过滤）
await fetch(root + "/api/kb/" + encodeURIComponent(kbName) + "/search?q=" + encodeURIComponent(q));

// 重建/嵌入进度轮询（状态页）
const s = await (await fetch(root + "/api/search/stats")).json();
if (s.reindex.running) { /* s.reindex.phase: counting|indexing, done/total */ }
if (s.vec.embed.running) { /* s.vec.embed.docs/done/total */ }
if (s.vec.download.running) { /* s.vec.download.status/completed/total */ }
```

高亮注意：服务端不给 HTML，只给 `matches`；前端务必**先转义再插 `<mark>`**（知识库页 `hl()`、列表页服务端 `search_index.highlight()` 都已按此实现）。

---

## 9. 已知限制与坑

1. **热重载不可靠**：`l_notepad_server` 的 `SrcWatchService` 改 `.py` 时常"只记重启、进程不换"（2026-09-16 17:10 一次直接把 8765 停掉 → nginx 全 502）。**改了 Python 请手动重启**；模板 `.html` 是即时生效的。
2. **FTS5 依赖**：需要 CPython 3.11+ 自带的 SQLite（含 FTS5）。旧环境无 FTS5 时索引建表会失败。
3. **中文必须先切分**：`unicode61` 分词器不切汉字（整段汉字会变成一个 token），所以索引/查询都走自定义二元切分——这也是"搜不到"类问题的首要排查点。查询侧另有两条护栏：**关键单字按前缀匹配**（`"卡" *`，只保留非低信息字）、**同义扩展**（`卡 ↔ 卡顿/卡死/阻塞`，见 `_SYNONYMS`）；索引侧靠 **IDF 泛词抑制**（覆盖率 ≥ 30% 的词块不参与打分）压制 `notepad/client/窗口` 这类 repo 泛词。
4. **模型质量**：`nomic-embed-text` 中文语义差（实测"部署流程"会误召回"牛肉面"），`bge-m3` 与 `bge-base-zh-v1.5` 才可用于中文。
5. **语义耗时**：向量点积是纯 Python（无 numpy），本机 711 块约 110-250ms；2 核服务器会更慢，且 2C4G 上 `bge-m3`（CPU 约 1.3-1.6GB）易 OOM —— 建议服务器用 `bge-base-zh-v1.5` 或直接 `L_NOTEPAD_VEC_ENABLED=0`。
6. **`sources` / `mode` 取值非法时**不会报错，按"不过滤"和"回退 hybrid/lex 逻辑"处理；调用方最好显式传值。
7. **`total` 语义**：`hybrid` 下等于词法命中数（语义只加分/兜底），`sem` 下等于语义命中数；分页要按同一 `mode` 翻页。
8. **归档与检索是两件事**：检索读的是**索引库**（本地 SQLite），归档读的是 **depot 版本库**；工作区文件没提交不影响搜索，但归档列表里不会出现。
9. **本机库索引的代价（2026-09-25）**：代码根不要指整棵树（`rez-package-source` 下有 43 万+ `.py`），超 `CODE_MAX_*` 会截断；知识库工作区是**手动建索引**（改完文件需再点一次「创建索引」），且**不参与 depot 归档**；代码块要占内存缓存额度（`L_NOTEPAD_VEC_CODE_CHUNKS`），代码量很大时会挤掉部分笔记块（缓存未命中=该文档本轮无语义分）。
10. **IDF 是全局统计**：`term_idf()` 用 FTS5 词表算全库文档频率，样本太少（`docs < 20`）时不启用；泛词判定按整库比例，所以「库很小 + 词很常见」时可能把有用的词也判成泛词。

---

## 10. 相关代码

| 文件 | 作用 |
|------|------|
| `search_index.py` | FTS5 索引维护、查询解析、打分重排、后台重建、`stats()` |
| `search_vec.py` | 分块/嵌入/向量存储与检索、模型目录与切换、下载与重嵌 |
| `depot_map.py` | 知识库 ↔ depot 归档映射（`{library}/{subpath}/{rel}` + 工作区 maps） |
| `routers/search.py` | `/api/search*` 端点 |
| `routers/kb.py` | `/api/kb/{kb}/search`、`/api/kb/{kb}/depot*`、工作区端点 |
| `templates/web_index.html` | 搜索索引状态页（试搜、KPI、重建/嵌入/下载进度） |
| `templates/web_search.html` | 全局搜索页（含「索引管理」手动建索引面板） |
| `templates/web_code.html` | 本机库文件只读查看页（行号 + 命中高亮） |
| `static/app.js` | `LN.renderSearchHits`（结果卡片渲染，笔记/知识库/本机库共用） |
| `templates/web_kb.html` | 知识库页（本库搜索、归档映射、托盘连接状态） |
