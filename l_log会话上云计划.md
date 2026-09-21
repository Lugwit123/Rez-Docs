# l_log AI 会话上云计划（参考 l_agent_chat 的会话存储）

> **性质**：**计划文档（未实施）**。先把目标、落点与取舍定下来，再动代码。
> 关联：[l_agent_chat会话存云与工作区.md](l_agent_chat会话存云与工作区.md)（同构实现，已跑通并验证）、
> [网盘版本库Depot设计.md](网盘版本库Depot设计.md)、[网盘版本库Depot演进计划.md](网盘版本库Depot演进计划.md)。
> 状态截至 2026-09-21。

## 0. 结论先行

**把 AI 会话搬进「它分析的那份日志所属的根」，随该根的库 `/l_log` 一起上云**；
agent 侧不再重复落盘。

| 环节 | 方案 |
|------|------|
| 落点 | `<根>/ai_chats/<sha1(key)[:16]>.json` → depot 路径 `/l_log/ai_chats/<sha1(key)[:16]>.json` |
| 库 / 工作区 | 复用 l_log 现有：库 `/l_log`、工作区 `l_log`、`local_root` = 根目录（**不新建库**） |
| 谁同步 | **l_log 自己**（复用 `depot_logs`），agent 不参与 |
| 同步范围 | 新增一条「会话」scope：`ai_chats/*.json`。`.json` **不进 `EXTS`** → 日志的 4 态列表不被会话污染 |
| 谁记账 | **rev 交给服务端**（`have` + `sync_plan`）；客户端只算**内容指纹**（见 §4.2） |
| agent 去重 | `POST /api/chat` 加 `persist` 开关（默认 `true` 保兼容），`agent_proxy` 传 `false` |
| 迁移 | `move` 现有 `~/.lugwit/l_log/ai_chats/*.json` → `<根>/ai_chats/`，首次提交（文件名不变 → 幂等） |

**为什么是这一版**：归属天然正确（会话跟日志同库同根）、只维护一份权威副本、复用 l_log 已有的
depot 客户端与 4 态框架，改动集中在一个包内。

## 1. 目标 / 非目标

**目标**

1. l_log 的 AI 会话（面板 / 独立窗口的多轮对话）**有云上副本**。
2. 会话与「它分析的那份日志」**归属清楚**：能回答「这份日志的对话在云上有哪些版本」。
3. 与 l_agent_chat 那套**同一套语义**（四态、不替用户选边、云端为真源），别再发明一套。
4. 支持多根：每个额外根各自的库装各自的日志**和**该根下日志的会话。

**非目标**

- 不改 depot 服务端。现有端点够用（§4.1 列了清单）。
- 不做「按单条会话（session_id）同步」：`ai_chat_store` 是**一份日志一个文件**（内含最多 10 份会话），
  同步单位就是那个文件（见 §5.6）。
- 不改 l_agent_chat 的会话语义（它的会话库继续只装「用它自己界面聊的对话」）。

## 2. 现状（代码定位）

### 2.1 现在有两套 store，各存各的

| 谁写 | 落到哪 | 上云吗 |
|------|--------|--------|
| **l_log 自己**（AI 面板 / 独立窗口） | `~/.lugwit/l_log/ai_chats/<sha1(key)[:16]>.json`（`backend/ai_chat_store.py:22`） | **不上云** |
| **l_agent_chat**（被 l_log 代理时） | `<l_agent_chat 存储根>/.l_agent_ws/sessions/session_<id>.json` | **上云 → `/l_agent_chat`** |

证据：`/l_agent_chat` 库里能列到标题带「【分析对象】追踪日志 【标题】完整日志: …」的会话 ——
那是 l_log 面板的 prompt 模板（`agent_proxy._build_turns`），不是 l_agent_chat 界面敲的。
成因：`agent_proxy` 把提问转给 `POST /api/chat`（history 每轮自带、代理侧无状态），
而 `/api/chat` **无条件** `_persist_chat(...)`（`l_agent_chat/.../app.py:1323`，没有「别存」开关）。

**所以现在的事实是**：会话确实上了云，但上的是 `/l_agent_chat`，**归属丢失**（只能从标题里的
【分析对象】文本反推），而且与 l_log 自己那份是**两份互不相干的副本**。

