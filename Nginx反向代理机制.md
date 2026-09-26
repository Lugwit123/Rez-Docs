# Nginx 反向代理机制

状态：**网络与路由现行主文档，2026-09-17 更新**。HTTPS/TLS 证书与续期见[Rez_pkg/HTTPS证书与域名申请总结.md](Rez_pkg/HTTPS证书与域名申请总结.md)；客户端登录入口配置见[标题栏提供的服务.md](标题栏提供的服务.md)。

> 2026-09-17 要点：**公网入口只有 443**（`conf/https.conf` 两个 server 块）；8080 收成 `listen 127.0.0.1:8080`（仅本机调试）；路由集合统一收敛到 **`conf/routes.conf`**（唯一一份 location 定义，被 443 与 8080 共同 `include`）。`duckdns.org` 域名已被国内网络按 **SNI 关键字拦截**、弃用（详见 §11），客户端与运维工具一律走 `https://121.196.144.88/...`。

> 凭据规则：文档不得记录明文账户密码；登录和服务调用使用受控账号凭据。

涉及包：

- `l_nginx/999.0` — nginx 预编译包（Windows）+ Python 管理 CLI（`nginx_cli.py`）
- 被代理的各服务：认证 1027 / 网盘 1028 / ChatRoom 1026 / 笔记 8765 / 脚本编辑器 8764 …
- 另有静态文档站 `/docs/`（`location /docs/` 读包内 `html/docs` 静态目录，非代理）

## 1. 背景与目标

内网/公网部署了多个 Web 服务（认证、网盘、聊天、笔记），各自监听不同端口：

- 每个服务都要配一次地址端口，客户端难记、难改
- 认证、笔记这类服务直接暴露端口风险高（公网可扫、可打）
- 需要 WebSocket（ChatRoom）、大文件上传（网盘）等特殊能力，各服务自己实现不了

用 **nginx 做统一反向代理**：

- 对外现行入口为 **443 HTTPS**；8080 已收为 `127.0.0.1:8080`，仅供本机诊断（443 与它共用同一份 `conf/routes.conf`，各自直接命中 location，443 **不再**经 8080 转发）
- 各后端只监听 `127.0.0.1`，不可被外部直接访问
- 客户端使用 HTTPS 入口（**IP 自签证书，无域名**）；路由 / 前缀全部由 nginx 收敛

## 2. 部署形态

```text
                    ┌──────────────── 服务器 ─────────────────┐
 客户端 (任意机器)   │                                         │
 ── 443 HTTPS ─────▶│  nginx (listen 443 ssl)  ← 公网入口      │
 (IP 自签证书)      │   │  /api/v1/*        ──▶ 127.0.0.1:1027 认证│
                    │   │  /baidu/*         ──▶ 127.0.0.1:1028 网盘│
                    │   │  /chat/*          ──▶ 127.0.0.1:1026 聊天│
                    │   │  /note/*          ──▶ 127.0.0.1:8765 笔记│
                    │   │  /script_editor/* ──▶ 127.0.0.1:8764 编辑器│
                    │   │  /docs/*          ──▶ 静态 html/docs 文档站│
                    │   └────────────────────────────────────  │
 本机调试           │  nginx (listen 127.0.0.1:8080)           │
 ── 8080 ──────────▶│   同一份 conf/routes.conf（仅回环可用）   │
                    └─────────────────────────────────────────┘
```

- nginx 与所有后端**同机**部署，`upstream` 里全是 `127.0.0.1:<port>`
- 硬性前提：**后端端口不对外监听**；对外只开 **443**（8080 只绑回环）
- 路由集合只有一份：**`conf/routes.conf`**，被 443 的 `conf/https_common.conf` 与 8080 的 server 块共同 `include`
- `/docs/*` 不转发，直接读包内 `html/docs/` 静态目录（autoindex 目录浏览，.md 以 text/plain 显示）
- nginx 为 Windows 官方预编译包，`-p <prefix>` 前缀模式运行（配置、日志、pid 全在 runtime 内）

### 2.1 端口总表

