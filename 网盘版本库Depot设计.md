# 网盘版本库（Depot）设计与实现

状态：**已实现设计主文档，截至 2026-09-17**。实施计划唯一台账见[网盘版本库Depot演进计划.md](网盘版本库Depot演进计划.md)；百度接口 md5 实测唯一事实源见[Rez_pkg/百度云接口元数据实测.md](Rez_pkg/百度云接口元数据实测.md)。（2026-09-17 增补：鉴权闸门与浏览器/客户端取内容链路，见 §6.1 与 §8.1。）

涉及包：

- `lugwit_baidu_netdisk/999.0` — blob 仓 + 元数据 + Web 页面
- `lugwit_netdisk_client/999.0` — PC 客户端（PySide6 + QWebEngineView）

## 1. 背景

跨地区传文件，公网带宽太贵。改用百度网盘做中转：
上传方推到网盘，下载方从网盘拉，走网盘自己的 CDN，不吃自建带宽。

在此之上加一层 **简化版 P4 版本管理**：改名/移动/回滚不产生任何流量。

## 2. 核心设计：内容寻址 blob 仓 + 数据库唯一权威

### 2.1 网盘只存 blob，只追加

blob 按**库根隔离**：同一 md5 在不同库分别有实体和元数据行；库内重复才直接复用，跨库可借百度秒传复制而不共享物理文件。

```text
/apps/Lugwit/.depot/blob/<md5[:2]>/<md5>
/apps/Lugwit/.depot/manifest/<cl//1000:04d>/<cl:08d>.json
```

- blob 用 md5 命名 → 同内容全库只存一份
- 网盘里**没有目录结构**，路径语义全在数据库里
- 因此：重命名 = 改一行数据库；回滚 = 新版本指向旧 blob。**零字节流量**

### 2.2 manifest 兜底

每次提交额外写一个几 KB 的 JSON（CL 号、提交人、说明、文件列表 + md5）。
数据库炸了，按 CL 顺序重放 manifest 即可完整重建元数据。

### 2.3 为什么不选另外两种

| 方案 | 否决理由 |
|------|----------|
| 网盘按真实目录结构存 | 改名/移动要真搬文件，跨地区重传 |
| 全量元数据只放网盘 JSON | 并发提交无法加锁，列目录要拉一堆小文件 |

## 3. 数据库表（落在 `chatroom` 库）

`depot_store.py`，asyncpg 裸连接池（URL 处理与 `credential_service.py` 同款；
旧的 `account_service.py` 已随 P4 泛化删除）。

| 表 | 作用 | P4 对应 |
|----|------|---------|
| `depot_blob(lib_root, md5 PK)`, `size`, `remote_path` | 库内内容 → 网盘路径；主键为 `(lib_root, md5)`，保证按库隔离 | — |
| `depot_changelist(id, owner, description, status)` | 变更列表，`status`=pending/submitted | CL |
| `depot_pending_file(cl_id, path, action, blob_md5, src_path, base_rev)` | 待提交区条目 | opened files |
| `depot_file_rev(path, rev, action, blob_md5, cl_id, owner)` | 文件某个版本 | revision |
| `depot_have(owner, path, rev) PK(owner,path)` | 某人本地是哪个版本 | have 表 |
| `depot_lock(path PK, owner)` | 独占签出 | `p4 edit` |

唯一索引：

- `ux_depot_file_rev ON depot_file_rev(path, rev)`
- `ux_depot_pending_path ON depot_pending_file(owner, path)` — 一人对一路径只能有一个待提交动作
- `ux_depot_cl_default ON depot_changelist(owner) WHERE status='pending' AND description=''` — 每人一个默认待提交列表

老库升级靠 `ALTER TABLE ... ADD COLUMN IF NOT EXISTS`，不用手动迁移。

### 3.1 两步工作流（和 P4 一致）

