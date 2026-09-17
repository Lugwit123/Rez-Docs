# Depot 库与工作区方案（对齐 P4 Client View）

> # ⚠️ 状态：**未实施 / 待评审（截至 2026-09-17）**
>
> 本文是工作区与 client view 的候选方案，不是当前 Depot 实现。当前已实现设计见[网盘版本库Depot设计.md](网盘版本库Depot设计.md)，后续计划见[网盘版本库Depot演进计划.md](网盘版本库Depot演进计划.md)。未完成评审、迁移与验收前，不得把本文字段、接口或数据模型当作已上线事实。

涉及包：

- `lugwit_baidu_netdisk/999.0` — 服务端：元数据库 + blob 仓 + Web 页面
- `lugwit_netdisk_client/999.0` — PC 客户端（PySide6 + QWebEngineView）

关联文档：

- 《网盘版本库Depot设计.md》— 已实现部分的实现说明
- 《Rez-Docs/Rez_pkg/lugwit_baidu_netdisk.md》— 使用文档（13 节）

## 1. 问题

现有 depot 已经实现了完整的两步工作流（`p4 edit/add/delete/move/revert/submit`）、
内容寻址去重、blob/dir 两种存储模式。但**用户侧的两个体验问题是真实的**：

1. **存取东西必须先决定"落到哪个库"**：逻辑路径首段即"库"，且每个库的物理形态由
   `depot_mode(root, mode)` 单独决定。想让它变成"一个库一个真实文件夹"，必须先去
   `PUT /api/depot/mode` 登记；不登记就是默认 `blob`——网盘上**看不见任何目录**，
   只有 md5 命名的 blob。
2. **没有"库 ↔ 本地文件夹"的映射**：调用方必须自己拼 depot 逻辑路径。上传时
   `?path=/xxx` 由调用方手工给出，本地文件在哪、和 depot 路径什么关系，**全靠人脑记**。

P4 的答案是一张 **client view**：

```text
Client: my-ws
Root:   D:\work
View:
    //depot/l_wchat/...   //my-ws/l_wchat/...
```

**depot 侧现在完全没有 client view**。本文给出补齐方案。

## 2. 现状核查（源码 + 实测）

### 2.1 数据库：7 张表，没有工作区维度

`depot_store.py` 的全部表与键：

| 表 | 主键 / 唯一键 | 关键列 | 有无工作区维度 |
|----|--------------|--------|---------------|
| `depot_blob` | `md5` | size, remote_path, created_at | — |
| `depot_changelist` | `id` + 部分唯一 `(owner) WHERE status='pending' AND description=''` | owner, description, status, created_at, submitted_at | ✗ |
| `depot_file_rev` | 唯一 `(path, rev)` | action, blob_md5, size, cl_id, owner | ✗ |
| `depot_have` | PK `(owner, path)` | rev | ✗ |
| `depot_lock` | PK `path` | owner, locked_at | ✗ |
| `depot_pending_file` | 唯一 `(owner, path)` | cl_id, action, blob_md5, size, src_path, base_rev, owner | ✗ |
| `depot_mode` | PK `root` | mode ∈ {blob, dir} | ✗（是"库"的雏形，但不含工作区） |

**结论：`owner`（JWT `sub`）就是工作区身份。** 7 张表里没有任何 `workspace` /
`client` / `local_root` / `device` 列。三处隐含假设印证"一 owner = 一份本地状态"：
`depot_have` PK、`depot_pending_file` 唯一键、默认 CL 的部分唯一索引，全部以 `owner` 为唯一维度。

### 2.2 物理布局：两种模式，新老布局并存

`depot_service.py` 的路径函数：

```text
version_depot_root(apps)  = {apps}/version_depot                       # 新统一根
blob_path(apps, md5)      = {apps}/version_depot/blob/<md5[:2]>/<md5>  # 新 blob
legacy_blob_path(...)     = {apps}/.depot/blob/<md5[:2]>/<md5>         # 旧 blob（只读兜底）
manifest_path(apps, cl)   = {apps}/.depot/manifest/<cl//1000:04d>/<cl:08d>.json
dir_mirror_root(apps)     = {apps}/version_depot/dir_mirror            # 新 dir 根
live_remote(apps, dpath)  = {apps}/version_depot/dir_mirror<dpath>     # 新 活文件
version_remote(..., rev)  = .../dir_mirror/<父>/.versions/<名>/vNNN/<名> # 新 历史
legacy_live_remote(...)   = {apps}<dpath>                              # 旧 活文件（只读兜底）
root_of_path(dpath)       = "/" + dpath 首段                           # "库"的定义
```