### 2.2 l_log 已有的 depot 能力（可直接复用）

`backend/depot_logs.py`：

| 函数 | 作用 |
|------|------|
| `library()` / `ws_name()` | 库（默认 `/l_log`）、工作区名（默认 `l_log`），可用 `L_LOG_DEPOT_LIBRARY` / `L_LOG_DEPOT_WS` 覆盖 |
| `ensure_library(lib, mode="dir")` | 建库 |
| `ensure_workspace(local_root, lib)` | 建/复用工作区：`local_root` = 根目录，`maps = [{depot_path: <库>, local_path: ""}]` |
| `list_files(local_root, lib, exts)` | 递归 `list?dir=` → `{rel: {size, rev, path}}` |
| `read_file(local_root, rel, lib, rev)` | `download` |
| `submit_file(local_root, rel, content, lib, desc)` | `submit_stream` |
| `depot_path(lib, rel)` | `/库/rel` |
| 认证 | `LUGWIT_ACCESS_TOKEN` > `LUGWIT_USER`/`LUGWIT_PASSWORD` > 凭据文件（`~/.lugwit/l_log/depot_auth.json`） |
| 入口 | `L_LOG_DEPOT_URL`，默认 **`http://127.0.0.1:1028`**（直连；l_agent_chat 走的是 `:8080/baidu` 反代） |

### 2.3 l_log 已有的版本状态

`backend/workspace_status.py`：**已经是四态**（`cloud` / `local` / `modified` / `synced` + `unknown`=归档不可达），
按 `rel` 比对，带 TTL 缓存与 `invalidate()`：

- `local_files(root_path)`：`os.walk` 根目录，`rel = relpath(full, root_path)`，**只收 `EXTS` 里的文本扩展名**，
  且**跳过点目录**
- `depot_files(root, lib)`：`depot_logs.list_files(..., exts=EXTS)`
- `EXTS` 不含 `.json`

### 2.4 前端触点（会话 I/O 全在一个文件里）

`backend/static/js/ai-agent-client.js`：

| 行 | 调用 |
|----|------|
| 431 | `GET /api/ai/chat/info?key=` —— 面板里显示「对话存在哪」，可点击复制 |
| 612 / 736 | `POST /api/ai/chat/drop`（删会话） |
| 724 | `POST /api/ai/chat/save` |
| 745 | `GET /api/ai/chat/load?key=&session_id=` |

后端对应：`app.py:1768 / 1776 / 1782 / 1788`。会话文件结构（`ai_chat_store`）：

```
{v: 2, key: <原文 key>, active_id: <sid>,
 sessions: [{id, savedAt, updated_at, history, trace, traceIndex}],  // 最多 MAX_SESSIONS=10
 updated_at}
```

## 3. 参考 l_agent_chat：7 条硬事实（照抄，别重新踩）

来自 [l_agent_chat会话存云与工作区.md](l_agent_chat会话存云与工作区.md) §4，都是实测结论：

1. **depot 路径平铺** `<库>/<rel>`，不带 `<user>` 段；`maps` 填**空列表**就是
   `/<库>/... ↔ <local_root>/...` 的隐式映射（l_log 现在填的 `{depot_path: <库>, local_path: ""}` 同义）。
2. **`sync_plan` 是「云 → 本地」的下载计划**（服务端原文：*p4 sync -n：该工作区落后于云端的条目*）。
   **本地改动服务端看不见**（要执行机扫后 `reconcile` 上报）→ 不能拿它当「本地改了什么」。
3. **`have` 不是云端权威清单**：删过的路径会以 `rev=<墓碑>` 留在里面。云端清单要问
   `GET /api/depot/list?ws=&dir=` **递归**（`list` 对 blob 库同样有效；`tree` 只列目录）。
4. **`submit_stream` 会隐式盖 `have`**；「云 → 本地」那一侧要显式
   `POST /api/depot/sync_done`，其 item 键名是 **`depot_path`**（传 `path` → 400），且**只加不减**。