```
p4 edit   → checkout()        加锁 + 进 depot_pending_file
p4 add    → mark_add()        内容先进 blob 仓，条目进待提交区（还没有版本号）
p4 delete → mark_delete()     标记待删
p4 move   → mark_move()       待提交区记一条（path=目标, src_path=源）
p4 revert → revert_pending()  从待提交区移除 + 解锁，不产生任何版本
p4 submit → submit_pending()  整批原子落库
```

关键点：**submit 之前 `depot_file_rev` 里什么都不写**。所以撤销是真撤销，不留垃圾版本。
blob 已上传但无人引用 → 孤儿，靠 GC 回收（尚未实现）。

`submit_pending()` 一个事务内：

1. `SELECT ... FOR UPDATE` 锁 CL 行，校验 status / owner
2. 展开 pending 条目 —— **`move` 一条展开成两条**（目标 `move` + 源 `delete`）
3. 逐路径校验 `depot_lock`，被别人锁住 → `DepotLocked` → HTTP 409
4. CL 改 `status='submitted'` + 写 `submitted_at`
5. 每条算 `max(rev)+1` 写 `depot_file_rev`，推进 `depot_have`，清自己的锁
6. 清空该 CL 的 `depot_pending_file`

### 3.2 一步直提仍然保留

老的 `submit_files()` 走 `store.submit()`，不经待提交区，一次调用直接产生版本。
`/api/depot/submit_stream` 和自动化脚本还在用。两条路并存，互不干扰。

`DepotLocked` 带 `.path` / `.owner`，Web 层映射成 HTTP 409。

### 3.3 其他查询

- `list_dir(prefix)`：`SELECT DISTINCT ON (path) ... ORDER BY path, rev DESC`，
  折叠子路径成目录，跳过 `action='delete'`
- `sub_paths(prefix)`：递归取存活文件，整目录移动用
- `sync_plan(owner, prefix)`：CTE `latest` LEFT JOIN `depot_have`，
  条件 `h.rev IS DISTINCT FROM l.rev` → 待同步清单
- `pending_status(owner, prefix)`：给文件列表标状态图标用

## 4. 上传层

`baidu_netdisk_api.py`。

### 4.1 秒传（之前是错的）

`precreate` 返回 `return_type == 2` 原本判定为**秒传命中，零上行**；判定逻辑保留，
但**本账号/应用实测命中不了**（见 `Rez_pkg/lugwit_baidu_netdisk.md` §14.4，2026-09-16 实测），
上传会走真分片。
旧代码把返回的空 `block_list` 当成"所有分片都要传"，白传一遍。现已在
`upload_file` 和 `upload_file_stream` 两处都拦截。

### 4.2 流式分片（10GB 文件不进内存）

```python
scan_file(path)  -> (总字节, 整文件 md5, 各分片 md5)   # 单次遍历
read_chunk(path, i)                                    # 按需 seek
```

`precreate` 返回的 `block_list` = **还缺哪些分片** → 天然断点续传。

## 5. 提交流程

`depot_service.submit_files()`：

```text
scan_file 得 md5
  → store.blob_get(md5) 命中？ 零网络，直接复用      (dedup)
  → 否则 upload_file_stream
       → precreate return_type==2 ?  秒传，零上行     (rapid；当前不可达，勿据此估算流量/空间收益)
       → 否则只传缺失分片                             (uploaded)
  → store.submit() 事务写元数据
  → write_manifest()   失败只记日志，不影响提交
```

返回统计：`{"uploaded", "rapid", "dedup", "bytes"}`。

其他操作：

- `revert_to(path, rev)` — 新建一版指向旧 blob，零流量
- `delete_files(paths)` — 写 `action='delete'`，blob 保留（历史仍可取）

## 6. Web 接口

`web_server.py`，页面 `/depot`，API 前缀 `/api/depot/`：

```text
status  library(?PUT)  list  list_recursive  tree  sessions  migrate
history  changes  change/{cl_id}  pending  locks  sync_plan  download(Range/ETag)
submit  submit_stream  revert  delete  move  import  edit_text
checkout  mark_delete  mark_move  mark_add_stream  revert_pending  submit_pending  cl_description
lock  unlock
workspace(GET/POST)  workspace/select  workspace/{id}(GET/DELETE)  workspace/{id}/owner
workspace/{id}/maps(PUT)  workspace/{id}/have  workspace/{id}/path  workspace/{id}/local_path
reconcile  sync_done
```

