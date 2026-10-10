<!-- lugwit-note
updated: 2026-10-08
updated_by: admin01
note: 把「已落地」改成经 grep+实测核对的实际情况：记录新符号名 _renew_once/_renew_access_once/refresh_token_of 与 2026-10-08 的实测证据。
-->
<!-- lugwit-note-history
2026-10-08 admin01 把「已落地」改成经 grep+实测核对的实际情况：记录新符号名 _renew_once/_renew_access_once/refresh_token_of 与 2026-10-08 的实测证据。
2026-10-08 admin01 §4.5 补记 2026-10-08 复核结论：原写法的修复此前只存在于本文档、源码里并没有；本次已在 l_WChat 源码落地并实测链式续期，另记 refresh 一次性轮换必须写回新 refresh。
2026-10-08 admin01 纠正 §4.5「access 只有 15min」的错误口径（实测 30 天），并补记 refresh 复用即撤销全会话的地雷与 app.py::_renew_once 去重修复
2026-10-08 admin01 记录「l_WChat 每次启动都要重登」的根因与修法（access 15min 只写 access cookie → 加 refresh cookie + 页面闸门无感续期）
-->
# 宝妈笔记 App 架构与发布（l_WChat）

> **历史/需核实警示（截至 2026-09-17）**：本文包含早期 `http`、`cleartext` 与 uvicorn `--reload` 描述，不能当作当前部署基线。现行 HTTPS 见[Rez_pkg/HTTPS证书与域名申请总结.md](Rez_pkg/HTTPS证书与域名申请总结.md)，网络路由见[Nginx反向代理机制.md](Nginx反向代理机制.md)，源码热重载见[src_hot_reload_源码热重载与主页常驻.md](src_hot_reload_源码热重载与主页常驻.md)。Python 改动须按服务当前机制手动重启核验。


> 项目根：`rez-package-source/l_WChat/999.0/src/l_WChat`
> 后端：FastAPI（`app.py`，端口 1234，uvicorn `--reload`）
> App：`wchat-android/`（Capacitor + Android WebView）
> 更新：2026-09-02

## 1. 架构总览

```
手机 App（Capacitor WebView）
   │  capacitor.config.json → server.url
   ▼
https://121.196.144.88        ← 云服务器（443 入口，App 实际加载的页面/接口）
   ▲
http://localhost:1234        ← 开发机（改代码先在这里验证）
```

**关键点：`capacitor.config.json` 的 `server.url` 指向云服务器，App 加载的 ALL 页面来自云。**
开发机上改完代码，手机 App 看不到任何变化，必须同步到云服务器。本地验证用浏览器开 `http://localhost:1234/...`。

- 壳里 `cleartext` 只对 `http://` 放行，https 强制；`network_security_config` 默认禁明文
- `webDir: "www"`（本地产物壳，实际页面全部由 server.url 提供）

## 2. 缓存坑（踩过最深的一个）

### 现象
云服务器代码已更新、浏览器访问正常，但手机 App 仍跑旧脚本（如表单字段丢失、新按钮不出现）。

### 根因
旧版后端返回的 HTML 无 `Cache-Control` 头，Android WebView 启发式缓存会**长期缓存**这些响应，且缓存能杀进程不死。App 读的是磁盘里的旧 HTML/旧 JS。

### 修复（两层，缺一不可）
1. **服务端**：`app.py` 中间件对所有 `text/html` 加 `Cache-Control: no-store, must-revalidate`（改 `app.py` 需重启服务生效；Jinja2 模板 `templates/*.html` 改动即时生效）。
2. **App 端**：`MainActivity.onCreate` 里 `WebSettings.LOAD_NO_CACHE`（versionCode ≥ 5 的包才有）。

### 应急
老包（versionCode < 5）清缓存：设置 → 应用 → 宝妈笔记 → 存储 → 清除缓存 → 重启 App。

## 3. 应用内更新（「更新软件」按钮）

