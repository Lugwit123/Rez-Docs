# lugwit_auth 统一用户授权服务设计

> 状态：**设计稿（待评审）**，2026-09-19
> 范围：把 `lugwit_auth`（1027）从"JWT + 密码 + 用户表"升级为**全公司唯一的用户授权服务**：
> 统一身份、统一登录窗口、统一授权判定、统一共享能力（凭据/偏好/收藏/审计），
> 业务服务只做业务，不再各自实现鉴权。
> 关联文档：
> - [网盘版本库Depot设计.md](网盘版本库Depot设计.md) §6.1（鉴权闸门与内容获取链路）
> - [Depot库与工作区方案.md](Depot库与工作区方案.md) §3.5（为什么 owner 不能继续当工作区）
> - [标题栏提供的服务.md](标题栏提供的服务.md) §3（标题栏"认证接入能力"现状）
> - [Nginx反向代理机制.md](Nginx反向代理机制.md)（路由与端口归属）
> - [Rez_pkg/lugwit_baidu_netdisk.md](Rez_pkg/lugwit_baidu_netdisk.md) §2.3（登录闸门）
>
> 约定：本文中「现状」= 已实现，附 `包:文件:行` 证据；「新增」= 本文设计，尚未实现。
> 路径前缀 `R = trayapp/rez-package-source`。

---

## 0. 一句话目标

**任何程序拿到"用户是谁、能做什么"，都只问 `lugwit_auth` 一次；用户只在一处登录，凭据只存一处，登录窗口只维护一份。**

---

## 1. 现状盘点（基线）

### 1.1 服务与端口

| 项 | 值 | 证据 |
|---|---|---|
| 服务 | `lugwit_auth` 999.0，`python -m lugwit_auth.auth_server` | `R/lugwit_auth/999.0/package.py:75` |
| 端口 | 1027（`LUGWIT_AUTH_PORT` / `--port`） | `auth_server.py:1287-1288` |
| 监听 | 默认 `0.0.0.0`；`.env` 写 `127.0.0.1` | `auth_server.py:1287`、`.env:15` |
| 外部入口 | 生产 `https://121.196.144.88/api/v1/*`；开发 nginx `127.0.0.1:8080` | `R/Rez-Docs/标题栏提供的服务.md:40-44` |
| 数据库 | Postgres `chatroom`（`postgresql+asyncpg://…@121.196.144.88:5432/chatroom`） | `config.py:28-33` |
| 进程内挂载 | `/baidu/*`（`LUGWIT_AUTH_MOUNT_BAIDU`） | `auth_server.py:139,167` |

### 1.2 接口现状（27 条，摘要）

| 分组 | 端点 | 鉴权 |
|---|---|---|
| 健康 | `GET /api/v1/health`、`/healthz`、`GET /api/v1/services/status` | 公开（**services/status 无鉴权**，暴露端口清单） |
| 服务监督 | `POST /api/v1/services/{name}/{stop,start,restart,reload}` | 管理员 |
| 登录页 | `GET /login`、`/api/v1/auth/login`(页面) | 回环自动登录 / 302 |
| 登出 | `GET /logout`、`/api/v1/auth/logout` | 公开（**仅清 cookie**） |
| 自动授权 | `POST /api/v1/auth/auto` | **仅回环** → `system01` |
| 主页/设置 | `GET /`、`/settings`（网盘 OAuth） | 用户 / 回环 |
| 令牌 | `POST /api/v1/auth/token`（OAuth2 password）、`/verify`、`/login`、`/register` | 公开（verify 需 Bearer） |
| 当前用户 | `GET /api/v1/auth/me` | 用户 |
| 用户管理 | `GET/POST /api/v1/users`…`PUT /api/v1/users/{id}`、`DELETE`、`PUT …/password` | 用户/管理员 |
| 收藏与凭据 | `GET/PUT /api/v1/accounts/custom-fields`、`/api/v1/accounts*`、`/api/v1/fav-items*` | 用户（按 owner） |

证据：`auth_server.py:305-1085`（逐条行号见 §附录 A）。

### 1.3 数据模型现状

| 表 | 作用 | 证据 |
|---|---|---|
| `users` | id/username/password_hash/status/display_name/email/phone/last_login_at | `models.py:12-27` |
| `user_roles` | user_id + role 字符串 + is_primary | `models.py:30-40` |
| `sessions` | refresh_token/revoked/user_agent/client_ip | `models.py:43-58` —— **建表但运行期零读写** |
| `l_notepad_accounts` / `l_notepad_custom_fields` / `l_notepad_fav_items` | 账号收藏 + 自定义字段 + 通用收藏（密码 Fernet 加密） | `account_service.py:87-135` |
| 权限/角色字典/应用注册/审计 | **不存在** | 全包 grep 无 `audit`、无 roles 字典表 |

角色取值 `user|admin|system`（`enums.py:12-20`），JWT 里编码成 int `0/1/2`（`auth_server.py:57-59`）。

### 1.4 token 机制现状

| 项 | 现状 |
|---|---|
| 算法 | HS256，**对称密钥**，默认硬编码 `chatroom-dev-secret-key-change-in-production-2024` |
| 密钥来源 | env `LUGWIT_AUTH_JWT_SECRET`（`config.py:35-38`），与 ChatRoom 共用 |
| claims | `sub`(username) / `role`(int) / `user_id` / `exp`（`auth_server.py:276-280`） |
| 有效期 | **30 天**（`jwt_service.py:15`），无刷新、无吊销、无黑名单 |
| cookie | `lugwit_token`，`path=/`，`samesite=lax`，**`httponly=False`**，7 天（`auth_server.py:663-665`） |
| 本地离线校验 | `gate.require_lugwit_token`（`R/lugwit_baidu_netdisk/.../gate.py:22-37`）只验签名+exp，**不查库、不校验 status/role** |
| 自动授权 | 回环（且**信任 `X-Forwarded-For`**）→ `system01`（`auth_server.py:174-192,712-720`） |

### 1.5 登录入口现状 —— 「是否提供登录窗口」

**半是**：

| 形态 | 有没有 | 位置 |
|---|---|---|
| **Web 登录页**（登录 + 注册双 tab） | ✅ | `templates/login.html`，路由 `GET /login?next=`（`auth_server.py:669`） |
| 用户中心主页 / 设置页 | ✅ | `templates/home.html`、`templates/settings.html` |
| **桌面登录窗口** | ⚠️ **不在 auth 包里，各包各抄一份** | `l_qframelesswindow/.../login_dialog.py` + `login_store.py`（token 存 `<data_dir>/auth_token.json`，**记住的密码明文** `remembered_login.json`）；`l_tray/.../lugwit_login/loginUI.py`；`l_WChat/.../templates/login.html` + `/api/lugwit/login` 代理；`l_notepad_client` 用标题栏登录按钮 |
| 标题栏角色 | 只提供"认证接入能力"（`auth_url`/`auth_route` + `serverConfigChanged` 信号），**不弹登录框、不存账号** | `R/Rez-Docs/标题栏提供的服务.md:37-46` |

### 1.6 已经沉在 auth 里的"共享能力"

| 能力 | 现状 | 问题 |
|---|---|---|
| 第三方账号收藏 + 密码加密托管 | ✅（Fernet，密钥 `~/.lugwit/l_notepad/.accounts_key`） | **表名/API 命名绑定 `l_notepad`**，其他包用不了 |
| 通用收藏（folder/cmd/url） | ✅ `fav-items` | 同上，命名绑死 |
| 服务进程监督（start/stop/restart/reload） | ✅ | 与"授权"职责混在一起 |
| 用户资料 | ✅ | 无头像/无扩展字段 |
| 登录失败/操作审计 | ❌ | 只有 `users.last_login_at`，失败静默（`user_service.py:246-254`） |

### 1.7 已确认问题清单（按严重度）

| # | 问题 | 证据 | 影响 |
|---|---|---|---|
| P0-1 | **回环自动授权 = 免密系统身份**，且信任 `X-Forwarded-For` | `auth_server.py:174-192` | 任何本机进程（或能伪造该头的请求）拿到 `system01`+admin；多用户数据无法隔离 |
| P0-2 | 无注销/吊销，JWT 30 天不可撤销 | `auth_server.py:693-709`、`jwt_service.py:15` | 令牌泄露无法止损 |
| P0-3 | 对称密钥 + 硬编码默认值，多包共用 | `config.py:35-38` | 一个包泄露密钥 = 全平台可伪造身份 |
| P1-1 | 无角色/权限模型，`role` 只当数据 | `auth_server.py:57`；`gate.py` 不判 role | 无法做细粒度授权 |
| P1-2 | 离线校验不查库（禁用/删除的用户 token 仍可用） | `gate.py:31-37` | 离职/停用账号可继续访问 |
| P1-3 | 各包各自解析 token，校验方式分裂成"本地验签" vs "网络 `/me`" | 6 处消费点（附录 A） | 行为不一致，难排查 |
| P1-4 | 桌面登录窗口 4 份实现，记住密码明文落盘 | §1.5 | 维护成本 + 泄密面 |
| P1-5 | 逻辑路径无 owner 校验（`download` 只验"已登录"） | `R/lugwit_baidu_netdisk/.../web_server.py:2859-2880` | 知道路径即可读别人的数据 |
| P2-1 | cookie 属性由前端 JS 手写，不带 samesite/domain | `login.html:73-76`、`home.html:294` | 配了 `LUGWIT_DOMAIN` 时清不掉 cookie |
| P2-2 | `services/status` 无鉴权但 OpenAPI 标为需认证 | `auth_server.py:317` vs `:109-115` | 文档/实现不一致 |
| P2-3 | DB URL 键名分叉（`LUGWIT_AUTH_DB_URL` vs `…DATABASE_URL`）；`init_db` 用同步引擎接 asyncpg 串 | `config.py:29` vs `init_db.py:17,12` | 初始化路径不可靠 |
| P2-4 | 模块 docstring 接口表与实现脱节；health 版本号 1.1.0 vs app 1.2.0 | `auth_server.py:6-21`、`:307` vs `:82` | 靠猜 |

