# l_WChat（宝妈笔记）上传直连百度 —— 改造计划

日期：2026-09-14 · 对象：`rez-package-source/l_WChat/999.0`（源码在 `999.0/src/l_WChat/`）
相关文档：《Rez_pkg/百度云接口元数据实测.md》（md5 可还原）、《网盘版本库Depot演进计划.md》（T2 客户端直传）

---

## 0. 结论摘要

**目标**：手机上传文件时，**文件字节不经我们的服务器**（手机 → 百度直连）；用户与页面业务代码**零感知**。

**可行性**：可行，但要三个前提同时满足：
1. **1028 网盘服务**新增两个"内部"端点：`upload_prepare`（precreate + locateupload，秒传短路）、`upload_finish`（create + 还原 md5 复核 + 登记）
2. **WChat 服务**新增两个"内部"端点（转发上述两个，并负责把结果接到本地业务），以及前端**统一上传拦截层**
3. **安卓壳**加一个小的**原生上传插件**（Kotlin + OkHttp，约 150 行）——因为 WebView 里 `fetch` 直连百度会被 CORS 拦住，而 dlink 还要求 `User-Agent: pan.baidu.com`（浏览器改不了 UA）

**必须同时修的安全问题**：当前 WebView 从 **明文 HTTP**（`http://121.196.144.88:1234`）加载页面，
JWT cookie 已在公网上裸奔；一旦下发百度 `access_token`，风险放大到"整个网盘"。
**建议把 HTTPS 作为本计划的第 0 期**。

**分期收益**：
| 期 | 内容 | 收益 |
|---|---|---|
| M0 | WebView 切 HTTPS | 安全前提 |
| M1 | **秒传探测**（零字节） | 重复内容一个字节都不用传，**客户端零改动**（只改服务端） |
| M2 | 小文件（≤4MB）直连 | 相册/背景图/餐图不再经服务器 |
| M3 | 大文件分片直连 + 进度 + 断点续传 | 音频/视频/大图不再经服务器 |
| M4 | 下载直连（dlink 下发）/ 懒回源 | 服务器出向流量再降 |

---

## 1. 现状（改造基线）

### 1.1 技术栈

| 层 | 实现 | 证据 |
|---|---|---|
| 后端 | Python 3.12 + FastAPI + uvicorn，端口 **1234**，入口 `src/l_WChat/app.py:258-284` | `package.py:16-34`（别名 `l_wchat_backend`） |
| 前端 | Jinja2 模板 + 原生 `fetch`（无框架），`templates/*.html` | `templates/base.html:22-28` 包装 `window.fetch` 补 `/l_wchat` 前缀 |
| 安卓壳 | **Capacitor 8.5**（`@capacitor/android ^8.5.0`），无任何业务 Kotlin/Java 代码 | `wchat-android/package.json:12-17` |
| WebView 配置 | `appId com.lugwit.wchat`、`server.url = http://121.196.144.88:1234`、`cleartext: true`、`allowMixedContent: true` | `wchat-android/capacitor.config.json:1-13` |
| 鉴权 | WChat 用 `config.json:29-30` 的 admin01/666 调 `/api/v1/auth/login` 拿 token，存 cookie `lugwit_token` | `services/baidu_netdisk.py:108-180` |

### 1.2 当前上传链路（全部经服务器）

```
手机(WebView) ──FormData──► WChat(1234) ──落本地磁盘──► 后台线程 ──► 1028 网盘服务 ──► 百度
                                     └─ 用 cookie lugwit_token 调 1028
```

- WChat 侧唯一写云端封装：`services/baidu_netdisk.py`（`upload_bytes:183`、`upload_file:215`、`_api:149`、`download_bytes:278`）
- **整文件一次上传**（`POST /api/files/upload_stream`，raw bytes），**无分片、无进度、180s 超时、无重试**
- 1028 侧**也没有**分片接口与直传票据接口（这是本次要新增的）

### 1.3 上传点清单（穷举，改造要全覆盖）