- **写入**：`ensure_dir_version()` 只走 `live_remote` / `version_remote`（新布局）
- **读取**：`_dir_mode_fs_id()` 按 `version_remote → live_remote → legacy_version_remote → legacy_live_remote` 四候选回退；`_blob_fs_id()` 按 `blob_path → legacy_blob_path` 回退

> 实测现状（2026-09-16 更新）：笔记/知识库统一在库 **`/notes`**（`mode=dir`），
> 活文件在 `/apps/Lugwit/version_depot/dir_mirror/notes/**`，历史在各自的 `.versions/<名>/`。
> 早期登记为 `dir` 的根 `/rez_pkg`（知识库名直接当库用）已废弃：其文件已 move 到
> `/notes/rez_pkg/`，旧库只剩空目录。即：现在**只有 `/notes` 一个 dir 根**，
> 知识库是它下面的**子路径**（见《lugwit_baidu_netdisk》§12 Depot 逻辑路径约定）。

### 2.3 接口：24 个端点，只有 1 个能表达"本地绝对路径"

`web_server.py` 的 `/api/depot/*` 完整清单（`_owner()` = JWT `sub`；库不可用 → 503）：

| 类别 | 端点 |
|------|------|
| 查询 | `status` `list` `tree` `history` `changes` `change/{cl_id}` `pending` `locks` `sync_plan` `download` |
| 一步提交 | `submit` `submit_stream` `revert` `delete` `move` |
| 两步工作流 | `mark_add_stream` `checkout` `mark_delete` `mark_move` `revert_pending` `submit_pending` `cl_description` |
| 锁 | `lock` `unlock` |
| 库模式 | `mode`(PUT) `modes` `migrate` |

**本地路径相关**：

| 端点 | 本地路径来源 |
|------|-------------|
| `POST /api/depot/submit` | `files[].local` —— **唯一**由调用方指定"服务端本地绝对路径"的入口 |
| `POST /api/depot/submit_stream` | body 字节流 → 服务端落 `state_dir()/.depot_<uuid>.bin` |
| `POST /api/depot/mark_add_stream` | 同上（流式分块写） |

`GET /api/depot/download` 返回 `StreamingResponse`，带 `X-Depot-Rev` 响应头。

### 2.4 前端：Workspace 骨架**已经写了**，但纯 localStorage、零后端

`web_depot.html` 里与工作区相关的现有代码：

| 部位 | DOM / 变量 | 行号 |
|------|-----------|------|
| 左栏标签 | `#tabDepot` / `#tabWorkspace` | 210 / 211 |
| 左栏工作区下拉 | `#wsSelect` | 217 |
| 左栏工作区树容器 | `#wsBody` | 223 |
| 中栏"工作区配置" tab | `#tabWsCfg` | 232 |
| 中栏配置面板 | `#bodyWorkspace`（`#wsListBox` 257 / `#wsNameInput` 263 / `#wsRootInput` 268） | 254–275 |
| 状态变量 | `wsList` / `wsCur` / `wsTree` / `wsOpen` | 343 |
| 持久化 | localStorage `depot_ws_v2`（兼容旧 `depot_ws_root`） | 344–358 |
| 函数 | `wsRoot()` `saveWs()` / `refreshWsSelect()` / `wsTreeNode()` `renderWorkspace()` `loadWorkspace()` / `showWorkspaceTab()` `openWsPanel()` `closeWsPanel()` / `renderWsPanel()` `saveWsForm()` `addWs()` `pickWsRoot()` | 355/356 · 943 · 969/1004/1018 · 1032/1038/1041 · 1044/1076/1092/1100 |

- 本地目录树靠 `window.lugwitBridge.treeDir(root, 3, 2000)` 现取（1024）
- 无工作区时 `showWorkspaceTab()` 自动跳中栏配置面板（1032）
- 教程文案仍写着"Workspace 标签暂未开放"（1662）

> **文档口径已过时**：《Rez-Docs/Rez_pkg/lugwit_baidu_netdisk.md》第 6 节说
> "Workspace 是后端没做的置灰功能"。**以代码为准**：前端骨架在，缺的是后端。

### 2.5 客户端：无状态下拉壳

`lugwit_netdisk_client` 的 Python 侧**不发任何 depot HTTP 请求**——业务全在服务端网页 JS 里。
它只通过 `bridge.py` 暴露 10 个本地能力 Slot：

```text
platform  pickFolder  pickFiles  revealInExplorer  fileMd5
fileInfo  listDir     treeDir    homeDir           openExternal
```