| 端口 | 服务 | 监听范围 | 对外可访问 | 代理前缀 |
|------|------|---------|:---:|---------|
| 443 | nginx HTTPS 入口（文档站也在此） | 0.0.0.0 | ✅（**公网唯一入口**，IP 自签证书） | — |
| 8080 | nginx 备用/调试口 | 127.0.0.1 | ❌（仅本机） | —（同一份 routes.conf） |
| 1027 | lugwit_auth 认证服务 | 127.0.0.1 | ❌ | `/api/v1/`、`/api/v1/auth`、`/login`、`/auth/` |
| 1028 | lugwit_baidu_netdisk 网盘 | 127.0.0.1 | ❌ | `/baidu/`（剥前缀）、`/api/` |
| 1026 | ChatRoom 聊天 | 127.0.0.1 | ❌ | `/chat/` |
| 1025 | ChatRoom 网页前端（Vite） | 127.0.0.1 | ❌ | `/chatfront/` |
| 8765 | lugwit_note 笔记 | 127.0.0.1 | ❌ | `/note/` |
| 8764 | l_script_editor 脚本编辑器 HTTP 服务 | 127.0.0.1 | ❌ | `/script_editor/`（跨机需令牌） |
| 8090 | l_homepage 聚合门户主页 | 127.0.0.1 | ❌ | `/homepage`、`/api/v1/services` |
| 8100 | l_mindmap 独立脑图 | 127.0.0.1 | ❌ | `/mindmap/` |
| 8110 | l_mindmap_mmd mm.js 脑图 | 127.0.0.1 | ❌ | `/mindmap-mmd/` |
| 1250 | l_agent_chat 对话服务 | 127.0.0.1 | ❌ | `/agent_chat/` |
| 8462 | l_model_hub LLM 模型注册中心 | 127.0.0.1 | ❌ | `/model_hub/` |
| 1234 | l_WChat 推送服务 | 127.0.0.1 | ❌ | `/l_wchat/`（App 现走 `https://121.196.144.88/l_wchat/`，壳里 `ServerSwitchPlugin.apiBase()` 对无端口的 nginx 入口自动补 `/l_wchat` 前缀） |

`/docs/` 挂在入口的 `location /docs/` 上，**不是代理**：`location /docs/ { root html; autoindex on; }`
（`<prefix>/html/docs`，即包内 `runtime/html/docs`），与上面的 upstream 端口无关。

## 3. 配置文件：`conf/lugwit.conf`

结构：`worker_processes / events / http`，**不包 `events{}` / `http{}` 外层套壳**（prefix 模式兼容 Windows 默认配置）。`http{}` 里 `include https.conf;`（443 入口），8080 的 server 块里 `include routes.conf;`。

```nginx
http {
    include       mime.types;
    # 大文件上传：相册视频最大 500MB（超 500MB 由客户端压缩）；网盘直传单文件可能 10G
    client_max_body_size   100g;
    client_body_buffer_size 16m;
    client_body_timeout    3600s;
    send_timeout           3600s;
    proxy_read_timeout     3600s;     # 大文件/WS 长连接超时拉起
    proxy_send_timeout     3600s;
    absolute_redirect      off;       # 相对 302 不被拼成 http://<IP>:8080/...（见 §4 注）

    # ===== 上游（后端服务都在本机）=====
    upstream lugwit_auth_backend          { server 127.0.0.1:1027; keepalive 16; }
    upstream lugwit_baidu_backend         { server 127.0.0.1:1028; keepalive 16; }
    upstream chatroom_backend             { server 127.0.0.1:1026; keepalive 16; }
    upstream lugwit_note_backend          { server 127.0.0.1:8765; keepalive 16; }
    upstream lugwit_script_editor_backend { server 127.0.0.1:8764; keepalive 16; }
    # 另有 l_homepage:8090 / l_agent_chat:1250 / l_mindmap:8100 /
    #      l_mindmap_mmd:8110 / l_model_hub:8462 / l_WChat:1234 / ChatRoom 前端:1025 …

    # ===== WebSocket 升级 =====
    map $http_upgrade $connection_upgrade {   # 无 Upgrade 请求头 → 空串；
        default upgrade;                      # nginx 不发空头，上游 keepalive 才生效
        ''      '';
    }
    proxy_http_version 1.1;                   # 上游 keepalive 必需
    proxy_set_header Host              $host;
    proxy_set_header Upgrade           $http_upgrade;
    proxy_set_header Connection        $connection_upgrade;

    # ===== HTTPS 入口（443，见 conf/https.conf）=====
    include https.conf;

    server {
        listen 127.0.0.1:8080;                # 仅本机调试口
        server_name _;
        include routes.conf;                  # 与 443 共用同一份路由表
    }
}
```

