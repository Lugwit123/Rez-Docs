# lugwit_auth 统一用户授权服务设计

> 状态：**设计稿（待评审）**，2026-09-19；**P0–P5 已完成并跑完验收**，
> **P6 读/写侧 + 消费方收尾完成**，**P7 逻辑备份/恢复 CLI 完成（Depot 侧前置脚本未做）**，2026-09-20（见 §9）
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
| `l_notepad_accounts` / `l_notepad_custom_fields` / `l_notepad_fav_items` | 账号收藏 + 自定义字段 + 通用收藏（密码 Fernet 加密）**→ 已被 P4 的 `credentials`/`favorites`/`user_profiles` 取代，旧表降级为冻结快照** | 旧 `account_service.py:87-135`（已删除） |
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
| 第三方账号收藏 + 密码加密托管 | ✅（**信封加密**：每行 DEK + KEK 包 DEK）。演进：`~/.lugwit/l_notepad/.accounts_key`（单层）→ 2026-09-20 主密钥归位 `~/.lugwit/lugwit_auth/master.key`（DPAPI）→ **2026-09-22 升级为信封加密**，详见 §9 P4 / **P4.5** | **表名/API 命名绑定 `l_notepad`**，其他包用不了 |
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
| 凭据密文在 PG、Fernet 主密钥在本机文件（**现状：`~/.lugwit/lugwit_auth/master.key`，DPAPI 包裹**；旧路径 `~/.lugwit/l_notepad/.accounts_key` 已 `.bak`） | ✅ 方向正确且已归位：路径不再绑死 `l_notepad`、DPAPI 按"用户+机器"加密（拷走文件换机/换用户也解不开）、可选 `LUGWIT_AUTH_MASTER_KEY` 注入不落盘；残留风险是**同机同用户被入侵**仍可解（要更高强度得走离线介质签名，见 §12 D4） |
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

> **落地状态（2026-09-20）**：P0 / P1 / P2 / P3 **已实现并跑完验收**；
> P4–P7 未开始。已落地的代码位置：`auth_server.py`（端点）、`jwt_keys.py`（RS256+JWKS）、
> `jwt_service.py`（双验）、`schema_upgrade.py`（幂等补列/建表）、`session_service.py`（会话/轮转/吊销）、
> `authz_service.py`（RBAC + ACL 判定）、`seed_roles.py`（角色种子 CLI）、
> `client.py`（离线验签）、`templates/login.html` + `home.html`（不再用 JS 读写 cookie）、
> `lugwit_baidu_netdisk/gate.py`（`require_perm()` + 离线降级）。
> 两个落地时的口径调整：
> 1. **access TTL 保持 30 天**（未收短到 15 分钟）：P5 的 SDK `refresh/ensure_token` 未落地前，
>    收短会让所有现存客户端与 7 天 cookie 集体 401。吊销首发靠 `token_version`（P1 已生效）。
> 2. **refresh 复用检测额外递增 `users.token_version`**：access 仍是 30 天时，只吊销 refresh
>    挡不住已被偷走的 access；检测到复用时连 access 一起作废（与 §7「吊销靠短 TTL + 版本号」同源）。
>
> ⚠️ 行为变更（P0 生效后）：`LUGWIT_AUTO_AUTH_ENABLED` **默认关** → `/api/v1/auth/auto`、
> `/login`、`/`、`/settings` 的**回环免登录全部需要显式开开关**。依赖它的本机消费方
> （`lugwit_baidu_netdisk/web_server.py:_auto_local_token`、`l_tray/depot_bridge.py`、
> `l_notepad_server/depot_map.py`）需在被启动的环境里设 `LUGWIT_AUTO_AUTH_ENABLED=1`，
> 否则改用 `LUGWIT_ACCESS_TOKEN`（l_scheduler 会注入）。

### P0 — 止血（0.5 天，不动接口签名）✅ 已实现
1. `/auth/auto` 收紧：不再信任 `X-Forwarded-For`；返回 `role=service`；加显式开关 `LUGWIT_AUTO_AUTH_ENABLED`（默认关，本机脚本设 env）。
2. `services/status` 加鉴权。
3. cookie 统一由服务端设置（`HttpOnly`）+ 清理 domain/path 一致性；把前端 JS 写 cookie 的地方改为调 API。
4. `LUGWIT_AUTH_JWT_SECRET` 为默认值时启动告警（不阻断）。

**验收**：`curl -X POST http://127.0.0.1:1027/api/v1/auth/auto` 在未开开关时返回 403；带 `X-Forwarded-For: 127.0.0.1` 的外部请求也 403。

### P1 — 会话与吊销（1~2 天）✅ 已实现
启用 `sessions`；加 `/auth/refresh`、`/auth/logout`（真撤销）、`/auth/sessions*`、`token_version` 字段；改密/禁用/删号即 `+1`。

**验收**：登录 → 改密 → 旧 access 访问 `/auth/me` 返回 401；`DELETE /auth/sessions/{id}` 后该 refresh 失效。
**实测**：27 项端到端断言全绿（注册/登录 → cookie-only 认证 → 会话列表 → refresh 轮转 →
旧 refresh 复用被拒且连 access 一起作废 → 改密后旧 access 401 → 登出后 refresh 失效）。

### P2 — 非对称签名与 JWKS（1 天）✅ 已实现
RS256 + `/…/jwks.json`；`gate` 增加公钥缓存；双验期（HS/RS 并存，按 `alg` 分支）。

**验收**：拿旧 HS256 token 仍可访问；新 token 用 JWKS 在**离线**（断网）的情况下验签通过。
**实测**：HS256 存量 token 本地验签通过；RS256 新 token 读本机 `~/.lugwit/lugwit_auth/jwks.json`
离线验签通过（auth 指向黑洞端口仍通过）；篡改 token 被拒。私钥落在密钥目录（0600，不进 PG/Depot）。

### P3 — RBAC + 授权判定（2~3 天）✅ 已实现
建 `roles/permissions/role_permissions/acl_grants` + `/authz/*` + `/roles`、`/users/{id}/roles`；`gate` 增加 `require_perm()`。

**验收**：`/authz/check` 对 `depot.read /l_agent_chat/u01` 返回 `allow=true` 给 owner、`false` 给他人；管理员 `allow=true`。
**实测**：P3 30 项 HTTP 断言 + gate 判定链 7 项全绿（角色字典/权限点 19 个、
改角色权限 200/未知权限点 422/未知角色 404、改用户角色后旧 access 401、
owner/具名授权/`*`/owner 快路径/管理员/无记录/代查 403、授权与撤销的所有权校验）。

