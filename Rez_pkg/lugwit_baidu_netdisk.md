# lugwit_baidu_netdisk 使用文档

状态：使用手册，截至 2026-09-17。已实现存储模型见[../网盘版本库Depot设计.md](../网盘版本库Depot设计.md)，计划与 T2 状态见[../网盘版本库Depot演进计划.md](../网盘版本库Depot演进计划.md)，百度 md5 实测事实以[百度云接口元数据实测.md](百度云接口元数据实测.md)为准。本文只说明使用、接口与操作语义，不重复底层实现证明。

> 把百度网盘当**内容寻址 blob 仓**用，在它之上做一套简版 Perforce：
> 提交 / 版本 / 回滚 / 签出 / 待提交列表。
> 改名、移动、回滚**零流量**——网盘上的文件一个字节都不动，只改数据库。

---

## 1. 这个包解决什么问题

跨地区传文件，自建公网带宽太贵。所以：

- **内容**放百度网盘，走网盘自己的 CDN，异地同事直接拉，不吃自建带宽
- **元数据**（谁在哪个版本、哪个 CL、谁签出了）放 Postgres（`chatroom` 库）
- 相同内容全司只存一份（md5 寻址）——注意：**秒传（`return_type=2`）本账号/应用当前命中不了**，
  去重仍靠 depot 的 md5 索引，但"零上行"暂不成立，实测见 §14.4

三块功能，互相独立：

| 功能 | 页面 | 说明 |
|------|------|------|
| **版本库（Depot）** | `/depot` | 主功能。P4 式版本管理 |
| 云盘文件管理 | `/files` | 网盘原始视图，不带版本 |
| 目录推送同步 | `/`（首页） | 本地目录 ↔ 网盘目录，watchdog 实时同步 |

---

## 2. 装与启

### 2.1 rez 包

```python
name = "lugwit_baidu_netdisk"
version = "999.0"
requires = ["python-3.12+<3.13", "pyyaml", "watchdog",
            "fastapi", "uvicorn", "pydantic", "asyncpg"]
```

> `requires` 里**故意不写 `lugwit_auth`**：`lugwit_auth` 反向依赖本包
> （进程内 `mount("/baidu")`），写上会形成 rez 解析环。运行期由
> `lugwit_auth` / `l_scheduler` 的环境带进来。

三个 alias：

| alias | 干什么 |
|-------|--------|
| `baidu_netdisk_web` | 起 Web 服务 |
| `baidu_netdisk_auth` | 命令行走 OAuth 授权 |
| `baidu_netdisk_push_sync` | 命令行跑目录同步 |

### 2.2 两种跑法

**A. 挂在 lugwit_auth 里（推荐，实际部署方式）**

`lugwit_auth` 启动时把本包的 FastAPI app 挂到 `/baidu`：

```
http://127.0.0.1:1027/baidu/depot      版本库
http://127.0.0.1:1027/baidu/files      文件管理
http://127.0.0.1:1027/baidu/           网盘首页
```

页面会被注入 `window.__API_BASE__ = "/baidu"`，前端所有请求自动带前缀。

**B. 单独起（调试用）**

```bat
wuwor lugwit_baidu_netdisk -- baidu_netdisk_web --port 1028
```

| 参数 | 默认 | 说明 |
|------|------|------|
| `--host` | `127.0.0.1` | 监听地址 |
| `--port` | `1028` | 端口 |

单独跑时 `__API_BASE__` 是空串，路径就是 `/depot`、`/api/depot/list`。

### 2.3 登录闸门

**所有页面和接口都要 `lugwit_auth` 登录**：每个 depot HTTP 请求都过 `gate.require_lugwit_token`（**本地验 JWT**，不依赖认证服务在线），token 从 cookie `lugwit_token`
或环境变量 `LUGWIT_ACCESS_TOKEN` 读。

- 页面端点未登录 → 302 跳 `/login?next=原路径`
- API 端点未登录 → 401 `{"detail": "未登录 lugwit_auth"}`
- **「本机免登录」已不再存在**（2026-09-20）：`web_server._auto_local_token` 与
  `_current_user` / `_page_user` 里的回环兜底分支**已删除**；登录态只认
  cookie `lugwit_token` 或 env `LUGWIT_ACCESS_TOKEN`。
  原因：`POST /api/v1/auth/auto` 按 P0 **默认关闭**（需 `LUGWIT_AUTO_AUTH_ENABLED=1`
  且只认"peer 回环 + 无转发头"，生产 nginx 前置下公网请求会被 XFF 挡住）。

> **2026-09-17 实测：本机自动授权静默失败。** `web_server._auto_local_token` → `POST {auth}/api/v1/auth/auto`
> 在本机**实测静默失败**（访问日志只有 401、无任何异常记录）。因此**三处调用方改为自带凭据**，
> 并在 401 时换新 token 重试一次：
>
> | # | 调用方 | 链路 |
> |---|--------|------|
> | 1 | `l_notepad_server/depot_map.py` 的 `http()` | note server → 1028 |
> | 2 | `l_tray/src/l_tray/depot_bridge.py` | 网页 → 托盘 19527 → 1028 |
> | 3 | `lugwit_netdisk_client`（本地桥） | 主窗口登录后 `bridge.setToken()`；另从持久化 WebEngine profile 的 cookie 抓 `lugwit_token` |
>
> **2026-09-20 更新**：三处的取 token 方式已全部改成 **env / 登录**，全仓再无 `/api/v1/auth/auto` 调用：
> `l_notepad_server/depot_map.py` 用 `require_token()`（env 没有就抛错，不再发匿名请求）；
> `l_tray/depot_bridge.py` 优先 **托盘会话 token**（浏览器授权/账号密码登录，refresh 存 DPAPI、可自动续期），
> 其次 env，再次 `LUGWIT_USER`/`LUGWIT_PASSWORD` 登录。详见 `l_tray.md` §3。
>
> token 获取顺序（现状）：托盘会话 token → env `LUGWIT_ACCESS_TOKEN` → 账号密码登录。

### 授权判定（P6：不再「登录即可读任意路径」）

2026-09-20 起，depot 的**读接口**（`/api/v1/depot/download`、`/list`、`/history`）与**写接口**
（`submit`、`submit_stream`、`delete`、`move`、`revert`、`checkout`、`edit_text`、`import`、
`mark_*`、`revert_pending`、`submit_pending`）在取数据/落库**之前**都过 `gate.require_perm()`：

- 判定链：管理员/系统角色本地放行 → 「路径首段 = owner」本机快判 → 跨用户问 auth
  `/authz/check` → auth 不可达时**只认自己的路径**（跨用户一律拒）；
