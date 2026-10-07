# l_notepad_server 使用文档

> `l_notepad_server` 是 L Notepad 的服务端（FastAPI / uvicorn，8765），提供 Web UI、REST API 与
> **知识库**（多知识库，每库独立路由 `/web/kb/{name}`，内含目录层级与多篇文章）。
> 不依赖桌面库（PySide6），认证经 `lugwit_auth`（1027）HTTP 接入，客户端/服务端经 REST + token 契约通信。
>
> 更新：2026-09-17

## 访问入口

- **知识库页面**：<http://localhost:8080/note/web/kb/rez_pkg>（`rez_pkg` 知识库，总览页 `/note/web/kb`）
- **入口页 `/`**：**302 → `/web`（我的笔记列表）**；未登录访问先 302 → `/login`（2026-09-17 变更，详见 §8）
- **登录账号**：使用受控账号凭据
- **登录密码**：使用受控账号凭据

> 登录页与网络入口见[../Nginx反向代理机制.md](../Nginx反向代理机制.md)；知识库访问权限由认证服务控制。

---

## 1. 拆包结构

原 `l_notepad` 拆成三个 rez 包（旧包 `l_notepad` 保留，仅作历史/迁移参考）：

| 包 | 角色 | 入口 alias | 说明 |
|---|---|---|---|
| `l_notepad_server` | 服务端（FastAPI/uvicorn，8765） | `l_notepad_api` / `l_notepad_api_reload` | 笔记业务 API + Web 页面；两个别名命令相同（热重载改由进程内 `L_SRC_WATCH` 负责，`--reload` 已弃用） |
| `l_notepad_client` | 客户端（Qt 桌面） | `l_notepad_client` / `l_notepad_ori` | 标题栏、登录、脚本编辑器等；**纯 PC 模式，不再拉起本地后端** |
| `l_notepad` | 旧单体包 | — | 拆包前的代码，保留供对照/迁移 |

服务端入口（`l_notepad_server/999.0/package.py`）：

```python
alias("l_notepad_api",        "python -m l_notepad_server.backend_server")
alias("l_notepad_api_reload", "python -m l_notepad_server.backend_server")   # 命令同左；热重载走进程内 L_SRC_WATCH
```

客户端启动（`l_notepad_client/999.0/package.py`）：

```python
alias("l_notepad_client", "python -m l_notepad_client.local_main")       # 纯 PC 模式：本地文件笔记，不起后端
alias("l_notepad_ori",    "python -m l_notepad_client.local_main_ori")   # 系统原生标题栏
```

> **2026-09 变更**：客户端不再有"内嵌后端"模式 —— 原来的 `l_notepad_with_api`（`python -m l_notepad_client.main`，
> 会 `subprocess` 拉起 `l_notepad_server.backend_server` 到 127.0.0.1:8765）连同 `web_ui.py` 已删除；
> 需要后端时单独启动 `l_notepad_api`（部署机上由服务侧/守护进程负责）。

### 1.1 requires

```python
requires = [
    "python-3.12.10",
    "fastapi", "uvicorn", "jinja2", "pydantic", "python_multipart",
    "l_qframelesswindow", "pytracemp",
    "watchfiles",
]
```

- **不直接依赖 `lugwit_baidu_netdisk` 包**——cloud_sync（云端镜像/网盘链接）经其 Web 服务 HTTP 接口接入
  （百度 token 在服务端侧；notepad 侧凭 env `LUGWIT_ACCESS_TOKEN` / 登录态调它 ——
  `depot_map.py` 的 `require_token()` 取不到就直接报错，**同机自动授权自 P0 起默认关、不再是取 token 途径**），
  避免拉入服务端重依赖
- `watchfiles` 为 `SrcWatchService` 的可选监视后端；服务不再使用 uvicorn `--reload` 作为现行机制。
- **重要**：截至 2026-09-17，`l_notepad_server` 的 `L_SRC_WATCH` 对 `.py` 改动实测不可靠。Python 改动后必须手动重启；模板是否刷新仍按现行热重载文档核验。

## 2. 端口与网络拓扑

- 服务端 `backend_server.py`：`--port` 默认 `L_NOTEPAD_PORT`（缺省 **8765**），只监听 `127.0.0.1`
- 统一入口走 **nginx 8080**：`/note/*` 剥前缀 → 8765
- 认证：`/api/v1/*` → lugwit_auth（1027）