---

## 2. 目标 / 非目标

### 2.1 目标

1. **一套身份**：所有端（桌面/浏览器/服务间）解析出的 `user` 完全一致，`system01` 只保留给"确实没有用户上下文"的后台任务，且必须显式声明。
2. **一处登录**：桌面统一登录窗口由 auth 侧 SDK + 一份 UI 组件提供，其他包只引用；密码不再落地。
3. **在线授权 + 离线验签**并存：短时 access token 本地验签（不依赖 auth 在线），长时 refresh 交服务端轮转；吊销靠短 TTL + token 版本号。
4. **能力下沉**：凭据托管、用户偏好、收藏、头像、审计、授权判定统一由 auth 提供；业务包不再各实现。
5. **可授权到资源**：提供 `/authz/check`，让 depot/笔记等做**按 owner / 共享 / 角色**的访问判定（补 P1-5）。
6. **向后兼容**：旧 token、旧 cookie 名、旧端点在三阶段内继续可用，各包不必同一天改完。

### 2.2 非目标

- 不做 IM/消息、不做文件存储（那是 depot 的职责）。
- 不接管业务 API 的流量（**auth 仍是"认证 + 授权判定"服务，不做网关**；与《标题栏提供的服务》§3「登录服务默认不拦截密码之外的业务流量」一致）。
- 不做短信/邮箱验证码（暂不引入第三方依赖）。

---

## 3. 设计原则

| 原则 | 落法 |
|---|---|
| 身份唯一来源 | JWT `sub` = username，`uid` = 数字 id；所有包只认这两个 |
| 服务端为准 | 账号状态（active/disabled/deleted）、角色变更必须能在 ≤5 分钟内生效 → access TTL 15 min + `token_version` |
| 离线可验 | 非对称签名 + JWKS，业务包本地验签，不把 auth 变成单点 |
| 最小暴露 | 桌面 UI 只拿 access（内存）+ refresh（系统钥匙串/DPAPI），不再明文存密码 |
| 能力下沉但不耦合 | 通用能力走 `/api/v1/{favorites,credentials,prefs,avatars}` 等**中性命名**，旧 `l_notepad_*` 表/端点在过渡期做兼容视图 |
| 可审计 | 所有敏感操作落 `audit_logs`（登录/失败/改密/角色变更/凭据读写/授权拒绝） |
| **在线不依赖 Depot** | 登录、验签、授权判定、令牌轮换只用 Postgres(+内存)；Depot 仅用于**备份、审计冷归档、二进制对象、跨机分发**四类非在线用途（见 §5.3、§13） |
| 密钥与数据分离 | 密文随数据走（PG/Depot），**主密钥与签名私钥既不进 PG 也不进 Depot**，走本机/离线介质（见 §5.3 D 类） |

---

## 4. 目标架构

### 4.1 分层

```
┌──────────────────────────── lugwit_auth (1027) ────────────────────────────┐
│ Identity    用户/资料/头像/状态        users, user_profiles                  │
│ Credential  密码/令牌/凭据托管          credentials(加密), token_version      │
│ Authorization 角色/权限/资源 ACL        roles, permissions, role_permissions, │
│                                        acl_grants,  POST /authz/check        │
│ Session     会话/设备/吊销             sessions, devices                     │
│ Client      应用注册/授权码/PKCE       clients, auth_codes                   │
│ Audit       审计                        audit_logs                            │
│ Shared      收藏/偏好/凭据/图片         favorites, prefs, credentials, files │
│ Ops         服务监督（**建议拆出**）     /api/v1/services/*                    │
└──────────────────────────────────────────────────────────────────────────────┘
        ▲ 本地验签(公钥 JWKS)          ▲ 在线判定(短 TTL)
        │                              │
   业务服务(1028/8765/…)          业务服务 / 桌面客户端
```

### 4.2 三种接入模式

| 模式 | 场景 | 凭据 | 校验方式 |
|---|---|---|---|
| A. 桌面客户端 | PyQt/无边框窗口程序 | `access`(15min, 内存) + `refresh`(30d, DPAPI/钥匙串) | access 本地验签；过期 → `POST /auth/refresh` |
| B. 浏览器 | Web 后台/工具页 | `lugwit_token` cookie（**HttpOnly**）+ CSRF 双提交 | 服务端读 cookie → 本地验签（同 A） |
| C. 服务间 | 后台任务、CI、本机脚本 | **service token**（见下） | 本地验签，`typ=service` |
| D. 本机自动授权（保留但收紧） | 本机脚本免登录 | 同 C 语义 | **必须显式** `LUGWIT_AUTO_AUTH=1` 或 `POST /auth/auto` 带 `X-Lugwit-Local-Proof`；解析成 `system01` 但**降权**为 `role=service`（不再 admin），且**不信任 `X-Forwarded-For`** |

**收敛 P0-1**：回环判定改为「peer IP ∈ {127.0.0.1, ::1}」**且**请求未经过代理（无 `X-Forwarded-For`）；`/auth/auto` 返回的 token 写 `typ=service, role=service`，仅能访问被显式允许的 scope（如 depot 的本机工作区），不再隐式获得 admin。

### 4.3 统一登录窗口与 SSO（重点）

现状：桌面登录窗口 4 份、密码明文落盘、token 只存 JSON。

**设计：登录逻辑下沉为 SDK，UI 只在标题栏维护一份。**

```
① 桌面登录（推荐：回环授权码，无密码落地）
   LoginDialog(标题栏组件) --打开--> http://127.0.0.1:1027/login?next=lugwit://auth/callback&state=<rnd>&code_challenge=<S256>
        │  用户在浏览器/内嵌 WebView 登录（或已登录则直接确认）
        ▼
   auth 302 → lugwit://auth/callback?code=<one-time>&state=<rnd>
        │  SDK lugwit_auth.client.LocalCallbackServer 接住
        ▼
   POST /api/v1/auth/token  {grant_type:"authorization_code", code, code_verifier, client_id:"desktop"}
        ▼
   ← {access_token(15min), refresh_token(30d), user{...}}
        │
   refresh 存 DPAPI（Windows Credential Manager），access 只留内存
```

- **SDK 职责**（`lugwit_auth.client`，供所有 Python 客户端 import）：
  `login_via_browser(client_id, scopes)`、`refresh()`、`logout()`、`ensure_token()`（自动续期）、`verify_local(token)`（离线验签）。
- **UI 职责**（`l_qframelesswindow` 的 `LoginDialog`）：只做两件事 —— 打开授权 URL、显示"已登录为 X / 登出"。**不再自己 POST 密码、不再存密码。**
- **降级路径**：无浏览器/内嵌 WebView 不可用时，才允许 `POST /auth/login`（用户名+密码），且 SDK 只在**内存**持有密码到拿 token 为止（不落盘）。
- **浏览器 SSO**：`GET /api/v1/auth/authorize?...` 出授权码 → 业务站 `POST /auth/token` 换取自己的会话 cookie（与现有 `lugwit_token` 并存过渡）。

### 4.4 部署前提：公网只开少量端口

现状（《Nginx反向代理机制》§2.1、§5、§7）：**公网唯一入口是 443**（IP 自签证书；将来 `lugwit.cn:8443` + LE，见该文 §11.4），
nginx 与所有后端**同机**，后端一律 `listen 127.0.0.1`，对外只放行少数端口。
本设计由此产生 **四条硬约束**：

1. **跨机流量只能走单入口 + 前缀**：`https://<入口>/api/v1/auth/*`（认证）、`/baidu/*`（Depot）。
   本文出现的 `http://127.0.0.1:1027` 只是**本机开发/CLI** 形态；生产由配置给 base（可含端口与前缀），
   **客户端不得硬编码端口**（`auth_url` 三层解析已支持带端口：`https://lugwit.cn:8443`，见《标题栏提供的服务》§3）。