5. **所有 P4 端点都要 `?ws=`**：同一账号名下有多个工作区时，服务端不会替你挑。
6. **P4 写操作要收口**：l_agent_chat 把写操作统一走一个 `_post_json()`，`depot_enabled=false` 时在那里一并拦住。
7. **本库会出「云端有元数据、内容取不到」**（`download` 404：`blob 不存在 … /apps/Lugwit/.depot/blob/f4`）。
   l_agent_chat 库里 7 个文件有 3 个是这种，**重传也补不上**（同 md5 走秒传没落地物理文件）。
   为它单列了 `BROKEN` 态（「⚠ 云端内容缺失」，只给「提交」重传）。l_log 侧会同样遇到。

## 4. 能力归属：哪些由版本库提供，哪些必须客户端做

服务端源码：`lugwit_baidu_netdisk/999.0/src/lugwit_baidu_netdisk/{web_server.py,depot_service.py}`。

### 4.1 版本库已经提供的（**白拿，别再造一遍**）

| 能力 | 端点 / 函数 | 说明 |
|------|-------------|------|
| 工作区 CRUD + 映射 | `workspace`(GET/POST)、`workspace/select`、`workspace/{id}/maps`(PUT)、`workspace/{id}/owner` | 建/选/改映射/转归属 |
| **路径互算** | `workspace/{id}/path?local=`、`workspace/{id}/local_path?path=` → `local_to_depot()` / `depot_to_local()` | 本地↔库内，**越界 400**。平铺时与客户端拼串等价，但多根 / 排除 / 子目录挂载只有服务端算得对 |
| **排除规则** | maps 里的 `exclude: true`（`is_excluded()`） | 命中自身或子树即排除；**只影响 `sync_plan` / `reconcile` 候选集**，不删历史 |
| 库管理 | `library`(GET/PUT)、`migrate` | 建库 / 改 mode / 迁库 |
| **云端清单** | `list?dir=`（`list_files` 同源） | 每文件 `rev/size/blob_md5/action/locked_by/pending/out_of_date/excluded/isdir`；`tree` 只列目录 |
| **历史与回滚** | `history?path=`、`download?path=&rev=`、`revert {path,rev}` → `revert_to()` | `revert` 是**零流量**回滚（新版本指向旧 blob）→ 客户端不用存旧内容 |
| **同步账本（rev）** | `workspace/{id}/have`、`sync_plan`、`sync_done` → `mark_synced()` | `have` = per-ws 的 `path→rev` 表（**服务端就是台账**）；`sync_plan` = 云 vs have 的下载计划 |
| **锁 / 多机保护** | `lock` / `locks` / `unlock`（`locked_by`、`locked_by_me`、`force`） | 解别人的锁要 `force` |
| **两步工作流** | `checkout` → `checkout()`、`mark_add_stream` → `mark_add()` | `checkout` = 加独占锁 + 进变更单（此时内容还没变）；`mark_add_stream` = 只把字节写进 blob 仓再登记 add |
| 变更单 | `pending`、`changes`、`change/{cl_id}`、`cl_description`、`submit_pending` → `submit_pending()`、`revert_pending` | `revert_pending` **会删 pending 行**（所以「先 `mark_add_stream` 再 revert」拿不到 blob-only 上传） |
| 提交 | `submit_stream`（一步，浏览器直传字节）、`submit`（批量）、`edit_text` | `submit_stream` 会隐式盖 `have` |
| 删除 / 移动 | `delete`（一步：mark_delete + 提交）、`mark_delete`（两步）、`move` / `move_dir` / `mark_move` | 改名别用「删+建」模拟 |
| **对账上报** | `reconcile` → `reconcile_items()` | 接收执行机扫描结果 `{local_path, action: add\|edit\|delete, md5, size}` → 落 pending |
| 清单校验（运维） | `write_manifest` / `manifest_*` + `depot_manifest_verify / replay / gc` | 可查「清单引用到的 blob 缺登记」那类漂移（T4 实测 71 条） |
| 老数据迁移 | `import` → `import_legacy_tree()` | ⚠ **只吃「网盘既有目录树」**（`dir=/apps/Lugwit/...`，带 `dry_run/limit/after/batch`）—— **不是**从客户端本地目录导入，见 §5.7 |

### 4.2 客户端**必须**自己做的（外包不了）