- 越权返回 **403**（不是 404/200 混淆）；`submit_pending` 取不到待提交列表时**拒绝落库**（fail-closed）；
- ACL 支持**目录继承**：在库根发一条授权即可覆盖 `<库>/<子目录>/…`（`authz_service._resource_candidates`）；
- auth 自己的备份任务用 `typ=service` 令牌 + 库根授权（`/l_auth_backup`）通过判定。


登录名就是 depot 里的 `owner`（提交人、锁的归属、have 表的主键之一）。

---

## 3. 首次配置

### 3.1 百度 OAuth 凭证

打开首页 → 设置，或直接 `POST /api/credentials`：

```json
{"client_id": "...", "client_secret": "...",
 "sign_key": "", "redirect_uri": "oob", "app_folder": "Lugwit"}
```

写到 `state_dir()/credentials.yaml`。读回来时 secret 会脱敏成 `abcd****wxyz`；
提交时如果传的是脱敏值，服务端保留旧值不覆盖。

### 3.2 授权换 token

1. `GET /api/auth/url` 拿授权链接 → 浏览器打开 → 百度给一串 code
2. `POST /api/auth/exchange {"code": "..."}` → 换到 access_token 并落盘
3. 过期了 `POST /api/auth/refresh` 用 refresh_token 续

`GET /api/state` 看当前状态：

```json
{"auth": {"token_ok": true, "credentials_ok": true, "token": {...}},
 "sync": {"alive": false, "config": {...}}}
```

### 3.3 版本元数据库

`depot_store.py` 用 asyncpg 连 `chatroom` 库（URL 处理复刻
`lugwit_auth` 的 `credential_service.py`）。表**首次连接时自动建**，
老库升级走 `ALTER TABLE ... ADD COLUMN IF NOT EXISTS`，不用手动迁移。

`GET /api/depot/status` 里 `db: true` 就是通了。`false` 时所有 depot 接口返回
503 `版本元数据库不可用`。

---

## 4. 版本库（Depot）—— 核心功能

### 4.1 概念对照

| 本系统 | Perforce | 说明 |
|--------|----------|------|
| depot 路径 | depot path | 逻辑路径 `/art/char/hero_d.png`，**和网盘物理路径无关** |
| revision | revision | 某路径的第 N 版，指向一个 blob md5 |
| changelist (CL) | changelist | 一次提交，原子 |
| 待提交列表 | pending CL | 改动的暂存区，submit 前不产生版本 |
| have 表 | have | 记录你本地是哪个版本 |
| lock | `p4 edit` 独占 | 别人提交该路径会 409 |
| blob | — | 内容本体，按 md5 存一份 |

### 4.2 网盘上的实际布局

```
/apps/Lugwit/version_depot/blob/<md5前2位>/<md5>              内容，全司去重
/apps/Lugwit/version_depot/dir_mirror/<根>/<逻辑路径>/...      dir 模式：活文件 + .versions/vNNN 快照
/apps/Lugwit/.depot/manifest/<cl÷1000>/<cl>.json              每次提交的清单（兜底）
```

**逻辑路径 → blob 的映射只存在数据库里**。所以：

- 改名 `/a/x.png` → `/b/y.png`：新写一条 revision 指向**同一个 md5**，零流量
- 回滚到 #3：新写一版指向 #3 的 md5，零流量，历史不被抹掉
- 删除：写一条 `action=delete`，blob 保留，老版本照样能下载

manifest 是**兜底**：数据库整个丢了，按 CL 号顺序回放这些 json 就能重建。

### 4.3 两步工作流（和 P4 完全一致）

改动**不会**立刻进版本库，先进待提交区，再整批提交：

```
签出/添加/删除/移动  →  待提交列表堆着  →  submit 才真正产生版本
                     ↘  revert 撤销，什么都不留
```

| 操作 | P4 | 接口 | 效果 |
|------|----|------|------|
| 签出 | `p4 edit` | `POST /api/depot/checkout` | 加锁 + 进待提交区 |
| 添加 | `p4 add` | `POST /api/depot/mark_add_stream` | 内容先传进 blob 仓，条目进待提交区 |
| 删除 | `p4 delete` | `POST /api/depot/mark_delete` | 标记待删 |
| 移动 | `p4 move` | `POST /api/depot/mark_move` | 记一条（含源路径） |
| 撤销 | `p4 revert` | `POST /api/depot/revert_pending` | 移出待提交区 + 解锁 |
| 提交 | `p4 submit` | `POST /api/depot/submit_pending` | 整批原子落库 |

**关键点：submit 之前 `depot_file_rev` 里什么都不写**，所以撤销是真撤销，
不留垃圾版本。已上传但没人引用的 blob 是孤儿，等 GC（尚未实现）。

`submit_pending` 在一个事务里干这些事：

1. `SELECT ... FOR UPDATE` 锁 CL 行，校验 status / owner
2. 展开条目 —— **一条 `move` 展开成两条**（目标 `move` + 源 `delete`）
3. 逐路径查锁，被别人锁住 → 409
4. CL 改 `status='submitted'`
5. 每条算 `max(rev)+1` 写 revision，推进 have 表，清自己的锁
6. 清空该 CL 的待提交条目

### 4.4 一步直提（脚本用）

不想走待提交区的话，还有一套一步接口，调用即产生版本：

| 接口 | 说明 |
|------|------|
| `POST /api/depot/submit` | 提交**服务端本地**文件（body 给绝对路径） |
| `POST /api/depot/submit_stream` | 浏览器/脚本直传，body 是原始字节 |
| `POST /api/depot/delete` | 直接标记删除 |
| `POST /api/depot/move` | 直接移动/重命名 |
| `POST /api/depot/revert` | 回滚到指定版本 |

两套并存，互不干扰。自动化脚本用一步接口更省事，人在界面上操作走两步。

### 4.5 去重的三种命中

提交时统计里会看到：

| 字段 | 含义 | 网络开销 |
|------|------|:---:|
| `dedup` | 数据库里已登记这个 md5 | 零 |
| `rapid` | 网盘 `precreate` 秒传命中（**当前实测不命中**） | 零上行 |
| `uploaded` | 真传了 | 有 |

> 秒传判定：`precreate` 返回 `return_type == 2`（**当前实测不命中**，见 §14.4）。
> 旧代码把返回的空 `block_list` 当成"所有分片都要传"，白传一遍，已修。

### 4.6 大文件

`scan_file()` 单次遍历同时算出总大小、整文件 md5、各分片 md5；
`read_chunk()` 按需 seek 读。10GB 文件也不会进内存。

---

## 5. Depot HTTP 接口

所有路径前面要拼 `__API_BASE__`（挂载时是 `/baidu`）。
出错时响应体是 `{"detail": "..."}`。

### 5.1 查询（GET）