## 4. 路由规则表（443 公网入口 / 8080 回环调试口，共用 `conf/routes.conf`）

> 路由集合的**唯一来源是 `conf/routes.conf`**：`conf/https_common.conf`（被 `conf/https.conf` 两个 443 server 块 `include`）与 `conf/lugwit.conf` 的 8080 server 块都 `include` 它。改路由只改这一份。

| 客户端路径 | 转发目标 | 剥前缀? | 后端收到 | 说明 |
|-----------|----------|:------:|----------|------|
| `/nginx-health` | nginx 自身 | — | `200 ok` | 精确匹配健康检查 |
| `/`（精确） | — | — | 302 → `/homepage` | 入口根路径跳门户 |
| `/`（其余） | l_homepage 8090 | 否 | `/...` | 门户主页兜底 |
| `/homepage`、`/homepage/*` | l_homepage 8090 | 否 | `/homepage/*` | 门户主页（独立网页登录，使用受控账号凭据） |
| `/svc-icon/*` | l_homepage 8090 | 否 | 现生成 SVG | 各服务卡片图标（绝对路径） |
| `/api/v1/services(/|$)` | l_homepage 8090 | 否 | 原样 | 门户服务状态/启停 API |
| `/api/v1/homepage(/|$)` | l_homepage 8090 | 否 | 原样 | 门户自身专属接口（如重启） |
| `/api/v1/nginx(/|$)` | l_homepage 8090 | 否 | 原样 | 门户「重载 Nginx」按钮 |
| `/api/v1/auth` | 认证 1027 | 否 | 原样 | 认证登录/登出 |
| `/login`、`/login/*` | 认证 1027 | 否 | 原样 | 统一登录页（支持 `next` 回跳） |
| `/api/v1/*` | 认证 1027 | 否 | `/api/v1/*` | 认证业务 API（账号/收藏/用户） |
| `/baidu/*` | 网盘 1028 | 是 | `/*` | 客户端路径带 `/baidu` 前缀 |
| `/api/*` | 网盘 1028 | 否 | `/api/*` | 网盘前端 API 是根路径（非 `/api/v1/`） |
| `/chat/*` | ChatRoom 1026 | 是 | `/*` | 剥 `/chat` |
| `/note/*` | 笔记 8765 | 是 | `/api/*` | `/note/api/notes` → 8765 收 `/api/notes`（带 `X-Forwarded-Prefix: /note`） |
| `/agent_chat/*` | l_agent_chat 1250 | 是 | `/*` | 剥 `/agent_chat` |
| `/mindmap/*` | l_mindmap 8100 | 是 | `/*` | 剥 `/mindmap` |
| `/mindmap-mmd/*` | l_mindmap_mmd 8110 | 是 | `/*` | 剥 `/mindmap-mmd` |
| `/auth/*` | 认证 1027 | 是 | `/*` | 用户中心主页（剥 `/auth`） |
| `/chatfront/*` | ChatRoom 前端 1025 | 是 | `/*` | 剥 `/chatfront` |
| `/l_wchat`（精确） | — | — | 302 → `/l_wchat/` | 补斜杠 |
| `/l_wchat/*` | l_WChat 1234 | 是 | `/*` | 剥 `/l_wchat`（带 `X-Forwarded-Prefix`） |
| `/script_editor`（精确） | — | — | 302 → `/script_editor/docs` | 脚本编辑器入口 |
| `/script_editor/*` | l_script_editor 8764 | 是 | `/*` | 远程执行服务，**跨机必须带令牌** |
| `/model_hub`、`/model_hub/` | — | — | 302 → `/model_hub/index` | 模型中心入口 |
| `/model_hub/*` | l_model_hub 8462 | 是 | `/*` | 剥 `/model_hub`（SSE 透传） |
| `/docs`（精确） | — | — | 302 → `/docs/` | 补斜杠 |
| `/docs/*` | 静态文件（包内 html/docs） | 否 | `/docs/*` | 文档站，autoindex，.md→text/plain，**公开无需登录** |

