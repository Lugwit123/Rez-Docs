# l_agent_chat 改造记录（会话存云与工作区 · 主循环加固）

> **合并说明**：本文由两篇原本「给接手者」的临时交接**合并而成**，去重后按「实现记录 / 待办与下一步」组织，并按主题分两部分。
> - 《`l_agent_chat会话存云与工作区.md`》——**会话改为「云为真源」的 P4 式改造**：已完成部分（全量回归 + 浏览器实测）+ **确切下一步** + 服务端 P4 API 全表；§13 / §14 已实现。→ **并入「第一部分」（§1~§15，编号保持原样，供其他文档对 `§4` / `§14` 的引用继续有效）**。
> - 《`Rez_pkg/l_agent_chat_交接_已修与待办.md`》——自述「**历史**」，含 `D:\TD_Depot\...` 老路径，仅作追溯。→ **并入「第二部分」（§16~§23）**。
>
> - **涉及包**：`l_agent_chat`（对话 agent，:1250）、`l_notepad_server`（知识库，:8765）、`l_agent_tool`（工具库）、`l_repo_sync_gui`（直推部署通道）。
> - **状态截至**：第一部分（会话存云）**2026-09-21 起，最新一节 §15 为 2026-09-26**；第二部分（主循环加固）**2026-09-29**。
> - ⚠️ **路径口径（重要）**：本机（开发机）仓库**已迁到 `E:\lugwit\trayapp`**（`rez-package-source/` 在其下）；远端（公网部署机）仍是 `D:\TD_Depot\Software\Lugwit_syncPlug\lugwit_insapp\trayapp`。
>   原文里的 `D:\TD_Depot\...\trayapp\rez-package-source\` 是**仓库搬到 E: 之前的当时环境（已失效）**，已在原处加注；而**远端工作区 `D:/TD_Depot/l_agent_ws/...` 仍是远端（D: 机器）的现行路径**。老路径处置汇总见 **§24**。

## 速览（该看哪一节）

| 想找什么 | 去 |
|---|---|
| **别重复踩的三条陷阱（先读）** | §17 |
| 会话存云：目标 / 模型 | §1 |
| 会话存云：已完成清单 | 第一部分「进度清单 · 已完成」+ §2 |
| 会话存云：**确切下一步（剩余 6 项待办）** | §11 |
| 工作区登记（卡点已解除） / 关键实测 | §3 / §4 |
| 状态三方比对 / 提交拉取只碰「稳的那一头」 | §5 / §6 |
| 测试 / SSE 进度 / P4 其余操作 / 逐条入口 | §7 / §8 / §9 / §10 |
| 状态自刷新与配置优先级 | §12 |
| 云端会话在侧栏可见、按需拉取（已实现） | §13 |
| `⚠ 云端内容缺失` 服务端去重缺陷（已修） | §14 |
| 工作区本体 `.code-workspace` | §15 |
| 主循环加固：目标与验收（G1~G8） | §16 |
| 主循环加固：一句话现状 | §18 |
| 主循环加固：已修 + 判据 | §19（知识库 §19.2 / 直推部署 §19.3） |
| 主循环加固：已验 / 未验 | §20 |
| 主循环加固：待办（✅ 全部办结，原文对照） | §21 |
| 复现工具与判据 / 红线 | §22 / §23 |
| 老路径（`D:`）处置汇总 | §24 |

---

# 第一部分　会话存云与工作区（云为真源 · P4 式）

> 源自《`l_agent_chat会话存云与工作区.md`》。**编号保持原样（§1~§15）**，以便其他文档对 `§4` / `§14` 的引用继续有效。

> **性质**：改造交接。已完成的部分已逐项验证（全量回归 + 浏览器实测），未完成的部分写明了确切下一步。
> 状态截至 2026-09-21（最新一节 §15 为 2026-09-26）。相关设计参照 `Depot库与工作区方案.md`（标注为「未实施/待评审」）与已实现的 `l_notepad` 那套。
> 本文的「实测」结论均以 `127.0.0.1:8080/baidu` 的真实服务为准，**推翻了早期几处按通用 P4 知识做的假设**，
> 也已订正本文自身写错过的两条（见 §4 末尾「订正」）。

## 进度清单（TODO）

图例：`[x]` 已完成并验证 ／ `[ ]` 未完成。明细见下面的编号小节（**未完成项集中在 §11**）。

### 已完成

- [x] **四态状态纯计算**（☁ 仅云端 / 📄 仅本地 / ✎ 已修改 / ✓ 已同步）
      → `depot_status.py`；`compute_status()` / `from_plan()` / `summarize()` / `pending_actions()`
- [x] **云端侧依据改用 `cloud_manifest()`**（`GET /api/depot/list?dir=` 递归）
      —— head 上真正存在的文件 + rev/大小/锁。**「☁ 仅云端」的墓碑残留问题就此消失**
      （`have` 里 `rev=<墓碑>` 的路径不再被当成云端文件）
- [x] **工作区已登记**：`ensure_workspace()`（幂等）+ `POST /api/depot/workspace-ensure`
      + 设置页「📌 登记工作区」按钮。实测 `ws_registered: true`
- [x] **`maps` 形状抄到了**：`[{depot_path, local_path, exclude}]`，库根 ↔ local_root 根
- [x] **depot 路径平铺**：`<库>/<rel>`，不带 `<user>` 段（由 `sync_plan` 实测判定）
- [x] **`local_root` = 存储根下的 `.l_agent_ws`**（不是 agent 的活动根）
- [x] **工作区本体换成 `.code-workspace`**（2026-09-26）：agent 自己的工作区 = 一份与 VS Code 同格式的
      `folders` 列表（`folders[0]` = 基准）；旧的 `<锚点>/.l_agent_ws/config.json`（`roots`/`active_root`）
      不再读写、不做迁移，运行期偏好进 `settings.json` 的内部键 —— 详见 §15
- [x] **提交只推变化项**（待推清单与「N 项待提交」同源）+ `force` 全量重推
- [x] **提交进度（SSE）**：`GET /api/depot/push/stream`，逐文件一帧；实测帧实时到达
- [x] **拉取只拉需要拉的 + 进度（SSE）**：`GET /api/depot/pull/stream`；`force` 全量拉
- [x] **拉取后 `sync_done` 盖章**（不盖则服务端 `have` 与本地对不上）
- [x] **删除会话时一并删云端**：`session_store.delete_session` → `depot_sync.delete_session_async`
      → `POST /api/depot/delete`（不删则云端那份永远挂着「☁ 仅云端」）
- [x] **P4 其余操作接线**：`pending()` / `locks()` / `checkout()` / `unlock()` / `delete()` /
      `submit_pending()` / `revert_pending()` / `reconcile()` + 对应 `POST /api/depot/*` 端点
- [x] **逐条操作入口（界面）**：
      - 侧栏每条会话 hover 出 `⬆ 提交这条` / `⬇ 拉取这条`（按 `can_push` / `can_pull` 显隐，
        同步态**不出现**，不打扰纯本地用法）
      - 设置页「工作区条目」表：每条一行（状态 + 锁归属）带 `⬆ 提交` `⬇ 拉取`
        `🔒 检出` / `🔓 解锁` `🗑 删云端`。**解锁 = 解锁 + 撤回变更单**（只解锁不撤回，
        变更单里会一直挂着「已打开但没动」的记录）
      - 对应单条端点：`/api/depot/push-one` / `pull-one`（只有本包管的三类 rel 被接受，
        设置照样脱敏）；`checkout` / `unlock` / `delete` / `revert-pending` 复用
      - 实测：检出 `settings.json` → 锁出现（`🔒 admin01/l_agent_chat`）+ 变更单 1 项 →
        解锁 → 全部复原
- [x] **状态判定改成三方比对 + 同步台账**（见 §5）：本地指纹 / 云端清单 / `depot_ledger.json`。
      起因是订正了 `sync_plan` 的语义 —— 它是「云 → 本地」的**下载**计划，不是上传计划；
      拿它当「本地改了什么」既**漏报**（本地改动服务端看不见，要 `reconcile` 上报）
      又**误报**（`have` 落后 ≠ 本地有改动）。订正后实测发现旧逻辑把 3 个真不一致的文件
      报成了「7/7 已同步」
- [x] **提交 / 拉取只碰「稳的那一头」**：批量提交只推 `local_only` + `broken`，
      批量拉取只拉 `cloud_only`；`✎ 已修改`（不一致）**批量一律不碰**，只走逐条 —— 这才叫
      「不替用户选边」（见 §6）
- [x] **新增「⚠ 云端内容缺失」态**：云端有元数据但 `download` 404（blob 没落到网盘）。
      单列一态并**只给「可提交」**（重传是唯一修法，拉取必定 404）。
      ⚠ **后半句当时是错的**：服务端去重没验存 → 死登记让重传空转（rev 涨、blob 不写）。
      2026-09-22 已在服务端修（提交前验存），详见 §14
- [x] **变更单与锁可见**：`workspace-status` 带 `pending`/`locks`；设置页状态行报
      「变更单未提交 N 项 · 🔒 已锁 M 个」；会话徽标被锁时显示 🔒
- [x] **状态自刷新**：30s 定时 + `visibilitychange`（切回本页立刻重取）
- [x] **配置优先级可见**：`_v()` 记录命中环境变量的键 → `get_runtime()["env_overrides"]`
      → 设置页给这些字段加 ⚠ 提示（**不改优先级契约**，见 §12）
- [x] **测试**：全量 **158 例**（`wuwor l_agent_chat -- l_agent_chat_test`）
- [x] **浏览器实测**：徽标随状态变化、提交进度逐条推进、登记工作区幂等（仍是 #68）、
      设置页状态行、逐条表的检出/解锁真点过、**侧栏逐条 ⬆ 真点过**
      （2 个 `⚠` 条目显示 `⚠⬆`，点后页脚给出「提交这条成功」）、
      **逐条 ⬆ 解掉不一致**（3 项朝本地推，云端 rev 9→10 / 5→6 / 5→6）、台账文件确实生成
- [x] **文档归档**：本文 + `README.md` 索引登记
- [x] **云端会话在侧栏可见、按需拉取**（change `cloud-session-lazy-pull`）：新增只读端点
      `GET /api/session/cloud` —— 只列「云端有、本机没有」的会话元数据（id / rel / rev / 大小 / 锁归属），
      **一条都不下载**；侧栏把它们作为虚拟条目混排（`☁ 09-20 18:44 · 未拉取`，按 id 时间倒序，
      与本地同序），点这条才 `POST /api/depot/pull-one` 拉这一条并切过去；拉取失败就地报原因、
      条目留在云端态。云端拿不到时退回纯本地列表、不报错。**语义未变**：云 → 本地仍只走显式拉取，
      批量拉取仍只碰 `cloud_only`。刻意不走 `status_items()`（那会逐条 download 比对 + 写台账），
      也不把云端清单挂进高频的 `/api/session/*`（每次切会话会多等一次递归 `list`，约 3s）。
      实测：全量 **329 例**通过；真实库里临时挪走一条会话 → 端点返回该条
      （`rev 4 · 4707B`）→ 侧栏出现「`☁ 09-20 18:44 · 未拉取`」、无删除/提交按钮、位置按 id 正确；
      挪回后条目消失。**逐条点击拉取未在真库上真点**（会改写 `current_session.txt` 并推云，
      留给用户自己点一次验证）

### 剩余

→ 已移入 **§11. 待办与下一步（会话存云的剩余 6 项）**。

## 1. 目标

把会话存储从「本地为准 + 异步推云」改成 **云（百度云 Depot）为真源，本地 `.l_agent_ws` 作为版本库的工作区（工作副本）**；
本地与云端不一致时**用状态显示**，不自动让任何一侧覆盖另一侧。

模型直接照 **Perforce**：depot ↔ workspace(client view / maps) ↔ `status` / `pending` / `have` /
`reconcile` / `sync_plan` / `sync_done`。

## 2. 已完成的

| 项 | 位置 | 验证 |
|---|---|---|
| 四态状态 + `from_plan`（用服务端清单算） | `src/l_agent_chat/depot_status.py` | 单元测试 |
| 云端清单 `cloud_manifest()`（`list` 递归） | `depot_sync.py` | 单元测试 |
| 逐条状态 `status_items()` | `depot_sync.py` | 单元测试 + 实测 |
| `depot_auto_push` 闸门（会话不自动推，设置镜像不受影响） | `depot_sync._session_push_allowed()` | 单元测试 |
| `ensure_workspace()` 幂等登记 | `depot_sync.py` + `app.py` | 单元测试 + 实测 |
| 提交/拉取 只处理变化项 + SSE 进度 | `depot_sync.{push,pull}_stream()` | 单元 + 端点 + 浏览器 |
| 删除会话 → 删云端 | `session_store.py` + `depot_sync.delete_session_async()` | 单元测试 |
| P4 其余操作（变更单/锁/检出/撤回/删除/对账） | `depot_sync.py` + `app.py` | 单元 + 端点测试 |
| 逐条操作入口（侧栏 + 设置页条目表） | `web/src/new/{NewApp,SessionList}.jsx` + `templates/settings.html` | 浏览器实测（检出/解锁） |
| 状态自刷新 | `web/src/new/NewApp.jsx` | 浏览器实测 |
| 优先级可见（`env_overrides`） | `config.py` + `templates/settings.html` | 单元测试 |

`workspace-status` 返回形状：

```json
{ "configured": true, "enabled": true, "auto_push": true,
  "library": "/l_agent_chat", "depot_workspace": "l_agent_chat",
  "local_root": "D:\\TD_Depot\\l_agent_ws\\.l_agent_ws",
  "sessions_local": 5, "ws_id": 68,
  "items": [{ "rel": "sessions/session_x.json", "status": "local_only",
              "label": "📄 仅本地", "rev": null, "local_size": 3768,
              "remote_size": null, "locked_by": "" }],
  "summary": { "cloud_only": 0, "local_only": 6, "modified": 0, "synced": 1,
               "total": 7, "dirty": 6 },
  "actions": [{ "rel": "…", "status": "local_only", "can_pull": false, "can_push": true }],
  "pending": { "lists": [{"id": 467, "files": 0, "items": []}], "files": 0, "error": "" },
  "locks": [{ "path": "/l_agent_chat/settings.json", "owner": "u2" }],
  "reason": "" }
```

> `local_root` 里的 `D:\TD_Depot\l_agent_ws\.l_agent_ws` 是**机器相关的示例值（当时环境）** —— 取该机「存储根」下的 `.l_agent_ws`；本机仓库已迁 `E:`，此值仅供参考。

## 3. 工作区登记（已解除的卡点）

原先每次提交被服务端 400 拒：

```
reachable: true            depot 服务通（127.0.0.1:8080/baidu）
library_registered: true   库 /l_agent_chat 已登记
ws_registered: false       ← 卡在这
last_error: 上传 /l_agent_chat/settings.json 失败 HTTP 400:
  "请指定工作区（ws 参数）：用户 admin01 有多个工作区或还没有工作区"
```

原因就是 `admin01` 名下**没有名为 `l_agent_chat` 的工作区**，而它**已拥有其它工作区**，所以服务端
无法替你挑一个。做掉下面三步即解除（已做成幂等的 `ensure_workspace()`）：

```python
_cfg()                                                   # (enabled, url, library, ws, timeout)
find_workspace(name, library)                            # GET /api/depot/workspace?all=true 里找
                                                         # (owner == LUGWIT_USER, name, library) 都相等的那份
_request("POST", f"{url}/api/depot/workspace", json={    # 没有才建
    "name": ws, "library": library,
    "local_root": <存储根>/.l_agent_ws,                   # 存储根，不是 agent 的活动根
    "host": "",
    "maps": [{"depot_path": library, "local_path": "", "exclude": False}]})
_request("POST", f"{url}/api/depot/workspace/select", json={"ws_id": id})
```

实测结果：`{"ok": true, "ws_id": 68, "created": true, "selected": true}` → `ws_registered: true`，
随后 `push_text()` 返回 `True`（HTTP 200）。

## 4. 关键实测（推翻了早期假设，省下重新摸索）

- **depot 路径是平铺的：`<库>/<rel>`，不带 `<user>` 段。**
  早期在这里写成 `<库>/<user>/<rel>`，理由是「auth 按路径首段判 owner」。**实测三条都否**：
  1. `PUT /api/depot/workspace/{id}/maps` 的文档原文：*空列表 = 回到 `/<library>/...` ↔
     `<local_root>/...` **隐式映射*** —— 平铺才是规范形状；
  2. `GET /api/depot/sync_plan` 按 `library + "/" + rel` 推导目标，**完全忽略 maps 里的 `depot_path`**；
  3. 向 `/l_agent_chat/system01/...` 提交**照样 200** —— 该端点**不按路径判 owner**。
- **`sync_plan` 是「云 → 本地」的**下载**计划，不是上传计划**。服务端原文注释：
  *「p4 sync -n：该工作区落后于云端的条目，并带上执行机上的目标路径」*
  （`lugwit_baidu_netdisk/999.0/src/lugwit_baidu_netdisk/web_server.py:2576`）。
  条目 `action`：`add`（云端有、`have` 里没有）、`edit`（两边 rev 不同）、`delete`（云端已删，
  该删本地那份）；`rev` 是**云端 head 版本号**，`have_rev` 是本地记录到的版本。
  → **不能再拿它当「本地改了什么」**：本地改动服务端**看不见**，要执行机扫描后
  `reconcile` 上报（同文档 §2.7、web_help §2.7 原文）。
- **本地改动只能自己记** → 因此加了**同步台账** `<存储根>/.l_agent_ws/depot_ledger.json`
  （`{rel: {sha, rev}}`，每次推/拉成功后写）。判定三方比对，见 §5。
  台账**不作为同步内容**（`local_files()` 只列设置/会话/指针三类）。
- **本库有一批 blob 物理缺失**：`download` 返回 404
  `blob 不存在: md5=… 目录不存在: /apps/Lugwit/.depot/blob/f4`，但 `list` 里那条 rev/大小都在
  —— 也就是「云端只有元数据、内容取不到」。服务端自己的文档记着同类漂移：
  *「清单引用到的 blob 登记缺 71 条」*（见 `网盘版本库Depot演进计划.md` T4）。
  实测 `l_agent_chat` 库里 7 个文件有 3 个是这种，**重传也没补上**（同 md5 走了秒传，
  没落地物理文件）。界面为它单列一个「⚠ 云端内容缺失」态。
- **`list` 才是云端清单的正解**（`GET /api/depot/list?ws=&dir=`，递归）。
  每条带 `rev / blob_md5 / size / action / locked_by / isdir`，`action` 是**该路径 head 修订的动作**
  （`delete` 说明已删）。`tree` 只列目录。
- **`have` 是客户端盖章的**，不是权威云端清单：`submit_stream` 会顺带盖章；「云 → 本地」那一侧要显式
  调 `POST /api/depot/sync_done`（item 键名是 **`depot_path`**，传 `path` → 400「depot 路径为空」）。
  它**只加不减**，删过的路径会以 `rev=<墓碑>` 留在里面 —— 所以状态**不再**用 `have` 当云端侧。
- **P4 端点都要 `?ws=`**：`admin01` 名下不止一个工作区，漏了就 400「请指定工作区（ws 参数，
  可传工作区名或 id）」。`mark_delete` / `submit_pending` / `checkout` / `lock` 都吃这个。
- **`maps` 条目键名**（`additionalProperties: true`，schema 查不到形状，只能抄）：
  `{"depot_path": "<库内路径>", "local_path": "", "exclude": false}`。
- **服务端已实现完整 P4 工作区模型**：`workspace` / `select` / `{id}` / `maps`(Client View) / `have` /
  `local_path` / `pending` / `status` / `reconcile` / `sync_plan` / `sync_done` / `checkout` /
  `lock`·`locks`·`unlock` / `changes` / `submit_pending` / `revert` / `history` / `tree` / `list` /
  `import` / `migrate` / `mark_add_stream`·`mark_delete`·`mark_move` / `delete`。
- **认证不在 depot 服务里**：POST `/baidu/api/v1/auth/login` → **404**；401 原文说「未登录
  **lugwit_auth**：请先通过统一认证登录」。`depot_sync._login()` 已实现该握手。
- **命名冲突**：`app.py` 里已有路由函数 `async def depot_status()`，所以模块必须**别名导入**
  （`from . import depot_status as depot_state`）。

### 订正（本文自己写错过的两条）

- ~~「`tree` 只列目录，对 `blob` 模式的库拿不到文件清单；`list` 对 blob 库返回空」~~ ——
  **`list` 对 blob 库能列文件**（本次改造正是靠它）。当时看到空列表，真因是工作区还没登记、
  请求被 400 拒，而 `list_dir()` 把失败**静默吞成了空列表**（已改成用 `_get_json()` 并让
  `cloud_manifest()` 把原因带出来）。
- ~~「`DEPOT_*` 环境变量优先，与 `rules_*` 那批（设置页优先）不一致」~~ —— **所有键都是
  「环境变量 > 设置页文件 > 默认」**，`rules_*` 也一样。真实问题是：`apply_settings()` 把值写回
  内存（本次立刻生效），**重启后又被环境变量盖回**。见 §9。

## 5. 状态怎么判（三方比对）

`depot_status.from_fingerprints(local, cloud, ledger)`：

| 来源 | 取法 |
|---|---|
| **本地** | `local_uploads()` = `{rel: 会上传的那份 bytes}`（设置是**脱敏后**的 dump —— 拿原文件比会把 `settings.json` 永远判成「已修改」） |
| **云端** | `cloud_manifest()` = `list?dir=` 递归（head 上真实存在的文件 + rev/大小/md5/锁） |
| **台账** | `load_ledger()` = 上次推/拉成功后记下的 `{sha, rev}` |

| 情形 | 结论 |
|---|---|
| 只有本地有 | `📄 仅本地`（可提交） |
| 只有云端有 | `☁ 仅云端`（可拉取） |
| 都有，台账吻合（sha + rev 都一致） | `✓ 已同步`（**不下载**） |
| 都有，sha 不符 | `✎ 已修改`（本地动过 —— 服务端看不到的那种） |
| 都有，rev 不符 | `✎ 已修改`（云端动过） |
| 都有，**没有台账** / 刚推完还没记上 rev | `UNKNOWN`（内部态）→ 下载 head 比一次：一致就 `✓` 并补进台账，不一致就 `✎` |
| 都有，`download` 404（云端只有元数据） | `⚠ 云端内容缺失`（可提交重传；不可拉取） |

- 为什么需要台账：服务端看不到本地改动（见 §4 的 `sync_plan` 订正）。光靠 `sync_plan` 会
  **漏报**（本地改了不吭声）也**误报**（`have` 落后 ≠ 本地有改动）—— 实测旧逻辑把 3 个
  真的不一致的文件报成了「7/7 已同步」。
- `UNKNOWN` 是内部态，出界前必须被解析掉（`status_items()` 会下载比对 + 写台账）。
- 台账只在推/拉成功后写；比对成功也算「学到了一件事」，顺手写进去，下次省掉那次下载。

## 6. 提交 / 拉取只碰「稳的那一头」

| 按钮 | 只处理 | 理由 |
|---|---|---|
| 批量 `☁ 提交` | `local_only` + `broken` | 推上去是「新增」或「重传」，不会覆盖别人的东西 |
| 批量 `⬇ 拉取` | `cloud_only` | 拉下来是「补齐缺失」，不会盖掉本地东西 |
| 逐条 `⬆` / `⬇`（侧栏 hover / 设置页条目表） | 任意（含 `modified`） | 用户看得见状态，自己选方向；拉取先备份 `.bak` |

`✎ 已修改`（两边都有但不一致）**批量按钮一律不碰** —— 那要「选边」，正是设计里说的
「不一致由状态显示，不替用户覆盖」。所以侧栏那行提示把三类分开报：
`1 项待提交 · 2 项不一致（逐条选方向）`，不让它藏进「待提交」里。



- `settings.json` 落在 `config.WORKSPACE_DIR/.l_agent_ws/`（由 `config.settings_file()` 决定），
  而会话落在 `workspace._storage_root()/.l_agent_ws/`。**切换工作区时两者会分叉** ——
  既有行为，本次没动。正常情况下两者相同。
- 密码 `lugwit_password` 上传前经 `depot_sync._redact` 剔除，不落云端；但它**明文存在本机
  `.l_agent_ws/settings.json`**（与设置页行为一致）。
- 关掉 `depot_auto_push` 后会话改动不自动推；但**删除会话仍会删云端那份**（`delete_session_async`
  不过那道闸门）—— 因为「删了本地却留个云端孤儿」在 P4 模型里说不通。若要连删除也尊重该开关，
  改 `depot_sync.delete_session_async` 加一个 `_session_push_allowed()` 判断即可。

## 7. 测试

```bat
wuwor l_agent_chat -- l_agent_chat_test
```

- `tests/test_depot_status.py`：状态比对（含 `from_plan`）+ 自动推闸门
- `tests/test_depot_workspace.py`（58 例）：平铺路径、`local_root`、`local_files` 清单、
  `cloud_manifest()` 递归与失败上报、`cloud_only_sessions()`（只列云端独有会话、剔除本地已有项、
  过滤非会话 rel、按 id 倒序、失败/关同步给原因、**期间零 `download`**）、
  `ensure_workspace` 的**入参拼装**（建成/复用/缺账号/
  关同步/建失败）、`status_items` 的三方比对（台账吻合不下载 / 无台账下载比对并记台账 /
  下载失败 → ⚠ 内容缺失）、`sync_done` 的键名、`push_stream`/`pull_stream` 的**筛选**
  （只推 `local_only`+`broken`、只拉 `cloud_only`、`modified` 批量不碰、拿不到状态不盲动、
  `force` 全量、`.bak` 备份）、**逐条** `push_one`/`pull_one`（只认三类 rel、设置脱敏、
  失败不请求）、P4 其余操作（rel↔库内路径、`?ws=`、关同步不发写操作、`revert_pending`）、
  删除会话通知 depot
- `tests/test_depot_status.py` / 模块自检：四态判定、`from_fingerprints` 纯函数（六种情形）、
  `pending_actions` 的方向映射、`summarize` 的 dirty 口径
- `tests/test_config_env.py`（6 例）：环境变量 / 设置页 / 默认 三级优先级与 `env_overrides`
- `tests/test_agent_endpoint.py`：`WorkspaceStatusTest`（未配置/不可达/关同步/登记失败 + 撞名回归）、
  `DepotPushStreamTest`（不可达 → 不推 + 原因；可达 → 只推变化项，用替身 depot 断言
  **只收到那一次提交**）、`DepotItemOpsEndpointTest`（逐条 `push-one` / `pull-one` /
  `checkout` / `unlock` / `delete` 走通；不自管路径不发请求）
- 夹具 `tests/agent_server.py` 支持 `AgentServer(url, settings=…, depot_enabled=bool, depot_url=…)`；
  默认**关云同步**，且把 depot 指向没人监听的本地端口 —— 免得测试去连真实网盘服务
- `_FakeDepot`：极小的 depot 替身（只答 `status` / `list` / `download` / `pending` / `locks`，
  记录 `submit_stream` 与各写操作），在端点层走通而不碰真实网盘

全量：**全绿**（`wuwor l_agent_chat -- l_agent_chat_test`；例数随功能增长，2026-09-22 为 476 例，
早期批次是 158 例。端点批默认不跑，加 `--endpoint`）。

## 8. 提交 / 拉取进度（SSE）

一次提交/拉取是**串行多次 HTTP**（一个文件一个请求），实测每文件约 3s。整批返回再显示，
界面上就是几十秒毫无反应 —— 所以把中间态发出来。两个方向**事件形状完全相同**：

```
GET /api/depot/push/stream[?force=1]      （SSE，GET 是为了兼容浏览器原生 EventSource）
GET /api/depot/pull/stream[?force=1]
data: {"kind":"start","total":3}
data: {"kind":"progress","index":1,"total":3,"rel":"settings.json","ok":true,"error":""}
...
data: {"kind":"done","ok":true,"total":3,"pushed":[...],"failed":[...],"error":""}
                                        （拉取是 "pulled"/"skipped"/"failed"）
```

- 实现：`depot_sync.push_stream()` / `pull_stream()` 是**各自唯一实现**（同步生成器），
  `snapshot_local()` / `restore_local()` 只是 `for event in ...` 吃到尾 —— 两条路不会各自漂移
- **只处理变化项**：`_push_work()` / `_pull_work()` 用 `status_items()` → `pending_actions()` 的
  `can_push` / `can_pull` 当待办清单，与界面的「N 项待提交」**同源**（否则按钮说 6 项、实际推 7 个）。
  拿不到状态时 `start.total=0` + `start.error=原因`，**一个都不动**（盲推/盲拉只会逐个 timeout）。
  `force=True` / `?force=1` 退回全量，用于服务端 `have` 与库对不上时救场（新机器第一次同步用拉取的 force）
- 收场话术分四种（`pushDoneText` / `pullDoneText` / 设置页 `syncDoneText`）：
  `未执行：<原因>` / `N 项成功 · M 项失败` / `云上已是最新，无需提交` / `已提交 N 项`
- 端点必须把同步生成器挪进线程（`_iter_in_thread`），且要**自己包 `_sse()` 帧**
  （`_sse_response` 只发裸 body，与 `/api/chat` 同一处坑）
- 前端两处都接了：侧栏用 `fetch-event-source`（`streamDepotPush` / `streamDepotPull`），
  设置页用原生 `fetch` + `body.getReader()` 手分帧。**不要用原生 `EventSource` 在 React 侧** ——
  它在流被服务端正常关闭后会**自动重连**，那会把整批文件再推一遍
- 实测帧是实时的（`start` 147ms，之后每约 3s 一帧），反代下靠 `X-Accel-Buffering: no` 不缓冲
- 拉取把「当前会话指针」排在最后，避免拉到一半应用就切走了会话
- 「只处理变化项」在**真服务**上验到的是「已同步 → 秒回无需提交 / 无需拉取」；
  「筛出 1 个、真发 1 次提交」由端点测试用替身 depot 验

## 9. P4 其余操作（变更单 / 锁 / 检出 / 删除 / 对账）

| 函数 | 端点 | 说明 |
|---|---|---|
| `pending()` | `GET /api/depot/pending?ws=` | 未提交的变更单（`lists[i].items` 是挂着的文件） |
| `locks()` | `GET /api/depot/locks?ws=` | 文件锁（`{path, owner, ...}`） |
| `checkout(rels)` | `POST /api/depot/checkout?ws=` | 标成「我编辑中」（`{paths: [...]}`，rel → 库内路径） |
| `unlock(rels, force)` | `POST /api/depot/unlock?ws=` | 解锁；`force` 才能解别人的（逐个调用） |
| `revert_pending(rels, cl_id)` | `POST /api/depot/revert_pending?ws=` | 把路径从变更单撤回（`p4 revert`） |
| `delete(rels, desc)` | `POST /api/depot/delete?ws=` | 删库内文件（服务端一次到位：mark_delete + 提交） |
| `submit_pending(cl_id, desc)` | `POST /api/depot/submit_pending?ws=` | 提交变更单 |
| `reconcile(items, cl_id)` | `POST /api/depot/reconcile?ws=` | 按给定条目对账（条目形状**不猜**，直通） |

- **写操作只有一个出口 `_post_json()`**，`depot_enabled=false` 时在那里统一拦住 → 一个写操作都不发
- **本包提交是即时的**（`submit_stream` 直接出 rev），所以正常情况下只有一份空白默认变更单；
  `pending.files > 0` 说明有东西挂在变更单里没提交
- 本包对应的应用层端点：`POST /api/depot/checkout` / `unlock` / `revert-pending` / `delete` / `reconcile`
- 顺带修掉一个真实缺口：**删本地会话时同步删云端那份**（否则它永远以「☁ 仅云端」挂着）

## 10. 逐条操作入口（界面）

批量按钮管不到的（`settings.json`、锁、删云端）走逐条入口：

| 位置 | 操作 | 显隐条件 |
|---|---|---|
| 侧栏会话条目（hover） | `⬆ 提交这条` | `can_push`（📄 仅本地 / ✎ 已修改） |
| 侧栏会话条目（hover） | `⬇ 拉取这条` | `can_pull`（☁ 仅云端 / ✎ 已修改） |
| 侧栏会话条目（徽标位） | `🔒`（锁归属进 tooltip） | `locked_by` 非空 |
| 设置页「工作区条目」表 | `⬆ 提交` `⬇ 拉取` `🔒 检出` / `🔓 解锁` `🗑 删云端` | 同上 + 云端有该条才给「删云端」 |

- 侧栏的显隐依据是 `workspace-status` 的 `actions[].can_push/can_pull`（在 `NewApp` 里并进
  `depotByRel`）—— 与「N 项待提交」**同一份数据**，不会各说各话。同步态**不出现**按钮，
  纯本地用法看到的列表与改动前一致（实测 0 个多余箭头）
- 单条端点：`push-one` / `pull-one` 只接受本包管的三类 rel（`settings.json` /
  `current_session.txt` / `sessions/session_*.json`），设置**照样脱敏**；
  拉取照样先备份 `.bak` 并 `sync_done` 盖章
- **解锁 = 解锁 + 撤回变更单**：只解锁不撤回，变更单里会一直挂着「已打开但没动」的记录
- 实测：检出 `settings.json` → 锁出现（`🔒 admin01/l_agent_chat`）+ 状态行出现「变更单未提交 1 项」
  → 解锁 → 全部复原
- 实测逐条 `⬆`：3 个 `✎ 已修改` 的会话/指针逐个点提交 → 云端 rev 9→10、5→6、5→6，
  大小与本地一致（批量按钮**故意**不碰这类，见 §6）

## 11. 待办与下一步（会话存云的剩余 6 项）

> 本节原文标题为「遗留 / 待判断」（原文为空）。此处填入原「进度清单 · 剩余」的 6 项，**编号保持不变**，以便其他文档对 §12~§15 的引用继续有效。

- [ ] **1. 本库有 blob 物理缺失，要服务端修**：`l_agent_chat` 库里 7 个文件有 3 个
      `download` 404（`blob 不存在 … /apps/Lugwit/.depot/blob/f4`），`list` 里元数据却齐。
      **重传也补不上**（同 md5 走秒传，没落地物理文件）。这不是客户端能修的 ——
      服务端文档记着同类漂移（`网盘版本库Depot演进计划.md` T4：「清单引用到的 blob 登记缺 71 条」）。
      客户端能做到的只是**如实显示**（⚠ 云端内容缺失）并允许重传
- [ ] **2. `reconcile()` 未接进流程**：条目形状**已经查清**（`{local_path, action: add|edit|delete,
      md5, size}`，见 `lugwit_baidu_netdisk` 的 web_help §2.7），但要走它得先把字节写进 blob 仓
      （`mark_add_stream` 或一步提交），本包用「一步提交」已够，故未接
- [ ] **3. `import` / `migrate` / `mark_move` / `revert`（回滚到旧版）未接**：当前用途用不到。
      `revert` 形状已知（`POST /api/depot/revert {path, rev, description}`，零流量指向旧 blob），
      要做「恢复历史版本」时接它即可
- [ ] **4. 设置页「☁️ 上传到百度云 / ⬇️ 从云拉取」与侧栏是两套措辞**（一个叫上传、一个叫提交），
      行为相同，是否统一文案未定
- [ ] **5. 首次刷新会慢**：没有台账时要逐条下载比对（实测 7 个文件约 6.8s，网盘服务每次请求约 3s）。
      比过一次就记台账，之后不再下载 —— 但**换了工作区或清掉台账**会重来一次
- [ ] **6. 云端条目只做到「列 + 按需拉取」**：`🗑 删云端` 仍只在设置页条目表；云端条目在顶部下拉 /
      历史抽屉（旧版 UI）里**不出现**（它们会引出重命名 / 删除语义）。若要在侧栏直接删别机的会话，
      得再补一个入口 —— 另外同库不同账号的会话目前**无法区分**（depot 路径平铺、无 `<user>` 段），
      要按 owner 过滤得先服务端支持分区

## 12. 状态自刷新与配置优先级

- **自刷新**：侧栏状态 30s 定时重取 + `visibilitychange` 切回本页立刻重取。
  起因：`depot_auto_push` 后台推云改了云端，没人事先通知界面 —— 徽标会一直停在旧状态。
- **配置优先级**：全局契约是 **环境变量 > 设置页文件 > 内置默认**（所有键一致）。
  麻烦在 `apply_settings()` 会把值写回内存 → **保存当次设置页赢、重启后环境变量赢**，
  用户看到的是「这个开关坏了」。**不改契约，改为让它可见**：
  `_v()` 记录命中环境变量的键 → `get_runtime()["env_overrides"]` → 设置页给这些字段挂 ⚠
  「改动本次生效，重启后仍以环境变量为准」。

## 13. 云端会话在侧栏可见、按需拉取

> 落地记录：OpenSpec change `cloud-session-lazy-pull`，已归档在
> `openspec/changes/archive/2026-09-22-cloud-session-lazy-pull/`；
> 主 spec 落在 `openspec/specs/cloud-session-list/spec.md`（4 需求 / 12 场景）。

**要解决的问题**：换一台机器打开 `l_agent_chat`，侧栏**看不到云端有多少会话** —— 只能整批
「⬇ 拉取」（等于把云上所有会话都下载一遍）或进设置页条目表逐条点。缺的是「先看到有哪些，
再决定拉哪条」这一环。

**接口**（只读，一条都不下载）：

```
GET /api/session/cloud  →  {"entries": [{"id","rel","rev","size","locked_by"}], "reason": ""}
```

- 判据：`status()` 探测可达 + `ws_id` → `cloud_manifest()` → 取 `rel` 形如
  `sessions/session_<id>.json` 且**本机 `local_files()` 里没有**的项（`depot_sync.cloud_only_sessions()`）。
- 拿不到时 `entries: []` + `reason`（未配置账号 / 关同步 / 不可达 / 未登记工作区），
  前端只当「没有云端条目」，**不弹错**。

**四个刻意选择**（都有代价，别回头改掉而不看理由）：

1. **独立端点，不动 `/api/session/{current,new,switch,delete,rename}` 的返回形状**：
   那五个是高频路径（切会话、每轮对话结束），而取云端条目要付一次递归 `list`（网盘约 3s）。
   挂在切会话上 = 每次点击多等 3s。
2. **不走 `status_items()`**：它无台账时会逐条 `download` 比对 + 写台账 + 可能判 ⚠，
   属重活且有副作用；这里只要「云端有哪些」。
3. **合并放前端**（`NewApp` 的 `listSessions`）：后端合并会让第 1 条的代价重新出现；
   且前端已有 `depotByRel` 这种「本地数据 + 云端数据并起来渲染」的模式。
4. **占位标题由 id 生成**（`☁ 09-20 18:44 · 未拉取`，id 即 `YYYYMMDD_HHMMSSmmm`）：
   为了标题去下载内容与「按需拉取」自相矛盾；拉取成功后标题自然换成文件里的 `title`。

**前端行为**：云端条目与本地会话按 id 时间倒序**混排**（两边都有时以本地那条为准）；
行里只给 `☁` 徽标 + tooltip（rel / `rev` / 大小 / 锁归属），**不给**重命名、删除、`⬆ 提交`
（本地没这个文件，这些操作只会失败）；点标题行 = `POST /api/depot/pull-one` → 成功后切到该会话
（拉取期间行内「☁ 正在拉取…」，防重复点）；失败就地显示原因、条目留在云端态。
刷新节奏复用既有钩子（初次加载 / 30s / `visibilitychange` / 推拉动作后），不新增定时器。

**状态还没到手时的转圈**：徽标是服务端算的，首次要等一次 depot `list`（实测约 4.5s 才出徽标）——
这段空白会被读成「这些会话没有云状态」。所以 `NewApp` 用 `depotLoading`（**只在首次**，
30s 轮询不置真，否则转圈周期性闪）驱动：每行徽标位给一个转圈 + 侧栏底部「正在获取云同步状态…」，
到手后一起换成徽标/提示。实测：0.5s 起底部 1 个转圈，1–4.5s 行内 5 个 + 底部 1 个，5s 起徽标到位。

**语义未变**：云 → 本地仍然只走显式拉取；批量「⬇ 拉取」仍然只碰 `cloud_only`；
`✎ 已修改` 仍只走逐条。

**验收（真库实测）**：全量 **329 例**通过（新增 5 例，含「断言期间没有任何 `download`」的只读契约）；
把一条非当前会话临时挪成 `.json.hide` → 端点列出该条（`rev 4 · 4707B`）、侧栏出现
「`☁ 09-20 18:44 · 未拉取`」、位置按 id 正确、该行无删除/提交按钮；挪回后条目消失、无残留。
**逐条点击拉取未在真库上真点**（会改写 `current_session.txt` 并推云，动到当前会话）。

**已知边界**：旧版 UI（`/classic` 的 `Header` 下拉与历史抽屉）不显示云端条目；
`🗑 删云端` 仍只在设置页条目表；同库不同账号的会话**无法区分**（depot 路径平铺、无 `<user>` 段）。

## 14. 「⚠ 云端内容缺失」重传修不好 —— 服务端去重缺「验存」（已修）

> 2026-09-22 修复。改的是 `lugwit_baidu_netdisk`（服务端），客户端只订正注释与文档口径。

**现象**（用户实际遇到的）：点侧栏那条 `⚠` 的 `⬆` 提交，界面上**没有任何进度**，
点完 `⚠` 还在 → 看着像"上传失败"。实际上 `push-one` **返回了 `ok:true`**、
云端 `rev` 也真涨了（`7→8`、`8→11`、`11→12`），但状态一直是 `⚠`。

**根因**（在服务端，不在客户端）：

- `depot_service.submit_files()` 提交 blob 模式文件时，去重判据是
  「`depot_blob` 里有 `(lib_root, md5)` 这行吗」→ 命中就 `stats["dedup"] += 1`、**不上传**；
  而 `store.submit()` 写 rev 是**无条件**的。
- 那几行 `depot_blob` 的 `remote_path` 指向**已不存在的旧布局路径**
  （`.depot/blob/<md5[:2]>/<md5>` 这段全局池迁移遗留）→ 于是「rev 涨、blob 不写、
  下载永远 404（`errno=-9` 路径不存在）、判 BROKEN、**重传也修不好**（每次都走进 dedup）」。

**修法**：提交前**验存**，登记行指向的文件不在网盘上就不算命中，改为真正重传。

- 新增 `depot_service.blob_row_alive(store, at, apps, md5, library, cache)`：`blob_get` 命中后
  再 `_remote_file_exists(登记的 remote_path)`，不存在就打印
  「blob 登记失效（网盘无此文件），本次改为重传」并返回 False。
- `submit_files()` 与 `mark_add()` 两处去重分支都改成 `if not await blob_row_alive(...): 上传`。
- `_remote_file_exists()` 多一个可选 `cache`：按**父目录**缓存 `list` 结果
  （md5 前两位相同的 blob 共用 `blob/<md5[:2]>/`，不缓存会为每条重复打一次网盘目录列举）。
  老的调用方（`materialize_dir_root`、`depot_migrate_blob_lib.py`）不传 cache，行为不变。
- **不需要动数据库**：修完再重传，服务端自己把 blob 补上并覆写 `remote_path`
  （原本准备的「删 `depot_blob` 行」这一步因此没做）。

**验收**：改完（服务端 `.dev_mod` 自动热重载，进程 15:35:01 重启）客户端重传那三条 →
`summary` 从 `synced 4 / broken 3` 变成 **`synced 7 / broken 0`**、`actions` 空、
`last_error` 空；`rev` 分别为 `current_session.txt 13`、`session_20260810_125756.json 9`、
`session_20260918_113501848.json 12`。侧栏 5 行徽标全 `✓`。
去重**命中**的那条路也补测了一次（推一个本来健康的 `session_20260920_185553997.json`）：
`ok:true`、状态仍 `synced`、`last_error` 空 —— 新加的验存没把正常去重弄坏
（rev `4→5` 是既有行为：`submit` 一律写新版本）。

`lugwit_baidu_netdisk` 自带的 HTTP 套件（`tests\test_depot_api.py`）**本次没跑起来**：
直连 `127.0.0.1:1028/baidu` 返回 `401 Unauthorized`（套件不支持带 token，属既有的环境问题，
与本次改动无关）。本次的服务端验证靠上面的真库重传。

**顺带补的界面反馈**（同一批问题的一半是"看着没反应"）：逐条 `⬆/⬇` 与点云端条目拉取
统一用 `busyRel` 标记 —— 那一行的按钮位置换成转圈并常显（原先按钮 hover 才显形、
点完就消失，且逐条操作**没有** SSE 逐条数字，只有批量才有「正在提交 3/7」）。

**检查清单提醒**：以后再看到 `⚠` 一直是 `⚠`、但 rev 在涨 —— 先怀疑服务端 dedup 的
"登记行 vs 网盘文件"不一致，别在客户端反复点。

## 15. 工作区本体：一份 `.code-workspace`（2026-09-26）

agent 自己的「工作区」不再用私有格式，**直接就是一份 `.code-workspace`**（与 VS Code 同格式）——
同一份文件也能给 VS Code 打开，两边看到的文件夹与顺序一致。

| 项 | 现在 |
|---|---|
| 工作区文件 | 默认 `<REZ_ROOTS[0]>/l_rez_src_ws.code-workspace`；`AGENT_CHAT_WORKSPACE_FILE` > 设置 `workspace_file` 可覆盖 |
| 文件内容 | 只写 `folders`（每项 `{path, name?}`：绝对，或**相对文件所在目录** —— 与 VS Code 同规则；同盘写相对、跨盘写绝对）；`settings` / `extensions` / `launch` / `tasks` 原样保留，agent 不增删 |
| 基准（相对路径 / shell cwd） | **`folders[0]`**：切基准 = 把该项移到数组首位（对 VS Code 侧零副作用） |
| 运行期偏好 | `<锚点>/.l_agent_ws/settings.json` 的内部键：`workspace_source` / `vscode_root` / `workspace_file` / `storage_root`（默认 = 锚点） |
| 旧格式 | `<锚点>/.l_agent_ws/config.json` 的 `roots` / `active_root`、`~/.lugwit/agent_chat_workspace.json`、`<根>/.agent_chat/` —— **不再读写、不做迁移** |

- **`save_settings()` 改成合并写**：设置页保存的是 `editable_keys()` 那一组，整体覆盖会把上面那几个
  内部键顺手抹掉；`config._INTERNAL_KEYS` 另外保证体检（`settings_warnings`）不把它们报成「未知键」。
- **接口兼容**：`/api/workspace` 的 `roots` / `active_root` 形状不变（前端不用改），新增 `workspace_file`
  字段；注入的多根说明与「其余文件夹必须绝对路径」照旧。
- **部署**：远端若没有那份工作区文件，`folders` 为空 → 视野根会退回存储根（锚点）；给远端补一份
  （`folders` 按它原有的 roots 写）即可 —— `push_pkg_to_remote.py` 只推包内文件、不覆盖它。
- 测试：`tests/test_workspace_file.py`（10 例：默认位置 / 相对路径解析 / `folders[0]` 基准 / 写回保留
  未知键与显式 `name` / 切基准与删除的顺序语义 / 不再产出 config.json）；测试隔离用
  `AGENT_CHAT_WORKSPACE_FILE`（否则会读写真仓库里那份，见 `tests/agent_server.py`）。

---

# 第二部分　agent 主循环与周边加固（2026-09-29）

> 源自《`Rez_pkg/l_agent_chat_交接_已修与待办.md`》。**该篇自述「历史」**：写于 2026-09-29，待办已**全部办结**（原文保留以备对照）；含 `D:\TD_Depot\...` 老路径，已加注（汇总见 §24）。
> 涉及包：`l_agent_chat`（对话 agent，:1250）、`l_notepad_server`（知识库，:8765）、`l_agent_tool`（工具库）、`l_repo_sync_gui`（直推部署通道）。
> 原文开头写「全部路径在 `D:\TD_Depot\Software\Lugwit_syncPlug\lugwit_insapp\trayapp\rez-package-source\`」—— **这是仓库搬到 `E:` 之前的当时环境（已失效）**；本机现为 `E:\lugwit\trayapp\rez-package-source\`，远端仍为该 `D:` 路径。
> 原文的「第 1 节 三条陷阱」在本文中即 **§17**。

## 16. 目标与验收标准（先看这节）

### 16.1 目标

**让 `l_agent_chat` 在真实任务上"说到做到"——它承诺的是「查清原因并修掉」，
验收就按这个承诺收，不按"给出了合理分析"收。**

今天暴露的本质：agent 表面上像个"不听话的模型"，实际是 **harness 在替它做决定**
（工具一失败就终止整轮、收尾门挂在单个出口上、信号传不到模型）。目标是把这些
**结构性坑**堵掉，让模型的努力能真正落到交付上。

一句话：**目标是「任务闭环」，不是「现象消失」。**

### 16.2 验收标准（每条都可复现、且**不能自证**）

| # | 目标 | 验收标准（可复现的判据） | 判据来源 |
|---|---|---|---|
| **G1** | **窄屏下消息卡片可正常滚动**（原始需求） | Playwright `375×667` 打开 :1250 → 找一条超过屏高 2/3 的长消息 → 在卡片上滑到边界后继续上滑 → **断言外层会话容器（`.au-viewport`）确实跟滚**；再点卡片确认「展开/收起」仍正常 | 断言结果 + 截图 |
| **G2** | agent 不再**提前终止**本轮 | 构造一次可恢复的工具失败（如给 `read_file` 一个不存在的路径）→ 事件流 `stop": true` **为 0**，且**该轮之后仍有新的 `tool_start`**（证明它换手段继续了） | `events/<session>.jsonl` |
| **G3** | **收尾门**三条路径都通 | 同一轮里：①它想收尾 → `收尾前请用户确认` ≥ 1；②点「继续做」→ `ask_required` +1 **且 `tool_start` 增加**（证明真接着干，不是原地再问）；③点「可以结束」→ `done": 1` | 同上（今天已验，需回归） |
| **G4** | 失败**有上限**、不会无限重试 | 造连续失败（≥3 次同一操作失败）→ 到上限后**主动收尾并说明**，而不是继续空转 | 同上 |
| **G5** | **等人回答时断线**不留僵尸会话 | 发起提问 → agent 调到 `ask_user` → **强行关掉浏览器** → 重新打开该会话：状态应为**「已中断」**（不是永远"运行中"、不是空白） | 会话界面 + 事件流 |
| **G6** | 远端与本地**代码一致**且服务在线 | `missing_files()` 对四包（`l_agent_chat` / `l_notepad_server` / `l_agent_tool` / `l_agent_market`）返回**缺 0 / 不一致 0**；远端 8765 / 1250 均 HTTP 200 | `deploy_verify` 判据 |
| **G7** | 假错误为 0 | 正常流程（含门控触发）事件流里 `"type": "error"` **为 0** | 事件流 |
| **G8** | 文档无欠账 | `l_agent_chat` 的 `doc/CHANGELOG.md` 补一条本次改动（`l_notepad_server` 的 v3.5.0~v3.5.5 已覆盖 §19.2，不重复补）；本文件末尾「未验」项清零或降级为「已知取舍」 | 文件存在 + 内容 |

### 16.3 完成定义（DoD）

**全部 G1~G8 都有实测判据即可交付**；任一条只有"看起来好了"没有判据，
**不算完成** —— 今天三次误判都是这么来的（见 §17 陷阱 1）。

判据必须**成对且不可自证**：例如 G2 要 `stop": true` 归零 **且** 后续有新的 `tool_start`；
G3 要 `ask_required` 增加 **且** `tool_start` 增加。**单看一个数一定会被骗。**

## 17. 三条陷阱（今天踩过，代价最大的一节）

### 17.1 陷阱 1：拿**单个指标**当"修好了"

今天三次「以为修好」，全是同一个错：

| 我当时的判据 | 真相 |
|---|---|
| 界面不再报错 = 修好了 | 实际是我删了那条 error 事件，**门还是没生效** |
| `done=1` = 会话正常收尾 | 实际是**被一次工具超时掐断**，回答不完整 |
| `远端缺 0` = 部署完整 | 那只是**存在性**，二进制字节坏的它看不出来 |

**正确做法：判据必须成对，且都不能自证。** 例：收尾门是否生效，要看
`stop": true` 归零 **且** `收尾前请用户确认` 非零 —— 两个数同时成立才算通。

### 17.2 陷阱 2：把 harness 的行为归因成**模型的性格**

我在第 2~7 轮反复看到「agent 失败一次就写报告收尾」，据此写了长篇分析说这是
「模型不重试」的习惯问题，还提了两轮**提示词修法**（插催促、堵"只输出结论文本"）。
**那些修法不可能生效** —— 因为真相是：

```python
# app.py（改前）：工具抛异常 / 编辑预检失败 → 直接终止整轮
agent_client._append_tool_turn(plan_messages, step_out, call, failed)  # 已经把失败喂给模型
stop_loop = True        # ← 然后立刻把门关了
break
```

**是 harness 主动终止，模型根本没机会重试。** 注释自己写着「可换别的工具」，代码下一行就把它掐了。
→ 教训：**看到"模型做了坏事"，先去循环控制里找有没有人在替它做决定。**

### 17.3 陷阱 3：跨版本线用「git 差集」推送 = 制造线上事故

今天真的把远端知识库推挂了（详见 §19.3）。**这条通道是给"同一条线的增量同步"设计的，
不是通用发布工具。** 跨版本线必须走**全量直传**（本地清单驱动）。

## 18. 一句话现状

agent 主循环的硬伤（**提前终止 / 收尾门被绕过 / 假错误 / 连败空转 / 断线僵尸**）今天已修并实测通过；
知识库检索的排序与默认档已调过；部署通道补了完整性校验与分批优化。
**原始需求「手机窄屏下消息卡片滑不动」也已闭环**（真因是 `overscroll-behavior:none`，见 §21.1）。
本轮收尾（2026-09-29）额外完成：连续失败计数器、断线中断标记验证、远端四包一致性 +
收尾门远端复验 + `skip_unchanged` 大包实测、`l_agent_chat` 补 CHANGELOG（§21.1~§21.5 均已办结）。

## 19. 已修（附实测判据）

### 19.1 agent 主循环（`l_agent_chat/999.0/src/l_agent_chat/app.py`）

| 改动 | 位置 | 判据（实测） |
|---|---|---|
| **工具失败不再终止整轮** | 去掉 3 处 `stop_loop = True` + `break`（工具抛异常 / 两条编辑预检失败） | 成对判据（`ac_g237_out.txt`）：`stop": true` **= 0** **且** 失败后**仍有非 ask_user 的新 `tool_start`**（实测：`read_file` 失败 1 次 → 下一步 step=1 继续 `read_file`） |
| **收尾确认门**（动手的回合收尾前必须调 `ask_user` 问用户） | `if not calls:` 里的收尾点 + `ask_user` 回答处（答"继续"则 `confirm_asked=False` 重新上膛） | 三条路径全验（`ac_g237_out.txt`）：①拦住 → `收尾前请用户确认` 标记 **= 1**；②答"继续做" → `ask_required` **= 3** **且** 门控后仍有真实 `tool_start`（门控前/后 = 2/1，证明确实接着干）；③答"可以结束" → `done": 1`（回答 1030 字） |
| **收尾门不再发 `type: error`** | 同上 | `error` 由 1 → **0**（原是我发的信息性提示，界面显示成红色错误） |
| **连续失败计数器**（§21.2） | 去掉「失败就终止」后补上限：`MAX_CONSECUTIVE_FAILURES`（默认 3）；`_note_tool_result(ok, tool)` 连续计数、成功清零；到上限置 `fail_limit_hit` 走收尾通道（**不置 `stop_loop`、不发 `type:error`**） | 造连续失败：失败 `tool_result`=3 / 最长连败段=3；收尾 status=`连续 3 次工具调用失败，本轮收敛收尾并说明`；`stop": true`=0；`error`=0；`done`=1（回答 1298 字） |
| **等回答时断开 → 标记中断** | `ask_user` 的 `await` 加 `except asyncio.CancelledError:` → `interrupt_flag.set()` + `run_box["interrupted"] = True` + `_mark(...)` 后 `raise` | ✅ 已验（两种断法各 8/8 PASS）：`active` 归 `None`、`last.running=False`、`last.interrupted=True`、落盘末条 `meta.aborted=True` 且正文含「连接中断」、`steps` 含「客户端断开」、`error`=0。<br>· FIN 断（`ac_g5_check.py`，输出 `ac_g5_out.txt`）<br>· **真 RST 强断**（`ac_g5_rst.py`，裸 socket + `SO_LINGER(1,0)`，输出 `ac_g5_rst_out.txt`；`ac_g5_check.py` 的 urllib3 路径在本地版本取不到底层 socket，会退化成 `close()`=FIN，故另建裸 socket 脚本保证真 RST） |
| **内联 XML 兜底认 DSML 方言** | 新增 `_normalize_dsml()`，入口归一化；`_INLINE_TAGS` 补全角竖线变体 | 单测 13/13（样本取自真实日志的 `done.reply`） |
| **`ask_user` 应有提示词约束** | `_PLANNER_SYSTEM` 加 `ASK, DON'T END` 段 | 与门控叠加后生效 |
| **`notepad_read` 404 兜底** | 代码源命中（`kb_name` 是 rez 包名）时直接当文件读；读不到给可执行错误 | 单测 8/8 |
| **`notepad_search` 透传质量信号** | 返回体加 `confidence` / `chunk_line` / `degraded` / `unindexed_packages` | 第三轮实测：模型自己在结论里写了「本次检索 degraded（top-1 置信度 0.44）」 |

### 19.2 知识库检索（`l_notepad_server`）

- 默认档 `hybrid` → `auto`（`routers/search.py`、`routers/kb.py`）：**不传 mode 138ms，原 1365ms**
- `auto` 回退判据 40/1.3 → **25/1.0**：回退率 65% → 11%，`p50 1397ms → 230ms`，`recall` 未丢
- `search_text` 提速：`l_agent_tool` 内置 `bin/rg.exe`（原来 PATH 无 rg → 走纯 Python 兜底，**一次几分钟**）；兜底路径改 `os.walk` 目录级剪枝 + 2MB 上限
- 重排默认关（`rerank_enabled` 默认 False）、导航块降权、块长度归一、RRF 融合、`confidence` 查询内校准
- **文档**：`Rez-Docs/Rez_pkg/l_notepad_搜索改造史.md`（一期 §5 / 二期 §6，含 §6.9 性能校准）

### 19.3 直推部署通道（`l_repo_sync_gui`）

**事故**：按 git 差集推 → 推了 `routers/kb.py`（新 `from .. import workspace_sync`）、
没推 `workspace_sync.py`（已 commit，不在差集里）→ 远端 `ImportError` → **线上 8765 宕机**。

已补（`deploy_verify.py` 新增 + `main._push_deploy_run` 接线）：

- `missing_files()`：拿本地清单问远端，把「本地有、远端没有」**并入本次上传批**
- 跳过条件加严：原「git 无差异就跳过」→「**无差异且远端不缺**才跳过」
- `DEFAULT_MAX_BODY` 60 KB → 2 MB（往返 90 → 9）；**批失败改为对半拆重试**（原整批静默丢弃）
- `skip_unchanged` **显式参数**（默认关）：先 `hash_files` 探测，内容一致的跳过（实测 76 文件 0.2s / 0 写入，见 `g6_push_align_out.txt`）
- 详细记录见 `l_repo_sync_gui/999.0/README.md` 末尾「直推部署通道：完整性加固」一节

## 20. 已验 / 未验（这条线必须分清楚）

| 项 | 状态 |
|---|---|
| 收尾门三条路径（拦住 / 放行 / 打回） | ✅ 本地实测（事件流判据见 §22） |
| 工具失败不终止整轮 | ✅ 成对判据：`stop": true = 0` **且** 失败后仍有新 `tool_start`（`ac_g237_out.txt`） |
| DSML 方言 / `notepad_read` 兜底 / `rg` 内置 | ✅ 单测 |
| **连续失败计数器** | ✅ 造 ≥3 次连续失败 → 主动收尾并说明；`stop": true`=0、`error`=0（见 §19.1） |
| 远端四包代码一致性 | ✅ 缺失 0 / 不一致 0（sha256 **逐文件**比对，`deploy_verify` 判据；`l_agent_chat` 151 / `l_notepad_server` 76 / `l_agent_tool` 20 / `l_agent_market` 13 全 `bad=0 missing=0`；远端 8765 / 1250 均 HTTP 200，见 `g6_final_out.txt`） |
| 远端 `bin/rg.exe` 字节完整性 | ✅ sha256 逐位一致 + `--version` rc=0 |
| **断线中断标记（CancelledError 分支）** | ✅ 本地实测：`active=None` / `last.running=False` / `meta.aborted=True` / `last.interrupted=True` / `error=0`；FIN 断与**真 RST 强断**（`ac_g5_rst.py`，`SO_LINGER(1,0)`）各 8/8 PASS（`ac_g5_out.txt` / `ac_g5_rst_out.txt`） |
| **收尾门在远端** | ✅ 远端复验 PASS：`{"session":"20260928_234035855","turn_len":116,"turn_types":{"agent":1,"status":20,"step_start":4,"tool_start":5,"tool_result":5,"file":1,"step_finish":4,"reasoning":31,"ask_required":2,"delta":42,"done":1},"gate_text":2,"ask_required":2,"done":1,"error":0,"stop_true":0,"dsml":0}`（`g6_remote_gate_turn_out.txt`，按「最后一个 agent 事件起」切片统计） |
| **`skip_unchanged` 在大包（`l_agent_chat`，151 文件）上** | ✅ 实测：探测 151 / 跳过 150 / 上传 1 / 0.4s（`g6_push_align_out.txt`；四包合计 `written=1 skipped=259 errors=0`；`l_notepad_server` 76 文件 0.2s 亦验） |
| **MCP 真浏览器端到端实测**（本机 :1250，与脚本化 SSE 测试互为交叉验证） | ✅ 真 Chrome（MCP Playwright，`headless=false`）跑完整一轮：会话 `20260929_140455702`，下发「①读不存在文件 ②读 `ask_q.txt` 贴前 3 行 ③收尾前先问我确认」→ 规划 1/80 步并行 `read_file`×2（1 失 1 成）→ 收尾门 #1（3 选项）→ 点「先查一下…是否在别处/路径写错」→ **打回后真接着干**（新 `run_command`×2 失败 → 改 `execute`）→ `execute` 全盘 glob 240s 超时 → 缩浅层重扫 1.21s 成功 → 收尾门 #2（自检清单 + 2 选项）→ 点「可以结束」→ `done`。**成对判据**（首回合 245 事件）：`step_start/finish` 6/6、`tool_start/result` 8/8、`ask_required/作答` 2/2、`done` 1、`"stop": true` **0**、`"type":"error"` **0**、DSML **0**（`ac_browser_e2e_out.txt` §一~§三；9 张截图 `C:\Users\wb.fengqingqing\Downloads\agent_run_*.png`） |
| **`run_command` 坏点与修复复验**（端到端实测中暴露） | ✅ 真故障：:1250 进程 PID 77160 启动于 `13:21:32`，而 `settings.py`（新增 SCHEMA 键 `command_cwd`）落盘于 `13:21:40`（**晚 8 秒**）→ 进程内是旧 SCHEMA → `settings.get("command_cwd")` 抛 `KeyError` → `_resolve_command_cwd` 崩 → `run_command` 全失败（seq30/31）。`wuwo svc restart l_agent_chat`（**18.3s 就绪**，新 PID 112912 @ `14:26:00`）后真浏览器复验：`(Get-Date).ToString("o")` → `state=completed, duration=0.36s, blocked=false, exit_code=0, cwd="D:\...\trayapp"` ✅（`ac_browser_e2e_out.txt` §五） |
| **`_DANGEROUS_PATTERNS` 误伤（已修）** | ✅ **已修**（2026-09-29）：`l_agent_tool/agent_tools.py` 的 `_DANGEROUS_PATTERNS` 由**子串匹配**改为**正则 + 词边界/盘符约束**（预编译 `_DANGEROUS_RE`，`re.IGNORECASE`）；`format ` → `(?<![\w-])format\s+[a-z]:`（要求盘符；`-Format o`、`Format-Table` 不再命中）。**成对判据**：①单测 20/20 PASS —— 安全 8 条（`Get-Date -Format o` / `Format-Table` / `Format-List` …）全 `blocked=False`，危险 12 条（含 `format C:` / `FORMAT D: /q`）全 `blocked=True`（`ac_fmt_out.txt`）；②`wuwo svc restart l_agent_chat`（**15.4s 就绪**，红线⑤）后 **HTTP/SSE 端到端复验**：`Get-Date -Format o` → `{"blocked": false, "exit_code": 0, "stdout": "2026-09-29T15:21:31.98+08:00"}`、事件流 `error=0`（`ac_fmt_e2e_out.txt`）。注：本次 MCP 真浏览器被外部进程抢占（页面反复跳到 `:1234/video-editor`），故端到端改走脚本化 HTTP/SSE，与上一行「MCP 真浏览器」互为交叉验证 |
| **工具卡显示代码改动 diff（`write_file` 补齐）** | ✅ **已补**（2026-09-29）：链路本就通 —— `edit_file` / `apply_patch` 的结果自带 `diff` → `app.py::_result_diffs` → `tool_result.diffs` → 前端 `ToolCard` → `DiffBlock`（逐行 +/- 着色、`+N -M`、折叠、按扩展名高亮）。缺口在 `write_file`（`l_agent_tool`）只返回 `{path, written, chars}`，而 `AUTO_APPROVE=1`（默认）下不走审批预览 → 卡片只剩「已写入 N 字符」。现于**写盘前**（与 checkpoint 同一时机）用 `_diff_preview` 算一份挂到 `pending_diff`，复用既有回挂逻辑进 `tool_result.diffs`；另修审批预览分支对**相对路径**直接 `Path(path)` 读盘（读到进程 cwd → old 为空 → diff 全变「新增」）改走 `agent_tools.resolve_workspace_path`。**成对判据**：①`edit_file` 路径 HTTP/SSE 端到端 → `ok=True diffs=1`、diff 含 `@@`（`ac_diff_e2e_out.txt`）；②`write_file` 路径 `wuwo svc restart l_agent_chat`（**18.1s 就绪**，红线⑤）后 → `diffs=1`、diff 含 `-line2` **且** `+line2-changed`（确证读到旧内容、非整文件当新增）、事件流 `error=0`（`ac_diff_write_e2e_out.txt`）。前端零改动。**G6**：全量直传 `written=2 skipped=258 errors=0`（仅 `src/l_agent_chat/app.py` + `doc/CHANGELOG.md`；`g6_push_align_out.txt`），四包 `bad=0 missing=0`、远端 1250/8765 HTTP 200（`g6_final_out.txt`）；远端 `wuwo svc restart l_agent_chat` RC=0，**独立证据**：监听 1250 的 PID 17416 启动于 `2026-09-29T15:43:20+08:00`、探活时 age 26.5s（`g6_remote_pid_out.txt`）。 |
| **远端 33 篇语料重建索引** | ⏸️ **已知取舍**：代码对齐 ≠ 数据对齐；重建属动远端数据，按 §23 第 3 条先拍板再做 |

## 21. 待办（✅ 全部办结 · 2026-09-29 收尾；原文保留以备对照）

### 21.1 手机窄屏下「消息卡片滑不动」—— ✅ 已闭环

> **办结（2026-09-29）**：真因 = `.bubble` 上叠的 `overscroll-behavior: none`
> 在不支持 `overflow: clip` 的引擎上退化成**内层滚动容器**，把手势掐断在气泡里、不往外传
> → 外层页面纹丝不动。已删掉两处 `overscroll-behavior: none`（只留 `overflow: hidden` / `clip`），
> 并在 `style.css` 就地留下「不要再叠 `overscroll-behavior:none` 当双保险」的踩坑注释。
> 375×667 实测（`ac_g1_check.mjs`，CDP `Input.dispatchTouchEvent` 派发滑动手势，**10/10 PASS**）：
> 长卡片限高生效（`clientH=438 / scrollH=1261`）；气泡不再是内层可滚容器（`overflow-y=clip`、
> `overscroll-behavior=auto`）；在卡片上滑到边界后继续上滑，外层 `.au-viewport` 跟滚
> （`300 → 494 → 600 → 600`）而气泡自身 `scrollTop` 纹丝不动（`0 → 0 → 0 → 0`）；点击展开/收起正常；
> **含负对照**：退化引擎 + 回填 `overscroll-behavior:none` → 卡死 `300 → 300 → 300`（证明断言确有分辨力）。
> 输出 `ac_g1_out.txt`（旧脚本 `ac_scroll_fix_probe.mjs` 仅作探测）。

症状：窄屏（375px）下长消息卡片滑到滑不动之后，**整页也跟着滑不动**，手势像被卡片吃掉。

**已知**：
- 第 4 轮 agent 用 `apply_patch` 改过 `l_agent_chat/999.0/web/src/style.css`（+14 行），
  **那次改动从未验证过**，而 `dist` 在这之后重建过 → **线上产物里可能已带着一个未验证的改动**
- 第 7 轮它用真浏览器（窄屏 + 派发事件）复现并**定位到** `.bubble` 的
  `overscroll-behavior: none` 叠加 `overflow: clip` 在退化引擎下变 `hidden`（`style.css:1157/1251/1288`）
- 但那次 `apply_patch` **预检失败**（`old_string 找不到`），**没落地**

**接手第一步应该是**：先确认 `style.css` 当前那 14 行改的是什么、要不要留；
再按第 7 轮的定位（`overscroll-behavior` → `contain`/`auto`、`overflow: clip` 的降级回退）
做最小修复；然后**用 Playwright 在 375×667 下实测**（滑到边界后继续上滑，断言外层 `.au-viewport` 是否跟滚）。

### 21.2 「连续 N 次失败才收尾」的计数器 —— ✅ 已办

> **办结（2026-09-29）**：`config.py` 新增 `MAX_CONSECUTIVE_FAILURES`（默认 3）；
> `app.py` 加 `_note_tool_result(ok, tool)` 连续计数（成功清零），到上限置 `fail_limit_hit`
> 走收尾通道。实测见 §19.1 / §20。

今天去掉了「失败就终止」，但**没加上限**。当前是「失败不再终止、也不设上限」，
理论上存在反复失败的场景。`app.py` 3976 附近的注释里写明了设计意图（连续计数，不是一次就收）。

### 21.3 断线中断标记的验证（§20 第一行）—— ✅ 已办

> **办结（2026-09-29）**：用 `SO_LINGER(1,0)` 发 RST 强断（`ac_g5_check.py`）验证通过：
> 会话不再停在未完成态；`active` 归 `None`、`last.running=False`、`last.interrupted=True`、
> 落盘末条 `meta.aborted=True` 且正文含「连接中断」、`error=0`。
> 关键点：必须**同时**置 `run_box["interrupted"]`（正常收尾那行在 `raise` 之后跑不到）。

需要构造：发起提问 → agent 调到 `ask_user` → **强行断掉浏览器** → 看会话是否停在未完成态。
如果仍停在未完成态，说明 `raise` 之后的收尾/落盘**没被执行**（生成器被取消时不会往下走），
那就得在**生成器外层**包一层取消处理。

### 21.4 收尾门在远端复验 + `skip_unchanged` 在大包上实测 —— ✅ 已办

> **办结（2026-09-29）**：先把本机四包**全量直传**远端并 `wuwo svc restart` 重启，
> 再在远端复验收尾门 —— 判据（按「最后一个 agent 事件起」切片，`g6_remote_gate_turn_out.txt`）
> `{"session":"20260928_234035855","turn_len":116,"gate_text":2,"ask_required":2,"done":1,"error":0,"stop_true":0,"dsml":0}`
> （PASS）；`skip_unchanged` 在 `l_agent_chat`（151 文件）实测（`g6_push_align_out.txt`）：
> 探测 151 / 跳过 150 / 上传 1 / 0.4s，四包合计 `written=1 skipped=259 errors=0`。
> 复验另含远端四包 sha256 逐文件一致（缺 0 / 不一致 0）、远端 1250 / 8765 HTTP 200（`g6_final_out.txt`）。

### 21.5 本次改动未写 CHANGELOG —— ✅ 已办

> **办结（2026-09-29）**：新建 `l_agent_chat/999.0/src/l_agent_chat/doc/CHANGELOG.md`
> （该包**首个** CHANGELOG），条目 `## v1.0.0 (2026-09-29) — 主循环闭环加固…` 逐条覆盖今天这批改动
> （失败不终止 / 连败上限 / 收尾门不发 error / `except` 的 `result = failed` 修复 / 断线中断标记 /
> 窄屏滑动修复 / 技能·插件市场 / `run_command` 默认 cwd / 工作区刷新 / `MarketApiTest`）。
> **`l_notepad_server` 不重复补** —— 其 `doc/CHANGELOG.md` 的 v3.5.0~v3.5.5 已逐条覆盖 §19.2 的检索改动
> （默认档改 `auto`、判据 25/1.0、RRF、导航降权、块长归一、`confidence`、`rerank` 默认关等）。

只有 `l_repo_sync_gui/999.0/README.md` 末尾记了「直推部署通道」那一节；
`l_agent_chat` 与 `l_notepad_server` 的 `doc/CHANGELOG.md` 该各补一条（今天这批改动量不小）。

## 22. 复现工具与判据

> 原文标题为「（都在 `d:\Temp\`）」—— **`d:\Temp\` 是当时的机器目录（当时环境）**；本机日志根已迁到 `E:/lugwit_rez/_logs_e`（见 `wuwo/config/config.yaml`），脚本目录随机器，按实际位置找。

| 脚本 | 用途 |
|---|---|
| `ac_ask.py` | 驱动浏览器问 agent。**读 `d:\Temp\ask_q.txt`（UTF-8，一行一条）**；会自动点收尾门的选项（第 1 次「继续做」、第 2 次「可以结束」） |
| `ev_answer.py` | 从事件流取 agent 的最终回答（`done.reply`）+ 工具调用清单 |
| `ev_forensic.py` | 事件流取证：类型分布 / 失败结果 / DSML 残留 / 工具时间线 |
| `test_dsml.py` / `test_notepad_read.py` | 单测（样本取自真实日志，**别用手打的近似样本**） |
| `deploy_verify` 自测 | 四包远端完整性（`missing_files`） |
| `remote_up.py` / `remote_fix.py` | 远端服务拉起 / 补推缺失文件（走 `DeployChannel`） |
| `ac_g1_check.mjs` / `ac_scroll_fix_probe.mjs` | G1 窄屏滑动（375×667 CDP 派发滚动手势，断言 `.au-viewport` 跟滚；含退化引擎**负对照**。输出 `ac_g1_out.txt`） |
| `ac_g237_check.py` | G2 / G3 / G7 回归（`stop": true`、收尾门三路径、`error`；输出 `ac_g237_out.txt`） |
| `ac_g4_check.py` | G4 连续失败上限（输出样本 `ac_g4_out.txt`） |
| `ac_g5_check.py` / `ac_g5_shot.mjs` | G5 断线中断（FIN 断 + 截图；输出 `ac_g5_out.txt`） |
| `ac_g5_rst.py` | G5 **真 RST 强断**（裸 socket + `SO_LINGER(1,0)`，`ac_g5_check.py` 取不到底层 socket 时的补充；输出 `ac_g5_rst_out.txt`） |
| `test_skip_lac.py` | `skip_unchanged` 在 `l_agent_chat` 大包上的探测/跳过实测（`test_skip_unchanged.py` 的同款） |
| `g6_push_align.py` | G6 本地清单驱动的**全量直传对齐**（四包，`skip_unchanged=True`；输出 `g6_push_align_out.txt`） |
| `g6_final_check.py` | G6 四包 `hash_files` + `missing_files` + 远端 HTTP 200 终检（输出 `g6_final_out.txt`） |
| `g6_remote_gate.py` / `g6_remote_gate_turn.py` | 远端收尾门复验（后者按「最后一个 agent 事件起」切片，避免混入历史轮次；输出 `g6_remote_gate_turn_out.txt`） |
| **MCP 真浏览器（`mcp_Playwright`）** | 真 Chrome 端到端实测（**不是**脚本化 SSE）：`playwright_navigate` 开 `http://127.0.0.1:1250/chat` → 新建会话 → 在 `div.composer-input[contenteditable="plaintext-only"]` 里输入 → `playwright_click` 点 `button:has-text("…")` 回答收尾门 → `playwright_screenshot` 留证 → `playwright_get_visible_text` 读回答。**调用前先读 schema**（`c:\Users\wb.fengqingqing\.trae-cn\mcps\...\mcp_Playwright\tools\*.json`） |
| **`ac_browser_e2e_out.txt`（证据）** | 「MCP 真浏览器端到端实测」的原始证据：会话 id / 截图清单 / 成对判据 / 工具失败 / 行为链 / `run_command` 根因 / post-restart 复验 / `format ` 误伤（§一~§六） |

**事件流路径**：
- **本地**：`D:\Temp\Log\.l_agent_ws\events\<session>.jsonl`
  —— ⚠️ **`D:\Temp\Log` 是当时的日志根（当时环境）**；本机日志根现为 `E:/lugwit_rez/_logs_e`（见 `wuwo/config/config.yaml`），实际取 `<存储根>/.l_agent_ws/events/`。
- **远端**：`D:/TD_Depot/l_agent_ws/.l_agent_ws/events/<session>.jsonl`
  —— 远端 `l_agent_chat` 的工作区根 = `D:/TD_Depot/l_agent_ws`（取自 `/api/settings` 的
  `workspace_dir`），**不是** `D:\Temp\Log`；查错路径会得出「远端从没跑过 agent」的**错误结论**。（此 `D:` 为远端机器，**仍有效**）

—— **每个 SSE 事件都即时落盘、无损**（见 `events.py` 头注释），所以它是排查的唯一权威来源；
界面上的显示可能滞后或被前端折叠，**不要拿界面当证据**。

> ⚠️ **`seq` 不是全局唯一键**：`wuwo svc restart l_agent_chat` 后，同一会话文件会**继续追加**新回合，
> 但 `seq` 从 1 重新计数（实测：旧回合到 245，新回合 1~147，同文件共 392 事件）。
> 事件**顺序**仍正确；若要按 seq 定位，须先按「最后一个 `agent` 事件」切片（同 `g6_remote_gate_turn.py` 的做法）。

**关键判据速查**：

```
stop": true                   应为 0     （工具失败不该终止整轮）
收尾前请用户确认               非 0      （收尾门真的走到了）
ask_required / done           成对       （问了几次、最后放行）
"type": "error"               应为 0     （不该有假错误）
DSML                          应为 0     （方言泄漏）
```

## 23. 红线（别碰）

1. **远端部署别用「git 差集」推跨版本线的东西** —— 今天就是这么把 8765 推挂的。
   要走**全量直传**（本地清单驱动），它的安全性来自「远端独有文件不会被覆盖或删除」。
2. **别覆盖远端 `.env`** —— 它是远端独有的，全量直传不会碰它，手工 rsync / 覆盖式同步会。
3. **远端 33 篇语料与本地不是一套**（本地 2548 篇）—— 代码对齐 ≠ 数据对齐，重建索引属动数据，先拍板。
4. **改 `.bat` / `.cmd` / `.ps1` 必须 CRLF + 无 BOM + 仅 ASCII**；`.md` 同目录保持一致的换行符
   （`Rez-Docs/` 是 CRLF，`rez-package-source/<包>/999.0/README.md` 是 LF —— 各自一致即可）。
5. **`.py` 改动不热重载**，必须 `wuwo svc restart <别名>`；服务操作一律走 `wuwo svc`，别 `wuwor` 直启。

---

## 24. 附录：老路径（`D:`）处置汇总

本机（开发机）仓库已从 `D:` 迁到 `E:\lugwit\trayapp`（`wuwo/config/config.yaml` 里日志根也已从 `D:/Temp/Log` 改为 `E:/lugwit_rez/_logs_e`）。原文中的 `D:` 路径按下列结论处置 —— **不适用的保留原文并加注「当时环境（已失效）」，不删除**：

| 原文路径 | 性质 | 结论 |
|---|---|---|
| `D:\TD_Depot\Software\Lugwit_syncPlug\lugwit_insapp\trayapp\rez-package-source\`（第二部分开头「全部路径」） | 本机旧仓库路径 | ⚠️ **当时环境（已失效）** —— 本机现为 `E:\lugwit\trayapp\rez-package-source\` |
| `D:\Temp\` 、`D:\Temp\Log`（§22 复现脚本与本地事件流） | 当时的机器目录 / 本地日志根 | ⚠️ **当时环境** —— 本机日志根现为 `E:/lugwit_rez/_logs_e` |
| `D:\TD_Depot\l_agent_ws\.l_agent_ws`（§2 `workspace-status` 示例的 `local_root`） | 机器相关的示例值 | ⚠️ 仅示例；实际取该机「存储根」下的 `.l_agent_ws` |
| `D:\...\trayapp`（§19.1 `run_command` 复验里的 `cwd="D:\...\trayapp"`） | 当时的运行时 cwd（证据原文） | ⚠️ 当时环境；保留为证据，勿照抄 |
| `D:/TD_Depot/l_agent_ws/.l_agent_ws/events/<session>.jsonl`（§22 远端事件流） | **远端**（D: 机器）工作区 | ✅ **仍有效** —— 远端根就是 `D:\TD_Depot\Software\Lugwit_syncPlug\lugwit_insapp\trayapp` |
| `c:\Users\wb.fengqingqing\...`（§22 MCP schema / 截图） | 本机用户目录（C:） | 机器相关，非本仓路径，随机器 |

---

# 第三部分　默认智能体由流程图驱动（2026-10）

## 25. 目标与现状

把默认智能体「无调用收尾」从纯硬编码护栏改为**流程图驱动**（`flow_engine.run_flow`），
图可可视化编辑。**现状一句话：`FLOW_ENABLED` 默认开，图来自
`~/.lugwit/l_agent_chat/flows/default.json`（用户目录优先 → 内置 `default_flow()` 兜底），
加载/执行失败自动回退硬编码护栏级联，不卡死对话。**

## 26. 改动点

| 文件 | 改动 |
|---|---|
| `config.py` | 新增 `FLOW_ENABLED`（默认 `1`）、`FLOW_NAME`（空 = `default`）；对应环境变量 `FLOW_ENABLED` / `FLOW_NAME` |
| `app.py` | 导入 `flow_engine` / `flow_spec`；循环前 setup（读图、定位无调用入口、注册护栏闭包/处理器）；`if not calls:` 分支接入 `run_flow` 子流程 |
| 护栏 | 硬编码级联**保留作兜底**（图不可用时的回退路径，不再作默认路径） |

可视化编辑入口：`l_mindmap_mmd`（:8110）`/flow/default`，编辑器与运行端共用同一份 flows 目录，
改图即生效（详见 `Rez_pkg/l_mindmap_mmd.md`）。

## 27. 验收

- 默认智能体行为与硬编码等价（default 图复刻原护栏结构：branch_calls 分流 → branch_goal 分流 →
  goal_nudge / gate 链 → end）；
- `FLOW_ENABLED=0` 或删掉 default.json 时回退硬编码，对话不卡死；
- 图在编辑器里改（拖动/加节点）保存后，运行时按新图推进。

---

# 第四部分　2026-10-05 / 10-06 之后（**已另有权威文档，本文只留指针**）

第三部分之后这条线又走了两轮，**别拿 §25–§27 的"现状一句话"当最新**：

| 主题 | 权威文档 |
|---|---|
| 图重画成 16 节点「一张图管一切」（熔断/停止/步数用尽走 `end_abort`；门节点 `params`、图级 `params`；`default` 与 `default_intelligent_agent` 共用 `default_intelligent_agent.json`，`default.json` 已改名）+ 对话里四层可见（路线条 / 徽标 / `/flow` 面板 / 轨迹回放）+ 沙箱自我验证（路由回放 / 演练模式 / 对话回放，`optimize_agent`） | `Rez_pkg/流程图智能体_图驱动控制流.md`（含 §3.0 / §3.1 / §7.2 / §7.3 / §10） |
| **对话按 `.code-workspace` 工作区隔离**（`sessions/<工作区 key>/`，归属自动判定，删掉"切换来源"）、发送队列（`↪`引导 / `⚡`立即发送 / `✎`编辑 / `×`删除；停止 = 真中断）、**「✎ 重新编辑」提问 = 开新分支**（与重新生成同一套分支树 `session_branches`） | `l_agent_chat使用指南.md` 的「会话管理」「消息操作与发送队列」 |
| **耗时治理（2026-10-07）**：① 工具清单缓存**过期不阻塞**（stale-while-revalidate，冷建 5.56s → 过期后再取 0.004s）；② `notepad_search` **结果缓存 + 启动预热**（首搜 8.14s → 0.31s，同参重查 0.00s）。**同轮实测**：同题 89.7s → 62.4s，但只有 ① 那 1.7s 能归因，其余是模型行为波动 | `l_agent_chat使用指南.md`「工具按需声明 / 工具清单缓存：过期不阻塞」；判据见包内 `doc/CHANGELOG.md`「耗时治理」两节 |
| **护栏与工具失败（2026-10-07）**：重复读护栏**作答期的「内联工具调用兜底」通道也走同一套**（此前是护栏外后门，不跑 PreToolUse、不进计数表），命中提示列出**已读区间**；工具失败结果渲染成「执行失败」而不是误导性的「(空文件)」 | `Rez_pkg/流程图智能体_图驱动控制流.md`（护栏覆盖范围）；包内 `doc/CHANGELOG.md` |
| **`ask_user` 多选（2026-10-07）**：`multi=true` → 候选变**勾选条目 + 「确认」才提交**（点候选不再立即提交），多选以 `；` 连成一条；`multi` 随 SSE / 落盘痕迹 / live 轮次都带，刷新不退回单选 | `l_agent_chat使用指南.md`「`ask_user`」与端点表 `POST /api/ask-answer` |
| **两条提示词实验片段**（`planner_batch_hint` / `answer_write_tight`，2026-10-07）：**受控 A/B 判无效故默认关**（隔离实例 4 臂 × 2 题 × 2 样本；并批那条还更慢）。留代码 + 运行期开关 `POST /__dev__/prompt-flags` | `l_agent_chat使用指南.md` 设置项表那两行；包内 `doc/CHANGELOG.md`「耗时治理第二轮」 |