## 3. 数据根目录（关键！）

数据根由 wuwo `config.yaml` 的 **`l_data_dir` 模板** 决定，默认 `{user}/.lugwit/main/{pkg}`，由 `paths.py` 解析：

```python
# l_notepad_server/.../paths.py
def _instance_data_root(pkg):
    # 解析顺序（第一个能的赢）：
    #   1. LUGWIT_DATA_ROOT            —— wuwo 注入的实例数据根（优先）
    #   2. LUGWIT_DATA_DIR_TEMPLATE    —— wuwo 按 config.yaml 展开好的 l_data_dir 模板（留 {pkg}）
    #   3. 树内 config.yaml 的 l_data_dir 模板（不在 wuwo 环境里跑的进程兜底）
    #   4. 兜底 ~/.lugwit/<包>
    ...
```

> 这条链只有一处实现：`l_notepad_server/.../paths.py` 的 `_instance_data_root()`
> （`l_notepad_client` 里有一份同名副本）。

> ⚠️ **包名变化 → 数据根目录变化。** 拆包后服务端包名是 `l_notepad_server`，
> 所以服务端数据根是 `~/.lugwit/main/l_notepad_server/`，**不是** 旧包的 `~/.lugwit/main/l_notepad/`。
> 两者是不同目录，数据不会自动跟着包名搬家。

数据根下的结构：

```
~/.lugwit/main/l_notepad_server/
├── notepad_list/            笔记（.md 等文件，含 _images/）
├── favorites/               收藏夹 + 剪贴板历史 + 热键配置
├── version_history.sqlite3  笔记版本历史
├── external_files.json      外部文件状态
├── note_order.json          笔记手动排序
├── notepad.sqlite3          服务端笔记库（笔记归属/共享等元数据）
├── account_favorites.json   账号收藏（本地缓存；云端走 auth）
├── auth_token.json          登录令牌
└── server_config.json       服务器地址配置（标题栏「服务器设置」写出）
```

> **2026-09-20 变更**：账号/凭据的加密密钥**已不在本目录**——
> 主密钥归位到 `~/.lugwit/lugwit_auth/master.key`（Windows 下 DPAPI 包裹；旧文件
> `~/.lugwit/l_notepad/.accounts_key` 已轮转并改名 `.bak`），且账号数据早已由
> `lugwit_auth` 统一托管（表从 `l_notepad_accounts` 泛化为 `credentials`/`favorites`，
> 端点 `/accounts*` 仍在、内部指向新表）。
>
> **2026-09-22 变更（信封加密）**：`credentials` 升级为**每行 DEK + KEK 包 DEK**
> （新列 `dek_wrapped`/`kek_id`）；主密钥只作 KEK。生产曾因「生产 KEK ≠ 加密数据的 KEK
> + dev 与生产共用同一 PG」导致 `/accounts` 503，已修复（KEK 对齐 + 迁移 + 升级信封）。
> 详见 `../lugwit_auth统一用户授权服务设计.md` §9 P4 / **P4.5**。

`notepad.sqlite3` 记录**归属**（owner）与**共享**（public）等元数据，笔记正文以文件形式存于
`notepad_list/`。旧 `notepad.sqlite3` 若为 0B 属正常（拆包前的旧版可能用文件存储、无库），归属表由启动迁移补齐。

## 4. 启动迁移机制（migrate_legacy_notes）

服务端 `create_app`（`backend_server.py`）启动时调用：

```python
# backend_server.py（create_app 内）
_migrated = note_access.migrate_legacy_notes(conn, notes_root, "admin01")
for _p in _migrated:
    note_access.set_public(conn, _p, True, "read")   # 迁移的旧笔记设为全部共享(只读)
```

- 把 `notepad_list/` 下**尚未在库中注册**的笔记文件登记给 `admin01` 并设为共享（幂等）
- 失败只打日志、**不阻断启动**

> ⚠️ 迁移**只在进程启动时跑一次**，且登记的是**启动时 notes_root 已存在的文件**。
> 若服务已启动后再拷入笔记文件，需**重启服务**让它重新执行迁移，否则新文件不会出现在列表里。

## 5. 拆包/换机后数据怎么来

