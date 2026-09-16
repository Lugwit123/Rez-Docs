# l_notepad 搜索接口使用文档

> 适用版本：`l_notepad_server` 999.0（2026-09-16，含倒排索引 + 向量语义检索 + 知识库归档映射）
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

---

## 1. 接口总览

### 1.1 搜索（`routers/search.py`，前缀 `/api/search`）

| 方法 | 路径 | 权限 | 说明 |
|------|------|------|------|
| GET | `/api/search` | 登录 | 检索用户可见的**笔记 + 知识库工作区**文档 |
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
| `sources` | str | 全部 | 逗号分隔来源过滤：`note`（个人笔记）/ `kb`（知识库工作区） |
| `mode` | str | `hybrid` | `lex` 纯词法 / `hybrid` 词法+语义 / `sem` 纯语义 |

### 1.2 知识库（`routers/kb.py`）

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/kb/{kb}/search?q=&mode=&limit=&offset=` | **单知识库范围检索**（作用域下沉到 SQL：`search_docs.kb_name = ?`） |
| GET | `/api/kb/{kb}/depot` | 归档映射（`library` / `subpath` / `base_path` / `ws_name` / `ws_id` / `local_root` / `remote_url`） |
| PUT | `/api/kb/{kb}/depot` | 改映射（body `{"library"?, "subpath"?, "ws_name"?}`；`null`=不改，`""`=回默认），改完自动重建工作区 maps |
| GET | `/api/kb/{kb}/depot/list?rel=` | 列归档目录（返回项带 `rel`，相对 `base_path`） |
| GET | `/api/kb/{kb}/depot/file?rel=&rev=0` | 读归档文件内容（`rev=0` = 最新） |
| POST | `/api/kb/{kb}/depot/submit?rel=&description=` | 把 body 原始字节提交为新版本 |
| GET/PUT | `/api/kb/{kb}/workspace[/file]` | 工作区本地目录（服务端模式；托盘模式见《知识库本机模式》） |
| GET | `/api/kb/{kb}/workspace/reveal` | 在**服务器**资源管理器打开目录 |

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
| `source` | `note` / `kb` |
| `kb_name` | 知识库名（`source=kb` 时） |
| `snippet` | **命中位置附近** 180 字符摘要（优先短语命中位置 → 否则命中词块最密集的窗口；找不到命中才回退开头） |
| `matches` | 高亮词列表（相邻 bigram 会合并，如 `创建`+`建包` → `创建包`） |
| `coverage` | 覆盖率 = 命中的查询词块数 / 总词块数（单字不计入） |
| `tf` | 词频分 = Σ min(单元出现次数, **5**) / (5 × 词块数) |
| `proximity` | 近邻度 = `1/(1+首现跨度/200)`，命中短语直接记 1.0 |
| `phrase_hits` | 引号短语精确命中次数 |
| `bm25` | FTS5 原始分（负值，越负越相关），列权重 标题 6 / 正文 1 |
| `vec` | 语义相似度（余弦，0~1；未参与则为 0） |
| `score` | 综合分（见下） |
| `updated_at` | 文档最后修改时间 |
| `open_url` | 前端打开地址：笔记 → `/web/{rel}`；知识库 → `/web/kb/{kb}?file={rel}` |

**打分公式**（`search_index._score()`，权重为模块常量）：

```
score = 3.0×短语命中 + 2.0×覆盖率 + 1.0×词频 + 1.0×近邻度 + 1.5×(-bm25)     // 词法部分
score += 1.2×vec                                                            // 混合模式叠加语义分
```

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

## 6. 索引维护与状态

- **索引源**：个人笔记（`notepad_list`，`source=note`）+ 各知识库工作区（`WORKSPACE_EXTS = .md/.markdown/.txt/.rst/.log`，`source=kb`）
- **建索引**：中文按二元切分后写入 FTS5（`search_fts`），原文与 `size/mtime` 存 `search_docs`
- **增量更新**：
  - 服务内增删改 → `file_store` 变更通知即时标脏，下次查询补索引；
  - 外部改动（桌面端落盘、托盘写入）→ **5 秒 TTL** 全量比对 `mtime/size`，内容没变不重读；
  - 后台重建期间 `refresh()` 直接跳过（搜索读旧索引，不阻塞）
- **`GET /api/search/stats` 字段**：`docs` / `fts_rows` / `fts_consistent` / `db_bytes` / `sources[]`（每源 `root/docs/bytes/last_indexed_at`，`deep=1` 加 `disk_files/missing/changed/extra`）/ `pending` / `scan_ttl_s` / `last_scan_ago` / `max_index_bytes` / `workspace_exts` / `reindex`（后台重建进度）/ `vec`（模型、块数、维度、嵌入进度、下载进度、目录）/ `deep` / `fts_integrity`

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
3. **中文必须先切分**：`unicode61` 分词器不切汉字（整段汉字会变成一个 token），所以索引/查询都走自定义二元切分——这也是"搜不到"类问题的首要排查点。
4. **模型质量**：`nomic-embed-text` 中文语义差（实测"部署流程"会误召回"牛肉面"），`bge-m3` 与 `bge-base-zh-v1.5` 才可用于中文。
5. **语义耗时**：向量点积是纯 Python（无 numpy），本机 711 块约 110-250ms；2 核服务器会更慢，且 2C4G 上 `bge-m3`（CPU 约 1.3-1.6GB）易 OOM —— 建议服务器用 `bge-base-zh-v1.5` 或直接 `L_NOTEPAD_VEC_ENABLED=0`。
6. **`sources` / `mode` 取值非法时**不会报错，按"不过滤"和"回退 hybrid/lex 逻辑"处理；调用方最好显式传值。
7. **`total` 语义**：`hybrid` 下等于词法命中数（语义只加分/兜底），`sem` 下等于语义命中数；分页要按同一 `mode` 翻页。
8. **归档与检索是两件事**：检索读的是**索引库**（本地 SQLite），归档读的是 **depot 版本库**；工作区文件没提交不影响搜索，但归档列表里不会出现。

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
| `templates/web_kb.html` | 知识库页（本库搜索、归档映射、托盘连接状态） |
