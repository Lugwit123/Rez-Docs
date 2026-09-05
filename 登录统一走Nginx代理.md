# 登录统一走 Nginx 代理

> 各客户端程序的登录/认证/账号/收藏等业务 API，全部收敛到 **nginx 统一入口 8080** 反向代理，
> 不直连后端端口（认证 1027、笔记 8765、网盘 1028 等只监听 `127.0.0.1`，不对外暴露）。
>
> 更新：2026-09-03

---

## 1. 背景与目标

- **对外只暴露 `8080`**，客户端只需配"服务器 IP + 8080"，无需知道各后端端口
- 路由前缀由 nginx 收敛（见《Nginx反向代理机制.md》第 4 节路由规则表）：
  - `/api/v1/*` → lugwit_auth（认证 1027），登录路由即 `/api/v1/auth/login`
  - `/note/*` → 笔记后端 8765（剥前缀，8765 收到 `/api/*`）
  - `/baidu/*` → 网盘后端 1028（剥前缀）
  - `/chat/*` → ChatRoom 1026（剥前缀）
- 开发机与公网部署机通过系统级环境变量 `Lugwit_deploy` 自动区分，无需每台手动改 host

## 2. 部署形态

```text
客户端 ──8080──▶ nginx 统一入口
                 │ /api/v1/auth/login ──▶ 127.0.0.1:1027 认证（登录）
                 │ /api/v1/*         ──▶ 127.0.0.1:1027 认证（账号/收藏/用户）
                 │ /note/api/*       ──▶ 127.0.0.1:8765  笔记（剥 /note）
                 │ /baidu/api/*      ──▶ 127.0.0.1:1028  网盘（剥 /baidu）
                 │ /chat/*           ──▶ 127.0.0.1:1026  ChatRoom（剥 /chat）
```

详见《Nginx反向代理机制.md》第 4、5 节。

## 3. 通用配置项

客户端程序经标题栏库 `l_qframelesswindow` 的 `ServerConfigStore`（`ServerSettingsDialog` 对话框）统一维护
服务器地址与路由，三层解析：**UI 持久化 > 环境变量 > 默认值**。

| 配置键 | 含义 | 生产默认（公网机） | 开发默认（开发机） |
|--------|------|--------------------|--------------------|
| `auth_url` | 认证服务地址（nginx 入口） | `http://121.196.144.88:8080` | `http://127.0.0.1:8080` |
| `auth_route` | 认证路由前缀 | `/api/v1/auth` | `/api/v1/auth` |
| `api_url` | 业务 API（各自前缀） | 见下 | 见下 |
| `log_server_url` | 远端日志服务 | 同 `api_url` | 同 `api_url` |

- 登录 / 认证端点拼接规则：`auth_url + auth_route + "/..."`（如公网登录路由 =
  `http://121.196.144.88:8080/api/v1/auth/login`）
- 业务 API 各自带前缀：笔记 `api_url = <host>/note`、网盘 `<host>/baidu`、聊天 `<host>/chat`

### 3.1 笔记程序示例

| 配置键 | 生产默认 | 开发默认 |
|--------|----------|----------|
| `auth_url` | `http://121.196.144.88:8080` | `http://127.0.0.1:8080` |
| `auth_route` | `/api/v1/auth` | `/api/v1/auth` |
| `api_url` | `http://121.196.144.88:8080/note` | `http://127.0.0.1:8080/note` |
| `log_server_url` | 同上 | 同上 |

## 4. 配置优先级与来源

读取优先级：**UI 持久化 > 环境变量 > 默认值**（由 `l_qframelesswindow` 的 `ServerConfigStore` 实现）。

1. **UI 持久化**：`~/.Lugwit/<包名>/server_config.json`（标题栏「服务器设置」写出的配置，会盖过一切）
   - 笔记：`~/.Lugwit/l_notepad_server/server_config.json`
2. **环境变量**：`LUGWIT_AUTH_URL` / `LUGWIT_AUTH_ROUTE` / `L_NOTEPAD_API_URL` / `L_NOTEPAD_LOG_SERVER` 等
3. **默认值**：各程序 `server_config.py` 里的 `_DEFAULTS`，**随 `Lugwit_deploy` 自动区分开发/公网机**

