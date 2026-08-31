# Nginx 反向代理机制

涉及包：

- `l_nginx/999.0` — nginx 预编译包（Windows）+ Python 管理 CLI（`nginx_cli.py`）
- 被代理的各服务：认证 1027 / 网盘 1028 / ChatRoom 1026 / 笔记 8765
- 另有静态文档站 `/docs/`（8080 上的静态目录，非代理，文件在包内 `html/docs`）

## 1. 背景与目标

内网/公网部署了多个 Web 服务（认证、网盘、聊天、笔记），各自监听不同端口：

- 每个服务都要配一次地址端口，客户端难记、难改
- 认证、笔记这类服务直接暴露端口风险高（公网可扫、可打）
- 需要 WebSocket（ChatRoom）、大文件上传（网盘）等特殊能力，各服务自己实现不了

用 **nginx 做统一反向代理**：

- 对外只暴露**一个入口端口 8080**（安全组 / 防火墙只放这一个）
- 各后端只监听 `127.0.0.1`，不可被外部直接访问
- 客户端只需配置"服务器 IP + 8080"，路由 / 前缀全部由 nginx 收敛

## 2. 部署形态

```text
                    ┌──────────────── 服务器 ─────────────────┐
 客户端 (任意机器)   │                                         │
 ── 8080 ──────────▶│  nginx (listen 8080)                    │
 HTTP/WS 统一入口    │   │  /api/v1/*  ──▶ 127.0.0.1:1027 认证  │
                    │   │  /baidu/*   ──▶ 127.0.0.1:1028 网盘  │
                    │   │  /chat/*    ──▶ 127.0.0.1:1026 聊天  │
                    │   │  /note/*    ──▶ 127.0.0.1:8765  笔记 │
                    │   │  /docs/*    ──▶ 静态 html/docs 文档站│
                    │   └────────────────────────────────────  │
                    └─────────────────────────────────────────┘
```

- nginx 与所有后端**同机**部署，`upstream` 里全是 `127.0.0.1:<port>`
- 硬性前提：**后端端口（1027/1028/1026/8765）不对外监听**，只开 8080
- `/docs/*` 不转发，直接读包内 `html/docs/` 静态目录（autoindex 目录浏览，.md 以 text/plain 显示）
- nginx 为 Windows 官方预编译包，`-p <prefix>` 前缀模式运行（配置、日志、pid 全在 runtime 内）

### 2.1 端口总表

| 端口 | 服务 | 监听范围 | 对外可访问 | 代理前缀 |
|------|------|---------|:---:|---------|
| 8080 | nginx 统一入口 + 文档站 | 0.0.0.0 | ✅（唯一放行口） | — |
| 1027 | lugwit_auth 认证服务 | 127.0.0.1 | ❌ | `/api/v1/`、`/api/v1/auth` |
| 1028 | lugwit_baidu_netdisk 网盘 | 127.0.0.1 | ❌ | `/baidu/`、`/api/` |
| 1026 | ChatRoom 聊天 | 127.0.0.1 | ❌ | `/chat/` |
| 8765 | lugwit_note 笔记 | 127.0.0.1 | ❌ | `/note/` |
| 8090 | l_homepage 聚合门户主页 | 127.0.0.1 | ❌ | `/homepage`、`/api/v1/services` |

`/docs/` 挂在 8080 上，**不是代理**：`location /docs/ { root html; autoindex on; }`
（`<prefix>/html/docs`，即包内 `runtime/html/docs`），与上面的 upstream 端口无关。

## 3. 配置文件：`conf/lugwit.conf`

结构：`worker_processes / events / http`，**不包 `events{}` / `http{}` 外层套壳**（prefix 模式兼容 Windows 默认配置）。

