# l_notepad 拆包架构与数据目录

> 更新：2026-09-03
> 背景：l_notepad 由单一包拆分为 **服务端 + 客户端**，数据根随 `data_dir` 模板变化，并存在启动迁移机制。

## 1. 拆包结构

原 `l_notepad` 拆成三个 rez 包（旧包 `l_notepad` 保留，仅作历史/迁移参考）：

| 包 | 角色 | 入口 alias | 说明 |
|----|------|-----------|------|
| `l_notepad_server` | 服务端（FastAPI/uvicorn，8765） | `l_notepad_api` | 笔记业务 API + Web 页面；`l_notepad_api_reload` 带 `--reload` |
| `l_notepad_client` | 客户端（Qt 桌面） | `l_notepad` / `l_notepad_ori` / `l_notepad_with_api` | 标题栏、登录、脚本编辑器等 |
| `l_notepad` | 旧单体包 | — | 拆包前的代码，保留供对照/迁移 |

关键入口（`l_notepad_server/999.0/package.py`）：

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

## 2. 端口与网络拓扑

- 服务端 `backend_server.py`：`--port` 默认 `L_NOTEPAD_PORT`（缺省 **8765**），只监听 `127.0.0.1`
- 统一入口走 **nginx 8080**：`/note/*` 剥前缀 → 8765（详见《笔记程序登录统一走Nginx代理.md》《Nginx反向代理机制.md》）
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
> 若服务已启动后再拷入笔记文件，需**重启服务**让它重新执行迁移，否则新文件不会出现在列表里
> （本次排查中笔记列表为空的根因：服务器在拷贝旧笔记前启动，迁移时目录为空）。

## 5. 拆包/换机后数据怎么来

服务端不会自动从旧包目录 `~/.Lugwit/l_notepad/` 搬数据（`paths.py` 的 `move_dir_contents`
只处理**程序包内**的旧位置，如 `_PKG_DIR/notepad_list`，不迁移另一个数据根目录）。因此：

1. **手动拷贝**旧笔记到新数据根（本次采用）：
   ```bat
   xcopy /E /Y "%USERPROFILE%\.Lugwit\l_notepad\notepad_list" "%USERPROFILE%\.Lugwit\l_notepad_server\notepad_list"
   ```
2. **重启服务**（主页「L Notepad 笔记」卡片 → 热更新/重启，或 `l_notepad_api_reload`），
   让 `migrate_legacy_notes` 把已拷入的笔记注册给 admin01。
3. 验证：`GET http://127.0.0.1:8080/note/api/notes`（需 token）应返回笔记列表。
   归属 admin01，与登录用户一致即可正常查看/编辑。

若旧数据原本就存在（非空），且只换包名不换机器，也可手动把整个数据根迁过去，注意保留
`server_config.json` 的 host 配置与 `auth_token.json`。

## 6. 排查速查

| 现象 | 排查 |
|------|------|
| 笔记列表空 | 数据根是否 `~/.Lugwit/l_notepad_server`（非旧包）；`notepad_list/` 是否有文件；是否在拷入后重启过服务 |
| 归属不对/看不到别人笔记 | `notepad.sqlite3` 归属表；`migrate_legacy_notes` 注册为 admin01 且设为共享 |
| 想改服务器地址 | 标题栏「服务器设置」或 `~/.Lugwit/l_notepad_server/server_config.json` |

## 7. 相关文档

- 《笔记程序登录统一走Nginx代理.md》— 登录/认证/API 收敛到 nginx 8080
- 《Nginx反向代理机制.md》— 路由与转发
- 《标题栏提供的服务.md》— 客户端标题栏能力
- 《Rez包创建和启动指导文档.md》— rez 包/启动/修饰符