落地时补的两条口径（设计原文没写细）：
1. **ACL 动作匹配**：`acl_grants.perms` 是粗粒度动作（`read,write,admin`），而 `/authz/check`
   的 `perm` 是权限码（`depot.read` / `depot.write.any`）→ 匹配时取权限码的**第二段**作动词
   （`depot.write.any`→`write`；`auth.users.read`→`users`，不会误命中 `read`），ACL 含 `admin` 视为全权。
2. **`/authz/check` 调用方约束**：设计写「Bearer(服务)」，但 P0 已把 `/auth/auto` 默认关、
   服务令牌引导路径没了 → 放宽为「服务身份(`typ=service`) **或**管理员可代查任意 subject，
   普通用户只能查自己」（否则它就是个权限探测接口）。

**未做（留给 P6）**：`audit_logs` 表与"改角色权限落审计"（§6.3 尾部要求）—— 与 §6.6 的
审计接口一起做，避免出现"只对角色改动记审计、登录/凭据全不记"的半截审计。

### P4 — 共享能力泛化（2 天）✅ 已实现（表/端点/迁移/密钥全做完）

#### 已完成：泛化表与端点（credentials / favorites / user_profiles）

1. **表**（`schema_upgrade.py`，幂等建表）：`credentials`（`secret_enc bytea`）、`favorites`、
   `user_profiles`（KV，主键 `(username, key)`）。
2. **服务**：`credential_service.py`（asyncpg，owner+scope 隔离；加解密复用 `secret_store`
   的主密钥，fail-closed）。
3. **端点**（`auth_server.py`）：`GET/POST /credentials`、`PUT/DELETE /credentials/{id}`、
   `POST /credentials/{id}/reveal`、`GET/PUT /credentials/custom-fields`、
   `GET/POST /favorites`、`PUT/DELETE /favorites/{id}`、`GET/PUT /prefs/{scope}`、
   `GET/PUT /users/{id}/profile`；均支持 `?scope=`。
4. **数据迁移**：`lugwit_auth_migrate_credentials`（`migrate_credentials.py`，幂等 + 标记位）：
   `l_notepad_accounts`→`credentials`（12 行，密文原样搬，同密钥不需重加密）、
   `l_notepad_fav_items`→`favorites`（9 行）、`l_notepad_custom_fields`→
   `user_profiles['credentials.custom_fields:l_notepad']`。
5. **旧端点不双写，改「同一份数据」**（**偏离设计原文，理由写在这**）：
   设计原写"旧表双写 + 兼容视图"，但旧表的**唯一读者是死代码**（`l_notepad_*/account_store.py`），
   双写只会造出两份真相 + 一套同步逻辑。改为：旧 `/accounts*`、`/fav-items*` **URL 与返回结构不变**，
   内部改调新的 `credential_service`（`scope='l_notepad'`，`reveal=True` 回明文 `password` 字段），
   旧表**保留为只读历史**（不再写入、不删）。
6. **口径（新增，设计未细化）**：
   - 凭据/收藏/偏好一律**按 owner 收紧**（管理员也不跨用户读写；跨用户管理留给 `/authz/*`）；
   - `reveal`：**本人凭据可直接读**（owner 即信任边界）；**跨用户**读需 `credentials.reveal` 权限，
     且 P6 审计上线后必须落审计；
   - `/credentials` 列表里 `secret` 恒为 `****`（只写不读），明文只能经 `reveal` 拿；
   - `/users/{id}/profile` 本人或管理员（与他人 profile 的读取不同，这是设计 §6.1 明确写的）。

**验收（P4 泛化部分）实测**：HTTP 断言 **31/31 通过**
（掩码/scope 隔离/reveal 两种来源/空 secret 不改密文/自定义字段名/收藏过滤/
prefs 往返/profile 越权 403/新旧端点互见同一份数据/owner 域边界）。

#### 已完成：主密钥归位（§12 D4 / §5.3 D 类）

1. **密钥存储抽象**：新增 `lugwit_auth/secret_store.py`：`load_master_key() / store_master_key()`；
   两条实现合并在同一模块：
   - **DPAPI**（Windows 默认；`CryptProtectData`，纯 ctypes **不引 pywin32**，按当前用户+机器加密）
   - **文件**（Linux/无 DPAPI；key 明文 + 文件 0600 / 目录 0700）
   - 来源优先级：`LUGWIT_AUTH_MASTER_KEY`（env，不落盘）> `<key_dir>/master.key` > 自动生成
2. **路径去 `l_notepad` 化 —— 无兼容回退**：运行时只认 `~/.lugwit/lugwit_auth/master.key`；
   旧路径 `~/.lugwit/l_notepad/.accounts_key` 仅由 `load_legacy_key()` 暴露给**迁移脚本**读取，
   **不参与密钥解析**。能这么干的前提已核实：`l_notepad_client/l_notepad_server` 的
   `account_store.py` 是**死代码**（全仓无 import，账号功能早已走 auth 的 `/accounts*`），
   没有活跃消费方读旧密钥 → 无需保留过渡分支。
   **一次性轮转已执行（2026-09-20）**：`lugwit_auth_rotate_master_key --apply --no-legacy-consumers`
   → 4 行 `custom_fields` 重加密、单事务提交、`master.key.prev` 留档、
   旧文件改名 `.accounts_key.bak`。
3. **`--apply` 安全阀**：不带 `--no-legacy-consumers` → 退出码 4、拒绝写库；
   默认 dry-run（逐行 `decrypt(旧)→encrypt(新)→decrypt(新)` 比对）。
4. **fail-closed**：主密钥不可用时 `_decrypt/_encrypt` **抛 `MasterKeyUnavailable`**
   （不再把密文当明文回吐、也不静默返回空密码），服务侧统一转 **503 + 明确 detail**。
5. **文档同步**：`.env` 增注释；旧 `account_service.py`（legacy 表专用实现）已随死代码清理删除。
6. **顺带修掉的回归**：P0 关掉 `/auth/auto` 后，auth 自己的每日 DB dump 任务靠该端点兜底拿身份 →
   401。按 §13.4 P-A「自签不自调」落地：新增 `lugwit_auth/tokens.py`（`auth_server` 与
   `db_backup` 共用签发），`db_backup._token()` 改签 **降权服务令牌**（RS256 + `typ=service`），
   且 `_http` 同时带 `Authorization` 与 `lugwit_token` cookie（netdisk 的 `_current_user` 只认 cookie）。

**验收（P4 增量）实测**
- ✅ dry-run：`l_notepad_accounts` 12 行、需重加密 4 行（custom_fields 里的密文值）、**回读差异 0**；
- ✅ 安全阀：`--apply` 不带确认 → 退出码 4、拒绝写库；
- ✅ DPAPI 往返一致、env 注入优先、`describe()` 能报出实际来源；
- ✅ 无主密钥时加/解密都抛错（不静默），服务返回 503；
- ✅ **轮转已落地并复核**：新密钥可解 6 个密文值、旧密钥对这 6 个值**已全部失效**
  （证明确实重加密）、`.bak` / `master.key.prev` 均在位、`secret_store` 不再导出 `legacy_in_use`；