- `fileMd5` = **单文件**整文件 md5（仅作本地 md5 计算；秒传探测当前不可用）
- `listDir` / `treeDir` = 列本地目录（`treeDir` 注释明说"给前端 Workspace 标签用"）
- **没有**：工作区配置、本地根目录记忆、路径映射、目录级扫描/差异比对

### 2.6 缺口汇总

| 维度 | 现状 | 缺什么 |
|------|------|--------|
| 库的实体化 | 隐式 = 路径首段；模式在 `depot_mode` | 库无名字/描述/属主/默认形态等元信息 |
| 工作区身份 | `owner` | 需 `owner + 工作区名` 二维 |
| 本地根目录 | 无 | 需 `local_root` 字段（属于**执行机**，不是服务端） |
| 路径映射 | 无，全手工 | 需映射表 + 换算函数 |
| 本地差异比对 | 无（只有单文件 md5） | 需 reconcile（本地扫描 vs `have`） |
| 前端 | 骨架在，存 localStorage | 需后端接口 + 持久化 |
| 客户端 | 无状态 | 需承担本地扫描/上传（或退化为"本机即服务端"） |

## 3. 概念模型

### 3.1 库（Library）

一个**受版本管理的逻辑命名空间**，对应 depot 逻辑路径的首段。

```text
Library {
  root         "/l_wchat"        # 逻辑根，唯一
  name         "宝妈笔记"         # 展示名
  description  "..."
  owner        "admin01"          # 库管理员
  mode         "blob" | "dir"     # 物理形态（由 depot_mode 升格而来）
  status       "active" | "archived"
}
```

`mode` 决定网盘上长什么样：

| mode | 网盘形态 | 适合 |
|------|---------|------|
| `blob` | 只有 `version_depot/blob/<md5[:2]>/<md5>`，**没有目录结构** | 程序数据、频繁覆盖的单文件 |
| `dir` | `version_depot/dir_mirror/<库>/...` 活文件 + `.versions/<名>/vNNN/` | 需要人肉在网盘浏览/下载的文件树 |

### 3.2 工作区（Workspace）

**一个库的一份本地检出**，等价 P4 的 client。

```text
Workspace {
  id          7
  owner       "admin01"
  name        "admin01-l_wchat-win"     # owner 内唯一
  library     "/l_wchat"
  local_root  "C:/Users/me/.lugwit/l_WChat/data"
  host        执行机标识（可选，仅提示用，用于判断 local_root 属于哪台机器）
  last_sync_at
}
```

**关键语义：`local_root` 是"执行机上的路径"，不是服务端的路径。**
服务端只存字符串；真正读写这个目录的是**该机器上的执行者**（本机即服务端时的服务端进程，
或远程时的客户端 bridge / CLI）。这与 P4 一致（client root 是客户端的本地属性）。

### 3.3 映射（View）

P4 的 View 是若干行 `//depot/...  //client/...`。本方案分两级：

**一级（MVP，覆盖 90% 场景）**：工作区 = 一个库 + 一个 `local_root`，隐式映射

```text
/<library>/xxx   ←→   <local_root>/xxx
```

**二级（完整）**：显式映射表，支持子目录重挂与排除

```text
depot_path          local_path
/l_wchat/成长记录    .                       # 库的子目录 → local_root 下同名
/l_wchat/相册        photos                  # → local_root/photos
-/l_wchat/缓存       -                       # 前置 - 表示排除（不参与检出/提交）
```

### 3.4 与 P4 对照

| P4 | 本方案 | 现状 |
|----|--------|------|
| depot | 库 `depot_library` | 隐式（路径首段） |
| client | 工作区 `depot_workspace` | ✗ 用 `owner` 代替 |
| client Root | `depot_workspace.local_root` | ✗ |
| client View | `depot_workspace_map` | ✗ |
| `p4 have` | `depot_have` | ✅ 已有（但是 owner 维度） |
| `p4 opened/pending` | `depot_pending_file` + `depot_changelist(status=pending)` | ✅ 已有（owner 维度） |
| `p4 edit` + lock | `depot_lock` + `checkout()` | ✅ 已有（path 全局唯一锁） |
| `p4 reconcile` | 本地扫描 → 生成 pending | ✗ |
| `p4 sync` | `sync_plan` + 下载 + 更新 have | ⚠️ 只算"云端更新"，不管本地新增/删除 |
| `p4 submit` | `submit_pending()` | ✅ 已有 |
| `p4 revert` | `revert_pending()` | ✅ 已有 |

### 3.5 为什么 `owner` 不能继续当工作区

1. **一个人有多台机器/多个检出目录**是常态。现在两个检出会共写同一份 `depot_have`，
   互相把对方的 have 推进/覆盖，`sync_plan` 结果错乱。