### 旧实现（已废弃）
系统 `DownloadManager` 下载 APK —— **Android 9+ 的 DownloadManager 是独立进程，无视应用的 `usesCleartextTraffic`，直接拒绝 http 明文下载且静默失败**，所以永远弹不出安装界面。

### 现实现
`AppUpdaterPlugin.java`：
- 应用内 `HttpURLConnection` 下载（支持 http 明文，15s/30s 超时）→ `getExternalFilesDir(Downloads)/baobaonote_update.apk`
- `FileProvider` + `ACTION_VIEW` 触发系统安装
- 权限不足发 `installError` 事件（`needPermission` / `downloadFailed` / 其他消息），`templates/about.html` 监听展示
- `downloading` AtomicBoolean 防重复点击

### versionCode 规则
- **覆盖安装必须 versionCode 递增**，否则装不上。
- `build-apk.bat` 第 2.5 步每次打包自动 +1（PowerShell `UTF8Encoding($false)` 写回，**禁用带 BOM 的 Set-Content —— BOM 会让 Gradle 报 `Unexpected character: '锘?'`**）。
- 注意：`templates/about.html` 里的「版本 1.1 (N)」是写死的静态文字，N 会落后于真实 versionCode。

## 4. APK 打包与分发

`wchat-android/build-apk.bat` 一键流程：

```
JDK21：D:\DevTools\jdk21\jdk-21.0.12.1+1
SDK  ：D:\DevTools\android-sdk
[1/4] npm install（node_modules 已存在则跳过）
[2/4] npx cap sync android
[2.5] versionCode 自动 +1（无 BOM 写回 build.gradle）
[3/4] gradlew clean assembleDebug（日志 D:\build_log.txt）
[4/4] 覆盖到 ..\static\apk\baoma.apk   ← App「更新软件」的下载源
```

安装新包的两种方式：
- USB：手机连电脑跑 `install-run.bat`
- 浏览器下载 `http://<开发机IP>:1234/static/apk/baoma.apk`

**发版检查单**：
1. 本机 `build-apk.bat` 打包成功
2. `static/apk/baoma.apk` 同步到**云服务器**同名目录（否则 App 更新下载到的还是旧包）
3. 源码同步到云服务器并重启服务（模板即时生效，`app.py` 等需重启）
4. 手机清一次 App 缓存（老包过渡期）

## 4.5 登录态持久化（2026-10-05 修「每次启动都要重登」）

> ⚠ **2026-10-08 复核订正**：本节此前描述的那套修复（`REFRESH_COOKIE_NAME` /
> `login_full` / `refresh_session` / `set_session_cookies` / 闸门无感续期）**在
> `rez-package-source/l_WChat/999.0/src/l_WChat` 源码里根本不存在** —— 全仓 grep
> 这些符号 0 命中，登录与 SSO 回调各自裸写一枚 `lugwit_token`、refresh 当场丢弃。
> 也就是说：症状「每次启动都要重登」的**成因仍在**，本节当时是「写了方案没落地」。
> **2026-10-08 二次核对（grep + 实测）**：那套符号在源码里当时确实仍 **0 命中**，本次是真的落地了，
> 但**实际符号名与本节原文不同**，别再按旧名字 grep（旧名记错会得出「没落地」的错误结论）：
> - `api/auth.py`：`REFRESH_COOKIE_NAME="lugwit_refresh"` / `REFRESH_COOKIE_MAX_AGE` / `login_full()`（返回 access+refresh 二元组）/
>   `refresh_session()`（`POST /api/v1/auth/refresh`）/ `set_session_cookies(resp, data)` / `refresh_token_of(request)`；`login()` 保留旧签名供既有调用方。
> - `api/routes.py`：SSO 回调与 `POST /api/lugwit/login` 都改用 `set_session_cookies()` 成对写；`/api/lugwit/logout` 连 `lugwit_refresh` 一起删。
> - `app.py::force_login_pages`：access 校验失败时先 `_renew_access_once()`→`_renew_once()`（同一枚 refresh 60s 内只提交一次，双检 + `threading.Lock`），
>   成功则照常出页并**把新的一对 cookie 写回**，失败才 302 `/login`。
>
> **2026-10-08 落地实测（本机 1234 + auth 1027，账号 admin01，curl/urllib 实跑）**：
> ① `POST /api/lugwit/login` → `200`，响应 `Set-Cookie` **两枚**（`lugwit_token` + `lugwit_refresh`）；
> ② 对照（证明没把登录拿掉）：坏 access 且无 refresh → `302 /login?next=%2F`；
> ③ 模拟「重开 App」（坏 access + 真 refresh）→ `307 → /postpartum` 静默续期放行，写回新的一对，新 access 取 `/api/v1/auth/me` `200`；
> ④ 地雷复核：**同一枚 refresh 并发 6 个**页面请求 → 全 `307`，同枚再单发一次仍 `307`（会话未被撤销）——去重生效；
> ⑤ `POST /api/v1/auth/refresh` 对无效票据返回 `401 {"detail":"refresh token 无效、已过期或已被使用，请重新登录"}`（轮换+复用检测确实存在，故去重不可省）。