1. **扫描本地**：服务端看不到客户端磁盘（本机同机部署虽物理可读，但那是巧合，不能当依赖）→
   `reconcile` 的 `items` 得客户端产出。
2. **内容指纹**：`md5` / `size` 客户端算 —— 服务端的 `blob_md5` 是**百度侧混淆指纹**，
   不能拿来跟本地算的比（`网盘版本库Depot演进计划.md` T1 结论）。
3. **选择与展示**：哪一侧算数（四态、「不替用户选边」）、`.bak` 备份、写回本地、徽标与进度。
4. **blob 物理缺失**只能服务端 / 网盘侧修（实测本库 3/7 是 `download` 404），客户端只能重传试。

### 4.3 相对初稿的 5 处订正（已落到下面各节）

1. **台账降级**：只记内容指纹 `{rel: sha}`，**rev 那半交给服务端 `have` / `sync_plan`** → 见 §5.3。
2. **白拿的先进面板**：`pending`（变更单）+ `locks`（锁）+ `sync_plan`（落后于云端的条目）→ 见 §5.5。
3. **删除 / 移动用服务端的**：`delete` / `move`（或两步 `mark_delete` + `submit_pending`）→ 见 §5.6。
4. **迁移不能用 `import`**（它只吃网盘既有树）→ 仍走 move + 首次提交，可附一次 `reconcile` → 见 §5.7。
5. **把白拿项补进方案**：`history` + `revert`（历史版本恢复）、`lock`（多机同改保护）、
   `checkout` + `submit_pending`（两步工作流）、maps `exclude`（会话不参与日志计划）→ 见 §5.1 / §5.2 / §5.5。

## 5. 设计

### 5.1 落点与命名

```
<根>/ai_chats/<sha1(key)[:16]>.json          ↔  /l_log/ai_chats/<sha1(key)[:16]>.json
```

- **rel** = `ai_chats/<hash>.json`，depot 路径 = `depot_path(lib, rel)` = `/l_log/ai_chats/<hash>.json` ——
  与 l_agent_chat 的 `sessions/session_<id>.json` **同构**，现有 `depot_logs` 一行都不用改就能提交/下载。
- **文件名保留哈希**（不改成 key 原文）：key 含日志标题与内容哈希，可能超长且带非法字符；文件名与
  depot 路径都只认 `sha1(key)[:16]`，**归属靠文件内的 `key` 字段**核对（`_load_sessions` 已做全等比对）。
- **key → 文件名可反算** → 界面上「这份日志在云上的会话」不需要云端检索，直接算出来查即可。
- **归属反查要先归一化 logId**：真实界面的 logId 形如 `LogList\a\b`（历史 junction 别名），
  要经 `log_roots.normalize_log_id()` 认成 `@l_log/a/b` 再拆根 —— 否则一律落到主根、
  永远判成「不上云」（详见 §6 阶段 1 的三条注意）。
- 若将来引入子目录映射，**路径换算问服务端**（`workspace/{id}/path` / `local_path`），别自己拼。

### 5.2 同步范围：新增一条「会话」scope

`.json` **不要**并进 `EXTS` —— 否则日志的 4 态列表里会混进会话文件、`file_tree` 也会把它们当日志列。

```python
SCOPES = {
    "logs": {"exts": EXTS, "subdir": ""},                       # 现状：根目录树的文本文件
    "chats": {"exts": {".json"}, "subdir": "ai_chats/"},        # 新增：会话
}
```

- `local_files` / `list_files` 按 scope 取子集；`workspace_status` 出**两组**状态（日志 / 会话）。
- depot 侧不需要新库、新工作区 —— 同一个 `local_root` + 同一个 `maps` 就够（都在根目录树内）。
- 如果希望「会话根本不参与日志的同步计划」，可以另给一条 `maps` 行把 `ai_chats/` 标 `exclude: true`
  （服务端 `is_excluded()` 认这个），但这只是候选集过滤，**不影响**我们显式提交会话。

### 5.3 记账：内容指纹自己算，rev 交给服务端

- **rev 层面**：不用自己记 —— 服务端 `have`（per-ws `path→rev`）+ `sync_plan` 就是账本；
  「云 → 本地」完成后按要求调 `sync_done` 盖章（键名 `depot_path`，见 §3 第 4 条）。