完整参数 / 返回 / 状态码见《Rez_pkg/lugwit_baidu_netdisk.md》§5（5.1 查询、5.2 待提交工作流、
5.3 一步接口、5.4 状态码、5.5 路径规则、5.6 工作区与库）。

约定：

- 身份取 JWT `sub` 字段（`_owner()`），缺失 → 401
- `_depot_err()` 统一映射：`DepotLocked`→409，`ValueError`→400，其余→500
- `_depot_ctx()` 一次拿齐 `(store, access_token, apps_root)`，库不可用 → 503
- `/api/depot/download` 解析 blob 的 `fs_id` → `dlink` → `StreamingResponse`，
  带 `X-Depot-Rev` 响应头

页面 `web_depot.html`（现为**三栏 + 底栏**：左树 250 / 中栏 / 右栏、底栏 170，中栏 / 右栏 / 底栏的
标签可拖动换位，布局细节见《Rez_pkg/lugwit_baidu_netdisk.md》§6）：标题栏抄
`l_notepad_server/templates/web_edit.html` 的 `.topbar` / `.topbar-meta` 样式。

### 6.1 鉴权闸门与内容获取链路（2026-09-17）

**每个 depot HTTP 请求都过 `lugwit_baidu_netdisk/gate.require_lugwit_token`** ——
**本地验 JWT**，不依赖认证服务在线；token 认 **cookie `lugwit_token`** 或环境变量 `LUGWIT_ACCESS_TOKEN`。

⚠️ 服务自带的「本机自动授权」（`web_server._auto_local_token` → `POST {auth}/api/v1/auth/auto`）
**在本机实测静默失败**（访问日志只有 401、无任何异常记录）。因此**三处调用方改为自带凭据**，
并在 401 时换新 token 重试一次：

| # | 调用方 | 链路 |
|---|--------|------|
| 1 | `l_notepad_server/depot_map.py` 的 `http()` | note server → 1028 |
| 2 | `l_tray/src/l_tray/depot_bridge.py` | 网页 → 托盘 19527 → 1028 |
| 3 | `lugwit_netdisk_client`（本地桥） | 主窗口登录后 `bridge.setToken()`；另从持久化 WebEngine profile 的 cookie 抓 `lugwit_token` |

**2026-09-20 更新（P0/P6 落地后）**：
- `/api/v1/auth/auto` **默认关闭**；netdisk 的 `_auto_local_token` 与回环兜底分支**已删** ——
  登录态只认 cookie / `LUGWIT_ACCESS_TOKEN`；
- 三处调用方改为 **env / 登录**：托盘走「托盘会话 token（浏览器授权或账号密码，refresh 存 DPAPI、可自动续期）」，
  note server 用 `require_token()`（没有就直接报错，不再发匿名请求）；
- 全仓已无 `/api/v1/auth/auto` 调用。
- token 获取顺序（现状）：托盘会话 token → env `LUGWIT_ACCESS_TOKEN` → `LUGWIT_USER`/`LUGWIT_PASSWORD` 登录。

**授权判定（P6）**：depot 的**读接口**（`download`/`list`/`history`）与**写接口**（`submit`/`submit_stream`/
`delete`/`move`/`revert`/`checkout`/`edit_text`/`import`/`mark_*`/`*_pending`）都先过
`gate.require_perm()`：管理员本地放行 → 「路径首段 = owner」快判 → 跨用户问 auth `/authz/check`
→ auth 不可达只认自己的路径；越权 **403**；ACL 支持**目录继承**（库根授权覆盖子路径）。

**灾备工具（T4 补齐）**：`tools/depot_manifest_verify.py`（清单 ↔ DB 四类漂移，只读）、
`tools/depot_manifest_replay.py`（默认 dry-run 出 SQL，`--apply` 单事务重建
`depot_changelist`/`depot_file_rev`/`depot_blob` + `setval`）。