2. `depot_pending_file` 唯一键是 `(owner, path)`：**同一人两个工作区对同一路径
   只能有一个待提交项**，无法并行编辑。
3. `depot_lock` 是 `path` 全局唯一锁，本身没问题（P4 也这样），但 `owner` 维度
   让"哪个工作区持锁"不可区分。
4. 前端 `#wsSelect` 已经允许多工作区（localStorage 里是数组），**后端对不上**。

## 4. 数据库变更

### 4.1 新表

```sql
-- 库：把 depot_mode 升格为带元信息的实体
CREATE TABLE IF NOT EXISTS depot_library (
  root         TEXT PRIMARY KEY,                    -- '/l_wchat'
  name         TEXT NOT NULL DEFAULT '',            -- 展示名
  description  TEXT NOT NULL DEFAULT '',
  owner        TEXT NOT NULL DEFAULT '',            -- 库管理员
  mode         TEXT NOT NULL DEFAULT 'blob',        -- 'blob' | 'dir'
  status       TEXT NOT NULL DEFAULT 'active',      -- 'active' | 'archived'
  created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- 工作区：P4 client
CREATE TABLE IF NOT EXISTS depot_workspace (
  id           BIGSERIAL PRIMARY KEY,
  owner        TEXT NOT NULL,                       -- JWT sub
  name         TEXT NOT NULL,                       -- owner 内唯一
  library      TEXT NOT NULL,                       -- 绑定的库 root
  local_root   TEXT NOT NULL DEFAULT '',            -- 执行机上的绝对路径
  host         TEXT NOT NULL DEFAULT '',            -- 执行机标识（提示/校验用）
  last_sync_at TIMESTAMPTZ,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX IF NOT EXISTS ux_depot_ws_name ON depot_workspace(owner, name);

-- 映射行（二级能力；一级留空表示隐式 <library>/... ↔ <local_root>/...）
CREATE TABLE IF NOT EXISTS depot_workspace_map (
  id           BIGSERIAL PRIMARY KEY,
  ws_id        BIGINT NOT NULL REFERENCES depot_workspace(id) ON DELETE CASCADE,
  depot_path   TEXT NOT NULL,                       -- '/l_wchat/成长记录'
  local_path   TEXT NOT NULL DEFAULT '',            -- 相对 local_root；'' = 同名
  exclude      BOOLEAN NOT NULL DEFAULT false       -- true = 不参与检出/提交
);
CREATE INDEX IF NOT EXISTS ix_depot_ws_map_ws ON depot_workspace_map(ws_id);
```

### 4.2 既有表加工作区维度

```sql
-- have：从 (owner,path) 升格为 (ws_id,path)，保留 owner 便于权限校验
ALTER TABLE depot_have ADD COLUMN IF NOT EXISTS ws_id BIGINT;
-- 回填：每个 owner 建默认工作区后 UPDATE
-- 新主键：唯一索引 (ws_id, path)；旧 PK (owner,path) 降级为普通索引
CREATE UNIQUE INDEX IF NOT EXISTS ux_depot_have_ws ON depot_have(ws_id, path);
CREATE INDEX IF NOT EXISTS ix_depot_have_owner ON depot_have(owner, path);

-- pending：唯一约束从 (owner,path) 改为 (ws_id,path)
ALTER TABLE depot_pending_file ADD COLUMN IF NOT EXISTS ws_id BIGINT;
DROP INDEX IF EXISTS ux_depot_pending_path;
CREATE UNIQUE INDEX IF NOT EXISTS ux_depot_pending_ws ON depot_pending_file(ws_id, path);

-- 默认 CL 的部分唯一索引：从 (owner) 改为 (ws_id)
ALTER TABLE depot_changelist ADD COLUMN IF NOT EXISTS ws_id BIGINT;
DROP INDEX IF EXISTS ux_depot_cl_default;
CREATE UNIQUE INDEX IF NOT EXISTS ux_depot_cl_default_ws
  ON depot_changelist(ws_id) WHERE status='pending' AND description='';

-- 锁：记录工作区（仍保持 path 全局唯一锁，语义不变）
ALTER TABLE depot_lock ADD COLUMN IF NOT EXISTS ws_id BIGINT;
```

> **保留 `owner` 列不删**：所有写路径仍用 `owner` 做属主校验（"这个 pending/锁是不是你的"），
> `ws_id` 只做分区。这样 `_owner()`、`DepotLocked` 的现有语义不变。

### 4.3 兼容与回填（一次性，幂等）

`connect()` 里的 `_SCHEMA_SQL` 追加一段回填：