| # | 场景 | 触发 | 前端 | 服务端 | 云端目录 |
|---|---|---|---|---|---|
| 1 | 相册照片 | 用户选图 | `templates/album.html:712-731` `FormData` | `api/routes.py:3856-3902` `upload_album_photo()` | `l_wchat/album/<group>` |
| 2 | 相册马赛克 | 保存编辑 | — | `api/routes.py:4039-4058` | 同上（覆盖） |
| 3 | 成长记录 JSON | 点"同步" | `templates/growth.html:1411` | `services/growth_record.py:213-246` → 并额外 `POST /api/depot/submit_stream`（`:407-444`） | `l_wchat/成长记录/` + depot `/l_wchat` |
| 4 | TTS 音频 | 点朗读 | — | `api/routes.py:349-381` `tts()` | `l_wchat/音频` |
| 5 | 公众号文章 JSON | 生成即备份 / 手动转存 | `postpartum.html:230` 等 | `services/postpartum_journal.py:326-346`、`api/routes.py:3603-3710` | `l_wchat/公众号每日推送文章` |
| 6 | 餐图 | 生成/重生成 | — | `services/postpartum_journal.py:579-584` | `l_wchat/diet_img/<date>` |
| 7 | 检查报告截图 | 用户上传 | `templates/inspection_reports.html:228-238` | `api/routes.py:2086-2136` | **仅本地，未上云** |
| 8 | 网页背景图 | 用户上传 | `templates/settings.html:733-739` | `api/routes.py:628-648` | `l_wchat/背景` |

> 3/4/5/6 是**服务端生成**的内容（JSON/图片/音频由服务器产出），没有"手机侧字节"，
> 属于"服务器 → 百度"，本计划不要求直连（服务器出向流量另算，见 M4）。

### 1.4 下载链路

全部经 WChat 本地文件路由（`GET /api/album/file/{name}` 等，`api/routes.py:3943`），
本地缺失时才用 `baidu_netdisk.download_bytes()` 从 1028 回源（`GET /api/files/meta` + `/api/files/download`）。
**没有一处直连百度 dlink**。

### 1.5 现状三个隐患

1. **明文 HTTP**：页面与 API 都走 `http://121.196.144.88:1234`，`lugwit_token` cookie 在公网明文传输
2. **全量中转**：手机上传的每一个字节都先落到服务器磁盘，再上传一次到百度（服务器带宽 ×2）
3. **无进度无续传**：大文件（音频、视频）弱网下只能重来

---

## 2. 目标与非目标

**目标**
- G1 用户无感：界面上仍是"选文件→上传"，不出现新按钮/新概念
- G2 业务代码无感：页面里的 `fetch('/api/album/upload', {body: FormData})` **一行不改**
- G3 字节优先直连：能直连就直连，**秒传命中时零字节**
- G4 永不失败感：任何直连异常自动回退现有上传路径（服务端代传），用户看不出差别
- G5 服务器不再做数据中继（M2/M3 完成后，相册/背景/餐图/音频的手机侧字节不再经过 1234）

**非目标**
- 不改动服务端生成内容的路径（TTS/餐图/文章 JSON 仍由服务器生成后上传）
- 不做 iOS（当前只有安卓壳）
- 不引入百度之外的存储

---

## 3. 总体方案

### 3.1 关键洞察：在 `fetch` 包装层做"透明改写"

`templates/base.html:22-28` 已经把 `window.fetch` 包了一层（只加 `/l_wchat` 前缀）。
**把直传逻辑挂在这里**，就能做到"页面代码零改动"：

```
页面代码：  fetch('/api/album/upload', {method:'POST', body: formData})   ← 保持不变
     │
 拦截层（base.html 的 fetch 包装）
     │  判断：POST + body 是 FormData + 含 File 且 > 阈值?
     ├─ 否 → 原样 fetch（现状路径）
     └─ 是 → 走直传流程 ↓（任一步失败 → 回退原样 fetch）
```

### 3.2 直传时序（M2/M3）