服务端不会自动从旧包目录 `~/.Lugwit/l_notepad/` 搬数据（`paths.py` 的 `move_dir_contents`
只处理**程序包内**的旧位置，如 `_PKG_DIR/notepad_list`，不迁移另一个数据根目录）。因此：

1. **手动拷贝**旧笔记到新数据根：
   ```bat
   xcopy /E /Y "%USERPROFILE%\.Lugwit\l_notepad\notepad_list" "%USERPROFILE%\.Lugwit\l_notepad_server\notepad_list"
   ```
2. **重启服务**（主页「L Notepad 笔记」卡片 → 热更新/重启，或 `l_notepad_api_reload`），
   让 `migrate_legacy_notes` 把已拷入的笔记注册给 admin01。
3. 验证：`GET http://127.0.0.1:8080/note/api/notes`（需 token）应返回笔记列表。
   归属 admin01，与登录用户一致即可正常查看/编辑。

若旧数据原本就存在（非空），且只换包名不换机器，也可手动把整个数据根迁过去，注意保留
`server_config.json` 的 host 配置与 `auth_token.json`。

## 6. 登录 / 认证 / API 统一走 Nginx 代理

登录/认证、账号/收藏、笔记业务 API 全部收敛到 **nginx 统一入口 8080** 反向代理，
不直连后端端口（认证 1027、笔记 8765 只监听 `127.0.0.1`，不对外暴露）。

```
客户端 ──8080──▶ nginx 统一入口
                 │ /api/v1/auth/login ──▶ 127.0.0.1:1027 认证（登录）
                 │ /api/v1/*         ──▶ 127.0.0.1:1027 认证（账号/收藏/用户）
                 │ /note/api/*       ──▶ 127.0.0.1:8765  笔记（剥 /note）
```

### 6.1 配置项

| 配置键 | 含义 | 生产默认（公网机） | 开发默认（开发机） |
|---|---|---|---|
| `auth_url` | 认证服务地址（nginx 入口） | `https://121.196.144.88` | `http://127.0.0.1:8080` |
| `auth_route` | 认证路由前缀 | `/api/v1/auth` | `/api/v1/auth` |
| `api_url` | 笔记/账号 API | `https://121.196.144.88/note` | `http://127.0.0.1:8080/note` |
| `log_server_url` | 远端日志服务 | 同上 | 同上 |

> 2026-09-17 更正：生产入口已从 `http://121.196.144.88:8080` 改为 **`https://121.196.144.88`**（
> 8080 已收成 `listen 127.0.0.1`，只供本机调试；公网唯一入口是 443）。
> 域名来自 wuwo 的 `LUGWIT_DOMAIN_URL`，而 `wuwo/config/config.yaml` 的 `domain` 已留空（只走 IP）。

登录 / 认证端点拼接规则：`auth_url + auth_route + "/..."`。

### 6.2 配置优先级与来源

读取优先级：**UI 持久化 > 环境变量 > 默认值**（由 `l_qframelesswindow` 的 `ServerConfigStore` 实现）。

1. **UI 持久化**：`~/.lugwit/main/l_notepad_server/server_config.json`（标题栏「服务器设置」写出的配置，会盖过一切）
2. **环境变量**：`LUGWIT_AUTH_URL` / `LUGWIT_AUTH_ROUTE` / `L_NOTEPAD_API_URL` / `L_NOTEPAD_LOG_SERVER`
3. **默认值**：`l_notepad_server/server_config.py` 里的 `_DEFAULTS`，**随 `Lugwit_deploy` 自动区分开发/公网机**

> ⚠️ 若要 `Lugwit_deploy` 生效，必须保证 `server_config.json` 里**没有**持久化的 host 键
> （`auth_url` / `api_url` / `log_server_url`），否则持久化会盖过机器类型默认值。

### 6.3 `Lugwit_deploy` 自动区分开发机 / 公网部署机

- `Lugwit_deploy=1`（或 `true/yes/on`）→ **公网部署机**，默认走生产 nginx 8080 统一入口
- 缺省 / `0` → **开发机**，默认走本机 nginx `127.0.0.1:8080`

`Lugwit_deploy` 需在**进程启动前**设好（系统级环境变量），模块导入时即决定默认值。

### 6.4 排查

