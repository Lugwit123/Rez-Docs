# l_agent_chat 会话存云与工作区改造

> **性质**：改造交接文档。已完成的部分已逐项验证（全量回归 + 浏览器实测），未完成的部分写明了确切下一步。
> 状态截至 2026-09-21。相关设计参照 `Depot库与工作区方案.md`（标注为「未实施/待评审」）与已实现的 `l_notepad` 那套。
> 本文的「实测」结论均以 `127.0.0.1:8080/baidu` 的真实服务为准，**推翻了早期几处按通用 P4 知识做的假设**，
> 也已订正本文自身写错过的两条（见 §4 末尾「订正」）。

## 进度清单（TODO）

图例：`[x]` 已完成并验证 ／ `[ ]` 未完成。明细见下面的编号小节。

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

## 1. 目标

把会话存储从「本地为准 + 异步推云」改成 **云（百度云 Depot）为真源，本地 `.l_agent_ws` 作为版本库的工作区（工作副本）**；
本地与云端不一致时**用状态显示**，不自动让任何一侧覆盖另一侧。

模型直接照 **Perforce**：depot ↔ workspace(client view / maps) ↔ `status` / `pending` / `have` /
`reconcile` / `sync_plan` / `sync_done`。

## 2. 已完成的

| 项 | 位置 | 验证 |
|----|------|------|
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
|------|------|
| **本地** | `local_uploads()` = `{rel: 会上传的那份 bytes}`（设置是**脱敏后**的 dump —— 拿原文件比会把 `settings.json` 永远判成「已修改」） |
| **云端** | `cloud_manifest()` = `list?dir=` 递归（head 上真实存在的文件 + rev/大小/md5/锁） |
| **台账** | `load_ledger()` = 上次推/拉成功后记下的 `{sha, rev}` |

| 情形 | 结论 |
|------|------|
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
|------|--------|------|
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
|------|------|------|
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
|------|------|----------|
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

## 11. 遗留 / 待判断

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
|----|------|
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