**症状**：每次打开 App / 浏览器进站都被弹回 `/login` 重新登录一遍。

**根因**：`api/auth.py` 里 `COOKIE_MAX_AGE = 30*86400` 只决定 **cookie 存活期**，
cookie 里装的 `lugwit_token` 是 **access JWT**（⚠️ 2026-10-08 实测订正：access 自己就有 **30 天**，解 `exp-iat = 2592000s`、登录响应 `expires_in=2592000`；原文写的「只有 15min」是过期口径，别再用它解释「每次都要重登」）（《lugwit_auth统一用户授权服务设计》§4.2
接入模式 A：access 15min + refresh 30d）。登录 / SSO 回调都只写 access 一枚 cookie，
refresh 直接被丢掉 → 15min 之后 `app.py::force_login_pages` 的 `verify_with_auth()` 必然失败，
每开一次 App 就重登。**「cookie 有 30 天」是假象，真正到期的是里面的票据。**

**修法**（不动登录本身）：
1. `api/auth.py`：新增 `REFRESH_COOKIE_NAME="lugwit_refresh"`（HttpOnly, path=/, 30d）、
   `login_full()`（拿完整 access+refresh）、`refresh_session()`（`POST /api/v1/auth/refresh
   {"refresh_token":…}`）、`renew_session(request)`、`set_session_cookies(resp, data)`。
2. `api/routes.py`：SSO 回调与 `POST /api/lugwit/login` 改用 `set_session_cookies()` 成对写 cookie；
   `/api/lugwit/logout` 连 `lugwit_refresh` 一起删（否则会被无感续期「登出失败」）。
3. `app.py::force_login_pages`：access 校验失败时先 `renew_session()` 用 refresh 换新令牌，
   成功则照常放行并在响应上重写 cookie（`set_session_cookies`），失败才 302 到 `/login`。

**验收（2026-10-05 实测，服务 127.0.0.1:1234 + 认证 1027，账号 admin01）**：
`GET /album` 带「失效 access、无 refresh」→ `302 /login?next=%2Falbum`（复现旧 bug）；
带「失效 access + 真 refresh」→ `200` + `Set-Cookie: lugwit_token=<新的 access>`（无感续期生效）。
认证契约另测：`login` 200 → `refresh` 200 → 新 access 取 `GET /api/v1/auth/me` 200。

### 4.6 续记（2026-10-08）：refresh 复用 = 撤销全部会话，闸门续期必须去重