1. 确认当前机器 `Lugwit_deploy` 与实际 host 是否匹配（开发机缺省/0，公网机 1）
2. 检查 `%USERPROFILE%\.Lugwit\l_notepad_server\server_config.json` 是否残留 host 键，用标题栏「服务器设置」清掉
3. 验证 nginx 路由：`curl http://127.0.0.1:8080/nginx-health`、
   `curl http://127.0.0.1:8080/api/v1/health`（认证）、
   `curl http://127.0.0.1:8080/note/api/health`（笔记）、
   `curl http://127.0.0.1:8080/note/api/notes`（应 401，证明剥前缀正确）

## 7. 知识库（KB）

- 网页端：`/web/kb`（总览）、`/web/kb/{name}`（某知识库）
- REST：`/api/kb/bases*`、`/api/kb/{name}/...`
- 数据库表：`knowledge_bases`（含 `workspace`）、`knowledge_categories`、`knowledge_articles`

### 7.1 知识库工作区页面（`/web/kb/{name}`）

页面把**本机工作区文件 + 云端 Depot 文章合并显示**，每项带状态徽标。

- **文件夹层级**：左侧「工作区笔记」列表按相对路径的目录层级递归展示（📂/📁 可展开/折叠），
  子目录与文件逐层缩进，便于在多级目录工作区中按文件夹定位笔记。

| 图标 | 状态 | 含义 |
|---|---|---|
| ☁ | 云端 | 只在云端（Depot），本机工作区没有 |
| 📄 | 仅本地 | 本机工作区有，尚未上传 |
| ✎ | 已修改 | 云端有、本机也有且内容不一致（本机改了未同步） |
| ✓ | 已同步 #N | 云端有、本机一致 |

- **合并与比对**：前端 `reconcileStatus()` 把云端（`/baidu/api/depot/list`）与本机
  （`/api/kb/{kb}/workspace`）按相对路径合并，对两者都存在的文件拉取本地内容 + 云端最新版比对得出 `modified`
- **预览**：本机存在的文件优先显示本地内容（能看到"已修改"的真实改动），纯云端文件回退读 Depot
- **编辑**：`.md` 笔记可用页面内置 `<textarea>` 编辑（✏ 编辑 / 💾 保存），保存走
  `PUT /api/kb/{kb}/workspace/file` 写回工作区，随后状态标为「已修改」
- **上传守卫**：`submitToBaidu()` 和「➕ 上传本地文件」只处理 `仅本地`/`已修改`；未修改（已同步）的直接提示跳过
- **右键菜单**：显示状态 + 版本 + 百度云地址（`dir` 模式显示真实物理路径，`blob` 模式显示逻辑路径），
  仅 `仅本地`/`已修改` 时提供「提交到百度云」
- **归档（版本库）内容读取优先级**（2026-09-17）：① 客户端本地桥 `window.lugwitBridge.depotDownload`
  → ② 托盘中转（`depot_download` 动作，经 19527）→ ③ 回退 `/baidu/api/depot/download`；
  托盘状态浮层新增一行「版本库读取：客户端本地桥 / 经托盘中转（基址） / 回退 `/baidu` 直连」。
  归档逻辑路径必须取映射的 `base_path`（形如 `/notes/rez_pkg/xxx.md`）——
  旧写法 `/<kbName>/<rel>`（缺 `/notes` 前缀）会 404「版本不存在」，本次已修。

### 7.2 工作区接口

| 接口 | 方法 | 说明 |
|---|:---:|---|
| `/api/kb/{kb}/workspace` | GET | 列工作区内可预览文本（.md/.txt/.rst/.log，含相对路径/大小） |
| `/api/kb/{kb}/workspace` | PUT | 设置工作区目录（body `{"workspace": "D:\\..."}`） |
| `/api/kb/{kb}/workspace/file?path=<rel>` | GET | 读取单个文档内容（预览/编辑加载） |
| `/api/kb/{kb}/workspace/file?path=<rel>` | PUT | 写入文档内容（编辑保存，body `{"content": "..."}`） |

- 相对路径有**防穿越校验**（`_safe_ws_path`），只能落在工作区目录内
- 写入仅允许可预览文本类型（`_WORKSPACE_EXTS`）

### 7.3 提交到百度云 Depot