2. **JWKS 必须落在已开放的前缀内** → 用 `/api/v1/.well-known/jwks.json`。
   根级 `/.well-known/` **不在 nginx 路由白名单**（`conf/routes.conf` 是 location 白名单制），放根级 = 公网取不到 → 跨机离线验签直接失效。
3. **回环判定在 nginx 前置后失效，必须同时要求「无转发头」** —— 这是个高危点：
   nginx 反代下 `request.client.host` **恒为 `127.0.0.1`**，而现有实现（`auth_server.py:174-192`）
   反而把 `X-Forwarded-For: 127.0.0.1` 当作"本机"证据 → **公网请求会被判成本机，直接拿到 `system01`+admin**。
   因此 §4.2 D 的回环自动授权必须同时满足：
   - 生产 `LUGWIT_AUTO_AUTH_ENABLED=0`（**默认关**，仅本机脚本显式开）；
   - 判定条件收紧为「peer ∈ {`127.0.0.1`, `::1`} **且** 无 `X-Forwarded-For` / `X-Real-IP` / `Forwarded` 头」。
4. **8080 若对外 = 明文降级**（无 TLS、`lugwit_token` 裸奔）：与 §11.1「8080 收成 `listen 127.0.0.1`」的既有决定冲突。
   要对外必须**显式决策**，且只作诊断用途，不能作为客户端入口 —— 客户端一律走 443（或未来 8443）。

> 运维口径以《Nginx反向代理机制》§2.1 端口总表 + §7 安全建议为准；本节只声明"授权服务在这套暴露面下的红线"。

### 4.5 无域名阶段的形态（**当前就在这个阶段**）与升级到域名的清单

**当前**：还没有域名 → 公网入口就是 `https://121.196.144.88`（443，**IP 自签证书**，nginx 侧 `C:/certs/lugwit`）。
本节把"这套形态下哪些已经能用、哪些将来要动"钉死，免得升级时漏项。

**已经覆盖的（不需要为域名做任何事）**

| 项 | 现状 | 为什么不受影响 |
|---|---|---|
| 桌面登录（PKCE 回环授权码） | SDK 打开 `https://121.196.144.88/login?next=lugwit://auth/callback`，回环回调 `http://127.0.0.1:<port>/callback` 换 token | 回调是 **http + 回环**，与证书、域名都无关（§4.3） |
| 客户端自签证书信任 | `l_qframelesswindow/ssl_support.py`：CA 查找顺序 `LUGWIT_CA_FILE` env > 包内 `config/ca_bundle.pem`（自签 CA + 公网根**合并单文件**）> `config/ca.pem` > `C:/certs/lugwit/ca.pem`；`install_default_ca()` 给进程默认 HTTPS context 打补丁 | 信任的对象是 **CA 文件而不是 host** → **换 IP / 换域名只换 bundle，不改代码**；`ca_bundle.pem` 的合并写法本身就为"自签 + 公信并存"预留了 |
| 令牌 | JWT 与 host 无关 | 换域名后旧 access/refresh **仍然有效**（不用全员改密） |
| 内部服务间调用 | 全走 `127.0.0.1` 回环 | 不经公网、不经证书 |

**升级到域名时要动的清单**（提前掌握，真到那天照做）

| # | 动作 | 备注 |
|---|---|---|
| 1 | nginx 加域名 server 块 + LE 证书（`C:/certs/lugwit_le`） | 走 443 必须 **ICP 备案**；想免备案就用 **8443**（《Nginx反向代理机制》§11.4 已定方向） |
| 2 | 客户端配置改 base（可带非标端口/前缀） | `auth_url`、`api_url`、`LUGWIT_DEPOT_BASE_URL`；**代码里不得硬编码端口**（§4.4 第 1 条） |
| 3 | 重生成 / 替换 `ca_bundle.pem` | 保持"自签 CA + 公网根"合并；过渡期 IP 与域名**并存**时这一份就够 |
| 4 | **用户需重新登录一次** | ⚠️ `lugwit_token` cookie 是 **host-only**（不写 `Domain`，§1.4）→ IP 上那份 cookie 不会带到域名。这是最容易漏的 UX 项 |
| 5 | CORS / Origin 白名单加新 origin | `l_tray` ExecServer 的 `L_TRAY_EXEC_ORIGINS`；Depot 侧白名单同理（见《网盘版本库Depot设计》§6.1） |
| 6 | `clients.redirect_uris` 加新 origin | 浏览器 SSO 与业务站回跳（§6.1） |
| 7 | 证书轮换演练一次 | `ssl_support` 的降级分支（**校验失败 → 不校验重试 + 只警告一次**）不能被当成常态：生产要求 CA 必须找到，否则等同 MitM 敞口 |

**现在就该做的一件事**：把 `LUGWIT_CA_FILE` 写进部署清单（与 §13.7 的"密钥离线保管"同级），
并让客户端启动时**显式打印当前生效的 CA 与是否走了降级分支** —— 现状只在"校验失败"时警告一次，容易被日志淹没。

**浏览器 / 内嵌 WebView 的信任（另一条独立的坑）**

`ssl_support` 那份 `ca_bundle.pem` 只管 **Python**（urllib/requests）。**外部浏览器（Chrome/Edge）与 QtWebEngine** 读的是
**Windows 系统根库** → 指向 `https://121.196.144.88/...` 的自签页面仍会告警 / 白屏。已有工具：
**`l_nginx/999.0/tools/install_lugwit_ca.bat`**（安装 / `/uninstall` / `/y`，自动提权、装前打印指纹待确认、装后按指纹自校验）。

| 形态 | 谁解决 |
|---|---|
| 桌面 Python | `l_qframelesswindow` 随包 `ca_bundle.pem`（§11.2） |
| Android WebView | `l_WChat` 内置 `lugwit_ca.pem` |
| 外部浏览器 / QtWebEngine | **`install_lugwit_ca.bat` 装系统根库**（一次覆盖两类；**待实测确认**） |
| 托盘 `l_tray` 的 Python | 同一条：装根库后走系统库也能过（否则它拿不到包内 CA） |

⚠️ 装根库是**机器级信任**，且当前自签形态下「服务器证书私钥」与「CA 私钥」是**同一把**
（`C:/certs/lugwit/privkey.pem`，自签 = leaf 即 CA）→ 外泄即可全平台中间人。只推可信内网机器；
长期应拆成"只签服务器证书的中间 CA + 短有效期"。详见《Rez_pkg/HTTPS证书与域名申请总结.md》§11.3。

---

## 5. 领域模型与表设计

### 5.1 表清单

| 表 | 动作 | 说明 |
|---|---|---|
| `users` | 改造 | 加 `token_version int default 0`（改密/禁用/角色变更时 +1 → 旧 access 立即失效）、`avatar_file_id`、`invited_by` |
| `user_profiles` | 新增 | 用户级偏好 KV（JSONB）——取代各包自己的 `server_config.json` 里"用户相关"那一半 |
| `roles` | 新增 | 角色字典：`code`(user/admin/service/auditor) + `name` + `is_system` |
| `permissions` | 新增 | 权限点字典：`code`(如 `depot.read`, `depot.write.any`, `auth.users.manage`) + `desc` |
| `role_permissions` | 新增 | 角色 → 权限点 |
| `user_roles` | 保留 | 改为引用 `roles.code`（唯一改动：约束取值） |
| `acl_grants` | 新增 | 资源级授权：`(resource_kind, resource_id, owner, grantee, perms)`；`grantee` 支持用户或 `*` |
| `sessions` | 启用 | `refresh_token_hash`、`token_version`、`expires_at`、`revoked_at`、`device_label`、`last_seen_ip`（**现状空表**） |
| `clients` | 新增 | 应用注册：`client_id`、`name`、`secret_hash`（机密客户端）、`redirect_uris`、`is_public`(PKCE) |
| `auth_codes` | 新增 | 授权码（一次性、60s、绑定 `code_challenge`/`client_id`/`state`） |
| `audit_logs` | 新增 | `at`、`actor`、`action`、`target`、`result`、`ip`、`ua`、`detail` JSONB |
| `credentials` | 新增（接管） | 泛化 `l_notepad_accounts`：加 `scope`(包名/域)、`kind`(password/token/ssh/…)、`secret_enc` |
| `favorites` | 新增（接管） | 泛化 `l_notepad_fav_items`：`owner` + `scope` + `kind(name/folder/cmd/url)` |
| `l_notepad_accounts` / `_fav_items` / `_custom_fields` | 迁移后保留为**兼容视图** | 过渡期两套可写，双写窗口结束后下线 |

### 5.2 DDL 草案（Postgres）