**新发现的地雷**：`lugwit_auth` 的 refresh 是「轮转 + 复用检测」的 —— 同一枚 refresh 被提交第二次，
服务端判定「令牌泄露」，直接**撤销该用户全部会话**。实测（本机 1027，admin01）：
`login` → `refresh(R1)` 200（R1→R2）→ 再 `refresh(R1)` → `401 {"detail":"refresh token 无效、已过期或已被使用，请重新登录"}`，
此后 `GET /api/v1/auth/me`（旧 access）**401**、`refresh(R2)` 也 **401** —— 全废。

**为什么会踩**：`app.py::force_login_pages` 里**每个被拦下的 HTML 请求**都会拿 cookie 里的 refresh 换新，
原来没有去重：并发（多标签/预取/WebView 恢复）或客户端没把轮转后的新 cookie 存住，
同一枚 refresh 就会被重复提交 → 一次误判把 cookie 全废 → 之后**每开一次 App 都只能重新登录**。

**已修（本仓库）**：
1. `app.py` 新增 `_PAGE_RENEW_TTL/_page_renew_cache/_renew_once()`：同一枚 refresh 60s 内**只提交一次**，
   并发请求复用同一份结果（双检 + `threading.Lock`），闸门改调 `_renew_once`。
   实测复核：同一枚 refresh **并发 6 个**页面请求 → 全部 `200`、拿到的是**同一对**新 cookie、
   会话未被撤销（`/auth/me` 与新 refresh 均 200）。
2. `templates/login.html`：统一认证入口原本是裸 `href="/api/lugwit/sso/start"` —— 导航类链接
   **不会**被 base.html 的 fetch 前缀补丁覆盖；线上实测 `GET https://121.196.144.88/api/lugwit/sso/start`→**404**
   （该主机根下没有 `/api/lugwit/*`，只有 `/l_wchat/api/lugwit/*`），改为 `window.__wurl()` 补前缀。
   ⚠️ **2026-10-08 复核订正**：源码里这行**仍是裸 `href`**（`templates/login.html:28`）——
   真正补前缀的是 `templates/base.html` 末尾的 `fixAttr('a[href^="/"]', 'href')`
   （DOM 就绪后统一给绝对路径链接加 `/l_wchat`），线上实测没坏。别再按「login.html 里有 `__wurl`」去找。

**2026-10-08 落地实测（127.0.0.1:1234 + auth 1027）**：

- 登录响应现在 **成对**下发 `lugwit_token` + `lugwit_refresh`（均 HttpOnly / Path=/ / Max-Age 2592000）。
- 模拟「access 已过期、重开 App」：`Cookie: lugwit_token=EXPIRED; lugwit_refresh=<有效>` →
  页面闸门**静默续期**并照常出页（`307 → /postpartum`），响应里写回**新的一对** cookie；
  连做 4 次全部 `307` 放行。
- 对照组（证明登录没被拿掉）：坏 access 且无 refresh → `302 /login?next=/album`；无 cookie → 同样 `302 /login`。
- **地雷（务必保留写回）**：lugwit_auth 的 refresh 是**轮换式一次性票据** ——
  拿它换过新 access 后，同一枚再来一次就是 `401 refresh token 无效、已过期或已被使用`。
  所以续期时 **access 与 refresh 必须一起写回**；只写 access 会在下一个周期照样断。

**验收（30 秒）**：手机/浏览器进站登录一次 → 完全杀掉 App/进程 → 重开，应**不再**弹登录页；
若仍弹，按上面两条依次核：① 闸门是否重复提交 refresh（看 l_WChat 日志里 401 的 detail）② 客户端是否把 `Set-Cookie` 存住了（DevTools → Application → Cookies，看 `lugwit_token`/`lugwit_refresh` 的 Expires 是不是 30 天后）。

### 4.7 真因订正（2026-10-08）：症状在 App 侧 —— WebView 的 cookie 不落盘

**§4.5 / §4.6 修的是真 bug，但不是这个症状的因。** 2026-10-08 用云端日志（手机 IP
`124.160.204.249`，源文件 `rez-package-source/l_WChat/999.0/src/l_WChat/wchat-android/.../MainActivity.java`）
把时间线拉出来，症状与「refresh 缺失」对不上：