工作区文档「⬆ 提交到百度云」调用 `lugwit_baidu_netdisk` 的 `/baidu/api/depot/submit_stream`。
Depot 支持多存储模式（blob / 目录镜像），按逻辑根登记，详见
《Rez_pkg/lugwit_baidu_netdisk.md》第 13 节。222

#### 7.3.0 归档文件的四个 kb 端点（2026-10-05 起含删除）

| 接口 | 方法 | 说明 |
|---|:---:|---|
| `/api/kb/{kb}/depot` | GET/PUT | 读 / 改归档映射（library / subpath / ws 名 / 本机工作区根） |
| `/api/kb/{kb}/depot/list?rel=&recursive=1` | GET | 列归档目录（递归时给全部文件 + `rev`/`action`/`excluded`） |
| `/api/kb/{kb}/depot/file?rel=&rev=` | GET | 读归档文件（`rev=0` 最新；已删文件 **410**） |
| `/api/kb/{kb}/depot/submit?rel=&description=` | POST | 提交新版本（body = 原始文本） |
| `/api/kb/{kb}/depot/delete?rel=&description=` | POST | **标记删除**（blob 与历史保留、可 revert），随即重索引该库 |

**删除入口（2026-10-05 新增，`depot_map.delete_file` + 路由薄包装）**：把「算映射 →
带 `ws` → 拼 depot 逻辑路径 → 带登录态 → 通知重索引」收进服务端一处，调用方只说
「哪个库、哪个文件」（`rel` 相对知识库子路径）。要点：

- `rel` **为空直接 400** —— 空 rel 折算出的是整个库的 `base_path`（等于把整库标删）；
- 走 depot 的 `POST /api/depot/delete`（**JSON body** `{"paths": [...], "description": ...}`，
  `ws` 在 query）；已删的文件再删一次是**幂等**的（`cl_id 0`，不写新版本）；
- **8765 自己的闸门读 `Authorization: Bearer` 或 cookie `l_notepad_token`**（不是 `lugwit_token`）
  —— 只有 **GET/HEAD 且本机直连**才免 token，POST 一律要登录态；
- 返回带 `warning`：本机工作区里还有同一文件时提醒"自动同步会把它重新传回来"
  （`_needs_upload` 对"归档取不到（含已删）"一律视作需要上传）。要真删得同时处理本机那份。
- 删完 `notify_kb_change` 立即重索引 → 旧行（词法 + 向量）走同一套清旧行。

#### 7.3.1 depot 登录态（服务端，2026-09-20）

depot（1028）每个请求都过 lugwit_auth 闸门，`/auth/auto` 回环兜底已按 P0 关闭，
所以笔记服务访问 depot 必须自带登录态，`depot_map.require_token()` 按序取：

1. 环境变量 `LUGWIT_ACCESS_TOKEN`（l_scheduler 登录后注入）；
2. 环境变量 `LUGWIT_USER` + `LUGWIT_PASSWORD`；
3. 机器本地凭据文件 `~/.lugwit/l_notepad_server/depot_auth.json`
   （不入库、不随包推送；内容 `{"lugwit_user": "账号", "lugwit_password": "密码"}`）。

都没有 → 知识库页「索引来源」处显示 `DepotError: depot 未配置登录态…`。
登录 token 有缓存，401 时自动换新重试；**补填凭据文件后不用重启服务**，
刷新知识库页面重新触发索引即可。

**管理员网页登录自动配置（2026-09-20）**：`role ∈ {admin, system}` 的账号在
网页登录成功且服务进程尚无 depot 登录态时，服务端自动把本次登录的账号密码
写成上述凭据文件（0600，不入库不推送）——即「管理员登录一次，depot 索引即可用」。
账号密码后来改了 → 旧凭据登录失败 → 下次管理员登录自动重播覆盖。
`LUGWIT_DEPOT_AUTO_SEED=0` 可关闭该行为。

### 7.4 笔记云镜像（已改为走 depot 库）

笔记的「云同步」不再往裸目录 `<apps>/notes` 写文件，而是把**笔记做成一个 depot 库**
（`/notes`，dir 模式），物理落在 `<apps>/version_depot/dir_mirror/notes/`：