```nginx
location = /nginx-health          { return 200 "ok\n"; }
location = /                      { return 302 /homepage; }
location /homepage                { proxy_pass http://lugwit_homepage_backend; }
location /svc-icon/               { proxy_pass http://lugwit_homepage_backend; }
location /                        { proxy_pass http://lugwit_homepage_backend; }             # 门户兜底
location ~ ^/api/v1/services(/|$) { proxy_pass http://lugwit_homepage_backend; }
location ~ ^/api/v1/homepage(/|$) { proxy_pass http://lugwit_homepage_backend; }
location ~ ^/api/v1/nginx(/|$)    { proxy_pass http://lugwit_homepage_backend; }
location /api/v1/auth             { proxy_pass http://lugwit_auth_backend; }
location = /login                 { proxy_pass http://lugwit_auth_backend; }
location /login/                  { proxy_pass http://lugwit_auth_backend; }
location /api/v1/                 { proxy_pass http://lugwit_auth_backend; }   # 认证（1027）
location /baidu/                  { proxy_pass http://lugwit_baidu_backend/; } # 带 URI → 剥匹配前缀（1028）
location /api/                    { proxy_pass http://lugwit_baidu_backend; }  # 网盘根路径 API（1028）
location /chat/                   { proxy_pass http://chatroom_backend/; }
location /note/                   { proxy_pass http://lugwit_note_backend/; }
location /agent_chat/             { proxy_pass http://lugwit_agent_backend/; }
location /mindmap/                { proxy_pass http://lugwit_mindmap_backend/; }
location /mindmap-mmd/            { proxy_pass http://lugwit_mindmap_mmd_backend/; }
location /auth/                   { proxy_pass http://lugwit_auth_page_backend/; }
location /chatfront/              { proxy_pass http://chatroom_frontend_backend/; }
location = /l_wchat               { return 302 /l_wchat/; }
location /l_wchat/                { proxy_pass http://lugwit_wchat_backend/; }
location = /script_editor         { return 302 /script_editor/docs; }
location /script_editor/          { proxy_pass http://lugwit_script_editor_backend/; }
location = /model_hub             { return 302 /model_hub/index; }
location = /model_hub/            { return 302 /model_hub/index; }
location /model_hub/              { proxy_pass http://lugwit_model_hub_backend/; }
location = /docs                  { return 302 /docs/; }
location /docs/                   { root html; autoindex on; }
```

> ⚠️ `absolute_redirect off;` **必须开**（2026-09-16 实测）：否则 nginx 会用绝对重定向把 `https://<IP>/` 302 成 `http://<IP>:8080/homepage`，而 8080 只绑回环 → 首页对外打不开。该指令放在 `http{}` 级，443 两个 server 块与 8080 调试口一起生效。

> 📦 **大文件上传在 location 上再写一遍**（2026-09-26）：`/l_wchat/` 与 `/baidu/` 两个 location 各自显式带
> `client_max_body_size 100g;` + **`proxy_request_buffering off;`**。前者防将来 http 级被改；后者让 nginx
> **边收边转**——默认行为是先把整份 body 落到 `client_body_temp` 再转发，500MB 视频等于多一份磁盘 I/O 与首字节延迟。
> 相册视频（≤500MB 原样上传、>500MB 客户端实时压缩）与网盘大文件直传都吃这两条。
> 验证：12MB mp4 走 `http://127.0.0.1:8080/l_wchat/api/album/upload` → 200，`size` 与实际字节一致。

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

#### 4.1.1 网盘（1028）的前缀适配：middleware 自剥，勿用 uvicorn root_path

网盘页面 `__API_BASE__="/baidu"`（取 `LUGWIT_NETDISK_PREFIX` env），前端 fetch `/baidu/api/*`。
上游适配靠网盘自身 `_strip_proxy_prefix` middleware（web_server.py）剥 `/baidu` 前缀，因此：

- **经 nginx**：`location /baidu/` 带 URI 剥前缀转发（收 `/api/*`），middleware 无操作 → 正常
- **浏览器直连** `http://host:1028/baidu/...`：middleware 剥前缀 → 正常（两种访问方式等价）

> ⚠️ **不要**把前缀传给 uvicorn `root_path`：uvicorn ≥0.38 会把 root_path prepend 进
> scope.path（starlette 再剥一次）。经 nginx 恰好抵消，但直连时页面拼出的 `/baidu/api/*`
> 会被再 prepend 成 `/baidu/baidu/*` → 全部 404（2026-09 已踩坑并改掉）。

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

以 l_notepad_client（使用 l_qframelesswindow 标题栏的服务器设置）为例：