```nginx
http {
    include       mime.types;
    client_max_body_size 100g;      # 网盘大文件直传，单文件可能 10G
    proxy_read_timeout   3600s;     # 大文件/WS 长连接超时拉起

    # ===== 上游（后端服务都在本机）=====
    upstream lugwit_auth_backend      { server 127.0.0.1:1027; keepalive 16; }
    upstream lugwit_baidu_backend     { server 127.0.0.1:1028; keepalive 16; }
    upstream chatroom_backend         { server 127.0.0.1:1026; keepalive 16; }
    upstream lugwit_note_backend      { server 127.0.0.1:8765; keepalive 16; }

    # ===== WebSocket 升级 =====
    map $http_upgrade $connection_upgrade {   # 无 Upgrade 请求头 → 空串；
        default upgrade;                      # nginx 不发空头，上游 keepalive 才生效
        ''      '';
    }
    proxy_http_version 1.1;                   # 上游 keepalive 必需
    proxy_set_header Host              $host;
    proxy_set_header Upgrade           $http_upgrade;
    proxy_set_header Connection        $connection_upgrade;

    server {
        listen 8080;
        # ... locations 见下
    }
}
```

## 4. 路由规则表（8080 入口）

| 客户端路径 | 转发目标 | 剥前缀? | 后端收到 | 说明 |
|-----------|----------|:------:|----------|------|
| `/` | 302 → `/homepage` | — | — | 入口根路径直接重定向到门户主页 |
| `/homepage`、`/homepage/*` | l_homepage 8090 | 否 | `/homepage/*` | 门户主页（独立网页登录 fqq/qwer，不用 lugwit_auth） |
| `/api/v1/services/*` | l_homepage 8090 | 否 | `/api/v1/services/*` | 门户主页的服务状态/启停 API |
| `/api/v1/*` | 认证 1027 | 否 | `/api/v1/*` | 认证服务自身路由（登录/账号/收藏/用户等） |
| `/api/v1/auth` | 认证 1027 | 否 | `/api/v1/auth` | 认证登录/登出（比 `/api/v1/` 更具体） |
| `/baidu/*` | 网盘 1028 | 是 | `/*` | 客户端路径带 `/baidu` 前缀 |
| `/api/*` | 网盘 1028 | 否 | `/api/*` | 网盘前端页面里的 API 是**绝对路径**（无前缀，非 `/api/v1/`） |
| `/chat/*` | ChatRoom 1026 | 是 | `/*` | 剥 `/chat` |
| `/note/*` | 笔记 8765 | 是 | `/api/*` | 笔记请求 `/note/api/notes` → 8765 收 `/api/notes` |
| `/docs/*` | 静态文件（包内 html/docs） | 否 | `/docs/*` | 文档站，autoindex 目录浏览，.md 以 text/plain 返回，**公开无需登录** |

```nginx
location = /               { return 302 /homepage; }                 # 根路径跳门户
location /homepage         { proxy_pass http://lugwit_homepage_backend; }  # 门户主页（l_homepage 8090）
location ~ ^/api/v1/services(/|$) { proxy_pass http://lugwit_homepage_backend; }  # 门户状态/启停
location /api/v1/auth      { proxy_pass http://lugwit_auth_backend; } # 认证登录/登出（1027）
location /api/v1/          { proxy_pass http://lugwit_auth_backend; } # 认证业务 API（1027）
location /baidu/           { proxy_pass http://lugwit_baidu_backend/; }  # 带 URI → 剥匹配前缀
location /api/             { proxy_pass http://lugwit_baidu_backend; }   # 网盘根路径 API（1028）
location /chat/            { proxy_pass http://chatroom_backend/; }
location /note/            { proxy_pass http://lugwit_note_backend/; }
location = /nginx-health   { return 200 "ok\n"; }                 # 精确匹配健康检查
location = /docs           { return 302 /docs/; }                 # 补斜杠，避免 autoindex 相对路径错乱
location /docs/            { root html; autoindex on; }           # 静态文档站（<prefix>/html/docs），不代理
```

### 4.1 核心机制：`proxy_pass` 带不带 URI 决定剥不剥前缀