⚠️ **两条已知的既有数据问题**（2026-09-20 演练发现，未修）：
1. `depot_blob` 缺 **71 条登记**（manifest 引用到的 (库, md5) 没有对应行）→ 读这些历史版本会落到兜底查找；
2. `depot_blob.remote_path` 存的是 `/apps/Lugwit/version_depot/version_depot/blob/…`（**多一层 `version_depot`**），
   而网盘真值是 `/apps/Lugwit/version_depot/blob/…` —— 实测 DB 那条 404、真值 200。


**内容怎么取（浏览器不再直连 depot）**：知识库页归档（版本库）内容读取优先级
**① 客户端本地桥 `window.lugwitBridge.depotDownload` → ② 托盘中转（`depot_download` 动作，经 19527）→
③ 回退 `/baidu/api/depot/download`**；无论走哪条都要带上 lugwit 登录态（桥/托盘内部自动补 token）。
托盘状态浮层新增一行「版本库读取：客户端本地桥 / 经托盘中转（基址） / 回退 `/baidu` 直连」。
归档逻辑路径必须取映射的 `base_path`（形如 `/notes/rez_pkg/xxx.md`）——
旧写法 `/<kbName>/<rel>`（缺 `/notes` 前缀）会 404「版本不存在」，本次已修。

**元数据 vs 内容（选错方向会白查一小时）**：

- depot **元数据在 PostgreSQL** → 列表类接口（`/api/depot/list`、`/api/kb/{kb}/depot/list`）**离线可用（200）**
- depot **文件内容在百度网盘**（`pan.baidu.com`）→ 机器连不上外网时取内容报 **500 / 经 nginx 502**，**与代码无关**
- 实测：`curl https://pan.baidu.com` 返回 `000` 时，列表 200、下载 500

### 6.2 工作区数据源：**页面直连 depot 服务**（2026-09-26 取代原「唯一走托盘」方案）

**现状（2026-09-26）**：`web_depot.html` 的工作区列表与增删改**直连 depot 服务的
`/api/depot/workspace*`**（带页面自己的统一登录 cookie），不再绕托盘 19527：

| 页面调用 | 等价托盘动作（保留但本页不再用） |
|---|---|
| `GET /api/depot/workspace?all=1` | `depot_workspace_list` |
| `POST /api/depot/workspace` | `depot_workspace_save` |
| `DELETE /api/depot/workspace/{id}`（批量逐个删，收集 `deleted`/`failed`） | `depot_workspace_delete` |
| `POST /api/depot/workspace/select` | `depot_workspace_select` |
| `POST /api/depot/workspace/{id}/owner` | `depot_workspace_set_owner` |

依据：托盘那几个动作（`l_tray/depot_bridge.py`）本来就只是**透传同样的 URL**
（`_http(token_override=...)`），页面自己带 cookie 能调；而托盘依赖还有个副作用 ——
托盘没起/晚起，页面就看不到自己的工作区。改直连后 401 也会由 `apiReq` 统一跳
`/login?next=<本页>`，不会静默失败。

**托盘现在只负责本机能力**：本地目录树（`depot_local_tree` / `depot_local_version`）、
在资源管理器中打开 / 定位本地文件（`depot_local_open`）、新建目录 / 新建文件 / 删除到回收站
（`depot_local_mkdir` / `depot_local_newfile` / `depot_local_delete`）、选本地路径 —— 以上都只作用于
**工作区 `local_root` 之内**（删除额外禁止删根目录本身，走回收站可还原）。工作区树与工作区卡片的
右键菜单用它们；浏览器模式下工作区标签页的本地树由托盘读，并用变更序号轮询自动刷新。
（浏览器模式要求托盘已登录：页面 HttpOnly cookie 读不到 token 时托盘回落自己的会话 token，没登录就拿不到工作区列表。）

<details>
<summary>历史方案（2026-09-17，已被上面取代，留档看取舍）</summary>