- **内容层面**：只有客户端能算 → 台账**只存指纹**：`{rel: sha}`，`sha = sha256(提交时的那份字节)`。
- 文件：**放库外**（纯本机状态，别让它自己变成待同步内容）——
  `~/.lugwit/l_log/ai_ledger.json`（与会话原位置一致，不污染根）。
- 三方判定（照 `depot_status.from_fingerprints` 的语义）：

  | 情形 | 结论 |
  |------|------|
  | 只有本地有 | `📄 仅本地`（可提交） |
  | 只有云端有 | `☁ 仅云端`（可拉取） |
  | 都有，指纹吻合且服务端 `have` 的 rev == 云端 head | `✓ 已同步`（**不下载**） |
  | 都有，指纹不符 | `✎ 已修改`（本地动过 —— 服务端看不到的那种） |
  | 都有，`have` 的 rev != head | `✎ 已修改`（云端动过） |
  | 都有，没有台账 / 刚提交完还没记 rev | `UNKNOWN`（内部态）→ 下载比一次，一致就补台账 |
  | 都有，`download` 404 | `⚠ 云端内容缺失`（只给「提交」重传） |

### 5.4 agent 侧去重（关键一步）

- l_agent_chat：`POST /api/chat` 的请求体加 `persist: bool = True`；为 `False` 时跳过 `_persist_chat(...)`
  （`app.py:1323`）。默认 `True` 保持现有界面行为不变。
- l_log：`agent_proxy` 转发时带 `persist: False`。
- 效果：`/l_agent_chat` 的会话列表回到「只装用它自己界面聊的对话」；l_log 的对话只存一份。
- 历史遗留：`/l_agent_chat` 里已经混进去的那些 l_log 对话，**列出来让用户自己删**
  （逐条 `🗑 删云端` 已在界面上），不做自动清理。

### 5.5 界面入口（白拿的先进来）

| 位置 | 现状 | 加什么 |
|------|------|--------|
| `ai-agent-client.js:431`（「对话存在哪」） | 只显示本地路径 | 加一行**云端状态**（四态徽标 + rev + 「⤴ 提交 / ⤵ 拉取」） |
| AI 面板 | 无 | 会话条目的四态徽标；**变更单**（`pending`：几个、几个文件）与**锁**（`locks`：谁锁的）直接展示 |
| 批量按钮 | 无 | `☁ 提交`（只推 `local_only` + `broken`）/ `⬇ 拉取`（只拉 `cloud_only`），带 SSE 进度 |
| 逐条 | 无 | `⬆` / `⬇`（不一致时由人选方向；拉取先备份 `.bak`）；**`🔒 检出` / `🔓 解锁`**（多机同改保护，服务端现成）；**`🕘 历史`**（`history` + `revert`，零流量回滚） |
| 变更单（可选） | 无 | 要走「打开→改→提交变更单」两步时：`checkout` → 改 → `submit_pending`；不想要就维持现在的一步 `submit_stream` |

### 5.6 删除与改名

- `POST /api/ai/chat/drop` 删的是「某个 key 下的一份或全部会话」→ 该文件内容变了，**走提交**（新版本）。
- 若删到文件空（`drop_chat` 清空）→ 本地删文件，**云端也删**：
  `POST /api/depot/delete {paths: [/l_log/ai_chats/<hash>.json]}`（服务端一次到位：mark_delete + 提交）。
  不删的话它会永远以「☁ 仅云端」挂在状态里 —— l_agent_chat 踩过这个坑（现已接上 `delete_session_async`）。
- 改文件名 / 挪目录：用服务端 `move` / `mark_move`，**别用「删 + 建」**（会白丢历史）。
- 注意同步单位：**一份日志一个文件**（内含最多 10 份会话）。所以「删掉第 3 份会话」=
  文件变 → 提交新版本；**不是**删云端文件。

### 5.7 迁移

1. 把 `~/.lugwit/l_log/ai_chats/*.json` `move` 到各根下的 `ai_chats/`。
   - 归属判定：文件内的 `key` 反查「这是哪份日志」→ 归到该日志所在根；查不到就归到默认根。
   - `L_LOG_AI_CHAT_DIR` 被显式设置时**不要迁移**，按 §5.8 处理。