1. `depot_mode` 的每行 → `INSERT INTO depot_library(root, mode) ON CONFLICT DO NOTHING`
2. 对所有出现过的 `owner`，各建一个默认工作区：
   ```
   name = '_default'，library = 该 owner 在 depot_file_rev 里最常见的首段（没有则 '/'），
   local_root = ''（未知，等用户填）
   ```
3. `UPDATE depot_have SET ws_id = (默认工作区 id) WHERE ws_id IS NULL`，pending / changelist / lock 同理
4. 之后 `ws_id` 若为 NULL（新写入漏传），读路径按 owner 的默认工作区兜底

**旧代码路径继续可用**：`mode_get/mode_set` 改为读写 `depot_library.mode`（或保留 `depot_mode` 作为视图），
`depot_migrate_dir.py`、`materialize_dir_root` 不受影响。

## 5. 接口变更

### 5.1 新增端点

| 方法 | 路径 | 请求 | 响应 | 用途 |
|------|------|------|------|------|
| GET | `/api/depot/library` | — | `{libraries:[{root,name,mode,status,file_count}]}` | 列库 |
| PUT | `/api/depot/library` | `{root,name?,description?,mode?}` | `{ok,root,mode}` | 建/改库（含 mode） |
| GET | `/api/depot/workspace` | — | `{workspaces:[{id,name,library,local_root,host,mapped_files}]}` | 列我的工作区 |
| POST | `/api/depot/workspace` | `{id?,name,library,local_root,host?,maps?[]}` | `{ok,id}` | 建/改工作区 |
| DELETE | `/api/depot/workspace/{id}` | — | `{ok}` | 删工作区（不动版本库） |
| GET | `/api/depot/workspace/{id}/status` | — | `{ws, library, plan_count, pending_count, mapped_files}` | 工作区概览 |
| GET | `/api/depot/workspace/{id}/sync_plan` | `dir=""` | `{plan:[{depot_path, local_path, action, rev, blob_md5, size}]}` | **带本地目标路径**的待同步清单 |
| POST | `/api/depot/workspace/{id}/reconcile` | `{items:[{local_path, depot_path, action, md5, size}]}` | `{files:[...]}` | 接收本地扫描结果 → 落到 pending |
| GET | `/api/depot/workspace/{id}/path` | `local=<相对/绝对>` | `{depot_path}` | 本地→depot 换算 |
| GET | `/api/depot/workspace/{id}/local_path` | `path=<depot>` | `{local_path}` | depot→本地 换算 |
| POST | `/api/depot/workspace/{id}/sync_done` | `{items:[{depot_path, rev}]}` | `{ok}` | 下载完回写 have |

### 5.2 既有端点兼容改法

所有以 `owner` 为维度的端点加**可选** `ws=<工作区名或 id>` 参数：

| 端点 | 改动 |
|------|------|
| `pending` `checkout` `mark_delete` `mark_move` `mark_add_stream` `revert_pending` `submit_pending` `cl_description` | `ws` 可选，缺省 → 该 owner 的 `_default` 工作区 |
| `sync_plan` | 保留旧签名；新增 `ws_id` 版本（5.1 已列） |
| `list` | item 增加 `ws_pending` / `ws_out_of_date`（该工作区视角的状态角标） |
| `download` | 不变（`X-Depot-Rev` 保留） |
| `submit` / `submit_stream` | 增加可选 `ws`，用于把"提交来源"记进 CL |

**旧调用方零改动**：不传 `ws` 就落到默认工作区。

### 5.3 关键契约

**`GET /api/depot/workspace/{id}/sync_plan`** —— 这是 `p4 sync -n`：

```jsonc
{
  "ws": {"id": 7, "name": "admin01-l_wchat-win", "local_root": "C:/.../data"},
  "plan": [
    {"depot_path": "/l_wchat/成长记录/growth_records.json",
     "local_path": "C:/.../data/成长记录/growth_records.json",
     "action": "edit", "rev": 3, "blob_md5": "f931...", "size": 659}
  ]
}
```

服务端**只算清单**，不碰本地文件。下载由执行机逐个调 `download?path=&rev=` 完成，
完成后调 `sync_done` 回写 `have`。

**`POST /api/depot/workspace/{id}/reconcile`** —— 这是 `p4 reconcile` 的**后半段**：

执行机先扫本地目录（`os.walk` + md5），与 `have` 比对得出：