```sql
-- 身份与令牌版本
ALTER TABLE users ADD COLUMN IF NOT EXISTS token_version int NOT NULL DEFAULT 0;
ALTER TABLE users ADD COLUMN IF NOT EXISTS avatar_file_id text;
ALTER TABLE users ADD COLUMN IF NOT EXISTS invited_by text;

-- 角色与权限
CREATE TABLE IF NOT EXISTS roles (
  code text PRIMARY KEY, name text NOT NULL, is_system boolean NOT NULL DEFAULT false,
  created_at timestamptz DEFAULT now()
);
CREATE TABLE IF NOT EXISTS permissions (
  code text PRIMARY KEY, name text NOT NULL, created_at timestamptz DEFAULT now()
);
CREATE TABLE IF NOT EXISTS role_permissions (
  role_code text NOT NULL REFERENCES roles(code) ON DELETE CASCADE,
  perm_code text NOT NULL REFERENCES permissions(code) ON DELETE CASCADE,
  PRIMARY KEY (role_code, perm_code)
);

-- 资源级 ACL（depot / 笔记 / 任意资源通用）
CREATE TABLE IF NOT EXISTS acl_grants (
  id bigserial PRIMARY KEY,
  resource_kind text NOT NULL,          -- 'depot' | 'notepad' | ...
  resource_id   text NOT NULL,          -- '/l_agent_chat/u01' | 'note:123'
  owner         text NOT NULL,          -- username，创建者
  grantee       text NOT NULL DEFAULT '*',  -- username 或 '*'
  perms         text NOT NULL,          -- 'read,write,admin'
  created_at timestamptz DEFAULT now(),
  UNIQUE (resource_kind, resource_id, grantee)
);
CREATE INDEX IF NOT EXISTS ix_acl_kind_id ON acl_grants(resource_kind, resource_id);

-- 会话（启用现成空表）
ALTER TABLE sessions ADD COLUMN IF NOT EXISTS refresh_token_hash text;
ALTER TABLE sessions ADD COLUMN IF NOT EXISTS token_version int NOT NULL DEFAULT 0;
ALTER TABLE sessions ADD COLUMN IF NOT EXISTS expires_at timestamptz;
ALTER TABLE sessions ADD COLUMN IF NOT EXISTS revoked_at timestamptz;
ALTER TABLE sessions ADD COLUMN IF NOT EXISTS device_label text;
ALTER TABLE sessions ADD COLUMN IF NOT EXISTS last_seen_ip text;

-- 客户端应用
CREATE TABLE IF NOT EXISTS clients (
  client_id text PRIMARY KEY, name text NOT NULL,
  secret_hash text,                     -- NULL = 公共客户端(PKCE)
  redirect_uris text[] NOT NULL DEFAULT '{}',
  is_public boolean NOT NULL DEFAULT true,
  created_at timestamptz DEFAULT now()
);
CREATE TABLE IF NOT EXISTS auth_codes (
  code_hash text PRIMARY KEY, client_id text NOT NULL, username text NOT NULL,
  code_challenge text, redirect_uri text, scopes text,
  expires_at timestamptz NOT NULL, consumed_at timestamptz,
  created_at timestamptz DEFAULT now()
);

-- 审计
CREATE TABLE IF NOT EXISTS audit_logs (
  id bigserial PRIMARY KEY, at timestamptz NOT NULL DEFAULT now(),
  actor text, action text NOT NULL, target text, result text NOT NULL,
  ip text, ua text, detail jsonb
);
CREATE INDEX IF NOT EXISTS ix_audit_actor_at ON audit_logs(actor, at DESC);
CREATE INDEX IF NOT EXISTS ix_audit_action_at ON audit_logs(action, at DESC);

-- 共享能力（泛化自 l_notepad_*）
CREATE TABLE IF NOT EXISTS credentials (
  id bigserial PRIMARY KEY, owner text NOT NULL, scope text NOT NULL DEFAULT '',
  name text NOT NULL, kind text NOT NULL DEFAULT 'password',
  username text, secret_enc bytea, server text, notes text,
  custom_fields jsonb, sort_order int DEFAULT 0,
  created_at timestamptz DEFAULT now(), updated_at timestamptz DEFAULT now()
);
CREATE INDEX IF NOT EXISTS ix_credentials_owner_scope ON credentials(owner, scope);
CREATE TABLE IF NOT EXISTS favorites (
  id bigserial PRIMARY KEY, owner text NOT NULL, scope text NOT NULL DEFAULT '',
  kind text NOT NULL, name text NOT NULL, value text,
  sort_order int DEFAULT 0, created_at timestamptz DEFAULT now()
);
CREATE TABLE IF NOT EXISTS user_profiles (
  username text NOT NULL, key text NOT NULL, value jsonb,
  updated_at timestamptz DEFAULT now(), PRIMARY KEY (username, key)
);
```

种子数据（`lugwit_auth_seed_roles`）：

| role | permissions |
|---|---|
| `user` | `self.read`, `self.write`, `favorites.*`, `prefs.*`, `credentials.own.*` |
| `service` | `self.read`, `depot.read`, `depot.write`（本机工作区；**不含** `auth.users.*`） |
| `admin` | 全部 `auth.*`、`depot.*`、`audit.read` |
| `auditor` | `audit.read`（只读审计） |

### 5.3 数据分层：哪些进 Postgres，哪些进 Depot

**铁律：在线关键路径永不依赖 Depot。** 登录、验签、授权判定、令牌轮换只能靠 Postgres(+内存)。
理由：Depot 的元数据本身就存在**同一个** Postgres（`chatroom`）里，其可用性并不高于 PG；
把在线依赖压到 Depot 只会把故障域从 1 个扩成 2 个（还多一个外部网盘）。依据见 §13.3。

**决策树（5 问定归宿）**

| # | 问题 | 归宿 |
|---|---|---|
| 1 | 是**密钥 / 根信任**吗？ | **离线人工保管**（PG、Depot 都不放）→ D 类 |
| 2 | 要**毫秒级生效** + 事务 + 唯一约束吗？ | **Postgres** → A 类 |
| 3 | 是 **append-only** 大日志、查询集中在近期吗？ | **PG 热 + Depot 冷归档** → B 类 |
| 4 | 是**二进制 / 大对象 / 要版本与跨机分发**吗？ | **Depot** → C 类 |
| 5 | **丢了能重建**（重新登录即可）吗？ | **不持久化**（内存/短 TTL）→ E 类 |

**逐数据集判定**