```nginx
location /note/ { proxy_pass http://lugwit_note_backend; }   # 无 URI：/note/api/notes 原样送给 8765
location /note/ { proxy_pass http://lugwit_note_backend/; }  # 有 URI：剥掉 /note/，8765 收到 /api/notes
```

- **不带 URI**（`proxy_pass http://host;` 结尾无斜杠）→ 匹配到的完整路径原样转发
- **带 URI**（`proxy_pass http://host/;` 结尾有斜杠）→ 剥掉 `location` 匹配的前缀，剩余部分转发

"后端收到什么"取决于**后端服务自己的路由**，两条规则对应两种服务形态：

| 后端路由形态 | 客户端写法 | location 写法 |
|-------------|-----------|--------------|
| 自带路径前缀（认证 `/api/v1/*`） | `/api/v1/login` | `proxy_pass http://host;`（不带 URI） |
| 根路径路由（笔记 `/api/notes`） | `/note/api/notes` | `proxy_pass http://host/;`（带 URI 剥 `/note`） |

### 4.2 核心机制：location 匹配优先级（`/api/v1/` 必须先于 `/api/`）

nginx 对 location 的匹配：**带 `^~` / 精确 `=` 优先，普通前缀按最长匹配**（不是配置文件里先到先得，但 `proxy_pass` 的转发目标互斥时，**顺序写错同样出错**）。

本项目里 `/api/` 被网盘占用、`/api/v1/` 是认证的，两者前缀重叠：

```nginx
location /api/v1/ { proxy_pass http://lugwit_auth_backend;  }   # 必须先声明（更长的前缀优先命中）
location /api/    { proxy_pass http://lugwit_baidu_backend; }   # 否则 /api/v1/* 会被 /api/ 抢走
```

客户端绝对路径 `/api/v1/auth/login` 命中 `/api/v1/`（最长前缀），网盘页面自己的绝对路径
`/api/state`、`/api/sync/*` 命中 `/api/`（网盘路由在根路径），互不干扰。

### 4.3 核心机制：WebSocket 升级透传

`map $http_upgrade $connection_upgrade`：

- 客户端带 `Upgrade: websocket` → 实值 `upgrade` → 请求头原样透传 → 上游完成 WS 握手
- 普通 HTTP 请求（无 Upgrade 头）→ 映射为空串 → **nginx 不发空 Connection 头 → 上游 HTTP keepalive 才能生效**（多个连接复用，不被打断）
- 配套：`proxy_http_version 1.1`（HTTP/1.0 无长连接）+ 上游 `keepalive 16`（连接池）

## 5. 客户端如何接入

以 l_notepad（使用 l_qframelesswindow 标题栏的服务器设置）为例：

| 配置键 | 填写值 | 实际请求 | nginx 处理 |
|--------|--------|----------|-----------|
| auth_url | `http://121.196.144.88:8080` | `/api/v1/auth/login` | `/api/v1/` → 认证 1027 |
| auth_route | `/api/v1/auth` | — | — |
| api_url | `http://121.196.144.88:8080/note` | `/note/api/notes` | 剥 `/note` → 笔记 8765 |
| log_server_url | `http://121.196.144.88:8080/note` | `/note/api/logs/...` | 同上 |

规律：**`<nginx入口> + <路由前缀>`**，前缀要能对上 nginx 的 location 且不与别的服务重叠。

## 6. 生命周期管理（`nginx_cli.py`）

统一入口（rez alias）：

```bat
wuwor l_nginx -- nginx_start            :: 启动；端口被占用时交互询问
wuwor l_nginx -- nginx_start --port 8081  :: 换端口启动（生成 override 配置）
wuwor l_nginx -- nginx_start --kill       :: 免交互：结束占用进程后按原端口启动
wuwor l_nginx -- nginx_start --force      :: 跳过冲突预检
wuwor l_nginx -- nginx_start --no-prompt  :: 禁止询问（后台/脚本）
wuwor l_nginx -- nginx_reload             :: 平滑重载（改完 conf 用，不丢请求）
wuwor l_nginx -- nginx_stop               :: 优雅停止；--force 超时强杀
wuwor l_nginx -- nginx_status             :: 状态 + 最近 error.log
wuwor l_nginx -- nginx_check              :: nginx -t 语法校验（改配置先跑）
wuwor l_nginx -- nginx_check --active     :: 校验当前运行实例用的配置
```