**结论：页面（`web_depot.html`）的工作区列表与增删改，唯一数据源是托盘服务
（`l_tray` ExecServer，`POST http://127.0.0.1:19527/run`）——不落 `localStorage`、
也不直连 `/api/depot/workspace`，且**不做任何回退**（托盘不在就明确报错）。**

之前的毛病：`wsList` 初始化先用 `localStorage["depot_ws_v2"]` 播种（还兼容更老的
`depot_ws_root`），再拉 `/api/depot/workspace` 覆盖。客户端里登录是异步的（标题栏登录
成功后才注入 `lugwit_token`），初始化那次拉取可能 401，而调用点 `.catch(function () {})`
把错误吞了 → 列表停在 localStorage 旧值，于是**浏览器与客户端工作区列表不一致**
（客户端少/为空），且工作区状态跨机器根本不成立——"哪个库对应哪个本地目录"是
**那台机器的属性**，本就该由本机代理（托盘）回答。

托盘侧新增三个网页白名单动作（`l_tray/depot_bridge.py`）：

| 动作 | 透传到 depot 服务 |
|------|------------------|
| `depot_workspace_list` | `GET /api/depot/workspace`（回 `owner` / `workspaces` / `libraries`） |
| `depot_workspace_save` | `POST /api/depot/workspace`（JSON：`name` / `library` / `local_root` / `host` / 可选 `id`） |
| `depot_workspace_delete` | `DELETE /api/depot/workspace/{ws_id}` |

</details>

调用约定（仍适用于仍在用托盘的调用方）：

- 请求体 `{"action": "<名>", "kwargs": {...}}`；成功信封 `{"ok":true,"result":"<repr>","data":<结构化结果>}`，
  **页面取 `data`**（`result` 只是 repr 字符串）；失败 `{"ok":false,"error":"<traceback>"}`。
- 跨源：执行机上的 `127.0.0.1:8080` / 生产域名等已在 ExecServer 的 CORS 白名单内
  （`DEFAULT_WEB_ORIGINS`，可用 `L_TRAY_EXEC_ORIGINS` 追加）；浏览器只能调 `web_actions`。
- 托盘的 depot 基址：`L_TRAY_DEPOT_URL` > `L_DEPOT_SERVICE_URL` > `http://127.0.0.1:1028`；
  token（2026-09-20 起）：**托盘会话 token**（菜单「登录」→ 浏览器授权/账号密码，refresh 存 DPAPI 可自动续期）
  > env `LUGWIT_ACCESS_TOKEN` > `LUGWIT_USER`/`LUGWIT_PASSWORD` 登录（401 换新重试）。
  `/api/v1/auth/auto` 已默认关、不再是取 token 途径（见 `Rez_pkg/l_tray.md` §3）。

**工作区按用户归属（2026-09-17 二次修正；2026-09-26 页面改直连后同样成立）**：
现在页面直连时身份来自**页面自己的 `lugwit_token` cookie**（服务端 `_owner(user)` 解析），
和当年「随 `kwargs` 传给托盘 → `_http(token_override=...)`」是同一个用户口径；托盘那条路
（给别的调用方用）依旧支持传 `token`。`depot_workspace` 的 `owner` 取 JWT `sub`，
`workspaces()/workspace_upsert()/workspace_delete()` 全部带 owner 过滤，所以**归属就是登录的人**
（实测新建得到 `47/admin01`）。只有没传 token / 没带 cookie 时才退回本机自动授权账号
（那会解析成 `system01`，是另一套视角，别混用）。

⚠️ 早期版本没传 token，导致页面看到的是 `system01` 名下那几条（本机视角）。现已改掉：
`GET /api/depot/workspace` 只返回**当前用户**的工作区，卡片与下拉都显示 `👤 owner`，
标题右侧显示当前登录用户，工作区标签一眼能看出归属。