```
WebView(拦截层)                WChat(1234)                 1028                 百度
  │ ① 本地算 4MB 分片 md5（JS）
  │ ② POST /api/upload/prepare ──► 转发 ──► precreate(path,size,block_list)
  │    {path,size,block_list}                     └─ 命中(return_type=2)?
  │ ◄── {rapid:true} ─────────────────────────────── 是 → 直接结束（零字节）
  │ ◄── {access_token, upload_host, uploadid, parts:[…]}   否
  │
  │ ③ 分片直传（**手机 → 百度**，经原生插件，绕 CORS/UA 限制）
  │    POST {upload_host}/rest/2.0/pcs/superfile2?method=upload
  │         &access_token=…&type=tmpfile&path=<dpath>&uploadid=…&partseq=N
  │    body = multipart/form-data，字段名 `file`
  │
  │ ④ POST /api/upload/finish ────► 转发 ──► create_file(...) + filemetas 复核（还原真 md5）
  │    {path,uploadid,size,md5}                    + 登记（depot/本地索引）
  │ ◄── {ok, url} ──────────────────────────────
  │
  └─ 任一步异常 → 回退：原样 fetch('/api/album/upload', …)（现状路径，落盘+后台同步）
```

### 3.3 为什么必须有原生插件（关键约束）

| 约束 | 说明 | 对策 |
|---|---|---|
| **CORS** | 页面 origin 是 `http://121.196.144.88:1234`，`fetch` 到 `https://d.pcs.baidu.com` 需百度放行该 origin → 不会 | 用 **CapacitorHttp**（`@capacitor/core` 自带，原生传输，无 CORS）或自写插件 |
| **UA** | 百度 dlink 下载要求 `User-Agent: pan.baidu.com`；WebView 改不了 fetch 的 UA | 同上（原生请求可自设 header） |
| **二进制 body** | CapacitorHttp 的 `data` 对 raw/multipart 二进制支持有限（可能要 base64，4MB 片 → 5.3MB 字符串，内存与开销都亏） | **自写 Kotlin 插件**（OkHttp 直发 multipart，带进度回调）——预计 150 行 |
| **明文 HTTP** | 票据里含百度 access_token，明文传输不可接受 | **M0 先切 HTTPS** |

> 先做一次**可行性探针**（半天）：用 CapacitorHttp 试发 1 个 4MB 分片到 `superfile2`。
> 通过 → 省掉自写插件（M3 只需 JS）；不通过 → 按计划写 Kotlin 插件。

---

## 4. 接口契约（内部端点，**不对外文档化**）

### 4.1 1028 网盘服务（新增）

**`POST /api/upload/prepare`**（或挂在 depot 下：`/api/depot/upload_prepare`）
```jsonc
// 请求
{ "path": "/apps/Lugwit/l_wchat/album/baby/2026-09-14_IMG_1234.jpg",
  "size": 5242880,
  "block_list": ["3f2a…", "8c01…"] }        // 4MB/片，顺序即 partseq
// 响应（秒传命中）
{ "rapid": true, "path": "…", "fs_id": 123, "md5": "<标准 md5>" }
// 响应（需上传）
{ "rapid": false,
  "upload_host": "https://c3.pcs.baidu.com",  // locateupload 实时结果，勿缓存
  "uploadid": "…",
  "access_token": "…",                        // 受开关控制，见 §7
  "parts": [0, 3, 5],                         // 百度告诉我们缺哪几片（断点续传）
  "block_size": 4194304 }
```
服务端动作：`locate_upload_host`（`baidu_netdisk_api.py:252`）+ `precreate`（`:283`）；
`path` 必须被约束在允许的根下（`/apps/Lugwit/l_wchat/**`），**禁止任意路径**。

**`POST /api/upload/finish`**
```jsonc
{ "path": "…", "uploadid": "…", "size": 5242880, "md5": "<客户端算的整文件 md5 或留空>" }
// 响应
{ "ok": true, "fs_id": 123, "md5_real": "<还原出的真 md5>", "size": 5242880 }
```
服务端动作：`create_file`（`:331`）→ `file_metas`/`meta_by_path` 复核
（**用 `decrypt_baidu_md5()` 还原真 md5 与客户端声明比对**，现在已可做到，实测 279/280）
→ 登记（`depot_blob` / WChat 的本地索引）→ 触发 WChat 侧后续（见 §6）。