- ✅ 自签服务令牌被 netdisk 接受（`_remote_exists` 不再 401），备份调度已正常启动；
- ⚠️ `custom_fields` 里另有 **3 个历史遗留密文**（异机/旧密钥写入）解不开 —— 改动前同样解不开，
  非本次回归；当前 UI 会原样显示密文串，待人工清理或忽略；
- ✅ 「`accounts_key` 全仓无引用」**已签收**：三个死文件（`l_notepad_client/account_store.py`、
  `l_notepad_server/account_store.py`、auth 的 `account_service.py`）已删除，
  代码里 0 引用（唯一命中是 `secret_store.py` 顶部一段"为何没有回退"的历史说明）；
- ✅ 活表 `credentials` 里 3 条不可解密的历史残值已清理，复核后**0 个不可解密值**；

#### 遗留（不属于本阶段阻塞项）

- `l_notepad_server` / `l_notepad_client` 仍走**旧 URL**（`/accounts*`、`/fav-items*`）——
  功能正常（已指向新表）；切到 `/credentials*`、`/favorites*` 属 P5/P6 的收尾，可随时做。
- 旧表 `l_notepad_accounts` / `l_notepad_custom_fields` / `l_notepad_fav_items` 保留为
  **冻结快照**（只读、不再写入），数据核验无误后可择期删表；快照里有 3 个历史残值密文
  （异机/旧密钥写入，不可恢复）未清 —— 活表 `credentials` 里的同类残值已清理。

### P4.5 — 信封加密与主密钥治理（2026-09-22）✅ 已实现并上线生产

#### 事故：生产 `/api/v1/accounts` 503「凭据主密钥不可用」

**现象**：生产认证服务（`121.196.144.88`）`GET /api/v1/accounts` 返回
`503 {"detail":"凭据主密钥不可用，已拒绝解密/写入: 凭据解密失败: "}`；账号收藏在客户端拿不到。

**根因（两个，叠加）**：
1. **主密钥不匹配**：`credentials` 的密文是用**开发机** `~/.lugwit/lugwit_auth/master.key`
   （指纹 `7ffa563119e0`）加密的，而生产 auth 用**它自己的** DPAPI `master.key`
   （指纹 `b8213eb2cb64`）→ 解不开 → fail-closed 503。
2. **dev / prod 共用同一个 PG**：本机开发实例 `lugwit_auth.auth_server`（`127.0.0.1:1027`）
   的 `DEFAULT_DB_URL` 硬编码指向**生产库**（`config.py:37`），于是开发实例用本机密钥
   把行写进了生产库；生产实例拿另一把密钥 → 读不了。**这是最该根治的一条。**

**即时修复**：把生产 `master.key` 对齐为数据密钥（备份原文件后写 `RAW1\n<key>`），
生产 auth 热重载（touch 源文件触发 `SrcWatchService`）后 503 消失、12 条账号恢复。

#### 方案：从「单层 Fernet」升级为「信封加密（Envelope Encryption）」

单层模型（旧）：`secret_enc = Fernet(master.key).encrypt(明文)`——主密钥直接加密业务数据，
多实例必须各持同一把主密钥、轮转必须重加密全部业务密文。

信封模型（新）：
```
每行随机 DEK（Fernet key）
secret_enc    = Fernet(DEK).encrypt(明文)
custom_fields = {k: Fernet(DEK).encrypt(v)}          # 每行共用一把 DEK
dek_wrapped   = Fernet(KEK).encrypt(DEK)             # 只有 ~100 字节
kek_id        = sha256(KEK)[:12]                     # 记是哪把 KEK 包的
```
- **KEK**（密钥加密密钥）= `secret_store` 的主密钥（env `LUGWIT_AUTH_MASTER_KEY` 或 `master.key`）；
- **DEK**（数据加密密钥）= 每行一把，只在解包后驻内存；
- 全系统需要共享的秘密从"整库加解密"缩小为**一把 KEK**；**轮转只重包 DEK**，业务密文零改动。

**改动文件**（`lugwit_auth/999.0/src/lugwit_auth/`）：
| 文件 | 内容 |
|------|------|
| `schema_upgrade.py` | `credentials` 增列 `dek_wrapped bytea`、`kek_id text`（幂等，启动自动补） |
| `secret_store.py` | **KEK 密钥环**：`kek_id()`、`primary_kek()`、`load_keyring()`（env + `master.key` + `keys/*.key`）、`export_kek()`、`import_kek()` |
| `credential_service.py` | 信封-only：`_encrypt_row()` 生成 DEK 并整行加密；`_row_dek(row)` 按 `kek_id` 解包 DEK；更新沿用本行 DEK；**无兼容回退**——未迁移行（`dek_wrapped IS NULL`）直接抛 `MasterKeyUnavailable`（fail-closed） |
| `migrate_to_envelope.py`（新） | 单层→信封**一次性回填**：默认 dry-run，`--apply` 单事务 + 逐行回读校验 |
| `rotate_master_key.py`（重写） | **只重包 DEK**（旧 KEK 解包 → 新 KEK 重包 → 更新 `dek_wrapped/kek_id`），dry-run/apply；切换窗口先把新 KEK 写进 `keys/<kek_id>.key` 兜底，成功后再写 `master.key` 并清理 |
| `key_escrow.py`（新） | KEK 离线托管：`export` / `import`（明文只走离线介质/secret manager） |
| `package.py` | 新增别名 `lugwit_auth_migrate_envelope`、`lugwit_auth_key_escrow` |

> 旧的 `rotate_master_key.py` 还在操作早已废弃的 `l_notepad_accounts` 表（工具腐化），
> 本次一并重写为 DEK 重包；`auth_backup.py` 用 `SELECT *` + bytea→base64，新列自动纳入，无需改。

#### 上线步骤（本机 / 生产都要一致）

```bat
rem 0) 迁移前：确认本机能解开现有数据（当前主密钥即"数据密钥"）
wuwor lugwit_auth -- lugwit_auth_migrate_envelope            rem dry-run：应报"待迁移 N 行"
wuwor lugwit_auth -- lugwit_auth_migrate_envelope --apply    rem 单事务落库

rem 1) 部署新的 credential_service/secret_store/schema_upgrade（服务会热重载）
rem 2) 多实例共享同一把 KEK：env LUGWIT_AUTH_MASTER_KEY，或
wuwor lugwit_auth -- lugwit_auth_key_escrow export D:\keys\lugwit_kek.txt   rem 本机导出
wuwor lugwit_auth -- lugwit_auth_key_escrow import D:\keys\lugwit_kek.txt   rem 目标机导入

rem 3) 以后轮转 KEK（秒级，只重包 DEK）：
wuwor lugwit_auth -- lugwit_auth_rotate_master_key --apply
```