| 数据 | 主存 | 进 Depot？ | 理由 |
|---|---|---|---|
| `users`（password_hash/status/**token_version**） | PG | 只作备份快照 | 每次登录/改密都写；要唯一约束；`token_version` 必须毫秒级生效 |
| `user_roles` / `roles` / `permissions` / `role_permissions` | PG | 只作备份 | `/authz/check` 每次读 |
| `sessions`（refresh 哈希/吊销/设备） | PG | ❌ | 轮换与吊销必须原子，Depot 给不了 |
| `clients`（client_id/secret_hash/redirect_uris） | PG | 只作备份 | 授权流程在线读 |
| `auth_codes`（一次性、60s） | PG（或内存） | ❌ **绝对不** | 一次性核销要原子；寿命 60s，外网往返纯浪费 |
| `invites`（邀请码核销） | PG | ❌ | 一次性核销要原子 |
| 登录失败计数 / 限流窗口 | 内存（多实例时才 PG） | ❌ | 丢了只是少限制几秒 |
| `audit_logs` | **PG 热（近 90 天）+ Depot 冷归档** `/l_auth_audit/<yyyy-mm>.jsonl.gz` | ✅ **是** | append-only、写多读少、要长期留存 —— Depot 强项；但绝不当在线查询表 |
| `credentials` **密文**（`password_enc`） | PG | ✅ 可选（跨机分发介质） | 密文上云无害（**主密钥不上云**）；单机可只用本机副本 |
| 头像 / 用户上传图片 | **Depot**（PG 只存 `avatar_file_id`） | ✅ **是** | 二进制塞 PG 是反模式 |
| `favorites` / `user_profiles` | PG（在线一致） | ✅ 仅作**周期导出**，不做在线唯一副本 | 小、写频、登录后 UI 立刻要用；将来要"多机实时一致"再让 Depot 当同步介质 + 本地缓存 |
| JWKS 公钥 | auth 本地文件 + 端点 | ❌ | 若逼业务包去 Depot 取公钥才能验签 → **造出新环**（§13.4 P-B） |
| Fernet 主密钥 / JWT 签名私钥 | 本机文件 / 离线介质 | ❌ **永不** | PG 或 Depot 任一泄露 = 全平台身份可伪造 / 全部凭据可解密（§12 D8） |
| 令牌明文 | 不存（只存哈希） | ❌ | —— |

**现状对照**

| 现状 | 评价 |
|---|---|
| 凭据密文在 PG、Fernet 主密钥在本机文件（`~/.lugwit/l_notepad/.accounts_key`，`account_service.py:25`） | ✅ **方向正确**；但密钥路径绑死 `l_notepad`，且与密文同机 → 单机被入侵即明文。建议主密钥改 DPAPI/离线介质（§12 D4） |
| `audit_logs` | ⚠️ 表还没建；建了也要按「PG 热 + Depot 冷」设计，别直接当在线表 |
| 头像 | ⚠️ 无落地（§5.1 的 `users.avatar_file_id` 正是指向 Depot） |
| 服务监督端点（start/stop/restart） | ❌ 与授权无关，是「auth 反向依赖各业务」的耦合，建议移出（§12 D2） |
| 是否有 auth 数据**在线**依赖 Depot | ✅ 没有，也不该有 |

---

## 6. 接口契约

> 全部前缀 `/api/v1`；标 `保留` 的与现状一致（不改签名），`新增` 为本文设计。
> 鉴权列：`公开` / `Bearer` / `管理员` / `服务`。

### 6.1 认证

| method | path | 鉴权 | 状态 | 说明 |
|---|---|---|---|---|
| POST | `/auth/login` | 公开 | 保留 | 密码登录；**新增** 失败计数与 `audit_logs`；返回 `{token, access_token, refresh_token, expires_in, user}` |
| POST | `/auth/token` | 公开 | 扩展 | 兼容 `password` 授权；**新增** `grant_type=authorization_code`（PKCE）、`refresh_token` |
| POST | `/auth/refresh` | 公开(凭 refresh) | **新增** | 轮转：旧 refresh 立即失效，返回新 access+refresh |
| POST | `/auth/logout` | Bearer/凭 refresh | 扩展 | **真注销**：撤销该 refresh 家族 + 可选 `all_devices=true` |
| POST | `/auth/verify` | Bearer | 保留 | 本地验签等价物（供无 SDK 的语言调用） |
| GET | `/auth/me` | Bearer | 扩展 | 加 `roles[]`、`permissions[]`、`avatar_url`、`token_version` |
| POST | `/auth/register` | 公开→**邀请制** | 变更 | 需 `invite_code`（`auth.enroll` 权限者可签发）；本机注册同步收紧（不再允许 1 位密码） |
| POST | `/auth/auto` | 仅回环+显式开关 | 收紧 | 返回 `role=service` 降权 token（§4.2 D） |
| GET | `/auth/authorize` | 公开(需登录) | **新增** | 授权码入口（浏览器 SSO） |
| GET | `/auth/sessions` | Bearer | **新增** | 我的登录设备列表 |
| DELETE | `/auth/sessions/{id}` | Bearer | **新增** | 踢掉某设备（消费方据此失败并重登） |
| GET | `/api/v1/.well-known/jwks.json` | 公开 | **新增** | 公钥集（离线验签）。**路径必须落在 nginx 已开放的前缀内**——根级 `/.well-known/` 不在路由白名单，公网取不到（§4.4 第 2 条、《Nginx反向代理机制》§4） |
| POST | `/auth/change-password` | Bearer | 保留(改名) | 成功后 `token_version+1` → 所有旧 access 失效 |

### 6.2 用户与资料

| method | path | 鉴权 | 状态 |
|---|---|---|---|
| GET | `/users` | 管理员 | 保留（加 `?q=&status=&role=` 过滤与分页） |
| GET/PUT | `/users/{id}` | 本人或管理员 | 保留（PUT 加 `roles[]`，管理员可改 status） |
| DELETE | `/users/{id}` | 管理员 | 保留（软删 + `token_version+1` + 撤销全部会话） |
| PUT | `/users/{id}/password` | 本人或管理员 | 保留 |
| GET/PUT | `/users/{id}/profile` | 本人或管理员 | **新增**（用户级偏好 KV，替代各包自建配置） |
| GET/PUT | `/users/{id}/avatar` | 本人或管理员 | **新增**（存 depot 或本地 `files`，返回 URL） |

### 6.3 角色与权限

| method | path | 鉴权 | 状态 |
|---|---|---|---|
| GET | `/roles`、`/permissions` | 管理员 | **新增** |
| PUT | `/roles/{code}/permissions` | 管理员 | **新增**（增删权限点，落审计） |
| PUT | `/users/{id}/roles` | 管理员 | **新增**（替代现在的 `PUT /users/{id}` 里混着的 role 写入） |

### 6.4 客户端与共享能力

| method | path | 鉴权 | 状态 | 说明 |
|---|---|---|---|---|
| GET/POST | `/clients`、`PUT/DELETE /clients/{id}` | 管理员 | **新增** | 应用注册（`client_id/secret/redirect_uris`） |
| GET/POST | `/favorites`、`PUT/DELETE /favorites/{id}` | Bearer | **新增（泛化）** | `scope` 参数区分包/域；`l_notepad` 版本转为兼容别名 |
| GET/PUT | `/credentials/custom-fields` | Bearer | **新增（泛化）** | 同上 |
| GET/POST | `/credentials`、`PUT/DELETE /credentials/{id}` | Bearer | **新增（泛化）** | `secret` 只写不读（返回 `****`）；读取需 `credentials.reveal` 权限且落审计 |
| GET/PUT | `/prefs/{scope}` | Bearer | **新增** | 用户级偏好（如 `prefs/l_agent_chat`） |
| GET | `/services/status` | 公开→**Bearer** | 收紧 | 端口清单不再裸奔 |

### 6.5 授权判定（供业务服务调用）

| method | path | 鉴权 | 状态 | 说明 |
|---|---|---|---|---|
| POST | `/authz/check` | Bearer(服务) | **新增** | 请求 `{subject, resource_kind, resource_id, perm}` → `{allow, reason, matched_rule}`；**离线降级**：调用方本地按 owner 快路径判定，只有跨用户/共享场景才回 auth |
| POST | `/authz/grant`、`/authz/revoke` | Bearer(资源 owner 或管理员) | **新增** | 写 `acl_grants` |
| GET | `/authz/grants?resource_kind=&resource_id=` | Bearer | **新增** | 列出某资源的授权（拥有者可见） |

**这条是补 P1-5 的关键**：`lugwit_baidu_netdisk` 的 `download/list/history` 在返回前调 `/authz/check`（或本地按"路径首段 = owner"快判），不再"登录即可读任意路径"。

### 6.6 审计

| method | path | 鉴权 | 状态 |
|---|---|---|---|
| GET | `/audit` | 管理员/auditor | **新增**（`?actor=&action=&from=&to=&limit=`） |
| GET | `/audit/me` | Bearer | **新增**（我看我自己的登录/敏感操作记录） |

### 6.7 废弃与兼容（过渡期 ≥ 2 个月）

| 旧 | 新 | 兼容做法 |
|---|---|---|
| `POST /api/v1/auth/login` 返回 `{token}` | 同端点加 `access_token/refresh_token` | 保留 `token` 字段（= access） |
| `GET /logout`（只清 cookie） | `POST /auth/logout`（撤销 refresh 家族） | 旧路由内部改调新逻辑，签名不变 |
| `l_notepad_accounts` / `fav-items` | `credentials` / `favorites`（`scope=l_notepad`） | 双写 + 兼容视图 |
| HS256 token | RS256 | 双验期：先按 `alg` 分支，JWKS 上线后逐步只发 RS256（§7） |

---

## 7. token 与密钥体系

| 项 | 现状 | 设计 |
|---|---|---|
| 签名 | HS256，对称，密钥多包共用 | **RS256**（`kid` 轮转；私钥仅 auth 可读，公钥走 `/.well-known/jwks.json`）；过渡期双验 |
| access TTL | 30 天 | **15 分钟** |
| refresh | 无 | **30 天**，一次性轮转（refresh family + 复用检测） |
| 吊销 | 无 | `users.token_version` + `sessions.revoked_at`；access 15 min 内自然过期，业务方对 401 立即重试一次 refresh |
| claims | sub/role/user_id/exp | `sub`(username)、`uid`、`roles[]`、`perms_hash`(可选)、`typ`(user/service)、`ver`(token_version)、`client_id`、`exp/iat/jti` |
| cookie | `httponly=False`、前端手写 | `lugwit_token`：**HttpOnly** + `SameSite=Lax` + `Secure`(HTTPS) + 统一由服务端设置/清除；CSRF 用双提交 token（读 cookie 的 JS 场景改为调 API） |
| 桌面存储 | token JSON + **明文密码** | access 仅内存；refresh 存 **DPAPI/Windows 凭据管理器**；永不落密码 |
| 密钥存放 | 默认硬编码 | `LUGWIT_AUTH_JWT_PRIVATE_KEY`（文件路径或 env，最低 2048-bit RSA）；启动自检"仍是默认值 → 拒绝启动并告警" |
| 服务间 | 无 | `clients.is_public=false` + `client_credentials` 授权签 `typ=service` token |
| 传输 | 443 单入口 + **IP 自签证书**（当前无域名，§4.5） | cookie 一律带 `Secure`（HTTPS 下）；客户端必须能找到 CA（`LUGWIT_CA_FILE` → 包内 `ca_bundle.pem` → …），**"校验失败→不校验重试"的降级分支不得成为常态**；浏览器 / QtWebEngine 走系统根库（`install_lugwit_ca.bat`） |

---

## 8. 与其他包的边界

| 能力 | 归属 | 备注 |
|---|---|---|
| 身份/令牌/会话/角色/权限/审计/凭据/收藏/偏好/头像/授权码 | **auth** | 本文 §6 |
| 桌面登录窗口 UI | `l_qframelesswindow`（组件） + `lugwit_auth.client`（逻辑） | UI 一份、逻辑一份 |
| 业务数据与资源 ACL 存储 | 各业务包（depot/notepad） | **判定**调 auth `/authz/check`，**存储**在业务侧或 auth 的 `acl_grants`（推荐后者的共享列表） |
| 服务进程监督（start/stop/restart） | **建议移出 auth**（交给 `l_tray` / `l_homepage`） | 现状在 auth 里（`auth_server.py:489-641`），与授权职责无关；若移出需保留 `/api/v1/services/status` 兼容一段时间 |
| 网页统一入口（nginx 路由）/HTTPS | 基础设施 | 见《Nginx反向代理机制》 |
| **Depot 的存储后端**（当前百度网盘 ↔ 未来自建云盘/公网服务器） | Depot 包内部（`get_backend()` 单例） | auth **只依赖 HTTP 契约**（`submit_stream`/`download`/`list`），不 import 任何云盘 SDK → **换后端 auth 零改动**（见《网盘版本库Depot设计》§12.7） |

---

## 9. 迁移路线（每阶段独立可上线 / 可回滚）

### P0 — 止血（0.5 天，不动接口签名）
1. `/auth/auto` 收紧：不再信任 `X-Forwarded-For`；返回 `role=service`；加显式开关 `LUGWIT_AUTO_AUTH_ENABLED`（默认关，本机脚本设 env）。
2. `services/status` 加鉴权。
3. cookie 统一由服务端设置（`HttpOnly`）+ 清理 domain/path 一致性；把前端 JS 写 cookie 的地方改为调 API。
4. `LUGWIT_AUTH_JWT_SECRET` 为默认值时启动告警（不阻断）。

**验收**：`curl -X POST http://127.0.0.1:1027/api/v1/auth/auto` 在未开开关时返回 403；带 `X-Forwarded-For: 127.0.0.1` 的外部请求也 403。

### P1 — 会话与吊销（1~2 天）
启用 `sessions`；加 `/auth/refresh`、`/auth/logout`（真撤销）、`/auth/sessions*`、`token_version` 字段；改密/禁用/删号即 `+1`。

**验收**：登录 → 改密 → 旧 access 访问 `/auth/me` 返回 401；`DELETE /auth/sessions/{id}` 后该 refresh 失效。

### P2 — 非对称签名与 JWKS（1 天）
RS256 + `/…/jwks.json`；`gate` 增加公钥缓存；双验期（HS/RS 并存，按 `alg` 分支）。

**验收**：拿旧 HS256 token 仍可访问；新 token 用 JWKS 在**离线**（断网）的情况下验签通过。

### P3 — RBAC + 授权判定（2~3 天）
建 `roles/permissions/role_permissions/acl_grants` + `/authz/*` + `/roles`、`/users/{id}/roles`；`gate` 增加 `require_perm()`。

**验收**：`/authz/check` 对 `depot.read /l_agent_chat/u01` 返回 `allow=true` 给 owner、`false` 给他人；管理员 `allow=true`。

### P4 — 共享能力泛化（2 天）
`credentials/favorites/user_profiles` 落地 + 旧表双写 + 兼容视图；`l_notepad_server` 切到新端点。

**含主密钥归位（§12 D4 / §5.3 D 类）**

1. **密钥存储抽象**：新增 `lugwit_auth.secret_store`，接口 `load_master_key()/store_master_key()`；实现两条：
   - `DpapiSecretStore`（Windows 默认；`CryptProtectData`，按当前用户+机器加密，不落明文）
   - `FileSecretStore`（Linux/无 DPAPI 环境；文件 0600 + 目录 0700，保留兼容）
2. **路径去 `l_notepad` 化**：主密钥从 `~/.lugwit/l_notepad/.accounts_key` 迁到 `~/.lugwit/lugwit_auth/master.key`；
   迁移脚本 `lugwit_auth_rotate_master_key`：读旧 key → 解密全部 `password_enc` → 用新 key 重加密 → 原子提交（失败回滚）→ 旧 key 改名 `.bak`
   （**注意：先确认 l_notepad 侧也已切到新接口，否则会把它读不了旧数据**）
3. **密钥与密文分机**（可选，二选一）：① 主密钥改由 `LUGWIT_AUTH_MASTER_KEY` 注入（部署时不落盘）；
   ② 保留落盘但用 DPAPI 包裹。二者都要求 §5.3 D 类的铁律不变：**密钥永不进 PG / Depot**。
4. **文档同步**：`account_service.py:8-9,25,28-36` 的注释与实现改成新路径；`l_notepad` 的共享密钥说明标注废弃期限。

**验收（P4 增量）**：
- 明文密码在磁盘上**搜不到**：`findstr /s /i "accounts_key"` 旧路径无引用；
- 迁移前后 `credentials` 全量解密校验一致（逐行 `_decrypt` 对比，0 差异）；
- 删除新 key 文件后服务拒绝解密并给出明确告警（不静默返回空密码）；
- `l_notepad` 旧端点读写同一份数据仍通过（与上面主验收同跑）。

### P5 — 统一登录窗口（2~3 天）
`lugwit_auth.client` 出 SDK（`login_via_browser/refresh/ensure_token/verify_local`）+ 回环回调；`l_qframelesswindow` 的 `LoginDialog` 改为只驱动 SDK；4 份旧实现逐步废。

**验收**：`l_notepad_client` 登录不再落盘密码；token 过期自动续期无感；登出后 refresh 也被撤销。

### P6 — 业务接入与 owner 强校验（按包推进）
`lugwit_baidu_netdisk` 的 `download/list/history` 接 `/authz/check`；`l_agent_chat` 的 depot 同步改为**带真实用户身份**（PKCE 或 `lugwit_token` 透传），不再吃回环 `system01`；`l_agent_chat` 的库/路径按用户分区（`/l_agent_chat_<user>` 或库内 `<user>/`）。

**验收**：A 用户登录后无法读取 B 用户的 `/l_agent_chat_b/**`（返回 403 而非 200/404 混淆）；`l_agent_chat` 的 depot 提交 owner 为登录用户而非 `system01`。

### P7 — 备份与灾备（1~2 天，依赖 Depot 侧前置）
按 §13 落地：`/l_auth_backup` 库（dir 模式）+ 每日/关键变更后异步 dump（本地 + Depot 双份，**密钥不入云**）+
`lugwit_auth_restore` CLI + **P-C/P-D/P-E 三条破环改造**（自签不自调、离线验签硬约束、恢复走 CLI）。
Depot 侧前置：补 `tools/depot_manifest_replay.py`、`depot_manifest_verify.py`（演进计划 T4）。

**验收**：① 停掉 Depot 后 auth 登录/改密/`/authz/check` 无感；② 空库 + 只给网盘 blob 与 manifest，用 CLI 完成 `users` 重建（§13.6 的 ①→⑥）；③ `latest.json` 的 checksum 校验能识别被篡改的 dump；④ 备份文件里**搜不到**私钥/主密钥。

---

## 10. 安全清单（威胁 → 对策）

| 威胁 | 对策 |
|---|---|
| 本机任意进程免密拿系统身份 | §4.2 D：显式开关 + 降权 `service` + 不信任 `X-Forwarded-For` |
| 令牌泄露 | access 15 min；refresh 轮转 + 复用检测；`sessions` 可踢 |
| 对称密钥泄露即全平台伪造 | RS256 + 私钥仅 auth；启动自检默认密钥 |
| 离职/停用账号继续访问 | `token_version` + 离线校验带 `ver` 比对（缓存"最低有效版本"≤60s） |
| Cookie 被 JS 读取（XSS 取 token） | `HttpOnly` + CSRF 双提交 + 严格 CSP |
| 桌面记住密码明文 | P5：改 PKCE 回环授权码，密码不落地 |
| 越权读他人 depot 路径 | `/authz/check` + 路径 owner 快判（P6） |
| 暴力破解 / 撞库 | 登录失败计数（`audit_logs` + 内存滑块）+ 可配置锁定阈值 |
| 审计缺失事后无法追责 | `audit_logs` 覆盖登录/失败/改密/角色变更/凭据读写/授权拒绝 |
| 配置/密钥散落各包 | `user_profiles` + auth 侧凭据托管统一收敛 |
| **反代下回环判定失真**（公网请求被判成本机 → 拿到 `system01`+admin） | `/auth/auto` 生产默认关；判定收紧为「peer ∈ {127.0.0.1, ::1} **且** 无 `X-Forwarded-For`/`X-Real-IP`/`Forwarded`」（§4.4 第 3 条、P0-1） |
| **中间人（自签证书降级被当常态）** | 客户端 CA 必须可找到且**启动即打印生效 CA 与是否走了降级分支**；`ssl_support` 的"不校验重试"只作保底并告警；浏览器/QtWebEngine 用 `install_lugwit_ca.bat` 装系统根库（只推可信机器） |
| **根证书私钥 = 服务器私钥**（自签 leaf 即 CA） | 拆分为「只签服务器证书的中间 CA + 短有效期」，私钥只留服务器（《HTTPS证书与域名申请总结》§11.3） |

---

## 11. 验收测试清单（curl 级，可直接抄）

```bash
A=http://127.0.0.1:1027
# P0
curl -s -X POST $A/api/v1/auth/auto                                   # 期望 403（未开开关）
curl -s -H "X-Forwarded-For: 127.0.0.1" $A/api/v1/services/status      # 期望 401
# P1
TOK=$(curl -s -X POST $A/api/v1/auth/login -H 'Content-Type: application/json' \
      -d '{"username":"admin01","password":"lugwit123"}' | python -c "import sys,json;print(json.load(sys.stdin)['access_token'])")
curl -s -X POST $A/api/v1/auth/refresh  -H "Authorization: Bearer $TOK" \
      -H 'Content-Type: application/json' -d '{"refresh_token":"..."}'  # 期望新 access+refresh
curl -s $A/api/v1/auth/sessions -H "Authorization: Bearer $TOK"        # 期望设备列表
# P2  离线验签（断网仍通过；生产走单入口时把 $A 换成 https://<入口>/api/v1）
curl -s $A/api/v1/.well-known/jwks.json
# P3  授权判定
curl -s -X POST $A/api/v1/authz/check -H "Authorization: Bearer $TOK" -H 'Content-Type: application/json' \
      -d '{"subject":"u01","resource_kind":"depot","resource_id":"/l_agent_chat_u01","perm":"read"}'
# P6  越权必须 403
curl -s -o /dev/null -w '%{http_code}\n' "http://127.0.0.1:8080/baidu/api/depot/download?path=/l_agent_chat_u02/settings.json&rev=0" \
      -H "Cookie: lugwit_token=<u01-token>"                             # 期望 403

# P0  传输与信任（无域名阶段；入口 = https://121.196.144.88）
curl.exe -s --cacert C:/certs/lugwit/ca.pem -o NUL -w '%{http_code}\n' https://<入口>/nginx-health
#   期望 200 —— 能带 CA 校验通过 = 客户端信任链正确；若报 self-signed → 说明没装 CA（去跑 install_lugwit_ca.bat）
#   自检口径：客户端/服务日志里**不该**出现"HTTPS 证书：… 降级不校验"的告警，出现即代表 CA 未生效（§4.5）
#   浏览器侧：装根库后重启浏览器，再开 https://<入口>/baidu/ 应无告警（QtWebEngine 需重启客户端）
```

---

## 12. 待确认决策

| # | 决策点 | 备选 | 建议 |
|---|---|---|---|
| D1 | 签名算法是否换 RS256 | 换 / 保持 HS256 | **换**（P2），双验期过渡 |
| D2 | 服务进程监督是否移出 auth | 移出 / 保留 | **移出**到 `l_tray`，auth 只留 status 兼容 |
| D3 | 注册策略 | 开放 / 邀请码 / 管理员建号 | **邀请码**（`/auth/register` 需 `invite_code`） |
| D4 | 桌面凭据存储 | DPAPI / 钥匙串 / 继续 JSON | **DPAPI**（Windows 主战场）+ `FileSecretStore` 兜底；**主密钥路径去 `l_notepad` 化**（`~/.lugwit/lugwit_auth/master.key`）+ 轮转脚本，落地步骤见 P4 |
| D5 | owner ACL 存哪 | auth `acl_grants` / 各业务自管 | **auth 统一**（跨资源可复用，共享列表集中） |
| D6 | depot 路径分区 | 一人一库 `/l_agent_chat_<user>` / 单库 `<user>/` | **单库 + `<user>/` 子目录**（库登记（`depot_library`）成本低、`dir` 模式可见性好）；若要 blob 物理隔离再改一人一库 |
| D7 | 停用账号的离线校验时效 | 立即 / ≤60s | **≤60s**（auth 维护"最低有效 token_version"缓存，业务方拉取） |
| D8 | auth 自身数据备份上云的范围 | 全量含凭据密文 / 不含密钥 / 只备元数据 | **数据全量上云，签名私钥与加密主密钥永不进 depot**（否则 depot 泄露 = 可伪造身份） |
| D9 | 是否补 `depot_manifest_replay.py` | 先补 / auth 备份先上 | **先补**（演进计划 T4 的待办）—— 它是"灾备承诺"能成立的前置，否则 DB 丢失时 Depot 与 auth 一起不可恢复 |
| D10 | 浏览器 / QtWebEngine 的证书信任路线 | 装系统根库 / 接 `certificateError` / 等域名 | **装系统根库**（`l_nginx/999.0/tools/install_lugwit_ca.bat`，一次覆盖浏览器 + QtWebEngine + 托盘 Python；只推可信机器）；`certificateError` 作兜底；域名 + LE 是终局（§4.5） |
| D11 | 自签 CA 是否拆中间 CA | 拆 / 保持"leaf 即 CA" | **拆**：现在服务器证书私钥就是 CA 私钥，泄露即可全平台中间人；改为"只签服务器证书的中间 CA + 短有效期"，私钥用途分离（§10） |

---

## 13. 与百度云 Depot 的依赖关系：问题与破环

### 13.1 问题陈述

**待答问题**：auth 的用户/凭据/审计数据想存进百度云 Depot，但 Depot 的每个 HTTP 请求都过
`gate.require_lugwit_token`（授权由 auth 提供）—— 是否存在"蛋和鸡"？

**结论先说**：**不是硬死锁**（auth 是令牌签发者，且 Depot 本地验签），但存在 **两处真环 + 一个未实现的工具缺口**。
现有文档只写了**半个前提**，见 §13.2。

### 13.2 现有文档的覆盖情况

| 文档 | 说了什么 | 缺什么 |
|---|---|---|
| `Rez_pkg/lugwit_baidu_netdisk.md` §2.3 | 每个 depot HTTP 请求都过 `gate.require_lugwit_token`（**本地验 JWT，不依赖认证服务在线**）；本机请求会自动去 `/api/v1/auth/auto` 换 token | 没把"离线验签"和"auth 自己存数据"关联；`/auto` 那句恰恰是弱环来源 |
| `网盘版本库Depot设计.md` §4 | manifest 是兜底：「数据库整个丢了，按 CL 号顺序回放这些 json 就能重建」 | 重建顺序与鉴权的关系没写 |
| `网盘版本库Depot演进计划.md` T4 | 计划 `tools/depot_manifest_replay.py --from <cl_id>`（*DB 灾难恢复用*）；执行前自动 `tools/depot_dump_tables.py` dump 到 `~/.lugwit/depot_backup/<stamp>/` | `depot_manifest_replay.py` / `depot_manifest_verify.py` **在 `tools/` 里并不存在**（仍是计划，T4 目标只到 dry-run + 生成 SQL） |
| `lugwit_auth` 侧 | —— | **完全没有**"auth 自身数据备份/恢复"的任何设计 |

### 13.3 逐层判定：哪些是真环

| 层 | 有环？ | 机理与证据 |
|---|---|---|
| 库依赖（import） | ❌ 无 | Depot 的 `gate.py` 只 import auth 的验签库（`jwt_service`），方向单向 |
| 运行期验签 | ❌ 无 | 本地验签 → auth 进程挂了也能验**已有** token（文档那句的本意，也是必须守住的硬约束） |
| 取新 token | ⚠️ **弱环** | `/api/v1/auth/auto`（回环自动授权）需要 auth 在线；auth 自己的备份任务若靠它取 token，就变成自我依赖。**auth 是签发者，本地就能签** → 应自签 |
| 同进程自调用 | ✅ **真环（自阻塞）** | auth 进程内挂载 `/baidu`（`auth_server.py:139,167`）。auth 若用阻塞 `requests` 调自己的 `/baidu/api/depot/submit_stream`，而该端点由**同一个事件循环**提供 → 请求永远等不到响应（自锁） |
| 灾备冷启动 | ✅ **真环** | Depot 元数据（`depot_library` / `depot_file_rev` / `depot_blob` / `depot_have` / `depot_lock`…）与 auth 的 `users` **同在 Postgres `chatroom`**（`lugwit_auth/.../config.py:28-33` vs `lugwit_baidu_netdisk/.../depot_store.py`）→ 库崩了两者一起没；而"从 Depot 恢复 `users`"又要求 Depot 先能服务（要有元数据 + 过闸门） |

### 13.4 破环原则（问题 → 方案）

| # | 问题 | 方案 | 解掉哪个环 | 验收 |
|---|---|---|---|---|
| **P-A 自签不自调** | auth 取 token 依赖 `/auth/auto`；HTTP 自调会自锁 | auth 需要 token 时本地 `_token_for(..., typ=service)`；需要 Depot 能力优先**进程内函数调用**（`import depot_service`），绝不 HTTP 自调自己的挂载点 | 弱环 + 同进程自阻塞 | 断掉 auth 的 `/api/v1/auth/auto` 后，auth 的备份任务仍能完成一次 submit |
| **P-B 离线验签是硬约束** | auth 离线时业务全废 | 业务包永不要求"auth 在线"才能验 token；签名密钥（RS256 公钥 / JWKS 缓存）必须**本地文件可得**，不得"只能从 auth 取"；JWKS 刷新失败时用缓存继续判定 | 运行期环 | 停掉 auth 进程，业务包用有效期内的 token 仍返回 200 |
| **P-C 备份与库解耦** | 恢复 `users` 要先有 Depot 元数据，元数据又和 `users` 同库 | auth 关键数据导出为**文件**（gzip json + schema_version + 校验和）：本地一份 + 作为 Depot blob 上传到固定路径 | 灾备环 | 在**空库**上仅凭网盘 blob + manifest 重建出 `users` |
| **P-D 恢复走 CLI，不走 HTTP** | HTTP 端点要过闸门，而没有 `users` 就发不出可信 token | 恢复路径全部走 CLI/本地：`depot_dump_tables.py`（已有）→ `depot_manifest_replay.py`（**待补**）→ `lugwit_auth_restore` CLI（新增）；服务侧恢复入口只允许「免鉴权 + 仅回环 + 显式确认」 | 灾备环 | 断网+断 auth，仅本机 CLI 完成一次全量恢复演练 |
| **P-E 顺序 + 容错** | 谁先起、谁卡谁 | 启动顺序 Postgres → auth → 业务（Depot 可先于 auth 起，只是拿不到新 token）；auth 的备份任务**异步 + 失败重试 + 不阻塞认证主流程**，Depot 不可用时 auth 照常提供认证 | 全部 | Depot 停掉后 auth 登录/改密/`/authz/check` 无感（只在日志留告警） |

### 13.5 目标态依赖方向

```
  自签 token / 进程内函数调用（不经 HTTP）
 lugwit_auth(1027) ─────────────────────────────► Depot + 百度网盘(1028)
      │  ▲                                              │
      │  │                                              │
      │  └── ② 业务包本地验签（同一密钥/公钥，auth 离线也算数）
      │                                                 │
      ▼                                                 ▼
     签发/撤销 ──────────────► 业务包(8765/1250/…) ◄──── ① 本地验签

 灾备链（全程无 HTTP 端点、无闸门、无 auth）：
   网盘 blob(外部) → manifest 文件 → CLI replay 重建元数据 → 空库 → auth dump → users
```

### 13.6 启动顺序 / 恢复顺序

**启动**（顺序不影响可用性，只影响"能否拿新 token"）：

```
Postgres(chatroom) → lugwit_auth(1027) → lugwit_baidu_netdisk(1028) → 业务(8765/1250…)
                      └─ 可后起：Depot 靠本地验签先提供读；只是拿不到新 token
```

**灾备恢复**（顺序不可颠倒，且 ③⑤ 必须在没有 `users` 表的状态下完成）：

```
① 百度网盘 blob 内容        外部存储，不依赖任何本机服务
② .depot/manifest/<桶>/<cl_id>.json   提交兜底清单（本地/云端）
③ CLI replay 重建 Depot 元数据          ← 需要补 depot_manifest_replay.py；免鉴权
④ Depot 可服务（本地验签，无需 auth 在线）
⑤ 从 Depot 拉 auth dump → 重建 users/roles/credentials/audit
⑥ 用户重新登录
```

### 13.7 auth 自身数据的备份设计（新增能力）

| 项 | 设计 |
|---|---|
| 备份对象 | `users`、`user_roles`、`roles`、`role_permissions`、`permissions`、`sessions`、`clients`、`audit_logs`、`credentials`（密文原样）、`favorites`、`user_profiles` |
| **不进备份** | 签名私钥 / 加解密主密钥（`LUGWIT_AUTH_JWT_PRIVATE_KEY`、Fernet 主密钥）——**必须走离线人工保管**；否则 Depot 泄露等于可伪造全平台身份 |
| 格式 | 单文件 `<stamp>.authdump.json.gz`：`{schema_version, exported_at, tables:{...}, checksum}` |
| 路径 | `/l_auth_backup/<YYYYMMDD-HHMM>/authdump.json.gz` + `latest.json`（指针，含 stamp/checksum/size） |
| 触发 | 每日一次 + 关键变更后（改密 / 角色变更 / 新增用户 / 密钥轮转）；异步、重试 3 次、失败只告警 |
| 本地副本 | `~/.lugwit/auth_backup/<stamp>/`（与 Depot 双份，互为兜底） |
| 权限 | 备份库 `/l_auth_backup` 设为 **dir 模式**（可见可人工取回）；仅 auth 的服务身份可写（`/authz/grant` 授权），管理员可读 |
| 恢复 | 新增 CLI `lugwit_auth_restore --from <dump|latest>`：校验 checksum → 事务内 upsert → 输出差异报告；**不提供 HTTP 恢复端点**（避免"恢复要先登录"的环） |
| 演练 | 季度一次：空库 + 新库名，跑完整 ①②③④⑤⑥，记录耗时（写进运维手册） |

### 13.8 与 §9 路线图的衔接

本节的落地项并入路线图 **P7（备份与灾备）**，其中 `depot_manifest_replay.py` 属 Depot 侧前置（对应 D9）。

---

## 附录 A：现状证据索引（用于回归比对）

| 事实 | 证据 |
|---|---|
| 唯一签发点 `_token_for` | `R/lugwit_auth/999.0/src/lugwit_auth/auth_server.py:276-280` |
| 30 天有效期 | `.../jwt_service.py:15` |
| 硬编码默认密钥 | `.../config.py:35-38` |
| cookie 属性 | `auth_server.py:663-665`；清 cookie `:706-708` |
| 回环自动授权 | `auth_server.py:174-192`、`:712-720` |
| 登录页路由 | `auth_server.py:669-688`；`templates/login.html:84,106,74-76` |
| 用户中心 / 设置页 | `auth_server.py:723-794` |
| 用户 CRUD | `auth_server.py:1016-1085`；`user_service.py:44-254` |
| 角色硬编码 | `auth_server.py:57`；`enums.py:12-20` |
| `sessions` 空转 | `models.py:43-58`（仅 `init_db.py:8` 引用） |
| 收藏/凭据表 | `account_service.py:87-135` |
| 服务监督端点 | `auth_server.py:317,489-641` |
| 消费方闸门 | `R/lugwit_baidu_netdisk/999.0/src/lugwit_baidu_netdisk/gate.py:22-37` |
| login 只验签名 | `.../gate.py:31-37` |
| 其他消费点 | `lugwit_baidu_netdisk/.../web_server.py:386,406`；`l_WChat/.../api/auth.py:37-70`；`l_notepad_server/.../auth.py:26-98`；`l_qframelesswindow/.../login_store.py:46-60`；`l_notepad_client/.../api_client.py:104-107` |
| 桌面登录窗口（明文密码） | `l_qframelesswindow/.../login_dialog.py`、`login_store.py:135-152`；`l_tray/.../lugwit_login/loginUI.py` |
| depot 读接口不校验 owner | `R/lugwit_baidu_netdisk/.../web_server.py:2859-2880` |
| 闸门语义（页面 302 / API 401） | `R/Rez-Docs/Rez_pkg/lugwit_baidu_netdisk.md:86` |
| 标题栏定位（认证接入能力，不弹框） | `R/Rez-Docs/标题栏提供的服务.md:37-46,69-71` |

## 附录 B：本文未做的事 / 后续

- 未写 Python 实现（本文只是设计；落地按 §9 分阶段提单）。
- 未定 `git` / 代码评审流程细节。
- 未覆盖移动端（`mobile_msg_height` 等 UI 偏好归 `user_profiles`）。

**待实测 / 待补（本会话新增，避免遗忘）**

| 项 | 现状 | 归属 |
|---|---|---|
| 装系统根库是否一次解决「浏览器 + QtWebEngine」 | **未在真机验证**（依据：Chromium 在 Windows 上用系统根库） | `install_lugwit_ca.bat` 已就绪，待真机跑一次（§4.5、D10） |
| `LUGWIT_DEPOT_ROOT_PREFIX` 显式化 | 仍是"从百度 token 派生 `apps_root`"的隐式耦合 | 《网盘版本库Depot设计》§12.5 / §12.8 |
| `depot_manifest_replay.py`、`depot_manifest_verify.py` | **不存在**（演进计划 T4 仍是计划） | 它是 §13 灾备承诺的前置（D9） |
| `ssl_support` 启动即打印"生效 CA + 是否降级" | 现在只在失败时告警一次，易被日志淹没 | §4.5「现在就该做的一件事」（约 10 行改动） |
| 主密钥 DPAPI 化 + 去 `l_notepad` 路径 | 仍是 `~/.lugwit/l_notepad/.accounts_key` | §9 P4 增量 / D4 |
| `/auth/auto` 收紧（不信任 XFF + 降权 + 开关） | 仍是 `auth_server.py:174-192` 的宽松实现 | §9 **P0**（最高优先） |