### 4.2 WChat 服务（新增，对前端内部使用）

| 端点 | 作用 | 备注 |
|---|---|---|
| `POST /api/upload/prepare` | 透传 1028，并补上"业务上下文"（相册分组、餐图日期） → 生成最终 `path` | 需要 `lugwit_token`（沿用现有鉴权） |
| `POST /api/upload/finish` | 透传 1028 + 写入 WChat 的本地索引/去重表 + 触发必要的本地动作 | 返回业务 id（如 album 的记录 id） |

**兼容**：现有的 `POST /api/album/upload`、`/api/config/bg`、`/api/inspection-reports/upload`
**全部保留**，作为回退入口（也是服务端生成内容的上传入口）。

---

## 5. 改造清单（逐文件）

### 5.1 前端（`999.0/src/l_WChat/`）

| 类型 | 文件 | 位置 | 预估 | 内容 |
|---|---|---|---|---|
| 新增 | `templates/static/upload_direct.js` | 新文件 | 200-300 行 | 分片器：读 File → 4MB 切片 → **MD5**（`crypto.subtle` 无 MD5，需内置小型 md5 实现 ~2KB）→ prepare → 原生直传 → finish → 进度回调 → 失败回退 |
| 改 | `templates/base.html` | `:22-28` fetch 包装 | +40 行 | 拦截 POST+FormData+File：调用 `uploadDirect()`；失败则 `return origFetch(...)` |
| 改 | `templates/album.html` | `:712-731` | +10 行 | 接进度回调显示（可选） |
| 改 | `templates/settings.html` | `:733-739` | +10 行 | 同上（背景图） |
| 新增 | `wchat-android/android/app/src/main/java/…/DirectUploadPlugin.kt` | 新文件 | ~150 行 | OkHttp 分片 POST（multipart 字段 `file`，自设 UA），进度回调、取消、超时、重试 |
| 改 | `wchat-android/android/app/src/main/java/…/MainActivity.java` | 注册插件 | +3 行 | `registerPlugin(DirectUploadPlugin.class)` |
| 改 | `wchat-android/capacitor.config.json` | `:5-9` | +2 行 | `server.url` 切 HTTPS（M0）；如走 CapacitorHttp 无需插件 |

### 5.2 WChat 服务端

| 类型 | 文件 | 位置 | 预估 | 内容 |
|---|---|---|---|---|
| 新增 | `api/routes.py` | 新路由 | ~120 行 | `/api/upload/prepare`、`/api/upload/finish`；路径白名单；幂等/去重 |
| 改 | `services/baidu_netdisk.py` | 新增方法 | ~80 行 | `upload_prepare()`、`upload_finish()`（调 1028 新端点）；保留 `upload_bytes` 作回退 |
| 改 | `api/routes.py` | `_baidu_upload_file():3838` | ~10 行 | 直传成功后跳过"服务器再传一次" |
| 改 | 各上传入口 | `album:3895`、`bg:648`、`tts:367`、`diet:579` | 各 ~5 行 | 记录"这份内容已在云端"的索引，避免重复中转 |

### 5.3 1028 网盘服务

| 类型 | 文件 | 位置 | 预估 | 内容 |
|---|---|---|---|---|
| 新增 | `baidu_netdisk_api.py` | 导出已有零件 | ~30 行 | 让 HTTP 层能直接用 `precreate` / `locate_upload_host` / `create_file`（现在只在内部用） |
| 新增 | `web_server.py` | 新路由 | ~120 行 | `/api/upload/prepare`、`/api/upload/finish`（含路径白名单、`decrypt_baidu_md5` 复核、depot 登记钩子） |
| 改 | `depot_service.py` | 复用 | ~20 行 | finish 后（若目标路径属某库）按库登记 blob（**用还原 md5，零下载**） |