```
21:55:23  GET /l_wchat/api/chat/heartbeat  200   referer=/l_wchat/album   ← 老进程，cookie 正常
21:55:31  GET /l_wchat            302  (无 cookie，referer 空 = 顶层新导航)
21:55:31  GET /l_wchat/           302  → /l_wchat/login?next=%2F  200
21:55:32  GET /l_wchat/api/lugwit/me   401        ← WChat 一个令牌都没收到
21:55:41  POST /l_wchat/api/lugwit/login 200 → 之后全 200/307 正常
```

- **同一次会话里 cookie 完全正常**（19:41:07 登录 → 19:41:08 `/album` 直接 200）；
  **进程一死一重启就一个都带不上**（新进程首个请求就 302 撞登录页）。当天 4 次
  `POST /api/lugwit/login`（09:03 / 12:29 / 19:41 / 21:55）≈ 4 次重开 App。
- 所以 refresh 那套**根本没机会用**：access 都没送到服务端，无从续期。§4.5 里
  「cookie 有 30 天是假象」的推断不成立——access 确实是 30 天，只是**没被持久化**。

**成因**：Android WebView 的 cookie 是**延迟落盘**的，进程被划掉/被系统杀时未 flush 的写入直接丢；
而 `localStorage`（`baby_style` / `tts_provider` / `mosaicMode` 这些用户看得见的设置）是另一套存储、
逐条提交 —— 这正好解释「设置不丢、只有登录态丢」。

**修法（App 侧，已落地）**：`MainActivity.java`
1. `onCreate` 里 `CookieManager.getInstance().setAcceptCookie(true)`；
2. 新增 `flushCookies()`，在 `onPause()` / `onStop()` 各调一次
   `CookieManager.getInstance().flush()`（切后台/划掉都覆盖）。

> ⚠️ **2026-10-10 补**：只靠上面这条**覆盖不全**（后台强停/省电清理/被 OEM 杀都不走生命周期回调，
> 实测装了带 flush 的包当天仍被弹 4 次）。已在同一个类里加「**原生备份 + 冷启动回灌**」，
> 并同时修掉服务端那个"auth 不可达就弹登录页"的脆弱点 —— 见 **§4.9**。

**验收**：登录一次 → 划掉 App → 重开不再弹登录；且 nginx `access.log` 里
`POST /l_wchat/api/lugwit/login` 不再「每次重开一条」。需要**重打 APK**（`build-apk.bat`
→ `static/apk/baoma.apk` → `deploy_l_wchat.py` 推云端 → App 内「更新软件」）。

### 4.8 第三个真因（2026-10-10）：一个客户端的 refresh 重放，把**所有人**的会话全量吊销

**症状**：**浏览器**（不是 App）进 `http://localhost:8080/l_wchat`，"每次 l_wchat 服务重启后
都要重登"，且一天里反复弹登录页。与 §4.7 的 App cookie 落盘无关。

**取证（2026-10-10 13:30–14:15，本机 1027 / 1234 / 8462）**：

- `users.token_version`：13:41 = **3170** → 13:48 = 3184 → 13:52 = 3187，约 1–2 次/分钟。
  自动 `+1` 的路径**只有一条** —— `session_service.rotate()` 的"复用 = 泄露"分支。
- 当天 auth 日志：`POST /api/v1/auth/refresh` = **58 × 401、0 × 200**；`login` = 69 × 200。
  即某客户端每 30–40s 拿一枚**已废**的 refresh 去撞，每次都在吊销该账号**全部**会话
  （浏览器 cookie、托盘 DPAPI、agent_chat 的 store 一起作废）。
- 会话表 200 行里 **193 行 UA = `python-requests/*`**，每 ~35s 新增一行。
- **停掉 `l_model_hub_server` 后 7 分钟，`refresh 401` 计数 59 → 59（纹丝不动）** ⇒ 肇事者就是 hub。