| 接口 | 参数 | 返回 |
|------|------|------|
| `/api/depot/status` | — | `{owner, db, apps_root, depot_root, changes:[最近1条]}` |
| `/api/depot/list` | `dir=/` | `{dir, owner, items:[...]}` 见下 |
| `/api/depot/tree` | `dir=/` | `{dirs:[子目录名]}` |
| `/api/depot/history` | `path=/a/b.png&limit=100` | `{path, revisions:[...]}` |
| `/api/depot/changes` | `limit=50` | `{changes:[...]}` 只含已提交 CL |
| `/api/depot/change/{cl_id}` | — | `{cl_id, files:[...]}` |
| `/api/depot/pending` | — | `{lists:[...], files:[...]}` 你的待提交区 |
| `/api/depot/locks` | `prefix=/` | `{locks:[{path, owner}]}` |
| `/api/depot/sync_plan` | `prefix=/` | `{prefix, plan:[...]}` 相当于 `p4 sync -n` |
| `/api/depot/download` | `path=/a/b.png&rev=0` | 文件字节流，`rev=0` 取最新 |

`/api/depot/list` 的 `items[]` 每项：

```json
{
  "name": "hero_d.png", "path": "/art/char/hero_d.png",
  "isdir": false, "rev": 7, "size": 4194304,
  "action": "edit", "owner": "zhangsan", "created_at": "...",

  "locked_by": "lisi",        // 被谁锁了，空串=没锁
  "pending": "edit",          // 你的待提交动作，空串=没有
  "pending_cl": 42,           // 待提交所在 CL
  "pending_src": "",          // pending=move 时的源路径
  "out_of_date": true,        // 你的 have 版本落后
  "pending_only": false       // true=还没提交过、只存在于待提交区
}
```

界面上的角标就是照这几个字段画的：
`✏` pending=edit ｜ `➕` pending=add ｜ `🗑` pending=delete ｜
`✂` pending=move ｜ `🔒` locked_by 是别人 ｜ `⬇` out_of_date

### 5.2 待提交工作流（POST，JSON body）

```jsonc
POST /api/depot/checkout        {"paths": ["/a/x.png"], "cl_id": null}
POST /api/depot/mark_delete     {"paths": ["/a/x.png"], "cl_id": null}
POST /api/depot/mark_move       {"moves": [{"src":"/a/x.png","dst":"/b/y.png"}]}
POST /api/depot/revert_pending  {"paths": ["/a/x.png"]}
POST /api/depot/submit_pending  {"cl_id": null, "description": "改了主角贴图"}
POST /api/depot/cl_description  {"cl_id": 42, "description": "新说明"}
```

`cl_id` 传 `null` / 不传 = 用你的**默认待提交列表**（每人一个，自动创建）。

添加文件走原始字节：

```bash
curl.exe -X POST "http://127.0.0.1:1027/baidu/api/depot/mark_add_stream?path=/art/x.png" ^
  -H "Content-Type: application/octet-stream" ^
  --data-binary "@D:/local/x.png"
```

### 5.3 一步接口（POST）

```jsonc
POST /api/depot/submit   {"files":[{"local":"D:/x.png","path":"/art/x.png"}],
                          "description":"..."}
POST /api/depot/revert   {"path":"/art/x.png", "rev": 3, "description":""}
POST /api/depot/delete   {"paths":["/art/x.png"], "description":""}
POST /api/depot/move     {"moves":[{"src":"/a/x.png","dst":"/b/y.png"}]}
POST /api/depot/move     {"src_dir":"/a", "dst_dir":"/b"}      // 整目录搬
POST /api/depot/lock     {"path":"/art/x.png", "force": false}
POST /api/depot/unlock   {"path":"/art/x.png"}
```

`submit_stream`（直传即提交）：

```bash
curl.exe -X POST "http://127.0.0.1:1027/baidu/api/depot/submit_stream?path=/art/x.png&description=改贴图" ^
  -H "Content-Type: application/octet-stream" --data-binary "@D:/local/x.png"
```

### 5.4 状态码

| 码 | 含义 |
|----|------|
| 400 | 路径非法（含 `..`、空段、以 `.depot` 开头）或参数不合逻辑 |
| 401 | 没登录 |
| 404 | 版本不存在 |
| 409 | **被别人签出**，`detail` 里有是谁 |
| 410 | 该版本已删除（`action=delete`，没有 blob） |
| 503 | 元数据库连不上 |

### 5.5 路径规则

`normalize_depot_path()` 统一处理：

- 必须以 `/` 开头，反斜杠自动转正斜杠
- 不允许 `..`、空段
- 首段不能是 `.depot`（内部保留）

---

## 6. 版本库界面（`/depot`）

组件和 **Perforce P4V 一一对应**，配色暗色：

```
菜单栏        文件 编辑 搜索 视图 操作 连接 工具 窗口 帮助
大图标工具栏  刷新·获取最新·提交 ∣ 签出·添加·删除·撤销
              ∣ 差异·时间线·版本图 ∣ 移动·下载·锁定·解锁 ∣ 取消
路径栏        可编辑 depot 路径 + ▾最近去过的目录 + 🔖书签
左栏  250px   [Depot | Workspace] + 排序/过滤 + depot 目录树
中栏          [Files | Pending (N) | Submitted]
              7 列表格：☐ 文件 版本 修改时间 大小 类型 状态
右栏  300px   选中文件的版本历史（回滚 / 下载）
底栏  170px   [Log | 历史]  每次 API 调用记一行，可拖高度
状态栏 22px   ● 数据库 | depot 根 | 待同步 N | 最新提交 #N | 👤 owner
```

三套右键菜单：

- **文件行**：签出 / 添加到待提交 / 标记删除 / 移动·重命名 / 撤销 /
  提交所在变更列表 / 获取最新 / 下载 / 查看历史 / 锁定 / 解锁
- **目录行**：进入 / 移动目录 / 在此添加文件 / 刷新
- **树节点**：进入 / 刷新此节点

置灰的按钮（差异 / 时间线 / 版本图 / 取消 / Workspace / 书签）
是后端还没有的功能，不是坏了。

> 操作没反应、不知道哪错了 → **先看底部 Log 面板**，
> 每次请求成功失败都会记一行 `[HH:MM:SS] METHOD 路径 → OK/错误`。

### 6.1 新手教程

第一次打开页面会自动弹出，之后可以从 **帮助 → 🎓 新手教程** 重看。两部分：

**界面漫游**（10 步）
遮罩挖洞高亮当前讲到的区域 + 气泡说明，依次走过菜单栏 → 工具栏 → 路径栏 →
目录树 → 文件列表和状态角标 → 待提交区 → 版本历史 → Log → 状态栏。

| 操作 | 键 |
|------|-----|
| 下一步 | `→` / `Enter` / 点「下一步」 |
| 上一步 | `←` |
| 退出 | `Esc` / 点「跳过」 |