| 行为 | 实现 |
|---|---|
| 镜像位置 | `cloud_sync._remote_base()` → `{apps}/version_depot/dir_mirror{depot_library}` |
| 推送一版 | `_push_upsert()` → `POST /api/depot/submit_stream?path=/notes/<rel>&ws=<ws>`（原始字节）。一次保存 = 一个版本，可在 depot 页面看历史/差异 |
| 删除 | `_push_delete()` → `POST /api/depot/delete`（写删除版；netdisk 侧同时删掉 dir 镜像里的活文件，`.versions` 快照保留） |
| 工作区 | 首次自动建 `notes-sync` 工作区并缓存 id（depot 接口都要带 ws） |
| 拉取 | 仍按文件列目录，但**跳过 `.versions/` 快照**（那是版本历史，不是笔记） |
| 「百度云地址」 | `note_browser_link()` → `{files_base}/depot?path=/notes/<rel>`，depot 页面支持 `?path=` 深链（自动定位并落到「📄 预览」标签） |

配置（`~/.lugwit/main/l_notepad_server/cloud_sync.yaml` 或环境变量）：

| 键 | 环境变量 | 默认 | 说明 |
|---|---|---|---|
| `use_depot` | `L_CLOUD_SYNC_USE_DEPOT` | `true` | 关掉则退回旧的裸目录 `<apps>/notes` 行为 |
| `depot_library` | `L_CLOUD_SYNC_DEPOT_LIB` | `/notes` | 笔记库 root（dir 模式） |
| `depot_workspace` | `L_CLOUD_SYNC_DEPOT_WS` | `notes-sync` | 提交用的工作区名（不存在会自动建） |

存量笔记（老 `<apps>/notes` 里的文件）用「全部同步到百度云」重推一遍即可进入库；
旧的裸目录确认无误后可删除。

## 8. Web UI：入口页与顶栏全局搜索（2026-09-17）

本次 Web UI 两处变化。**模板由 Jinja 缓存，改模板后必须重启 8765 才生效。**

### 8.1 入口页 `/` 改为 302 → `/web`

- 访问 `/` 不再渲染欢迎页，改为 **302 重定向到 `/web`（我的笔记列表）**；`templates/index.html` **已删除**
- 未登录访问 `/` 先 **302 → `/login`**；`login.html` 登录成功本来就跳 `/web`
- 顶栏导航去掉「🏠 首页」项，品牌 logo 链接改为 `/web`

### 8.2 顶栏全站统一搜索框

由 `base.html` 注入，**所有页面都有**（**含 `/web/index` 搜索索引页**）：

- **DOM**：`#global-search-input`（输入框）+ `#global-search-pop`（弹窗容器）
- **交互**：输入即弹窗（**220ms 防抖**），请求 `/api/search?q=…&limit=8&mode=lex`；点击结果打开对应文档
- **样式与「搜索索引」页同源**：结果 CSS 抽到 `static/app.css`（`.hit / .hit-head / .badge-score / .snip / .sig*`），
  渲染逻辑抽到 `static/app.js` 的 **`LN.renderSearchHits(hits)`**；搜索索引页（`web_index.html`）也改用它 → 两处永不漂移
- **各页原有搜索框去重**：
  - `web_list.html`、`web_kb_overview.html`：删掉顶栏搜索框（统一用全局的）
  - `web_kb.html`：「搜索本知识库」框**下移到页内工具栏**（id `kb-search-input` 不变）

## 9. 排查速查

| 现象 | 排查 |
|---|---|
| 笔记列表空 | 数据根是否 `~/.lugwit/main/l_notepad_server`（非旧包）；`notepad_list/` 是否有文件；是否在拷入后重启过服务 |
| 归属不对/看不到别人笔记 | `notepad.sqlite3` 归属表；`migrate_legacy_notes` 注册为 admin01 且设为共享 |
| 想改服务器地址 | 标题栏「服务器设置」或 `~/.lugwit/main/l_notepad_server/server_config.json` |
| 登录/API 不通 | 见第 6 节排查（Lugwit_deploy、server_config.json 残留、nginx 路由） |

## 10. 相关文档

- 《Nginx反向代理机制.md》— 路由与转发
- 《标题栏提供的服务.md》— 客户端标题栏能力
- 《Rez包创建和启动指导文档.md》— rez 包/启动/修饰符/自动下载依赖
- 《Rez_pkg/lugwit_baidu_netdisk.md》— 云端 Depot 多模式存储与迁移