**成因**：`l_model_hub/auth_secrets.py::_refresh()` 换不动时**把这枚已废 refresh 留在内存**
（`_token["refresh"]`），轮换响应缺新 `refresh_token` 时还会**沿用旧的**；下一次又把它提交一次
→ 每次都触发全量吊销 + `token_version+1`。WChat 的重启只是"你去看页面的那一刻"。

**修法（已落地）**：

1. **服务端治本①·宽限窗** `lugwit_auth/session_service.py::rotate()` —— 新增 `_REUSE_GRACE_S`
   （`LUGWIT_REFRESH_REUSE_GRACE_S`，默认 60s）：刚轮换完又被重放（并发 / 重试 / 多进程共享
   同一枚 / 客户端没存住新 cookie）**只补发一枚新 refresh，不做任何吊销**；**超出宽限窗才**
   按泄露处理。两条路径**都用 `_logger.warning`**（本服务没配 logging handler，`info` 不落日志，
   而"有客户端在重放"恰恰是这类事故唯一的现场线索）。
2. **服务端治本②·按族吊销**（`sessions.family`，`schema_upgrade.py` 加列 + 老行补号）：
   登录**新开一族**、轮换**继承族号**；泄露只吊销**那一族**（原来是吊销该账号全部会话 → 无辜
   设备连坐）。`token_version+1` 照旧（把可能落到别人手里的 access 一起作废），但别的族的
   refresh 会话**不受影响**。
3. **hub 侧** `l_model_hub/auth_secrets.py` —— 新增 `_drop_refresh()`：refresh 失败（401/400）
   或轮换响应缺 access/新 refresh 时**立刻清掉内存票据**，下次走 `_login()`；绝不再把一枚
   已废 refresh 拿去撞。
4. **SDK 侧** `lugwit_auth_client/client.py::refresh()` —— 401 时把本地那枚废 refresh **清掉**
   （`_drop_dead_store()`，只清"存的确实就是这一枚"的情况），任何用 SDK 的客户端都不会每轮重放。
5. **会话行 churn（`login` 每 ~68s 一条）**：`l_model_hub/usage_store.py::push_to_depot()` 原本
   **每次推送都重新 login**（flush 60s 一次 → 60s 一条新 session，一天一千多条；"会话表 193/200 行
   是 `python-requests`"就是这么来的）。改为按响应 `expires_in` 缓存 token（`_depot_login`），
   只有 depot 回 401 时才丢缓存重登。

**实测（改完 + 重启）**：① 同一枚 refresh 连发两次 → **都 200**（原来第二次 = 全量吊销）；
② 隔 65s 再重放 → 401，但**别族的 refresh 仍能 200**（家族隔离生效）；③ 重启后 `refresh` 全是 200、
**0 × 401**，`token_version` 不再增长；④ WChat `POST /api/lugwit/login` → `/growth` **200**。

**同一模式还留在别处（未改）**：`l_notepad_server/depot_map.py`、`l_log/backend/depot_logs.py`、
`l_agent_chat/depot_sync.py` 也是"推 depot 前先 login 一次"——同样会持续堆 session 行（**无害**，
不会吊销任何人；要清账照 `usage_store._depot_login` 的写法加缓存即可）。

### 4.9 公网仍被弹登录页（2026-10-10 晚）：两个都不在服务端吊销上的成因

**症状**：公网手机上仍"经常要重新登录"。

**先排除服务端**（公网日志实读，2026-10-10 全天）：auth 侧 `POST /api/v1/auth/refresh` = **1 次**（401）、
`宽限窗内重放` = **0**、`判定为泄露` = **0** → **没有任何全量/按族吊销**。当天真正被弹 + 手填密码的
**只有 2 次（13:52:33、14:08:34）**，全是 Android `MAG-AN00`（另有 12:23 一台 `ASUS_X00TD`）；
桌面浏览器 **0 次**。附注：nginx 里 779 次 `/login` 命中是 `127.0.0.1` 的 `python-requests`
（每 ~68s 一次，**14:50:17 后归零** —— 就是 §4.8 那个 hub depot 流程，被 token 缓存止住了），不是用户。

