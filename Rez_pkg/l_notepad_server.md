# l_notepad_server 使用文档

> `l_notepad_server` 是 L Notepad 的服务端（FastAPI / uvicorn，8765），提供 Web UI、REST API 与
> **知识库**（多知识库，每库独立路由 `/web/kb/{name}`，内含目录层级与多篇文章）。
> 不依赖桌面库（PySide6），认证经 `lugwit_auth`（1027）HTTP 接入，客户端/服务端经 REST + token 契约通信。
>
> 更新：2026-09-03

## 访问入口

- **知识库页面**：<http://localhost:8080/note/web/kb/rez_pkg>（`rez_pkg` 知识库，总览页 `/note/web/kb`）
- **登录账号**：`admin01`
- **登录密码**：`666`

> 登录页：`http://localhost:8080/note/login`；知识库需登录后访问，发布/查看均要求具备笔记访问权限（admin01 可见全部）。

---

## 1. 拆包结构

原 `l_notepad` 拆成三个 rez 包（旧包 `l_notepad` 保留，仅作历史/迁移参考）：

| 包 | 角色 | 入口 alias | 说明 |
|----|------|-----------|------|
| `l_notepad_server` | 服务端（FastAPI/uvicorn，8765） | `l_notepad_api` | 笔记业务 API + Web 页面；`l_notepad_api_reload` 带 `--reload` |
| `l_notepad_client` | 客户端（Qt 桌面） | `l_notepad` / `l_notepad_ori` / `l_notepad_with_api` | 标题栏、登录、脚本编辑器等 |
| `l_notepad` | 旧单体包 | — | 拆包前的代码，保留供对照/迁移 |

服务端入口（`l_notepad_server/999.0/package.py`）：

```python
alias("l_notepad_api",        "python -m l_notepad_server.backend_server")
alias("l_notepad_api_reload", "python -m l_notepad_server.backend_server --reload")
```

客户端启动（`l_notepad_client/999.0/package.py`）：

```python
alias("l_notepad",           "python -m l_notepad_client.local_main")        # 自定义无边框标题栏
alias("l_notepad_ori",       "python -m l_notepad_client.local_main_ori")    # 系统原生标题栏
alias("l_notepad_with_api",  "python -m l_notepad_client.main")              # 拉起内嵌后端
```

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
  （同机 127.0.0.1 自动授权，服务端复用其持有的百度 token），避免拉入服务端重依赖
- `watchfiles` 来自 `requires`（而非硬编码 `E:\py_flow\py312` PYTHONPATH hack），供 uvicorn `--reload`
  覆盖全文件类型热更新（依赖缺省时 wuwo 会自动下载，见《Rez包创建和启动指导文档.md》）

## 2. 端口与网络拓扑

- 服务端 `backend_server.py`：`--port` 默认 `L_NOTEPAD_PORT`（缺省 **8765**），只监听 `127.0.0.1`
- 统一入口走 **nginx 8080**：`/note/*` 剥前缀 → 8765
- 认证：`/api/v1/*` → lugwit_auth（1027）

## 3. 数据根目录（关键！）

数据根由 wuwo `config.yaml` 的 **`data_dir` 模板** 决定，默认 `{user}/.Lugwit/{pkg}`，由 `paths.py` 解析：

```python
# l_notepad_server/.../paths.py
def _resolve_data_root():
    tmpl = _read_data_dir_template()          # 例如 "{user}/.Lugwit/{pkg}"
    return tmpl.replace("{user}", home).replace("{pkg}", _PKG_NAME)  # 或回退 ~/.Lugwit/<包名>
```

> ⚠️ **包名变化 → 数据根目录变化。** 拆包后服务端包名是 `l_notepad_server`，
> 所以服务端数据根是 `~/.Lugwit/l_notepad_server/`，**不是** 旧包的 `~/.Lugwit/l_notepad/`。
> 两者是不同目录，数据不会自动跟着包名搬家。

数据根下的结构：

```
~/.Lugwit/l_notepad_server/
├── notepad_list/            笔记（.md 等文件，含 _images/）
├── favorites/               收藏夹 + 剪贴板历史 + 热键配置
├── version_history.sqlite3  笔记版本历史
├── external_files.json      外部文件状态
├── note_order.json          笔记手动排序
├── notepad.sqlite3          服务端笔记库（笔记归属/共享等元数据）
├── account_favorites.json   账号收藏
├── .accounts_key            账号加密密钥
├── auth_token.json          登录令牌
└── server_config.json       服务器地址配置（标题栏「服务器设置」写出）
```

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
|--------|------|--------------------|--------------------|
| `auth_url` | 认证服务地址（nginx 入口） | `http://121.196.144.88:8080` | `http://127.0.0.1:8080` |
| `auth_route` | 认证路由前缀 | `/api/v1/auth` | `/api/v1/auth` |
| `api_url` | 笔记/账号 API | `http://121.196.144.88:8080/note` | `http://127.0.0.1:8080/note` |
| `log_server_url` | 远端日志服务 | 同上 | 同上 |

登录 / 认证端点拼接规则：`auth_url + auth_route + "/..."`。

### 6.2 配置优先级与来源

读取优先级：**UI 持久化 > 环境变量 > 默认值**（由 `l_qframelesswindow` 的 `ServerConfigStore` 实现）。

1. **UI 持久化**：`~/.Lugwit/l_notepad_server/server_config.json`（标题栏「服务器设置」写出的配置，会盖过一切）
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
|------|------|------|
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

### 7.2 工作区接口

| 接口 | 方法 | 说明 |
|------|:---:|------|
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

## 8. 排查速查

| 现象 | 排查 |
|------|------|
| 笔记列表空 | 数据根是否 `~/.Lugwit/l_notepad_server`（非旧包）；`notepad_list/` 是否有文件；是否在拷入后重启过服务 |
| 归属不对/看不到别人笔记 | `notepad.sqlite3` 归属表；`migrate_legacy_notes` 注册为 admin01 且设为共享 |
| 想改服务器地址 | 标题栏「服务器设置」或 `~/.Lugwit/l_notepad_server/server_config.json` |
| 登录/API 不通 | 见第 6 节排查（Lugwit_deploy、server_config.json 残留、nginx 路由） |

## 9. 相关文档

- 《Nginx反向代理机制.md》— 路由与转发
- 《标题栏提供的服务.md》— 客户端标题栏能力
- 《Rez包创建和启动指导文档.md》— rez 包/启动/修饰符/自动下载依赖
- 《Rez_pkg/lugwit_baidu_netdisk.md》— 云端 Depot 多模式存储与迁移