2. 首次提交（文件名不变 → 幂等；同名同内容不产生新内容）。
3. 可选：把一次本地扫描的结果经 `reconcile` 上报，让服务端把「本地已有的会话」记进变更单，
   再用 `submit_pending` 一次提交 —— 与「直接 `submit_stream`」等效，选一条即可。
4. **不要**指望 `POST /api/depot/import`：它只吃**网盘既有目录树**（见 §4.1 末行）。
5. 迁移是**一次性**的：`ai_chat_store.chat_dir()` 改为返回 `<根>/ai_chats`，之后新会话直接落在根里。

### 5.8 与环境变量覆盖的关系

`L_LOG_AI_CHAT_DIR` 覆盖时，store **不在根目录里** → 算不出 rel → **无法上云**。
处理：`chat_info` 里已有的 `env_override` 字段直接用来在界面提示
「会话目录被 `L_LOG_AI_CHAT_DIR` 覆盖，本次不上云」，并**不要**静默按旧路径提交。

## 6. 分步实施（每步单独可验）

### 阶段 1 已完成（2026-09-21）

- `ai_chat_store`：落点跟归属走 —— `chat_dir_for(key)` = `L_LOG_AI_CHAT_DIR` > `<根>/ai_chats` > 旧的全局目录；
  `chat_root(key)` 只认「**绑了库的额外根**」。`chat_info` 多了 `root` / `root_path` / `library` /
  **`syncable`** / **`sync_block_reason`** / `legacy_file` / `legacy_exists`（阶段 7 的界面直接用这几个）。
- `log_roots`：新增 `normalize_log_id()` 与 `root_of_log_id()`。
- 测试：`backend/tests/test_ai_chat_store.py`（7 例，临时目录 + 替掉归属解析，不碰真实目录）。
- 验收（真服务 `127.0.0.1:8003`）：真实界面 key
  `trace_log|LogList\local_update_logger\update_logger_trace|x` → `syncable: true`、
  `dir = D:\Temp\Log\ai_chats`、`library = /l_log`。

**实施中踩到的两件事（后面几步都要注意）**

1. **真实 logId 长这样**：`LogList\local_update_logger\update_logger_trace`（反斜杠 + `LogList` 前缀）——
   `LogList` 是 junction 摘除前的老目录名，`log_roots.LEGACY_DIR_ALIASES = {"LogList": "l_log"}` 把它
   承接给额外根 `@l_log/...`。**归属反查必须先归一化**（`normalize_log_id`），直接 `_split` 会全落到主根、
   于是永远判成「不上云」—— 方向正好反了。
2. **主根在包内**（`config.LOGS_DIR = backend/logs`）且**主根没有 library**：
   - 往里写会触发 dev_mod 热重载（每轮对话一次）—— 原 `ai_chat_store` docstring 的告诫正是这个；
   - 主根的日志本来就不参与版本库。
   → 所以「主根 / 没绑库的额外根 / 解析不出」一律退回旧全局目录，并在 `chat_info` 里说明原因。
3. **l_log 后端是平铺导入**（`app.py` 里是 `import log_roots`，不是包内相对导入）。
   在 `ai_chat_store` 里写 `from . import log_roots` 会在服务里 `ImportError`（离线以包方式跑才有幸通过）。
   现在两条都留了。改这个包的代码请照平铺风格写。
4. 热重载有**熔断**（连续改动会触发退避，`/__dev__/src_watch` 的 `breaker`）——
   改完 `.py` 若行为不像新版，先看那里，别急着杀进程。

### 阶段 2 已完成（2026-09-21）

`tools/migrate_ai_chats.py`（默认**干跑**，`--apply` 真搬）：

- 读每个会话文件里的 `key` → 反查所属根 → 只搬「绑了库的额外根」里的；
  其余留在原地并**说明原因**（主根在包内 / 没绑库 / 没有 key 字段）。
- 目标已有**同一份**内容 → 只清源（幂等）；目标已有**不同**内容 → 跳过并报告，**绝不覆盖**。
- 测试：`backend/tests/test_ai_chat_store.py::MigrationToolTest`（4 例，临时目录）。

**实际执行记录**