## 变更记录（2026-10-04 / 10-05）

- **新增归档删除入口 `/api/kb/{kb}/depot/delete`（2026-10-05）**：以前要删归档只有一个办法 ——
  拿 cookie `lugwit_token` 直打 depot 1028 的 `POST /api/depot/delete`，自己算逻辑路径与工作区名。
  现在收进 8765 一处（`depot_map.delete_file` + `routers/kb.py` 薄包装）：`rel` 必填（空 → 400，
  否则折算出整个库的 `base_path`）、走 JSON body、幂等（已删再删 = `cl_id 0`）、删完
  `notify_kb_change` 即时重索引。实测闭环：提交临时文档 → 列表可见 → 删 → 列表 56→55、
  `depot/file` 410、索引三张表零残留、搜索 0 命中。**踩到的坑**：8765 的闸门认
  `Authorization: Bearer` 或 cookie **`l_notepad_token`**（不是 `lugwit_token`，那是 1028 的口径），
  且只有本机直连的 GET/HEAD 免 token。
  详见 §7.3.0。

- **知识库工作区的自动同步节奏：5s 轮询 / 3s 防抖 → 20s / 10s**。改了两处、缺一不可：
  ① 代码默认（`workspace_sync.WS_SCAN_TTL_S` / `WS_DEBOUNCE_S`，环境变量
  `L_NOTEPAD_WS_SCAN_TTL_S` / `L_NOTEPAD_WS_DEBOUNCE_S`）；② `app_settings` 的**运行时值**
  （`ws_autosync_interval_s` / `ws_autosync_debounce_s`，用 `PUT /api/search/ws_autosync` 改）。
  为什么两处都要改：`_load_settings()` 只在 DB 里的值能解析成 `>0` 时才覆盖默认 —— 只改代码可能**不生效**。
  影响：文档从保存到"可被搜到"的延迟由 ≈8s 变为 **≈30s**（20s 轮询 + 10s 防抖）。
  注：该设置端点用 **`PUT`**（`POST` 会 405），且**需要登录**（无 token 是 401）。
- **`depot_map.list_tree(..., live_only=True)`**：索引侧 3 处列举调用都传了它，用于排除"删除状态"条目。
  但**实测更正**：归档的 `/depot/list` **本身就排除已删文件**（标删后该库条目 33 → 29），
  所以这个过滤目前是**冗余但无害**的保险；真正的信号是"列表里没有它"。
  详见 `知识库归档删除同步与索引清理_计划.md` §2.3。
- **已删文件的 `depot/file` 回 410**（「该版本已删除…」；blob 与历史版本保留，只是 `rev=0` 不再可读）。
  → `notepad_read` 的归档兜底在已删文件上会**自然失败**（符合"已删不兜底"的约定）。
- **索引清旧行（2026-10-04 收尾）**：以前只清**词法侧**，向量侧（`vec_docs`/`vec_chunks`）
  要等"下一次嵌入"才消失 —— 而嵌入那趟**只由 `tools/vec_rebuild.py` / 显式重嵌触发**，
  可能几天不跑一次（实测 `last_embedded_at` 停在 2026-10-02，4 篇已删文档仍在语义召回里）。
  改了三处：
  ① `search_index._drop()` **两侧同删**（顺手调 `search_vec.drop_doc`，连带把内存块向量缓存标脏）
  —— 它是一切删除路径的唯一出口（笔记消失 / 代码消失 / `never_index` / kb 清旧行 …），修一处全受益；
  ② `search_vec.purge_orphan_vec()`：摘"没有对应词法行"的向量行（从 `_pending_docs` 里抽出来，
  让周期兜底也能调）；
  ③ `search_index.purge_orphan_index()`：兜底清两类残留 —— **已删知识库**的词法行
  （`knowledge.delete_base` 不清检索索引，这些行再没机会被重算）+ 上面的向量残留；
  挂在 **kb 兜底 tick（300s）** 与**启动预热**上，`kb_bases()` 为空时直接返回。
  核对：`vec_docs` 无词法行 = 0 / `vec_chunks` 无对应文档 = 0 / 已删文档一行不剩。
  详见 `知识库归档删除同步与索引清理_计划.md` §2.4。