**上手任务清单**（右下角面板）
漫游结束自动弹出，也可以从 **帮助 → ✅ 上手任务清单** 打开。7 个任务：

```
浏览一个目录 → 添加文件到待提交 → 提交 → 签出 → 撤销 → 看历史 → 移动/重命名
```

**必须真做完才打勾**，不是点一下就算。原理是页面在 `api()` 成功回调里派发
`depot-api` 自定义事件，教程模块监听并按 endpoint 匹配：

```js
window.addEventListener("depot-api", function (ev) {
  // ev.detail = { method, path, data }
});
```

进度存 `localStorage`（`depot_tasks_v1` / `depot_tour_seen_v1`），
换浏览器或清缓存会重来。**帮助 → 重置教程进度** 可以手动清零。

面板标题栏可点着折叠；关掉后右下角留一个 `✅ 上手任务` 的小按钮，
全部完成后才不再出现。

页面同时跑在浏览器和 PC 客户端（`lugwit_netdisk_client`，QWebEngine）里。
客户端里额外有 `window.lugwitBridge`（QWebChannel 注入，能算本地文件 md5、
选本地文件）；浏览器里这个对象不存在，页面用特性检测降级。

---

## 7. 云盘文件管理（`/files`）

网盘原始视图，**不带版本管理**。临时文件、参考图放这，不占版本库。

| 接口 | 说明 |
|------|------|
| `GET /api/files/list` | 列目录 |
| `GET /api/files/meta` | 按路径查 fs_id / 大小 / 类型 |
| `POST /api/files/mkdir` | `{dir, name}` 新建文件夹 |
| `POST /api/files/delete` | `{paths:[...]}` |
| `POST /api/files/rename` | `{path, new_name}` |
| `GET /api/files/download` | 服务端代理 dlink 下载 |
| `GET /api/files/stream` | 在线预览（视频/音频/图片/PDF/文本等），inline + Range |
| `GET /api/files/thumb` | 缩略图代理 |
| `POST /api/files/upload` | `{local_path, remote_dir, remote_name, auto_mkdir, overwrite}` 传**服务端本地**文件 |
| `POST /api/files/upload_stream` | `?dir=&name=` + body 原始字节，浏览器直传；响应含 `rapid`（秒传命中＝零上行；**当前实测不命中**，见 §14.4）与 `md5_real` |
| `POST /api/upload/prepare` | **客户端直连百度的第一步**：`{dir, name, size, block_list}` → 秒传探测 / 直传票据（见 §14） |
| `POST /api/upload/finish` | **客户端直连百度的最后一步**：`{dir, name, size, uploadid, block_list, md5}` → create + 复核（见 §14） |
| `POST /api/files/download_local` | 下载到服务端本地目录 |

`overwrite=true` → `rtype=3` 同名覆盖；`false` → `rtype=1` 同名重命名。

**授权失效要能看见（2026-09-18 修）**

百度侧 errno `-6`（鉴权失败）/ `111`（access_token 过期）过去被一律当 **404** 抛（前端只认 401），
且 `GET /api/state` 的 `token_ok` 只看「本地有没有 token 字符串」→ 页面一直显示「已授权」，
用户只看到一个 3.5 秒的 toast，无从判断是证书、路由还是授权。现在：

| 表现 | 现状 |
|------|------|
| 文件类接口遇 errno `-6`/`111` | **401** + `{"detail":"…","code":"baidu_auth_expired","errno":-6}`（前端据 `code` 与「未登录 lugwit_auth」区分） |
| 所有页面的顶栏 | 红色横幅：写清 errno 与 `errmsg`，带「刷新 Token」「去重新授权」按钮 |
| 前端 `api()` | 命中 `code=baidu_auth_expired` → 自动用 refresh_token 续期**一次**并重放原请求；失败才把错误抛给调用方（上传类请求体已发完，不重放） |
| `GET /api/state` | 新增 `auth.token_valid`（True/False/null＝未知）、`token_errno`、`token_msg`；`token_ok` 语义不变（仅表示本地有 token） |
| `GET /api/account` | 不再吞百度报错：失败时 `auth_ok=false` + `auth_errno` + `auth_msg`（`quota_ok`/`quota_msg` 同理） |
| `POST /api/auth/refresh` | 刷新后**真打一次百度**校验，响应带 `valid`；`valid=false` 表示 refresh_token 也废了，须重新授权 |

`token_valid` 的探测结果按 token 缓存 `LUGWIT_NETDISK_TOKEN_PROBE_TTL` 秒（默认 60；设 `0` = 每次真打）。
探测打的是 `uinfo`（纯鉴权接口）：**百度只要回非 0 errno 就判失效**——实测无效 token 返回
`errno=20017`，不在 `-6`/`111` 里，所以文件接口的判据仍是 `-6`/`111`，探测则不看码。
网络异常（exception）判 `null`＝未知，不误报失效。

### 7.1 全类型文件预览

`/files` 页所有文件都有「预览」按钮（原「播放」只支持图片/视频）。分类由
`web_server.py::_media_kind()` 按扩展名判定，列表/相册接口每项带 `media` 字段：

| media | 扩展名 | 预览方式 |
|-------|--------|---------|
| `image` | jpg/png/gif/webp/bmp/avif/svg | `<img>` + 缩略图 |
| `video` | mp4/mov/m4v/webm/mkv/avi/ts/flv | `<video>`（Range 拖进度条）+ 缩略图 |
| `audio` | mp3/flac/m4a/aac/ogg/opus/wav/wma/ape/mka/mid… | 播放器面板 |
| `pdf` | pdf | `<iframe>` 内嵌浏览器 PDF 阅读器 |
| `text` | txt/md/json/yaml/xml/py/js/ts/c/cpp/go/sh/sql/csv/log/srt… 及 LICENSE/README/Makefile 等无扩展名文件 | 阅读器面板：行号列 + 行悬停高亮，长行折行不错位；编码自动检测（BOM→UTF-8 严格校验→回退 GBK/GB18030），顶部工具栏可手动切换编码（UTF-8/GBK/Big5/Shift_JIS/EUC-KR/UTF-16 等）并显示行数/截断信息；超 2 万行截断 |
| `office` | doc/docx/xls/xlsx/ppt/pptx/wps/et/dps | 无法在线渲染 → 提示 + 下载按钮 |
| `file` | 其余（zip/exe/rmvb…） | 同上，提示 + 下载按钮 |

实现要点：

- **独立预览路由**：`/files/view?fs_id=..&name=..&dir=..`（挂载模式
  `http://127.0.0.1:1027/baidu/files/view?...`）直开单个文件预览，可收藏/分享。
  页面内打开预览/左右切换时 `history.pushState` 到该路由，Esc/关闭/浏览器后退
  都能回到列表（`popstate` 双向同步）；直开路由后关闭则 `replaceState` 回
  `/files?dir=..`。旧深链 `?dir=..&open=/path/file` 仍可用。