```
$ python tools/migrate_ai_chats.py --apply
结果：搬走 2，清重 0，留在原地 0，失败 0
```

| 文件 | 大小 | SHA256(前 16) 迁前 = 迁后 | 校验 |
|------|------|--------------------------|------|
| `3a0cc23dcb977ddf.json` | 537B | `4EA066A5BD28F7AA` | ✓ |
| `c0fe984769b093c0.json` | 24112B | `EF8BCC9C8F84D73F` | ✓ |

- 旧目录 `~/.lugwit/l_log/ai_chats/` → **0 份**；新位置 `D:\Temp\Log\ai_chats\` 两份都在。
- `/api/ai/chat/info` → `syncable: true` / `exists: true` / `dir = D:\Temp\Log\ai_chats`。
- `/api/ai/chat/load` → `chat` 读得回（1 份会话、**3 份会话**各一），`history` 2 条。
- 再跑一次干跑 → 「旧目录里没有会话文件，无需迁移。」（**幂等**）

### 阶段 3 已完成（2026-09-21）

`workspace_status.py` 加 scope（`SCOPES = {"logs": …, "chats": {"exts": {".json"}, "subdir": "ai_chats/"}}`）：

- `local_files(root, scope)` / `depot_files(root, lib, scope)` 按 scope 取子集
  （`_in_scope` / `_other_subdirs` 从 `SCOPES` 推导，**不硬编码** `ai_chats/`）；
  比对体抽成 `_scope_status(root, lib, scope, deep)`。
- **顶层保持「日志」那一组不变**（`files`/`counts`/`archive`/`by_rel`…）→ 老前端一行不用改；
  会话另起 `chats` 一组（同样带 `archive`/`counts`/`files`/`by_rel` + `dir`）。
- 缓存仍是 `(root, lib, deep)` 一条，两组一起算/一起失效；端点 `/api/workspace-status`
  直接返回，无需改动。
- 测试：`backend/tests/test_workspace_status_scopes.py`（7 例，库端全部打桩）。
  其中最要紧的一条是**不变式**：会话文件增删改后，日志那组的 `files` 与 `counts` 一字不变。

**实测**（`GET /api/workspace-status?root=l_log`）：

| 组 | 结果 |
|----|------|
| `logs` | 420 个文件（413 local / 4 cloud / 3 synced），**不含任何 `ai_chats/*`** |
| `chats` | 正是迁移过来的 2 份（`ai_chats/3a0cc23d….json`、`ai_chats/c0fe9847….json`），均 `local`（云上还没有）；`dir = D:\Temp\Log\ai_chats` |

### 阶段 4 起（待做）

| 阶段 | 内容 | 验收 |
|------|------|------|
| 1 | ✔ 已完成（见上） | — |
| 2 | ✔ 已完成（见上） | — |
| 3 | ✔ 已完成（见上） | — |
| 4 | 会话 scope 的提交 / 拉取（拉取后调 `sync_done` 盖章） | 提交后 `list?dir=/l_log/ai_chats` 能看到 `<hash>.json`；拉回内容一致 |
| 5 | 台账**只记指纹** `{rel: sha}`；rev 用服务端 `have` / `sync_plan`；补 `UNKNOWN` / `BROKEN` 两个态 | 本地改一份会话 → ✎；只改云端 → ✎；两边没动 → ✓ 且**不下载**；blob 缺失 → ⚠ |
| 6 | l_agent_chat 的 `persist` 开关 + `agent_proxy` 传 `false` | 在 l_log 面板问一轮 → `/l_agent_chat` 的会话数**不增加**；l_log 的会话文件更新 |
| 7 | 界面：四态徽标 + 变更单/锁展示 + 批量（SSE 进度）+ 逐条 ⬆⬇ / 🔒检出 / 🕘历史；`drop` 清空时删云端 | 逐条点过；`drop` 后云端那份消失、状态不再挂「仅云端」；回滚到旧版成功 |
| 8 | 文档 + `README.md` 索引登记 | — |

## 7. 验证清单

```bat
rem 会话是否进了库
curl "http://127.0.0.1:1028/api/depot/list?ws=l_log&dir=/l_log/ai_chats"

rem 单份会话的云上版本 / 历史
curl "http://127.0.0.1:1028/api/depot/history?ws=l_log&path=/l_log/ai_chats/<hash>.json"
```

- 本地改一份会话 → 状态 ✎ → 逐条 `⬆` → 云端 rev +1、状态 ✓。
- 删一份会话到清空 → 本地文件消失 + 云端 `delete` → 状态不再出现该项。
- 在 l_log 面板问一轮 → `l_agent_chat` 的 `sessions/` 目录**不新增文件**（`persist:false` 生效）。
- 断网 / 服务不可达 → 状态给结构化原因、**一个都不推**（不盲推）。

## 8. 风险与未决项

1. **落点**（已给推荐）：`<根>/ai_chats/`（库内容 = 根目录树，rel 算得出）
   vs 原地 `~/.lugwit/l_log/ai_chats/` + 给 l_log 加「一个库多个本地根」的抽象（不搬文件，但抽象更绕）。
2. **实现深度**：
   - **薄**：只做「会话 scope 的提交/拉取/两组状态」，用 l_log 现有 `workspace_status` 框架。
   - **厚**：把 l_agent_chat 那套（`depot_status.from_fingerprints` / 台账 / `push_stream`+`pull_stream`（SSE）/
     `push_one`+`pull_one` / `pending_actions`）**抽成共享模块**，两个包都用 —— 一次投入两边受益，
     但要动 l_log 现有的状态实现，且 l_agent_chat 侧要接受一次重构。
3. **会话在界面上怎么露**：跟日志混在一个列表（加一列/一组）还是单独一个「AI 对话」页签。
4. **多根归属**：一份会话的 key 反查不到任何根（例如独立窗口聊的）→ 归默认根，还是在界面上让用户选。
5. **`ai_chats/` 目录会不会被当成日志根的子目录展示**：`file_tree` 只列文本扩展名，理论上不会；
   要在阶段 3 实测确认（否则加进 `.` 前缀目录或走 maps `exclude`）。
6. **两步工作流要不要上**：`checkout` + `submit_pending` 完整但多一步；当前 l_log 是一步 `submit_stream`。
   建议：**先维持一步**，等真有「多人同时改同一份会话文件」的痛感再切。

## 附：关键代码位置

| 位置 | 说明 |
|------|------|
| `l_log/999.0/backend/ai_chat_store.py` | 会话存取（`chat_dir` / `save_chat` / `load_chat` / `drop_chat` / `chat_info`） |
| `l_log/999.0/backend/depot_logs.py` | depot 客户端（`library` / `ws_name` / `ensure_*` / `list_files` / `read_file` / `submit_file`） |
| `l_log/999.0/backend/workspace_status.py` | 现有四态（`EXTS` / `local_files` / `depot_files` / `workspace_status`） |
| `l_log/999.0/backend/agent_proxy.py` | 代理到 l_agent_chat（`_build_turns` / `history_turn_limit`） |
| `l_log/999.0/backend/app.py:1768-1792` | `/api/ai/chat/{save,load,info,drop}` |
| `l_log/999.0/backend/static/js/ai-agent-client.js:431,612,724,736,745` | 前端会话 I/O 与「对话存在哪」 |
| `lugwit_baidu_netdisk/999.0/src/lugwit_baidu_netdisk/web_server.py` | 服务端路由全表（§4.1 的来源） |
| `lugwit_baidu_netdisk/999.0/src/lugwit_baidu_netdisk/depot_service.py` | 服务端能力实现（`submit_files` / `checkout` / `mark_delete` / `delete_files` / `move_files` / `revert_to` / `workspace_sync_plan` / `reconcile_items` / `mark_synced` / `local_to_depot` / `depot_to_local` / `is_excluded` / `import_legacy_tree`） |
| `l_agent_chat/999.0/src/l_agent_chat/depot_sync.py` | **参考实现**：`cloud_manifest` / `local_uploads` / 台账 / `status_items` / `push_stream` / `pull_stream` / `push_one` / `pull_one` / `sync_done` / P4 操作 |
| `l_agent_chat/999.0/src/l_agent_chat/depot_status.py` | **参考实现**：四态 + `UNKNOWN` + `BROKEN`、`from_fingerprints` / `pending_actions` / `summarize` |