---

## 6. 数据一致性设计（重要，别漏）

现状：文件先落 WChat 本地磁盘，再上传百度；本地副本被缩略图、马赛克、AI 分析、微信推送复用。
**直传后手机不再把文件发给 WChat**，所以要定义"本地副本从哪来"：

| 方案 | 做法 | 优点 | 缺点 |
|---|---|---|---|
| **A 懒回源（推荐）** | 本地只在需要时（缩略图/马赛克/AI/微信）从百度回源一次并缓存 | 服务器不再承担"上传中转"，出向流量可控 | 首次访问有延迟；要改取数点 |
| B finish 后后台拉回 | `upload/finish` 成功后由 WChat 后台从百度下载一份到本地 | 改动最小（本地语义不变） | 服务器出向流量 = 文件大小（等于把入向换成了出向） |
| C 混合 | 小图/缩略图 lazy；原图/AI 需求触发时再拉 | 折中 | 逻辑略复杂 |

**建议 M2 用 B（先保证功能不回归），M4 切 A（真正省流量）**。
另外：`growth_record` 已经在往 depot 提交（`growth_record.py:407-444`），
直传改造后这条路保持"服务器生成 → 提交"，不受影响；若以后手机直接传成长记录附件，走同一 prepare/finish。

---

## 7. 安全设计

| 项 | 要求 |
|---|---|
| **HTTPS** | M0 必做：`capacitor.config.json.server.url` 与 nginx 入口上 TLS。否则 `lugwit_token` 与下发的百度 token 都是明文 |
| **token 下发开关** | 1028 加环境变量 `LUGWIT_UPLOAD_TICKET_TOKEN`（默认 **0=关**）。关时 `prepare` 不返回 `access_token` → 前端自动回退服务端代传（零风险）；开启时才下发（配合 HTTPS + 专用应用授权） |
| **路径白名单** | `prepare` 只接受 `/apps/Lugwit/l_wchat/**`（或 depot 允许的库根），防越权写任意目录 |
| **票据短时性** | `uploadid` 与 path+size+block_list 绑定、生命周期短；`access_token` 无法单独撤销 → 建议**给手机上传单独建一个百度应用授权**（凭证独立、可单独撤销），配置 `LUGWIT_BAIDU_UPLOAD_TOKEN_FILE` 隔离 |
| **日志脱敏** | 服务端/客户端日志禁止打 `access_token`、dlink |
| **最小暴露** | 只在直传窗口内持有 token（内存），不落 localStorage/sqlite |

---

## 8. 分期实施

| 期 | 内容 | 改动面 | 验收 |
|---|---|---|---|
| **M0** | HTTPS（nginx + `server.url`） | 配置 | 浏览器/抓包看不到明文 cookie |
| **M1** | **秒传探测**：`upload_prepare` 只做 `precreate`，命中则服务端 `create` 登记，客户端**不上传** | 1028 + WChat（**前端零改**） | 重复上传同一张照片：手机流量 0、服务器入向 0；库里出现新版本 |
| **M2** | 小文件直连（≤4MB，单分片）：CapacitorHttp 探针 → 通过则 JS 直传，否则自写插件 | 前端 + 插件 + 两端端点 | 手机上传 1 张图：抓包显示请求走 `d.pcs.baidu.com`；服务器 `netstat`/带宽计数≈0（仅几 KB 元数据） |
| **M3** | 大文件分片直连 + 进度 + 断点续传（`parts`）+ 失败回退 | 前端 + 插件 | 弱网中断后重试只传缺片；进度条正确；回退路径可用 |
| **M4** | 下载直连（下发 dlink + UA）/ 本地懒回源（§6-A） | 1028 + WChat + 前端 | 相册原图/音频由手机直连 `d.pcs.baidu.com`；服务器出向流量下降 |

**每期都必须做的一件事**：回退开关。任一期上线都保留"全局关闭直传"的开关（环境变量/配置项），
出问题一键回到现状。

---

## 9. 测试与验收方法