#### 实测（2026-09-22）

- dry-run：`credentials 共 12 行，待迁移 12 行；KEK=7ffa563119e0`；
- `--apply`：`已迁移 12 行（事务已提交）`；
- 生产 `GET /api/v1/accounts` → **200，12 条**；`custom_fields`（`api_key`/`api_id`/`base_url`）解密正常；
- 本机 auth 同库同 KEK → **200，12 条**；
- 自测：信封往返、掩码、密钥环、轮转重包、旧数据解析、未迁移行 fail-closed —— 全绿。

#### 安全边界与遗留

- **客户端永远不需要 KEK**：客户端只带**用户 token** 调 `/accounts`，服务端解密后返回明文；
  客户端本地缓存用 DPAPI（绑定当前用户+机器）。已核对客户端 bundle **不含** `master.key`/`LUGWIT_AUTH_MASTER_KEY`。
- **KEK 份数**：`LUGWIT_AUTH_MASTER_KEY` 明文注入到 N 台机 = N 份。多实例的**目标形态**是
  把 KEK 收进 **Vault / 云 KMS**（服务端用实例身份取用，KEK 不落盘、不进镜像/仓库），
  信封化后需要共享的只剩这一把 KEK。
- **dev / prod 必须隔离库**：给开发实例设 `LUGWIT_AUTH_DB_URL` 指向本机 PG，杜绝再互相污染。
- **KEK 离线托管**：`lugwit_auth_key_escrow export` 存离线；现在只有本机 + 生产两份文件，
  换机即丢是单点风险。
- 本次修复中 KEK 曾**经 `8764` 明文 HTTP** 传到生产，建议尽快 `rotate_master_key --apply` 让旧值作废。
- ⚠️ 本次改动是**无兼容**的：新代码对未迁移行直接拒绝。**代码上线必须与 `--apply` 迁移同步**，
  否则服务会 fail-closed（这是刻意的）。

### P4.6 — 密钥集中治理：lugwit_auth 单一持钥（2026-09-22）✅ 已实现并上线生产

**目标**：把"密钥管理 + 加解密"全部收进 `lugwit_auth`，**其他包/库不持钥、不做加密**。
只要密钥只存在于 auth 进程，"共库失配""每个库各自管密钥"这类问题就从根上消失。

**四层能力**：

1. **严格模式（消除事故机制）**：`secret_store.require_master_key()`，env
   `LUGWIT_AUTH_REQUIRE_MASTER_KEY`，**默认开启**——主密钥缺失时**拒绝自动生成**
   （旧行为会静默生成一把新密钥 → 多实例失配，正是 2026-09-22 的机制）。
   仅首次全新部署可临时 `=0` 放行一次。
2. **KEK 指纹自检**：`GET /api/v1/health` 增加 `kek` 字段
   （`{available, kek_id, source, keyring_size}`，**不含密钥**）。多实例比对该 `kek_id`
   即可发现失配。实测本机与生产均为 `7ffa563119e0`。
3. **中心密钥存储**：`GET/PUT/DELETE /api/v1/secrets/{namespace}[/{key}]`。
   `namespace` = 包的命名空间（映射 `credentials.scope`），秘密存在 auth 的库里，
   其他包**零密钥、零加密逻辑**。列表不回值，单读才回明文。
4. **crypto-as-a-service**：`POST /api/v1/crypto/wrap`（生成 DEK + KEK 包好，回一次性明文 DEK）
   / `POST /api/v1/crypto/unwrap`（用 KEK 解回 DEK）。给"密文必须留在自己库里"的包：
   本地用 DEK 加密、只存 `dek_wrapped`，**KEK 始终不出 auth**。

**新增/改动**：
| 文件 | 内容 |
|------|------|
| `secret_store.py` | `require_master_key()`（严格模式）、`kek_status()`（自检）；`load_master_key` 在严格模式下拒绝自动生成 |
| `envelope.py`（新） | 全项目唯一的信封原语：`kek_fingerprint()` / `new_dek()` / `wrap_dek()` / `unwrap_dek()` / `new_wrapped_dek()` |
| `credential_service.py` | 改用 `envelope.py` 的包/解原语（去掉重复实现） |
| `auth_server.py` | `/health` 加 `kek`；新增 `/api/v1/secrets/*`、`/api/v1/crypto/wrap|unwrap` |

**硬规则（防复发）**：除 `lugwit_auth` 外，全仓**禁止** `import lugwit_auth.secret_store`、
禁止自造 Fernet/主密钥。共享秘密一律走 auth 的 HTTP（`/secrets` 或 `/crypto`）。
唯一例外：**纯本机、per-user 的缓存**（如 `l_notepad_client` 的本地 DPAPI 文件）——那是另一信任域。

**实测**：本机/生产 `/health` 均返回 `kek_id=7ffa563119e0`；`/crypto/wrap → unwrap` 往返一致；
`/secrets/{ns}` 可读；严格模式单测（无 key 拒绝生成 / 显式放行才生成 / 有 key 正常读）全绿。

#### 增量（2026-09-22 续）：可插拔 KEK 来源 + 协调轮转 + 托管 + 告警

1. **KEK 可插拔来源（Vault/KMS 对接点）**：`load_master_key` 来源优先级扩展为
   `LUGWIT_AUTH_MASTER_KEY`（env）> `LUGWIT_AUTH_MASTER_KEY_FILE`（Vault Agent / K8s secret 挂载）
   > `LUGWIT_AUTH_MASTER_KEY_CMD`（执行命令取 stdout，如 `vault kv get -field=key ...`）
   > 本机 `master.key`。**密钥仍不进 PG/Depot/镜像**。
2. **协调轮转（多实例零停机）**：`lugwit_auth_rotate_master_key --new-key-file <f>` 支持用外部指定的
   新 KEK；流程为「先把新 KEK 放进各实例密钥环（`keys/<kid>.key`）→ 单事务重包 DEK → 各实例切主密钥」，
   窗口期内新旧都能解，**无停机**。
   **已执行**：`7ffa563119e0 → f01d4dae9395`（12 行 DEK 重包；本机与生产均已切到新 KEK；
   旧的明文传输密钥已退役）。
3. **离线托管**：`lugwit_auth_key_escrow export` 已导出新 KEK（44 字节原始 Fernet key）供离线保管。
4. **告警基础**：`GET /api/v1/health` 增加 `decrypt_failures`（进程内累计解密失败，KEK 失配/密文损坏会累加），
   配合 `kek.kek_id` 供外部巡检/告警。