- **后端不挑类型**：`/api/files/stream` 本来就对任意文件 inline 代理 dlink
  （带 `User-Agent: pan.baidu.com`）+ Range 透传，本次只加了 MIME 兜底表
  `_STREAM_MIME`——Windows 下 `mimetypes` 读注册表常缺 pdf/audio 条目，
  缺了浏览器会把 pdf/mp3 当二进制触发下载而不是预览。
- **前端 viewer 按分类渲染**：`web_files.html` 的 `openViewer()` 分支
  video/image/audio/pdf/text/other；文本用 `fetch` + `TextDecoder(stream)`
  流式读 2MB 后 cancel，不整文件进内存。相册视图非图/视频文件显示分类图标块。
- **深链打开预览**：`/files?dir=<目录>&open=<完整路径>` —— 列目录加载完成后
  自动弹出该文件的预览（按文件名在当前目录匹配）。宝妈笔记文章转存成功后，
  页面底部的链接即指向此深链（`/api/article/save-to-baidu` 返回
  `preview_query` 字段），点开直接是那篇 md 的阅读视图。
- 修改了 `web_files.html`（模块级缓存进 `_FILES_HTML`），**需重启服务**才生效。

### 7.2 预览功能验证记录（2026-09-05）

访问入口（两种部署方式，见 2.2）：

```
挂载 lugwit_auth：http://127.0.0.1:1027/baidu/files     （__API_BASE__ = /baidu）
单独起服务：      http://127.0.0.1:1028/files           （独立 uvicorn，端口 1028）
```

本次验证用的是单独跑法：`wuwor lugwit_baidu_netdisk -- baidu_netdisk_web`，端口 1028。

服务重启后实测（`/media`、`/apps` 三层扫描 + stream 接口探针）：

| media | 文件数 | stream 实测 Content-Type |
|-------|-------:|--------------------------|
| video | 195 | `video/x-matroska`（mkv） |
| file | 89 | `application/zip` |
| image | 78 | — |
| text | 33 | `text/plain; charset=utf-8` |
| audio | 24 | `audio/mpeg` |
| office | 7 | — |
| pdf | 1 | `application/pdf` |

列表分类、页面 200、本机自动授权均正常；大写扩展名（`.RMVB`）正确归入 `file`。

---

## 8. 目录推送同步（首页）

本地目录 ↔ 网盘目录，watchdog 监听实时推。和版本库是**两套东西**，
同步不产生版本记录。

配置写在 `state_dir()/setting.yaml` 的 `baidu_push` 段：

| 字段 | 默认 | 说明 |
|------|------|------|
| `local_root` | — | **必填**，本地根目录 |
| `remote_subpath` | `lugwit_push` | 网盘上的子目录 |
| `enabled` | `false` | 总开关 |
| `bidirectional` | `false` | 双向（云端改动也拉回来） |
| `debounce_seconds` | `1.5` | 文件改完等多久再传，防抖 |
| `initial_push` | `true` | 启动时先全量推一遍 |
| `poll_interval_seconds` | `120` | 拉云端改动的轮询间隔 |
| `propagate_delete` | `false` | 本地删了要不要删云端 |
| `delete_confirm_delay_seconds` | `3` | 删除确认延迟，防误删 |
| `ignore_suffixes` | `.tmp,.swp,.log,...` | 忽略后缀 |
| `ignore_dirs` | `.git,__pycache__,node_modules,...` | 忽略目录 |

接口：

| 接口 | 说明 |
|------|------|
| `PUT /api/sync/config` | 保存配置 |
| `POST /api/sync/start` | 起子进程 |
| `POST /api/sync/stop` | 停 |
| `POST /api/sync/restart` | 重启 |
| `GET /api/sync/log` | 日志尾巴（默认最后 120KB） |
| `POST /api/sync/verify` | 对比统计，不写盘 |
| `POST /api/sync/test_upload` | 传一个测试文件验证链路 |

> 模块头注释里写的 `GET /api/sync/pull` **实际不存在**，
> `verify` 也是 POST 不是 GET，那段注释是过时的。

同步跑在**独立子进程**里（`python -m lugwit_baidu_netdisk.baidu_netdisk_push_sync run --config ...`），
pid 记在 `state_dir()/.baidu_web_sync.pid`，日志 `state_dir()/baidu_push_sync.log`。

命令行直接跑也行：

```bat
wuwor lugwit_baidu_netdisk -- baidu_netdisk_push_sync run --config 路径/setting.yaml
```

---

## 9. 自动化测试

```
tests/test_depot_api.py       12 段，跑 depot 全部 HTTP 接口
tests/test_depot_pending.py   10 段，跑两步工作流（57 断言）
tests/test_upload_direct_server_side.py  服务端自证直传链路（真百度：precreate→分片→create→复核）
tests/test_upload_direct_client.py       外部客户端全链路（登录→prepare→分片直传→finish）
tests/purge_autotest.py       清理测试数据（直连数据库）
```

跑法（要求服务已启动）。这两个套件只用标准库 `urllib`，
随便哪个 3.12 解释器都能跑：

```bat
python tests\test_depot_api.py
python tests\test_depot_pending.py
```

| 参数 | 说明 |
|------|------|
| `--base` | 服务地址，默认 `http://127.0.0.1:1027/baidu` |
| `--keep` | 跳过收尾清理，留着数据人工看 |

- 测试根目录带时间戳（`/_autotest/<MMDD_HHMMSS>`），可重复跑
- 输出末行 `RESULT PASS` / `RESULT FAIL`

清理累积的测试数据。这个要**直连数据库**（需要 `asyncpg`），
所以必须在 rez 环境里跑：

```bat
wuwor lugwit_baidu_netdisk -- python tests\purge_autotest.py         只预览，不动数据
wuwor lugwit_baidu_netdisk -- python tests\purge_autotest.py --yes   真删
```

| 参数 | 说明 |
|------|------|
| `--prefix` | 要清的前缀，可重复给。默认 `/_autotest`、`/_autotest_gui`、`/_movetest`、`/_movetest_moved` |
| `--yes` | 确认执行，不加只打印将要删什么 |

删 `depot_file_rev` / `depot_have` / `depot_lock` 里的记录和变空的 CL，
**blob 不删**（可能被别的路径引用）。

界面侧的测试在 `lugwit_netdisk_client/999.0/tests/`（`gui_suite.py` +
`p4v_suite.py`），通过客户端的 8764 端口注入进程内执行。

---

## 10. 排障