1. **curl 模拟手机**（不需要真机就能验服务端）：
   ```bash
   # 1) 算 block_list（用我们的工具或本地 md5）
   # 2) prepare
   curl -s -X POST http://127.0.0.1:1234/api/upload/prepare -H 'Content-Type: application/json' \
        -d '{"path":"/apps/Lugwit/l_wchat/album/test/a.bin","size":17,"block_list":["<md5>"]}'
   # 3) 用返回的 host/uploadid 直传（模拟手机）
   curl -X POST "$HOST/rest/2.0/pcs/superfile2?method=upload&access_token=$AT&type=tmpfile&path=$P&uploadid=$UP&partseq=0" \
        -F "file=@a.bin"
   # 4) finish
   curl -s -X POST http://127.0.0.1:1234/api/upload/finish -H 'Content-Type: application/json' -d '{...}'
   ```
2. **服务器流量验证**：上传前后对比服务器的网卡计数（`netstat -e` / 任务管理器）——只应看到 KB 级变化。
3. **回归矩阵**：8 个上传点（§1.3）逐个过一遍（含失败回退：断开百度域名解析后上传，应自动走回退且成功）。
4. **一致性**：上传后 `decrypt_baidu_md5(上报值) == 本地 md5`（我们有工具 `baidu_md5_probe.py`）。
5. **弱网**：限速/断网重试；`parts` 续传只补缺片。
6. **安全**：抓包确认无明文 token；`prepare` 越权路径被拒（`/etc/passwd`、`../`）。

---

## 10. 风险与回滚

| 风险 | 影响 | 对策 |
|---|---|---|
| CapacitorHttp 不支持 multipart/二进制 | M2/M3 卡住 | 提前做探针；不通过就写 Kotlin 插件（已列清单） |
| 百度上传域名/接口变更 | 直传失败 | 失败即回退服务端代传（G4），业务不受影响 |
| token 泄露 | 整个网盘风险 | 专用应用授权/小号 + HTTPS + 开关默认关 |
| 本地副本缺失导致功能回归 | 缩略图/AI/微信异常 | M2 用"finish 后后台拉回"（§6-B），M4 再切懒回源 |
| 安卓壳改动需要重新打包/分发 | 用户升级成本 | 插件改动独立成一次发版；M1 不依赖插件（纯服务端）可先行 |
| 明文 HTTP 未修就下发 token | 严重安全事故 | **M0 硬前置**，未上 HTTPS 不允许打开 `LUGWIT_UPLOAD_TICKET_TOKEN` |

**回滚**：`LUGWIT_UPLOAD_TICKET_TOKEN=0` + 前端拦截层开关关闭 → 立刻回到现状（服务端中转）。

---

## 11. 工作量估算

| 期 | 服务端 | 客户端 | 合计 |
|---|---|---|---|
| M0 | 0.5d（nginx/证书） | 0.5d（配置+发版） | 1d |
| M1 | 1d | 0 | 1d |
| M2 | 0.5d | 1d（探针+JS） 或 2d（写插件） | 1.5-2.5d |
| M3 | 0.5d | 1.5d（分片+进度+续传+回退） | 2d |
| M4 | 1d | 0.5d | 1.5d |
| **合计** | **3.5d** | **3.5-4.5d** | **7-8 人日** |

---

## 12. 待验证清单（动手前先做）

1. **CapacitorHttp 能否发 multipart/二进制**到 `d.pcs.baidu.com`（决定 M2/M3 走 JS 还是必须写 Kotlin 插件）
2. `superfile2` 是否严格要求 `User-Agent: pan.baidu.com`（我们服务端是带的；客户端要复刻）
3. 百度上传 host（`locateupload` 结果）是否对**移动网络出口 IP** 可用（服务端与手机出口不同 → 需实测）
4. `uploadid` 的有效期与并发限制（多分片并发几路合适）
5. `precreate` 的 `block_list` 上限与顺序要求（超长文件分片数）
6. HTTPS 证书方案（自签 vs 正式）与 WebView `cleartext` 关闭后是否影响其他资源