5. **收敛"自造加密"**：全仓排查确认共享密钥加密**只在 `lugwit_auth`**；
   删除了 `l_qframelesswindow/_auth_client/secret_store.py` 里**死代码** `load_master_key/store_master_key`
   （客户端永不持 KEK）。`l_tray/depot_bridge.py` 的 DPAPI 属**纯本机 per-user** 例外（另一信任域）。

**遗留**：① KEK 目前仍是「本机文件 + 可选 env/FILE/CMD」，尚未实际接入 Vault/KMS（对接点已就绪）；
② 告警只到"可巡检"层面，主动告警（指纹不一致/`decrypt_failures>0` 推送）需外部系统；
③ 离线 escrow 文件需人工转存到离线介质后删除本机副本。

### P5 — 统一登录窗口（2~3 天）✅ 已实现（服务端 PKCE + SDK + 三处 UI 收敛）

**已完成：服务端授权码（PKCE）**

- 新增 `oauth_service.py` + 两张表（`schema_upgrade.py`）：`clients`（`client_id`/`redirect_uris`/`is_public`）、
  `auth_codes`（**只存 SHA-256 哈希**、60 秒过期、原子 `UPDATE … WHERE consumed_at IS NULL` 核销）。
  内置客户端种子（幂等）：`desktop`（回环 `http://127.0.0.1`/`[::1]`/**任意端口** + `lugwit://auth/callback`）、
  `web`（浏览器 SSO）。
- 新增 `GET /api/v1/auth/authorize`：校验 client + `redirect_uri`（回环允许任意端口）→ 未登录 302 到
  `/login?next=<本 URL>` → 已登录发一次性 code 并 302 回 `redirect_uri?code&state`。
- `POST /api/v1/auth/token` 扩展 `grant_type=authorization_code`（+ `code_verifier` PKCE S256 校验），
  password 授权（Swagger/CLI）保持原样。
- `templates/login.html`：`next` 改用 `| tojson` —— 否则带 query 的 authorize URL 会被 HTML 转义成
  `&amp;`，登录后跳错地址（PKCE 流程的隐性坑）。

**已完成：SDK（`lugwit_auth.client`）**

`make_pkce()`、`LocalCallbackServer`（127.0.0.1 回环、state 校验、超时可配、`stop()` 后仍能取
`redirect_uri`）、`login_via_browser()`（开浏览器 → 接授权码 → 换 token → **只把 refresh 落盘**）、
`login_with_password()`（无浏览器时的降级路径，密码只在内存）、`ensure_token()`（剩余寿命 <60s 自动轮转
refresh，无感续期）、`logout()`（撤销 refresh + 清本地）、`access_token()`。
新增 `token_store.py`：refresh 存 `<LUGWIT_AUTH_KEY_DIR>/desktop_tokens.json`，Windows 下 **DPAPI 包裹**；
**access 只在内存**，密码从不落盘。

**已完成：「密码不落盘」**

`l_qframelesswindow/login_store.py`：`save_remembered()` 只写**用户名**（旧签名带 password 直接报 TypeError，
防止有人再用老写法）；`load_remembered()` 会把历史文件里的 `password` 字段**从磁盘上抹掉**再返回；
`login_dialog.py` 不再回填/保存密码。

**验收实测**
- P5 全链路 **24/24**：PKCE 与 redirect_uri 校验（未知 client/未注册 redirect/缺 code_challenge → 400）、
  未登录 302 到登录页并带 next、`login_via_browser` 走通（脚本化"浏览器"替代 `webbrowser.open`）、
  access 是 RS256 且可离线验签、**授权码复用 → 400**、**错误 code_verifier → 400**、
  **落盘是 DPAPI 且不含密码/access**、`ensure_token` 自动续期（refresh 已轮转）、
  **登出后服务端 refresh 撤销（revoked=1）+ 本地清空 + 旧 refresh 401**。
- `LoginStore` **5/5**：新写入只有 username/auto_login、历史明文被抹掉、旧签名被拒。
- 7 个包编译全绿。

**已完成（UI 收敛，三处全做完）**

1. **标题栏登录（`l_qframelesswindow`）**：`LoginDialog` 重写为「浏览器登录（主）/ 账号密码（降级，默认收起）」
   两个动作 + 已登录态显示 + 登出；网络与浏览器等待都丢到工作线程，UI 不冻结。
   `login_store.py` 新增 `sdk_client()`（**软依赖**懒导入 —— 不给共享 UI 库塞 fastapi/sqlalchemy 栈）、
   `restore_session()`（用 DPAPI 里的 refresh 自动续期，替代旧的 token 文件 + HTTP `/verify`）、
   `logout_session()`（撤销 refresh + 清本地）。`L_FramelessMainWindow.restoreSavedLogin()/logout()`
   一并改走这两个入口。
2. **托盘（`l_tray`）**：`Tray.login()` 原来是 `pass` + 不可达的旧启动代码（启动 `lugwit_login/loginUI.py`
   那套自带 MySQL/FastAPI 的实验栈）→ 改为调 SDK 的 `login_via_browser()`，成功后
   `depot_bridge.set_session_token()`；`depot_bridge.token()` 优先级变为「托盘会话 token > env > 配置账号登录」。
   `lugwit_login/loginUI.py` 摘掉 `save_credentials()/load_credentials()` 与「保存密码/自动登录」复选框
   （该目录是个人实验沙盒，含 `day1.ipynb`/`registered_users.json`，**未删**）。
3. **`l_WChat` 网页登录**：新增浏览器 SSO 两条路由 `/api/lugwit/sso/start`（带 PKCE + state 跳 authorize）
   与 `/api/lugwit/sso/callback`（校验 state → 授权码换 token → 写本域 `lugwit_token` cookie，HttpOnly）；
   登录页加「🌐 用统一认证登录（推荐）」，账号密码输入收进 `<details>` 作为降级。auth 侧只加
   `make_pkce/sso_authorize_url/exchange_code` 三个纯 `requests` 函数，**不引入 lugwit_auth 包依赖**。

**验收实测（UI 收敛）**
- `login_store` SDK 辅助 **6/6**：`_auth_root` 去后缀、无登录态返回 None、`login_with_password` 成功、
  **未落盘任何密码文件**、`restore_session` 拿回 access+用户名、登出后 restore 返回 None。
- `depot_bridge` **6/6**：无登录态为空串、会话 token 生效、env 不覆盖会话 token、清掉后回落 env、
  配置账号可登录、源码里不再有 `/auth/auto` 调用。
- `l_WChat` SSO **9/9**：start 302 到 authorize（S256）、state/verifier HttpOnly cookie、
  state 不匹配 400、真授权码、callback 302 + HttpOnly 登录 cookie（RS256）、清临时 cookie、
  该 cookie 能过 `/api/lugwit/me` 闸门。
- 7 个包编译全绿。

**剩一件可选收尾**：access TTL 仍 30 天 —— SDK 的 `ensure_token()` 与三处 UI 都已就位，
可按 §7 收短到 15 分钟（消费方会自动续期）。

### P6 — 业务接入与 owner 强校验（按包推进）🟡 读侧+写侧+l_agent_chat 分区已完成；两个次级调用方待清

**已完成：netdisk depot 读接口的 owner 强校验**

`web_server.py` 的 `download` / `list` / `history` 在**取数据之前**调 `gate.require_perm()`：
管理员/系统角色本地放行 → 「路径首段 = owner」本机快判 → 跨用户/共享场景问 auth `/authz/check`
→ auth 不可达时只认自己的路径。判定失败回 **403**（不是 404/200 混淆）。
新增辅助 `_depot_perm(request, user, dpath, perm)` + `_auth_token(request)`（跨用户判定要把
原始 token 带给 auth）。`list` 对库根（如 `/l_agent_chat`）只做登录校验，库根下按 `<user>/` 隔离。

**已完成：`l_agent_chat` 的 depot 路径按用户分区 + 真实身份**

`depot_sync.depot_path()` 由平铺 `{library}/{rel}` 改为 **`{library}/{user}/{rel}`**
（设计 §12 D6），`user` 取自 `LUGWIT_USER`（同时是 `_login()` 用的账号）：
- 没配身份 → **直接报错**，不再悄悄写平铺路径（那等于所有人互相覆盖同一份文件）；
- 身份含路径分隔符 → 拒绝；
- `config.LUGWIT_USER` 的注释改为"必填"（原因：`/auth/auto` 回环兜底已按 P0 默认关）。

**验收实测**
- netdisk 读接口 **11/11**：A 读 B 的 `download` → **403**、读自己的路径 → 过权限层（404）、
  管理员 → 过权限层、无凭据 → 401；`history`/`list` 同样 403；B 用 `/authz/grant` 授读权后
  A 立即可通过、`/authz/revoke` 后立刻恢复 403（ACL 生效，无需重启）。
- `l_agent_chat` 路径 **5/5**：`settings.json` → `/l_agent_chat/u01/settings.json`、
  `sessions/session_1.json` 分区正确、反斜杠/前导斜杠归一、无身份报错、身份含分隔符报错。

**未做（写侧）**：原计划留作后续，**本轮已补齐**（见下）。

**已完成（本轮续）：写侧 + 消费方收尾**

写侧在**取数据/落库之前**调 `_depot_write_perm(request, user, paths)`（同一 `depot.write` 判定）：
`submit`、`submit_stream`、`delete`、`move`（源与目标都判）、`revert`、`checkout`、`edit_text`、
`import`（非 dry-run 才判）、`mark_delete`、`mark_move`、`mark_add_stream`、`revert_pending`；
`submit_pending` 落库前把**待提交列表里的每个路径**都判一遍，且**取不到列表就拒绝**（fail-closed，
不因判权取数失败而放行）。`cl_description`/`lock`/`unlock` 仍只做登录校验（仅改描述/建议锁）。

消费方不再依赖 `/api/v1/auth/auto`（该端点 P0 起默认关）：
- `l_tray/depot_bridge.py`：删 `_auto_token()`；登录态 = `LUGWIT_ACCESS_TOKEN` >（`LUGWIT_USER`/`LUGWIT_PASSWORD` 登录）；
  网页传来的真实用户 token 仍走 `token_override` 优先；取不到时打印一次明确告警（不再静默发匿名请求）。
- `l_notepad_server/depot_map.py`：同样删 `_auto_token()`，新增 `require_token()` —— 没登录态**直接抛
  DepotError 并说明怎么配**，不再发匿名请求等 401。
- `lugwit_baidu_netdisk/web_server.py`：删掉 `_auto_local_token()` 与 `_current_user`/`_page_user` 里的
  回环兜底分支（未登录就是 401 / 跳登录页）。

**验收实测（新增）**
- 写侧 **11/11**：A 往 B 的路径 `submit_stream`/`delete`/`move`/`revert`/`edit_text`/`mark_add_stream`
  → **403**；A 写自己的路径 → 过权限层（无工作区 → 400）；B `grant write` 后 A 立即可写、
  `revoke` 后立刻恢复 403。
- 读侧回归 **11/11** 仍全绿（删兜底后读判定未受影响）。

**待清（同一模式，不在本轮两个消费方内）**：已清（见下）。

**已完成（本轮续 2）：最后三处 `/auth/auto` 调用方改成 env/登录**

| 位置 | 改法 |
|---|---|
| `ChatRoom/backend/app/core/auth/facade/auth_facade.py` | 删 `_auth_service_auto()` → 新增 `_auth_env_login()`：`LUGWIT_ACCESS_TOKEN` env >（`LUGWIT_USER`/`LUGWIT_PASSWORD` 登录）；`auto_login()` 未配置时返回 **503 + 明确指引**（不再 500）；env-token 场景经 `/auth/me` 反查用户名 |
| `l_notepad_client/account_favorites_widget.py` | `_ensure_api_token()` **不再自取 token**：登录态只认 `api.token`（登录后已就位）或 env `LUGWIT_ACCESS_TOKEN`；未登录返回 False（回退本地数据）并打印一次提示；顺手删掉因此不再使用的 `server_config` 导入 |
| `lugwit_baidu_netdisk/tests/bench_depot.py` | `get_token()` 改为 env token > 账号登录；两者都没有时 `SystemExit` 并说明怎么配 |

**验收实测（新增）**：三处各 3–4 项断言全绿
（env token 生效 / 账号登录拿到 RS256 token / 无配置时明确失败而非静默匿名请求）。
ChatRoom 侧用 AST 抽出 `_auth_env_login` 原样执行验证（ChatRoom 的 rez 环境本身缺 `l_notepad`
包族、`ChatRoom -- python` 起不来，与本改动无关）。

### P7 — 备份与灾备（1~2 天，依赖 Depot 侧前置）✅ 已完成（含 Depot 侧前置脚本）

**已完成：auth 自身数据的逻辑备份（`auth_backup.py`）**

- 按表 dump（设计 §13.7 的备份对象）：`users/user_roles/roles/role_permissions/permissions/
  sessions/clients/auth_codes/credentials/favorites/user_profiles/acl_grants`
  （缺表自动跳过并记在结果里；`bytes` 列如 `secret_enc` 以 base64 包装、时间以 ISO 存）。
- 单文件 `authdump.json.gz` = `{schema_version, exported_at, stamp, tables, checksum}`；
  `checksum = sha256(canonical(tables))`，读取时**必须**校验 → 能识别被篡改/损坏的 dump（验收 ③）。
- 双份落盘：本地 `~/.lugwit/auth_backup/<stamp>/`（0600）+ Depot
  `/l_auth_backup/<stamp>/authdump.json.gz` 与 `/l_auth_backup/latest.json`（指针，含 stamp/checksum/size）。
- 触发：每日定时（`start_scheduler`）+ **关键变更后异步**（改密 / 删号 / 角色变更 → `trigger_async`，
  3 次重试、失败只告警，不阻塞认证主流程 —— §13.4 P-E）。
- **密钥不入云**：dump 文本扫 `BEGIN PRIVATE KEY`/`master.key`/`LUGWIT_AUTH_MASTER_KEY` 等特征，
  命中立刻告警（验收 ④）。
- CLI：`lugwit_auth_backup_now`（立刻备份 / `--no-upload` / `--verify <file>` / `--status`）。

**已完成：恢复 CLI（`restore_auth.py` / `lugwit_auth_restore`）——P-D「恢复走 CLI」**

- `--from <file>` 或 `--latest`；先校验 checksum（不通过退出码 3）→ 单事务内按复合主键 upsert
  （`users.id`/`user_profiles(username,key)`/`role_permissions(…)`/`clients.client_id`/`auth_codes.code_hash`…）
  → 对齐自增序列（`setval`，否则恢复后新插入会撞主键）→ 输出每表 `dump/upsert/before→after` 差异报告。
- `--dry-run` 演练后回滚。**不提供 HTTP 恢复端点**（那会变成"恢复要先登录"的环）。

**顺带补的两处（P7 才暴露出来）**

1. **`/authz/check` 支持服务身份**：auth 自己的备份任务是 `typ=service` 令牌，
   但 `require_user` 要求存在 `users` 行 → 401。改为：服务身份按 **token 里声明的 `roles`** 解析权限
   （`svc.authz.check` 来自 seed 的 `service` 角色），不要求有用户行；普通用户/管理员路径不变。
2. **ACL 支持目录继承**（`authz_service._resource_candidates`）：按"自身 → 各级父目录"查授权，
   所以**在库根发一条** `/l_auth_backup`（grantee=服务身份，read,write）就能覆盖
   `<库>/<stamp>/authdump.json.gz`。库根授权由 `auth_backup.ensure_grant()` 在服务启动时幂等写入。

**验收实测**
- dump/恢复 **13/13**：checksum 生成与校验、包含 users/credentials、**不含私钥/主密钥特征**、
  本地副本落盘、**篡改后 checksum 不匹配且恢复 CLI 退出码 3**、dry-run 回滚、
  真恢复后各表行数不变（upsert 幂等）、**恢复过程对齐自增序列**、`--latest` 取最新。
- ACL 继承 **6/6**：候选路径顺序、未授权拒绝、库根授权后子路径（含"另一天"）允许、
  其他主体仍拒、库外路径不受影响。
- **真备份上传成功**：`/l_auth_backup/20260920-1415/authdump.json.gz`（本地 3 份 5KB 副本；
  `acl_grants` 里可见库根授权行）。

**已完成（本轮续）：Depot 侧前置脚本（D9 / 演进计划 T4）**

- `tools/depot_manifest_verify.py`：递归读清单目录 → `(path,rev)` 唯一性/rev 递增检查 →
  与 `depot_file_rev`（action/md5/size）逐条比对 → 四类漂移（清单有库里没有 / 库里有清单没有 /
  字段不一致 / blob 登记缺失）+ `--json` 报告；退出码 0 一致 / 1 漂移 / 2 清单不可解析；纯只读。
- `tools/depot_manifest_replay.py`：按 cl 顺序生成重建 SQL（`depot_changelist` + `depot_file_rev`
  + `depot_blob`），默认 **dry-run**（可 `--sql-out` 落文件），`--apply` 单事务写库；
  `--blob-mode ensure|only-existing|none`；`--from-cl/--to-cl` 分段；
  **补 `setval` 对齐自增序列**（否则回放后新建 CL 撞主键）。
- 两者的分工写在工具 docstring 里：replay 重建"提交历史"，工作区/have/锁属运行态不在此恢复。

**验收实测（Depot 侧）**：**13/13 通过**（用真实库状态合成的全量清单 178 份 / 868 条版本）
- replay dry-run：SQL 覆盖全部清单条目、含 2 条 `setval`、BEGIN/COMMIT 包裹；
- verify：版本表与清单**完全一致**（`--no-blob` 时 0 漂移）；带 blob 检查时只报
  「blob 登记缺失 71 条」——这是**库里既有的真实漂移**（清单引用的内容没有登记行），
  不是工具误报；
- 篡改检出：清单多一条 → 「清单有、库里没有」；清单少一条 → 「库里有、清单没有」（均 rc=1）；
- `--apply --blob-mode none` 幂等：changelist 183 / file_rev 868 前后不变，再 verify 仍一致。

**灾备演练（①→⑥，2026-09-20 在临时库 `chatroom_dr_drill` 真跑一遍）**

| 步 | 做了什么 | 实测结果 |
|---|---|---|
| ① | 网盘 blob 物理内容（外部存储） | 抽样 8 条 `version_depot/blob/<库>/<md5[:2]>/<md5>` 全部 **200 存在** |
| ② | `.depot/manifest/0000/<cl>.json` 兜底清单 | 抽样 6 份全部存在（291–321 bytes）；另用真实库状态合成全量清单 178 份 / 868 条做回放输入 |
| ③ | `depot_manifest_replay.py --apply` 在**空库**重建 Depot 元数据 | 临时库从 0 → changelist 178 / file_rev 868（= 清单全量；源库多 9 个 CL 是清单快照之后的提交） |
| ④ | Depot 可服务（元数据侧不依赖 auth 在线） | 从重建库读出 `head(/rez_pkg/中文测试_abc.md).rev=2`、根目录列出 6 个库 |
| ⑤ | `lugwit_auth_restore --from <dump>` 重建 auth 数据 | 12 张表行数与 dump **逐表一致**（users 18 / credentials 12 / roles 4 / role_permissions 32 …），夹具账号 `password_hash` **逐字节一致** |
| ⑥ | 用恢复出来的账号登录 | 指向临时库的实例：登录 **200** + RS256 access + `/auth/me` 返回该用户与角色；dump 里没有的账号 **401**；生产实例不受影响 |

**演练暴露并修掉的真 bug**：`models.py` 的时间列没写 `DateTime(timezone=True)` → **全新库** `create_all`
会建成 `timestamp without time zone`，而 `session_service.create()` 传的是 aware datetime →
asyncpg 报 `can't subtract offset-naive and offset-aware datetimes` → **新部署第一次登录就 500**。
已修：模型改成 `DateTime(timezone=True)`；`schema_upgrade` 增加**条件转换**
（只在该列确实是"无时区"时才 `ALTER … TYPE timestamptz USING … AT TIME ZONE 'UTC'`，
避免误动已有的 timestamptz）。生产库三列复核均为 `timestamp with time zone` ✓。

演练环境已清理（临时库删除、夹具账号删除、生产库回到 users 17 / sessions 0 / acl_grants 1）。
演练脚本在 `D:\Temp\Log\drill_*.py`（临时目录，季度演练可照抄流程：建库 → 回放 → 恢复 → 登录）。

**另发现（Depot 侧既有问题，未改）**：
1. `depot_blob` **缺 71 条登记**（清单引用到的 (库, md5) 没有对应行）→ 读这些历史版本会落到兜底查找；
2. `depot_blob.remote_path` 存的是 `/apps/Lugwit/version_depot/version_depot/blob/…`（**多一层 `version_depot`**），
   而网盘上真实路径是 `/apps/Lugwit/version_depot/blob/…` —— 实测 DB 里那条路径 404、真值 200。
   这两条都会影响历史版本下载，属 Depot 数据/代码侧修复范围（本设计只做标注）。

**已有的定时/触发与旧物理备份**：`db_backup.py`（整库 `pg_dump`）仍在跑，与本节的**逻辑 dump** 互补：
物理备份恢复快但要 postgres 工具，逻辑 dump 可读、可校验、可跨版本搬。

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
| 收藏/凭据表 | `credential_service.py`（P4 泛化后；旧 `account_service.py` 已删） |
| 服务监督端点 | `auth_server.py:317,489-641` |
| 消费方闸门 | `R/lugwit_baidu_netdisk/999.0/src/lugwit_baidu_netdisk/gate.py:22-37` |
| login 只验签名 | `.../gate.py:31-37` |
| 其他消费点 | `lugwit_baidu_netdisk/.../web_server.py:386,406`；`l_WChat/.../api/auth.py:37-70`；`l_notepad_server/.../auth.py:26-98`；`l_qframelesswindow/.../login_store.py:46-60`；`l_notepad_client/.../api_client.py:104-107` |
| 桌面登录窗口（明文密码） | `l_qframelesswindow/.../login_dialog.py`、`login_store.py:135-152`；`l_tray/.../lugwit_login/loginUI.py` |
| depot 读接口不校验 owner | ✅ **已修（P6）**：`web_server.py` 的 `download/list/history` 取数据前调 `gate.require_perm()`，越权返回 403 |
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
| `depot_manifest_replay.py`、`depot_manifest_verify.py` | ✅ **已补齐**（P7 前置，T4）：verify 纯只读四类漂移 + 报告；replay dry-run 出 SQL / `--apply` 单事务 + setval。回环实测 13/13 | D9 |
| `ssl_support` 启动即打印"生效 CA + 是否降级" | 现在只在失败时告警一次，易被日志淹没 | §4.5「现在就该做的一件事」（约 10 行改动） |
| 主密钥 DPAPI 化 + 去 `l_notepad` 路径 | ✅ **已完成并轮转**（新路径 DPAPI 包裹；旧密钥已 `.bak`；无回退分支） | §9 P4 增量 / D4 |
| **信封加密（KEK 包 DEK）+ 密钥环 + 迁移/轮转/托管工具** | ✅ **已实现并上线生产**（2026-09-22；`credentials` 12 行已迁移；生产 `/accounts` 200） | §9 **P4.5** |
| 生产主密钥与开发库耦合（dev auth 连生产 PG、各用不同 KEK → 503） | ✅ **已修复**：生产 KEK 对齐 + 升级信封；**根治仍需 dev 独立库**（`LUGWIT_AUTH_DB_URL`） | §9 **P4.5** / D4 |
| KEK 集中托管（Vault/KMS）+ 离线 escrow + 轮转 | ⚠️ **待做**：现为 `master.key` 文件 + env 注入（多实例=多份）；`key_escrow` 已就绪 | §9 **P4.5** / D4 |
| **密钥集中治理（严格模式 + 指纹自检 + `/secrets` + `/crypto`）** | ✅ **已实现并上线生产**（2026-09-22；本机/生产 `/health` 指纹一致 `7ffa563119e0`） | §9 **P4.6** |
| 其他包"自造加密/自持密钥"逐个收敛到 auth | ⚠️ **待做**：约定"除 auth 外禁 Fernet/主密钥"，需全仓排查 | §9 **P4.6** |
| "解密失败/KEK 指纹不一致"告警 | ❌ **未做**：现仅有 503 与 `/health` 指纹可供外部巡检 | §9 **P4.6** |
| l_notepad 的**死文件** `account_store.py`（client/server 各一份） | ✅ **已删除**（连同 auth 的 `account_service.py`）→ 「`accounts_key` 无引用」已签收 | §9 P4 |
| `custom_fields` 里 3 个历史遗留密文（异机/旧密钥写入）现在解不开 | ✅ **已清理**（活表 `credentials`；复核 0 不可解密）；旧表快照里同类残值保留原样 | 数据清理项 |
| `/auth/auto` 收紧（不信任 XFF + 降权 + 开关） | ✅ **已落地**（P0） | §9 P0 |
| 本机消费方改用 `LUGWIT_AUTO_AUTH_ENABLED=1` 或 `LUGWIT_ACCESS_TOKEN` | **部分已改**：`l_agent_chat` 改为「必须配 `LUGWIT_USER`/`LUGWIT_PASSWORD` 走登录 + 路径按 `<user>/` 分区」（P6）；`lugwit_baidu_netdisk/web_server.py:_auto_local_token`、`l_tray/depot_bridge.py`、`l_notepad_server/depot_map.py` 仍按"回环即可拿 token"假设 | P6 收尾 |
| netdisk depot **写侧**（submit/delete/move/revert/…）接 `depot.write` | ✅ **已接**（本轮；`submit_pending` 落库前逐路径判、取不到待提交列表即拒绝） | §9 P6 |
| 仍调 `/auth/auto` 的次级调用方 | ✅ **已全清**：`ChatRoom/.../auth_facade.py`（改 `_auth_env_login`）、`l_notepad_client/account_favorites_widget.py`（不再自取 token）、`lugwit_baidu_netdisk/tests/bench_depot.py`（env/登录）；全仓 `auth/auto` 仅剩端点定义与注释 | §9 P6 |
| access TTL 收短到 15 分钟 | **有意保留 30 天**（P5 SDK 未落地，收短会打断所有现存客户端） | P5 落地后随 SDK 一起收短（§7） |
| `gate.py` 的 RS256 公钥缓存 | ✅ 通过 `lugwit_auth.client.verify_token`（读本机 jwks.json，缺失才联网） | §9 P2 |
| legacy 消费方本地 HS 验签 | `ChatRoom/backend/app/main.py:192-194` 自建 `JwtService`（验的是自己的 `chatroom_token`，不受影响）；其余消费方走 HTTP `/auth/verify` 或 `client.verify_token` | 已确认无 RS256 破口 |