**顶栏红条「百度授权已失效（errno=-6）」/ 文件页提示 `读取目录失败 errno=-6`**
errno `-6` = 鉴权失败、`111` = access_token 过期（判据同 `baidu_netdisk_push_sync`）。
本地 token 文件在、百度不认，就是这种情况：先点横幅「刷新 Token」；仍失效说明 refresh_token 也废了
→ 按 §3.2 重新授权。**不是**证书/路由问题：证书问题表现在 TLS 握手（`ERR_CERT_*` / 000），
路由问题的 404 不带百度 errno。旧版本（2026-09-18 前）会把这类失败报成 404 且只弹 3.5 秒 toast，
排查时容易误判——现已统一走 401 + `code=baidu_auth_expired`。

**`db: false` / 503 版本元数据库不可用**
Postgres 连不上。检查 `chatroom` 库连接串（和 `lugwit_auth` 用同一套）。

**401 未登录 lugwit_auth**
非本机请求必须带 cookie `lugwit_token` 或环境变量 `LUGWIT_ACCESS_TOKEN`。
本机请求会自动换 token —— **但本机自动授权 2026-09-17 实测静默失败**（见 §2.3），
所以三处调用方（note server / 托盘 `depot_bridge` / 客户端本地桥）改为**自带凭据 + 401 换新 token 重试一次**；
若仍 401，先查调用方 token 来源（托盘会话 token / env `LUGWIT_ACCESS_TOKEN` / 账号密码登录 ——
`/api/v1/auth/auto` 已默认关、相关调用已全清），再看 `lugwit_auth` 是否起来。

**列表能通、下载 500 / 经 nginx 502（内容取不到）**
depot **元数据在 PostgreSQL**（`/api/depot/list`、`/api/kb/{kb}/depot/list` 离线可用、返回 200），
**文件内容在百度网盘**（`pan.baidu.com`）。机器连不上外网时取内容就报 **500 / 经 nginx 502**，**与代码无关**。
实测：`curl https://pan.baidu.com` 返回 `000` 时，列表 200、下载 500。

**409**
路径被别人签出了，`detail` 里写了是谁。等对方提交/撤销，
或让管理员 `POST /api/depot/unlock`。

**410 该版本已删除**
你下载的那一版 `action=delete`，本来就没有内容。下更早的版本。

**页面改了不生效**
`web_server.py` 模块级缓存了 HTML（`_DEPOT_HTML = ...read_text()`），
必须重启服务进程。开发时 uvicorn 的 `--reload` 只盯 `lugwit_auth` 目录，
改本包不触发，可以碰一下 `lugwit_auth/auth_server.py` 强制重载：

```bat
copy /b auth_server.py+,, auth_server.py
```

**上传很慢 / 明明是重复文件还在传**
看返回的 `stats`。如果 `uploaded` 不为 0 而你确信内容重复，
检查 md5 是否真的一致（改过一个字节就是新 blob）。
另：**秒传当前命中不了**（见 §14.4 实测），所以"重复内容零上行"这条现在不成立。

**客户端直传：`ticket_enabled=false` / 拿不到 `access_token`**
服务端没开 `LUGWIT_UPLOAD_TICKET_TOKEN`（默认关）。要直传就设成 `1` 并**重启 1028**
（该变量是启动时读的）。没开时前端会自动回退服务端代传，功能不受影响。

**客户端直传：403「直传票据只通过 HTTPS 下发」**
请求是明文入口（如 `http://<ip>:1234`）。票据＝账号级百度凭据，明文过公网等于交出网盘。
走 HTTPS 入口（`https://<ip>/l_wchat/…`）；本机回环直连（无 `X-Forwarded-For`）不拦。

**客户端直传：`md5_match=false` 被拒（502）**
只可能出现在**单分片**（≤4MB）文件上。多分片时百度报的 md5 与整文件 md5 不是一回事，
`md5_match` 返回 `null`（不可比），服务端只比 `size`（见 §14.3）。

**客户端直传：分片上传返回 `errno=-999` / 分片 md5 对不上**
先看响应体——**成功时可能没有 `errno` 字段**（只有 `md5`+`request_id`），
把"缺 errno"当成失败是误判；判失败请用 `errno != 0`，并核对回的分片 md5。

**数据库丢了**
按 CL 号顺序回放 `/apps/Lugwit/.depot/manifest/` 下的 json 即可重建
revision 表。blob 本身在网盘上，没丢。

---

## 11. 明确不做的事

| 不做 | 理由 |
|------|------|
| 局域网直连 / WireGuard / Headscale | 组网运维成本 > 收益，网盘 CDN 已够用 |
| 抽象 `Transport` 层适配多种网盘 | 只有一个百度网盘，抽象是负收益 |
| blob GC | 孤儿 blob 目前靠人工，量小不值得做 |
| 文本 diff / 三方合并 | 目标是二进制美术资产，合并没意义，用锁 |
| 多分支 / stream | 需求没出现 |

---

## 12. 在知识库工作区中使用（l_notepad 集成）

知识库（`l_notepad_server`）新增了**工作区**功能：把知识库绑定一个本地目录，
在网页上预览目录里的 `.md` 文档，并一键提交到本包的 Depot 版本库（百度云）。

### 12.1 配置工作区

每个知识库可设置一个工作区目录（`knowledge_bases.workspace`）：

```
PUT /note/api/kb/{kb_name}/workspace
{"workspace": "D:\\...\\Rez-Docs"}
```

例如 `rez_pkg` 知识库的工作区指向 `rez-package-source\Rez-Docs`。

### 12.2 工作区接口（l_notepad）

| 接口 | 说明 |
|------|------|
| `GET /api/kb/{kb}/workspace` | 列出工作区内可预览文本（.md/.txt/.rst/.log，含相对路径/大小） |
| `GET /api/kb/{kb}/workspace/file?path=<rel>` | 读取单个文档内容（预览） |
| `PUT /api/kb/{kb}/workspace` | 设置工作区目录 |

> 路径做了防穿越校验，相对路径只能落在工作区目录内。

### 12.3 提交到百度云 Depot

知识库「工作区」页面对每个文档提供「⬆ 提交到百度云」，即调用本包的一步直提接口：

```
POST /baidu/api/depot/submit_stream?path=/notes/<kb_name>/<相对路径>&description=提交 <相对路径>
Content-Type: application/octet-stream
Body: 文件原始字节
```

**Depot 逻辑路径约定（2026-09 修正）**：`{library}/{subpath}/{工作区内相对路径}` ——
**库 = 逻辑路径首段**，知识库只是库下的**子路径**。笔记/知识库统一落在库 `/notes`：

```
/notes/rez_pkg/Rez_pkg/l_script_editor.md      # 知识库 rez_pkg → 子路径 rez_pkg
/notes/xxx.md                                   # 个人笔记同步（cloud_sync，depot_library=/notes）
```