| 配置键 | 填写值 | 实际请求 | nginx 处理 |
|--------|--------|----------|-----------|
| auth_url | `https://121.196.144.88` | `/api/v1/auth/login` | `/api/v1/` → 认证 1027 |
| auth_route | `/api/v1/auth` | — | — |
| api_url | `https://121.196.144.88/note` | `/note/api/notes` | 剥 `/note` → 笔记 8765 |
| log_server_url | `https://121.196.144.88/note` | `/note/api/logs/...` | 同上 |

规律：**`<nginx入口> + <路由前缀>`**，前缀要能对上 nginx 的 location 且不与别的服务重叠。
（8080 只绑回环、仅本机可用；对外一律走 `https://121.196.144.88`（443），2026-09-16/17，见 §2.1、§11）

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

本机调试口 8080 被别的进程占用时：

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

- 防火墙 / 安全组**只放行 443**（2026-09-17 更正：8080 已收成 `listen 127.0.0.1:8080`，只供本机调试，不再是对外入口）；后端端口（1027/1028/1026/8765/8764/8090/8100/8110/1250/8462/1234/1025）只监听 `127.0.0.1`
- 后端服务启动参数里绑定 `127.0.0.1`（如 `backend_server --host 127.0.0.1`），双保险
- 健康检查暴露一个只读端点 `GET /nginx-health`（`location = /nginx-health`），供监控探测，不打到任何后端
- **nginx 入口不做 Basic Auth**：业务路由（`/note` `/baidu` `/chat` `/api`）由各上游服务自己的认证负责；门户主页 `/homepage` 由 l_homepage 提供独立网页登录（凭据由受控配置提供，不查 lugwit_auth 账号库），文档站 `/docs/` 公开无需登录
- **收/放端口时的连带清单**（2026-09-19 补充，改暴露面前逐条核对）：
  1. `conf/routes.conf` 是**唯一** location 源，443 与 8080 共用 → 新增/改前缀只改这一份，改完 `nginx_check` → `nginx_reload`；
  2. **客户端 base 不得硬编码端口**：`auth_url` 走"包覆盖 > 环境变量 > 默认"三层，域名带非标端口（如 `https://lugwit.cn:8443`）时只需改配置（见《标题栏提供的服务》§3）；Depot 侧的 `LUGWIT_DEPOT_BASE_URL` 同理；
  3. **新增的公网路径必须是 location 白名单里的前缀**：白名单外的根级路径（如 `/.well-known/`）公网取不到 —— 需要的端点要挂到已开放前缀下（例：JWKS 用 `/api/v1/.well-known/…` 而不是根级）；
  4. **回环判定会因反代而失真**：反代下 `request.client.host` 恒为 `127.0.0.1`，凡"本机才允许"的逻辑（如 `lugwit_auth` 的 `/api/v1/auth/auto`）必须额外要求**无 `X-Forwarded-For` / `X-Real-IP` / `Forwarded` 头**，否则公网请求会被当成本机；生产建议该能力默认关。
  **✅ 2026-09-20 已落地**（P0-1）：`/auth/auto` 改为 `LUGWIT_AUTO_AUTH_ENABLED` **默认关**、判定收紧为「peer ∈ {`127.0.0.1`,`::1`} **且** 无转发头」、返回 `role=service` 降权令牌；全仓消费方已改为 env / 登录取 token，`_auto_local_token` 兜底与相关调用已删（见 `lugwit_auth统一用户授权服务设计.md` §9 P0）。
  5. **8080 对外 = 明文降级**：无 TLS，cookie/token 裸奔；对外只能作诊断，不能当客户端入口（§11.1 已收成回环，放开需显式决策）。
  6. **当前无域名（IP 自签）→ 将来上域名的两件事**：
     - 客户端信任的是 **CA 文件不是 host**（`l_qframelesswindow/ssl_support.py`：`LUGWIT_CA_FILE` > 包内 `config/ca_bundle.pem` > `config/ca.pem` > `C:/certs/lugwit/ca.pem`，且 `ca_bundle.pem` 已把"自签 CA + 公网根"合并成单文件）→ **换 IP/域名只需重生成该 bundle，不改代码**；过渡期 IP 与域名并存时同一份 bundle 即可。
     - `lugwit_token` 是 **host-only cookie**（不写 `Domain`）→ **换域名后用户要重新登录一次**（旧 cookie 不跨 host），但 token 本身与 host 无关、仍然有效（不用全员改密）。

