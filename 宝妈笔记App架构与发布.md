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

**验收**：登录一次 → 划掉 App → 重开不再弹登录；且 nginx `access.log` 里
`POST /l_wchat/api/lugwit/login` 不再「每次重开一条」。需要**重打 APK**（`build-apk.bat`
→ `static/apk/baoma.apk` → `deploy_l_wchat.py` 推云端 → App 内「更新软件」）。

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