映射关系由 `l_notepad_server/depot_map.py` 维护：`knowledge_bases.depot_library`（默认 `/notes`）、
`depot_subpath`（默认 = 知识库名）、`depot_ws`（默认 `kb-<知识库名>`）；每个知识库对应一个 depot
工作区，并登记 `maps=[{depot_path: "/notes/<kb>", local_path: ""}]`，使本地 `xxx.md`
↔ `/notes/<kb>/xxx.md` 一一对应。多知识库之间靠子路径隔离。

> 历史：早期实现把**知识库名当成库**（`/rez_pkg/xxx.md`），与笔记同步用的库 `/notes` 不一致；
> 2026-09-16 已改为上述"库 + 子路径"模型，旧库 `/rez_pkg` 下的文件已 move 到 `/notes/rez_pkg/`。

提交即产生版本（rev），可走 Depot 的查询/回滚/签出等全套 P4 语义。

浏览器经 nginx `/note` 页面加载文档、经 l_notepad 后端 `/api/kb/{kb}/depot/*` 提交，复用 lugwit_auth 登录态即可。

---

## 13. 存储多模式（blob / 目录镜像）

Depot 支持**按逻辑根**（如 `/notes`）切换物理存储模式，各库可各取所需：

| 模式 | 物理存储 | 适用 |
|------|---------|------|
| `blob`（默认） | 内容寻址 blob（`version_depot/blob/<md5>`），去重 | 大文件 / 多版本 / 二进制 |
| `dir` | 按目录结构镜像 + `.versions/vNNN` 版本夹，**人可在百度云客户端浏览** | 小文本笔记 / 文档 |

### 13.1 登记模式

```http
GET  /baidu/api/depot/modes            # 列出所有根的当前模式
PUT  /baidu/api/depot/mode             # 登记某根的模式
Content-Type: application/json
{"root": "/notes", "mode": "dir"}
```

模式存于 `depot_mode` 表（root, mode），未登记默认 `blob`。

### 13.2 `dir` 模式物理布局

逻辑路径 `/notes/rez_pkg/Rez_pkg/l_script_editor.md` 对应：

```
/apps/Lugwit/version_depot/dir_mirror/notes/rez_pkg/Rez_pkg/l_script_editor.md                 ← 最新版（活文件，可直接下载）
/apps/Lugwit/version_depot/dir_mirror/notes/rez_pkg/Rez_pkg/.versions/l_script_editor.md/
  v001/l_script_editor.md                                        ← 历史快照
  v002/l_script_editor.md
```

- 最新版同时写到"活文件"真实路径，百度云客户端直接可见/可下载
- 历史版本放同目录 `.versions/<文件名>/vNNN/` 快照（同内容本应秒传不额外占空间；**秒传当前实测不命中，见 §14.4**，所以会按真文件各占一份）
- 版本元数据仍由 `depot_file_rev` 表记录（rev/history/回滚语义不变）
- 提交时预取下一版号 `next_rev = head.rev + 1`，上传到活文件 + vNNN，再以显式 rev 落库

### 13.3 下载兼容

`GET /baidu/api/depot/download?path&rev`：
- `dir` 模式：优先取 `vNNN` 快照，缺失（如早期 blob 迁移数据）退回活文件
- `blob` 模式：按原逻辑走内容寻址 blob

### 13.4 列表附带物理路径

`GET /api/depot/list` 对每个文件项附加：

| 字段 | 说明 |
|------|------|
| `mode` | 该文件所在根的存储模式（`blob` / `dir`） |
| `remote_path` | `dir` 模式下活文件的真实物理路径（`/apps/<应用>/version_depot/dir_mirror/<逻辑路径>`），`blob` 模式为空串 |

前端右键菜单据此在 `dir` 模式下显示/复制「百度云物理路径」（人可定位），`blob` 模式仍显示逻辑路径。

### 13.5 迁移（blob → dir）

根切到 `dir` 模式后，**早期用 blob 提交的文件**仍存在 `version_depot/blob/<md5>` 里、`dir` 模式下载会 404。
用迁移接口把每个存活文件的最新内容从 blob 取回，按 `dir` 模式重新提交（写活文件 + vNNN 快照）：

```http
POST /api/depot/migrate
Content-Type: application/json
{"root": "/notes", "dry_run": true}    # dry_run 只预览不执行（库 = 逻辑路径首段）
```

实现：`depot_service.materialize_dir_root()`（遍历 `store.sub_paths` → `_download_blob_to_tmp` 下载 blob → `submit_files` 走 dir 分支重写活文件 + vNNN 快照并落库）。

返回：

```json
{"migrated": ["/notes/rez_pkg/xxx.md"], "failed": [{"path": "...", "error": "..."}],
 "migrated_count": 1, "failed_count": 0}
```

- 迁移后 rev 会 +1（新提交），历史 rev 仍由 DB 保留
- 已在 `dir` 模式直传的文件没有 blob，迁移时计入 `failed`（`blob 文件不存在`）属正常，可忽略

### 13.6 注意事项

- `dir` 模式的删除/移动只改 DB 元数据，**不会物理删/移百度云文件**（活文件与历史快照保留，仅不再出现在列表）
- 切换模式不影响已提交的历史（rev 元数据在 DB 里），但新旧提交的物理存放位置不同
- 切换前请确认该根下的提交均可用（`blob`→`dir` 后，旧 blob 内容下载仍走回退逻辑，或用 13.5 迁移）

---

## 14. 客户端直传百度（prepare / finish）

**用途**：让**客户端（手机 App / 外部脚本）把文件字节直接送到百度**，服务器只参与几 KB 的
元数据协商（不再当"上传中转站"）—— 目的是省服务器公网流量。

```
客户端 ──① POST /api/upload/prepare──► 1028 ──► 百度 precreate（只报 size+分片 md5）
        ◄── uploadid / upload_host / parts（可选带 access_token）──┘
客户端 ──② POST 分片到 upload_host/rest/2.0/pcs/superfile2?…──► 百度   ★ 字节走这里，不经服务器
客户端 ──③ POST /api/upload/finish───► 1028 ──► 百度 create + 复核 → fs_id / size / md5
```

### 14.1 两个端点

| 端点 | 入参 | 出参 |
|------|------|------|
| `POST /api/upload/prepare` | `{dir, name, size, block_list[], overwrite=true}` | 命中秒传：`{rapid:true, fs_id, path, size, md5_real}`；未命中：`{rapid:false, uploadid, upload_host, parts[], block_size, ticket_enabled[, access_token]}` |
| `POST /api/upload/finish` | `{dir, name, size, uploadid, block_list[], md5="", overwrite=true}` | `{ok:true, fs_id, path, size, size_match, md5_real, md5_match, md5_comparable}` |