```jsonc
{"items": [
  {"local_path": ".../新文件.json", "depot_path": "/l_wchat/新文件.json", "action": "add", "md5": "...", "size": 123},
  {"local_path": ".../growth_records.json", "depot_path": "/l_wchat/成长记录/growth_records.json", "action": "edit", "md5": "..."},
  {"local_path": "", "depot_path": "/l_wchat/旧文件.json", "action": "delete"}
]}
```

服务端只负责：映射校验 + 内容进 blob（`add/edit` 需带字节流，见 6.3）+ 写 pending。
**扫描必须在执行机做**——服务端读不到用户机器上的文件。

### 5.4 状态码

沿用 `_depot_err()`：`DepotLocked` → 409，`ValueError` → 400，库不可用 → 503。
新增：工作区不存在 → 404；`local_path` 越出 `local_root` → 400（见 §10）。

## 6. 本地代理：谁执行本地扫描

### 6.1 三种拓扑

| 拓扑 | 场景 | 谁读写 local_root | 内容怎么传 |
|------|------|------------------|-----------|
| **A. 本机即服务端** | 网盘服务跑在 `127.0.0.1:1027`（当前 L_WChat 环境） | 服务端进程自己 | `POST /api/depot/submit` 的 `files[].local` |
| **B. 客户端桥** | `lugwit_netdisk_client`（QWebEngine + bridge） | 客户端 Python（`bridge.py`） | `submit_stream` / `mark_add_stream`（字节流） |
| **C. CLI 代理** | 无 GUI 的机器 / 脚本 | 独立 CLI | 同 B |

拓扑 A 已经能跑（`submit` 收 `files[].local`），B/C 需要新增客户端侧逻辑。

### 6.2 reconcile 必须在本地侧

服务端只有 `have`（"本地应该是哪一版"），**不知道本地实际有什么**。
"本地多出来的文件（add）"和"本地删掉的文件（delete）"这两类差异，
只有摸得到文件系统的一侧才能算出来。所以：

- 服务端提供 `reconcile` 的**接收端**（校验 + 落 pending）
- 执行机提供 `reconcile` 的**计算端**（`os.walk` + md5 + 对比 `have`）

现有 `sync_plan()` 只覆盖"云端比 have 新"（`h.rev IS DISTINCT FROM l.rev`），
**不覆盖本地新增/删除** —— 这是必须补的那一半。

### 6.3 内容上传的两条路

| 场景 | 接口 |
|------|------|
| 执行机能直接把文件路径给服务端（拓扑 A） | `POST /api/depot/submit` `files[].local` |
| 执行机与服务端不同机（拓扑 B/C） | `mark_add_stream` / `submit_stream` 逐文件传字节 |

> 现状提醒：`mark_add` 在服务端签名里带 `access_token`，但**只上传 blob、不进版本**；
> 内容必须在 reconcile 阶段就上传，`submit_pending` 才能只在 DB 里落库。

## 7. 前端改动（`web_depot.html`）

**好消息：骨架大半已在**，改动是"接后端"而不是"重写"。

| 部位 | 现状 | 改动 |
|------|------|------|
| `wsList/wsCur`（343） | localStorage `depot_ws_v2` | 改为启动时 `GET /api/depot/workspace`；本地 localStorage 仅作缓存 |
| `saveWs()`（356） | 只写 localStorage | 改调 `POST /api/depot/workspace` |
| 初始化（344–358） | 读 localStorage | 加**一次性导入**：本地有、后端没有 → 提示"是否导入为后端工作区" |
| `renderWsPanel/saveWsForm/addWs()`（1044/1076/1092） | 纯本地 | 接后端；加"库"下拉（来自 `GET /api/depot/library`）与 `mode` 展示 |
| `loadWorkspace()`（1018） | `bridge.treeDir` 拉本地树 | 叠加 per-file 状态角标：`have` 落后/`pending`/`out_of_date`（复用 `statusIcons()` 705 风格） |
| `#tabWorkspace`（211） | 可用但无后端 | 真正开放；去掉教程文案"暂未开放"（1662） |
| 工具栏 | 已有 刷新/获取最新/提交/签出/添加/删除/撤销 | 「获取最新」接 `workspace/{id}/sync_plan` + 下载 + `sync_done`；「添加/签出/提交」带上 `ws` 参数 |
| 中栏 | `Files / Pending / Submitted` | `Files` 增加"工作区视角"角标列 |
| 新面板 | 无 | 工作区配置里加"映射行"编辑（depot_path / local_path / 排除） |
| 浏览器降级 | `HAS_BRIDGE`（337） | 无 bridge 时工作区只读（不能扫本地），给出明确提示 |

## 8. 架构修改清单（按文件）