**成因①（服务端脆弱点）**：WChat 页面闸门把 **auth 不可达当成"未登录"** ——
`verify_with_auth` 对"连不上"与"token 无效"都返回 `None`，`_page_login_ok` 于是判 False、续期也失败
→ 302 `/login`，而且这个 False 还进缓存 60s。**结果：auth 一重启/网络一抖，那一分钟内所有人刷新都被弹**。

**修法①**（`l_WChat`，已落地并实测）：
- `api/auth.py::verify_with_auth`：不可达**抛 `AuthUnreachable`**（只有"服务端明确拒绝"才算没登录）。
- `app.py::_page_login_ok`：三态 `True/False/None`，`None` **不进缓存**；`force_login_pages` 遇 `None`
  **放行页面**（fail-open，不写 cookie），并打一条 60s 节流的 `[gate]` 告警。
- `api/routes.py::_require_login`：不可达 → **503**（"现在验不了"），不再 401（前端把 401 当登录态失效）。

| 场景（停掉 auth 实测） | 改后 | 改前 |
|---|---|---|
| auth 挂 + 有 cookie → 页面 | **200** | 302 → 登录页 |
| auth 挂 + 有 cookie → `/api/lugwit/me` | **503** | 401 |
| auth 挂 + 无 cookie → 页面 | 302 `/login?next=…`（不变） | 同 |
| auth 恢复 → 页面 / `/me` | 200 / 200 | 同 |

**成因②（手机端）**：WebView 的 cookie 库**延迟落盘**，`onPause/onStop` 里 flush 覆盖不全
（后台强停、省电清理、被 OEM 杀都不走生命周期回调）。实测证据：手机 10-08 22:23 已下载过带 flush
的包（线上 APK 4741050 字节与本地同源），当天（10-10）**仍被弹 4 次**。

**修法②**（B-1：原生备份 + 冷启动回灌，`MainActivity.java`，已落地）：
- `saveCookies()`：用 `CookieManager.getCookie(当前页 URL)` 把整串 cookie 存进 **原生 SharedPreferences**
  （键 = 该页面的 origin）；**看不到 `lugwit_token=` 就清掉备份**（登出后别又灌回来）。
  时机：`onPause` + 前台每 30s 一次（Handler 轮询 —— 被强杀时不走生命周期，只靠 onPause/onStop 兜不住）。
- `restoreCookies()`：在 `super.onCreate()` **之前**（WebView 发第一个请求前）调用；库里已有 `lugwit_token`
  就跳过（别用旧备份盖掉好的），否则**逐条** `setCookie(origin, "k=v; Path=/")` + `flush()`。
  两个坑：`setCookie(url, value)` 把整串当**一条** cookie 解（`a=1; b=2` 会把 b 当属性）→ 必须逐条；
  `Path` 必须显式给 `/`（Android 默认按 url 路径推导会种成 `/l_wchat`，而 WChat 种的是 `/`）。
- 放**原生**而不是 localStorage：JS 读不到、XSS 偷不走（localStorage 方案见"仍可做"）。

**交付状态**：`build-apk.bat` 重打 + 覆盖 `static/apk/baoma.apk` → 推云端 → **手机上更新一次**才生效。

> 🧱 **构建坑（2026-10-10 实测）**：`wchat-android` 树里有 **134 个文件**（`res/mipmap-*` 图标 +
> `node_modules/@capacitor/**`）被压成 `SparseFile + ReparsePoint`（WOF 压缩产物）——`dir` 看不出来，
> 得看 `Get-Item … | Select-Object Attributes,LinkType`。Gradle 的输入快照不认这种文件，直接
> `java.io.IOException: Cannot snapshot …: not a regular file` 把整次 build 打挂（任务点随机，
> 先报 `:app:processDebugNavigationResources`、修完又报别的文件）。**修法**：读出来重写一遍即还原成
> 普通文件；已在 `build-apk.bat` 的 gradle 之前加了一道自动还原（只动"有 ReparsePoint 且无 LinkType"
> 的，避开真符号链接）。