- `block_list` = 按 4MB 分片、顺序即 `partseq` 的分片 **md5**（客户端自己算）
- `upload_host` 由 `locateupload` 实时给出，**不要缓存**
- 两条都要求登录（cookie `lugwit_token`），路径必须落在白名单内

### 14.2 环境变量

| 变量 | 默认 | 说明 |
|------|------|------|
| `LUGWIT_UPLOAD_TICKET_TOKEN` | **空＝关** | 关时 `prepare` **不下发** `access_token` → 客户端拿不到百度凭据、只能回退"服务器代传"（现状）。开启才会下发（必须已是 HTTPS 入口） |
| `LUGWIT_UPLOAD_ROOTS` | `/apps/Lugwit/l_wchat` | 直传可写根白名单（`os.pathsep` 分隔）。不限定范围等于开放"往网盘任意路径写" |

### 14.3 内容复核规则（实测结论，别想当然）

| 情况 | 实测 | 处理 |
|------|------|------|
| 单片（≤4MB） | 百度回的是**加密 md5**，`decrypt_baidu_md5()` 解回来**正好等于**整文件 md5（`create` 与 `file_metas` 一致） | `md5_match` 单片为 True/False，可硬校验 |
| **多分片（>4MB）** | 百度报的 md5 **不等于**整文件 md5（5MB+ 例：本地 `513641b7…`，百度解出 `d310b6f4…`；不是 slice-md5、也不是块 md5 拼接） | `md5_match=null`、`md5_comparable=false`；**只比 `size`**（各分片 md5 由调用方在直传时逐个核对） |
| 分片上传响应 | 成功体可能**不带 `errno`**（只有 `md5` + `request_id`） | 判失败只能用 `errno != 0`；并额外核对回的分片 md5 与本地一致 |

### 14.4 秒传（`return_type=2`）现状：本应用命中不了

2026-09-16 三次实测：① 同路径上传成功后 1s / 10s 再 `precreate` → 都是 `return_type=1`；
② 同内容**不同路径**再 `precreate` → 也是 1；③ 百度专用 `method=rapidupload`（`content-md5`
+`slice-md5`、加 `rtype`、用 `block_list` 三种写法）**全被 `31023 param error` 拒**。
⇒ 本账号/应用当前没有秒传能力：`prepare` 的 `rapid=true` 属于"代码路径在、能力不在"，
`depot_service` 里 `rapid` 计数长期为 0 也属正常。等百度侧开通后用
`tests/test_upload_direct_server_side.py --expect-rapid` 复查。

### 14.5 安全边界（必须知道）

- 下发的 `access_token` 是**账号级**的（约 30 天有效、**无法单独撤销**，只能撤销整个应用授权）。
  所以 `prepare` 的闸门 = **登录 + HTTPS**：明文入口请求一律 403
  （`l_WChat` 侧实现；本机回环直连且无 `X-Forwarded-For` 时视为安全，方便本机测试）。
- 回滚：删掉/置 0 `LUGWIT_UPLOAD_TICKET_TOKEN` → 客户端拿不到凭据 → 全部走服务器代传。
- 想要"客户端不持有账号级凭据"，只能：① 客户端用自己的百度账号走 OAuth 授权；或
  ② 专用小号 + 独立应用授权。百度**没有** STS/一次性上传凭据（`superfile2` 必须带 `access_token`）。

### 14.6 自动化测试

```bat
:: 服务端自证（真百度、不需开票据开关）：precreate → superfile2 → create → 复核
wuwor lugwit_baidu_netdisk -- python tests\test_upload_direct_server_side.py --apps /apps/Lugwit

:: 外部客户端全链路（登录 → prepare → 本机出口直传百度 → finish → 删）：需服务端已开票据开关
python tests\test_upload_direct_client.py --host https://121.196.144.88
```

| 参数 | 说明 |
|------|------|
| `--size` | 文件大小，默认 1MB（`test_upload_direct_server_side.py` 默认 5MB+12345，跨 2 片） |
| `--dir` | 测试目录（默认 `l_wchat/_probe_direct`，不进相册索引） |
| `--keep` | 保留测试文件（默认删） |
| `--expect-rapid` | 要求二次 prepare 命中秒传（当前不满足，只有百度开通后才该加） |
| `--user/--password` | 客户端测试用；不给时读 WChat 的 `config.json` |

前端 JS（`l_WChat/999.0/src/l_WChat/static/upload_direct.js`）的本机验证：
`l_WChat/999.0/tests/test_direct_upload_local.mjs`（Node 跑真实 JS + 真发百度），
或用 Playwright 打开本机相册页（直传计划、验收状态与现行限制见[../网盘版本库Depot演进计划.md](../网盘版本库Depot演进计划.md)的 T2）。

### 14.7 已知限制

- 生产可用的前提是客户端有**原生 HTTP 通道**：浏览器 `fetch` 打 `*.pcs.baidu.com` 会被 CORS 拦，
  且 `User-Agent` 在浏览器里是禁止头（JS 设不了，百度又要求 `pan.baidu.com`）。
  JS 侧已固化优先级：`window.LwDirectUpload.uploadPart(url, headers, base64) → {status, text}`
  → `CapacitorHttp` → 回退服务器代传。
- `prepare` 只覆盖 `LUGWIT_UPLOAD_ROOTS` 白名单内的路径。

---

## 15. 客户端与本地桥（`lugwit_netdisk_client`，2026-09-17）

PC 客户端（PySide6 + QWebEngineView）本次新增：

- **启动登录窗**：注入 `LoginStore(data_dir=~/.Lugwit/lugwit_netdisk_client)`（库自带标题栏登录按钮 +
  启动静默恢复）；**没有任何已保存 token 时，启动 400ms 后自动弹登录对话框**
- **登录成功**：把 token ① 注入 WebEngine cookie（域名/路径按 `base_url` 设）② `bridge.setToken(token)`，然后 reload；
  **登出**清 cookie + 清桥 token
- **本地桥新增 4 个 QWebChannel 槽**（返回 JSON 字符串，**本地进程内直连 depot 服务**，基址
  `L_CLIENT_DEPOT_URL` > `L_DEPOT_SERVICE_URL` > `http://127.0.0.1:1028`）：
  `depotBase` / `depotList` / `depotDownload` / `depotVersions`
- 与托盘的 `depot_bridge`（见 §2.3 表 #2）是**同一套语义**（谁在谁服务），两处都做路径校验
  （绝对路径、无 `..`、单文件 8MB 上限、文本类扩展名回文本否则 base64）

页面里 `window.lugwitBridge` 存在时优先走本地桥（如知识库页归档内容读取，见《Rez_pkg/l_notepad_server.md》§7.1），
浏览器里该对象不存在则降级到托盘中转 / `/baidu/api/depot/download` 直连。桥内自动带 lugwit 登录态。



