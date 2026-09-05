# 宝妈笔记 App 架构与发布（l_WChat）

> 项目根：`rez-package-source/l_WChat/999.0/src/l_WChat`
> 后端：FastAPI（`app.py`，端口 1234，uvicorn `--reload`）
> App：`wchat-android/`（Capacitor + Android WebView）
> 更新：2026-09-02

## 1. 架构总览

```
手机 App（Capacitor WebView）
   │  capacitor.config.json → server.url
   ▼
http://121.196.144.88:1234   ← 云服务器（App 实际加载的页面/接口）
   ▲
http://localhost:1234        ← 开发机（改代码先在这里验证）
```

**关键点：`capacitor.config.json` 的 `server.url` 指向云服务器，App 加载的 ALL 页面来自云。**
开发机上改完代码，手机 App 看不到任何变化，必须同步到云服务器。本地验证用浏览器开 `http://localhost:1234/...`。

- `cleartext: true`（允许 http 明文）
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

仓库内**无自动部署脚本**，云服务器（121.196.144.88）需手动同步：
- 源码：`src/l_WChat/` 全量
- 产物：`static/apk/baoma.apk`
- 同步后重启 uvicorn（`--reload` 只在 dev 用）