### 6.1 端口冲突交互

入口 8080 被别的进程占用时：

- 交互询问：`[1] 结束占用进程 / [2] 换端口 / [3] 取消`，**5 秒无操作自动选 [1]**（可 `--no-prompt` / `--port` / `--kill` 显式指定，避免后台误杀）
- 选 [2] 换端口 → 基于包内主配置生成 **override 副本**（`runtime/conf/lugwit.override.conf`），只替换 listen 端口，**绝不改包内 conf/**

### 6.2 运行时状态：`.active_conf` + pid

- `.active_conf` 记录"当前实例到底用哪份 conf"（包内主配置 or override）——reload / stop 必须与 start 用**同一份**
- pid 文件 `runtime/logs/nginx.pid`；status/stop 时自动清理陈旧的 pid 文件
- **`-c` 必须带**：`nginx -s reload/stop` 也要先解析配置才能定位 pid 文件，不传 `-c` 会去解析 `<prefix>/conf/nginx.conf` 直接 `[emerg]` 失败
- **启动必须 detached**：Windows 下 nginx.exe 本身就是前台 master 进程，用 `subprocess.run` 会等它退出卡死，超时杀掉还会留孤儿 worker（`nginx_cli` 已用 `DETACHED_PROCESS` 处理）

### 6.3 运行时目录

```text
l_nginx/999.0/
├── conf/lugwit.conf            # 包内主配置（只读产物，property 部署）
├── bin/nginx.exe               # Windows 官方预编译包
└── runtime/                    # 运行期 prefix（日志 / pid / 临时文件）
    ├── conf/.active_conf       # 当前生效 conf 的路径记录
    ├── conf/lugwit.override.conf   # 换端口生成的副本（自动生成，勿手改）
    └── logs/                   # error.log / access.log / nginx.pid / nginx-cli.log
```

## 7. 安全建议

- 防火墙 / 安全组**只放行 8080**；后端端口（1027/1028/1026/8765/8090）只监听 `127.0.0.1`
- 后端服务启动参数里绑定 `127.0.0.1`（如 `backend_server --host 127.0.0.1`），双保险
- 健康检查暴露一个只读端点 `GET /nginx-health`（`location = /nginx-health`），供监控探测，不打到任何后端
- **nginx 入口不做 Basic Auth**：业务路由（`/note` `/baidu` `/chat` `/api`）由各上游服务自己的认证负责；门户主页 `/homepage` 由 l_homepage 提供**独立网页登录**（账号 fqq/qwer，可环境变量覆盖，不查 lugwit_auth 账号库），文档站 `/docs/` 公开无需登录

## 8. 排查手段

1. 改完配置先 `wuwor l_nginx -- nginx_check`（语法校验，不生效），再 `nginx_reload`
2. 验证各路由：

```bat
curl http://127.0.0.1:8080/nginx-health          :: ok（nginx 本身活着）
curl http://127.0.0.1:8080/api/v1/health         :: 认证服务
curl http://127.0.0.1:8080/note/api/health       :: 笔记后端 {"ok":true}
curl http://127.0.0.1:8080/note/api/notes        :: 应 401（需登录），证明剥前缀正确
curl http://127.0.0.1:8080/docs/                  :: 文档站目录列表（200 + autoindex HTML）
```

3. 看日志：`runtime/logs/error.log`（nginx 层报错）、`runtime/logs/access.log`（请求命中哪个 location）、`nginx-cli.log`（CLI 操作记录）
4. 新加后端服务的接入步骤：后端同机起服务 → `conf/lugwit.conf` 加 `upstream` + `location`（注意剥前缀规则与 location 顺序）→ `nginx_check` → `nginx_reload`

## 9. 已知注意事项

- 改 conf 后必须 `nginx_reload` 才生效（reload 是平滑切换 worker，不丢请求）
- 换入口端口是"临时 override"，重启 nginx 后不再记录（`.active_conf` 清空回落到主配置）
- 新增服务时路径前缀**不能与现有 location 前缀重叠**（`/api/` 已被网盘占用，笔记服务因此用 `/note/`）
- `/docs/` 文档站内容是**打包时的静态快照**（复制自 `rez-package-source/Doc` 目录）；Doc 新增/修改文档后需同步复制到 `l_nginx/999.0/runtime/html/docs/`（静态文件改动即时生效，无需 reload）

## 10. 近期变更记录（2026-08-31）

### 10.1 移除 nginx 层 Basic Auth

之前 `server` 块开了全局 `auth_basic "Lugwit Portal"` + `auth_basic_user_file ../conf/.htpasswd`，只对 `/`、`/api/v1/auth` 等少数 location `auth_basic off`，导致 `/note/`、`/baidu/`、`/chat/`、`/api/`、`/docs/` 都要 Basic Auth，而笔记等客户端不携带 Basic Auth 凭据 → nginx 直接返回 401，**笔记程序无法登录**。

处理：移除全部 `auth_basic` / `auth_basic_user_file`，业务路由交给各上游自己的认证；`/docs/` 文档站公开。

> 遗留：`conf/.htpasswd` 与 `gen_htpasswd.py` 不再被 nginx 使用（`gen_htpasswd.py` 的 apr1 算法本身也是简化错误的，勿再用于生成；如将来要 Basic Auth 需重写）。

### 10.2 门户主页 `/homepage` + 独立网页登录

新增 l_homepage 上游（`127.0.0.1:8090`）：

- `location = /` → `return 302 /homepage;`（入口根路径跳转到门户主页）
- `location /homepage` → l_homepage(8090)，l_homepage 新增路由：
  - `GET /homepage`：未登录 302 `/homepage/login`，已登录显示聚合主页
  - `GET/POST /homepage/login`：网页表单登录（账号 fqq/qwer，**独立账号，不查 lugwit_auth**，可用环境变量 `L_HOMEPAGE_USER`/`L_HOMEPAGE_PASS` 覆盖）
  - `POST /homepage/logout`：清除登录 cookie
- 会话：HMAC-SHA256 自签名 cookie（`lugwit_homepage_auth`，HttpOnly，标准库实现，无第三方依赖）

> 提示：`templates/login.html` 为登录表单（fetch 提交 JSON，未用 `Form`，避免依赖 `python-multipart`）。

### 10.3 修复 `/api/v1/` 被 `/api/` 劫持

`location /api/` 会把所有 `/api/*` 转到网盘(1028)。笔记登录后要拉账号/收藏/用户数据，请求 `/api/v1/accounts`、`/api/v1/fav-items`、`/api/v1/users` 等，本应到认证(1027)，却被 `location /api/` 劫持到网盘（网盘未启动时 → **502**，客户端反复重试导致界面卡顿）。

修复：新增更具体的 `location /api/v1/ → lugwit_auth_backend(1027)`（最长前缀优先命中），`/api/v1/*` 归认证，网盘的 `/api/state`、`/api/sync/*` 等根路径 API 仍归网盘。

### 10.4 运维提示：包目录内残留 `l_homepage/` 会破坏 rez 解析

`l_nginx` 的 `package.py` 曾把 `L_HOMEPAGE_ROOT` 指到 `{root}\..\l_homepage`，运行期会在 `l_nginx/` 包目录下生成 `l_homepage/runtime/`（pid/日志）。rez 会把 `l_nginx/l_homepage/` 误当成 `l_nginx` 的一个版本（无 `package.py`），导致 `wuwor l_nginx -- ...` 报 `Missing package definition file` 无法解析。

处理：把残留目录移出包目录（如 `_tmp/`）。注意排查 `l_nginx/` 下是否又出现 `l_homepage/` 这类非版本子目录。