**多选删除**：`depot_workspace_delete` 支持 `ws_id`（单个）与 `ws_ids`（数组，批量），批量
逐个删、不因个别失败中断，返回 `{"deleted":[...],"failed":[{"id":...,"error":...}]}`。
页面侧：每张卡片一个勾选框 + 「全选」+ 「🗑 删除选中 (N)」（N=0 时禁用），删除前 confirm
列出待删名称，删完清空选中并重拉列表。

**改了动作不用重启托盘**（动作表是 `ExecHandler` 的类属性，注册即对运行中的服务生效）：

```bat
curl -s -X POST http://127.0.0.1:19527/run -H "Content-Type: application/json" ^
     -d "{\"module\":\"l_tray.depot_bridge\",\"function\":\"reregister\",\"reload\":true}"
```

`reload: true` 让托盘重新加载 `depot_bridge` 拿到新的 `_WEB_ACTIONS`，`reregister()` 再灌进
运行中的 ExecServer。验证：`curl -s http://127.0.0.1:19527/health` 的 `web_actions` 里应出现
`depot_workspace_*`。

实测（2026-09-17，客户端 8769 内重载页面）：

- 列表 = **当前登录用户**的工作区：`✓ admin01-l_wchat  👤 admin01  库 /l_wchat  未设本地路径`；
  标题右侧 `👤 admin01`，下拉 `✓ 📁 admin01-l_wchat · 👤 admin01 · /l_wchat · 未设本地路径`。
- 多选：勾一张卡 → 按钮变「🗑 删除选中 (1)」并解禁；「全选」勾上即全选、再点即清空。
- 归属 + 批量删除实测：页面内建 `zz-multi-del` → 回包 `47/admin01`；`ws_ids=[47]` 批量删 →
  `{"deleted":[47],"failed":[]}`，再列表只剩 `2:admin01-l_wchat`（测试数据自清）。
- `localStorage.depot_ws_v2` 里旧的 `admin01-l_wchat` **原封未动**——页面既不读也不写，解耦确认。

已实测：`list` / `save` / `delete`（含 `ws_ids` 批量）三件都跑通，见上面的实测记录。

## 7. 路径安全

`normalize_depot_path()` 拒绝：空路径、含 `..`、首段为 `.depot`。

## 8. 客户端

见《Rez包创建指导文档》第 12 章（QtWebEngine 两个坑 + QWebChannel 桥）。

```bat
wuwor lugwit_netdisk_client -- netdisk_client_doctor   :: 体检
wuwor lugwit_netdisk_client -- netdisk_client          :: 启动
```

### 8.1 登录与本地桥（2026-09-17）

- **启动登录窗**：注入 `LoginStore(data_dir=~/.Lugwit/lugwit_netdisk_client)`（库自带标题栏登录按钮 +
  启动静默恢复）；**没有任何已保存 token 时，启动 400ms 后自动弹登录对话框**
- **登录成功**：把 token ① 注入 WebEngine cookie（域名/路径按 `base_url` 设）② `bridge.setToken(token)`，然后 reload；
  **登出**清 cookie + 清桥 token
- **本地桥**（QWebChannel）新增 4 个槽，返回 JSON 字符串，**本地进程内直连 depot 服务**
  （基址 `L_CLIENT_DEPOT_URL` > `L_DEPOT_SERVICE_URL` > `http://127.0.0.1:1028`）：
  `depotBase` / `depotList` / `depotDownload` / `depotVersions`
- 与托盘的 `depot_bridge` 是**同一套语义**（谁在谁服务），两处都做路径校验
  （绝对路径、无 `..`、单文件 8MB 上限、文本类扩展名回文本否则 base64）

因此页面在 PC 客户端里**优先走本地桥取内容**（见 §6.1），不再依赖浏览器直连 depot。

## 9. 已验证

真实网盘 + 真实 `chatroom` 库跑通：提交 / 去重 / 秒传（**未验证：接口命中不了**，见 `Rez_pkg/lugwit_baidu_netdisk.md` §14.4）/ 历史 / 回滚 / 加锁 /
锁冲突 409 / sync_plan / blob 路径 / manifest 路径。测试数据已清理（errno 0）。

## 10. 已知未做