| 文件 | 改动 | 原因 |
|------|------|------|
| `depot_store.py` | 加 3 张表 DDL + 回填 SQL；`have_*` / `pending_*` / `default_cl` / `submit*` / `sync_plan` / `lock` 增加 `ws_id` 维度（默认工作区兜底）；新增 workspace/library CRUD 方法 | 工作区是一等实体 |
| `depot_service.py` | 新增 `resolve_workspace()`、`map_local_to_depot()`、`map_depot_to_local()`、`workspace_sync_plan()`、`reconcile_items()`；`submit_files` 增加 `ws` 参数 | 映射换算 + 待同步清单 |
| `web_server.py` | 新增 §5.1 的 11 个端点；既有端点加可选 `ws`；`_ws_ctx()` 辅助（解析工作区+校验归属） | 对外契约 |
| `web_depot.html` | §7 的改动（接后端、状态角标、映射编辑、开放 tab、教程文案） | 前端骨架接后端 |
| `lugwit_netdisk_client/bridge.py` | 新增 `scanDir(root, ignore, max)`（`os.walk` + md5，返回可 reconcile 的 items）；`pickWsRoot` 已有 | reconcile 计算端 |
| `lugwit_netdisk_client/main.py` | 无强制改动（页面已在客户端内跑） | — |
| `tools/depot_migrate_dir.py` | 不变（仍按库 root 迁移）；`--root` 语义与库对齐 | 复用 |
| `tests/test_depot_workspace.py` | **新建**（见 §11） | 回归 |
| `Rez-Docs/网盘版本库Depot设计.md` | 补 §3 表清单（漏了 `depot_mode`）、补新老布局说明 | 文档与代码对齐 |
| `Rez-Docs/Rez_pkg/lugwit_baidu_netdisk.md` | 更新第 6 节"Workspace 未实现"的过时表述；新增库-工作区一节 | 文档与代码对齐 |

## 9. 迁移

### 9.1 `depot_mode` → `depot_library`

模式是库的属性，语义完全重合。保留 `depot_mode` 表一段时间做只读兜底，
写入统一走 `depot_library`；`mode_get/mode_set` 内部改指向新表。

### 9.2 旧数据 → 默认工作区

见 §4.3。**不搬任何 blob、不改任何 `depot_file_rev`** —— 只加 `ws_id` 维度，
历史版本一律不动。这是本次迁移的安全底线。

### 9.3 前端已有工作区

`depot_ws_v2` 里可能已有用户手填的 `name/local_root`。启动时：
后端有同名 → 用后端；后端没有 → 提示"导入"（用户确认后 `POST /api/depot/workspace`），
**不静默覆盖**。

### 9.4 L_WChat 成长记录怎么落

以现有实现为例，切到工作区模型后：

```text
库：      /l_wchat            mode = blob（保持零风险：不产生第二份活文件）
工作区：  name=admin01-l_wchat-win
          local_root = C:/Users/<u>/.lugwit/l_WChat/data
映射：    隐式  /l_wchat/... ←→ <local_root>/...
```

客户端只见"工作区"，`/api/depot/workspace/7/sync_plan` 直接算出
`data/成长记录/growth_records.json` 该下载到哪 —— **调用方不用再手写 depot 路径**。
这是本方案对 L_WChat 最直接的价值。

## 10. 安全

1. **映射路径校验**：`local_path` 必须是 `local_root` 的相对路径，且
   `normpath(join(local_root, local_path))` 必须仍以 `normpath(local_root)` 为前缀。
   否则 400。禁止 `..`、绝对路径、盘符切换。
2. **`local_root` 的信任边界**：它是执行机自己上报的字符串。服务端**不得**用它去读自己的盘
   （除非拓扑 A 明确声明"我就是执行机"）。拓扑 A 下 `submit` 的 `files[].local`
   也要校验落在该工作区 `local_root` 内，防止越权读服务端任意文件。
3. **越权**：所有 workspace 端点先校验 `depot_workspace.owner == _owner(user)`，否则 404
   （不用 403，避免泄露工作区是否存在）。
4. **删除语义**：删工作区只删 `depot_workspace(+map)`，**不删版本、不删 blob、不删 have 之外的任何东西**。
5. `depot_lock` 保持 path 全局唯一——跨工作区互斥，符合 P4 语义。

## 11. 测试计划

### 11.1 新建 `tests/test_depot_workspace.py`

照抄现有模板（`test_depot_api.py` / `test_depot_pending.py`）：

- 纯标准库 `urllib` 直连 HTTP，`check(name, cond, detail)` + `section(t)` + 末行 `RESULT PASS|FAIL`
- 独立时间戳前缀 `/_autotest_ws/<MMDD_HHMMSS>`，可重复跑
- CLI `--base` / `--keep`；`cleanup()` 走 `revert_pending` + `mark_delete` + `submit_pending`
- `if __name__ == "__main__": raise SystemExit(main())`