**仍可做**：B-2（服务端 `restore_key` + 前端 localStorage）—— 覆盖浏览器侧；代价是要么把 refresh
放服务端保管，要么让 token 落到 JS 可读的存储，安全面都更大。

## 5. 闹钟（循环闹钟 + 全屏响铃）

### 数据
- `api/routes.py`：`AlarmBody` 含 `interval_days`、`cycle_reminders`、`cycle_start_date`、`repeat_count`、`repeat_max_seconds`
- 存储于 `USER_DATA_DIR/alarms.json`，保存用 `{**旧, **新}` 合并（旧服务端没有新字段时会被 Pydantic 静默丢弃——云服务器必须同步到含新字段的代码）

### 循环定位算法（前后端一致）
```text
diffDays = 今天(零点) - cycle_start_date
idx = diffDays < 0 ? 0 : diffDays % interval_days
cycle_start_date 为空 → idx = 0
```
- 原生侧：`AlarmSchedulerPlugin`
- 网页侧：`templates/alarms.html` 的 `todayCycleIndex()`，列表卡片显示「📍 今天是第 N 天 · HH:MM「label」」
- 编辑回填：`editAlarm()` 从 `a.cycle_start_date` 恢复表单（若为空 → 云端代码/页面是旧的）

### 全屏响铃
- `AlarmAlertActivity`：全屏详情2 + 关闭页；`singleInstance` + `taskAffinity=""` + `excludeFromRecents`；API 27+ 用 `setShowWhenLocked/setTurnScreenOn`，低版本退回窗口 flag
- `AlarmReceiver`：通知渠道 `setBypassDnd(true)`；`setFullScreenIntent(alertPi, true)`；`notifId = abs(label.hashCode()) % 100000`；「⏹ 关闭闹钟」广播同时停铃声 + 取消通知
- 权限：`AndroidManifest.xml` 加 `USE_FULL_SCREEN_INTENT`；厂商 ROM（小米/华为等）若被降级为横幅，需在系统设置里允许「全屏通知/锁屏显示」

## 6. 文章页原文链接

`templates/postpartum.html` / `pregnancy.html` / `motherhood.html` 底部均有醒目的「📄 原文链接（点击打开）」块。链接由页面 JS 用 `location.origin`（App 内即云服务器地址）拼当日 URL；`shareUrlBase()` 经 `/api/lan-ip` 把 localhost 换成局域网 IP，便于手机浏览器直接打开。

## 7. 服务部署

云端就是**部署机**（`121.196.144.88`，远端树根 `D:\TD_Depot\Software\Lugwit_syncPlug\lugwit_insapp\trayapp`），
同步走直推脚本（**不是**手工拷文件）：

```bat
wuwo\wuwor.bat l_repo_sync_gui -- python rez-package-source\l_repo_sync_gui\999.0\deploy_l_wchat.py           :: 推整包 + card 重启 1234
wuwo\wuwor.bat l_repo_sync_gui -- python ...\deploy_l_wchat.py --dry                                          :: 只报 sha256 差异
```

- 内容级比对：`--dry` 会逐文件 sha256 报「已一致 / 内容不同 / 远端缺」—— 这是「到底传上去没有」的直接证据，
  别看退出码。**只推内容变了的**，`static/apk/baoma.apk` 也在清单里（重打 APK 后一并推）。
- 远端 `1234` 是主页托管服务：脚本走**卡片 API** 重启（带重启锁/守护/历史），随后探活 `/api/album/views`。
- ⚠️ 旧的「仓库内无自动部署脚本、手工同步」口径作废（2026-10-08 订正）。