- errno 31034 限流退避
- 分片固定 4MB（VIP 账号可用 32MB）
- 并行 Range 下载
- 孤儿 blob GC
- 客户端 `fileMd5` 尚未接进提交流程（本地探秒传）
- 传输 daemon（SQLite 任务表 + 断点续传）

## 11. 明确不做

LAN 直连 / WireGuard 打洞 / Headscale / `Transport` 抽象层。
只做网盘一条路。过早抽象比不抽象更糟。

> **（2026-09-19 补充）该判据不变**：现在仍然只做百度一条路，不写第二个后端、不建插件框架、不改表结构。
> 但不妨碍把「将来可能换成自建云盘 / 自建公网服务器」这件事**预留成一条明确的缝**，见 §12
> —— 只定义接口形状、配置键与迁移路径，不写多后端代码。

## 12. 存储后端抽象（预留：未来自建云盘 / 自建公网服务器）

### 12.1 与 §11 的关系

§11 反对的是**现在**去建 `Transport` 抽象层 / LAN 直连 / 打洞。本节只做三件"零成本"的事：

1. 把百度特有的怪癖**收口**到一个模块边界内（现在它们散在 `depot_service` 里）；
2. 把**根前缀**从"百度 token 派生"改成**显式配置**（迁移时最痛的一条）；
3. 写明换后端的**迁移路径**，使将来是"换 driver + 搬实体"，而不是"重写"。

### 12.2 现状耦合面（证据）

| 层 | 现状 | 后端无关？ |
|---|---|---|
| HTTP 层 `web_server.py:2107,2859` | 只讲 `path`/`rev`/`ws` | ✅ |
| 元数据 `depot_store.py` | `depot_blob.remote_path` 存"后端内路径" | ✅（仅前缀语义相关） |
| manifest / 日志 / GC 工具 | cl_id json、按 `remote_path` 取文件 | ✅ |
| 闸门 `gate.py` | JWT 本地验签 | ✅ |
| **`depot_service.py`** | **8 处 `from .baidu_netdisk_api import …`**：`scan_file`、`upload_file_stream`、`meta_by_path`、`file_list`、`file_metas`、`download_dlink_to_path`、`normalize_dlink`，外加 `baidu_netdisk_auth.state_dir` | ❌ **唯一要收口的地方** |
| 百度特有语义 | 秒传(`rapid`)、**非报备应用 md5 不可信**(`md5_real`)、`dlink` 直链、`opera=copy`、`mkdir_cache`、限流 errno 31034 | ❌ 应被 driver 吞掉，不上浮 |

### 12.3 最小接口（driver seam，草案）

```python
class StorageBackend(Protocol):
    name: str                 # 'baidu' | 's3' | 'webdav' | 'localfs'
    root_prefix: str          # 取代 token 派生的 apps_root：'/apps/Lugwit' 或 's3://bucket/depot'
    caps: BackendCaps         # 能力标志，见 12.4

    def put(self, local: Path, remote: str, *, mkdir_cache=None) -> PutResult: ...
    def get(self, remote: str, dest: Path) -> None: ...
    def meta(self, remote: str) -> Meta | None: ...          # md5 可信度由 caps 表达
    def listdir(self, remote: str) -> list[Entry]: ...
    def exists(self, remote: str) -> bool: ...
    def mkdirs(self, paths: Iterable[str], cache: set[str]) -> None: ...
    def copy(self, src: str, dst: str) -> None: ...          # 可选；无实现则上层退化 get+put
    def delete(self, remote: str) -> None: ...               # GC 用
```

配套：`get_backend()` 按 `LUGWIT_DEPOT_BACKEND` 返回单例；`depot_service` 只 import 它，不再直接 import `baidu_netdisk_api`。

### 12.4 能力矩阵（缺什么就退化成什么）