## 8. 排查手段

1. 改完配置先 `wuwor l_nginx -- nginx_check`（语法校验，不生效），再 `nginx_reload`
2. 验证各路由：

```bat
curl http://127.0.0.1:8080/nginx-health          :: ok（本机调试口，nginx 本身活着）
curl http://127.0.0.1:8080/api/v1/health         :: 认证服务
curl http://127.0.0.1:8080/note/api/health       :: 笔记后端 {"ok":true}
curl http://127.0.0.1:8080/note/api/notes        :: 应 401（需登录），证明剥前缀正确
curl http://127.0.0.1:8080/docs/                  :: 文档站目录列表（200 + autoindex HTML）

curl.exe -k https://121.196.144.88/nginx-health   :: 公网入口 200（自签证书，-k 跳过本机校验）
```

3. 看日志：`runtime/logs/error.log`（nginx 层报错）、`runtime/logs/access.log`（请求命中哪个 location）、`nginx-cli.log`（CLI 操作记录）
4. 新加后端服务的接入步骤：后端同机起服务 → `conf/lugwit.conf` 加 `upstream` → **`conf/routes.conf` 加 `location`**（注意剥前缀规则与 location 顺序）→ `nginx_check` → `nginx_reload`
5. 公网不通先分清"哪一层"：
   - `http://121.196.144.88:8080/*` → **公网不可达**（8080 只绑回环，curl 连接失败是预期）
   - `https://121.196.144.88/nginx-health` → 期望 **200 OK**（nginx 1.27.5，自签证书）
   - `https://lugwit.duckdns.org/*` → **握手被 RST**（SNI 拦截，见 §11）

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

## 11. 2026-09-17：公网入口收敛 443、路由单源、域名弃用

### 11.1 公网入口从 8080 改为 443

- 以前"对外只暴露 8080"的做法**已废弃**。现状：
  - `conf/lugwit.conf` 的 8080 收成 **`listen 127.0.0.1:8080`**（只绑回环，本机调试口）
  - **公网入口是 443**：`conf/https.conf` 两个 server 块 —— 默认块 = 自签 IP 证书 `C:/certs/lugwit`；域名块 = `lugwit.duckdns.org` 用 Let's Encrypt `C:/certs/lugwit_le`
- 实测证据：
  - `http://121.196.144.88:8080/*` → **公网不可达**（curl 连接失败）
  - `https://121.196.144.88/nginx-health` → **200 OK**（nginx 1.27.5，自签证书）

### 11.2 路由集合收敛到 `conf/routes.conf`

- 之前 location 集合散在两处；现在**只有一份** `conf/routes.conf`，被 `conf/https_common.conf`（443）和 `conf/lugwit.conf` 的 8080 server 块共同 `include`。**443 不再经 8080 转发**，两处各自直接命中同一套 location（少一跳）。

### 11.3 `duckdns.org` 域名被 SNI 拦截、弃用

- 实测：`https://lugwit.duckdns.org/*` → **TLS 握手被重置（RST）**；且**任意 `*.duckdns.org`（含不存在的子域）都一样被 RST**，而任意其它未知 SNI（如 `no-such-host.example.com`）却能正常握手。
- 结论：**国内网络按 `duckdns.org` 关键字做 SNI 拦截**，不是服务器/证书问题，改 nginx 也救不了 → 域名路线弃用 DuckDNS。
- 客户端与运维工具默认入口已从失效域名改为 IP：`https://121.196.144.88/script_editor`（`l_nginx/tools/remote_exec.py`、`remote_sync.py`、`remote_push.py` 默认 host；新增 `tools/_tls.py` 统一处理自签证书：CA 校验优先、失败降级不校验并警告）。详见[Rez_pkg/HTTPS证书与域名申请总结.md](Rez_pkg/HTTPS证书与域名申请总结.md)。

### 11.4 域名方向（2026-09-17 拍板）：`lugwit.cn` + 8443 + LE

- 决定**购买 `lugwit.cn`，用非标端口 `8443` + Let's Encrypt（阿里云 DNS-01 自动续）**，理由是**免 ICP 备案**；443 暂时保留 IP 自签（纯 IP 访问不受备案影响）。
- 若要"域名 + 443 不带端口"就必须 **ICP 备案**（国内 ECS 未备案域名访问 80/443 会被拦）。