> ⚠️ 若要 `Lugwit_deploy` 生效，必须保证 `server_config.json` 里**没有**持久化的 host 键
> （`auth_url` / `api_url` / `log_server_url`），否则持久化会盖过机器类型默认值。
> 可用标题栏设置或 `store.clear(["auth_url","api_url","log_server_url"])` 清掉。

## 5. `Lugwit_deploy` 自动区分开发机 / 公网部署机

系统级环境变量 `Lugwit_deploy`：

- `Lugwit_deploy=1`（或 `true/yes/on`）→ **公网部署机**，默认走生产 nginx 8080 统一入口
- 缺省 / `0` → **开发机**，默认走本机 nginx `127.0.0.1:8080`

```python
# <包>/server_config.py（与 cloud_sync 同规则）
def _is_prod() -> bool:
    return os.environ.get("Lugwit_deploy", "0").strip().lower() in ("1", "true", "yes", "on")

_HOST_PREFIX = "http://121.196.144.88:8080" if _is_prod() else "http://127.0.0.1:8080"
_DEFAULTS = {
    "auth_url": _HOST_PREFIX,
    "auth_route": "/api/v1/auth",
    "api_url": _HOST_PREFIX + "/note",       # 各程序换成自己的前缀
    "log_server_url": _HOST_PREFIX + "/note",
}
```

`Lugwit_deploy` 需在**进程启动前**设好（系统级环境变量），模块导入时即决定默认值。

### 5.1 解析结果对照

| 机器 | `Lugwit_deploy` | `auth_url` | 登录路由 |
|------|:---:|-------------------------|--------------------------------------------|
| 开发机 | 缺省/0 | `http://127.0.0.1:8080` | `http://127.0.0.1:8080/api/v1/auth/login` |
| 公网部署机 | 1 | `http://121.196.144.88:8080` | `http://121.196.144.88:8080/api/v1/auth/login` |

## 6. 各入口如何接入

| 入口 | 代码位置 | 行为 |
|------|----------|------|
| 桌面登录对话框 | 各客户端 `LoginDialog` | POST `auth_url + auth_route + "/login"`，成功后保存 token |
| 桌面 token 验证 | `auth.py verify_token` | POST `auth_url + /api/v1/auth/verify` |
| 桌面账号/收藏 | `api_client.py` / `account_favorites_widget.py` | `auth_url + /api/v1/...` |
| Web 登录页 | 各服务 `routers/web.py` `/api/auth/login` | 调认证服务，设置 HttpOnly cookie |
| Web 未登录重定向 | 各服务 auth 中间件 | 页面重定向 `{前缀}/login`，API 返回 401 |
| 网盘页面 | `__API_BASE__`（nginx `/baidu` 前缀） | 经 `/baidu/api/*` 访问 |

> 标题栏库 `l_qframelesswindow` 提供统一的 `ServerConfigStore` / `ServerSettingsDialog`，
> 各程序接入方式见《标题栏提供的服务.md》。

## 7. 排查

1. 先确认当前机器 `Lugwit_deploy` 与实际 host 是否匹配：
   - 开发机应为缺省/0，公网部署机应为 1
2. 检查持久化配置是否残留 host 键：
   ```bat
   type %USERPROFILE%\.Lugwit\l_notepad_server\server_config.json
   ```
   若含 `auth_url` / `api_url` / `log_server_url`，用标题栏「服务器设置」清掉或改对。
3. 验证 nginx 路由（见《Nginx反向代理机制.md》第 8 节）：
   ```bat
   curl http://127.0.0.1:8080/nginx-health
   curl http://127.0.0.1:8080/api/v1/health        :: 认证服务
   curl http://127.0.0.1:8080/note/api/health      :: 笔记后端
   curl http://127.0.0.1:8080/note/api/notes       :: 应 401（需登录），证明剥前缀正确
   curl http://127.0.0.1:8080/baidu/api/health     :: 网盘后端
   ```

## 8. 相关文档

- 《Nginx反向代理机制.md》— 路由规则表、proxy_pass 剥前缀机制、排查
- 《标题栏提供的服务.md》— ServerConfigStore / ServerSettingsDialog 通用接入
- 《Rez_pkg/l_notepad_server.md》— 笔记程序登录接入示例