| 能力 | 百度网盘 | 自建（对象存储 / WebDAV / 本地 FS） | 缺失时的退化 |
|---|---|---|---|
| `supports_rapid_upload` | ✅（命中率不稳，见 §14.4） | ⚠️ S3 无跨桶秒传；本地 FS 可用硬链 | 全量上传（慢但正确） |
| `trusted_md5` | ❌（非报备应用只有混淆指纹） | ✅（单段 ETag 可当 md5；本地 FS 直接算） | 一律本地读字节算 md5（**现状已如此**） |
| `supports_server_copy` | ✅ `opera=copy` | ⚠️ S3 `CopyObject` / WebDAV `COPY` | 上层 get+put |
| `direct_download_url` | ✅ `dlink`（有时效） | ✅ 签名 URL 或直读 | 服务端流式代理 |
| `path_style` | 绝对路径 `/apps/...` | bucket+key / WebDAV URL | 由 driver 归一成 `root_prefix + 逻辑段` |
| 限流 | errno 31034 退避 | HTTP 429/503 | 统一成 `RateLimited` 异常 + 退避 |

### 12.5 现在就该定下来的三个配置键

| 键 | 现状 | 预留语义 |
|---|---|---|
| `LUGWIT_DEPOT_BACKEND` | 无（隐含 baidu） | `baidu`（默认）｜将来 `s3` / `webdav` / `localfs` |
| `LUGWIT_DEPOT_ROOT_PREFIX` | 隐含 = token 里的 `apps_root`（`/apps/Lugwit`） | **显式**配置；baidu 后端缺省时仍回退 token 派生（向后兼容） |
| `LUGWIT_DEPOT_BASE_URL` | 隐含 `http://127.0.0.1:8080/baidu`（本机调试口） | **跨机一律走 nginx 单入口 + 前缀**：生产 `https://<入口>/baidu`（将来域名可带非标端口）。公网只开 443（+ 未来 8443），后端端口不对外 → 本机开发用回环 8080/1028，**跨机不得直连 1028**。见《Nginx反向代理机制》§2.1、§7 |
| 后端凭据 | `baidu_netdisk_token.local.json` | 按后端分文件：`~/.lugwit/baidu_netdisk/*`、将来 `~/.lugwit/depot_s3/*` … |

> `depot_blob.remote_path` 里**已经**是后端内完整路径，所以"换后端 = 改前缀 + 搬实体"，**不用改表**。

### 12.6 换后端的迁移路径（真到那天照做）

1. 新增 driver（实现 §12.3 的 8 个函数）+ 注册进 `get_backend()`；
2. `tools/depot_migrate_backend.py --from baidu --to s3 --lib <库> --dry-run`：
   逐 blob `get` → `put` → 校验 size/md5 → `UPDATE depot_blob SET remote_path=…`；
   **复用现有范式**：`depot_migrate_blob_lib.py` / `depot_migrate_dir.py` 的「dry-run 优先 + 云端先动 DB 后动 + 失败即中止」；
3. 切 `LUGWIT_DEPOT_BACKEND` + `ROOT_PREFIX`，跑 `depot_manifest_verify`（待补，见演进计划 T4）+ 抽样下载；
4. 双写/只读期 → 观察期 → 下线旧后端凭据；
5. dir 模式的活文件由 driver 落盘到新后端（`.versions/vNNN` 语义与保留策略不变）。

### 12.7 对 auth 的影响：**零**

auth 只依赖 **HTTP 契约**（`submit_stream` / `download` / `list`），不 import 任何百度 SDK；
`/l_auth_backup` 的备份与恢复逻辑与后端无关。**换后端不动 auth** —— 这正是"预留扩展能力"要保住的性质
（见《lugwit_auth统一用户授权服务设计》§13.7、§13.4 P-B 的离线验签同样与后端无关）。

### 12.8 现在**不做**的（守住 §11）

- 不写第二个 driver；不引入 `Transport` / 多后端插件框架；不把 driver 暴露成对外 API；不改 DB 表结构；
- **唯一"现在做"的动作**：把 `LUGWIT_DEPOT_ROOT_PREFIX` 显式化 —— 它是"token 派生"这层百度耦合里
  最容易被遗忘、迁移时最痛的一条。