### 11.2 用例清单

| 函数 | 断言 |
|------|------|
| `t_library_crud` | 建库 / 列库 / 改 mode / 重复建幂等 |
| `t_ws_crud` | 建工作区（owner 内重名 409/400）/ 列 / 改 / 删 |
| `t_ws_isolation` | **同一 owner 两个工作区**：A 的 pending 不出现在 B；A 的 have 不被 B 推进 |
| `t_ws_default_compat` | 不传 `ws` 的旧调用落到 `_default` 工作区，行为与改造前一致 |
| `t_path_map` | `local→depot` / `depot→local` 双向换算；`..`、绝对路径、越出 root 一律 400 |
| `t_map_lines` | 子目录重挂 + 排除行生效（排除项不出现在 sync_plan/reconcile） |
| `t_sync_plan_local` | plan 里带正确的 `local_path`；`sync_done` 后 have 推进、plan 清空 |
| `t_reconcile_add_edit_delete` | 三类差异都能生成 pending；`add` 内容进 blob 但**不产生版本** |
| `t_migrate_backfill` | 造旧格式（owner 无 ws_id）数据 → 连库后自动回填默认工作区，历史 rev 不变 |
| `t_lock_across_ws` | 工作区 A 持锁 → 工作区 B `checkout` 同路径 409 |

### 11.3 回归

- `test_depot_api.py`、`test_depot_pending.py` 必须**全绿且不改一行**（兼容性硬指标）
- `purge_autotest.py` 增加 `/_autotest_ws` 前缀

## 12. 分阶段实施

| 阶段 | 内容 | 交付标准 |
|------|------|---------|
| **P0 库实体化** | `depot_library` 表 + `library` 接口 + `depot_mode` 兼容读写 | `modes` 与 `library` 返回一致；旧脚本不用改 |
| **P1 工作区（无本地扫描）** | `depot_workspace` / `map` 表 + `ws_id` 回填 + workspace CRUD + `sync_plan`（带 local_path）+ 既有端点加 `ws` | 同一 owner 多工作区互不干扰；`test_depot_workspace.py` 前 7 项绿 |
| **P2 本地闭环** | `reconcile` 接收端 + `sync_done` + 客户端 `bridge.scanDir` + 前端接后端（状态角标、获取最新、开放 tab） | 端到端走通"扫本地 → 待提交 → 提交 → 另建设备 sync 下来" |
| **P3 L_WChat 落地** | 成长记录切到工作区模型（库 `/l_wchat` blob 模式） | 调用方不再手写 depot 路径 |

每阶段独立可发布，P0/P1 纯后端，不影响现有前端与脚本。

## 13. 明确不做

| 不做 | 理由 |
|------|------|
| P4 完整 View 通配符语法（`//depot/...` 多行 include/exclude 全语义） | 先做"单库 + 单根 + 子目录/排除"，覆盖绝大多数场景；语法爆炸收益低 |
| 一个工作区跨多个库 | 映射与权限都会复杂化；先限制单库，需要时建两个工作区 |
| 实时双向自动同步 | 与现有 `baidu_push.propagate_delete=false` 的保守取向一致；且"本地删 → 云端删"风险高 |
| 服务端代替客户端读本地盘（除显式声明拓扑 A） | `local_root` 是执行机属性，服务端越权读盘是安全问题 |
| 在线编辑冲突自动合并 | 沿用 `depot_lock` 独占（P4 语义）+ `base_rev` 过期检测 |
| 重写现有两步工作流 | 它已经对了（`test_depot_pending.py` 10 项覆盖）；本次只加维度，不改语义 |

## 14. 待决策

1. **`depot_mode` 是改名还是保留双写**：改名干净但要动三条读路径；双写安全但两处真相。
   倾向：**保留 `depot_mode` 作只读兜底 + `depot_library` 为唯一写入口**。
2. **默认工作区的 `local_root` 取什么**：留空（让用户填）还是按 `host + owner` 猜一个？
   倾向：**留空**，前端提示补填。
3. **`ws` 参数的取值形态**：名字还是 id？名字可读但改名会让旧脚本失效。
   倾向：**两者都收**（数字按 id，其余按 name）。
4. **P2 的扫描端放哪**：`lugwit_netdisk_client/bridge.py`（有 QWebEngine 环境）
   还是独立 CLI（可无 GUI 跑）？倾向：**先 bridge，再抽 CLI**，两者共用同一份扫描实现。
