# l_mindmap_mmd —— 脑图 / 流程图编辑器（mm.js 迁移 · React Flow）

> 面向 AI 助手的包文档。编辑、排障本包功能前先读本文件；涉及 `.solo` 守卫 / 热重载 / 反代的
> 机制细节见 `solo_单实例守卫模式.md` / `src_hot_reload_源码热重载与主页常驻.md` / `Nginx反向代理机制.md`。
>
> ⚠ 本包的服务上跑着**两套**编辑器，别混：`/flow/<名字>` = **agent 流程图**（本文）；
> **`/pygraph/<名字>` = 节点 python**（`l_node_engine` 引擎，跑任意 python 对象/函数/类）——
> 那一套的用法、执行面开关、白名单与坑见 **`l_node_engine.md`**。

## 这是什么

`l_mindmap_mmd`（端口 **8110**）是本仓库的脑图 / 流程图可视化编辑器，也是
`l_agent_chat` 默认智能体流程图（`default_intelligent_agent.json`，2026-10-04 由
`default.json` 改名）的**可视化编辑入口**：
图和运行端（`l_agent_chat/flow_spec.py` / `flow_engine.py`）共用**同一份 flows 目录**，这里改完即生效。

前端基于 **React Flow**（`@xyflow/react`），构建产物在 `web/dist`，由 FastAPI（`src/l_mindmap_mmd/server.py`）托管。
另有两枚前端依赖：`marked` + `dompurify` —— python 节点图的**注释节点**（`Note`）在画布上渲染 Markdown
（marked 解析、DOMPurify 消毒，见 `web/src/markdown.js`；两步都不能省）。

## 启动 / 服务

```cmd
wuwo svc status l_mindmap_mmd
wuwo svc log l_mindmap_mmd -f
wuwo svc open l_mindmap_mmd
```

- 服务别名：`l_mindmap_mmd_server`（= `python -m l_mindmap_mmd.server --host 127.0.0.1 --port 8110`，
  **只绑本机**：`/api/pygraphs/{name}/run` 是任意 python 执行面，不许对外暴露）、
  `l_mindmap_mmd_server_exec`（同上但 `L_MINDMAP_ALLOW_RUN=1`，**打开执行面**；卡片当前指这条）与
  `l_mindmap_mmd_server_net`（**同一端口 8110，只把 host 换成 `0.0.0.0`**，执行面同样开 —— 见下面「托盘服务」）。
- **🖥 托盘面板 = exec 服务（`/docs`）+ **树状资源浏览器**（读写托盘那台的盘）**
  （2026-10-18 用户口径：「仅仅显示 19527/docs 服务就行了」+「让 19527 为 pygraph 提供访问托盘服务
  所在主机的文件的能力」+「默认授权所有目录」+「请使用树状图来显示才对」）：
  - **① 服务本身**：`l_tray/exec_server.py`（`127.0.0.1:19527`）—— 状态徽标 + 「📄 打开 /docs」。
    ⚠ `/docs` 是 `<a href>`，**只有本机浏览器**打得开；从别台打开 pygraph 时点它会在**你自己那台**
    找 19527（文件读写不受影响，见下）。
  - **🔗 本图 URL（只剩顶栏那个按钮）**：编辑器顶栏的 `🔗` = 复制本图完整地址，地址由 `pyApi.graphShareUrl(name)` 拼
    （**保留页面已有参数** `tray=` / `tid=` / 布局等，只把 `name` 换成当前图 ⇒ 从 `lugwit://` 回跳来的页面复制出去
    会带上「来源托盘 + 令牌」）。托盘面板里那块「🔗 本图 URL」**2026-10-19 按用户口径「删除这个组件」已删**（探针守 `blocks=0`）。
  - **ⓘ 性能信息**（2026-10-19 用户口径：「这里加一个信息按钮，显示托盘服务的性能信息」）：托盘服务标题行那个 `ⓘ`
    点开显示 **TES 进程**的指标 —— 后端（rust / python）、端口 / PID、CPU%、内存 RSS、线程数、运行时长、
    `/health` 往返毫秒、动作数、命令行、发现文件；再点收起，块里 `⟳` 刷新。
    后端 `GET /api/tray/perf`：pid 从发现文件取（Rust 版 = 独立进程；Python 版 = 托盘进程；与本服务同一进程时标 `same_process`），
    指标用 `psutil`（`l_app_ready` 的依赖里有）；**backend 不许猜**：发现文件没写 `impl` 就看命令行里有没有 `l_tray_exec`。
    块里还有一行 **📂 读盘（展开）**：次数 / 累计 ms / 最近一次 ms + 目录名 + 失败次数 ——
    **每展开一个文件夹就更新**（2026-10-19 用户口径：「这里显示读取硬盘层级的，每次展开文件夹都更新」）。
    ⚠ 它是**前端本次页面会话**的累计（`loadNode` 计时），刷新/换图就归零；不是 TES 侧的历史统计。
  - **② 📁 树状资源浏览器**：盘符下拉作根 → `.pyg-tray-tree` 里 `▸/▾` 就地展开/折叠、按层缩进；
    文件行带大小（悬停还有时间），非文本显示 📦 且不可编辑；点文本文件进编辑器 → 💾 存回 / 👁 定位。
    **懒加载**：展开一个目录才要那一层（`/api/tray/fs/list?root=&sub=`）—— 默认全盘也不会一次扫穿。
  - **☰ 树视图 / ▦ 图标视图**（2026-10-19 用户口径：「参考内容浏览器，提供树视图和图标视图」）：
    标题行的 `☰` / `▦` 两个按钮切换（选哪个存 `localStorage["pygraph.trayView"]`，换页面还在）。
    · `☰` 树视图：逐级展开（默认）；
    · `▦` 图标视图：只显示**当前目录**的格子（📁 目录 / 🗂 `.lproject` / 📄 文本 / 📦 其它，名字两行截断），
      顶部有自己的**面包屑**（点一段跳那级）、`⬆ ..` 回上级；进目录用 `curDir`（树里展开目录时也会同步它）。
  - **⬆ 上一级 / ⭐ 收藏**（2026-10-18 用户口径：「看不到回到上移层级的按钮」+「我可以给托盘服务的
    资源管理器的文件或者文件夹添加到收藏」）：
    · 树没有"当前目录"概念，所以「⬆ 上一级」= **折叠最后展开的那层**（`openStack` 记展开顺序，
      栈空时按钮禁用/提示"已在最上层"）；
    · 每行右侧一颗 ☆（目录与文件都能收），点一下进面板里的「⭐ 收藏（N）」区；点收藏项**定位过去**：
      目录 = 逐级展开它，文件 = 展开父目录并打开。存 `localStorage`（`pygraph.trayFav.v1`，本机偏好，
      不写工程、不占磁盘接口）。
    · **树里不画盘符根行**（2026-10-18 用户口径：「第一个文件夹上面是盘符对么，其实没必要存在的，
      我们已经选择了盘符的」）—— 盘符由上面的下拉表达，树直接从它的一级目录开始；
      ⬆ 的展开栈也不再把根算一层。
    · **双击收藏的图 = 打开那张图**（「双击收藏的图要能打开」）：收藏项双击 → `onOpenGraph` →
      编辑器按 `assets[].path` 反查图地址 → `openGraph`；树里的 `.json` 双击同理；
      对不上（不在已登记工程里）就提示先点它的 `.lproject` 登记。
    · **窄面板自适应**（2026-10-18 用户报「缩小面板宽度这个收藏夹组件没有自适应，会被截断」）：
      `.pyg-tray-panel/.pyg-tray-sec/.pyg-tray-fav/.pyg-tray-filerow` 全加 `min-width: 0`，
      行按钮 `flex: 1 1 0`（不是 `auto`，否则按内容撑宽）⇒ 面板压到 160px 也不横向溢出，长名用 `…` 截断。
  - **🗂 `.lproject` = 工程入口**（2026-10-18 用户口径：「托盘服务里面的文件管理器如果遇到 `.lproject` 文件，
    我希望可以直接双击打开这个项目」）：树里 `.lproject` 显示成绿色 🗂（`.pyg-tray-tree-row.is-project`），
    **点它 = 打开这个工程** —— `root` 取它所在目录、工程名 = 文件名去后缀 → `registerProject`（幂等，
    已登记不动它的 `.lproject`）→ 打开 `.lproject` 的 `entry`，没有就打开该工程第一张图
    （`PyGraphEditor.openProjectFromLproject`）。实测：点 `lugwit_agent_a.lproject` → 顶栏切到该工程。
  - **默认全盘可用**（`l_tray/kb_ws.py` 的 `KB_WS_ALLOW_ALL`，默认 `1`）：`kb_ws_roots` 直接回盘符
    （实测这台 9 条：`C:/ D:/ E:/ J:/ K:/ L:/ P:/ Z:/` + 旧白名单），不必先"选目录授权"；
    设 `KB_WS_ALLOW_ALL=0` 回到白名单模式（`kb_ws_grant` 弹托盘目录框）。写只收
    `.md/.txt/.rst/.log`、单文件 4MB。
  - **一层 + 所有文件**：`kb_ws_list(root, rel, depth=1, all=1)`；`all=1` 连非文本一起列
    （每条带 `previewable`）；`depth=0` 仍递归（知识库兼容）。
  - 后端：`GET /api/tray/fs/roots|list`、`GET /api/tray/fs/read`、`PUT /api/tray/fs/write`、
    `POST /api/tray/fs/grant|reveal`（全经 `_l_tray_call` → `POST 19527/run`）。
    ⚠ **端口不写死**（用户 2026-10-18：「这个端口实际上没法固定，比如被别的程序占用」）：
    `_l_tray_exec_port()` 取 `~/.Lugwit/run/tray.json`（托盘起的 exec 服务自己写，含 `port`）>
    `L_TRAY_EXEC_PORT` > 19527；exec 服务侧被占用会**自动往后挪**（最多 20 个，用 connect 探测——
    Windows 的 `allow_reuse_address` 允许重复绑同端口，不能靠 bind 报错判断）。
    前端显示地址与「📄 打开 /docs」优先用协议回跳的 `?tray=host:port`。
    ⚠ 调 `/run` **必须带 `Origin`**：exec 服务把带 Origin 的当网页请求、只放行 `web_actions`，
    而 `kb_ws_*` 只在 `web_actions` 注册过；不带 Origin 会走 `action_registry` → 404（实测踩到）。
    ⚠ `fs/roots` 按本机磁盘过滤不存在的路径；⚠ 老托盘不认 `all` ⇒ 后端自动回退成只列文本文件。
  - 面板里**没有**：注册协议块、托管服务列表、动作清单、跨机工程/授权那套（后端接口都还在，只是不显示）。
  - 自证：探针 `node tools/web_probe.cjs tray`（**16 项**：`/api/tray/local` 在线 / roots 带盘符 /
    面板「托盘服务·在线」/ `/docs` 链接 / `fs/list` 一层 / **是树**（`.pyg-tray-tree` + 根/子目录行）/
    根节点有 caret / 点子目录就地展开 / 点文本文件读回内容 / 面板里只剩这些）。
    ⚠ `kb_ws` 在**托盘进程内**，改它要**重启托盘**（探针对旧托盘判 SKIP）。
    面板「托盘服务·在线」/ `/docs` 链接 / 📁 块 + 盘符下拉 / `fs/list` 一层 / 列表有内容 / 面包屑 /
    点目录能进去 / **面板里没有「本图 URL」块** / `.lproject` 能在树里列出 / **⬆ 上一级能折叠** / **☆ 收藏进收藏区** / **160px 窄面板不横向溢出** / **📍 定位当前图（<8s，高亮到文件行）** / **无盘符根行** / **双击收藏的图能打开** / 面板里只剩这些）。⚠ `kb_ws` 在**托盘进程内**，改它要**重启托盘**（探针对旧托盘判 SKIP）。
    面板「托盘服务·在线」/ `/docs` 链接 / 📁 块 + 盘符下拉 / `fs/list` 一层 / 点目录能进去 /
    面板里只剩这些）。⚠ `kb_ws` 的改动在**托盘进程内**，改完要**重启托盘**才生效（探针对旧托盘判 SKIP）。
    `/docs` 链接 + `target=_blank` / **面板里只剩这一项**）。
    动作族 / `lugwit://` 已注册 / 面板三块 / 托管服务 ≥5 / **旧那套（当前工程·请求访问·远端托盘）不在**）。
    `run` 标不开放）/ 本机-远端分区 / 远端「请求访问 + 未授权」/ 切远端 / 切回本机）。
- 带 `.dev_mod`（`L_DEV_MOD`）：改 `src/**` 会热更自重启；改 `web/src/**` 需 `npm run build`
  （产物进 `src/l_mindmap_mmd/static/dist`）后浏览器**硬刷新**——`main.js` 无版本哈希，易被缓存。
- 前端开发：`wuwo svc` 之外的 `l_mindmap_mmd_web_dev` 别名起 Vite :5175，`/api` 代理到 :8110。
- **开发 ⇄ 构建 两个入口可以互相切**（2026-10-16 用户口径：「右上角导航下拉框加一个开发/构建模式切换按钮」）：
  `NavMenu`（右上角 `...`）里多一项 `🧪 切到开发模式 / 🏗 切到构建模式` —— 同一台机上就是**前缀差一个 `-dev`**
  （`/mindmap-mmd/` ↔ `/mindmap-mmd-dev/`），切的时候保留当前路径与查询。只在反代入口显示
  （直连 `:8110` 没有 `-dev` 那套，不渲染这个链接）。
  ⚠ **性能一律以构建入口为准**：开发那套是 Vite 未打包 + React 开发模式，同一条链实测
  `fire_time` 构建 **8 ms/帧** vs 开发 **146~248 ms/帧**（慢 20~30 倍）。
- **人在环（HITL，2026-10-12）**：图里的 `Ask` 节点会让跑图**挂起等人**。通道 =
  SSE 反向事件 `ask`（`{run_id, node, prompt, options, default}`）+ `POST /api/pygraphs/runs/{run_id}/input`
  （body `{"value": "…"}`，唤醒跑图线程）。前端在**运行日志上方**的询问条里作答（候选按钮/回车/发送）。
  等待上限 `L_MINDMAP_ASK_TIMEOUT`（默认 180s）；超时、客户端断开、执行超时 → **唤醒后让该节点失败**
  （绝不返回空串：审批被跳过却继续跑比报错危险）。对不上号的提交（已答过/已超时/run_id 错）→ 404 + 说明。
- **跨轮状态 / 会话续跑（2026-10-12）**：跑图 body 可带 `{"session": "<会话id>"}` —— 宿主先读
  `pygraphs/_sessions/<图名>/<会话id>.json`（`{name: JSON 值}`），把值灌进对应 `StateIn` 节点的 `default`；
  跑完把 `StateOut` 的输出写回并落盘（`result` 事件里带 `session` / `state_keys`，日志会写一句）。
  配套接口：`GET /api/pygraphs/{name}/sessions`（清单：id/键/更新时间）、
  `GET|DELETE /api/pygraphs/{name}/sessions/{id}`（读 / 清空）。**不带 `session` = 无状态跑一次**（老行为）。
  前端在运行日志头有「会话」输入框（浏览器偏好，存 localStorage，候选来自上面那个清单）+「清空」。
- **卡片缩略图随结果回来（可选，2026-10-16）**：跑图 body 可带 `{"thumbs": true}` —— `result` 事件里
  多一个 `thumbs: {节点id: "data:image/png;base64,…"}`（只含**这轮真跑到**的节点，取的是
  `_cache_renders` 刚写进内存那份小图，不额外渲染）。给编辑器顶部「🖼 卡片动画」用（见下），
  不带这个键就一点都不多算/多传。
- **权威结构校验（2026-10-12）**：`POST /api/pygraphs/{name}/validate`（body 可带 `{"prompt": …}`，
  可传**图文件形状**或引擎 prompt；不带就读盘上的图）→ `{ok, errors, warnings}`。
  实现是引擎的 `validate_structure`：**与执行器共用同一套计划器**（括号配对 / 嵌套限制 / 真环都问
  `_build_loop_plans` / `_build_if_plans` / `_topological_sort`），所以"校验过了 = 真能跑"，
  不会出现"校验说没事、跑起来 GraphPlanError"。编辑器「校验」按钮 = 本地快查（立刻出行内标记）
  + 这条权威校验（日志会分别报两边的条数）。脏图（非对象节点等）不会 500 —— 兜成一条可读 error。
- **轨迹里的分支决策**：`IfStart` 会发一条 `executed{node, if_taken}` —— 轨迹的 `steps` 里就有
  `{"node": "2", "if_taken": true}`（回放时步骤条上悬停能看到"条件为真/假"，运行日志里是
  `🔀 如果 2：条件为真 → 执行区内`）。`else` 三件套（2026-10-16）再补一条同形状的
  `{"node": "<ElseStart id>", "else_taken": bool}` —— 走的哪一支在回放里一样看得见。
- **pygraph 版轻智能体 / 双跑对照（2026-10-12）**：`tools/agent_lite_ab.py` —— 一张只读小图
  （`AIText 规划 → Compare/If → glob → AIText 成句`）跑一轮并把指标打出来，用来给"把智能体控制流
  编成图"取可比数据（Arm B）；Arm A（老 agent）走隔离实例。实测与踩坑见
  `Rez-Docs/Rez_pkg/pygraph版轻智能体_双跑对照.md`（含**节点缓存 × 括号节点**那个 bug 的记录 ——
  2026-10-12 已修：`IfStart` 与循环括号一并排除出增量缓存，带 if 的图连跑两次不再跑坏；
  引擎侧坑记在 `l_node_engine.md` 坑 16）。
- **多步问题集真值机检（2026-10-12）**：`tools/ab_truth.py` —— 给 A/B 对照的三道多步题
  （q2 二跳计数 / q3 写+审批 / q4 跨轮）**现算真值**并判答案对错（`--check-file`，中文答案走文件别走命令行）。
  纯标准库、不花 token；计划 §2.1 的判据靠它落地（"看着像对"不算）。
  采集器侧：`tools/agent_lite_ab.py --q q2`（Arm B 的二跳变体：`glob` 用 `*/*.py` **一层** +
  成句改"分组计数排序" —— 实测 `answer_ok=True` / 9.58s；⚠ 别写成 `**/*.py`：递归整棵树会把
  提示词撑爆，300s 也回不来）与 `--q q4`（**跨轮**：同一个 session 跑两轮、中间放一个新 `.py`
  → "多了几个"的真值是 1，蒙不出来；图里用 `StateIn`/`StateOut`，跨轮状态由宿主存取，跑完自动清会话）
  都已把真值机检接进那一轮（`answer_ok` / `truth_missing` 进 JSONL），并会**显式报宿主的墙钟超时**。
  run 端点 2026-10-12 起支持 **per-request 超时**：正文 `{"timeout": N}`（夹 1..3600，默认仍是
  env `L_MINDMAP_RUN_TIMEOUT`=60s）—— 采集器给长题默认 `--timeout 300`。
  **慢节点（视频出片）2026-10-15 起由编辑器自动带**：`PyGraphEditor.doRun` 按图里的节点算
  `runTimeoutFor()`（`AIVideo` 每个 **900s**、`AIImage` 240s…，60s 打底，上限 3600）→ `pyApi.runGraph`
  的 `timeout`。为什么必须算：默认 60s 会把出片那一轮**直接掐掉**（实测：SSE 跑到
  `视频任务 processing（… 已等 12s）` 之后就是 `执行超时（60s）`；带超时后同一张图正常收尾）。
  ⚠ `AIVideo` 那 900s 是照**免费档要排队**给的：2026-10-15 实测 `agnes` Flash 一颗节点
  "建任务 → 出片"花了 **~9 分钟**（420s 那档会把已经排到一半的任务掐掉）。
  **节点自己报的进度**同轮进了画布：引擎节点推 `executing` + `note`（`ai_nodes._emit_note`，节点侧异步轮询
  每变一次状态推一条）→ 宿主原样转发 SSE → 编辑器把它挂到卡片上（`.pyg-node-runnote`，一行黄底小字
  "⏳ 视频任务 processing（minimax · 已等 30s）"）并在流水里只写**变化**的那一条（每 5s 一条会淹掉日志）。
  对话驱动那一档走 `GET /api/pygraph-live`（agent 侧白名单同轮加了 `note` / `progress`）。
  ⚠ 跑图时**别重建 dist**（`vite build` 改 `static/dist/*` 会让 dev_mod 热重启宿主，把正在跑的那一轮 SSE 掐断）。
- **工程（Project）· 2026-10-13 P0**：**一个工程 = 图目录下的一个大目录**，里面装资产（图 / 以后的
  结构体 / 类）。地址规则：`<工程>/<资产>`（**只一层**；每段仍走 `_FLOW_NAME_RE`，额外禁 `..`）——
  `_ensure_pygraph_name` / `_pygraph_project`；**老的无斜杠图名照旧**（= 根上的资产，向后兼容）。
  接口：`GET /api/pygraphs` 每项带 `name`(地址) / `asset` / `project`；`GET /api/pygraphs/projects`
  列真工程（`_*` 与 `media` / `models` 不算）。存盘会自动 `mkdir -p` 建工程目录。
  ⚠ **路由**：13 条带图名的路由从 `{name}` 升级为 **`{name:path}`**（子路由：`/run` `/validate` `/traces`
  `/ab` `/sessions` …）；**裸路由（`GET/PUT/DELETE /api/pygraphs/{name}`）用自定义转换器
  `{name:asset}`**（最多两段）—— 否则贪婪的 `{name:path}` 会把 `/api/pygraphs/工程/资产/run`
  整个吞成图名（`/run` 那条注册在裸路由之后，实测踩到）。
  实测验过：工程内建图 → 跑出结果 → 列表/工程接口 → **老的 23 张平铺图一张不坏**。
  **结构体资产**（P1，2026-10-13）：`{"kind":"struct","fields":[{"name","type","default"},…]}` —— 与图走
  **同一个** `PUT /api/pygraphs/<工程>/<名>`（`kind=struct` 走字段表校验：重名/坏形状当场 400），
  `GET /api/pygraphs/structs[?project=]` 列出来；**它没有 `nodes`，所以不会被 `/api/pygraphs` 当图列**。
  **按名字引用**（v2，2026-10-13）：图里 `MakeStruct` 只填 `struct: "<工程>/<名字>"` → 宿主
  `_inline_structs`（loader = `_load_struct_asset`，与子图 `_inline_subgraphs` 同构）在内联阶段把字段表
  灌进 `fields`（**JSON 文本**）；引用不到 → 校验回 `struct_ref` / 跑图 400 —— **当场说清**，不等到节点报错。
  **类资产**（P3 v1，2026-10-13）：`{"kind":"class","fields":[…],"methods":{方法名: 图地址}}` —— 同一个 `PUT`
  校验（字段表 + 方法表都要有；方法表的图地址不能空），`GET /api/pygraphs/classes` 列出来（带字段与方法）。
  图里：`ConstructClass(class="工程/类")` 造实例（**实例就是 dict**）、`CallGraphMethod(instance, class, method)`
  调方法 —— 宿主 `_inline_methods` 按「类 → 方法 → 图」把方法图**内联成 `sub`**（方法图里 `Input(self)` 拿实例），
  解析不到 → 校验 `method_ref` / 跑图 400。内联顺序：**结构体 → 类方法 → 子图**。
  **方法图也能单独跑 / 单独存**（2026-10-18）：编辑器里打开类资产会自动进第一个方法，
  地址是**虚拟地址** `工程/类#方法`（`%23`）—— `GET` / `PUT` / `POST …/run` 都认：
  `GET` 回方法图（`params` 用图级默认值）、`PUT` 写回类 JSON 的 `methods[方法].graph`、
  **`run` 走 `_run_virtual_method`**（2026-10-18 补：以前只有 GET/PUT，▶ 一点就 404）。
  跑的时候段切分/引擎区括号/图入参照旧（`EngineStart(engine=muse)` 那套照样生效），
  ⚠ 只是**不支持 session / 增量缓存 / 时间线**（方法图没有独立文件，历史与缓存无处落）。
  **内容浏览器已做**（P0 收尾，2026-10-13）：**底栏标签**「📂 内容浏览器」（`pyTabs.js` 的 `TAB_DEFS.browser`，
  `home:"bottom"`、**排在「📋 运行日志」前面**）按**工程分组**列出三类资产
  （🐍图 / 🧱结构体 / 🧩类），点**图**=打开、点**结构体/类**=往画布落
  `MakeStruct(struct=…)` / `ConstructClass(class=…)` 并填好名字，标题上的 **↻** 手动刷新
  （也在图列表变长 / 换图时自动刷）。**图那一行带「N 节点」+「🔗 引擎」**（`/api/pygraphs` 的 `nodes`/`engines`，
  见下条），工程组头带「· N 项资产」—— 查"这张图多大、跑在哪台"不用点进去。
  ⚠ **只显示登记过的工程**（2026-10-13 用户**第二次**提"内容浏览器不应该显示工程外文件"）：默认工程
  （`""` = 老图目录）与它下面的散目录**一律不进列表**；那个 👁「引擎内容」开关（UE 的 `Show Engine Content`）
  **连同那一组一起删了** —— 点一下就能把一堆散图/散目录放出来，等于默认没做到"只显示工程"。
  根上的图照样打得开：**`?name=<名字>` 地址**或在图上另存进工程（`assetTree` 里 `.filter(([proj]) => proj !== "")`）。
  **右上角导航菜单（2026-10-15 用户口径）**：按钮是「**`•••` + 一枚下拉箭头**」——
  三个圆点居中、22px 大字距；箭头是 **CSS 画的 chevron**（不是文字 `▾`：边缘干净、开合时
  平滑旋转 45°↔-135°，`.nav-arrow`）。菜单最上面是「布局」三件（下面还有一行**状态提示**：
  `已存过全局自定默认摆法` / `还没存过自定默认（复位 = 内置摆法）`）：
  · **↺ 恢复默认布局** —— 复位成"存过的全局默认"（没存过就是内置那份）；
  · **💾 保存当前布局为默认布局** —— 把**现在这个摆法**存成全局默认（之后就复位到它）；
  · **⇤ 清除自定默认（回到内置）** —— 没存过时这颗是灰的（读全局存档里的 `layout.base` 判的）。
  · **分组小标题与横线同一行**（2026-10-17 用户：「这个面板的标题和横线也合并」）：`.nav-sec` 不再是
    "整行 `border-top` 压一条线、文字另起一行"，改成 flex + `::after` 一条线吃满剩余宽度
    （与「打开工程」下拉的 `.pyg-projsel-head` 同一套观感）。自证 `web_probe navmenu` ①″。
  · **菜单底部那两个长列表已删**（2026-10-17 用户：「这两组不该显示」—— 指的是「流程图（N）」
    「Python 节点图（N）」**列表本身**，不是"点了打不开"）：现在菜单只有 运行模式 / 布局 / 窗口 / **页面**
    四组；找图走「🔍 搜资产」/「打开工程」下拉 / 内容浏览器。**「页面 ▸ 🐍 Python 节点图」那颗还在**
    （落地页 = **第一张图**，`listGraphs()[0]`；没图就 `untitled`）。
  · ⚠ 这类图的入口地址一律用 **`?name=` 形式**（`NavMenu.jsx` 的 `graphHref`）：`/pygraph/<图名>` 在图名
    带斜杠（`工程/Functions/x`）时**必然 404** —— `encodeURIComponent` 出的 `%2F` 会被服务端解回 `/`，
    路径变多段，`@app.get("/pygraph/{name}")` 匹配不上（排查这条时先踩的就是它）。
    自证 `web_probe navmenu`：菜单里**零条**「流程图/节点图」列表项 + 页面那颗 `?name=` 且 `GET` 200。
  ⚠ **布局可以有多套**（2026-10-17 用户口径：「我希望布局可以存在多个可以么，保存布局是可以选在保存到默认布局
  还是新建布局，然后每个图的 json 的字段中显示自己使用的布局是哪个，默认都使用默认布局」）：
  - **表在服务端、按用户一份**：`GET/PUT /api/pygraphs/ui-layout` → `{layouts: {名字: 树}, names, default,
    path, saved…}`，落盘 `<pygraphs>/_ui/<用户>/__layout__.json`（`version: 3`，
    `UI_LAYOUT_KEY = "__layout__"`）。老存档（v2 `{layout}`、更早 `{layouts, current}`）读的时候**迁进表**
    （单套 → `默认布局`；老"工作区"各自成一套，用户当年排过的不丢）。
  - **归属在图自己身上**：图 JSON 的 `layout` 字段（`pySchema.normalizeGraph` / `toStorage` **两处白名单都要有**，
    少一处就是"保存一次就回默认布局"）。缺省 = `默认布局`；要的那套被删了 → 落回 `默认布局` 并说一句。
  - **拖动 = 只改内存**（用户口径 2026-10-17「改动算草稿，点保存才写回」+ 2026-10-18「请不要使用草稿布局」）：
    布局变化**不写服务端、也不写本机草稿**（"草稿"这个东西已删）：以前拖一下就往
    `localStorage["pygraph.ui.draft.<名字>"]` 存一份、打开时**草稿优先** —— 结果一份旧草稿会永久盖住服务端那份
    （实测：草稿里 `closed:["viewer2d"]` ⇒ 默认布局里明明有 2D，界面上永远没有，刷新也不自愈）。
    现在：打开页面**一律以服务端那份为准**（没有就内置默认）；没保存的拖动刷新即丢；
    编辑器挂载时还会**一次性清掉**老键（`pygraph.ui` / `pygraph.ui.draft.*`）。
    `💾 保存布局…` 弹出对话框二选一 —— **保存到当前这套**（覆盖，用同一套的别的图一起变）/ **新建一套**（这张图改用它）。
  - ⚠ **「＋」菜单要能滚**（2026-10-17 用户：「添加标签面板菜单显示不了应该添加滚动滑块」）：标签条住在
    `.pyg-tree-cell`（`overflow: hidden`）里，而那颗 ＋ 列的是**所有别的标签**（二十来项，只多不少）
    —— 旧实现 `position:absolute; top:26px` 在"格子矮 / 组贴着窗口底"时，底下几项**看不见也点不到**。
    现在 `PyTabStrip.openAddMenu` 按 ＋ 的 `getBoundingClientRect` 给**视口坐标**（`position: fixed`，
    逃出祖先裁剪），并挑"往下还是往上"、把 `max-height` 夹进可用空间；`.pyg-tab-menu` 加
    `overflow-y: auto`（滚动条样式走文件顶部那套 `::-webkit-scrollbar`，**别再写 `scrollbar-width`**：一写它
    webkit 那几条就全失效）。右键菜单（`.is-ctx`）故意 `max-height: none`（就两条，不该出滚动条）。
    自证：`web_probe.cjs tabmenu`（11 项，含把视口压到 `980×420` 的滚动那条）。
  - ⚠ **几个动作**（全在右上角 `•••` →「布局」）：清单里点一套 = **这张图**改用它（名字立即写进图 JSON）；
    `✏ 重命名当前布局…` → 表里的键 + 当前图的 `layout` 一起换（**别的图还写着旧名字的，打开时退默认**）；
    `✕ 删除当前布局…`（`默认布局` 删不掉）→ 用它的图退默认；`↺ 复位到已保存` = 丢掉**没保存的拖动**、回到这套上次存的样子。
    「自定默认（`base`）」那一套**已删掉**（`usePyLayout` 里 `saveAsDefault`/`clearDefault` 一并删除）。
  - ⚠ **换布局不算"图内容改动"**：`PyGraphEditor` 的脏标记用 `snapOfGraph()`（`toStorage` 后**摘掉 `layout`**）
    比 —— 否则换个布局整个图就被标成"未保存"（红按钮）。
  - ⚠ **写 `layout` 只改盘上那一项**：`persistGraphLayout()` 先 `getGraph` 读回盘上那份、只补 `layout` 再存
    —— 直接拿内存 `graph` 存会把用户**还没保存的编辑**一起写进去。
  - **`localStorage["pygraph.ui"]` 彻底退场**：`usePyLayout` 传 `persistLocal: false`，编辑器也不再存按名字的草稿
    （2026-10-18 用户口径「请不要使用草稿布局」）—— 布局只有"服务端那份"和"内存里没保存的拖动"两种状态。
  - ⚠ **三个坑**（都实测踩过）：① 弄乱布局**别用"拆分"**（新组是空的、会被 `prune` 剪掉 → 跟存的那份对不上），
    用**折叠**；② **判据别读早**：写盘是异步的，"复位完立刻读"会拿到上一份；③ **开图时别用本地布局去盖存档**
    （老实现 `.finally` 无条件推一次，那一刻 `tabsRef` 还是**套用存档之前**的本地布局）。
  ⚠ 布局**跟图无关的那一半仍然成立**：一套布局被多张图共用，改它（保存到当前这套）所有用它的图一起变；
    但**用哪套**是每张图自己的（所以"切图不跳"只在使用同一套时成立）。
  ⚠ **页面跑着旧包**这类事故（2026-10-15 用户报"能保存无法恢复"，根因之一就是页面还是旧包）：
  工具栏那枚 `🕒 <构建时间>` 是"这页是不是旧 bundle"的判据；**并且**编辑器每 20s 探一次
  `/api/pygraphs/ui-version`，发现产物的 `built_at` 变了就弹一颗 **`⬆ 界面已更新（…）· 点这里刷新`**
  （`.pyg-newui`）—— **刻意不自动刷新**（画布上可能有没保存的改动），只提示由人点。
  自证：`web_probe.cjs navmenu` 里用 `page.route` 把 `ui-version` 打桩成"换版"再看它弹不弹。
  实现跨两个组件：菜单是**全局件**（`NavMenu.jsx`，main.jsx 挂的）拿不到编辑器的 `tabs` / 布局表，
  所以它只 `document.dispatchEvent(new CustomEvent("pygraph:layout-use|layout-save|layout-rename|layout-delete|reset-layout",
  {detail:{name}}))`，编辑器监听后自己动手（清单从 `window.__pygLayoutState` 读，编辑器每次渲染后挂）。
  **每个动作都会打日志**（"已保存布局 X / 已新建 X，这张图开始用它 / 已删除 X"），写不上也会**明说**
  （本机直连写接口要登录态，以前静默 `catch` → 表现成"能保存但刷新就没了"）。**不在 pygraph 页时全置灰**。
  **💥 爆炸（**只拆当前这一页**）**（2026-10-19 用户口径：「每个面板的右键能不能添加爆炸模式，可以平铺面板内的组件」
  → 澄清「加在标签的 label 的右键中」→ 再澄清「**不是爆炸当前标签组，仅仅是爆炸当前标签页**」+「把这一页内部区块拆开平铺」）：
  **标签 label 的右键菜单**里有「💥 爆炸当前标签页（拆开区块）」/「▦ 收拢（合回这一页）」（组头 `▾` 菜单里也有同一项，
  炸的是 `active` 那张）。做法：只给**当前这一张**标签的 `.pyg-tabbody` 加 `is-exploded`，CSS 用
  `display: grid` + 子级 `display: contents` **穿透一层包装**，把面板根 div 的直接子元素各占一格 ——
  不需要每个面板配合；**组里其它标签一个都不动**（标签总数不变）。
  状态是**布局属性** `group.explodeKey`（存标签 key，跟布局存档一起保存；该标签被删/搬走会自动收拢）。
  ⚠ 独占面板（`canvas` / `viewer3d` / `viewer2d`）不能拆，给「⛔ 独占面板…不能拆开平铺」占位
  （白名单 `PyLayoutView.jsx` 的 `EXPLODE_BLOCKED`）。自证：探针 `node tools/web_probe.cjs explode`。
  **每个标签组的最小宽**（2026-10-19 用户口径：「属性标签组是不是有一次约束自身最小宽度的功能，其他标签组也得有」）：
  `useGroupMinWidth` 原先**只量组头**（保证标签 label 不被省略号吃掉），标签少的组下限就很小、一拖窄面板体就被压扁；
  现在取 `max(组头宽 + 24, 组内标签的 TAB_MIN_W 下限)` 写到 `.pyg-tree-cell` 的 `min-width`。
  下限表是 `PyLayoutView.jsx` 里的 `TAB_MIN_W`（palette 195 / tray 215 / props 230 / chat 260 …；
  画布 / 3D / 2D 是独占视图，**不给下限**，可以随便缩）。实测各组：225 / 316 / 280 / 474 / 168 / 375 px。
  ⚠ **反代入口（`:8080/mindmap-mmd/…`）下三件全置灰**（2026-10-16 用户报"无法使用"）：判"在不在 pygraph 页"
  的 `currentRel()` 原来拿 `location.pathname`（`/mindmap-mmd/pygraph`）去和 `APP_BASE`（**完整 URL**
  `http://host:8080/mindmap-mmd`）比前缀 —— 永远比不中 → `rel` 带着反代前缀 → `isPyGraph` 假 → 三件 `disabled`
  （表现是"点了没反应"，一眼看不出根因）。修法：另算一个**路径前缀** `APP_PATH_PREFIX`
  （`new URL(APP_BASE).pathname`）再去掉；探针**跑两遍**验这条（直连一遍 + `MM_BASE=http://127.0.0.1:8080/mindmap-mmd` 一遍）。
  自证：`node .../web_probe.cjs navmenu`（**34 条**：`•••` 字号 / 箭头会转 / 分组标题横线 / 布局四件 +
  清单与 ◉ / 状态提示（这张图用哪套 + 有未保存改动）/ 图 JSON 里 `layout` 的读写 **（新建 → 切回默认 →
  改名 → 删除）** / **拖动只改内存**（等 2.2s 服务端那份没变）/ 保存覆盖后服务端真存下 / 复位回已保存 /
  另一张图仍用默认 / 换版提示条；⚠ 场景改的是**用户的布局表** ⇒ **跑前备份、跑完还原**）。
  - ⭐ **顶栏「打开工程」下拉**（2026-10-13 用户口径：以前是**图名**列表、点一下开那张图）：
    现在列**登记过的工程**（`/api/pygraphs/projects`，读的是 `projectMeta`）；选一个 = **底栏「📂 内容浏览器」
    切到它**（`tabs.activate("browser")` + `setBrowserAt({proj, path:""})`）——**不打开任何图**，图在浏览器里点着开。
    当前图不在任何登记工程里（根上的图）时多一个 `（根上的图 · 不在工程里）` 占位项（免得下拉显示在骗人）；
    末尾是 **`＋ 新建工程…`**（2026-10-13 用户口径：原来是「＋ 新建图…」）—— 选中它会**切到内容浏览器
    并打开"新建工程"表单**（工程名 + 工程目录，`setNewProj("")`；表单本体 `.pyg-proj-new` 就住那一格里）。
    ⓘ 同一轮还把顶栏那个 **「另存为…（图名）」输入框删了**（2026-10-13 用户口径）：图名看地址 `?name=`／
    内容浏览器里选中那行；**改名走右边的「存为…」**（它本来就干这件事）。
  - ⭐ **两组 + 一条横线（2026-10-13 晚；第二组当晚又改过口径）**：① **工程**（按 `updated_at` **最近优先**）；
    ② 一条**禁用的分隔项**当横线（原生 `<select>` 画不出真 `<hr>`，用 `────────────────` 占位）；
    ③ **最近打开的图** —— 读**工程文件里的 MRU**（`.lproject.recent`，写入口 `POST /api/pygraphs/projects/recent`，
    打开一张已落盘的图时由前端记一条、最近的在前、截 20 条）。**它原来不是这个**：那版是"当前工程里的图"
    按 `updated_at` 倒序 —— 图一多就把下拉淹了，而且"最近改过"≠"我最近在用"（用户口径要的是后者）。
    MRU **跟工程走**（换机器/换浏览器打开这个工程，常用的还在前面）。标签 `🐍 <工程内相对路径>（N 节点）`；
    选中它 = **直接打开那张图**（值前缀 `__graph__:`，走 `openGraph`）。
    这一组跟着**当前图所在的工程**走（`curProj`），不是"浏览器现在停在哪"；对不上资产表的条目（图被删了）
    直接不列（别让下拉点出 404）。    地址两种写法都认：规范地址 `工程/Functions/x` 与别名 `pro/Functions/x`（⚠ 这是**资产表反查**的宽容，不是入口地址 —— URL/API 入口写别名一律 400）
    （首段 `pro` = "本工程"）—— 前端拿资产表反查（`assetByAddr`），记的时候统一落**规范地址**。
    默认工程（老图目录）没有工程文件 → 这一组恒空。

### 播放开关（`▶ 动画`，2026-10-14）—— 不摆时间线也能动画

用户口径：「**`Time` 节点不依赖时间线，而是引擎时间**」。于是两层都改了：

- **引擎**（`l_node_engine/graph.py`）：宿主**没喂**时钟时，引擎给**自走的那份** —— 每次
  `execute_graph` 帧号 +1（`_CLOCK_TICK`），`time = 帧/ENGINE_FPS`（24）、`duration = 1.0`
  ⇒ `Time.phase` 每 24 帧扫一遍 0..1。宿主喂了就以喂的为准（时间线那档）。
- **编辑器**：工具栏多一个 **`▶ 动画`**（浏览器偏好 `localStorage["pygraph.anim"]`）——
  开着就按 fps **不停重跑**这张图、**不喂时钟**（用内联图 → 不落盘）。
  三条与时间线播放同款的口径：按 fps 节流、上一轮没回来就跳过、结构错不发。

⚠ 实测（2026-10-14，`demo_bamboo_weave` 2048²、~20 节点）：**0.44 fps** —— 每帧整条下游都要
重算（`Time` 是 `ALWAYS_RUN` ⇒ 它的下游**全都不可缓存**，结构指纹链断在那儿）。所以"动画快不快"
= 这张图多大；要跟手就把尺寸改成**相对父级** + 把「⇄ 接口」的基准尺寸调小（512² 大约快 16 倍）。
**这个数不用自己掐表**：`Time` 的 `fps` 输出（2026-10-15 加）就是实测帧率，直接显示在卡片上。

⚠ **"Time 节点慢"是误诊**（2026-10-16 实测）：`Time` 一帧 **0.01 ms**（算 4 个标量）。
`fps` 个位数 = **整张图**在重跑。512²/18 节点的火焰图，一帧的账单（宿主侧 `execute_graph` +
`_tex_render_png`，与编辑器动画同一条路）：

| 一帧 252 ms 花在 | ms |
|---|---|
| `glow` TexBlur（高斯=3 次方框） | 59.8 |
| `mv2` / `mv1` TexTransform | 35.5 / 34.0 |
| `st2` / `st1` TexResize | 28.8 / 28.7 |
| `bl1` TexBlur | 26.8 |
| `mask` / `mul` TexPixelProcessor | 21.2 / 14.0 |
| 渲染一帧输出 PNG 512²（+ 缩略图 12KB） | 18.8（另算，跑完立刻 `_cache_renders` 那步） |

**三个能动的旋钮**（都量过）：
1. **降尺寸** —— 512²→256² 直接 ÷4（对照：同结构的 `fire_keys` 是 256²，一帧只 57 ms）；
2. **走 GPU** —— 「▶ 动画」在整链已迁移时 3~6 ms/帧（`l_tex_nodes.md` §9）；时间线那条**永远是 CPU**
   （它每帧 POST `/run`），所以 GPU 只帮"不摆时间线"那档；
3. **把逐像素算子接 numba**（`l_tex_nodes/accelerate.py`）—— 双线性重采样那三个接完之后，
   上表这 252 ms → **168 ms**（`TexResize`/`TexTransform` 从 29/35 ms 掉到 <2.4 ms，掉出前 8 名；
   `fire_keys` 同步 57 → 34 ms）。剩下的都是**带宽限制**的向量化代码（前缀和模糊、表达式像素处理器），
   接 numba 没收益 —— 别再往那儿使劲。
   ⚠ 工具：`wuwor l_mindmap_mmd -- python rez-package-source/l_mindmap_mmd/999.0/tools/time_perf.py [图名]`
   —— 就是上面这张账单的出处（`Time` 单节点 / 整图一帧 + 逐节点排行 / 渲一帧 PNG）。
   两个口径写在它的头部：**关键帧驱动的图量不出来**（不喂那些参数 ⇒ 缓存全复用、报 0 ms）、
   **绝对数受本机负载影响**（见过 60~200 ms 的差，比前后要背靠背跑）。

**三个"fps"别混**（用户口径 2026-10-16：「哪些显示数据能更加准确，能知道画布真实的性能」——
以前只有第一个数，于是"动画 4 fps 而读数 60 fps"看着像撒谎）：

| 显示在哪 | 量的是什么 | 口径 |
|---|---|---|
| 画布右下 `N 重绘` | **rAF 重绘率**（`useFps` 数 requestAnimationFrame 次数） | 上限 = 显示器刷新率（60Hz 屏恒 60）。**跟动画多快无关**，只说明"画布还活着" |
| 画布右下 / 2D 取景器头上 `动画 N fps` | **画面到底多久换一次**（端到端） | 相邻两帧"起点到起点"的 **2 秒窗口平均**（与引擎那份同一口径）⇒ 含跑图 + 出图（PNG 编码）+ 传输 + 节流 + "上一帧没回来跳过的那些拍"。**只在播放时有值，停播即清空**（留一个上次的数比不显示更糟） |
| `Time` 卡片上的 `fps` 输出 | **引擎侧**"这张图被重跑了多快"（`graph.measured_fps`：2 秒窗口跑了几拍） | **不含**出图/传输/解码 ⇒ 与上面那个对不上是正常的，差的就是那几步 |

**「WebGPU 版 vs numba 版差多少」**（用户口径 2026-10-16 问的）—— 两条量具、同一份 prompt：

| 量法 | 命令 |
|---|---|
| GPU 那一列（页面里 `runGpu`，**等 `onSubmittedWorkDone`**） | `node rez-package-source/l_mindmap_mmd/999.0/tools/web_probe.cjs gpucpu` |
| CPU/numba 那一列（同一份用例喂引擎） | `wuwor l_mindmap_mmd -- python .../tools/time_perf.py --gpu-cpu` |

实测（1024²，热身后 min-of-3，ms）：

| 用例 | GPU | CPU(numba) | 差 |
|---|---|---|---|
| **地板**（64² 一个噪波） | 3.8 | 0.12 | —— 这是"一次派发+同步"（GPU）/ "一次跑图"（CPU）的固定开销 |
| fbm 5 层噪波 | 2.5 | 33.2 | **≥13×** |
| 高斯模糊 r=8 | 3.1 | 115.1 | **≥37×** |
| 方向模糊 n=16 | 3.4 | 13.5 | **≥4×** |
| resize→transform→warp 链 | 3.6 | 7.4 | **≥2×** |

⚠ 三条口径（不写清就会读出错误结论）：
1. **GPU 那列全压在"地板"上**（~3ms 的提交+同步往返）⇒ 1024² 这点算力**还没把 GPU 撑开**，
   比值只能写"≥"（分母被地板抬高了）。往 2048² / 更多采样上走才会拉开（旧实测：2048² 三层 fbm ≈3–6ms）。
2. **只量"计算"**：GPU 那档不落盘、不走 PNG；CPU 那档还要 **`_tex_render_png`**（512² ≈ 19ms）
   + 传输 + 浏览器解码。谈"动画多快"得按端到端算（下面那张）。
3. 两边都得**热身后取 min**：GPU 首次含 shader 编译、CPU 首次含 numba 编译（那次能到几秒）。

**同一张真图（`fire_time` 512²/18 节点）的端到端一帧**：
- **GPU**：`work` **27 ms**（其中真·GPU 计算 ~3ms，其余是内联 prompt + 22 张卡片的状态更新 + 提交）
  ⇒ 节流 24 fps 下实测 **20 fps**（读数里就是它）；
- **CPU**：引擎一帧 **165ms** + 渲 PNG **19.9ms** + 传输/解码 ≈ 10–20ms ⇒ **≈200 ms/帧**（≈5 fps）
  ⇒ 端到端 **≈7.4×**；只看计算则是 **3ms vs 185ms ≈ 60×**。
⇒ 结论：**GPU 那条早就不是瓶颈了**（3ms），瓶颈变成"每帧重建/更新 22 张卡片 + 内联 prompt"（27ms 里的大头）
—— 2026-10-16 当天就把它做成了**增量**（见下节）。

### 「能到 100fps 甚至突破显示器刷新率吗」（2026-10-16）

**先把"算"和"看"分开，再看实测（见下一节的表：目标 120fps 实测只有 ~61fps）**：

| | 上限 | 为什么 |
|---|---|---|
| **计算速率**（图被推进多快：`Time` 前进、下游重算、烘焙/导出吃到的东西） | **代码上不受刷新率限制**（目标帧率可填到 **480**）；但**实测受每帧总开销限制**（2026-10-16 量到 ~61fps） | 2026-10-16 起 `▶ 动画` 那条腿**改成定时器驱动**（原来是 `requestAnimationFrame`，跟 vsync 走 ⇒ 上限就是 60/120/144） |
| **屏幕能看到多少** | = 显示器刷新率（60/120/144…）+ 合成器 vsync | 物理规律，改不了：面板一秒钟只亮那么多下 |

读数里**两个数并列**就是这件事：`60 重绘 · 动画 143 fps`（前者 = rAF 重绘率 = 你能看到的，
后者 = 端到端更新速率 = 图真被推进的速度）。所以"突破物理限制"只在**计算/导出/驱动下游**这个意义上成立：
`Time.phase` 真的按 143fps 在走，只是屏幕每秒给你看 60 张。

量具（`node .../web_probe.cjs fps1024`）：1024² 的 GPU 图先按目标 30fps 量一次**一帧成本**
（实测 **8ms ⇒ 算力上限 ~125fps**），再把目标设成 **120fps** 量一次 —— 判据是**实测 >70fps**
（headless 的 rAF 约 60 ⇒ 超过它只可能是真按定时器在跑 ⇒ 证明已脱离刷新率上限）。
⚠ 定时器那条腿**每一拍仍然"上一帧没回来就跳过"**（`gpuBusy || runRef.current`）⇒ 一帧比间隔还贵时，
实测帧率自然掉到 `1000/一帧` 那个上限，不会堆积。

**结论（2026-10-16 实测，别只看口号）**：循环**代码上**已经不在 rAF 上（改定时器驱动，目标帧率可填到 480），
但**实测达不到 100fps**，而且离得不算近：

| 量什么 | 数字 |
|---|---|
| 真图 `fire_time`（18 节点，改成 **1024²** 之后）**一帧 work** | **7~8 ms**（max 10~20）⇒ 理论算力上限 **~125 fps** |
| 目标 **24fps**（默认）→ 实测 | 19.7~22 fps（跟得上 ✓） |
| 目标 **60fps** → 实测 | **49.5 / 49.8 fps**（**跟不上** ✗，节拍被那笔固定开销吃掉） |
| 目标 **120fps** → 实测 | **61.4 fps**（不节流时反而比"按 60 排"高） |
| 12 节点的合成图（三条 fbm+高斯+变换链）目标 120 | 69.6 fps |

⇒ **60fps 不稳**（按 60 的节拍只到 ~50；不节流能擦到 ~61）。差在哪：每帧除了那 7~8ms 的 `work`，
还有 **8~12ms 的调度 + 呈现往返** ⇒ 真实周期 16~20ms。⇒ 想稳 60 得把**每帧总开销**压到 ≤16.7ms 以内
（现在是 16~20，正好压在线上）；想上 100 得压到 ≤10ms（`work` 已 8ms 贴着 ⇒ 只能从调度/呈现那笔里省，
或让一帧算得更少）。

⇒ 差距在哪：每帧除了那 8ms 的 `work`，还有 **~8ms 的调度 + 呈现往返** ⇒ 周期 ~16ms ⇒ ~61fps。
**"算"与"刷"解绑是做到了**（读数里两个数并列：`61.4 fps` vs `重绘 60`），但**每帧的固定开销**成了新的天花板。
要真上 100fps，得把**每帧总开销**压到 ≤8ms —— 而 `work` 已经 8ms 贴着了 ⇒ 得从调度/呈现那 8ms 里省
（或让一帧算得更少：降尺寸/减节点）。
⚠ 屏幕那半**永远**破不了（`重绘` 就是它）；能破的只是"图被推进的速度"（烘焙/导出/驱动下游吃这个）。

量具：`set MM_FPS_GRAPH=project_name/demo/fire_time && node .../web_probe.cjs fps1024`
（⚠ 入口地址首段要写**真工程名**；别名 `pro` 只在图内引用里成立，写进 `?name=` 会被服务端 400 —— 见「别名只做图内引用」一节。
不设 `MM_FPS_GRAPH` 就用自造的三条链临时图）。判据只写"**不低于 rAF 重绘率**"——
**别写成"能达到 120fps"**，实测差得远（写死高目标就是假红）。

**每帧相位（2026-10-16 挖开销，`data-frame-phases`）**：`prep=0 run=5~6 pub=0`（ms）——
帧内那 6ms **全在 `runGpu`**（内联图缓存 + 每帧 React 更新搬去 `liveValues` 之后，另外两项实测是 0）。
⇒ 剩下那 7~8ms 全在**帧外的浏览器管线**：React 提交 + 画布呈现 + "定时器只能等渲染步之后才轮到"
（`MessageChannel` 那条路实测更差：postMessage 任务排在渲染之后 ⇒ 反而对齐 vsync）。所以**帧内已经没肉了**，
再快只能让一帧算得更少（降尺寸/减节点）或改"呈现方式"。

**四个目标帧率各量一遍（真图 1024²，可重复）**：

| 目标 | 实测 | 备注 |
|---|---|---|
| 24（默认） | 19.7~22 | 跟得上 |
| **60** | **49.5 / 49.8** | **最差的一档**：目标落在管线量化台阶上（每帧 work 6ms + 帧外 ~8ms ⇒ 真实周期 ~14~19ms，按 16.7ms 排必然被"等渲染步"吃掉） |
| 90 | 62.8 | ✓ 稳过 60 |
| 120 | 61.4 / 72.1 | ✓ 最好一档（抖动 ±） |

⇒ **实操建议**：想要"画面稳过 60"，把「🎞 时间线」的 fps 设 **90~120**（别设 60 —— 那档最吃亏）；
设 24 之外的任何值都只是节流上限，实测速率由上面这条公式定。

**① 取景器不再"每帧重绘"（2026-10-16 用户口径：「第一个你建议取消每帧重绘取景器」）**
以前每帧的链路是：编辑器发布新输出 → `TexViewer` 收到新的 `gpu`/`meta` **对象** → **React 重渲这个组件** →
绘制 effect 才画 → 浏览器再 commit 这次渲染并合成。那笔"React diff + commit + 合成"实测 **~8ms/帧**，
正好是帧周期的一半（帧内只有 6ms）。
现在改成**订阅到新帧就地画**：
- GPU 那条：`subscribeGpu` 的回调里直接 `presenter.draw(out, channel)`,**不 setState**
  （只有"换设备 / 换尺寸 / 换通道数"才 `setGpuState` 走一次 React）；
- CPU 那条：`liveValues.subscribeFrame` 的回调直接 `load()`，解码完**就地 `paintNow()`**；
  `meta`（头部那几个字）与 `loading` 都改成"**只在真变时** setState"（`sameMeta` 比较）。
换通道、换节点、"跑完一轮"这类**低频**刷新照旧走 React（`refreshKey` / `[channel, meta]` 那两条 effect）。

**② `Time` 节点自带 `fps`，动画不再看时间线（2026-10-16 用户口径：「time 节点驱动的我希望能跑的 fps
不受时间线控制，可以在 time 节点修改 fps」）**
`Time` 多了一个 **`fps` 入参**（FLOAT，1~480，默认 24）：`▶ 动画` 那一拍**以它为目标帧率**，
没填（或图里没有 `Time`）才回落「🎞 时间线」那个 fps —— 每拍现读图，在卡片上改一下即时生效。
⚠ 它与**输出**里的 `fps` 不是一个东西：输入是"**目标**"，输出是"**实测**这张图真被跑了多快"。
⚠ `time = 帧/ENGINE_FPS(24)` 那条折算**没改**：填 120 时动画跑得比"实时"快 5 倍 —— 这正是"计算速率"那件事。

量具：`set MM_FPS_GRAPH=project_name/demo/fire_time` + `MM_FPS_TARGET=<目标>` → `node .../web_probe.cjs fps1024`
（会打 `work` 中位 + 相位 + 重绘率；⚠ 地址首段写**真工程名**，别写别名 `pro`）。

### 「项目外不允许有图」= 服务端**硬约束**（2026-10-16）

用户口径：「修改代码，硬性约束禁止运行不在项目中的任何图」。做法（**拦在代码里，不靠文档自觉**）：

- **闸在哪**：`server._assert_graph_in_project(p)` —— 只跑**已登记工程目录里**的图。
  挂在两个"会执行图"的入口：**`POST /api/pygraphs/{name}/run`**（唯一的前端跑图入口，混引擎 / 远端转发**之前**）
  与 **`POST /api/pygraphs/{name}/ab`**（沙箱回放也是执行）。
- **默认根不算工程**：实例数据根下 `<pygraphs>/*.json` 里摊着的图（老图目录那条）**一律 403**，
  报错带**可行出路**（移进工程目录 / 把目录登记成工程）。渲染 / 缩略图 / `/renders` 只是读缓存，**不拦**
  （它们不执行）。
- 实测：项目外的图 `/run` → **403**（`这张图不在任何**已登记工程**里，不能跑…`）；工程内的图 → **200**。
- ⚠ **连带改动**：探针（`tools/web_probe.cjs`）的临时图以前写在默认根 ⇒ 会被自己的护栏打死。
  现在裸名一律映射成 **`<TMP_NS>/tmp_probe_x`**（`TMP_NS` = **运行时**从 `GET /api/pygraphs/projects` 取第一条**有名字**的工程 —— 默认根那条 `name` 是空串，不是它；`MM_TMP_NS` 可覆盖），
  **地址与磁盘路径都落在工程里**（跑完照旧删掉）—— 场景代码里那些裸名一个都没改，集中在
  `probeAddr()` / `graphPath()` / `putGraph()` / `dropGraphApi()` / `openGraph()` 这几处。

### 压测案例：22 节点 / 全 1024² / `Time` 驱动（2026-10-16 实测）

图在工程里：**`pro/demo/demo_perf_stress`**（正式名 `project_name/demo/demo_perf_stress`；文件在
`rez-package-source/l_agent_tool/999.0/src/lugwit_agent/demo/demo_perf_stress.json`，
2026-10-16 工程从 `lugwit_agent` 改名成 **`project_name`** —— 名字派生自 `<根>/<名>.lproject`，
而**目录仍是 `lugwit_agent/`**）：
用户口径 2026-10-16「做一个复杂的案例测试一下性能」）。组成：噪波 5 层（fbm×2 / worley（512² 生成再重采样到 1024²）/ perlin / value）、
高斯模糊 ×2（r=6 / 2.5，repeat 无缝）、**方向模糊 ×2（n=32 / n=20）**、域变形链两层、逐像素表达式 ×2、
混合 ×4（overlay / multiply / screen / add）、色阶 ×2 / 反相 / 变换（旋转 + `Time.time` 驱动位移）。

| 腿 | 一帧 | 上限 / 实测 |
|---|---|---|
| **GPU**（`▶ 动画`，`Time.fps=120`） | `work` 中位 **19~25 ms**（相位 `prep=0 run=11~13 pub=0`） | **算力上限 49~53 fps**；**实测 31~34 fps**（`重绘 60` 并列） |
| **CPU**（时间线/宿主那条） | **964 ms**（最小 904 / 最大 1197） | **0.93 ~ 1.0 fps**；出图（1024² PNG+缩略图）**107 ms** |

⇒ 同一张图 **≈ 38~50 倍**。CPU 那条腿的热点（`time_perf.py`）：`sh1` 高斯模糊 404ms（含首帧冷启动）>
`bl1` 高斯 137 > `px2` 逐像素 129 > `px1` 逐像素 110 > `lv2` 色阶 60 > `mx2` 混合 54 > `n1` 噪波 43。

⚠ 两个读数口径（别读错）：
- 探针对这张图的判据**只敢写"真量到了数"**（`fps/work/cap` 都 > 0）：不同图差得很远（18 节点火焰
  49~72fps；这张 22 节点 31~34fps），拿固定 fps 当判据就是假红（踩过：`fps > 44` 在这张图上假红）。
- **重图首帧**要把所有 shader/管线编出来，4s 预热常常还没出第一帧 ⇒ 探针预热已放宽到 **8s**
  （踩过：`work=0` 假红）。

量法：`set MM_FPS_GRAPH=project_name/demo/demo_perf_stress` → `node tools/web_probe.cjs fps1024`（GPU 腿；⚠ 首段写真名，写 `pro/…` 页面 400 ⇒ 探针假红）；
`wuwor l_mindmap_mmd -- python tools/time_perf.py demo/demo_perf_stress`（CPU 腿 + 逐节点热点）。

### 「1024² 能跑 25fps 吗」（2026-10-16，实测）

**能 —— 但只在 WebGPU 那条腿（`Time` 驱动 + `▶ 动画`）上；时间线/CPU 那条腿不能。**

| 腿 | 1024² 一帧 | 结论 |
|---|---|---|
| **GPU**（`▶ 动画`，12 节点：三条 fbm+高斯+变换链 → 混） | **work 中位 8 ms**（max 11）⇒ 算力上限 **~125 fps** | 25fps 的预算（40ms）只用掉 1/5 ⇒ **绰绰有余** |
| **CPU**（时间线：每帧 `POST /run` + 出 1024² PNG + 传输 + 解码） | 单帧 curl **235 ms**；实跑读数是 **1.32 s/帧**（0.8 fps） | 25fps 需要 40ms ⇒ **差 20~30 倍，不可能** |

量具：`node .../web_probe.cjs fps1024`（GPU 那列）+ `fire`（时间线那条腿）+ `tools/time_perf.py`（逐节点）。

⚠ 四条前提（少一条 GPU 那条腿就不成立）：
1. 用 **`Time` 节点驱动** + 工具栏 `▶ 动画`（**不是**时间线 ▶ —— 关键帧要宿主插值，那条腿永远是 CPU）；
2. 这张图**所有**节点都在 `texGpu.js` 的已迁移表里（有一个没迁 ⇒ 整拍退 CPU，见 §九的 `pending` 清单）；
3. **目标帧率**：「🎞 时间线」的 `fps` 就是节流上限（默认 24）⇒ 要 25 就设到 25+（读数里那个"实测 fps"是**真达到**的，低于目标说明帧贵或机器忙）；
4. 机器别同时在跑重活；WebGPU 偶尔抖一下时现在是**退避 5s 自动重试**（不会永远回不去）。

想让手上那张（现在是时间线/关键帧的）也吃到这个数，两条路：**把驱动方式换成 `Time`**（参数接 `Time.phase`），
或者**把基准尺寸降下来**再留时间线。

### 「⏱ 节点耗时」开关 + 徽标挪到卡片左上角（2026-10-16）

用户口径：「加一个开关按钮，可以在节点右上角显示节点消耗时间，**显示在节点顶部，不是内部**；
（已有就改到顶部左上角）」+「没有必要在每帧显示节点是不是 GPU 引擎，一次显示就不要在每帧刷新了」。

- 那个徽标**以前就有**（`duration_ms` → `.pyg-node-dur`），但挂在卡片**标题行里**（占正文位置）——
  现在拆成两件事：**位置**改成浮在卡片**左上角**（`position:absolute; left:2px; top:-19px`，
  `pointer-events:none` 免得压住标题的点击/拖拽）；**开关**是顶部 `⏱ 节点耗时：开/关`
  （浏览器偏好 `pygraph.showdur`，默认开 = 与以前一样），关掉后 `PyNode` 不渲染那个 `span`。
- **播放帧不再刷新它**：以前 `executed` 事件逐节点逐帧 `setLastDur`（每次新建对象 ⇒ 真更新）
  ⇒ **每帧把 22 张卡片全重渲一遍**。现在播放帧里它 ≤0.5s 刷一次、关掉开关时整个不收；
  要看"哪一步慢"停播看即可（那个数本来就只用于这个诊断）。
- GPU 那条的**占位值**（卡片上那句 `WebGPU(512x512)` = "这步在哪条腿跑的"）以前也随每帧发布，
  现在按 `图#节点#文本` 记一笔、**只发一次**（用户口径：一次显示就别每帧刷）。
- 量具：`node rez-package-source/l_mindmap_mmd/999.0/tools/web_probe.cjs durperf`
  —— 同一张图（`fire_time`，GPU 腿）跑两轮采样（开关开 / 关），比 `data-frame-work` 与帧率。

**结论（实测 `durperf`，`fire_time` 512²/18 节点、GPU 腿、各采 10 拍去掉头两拍）**：

| | 一帧 `work`（中位） | 帧率 | 屏上徽标 |
|---|---|---|---|
| `⏱ 节点耗时：开` | **8 ms** | 20 fps | 18 个 |
| `⏱ 节点耗时：关` | **9 ms** | 20 fps | 0 个 |

⇒ **"显示"本身几乎不花钱**（8 vs 9ms、fps 一样）；**贵的是"每帧都去更新它"** ——
改之前 `executed` 逐节点逐帧 `setLastDur`（每次新建对象 ⇒ 真更新）⇒ **每帧重渲所有卡片**：
那笔钱和 `setNodes`/`setViewerKey`/`setFrameStat` 一起，在"动画帧增量更新"那轮被砍掉
（GPU 一帧 `work` 27 → 15 → 11 → **8 ms**）。
⚠ 顺带量出一个**尖峰**：改之前"低频 state"那一批（`setViewerKey` + `setThumbEpoch` + `setVersion`
+ 面板值）会**落在同一帧上**，那一拍 `work` 冲到 **122 ms**（探针里那两拍）。它们**播放帧一个都不需要**
（取景器由 `commitFrame` 驱动、缩略图走 store、卡片高度不因数值变化 ⇒ 不用每帧重建）⇒ 全删，
只留面板值 2Hz ⇒ 尖峰消失（现在 8/8/8/9/10 ms 一排平的）。

### 同一格再标「GPU / CPU」（2026-10-16 补）

用户口径：「我觉得程序纹理图应该显示使用的是 gpu 还是 cpu」+「标在那个节点耗时那里」。
原来只有 GPU 那条腿有痕迹（取景器头上那枚 `WebGPU`、卡片里的 `WebGPU(512x512)` 占位）——
CPU 那拍**什么都不显示**，于是"看着能动、其实全在 CPU"（2026-10-16 那个 0.4fps 就是这种）。

- 徽标（`.pyg-node-dur`）里加一段：`⚡ GPU` = 这一步上次在**浏览器 WebGPU** 上算的（那条腿整图一把提交、
  **不逐节点回报耗时** ⇒ 不写 ms，端到端看画布右下「动画 N fps」）；`⏱ 12 ms CPU` = 宿主跑的那拍。
  两个渲染后端**都要实现**：DOM 在 `PyNode.jsx`，canvas 在 `CanvasSurface.jsx`（手绘，同一句文案）。
- 数据来自新的一本账 `lastBackend`（`{节点id: "webgpu"|"cpu"}`）：
  · **CPU**：宿主那拍按 `executed` 事件逐节点记（**不看 duration** —— 增量复用的那批节点没有耗时，
    但它们的值也是宿主那份）；与 `lastDur` 共用同一个节流闸（徽标关着就不收这本账，省掉整图重建）。
  · **webgpu**：GPU 一帧成功后在 `runGpu` 那支按 `图#版本` **只标一次**，同时把上次 CPU 留给这些节点的
    ms **抹掉**（不然徽标上挂着一个不是这次的数）。逐帧写状态 = 把刚砍掉的"每帧重渲所有卡片"请回来。
  · 悬停里给**为什么没走 GPU**（`texGpu.gpuBlockReason()`：没有 `navigator.gpu` / 哪些节点没迁（带类名）/
    GPU 刚失败正在 5s 退避）—— 那句说明与 `canRunGpu()` **同一套判据**，别只改一边。
- 判据：`web_probe.cjs durperf` 两条 —— 实时+动画（WebGPU 腿）下 24 张卡全 `G:GPU`；停播 + 关实时后
  `▶ 运行`（宿主那拍）24 张全 `C:… CPU`，一个 `GPU` 都不许剩。实测两条都过（`PASS 7/7`）。
- ⚠ 顺带两个坑（同轮踩到、已修）：
  ① 时间线播放那腿**必须自己点开「⚡ 实时」**（播放循环里那句 `if (live && …)` 才发 `/run`；而**切图会把
  实时关掉**）—— 漏了就是 `fire_keys … frames=1` 的**假红**；
  ② 探针里 `.pyg-viewer canvas[data-backend="webgpu"]` 要 `.first()`：底栏那格「🖼 2D 视图」也挂着时
  同页面有**两个**取景器画布，严格定位直接 `resolved to 2 elements`。

### 「参数改动也走 WebGPU」（2026-10-16）

用户口径：「参数改动也走 WebGPU」。原来那条腿**只有动画/实时那一拍**走 WebGPU，改一个参数却回
**宿主 CPU** 跑 —— `lava_flow`（2048²/37 节点）改一下等 **0.8~2.4 s**，等于"改一下等一下"。

- 实现：把动画里那一拍抽成 `PyGraphEditor` 的 **`gpuFrameRef`（`gpuFrameImpl` + 入口）**，两条路共用：
  · 动画：`{cacheTag: "anim-full"|"anim-time", frameStats: true}`（整图 / 只吃 `Time` 的子树）；
  · 改参数：`runLocalForParam` 先 `await doSave()`（**先落盘**，GPU 那条腿不经宿主），再
    `{cacheTag: "param:<节点>", stateNow: true, wait: true}`；返回 `false`（没迁/没 WebGPU/正退避）才走宿主 `doRun`。
- 三个必须一起看的口径：
  · **`cacheTag` 必须区分"这份 prompt 是哪一种"**：内联子图那份缓存只按 `图#版本` 记，而"整图 / 只吃
    `Time` 的子树 / 某节点+上游"是**三份不同的 prompt** —— 不编进 key 就会把上一份端出来（跑错图、还没声）；
  · **排队**：动画一直在占这条腿（一拍几十 ms）⇒ 改参数那拍 `wait` 一次（最多 ~300ms）再跑；
    **动画那一拍自己也要挂进同一条链**，否则"繁忙窗口里等不到"→ 又掉回宿主 CPU（实测踩到）；
  · `gpuBusyRef` / `gpuBlockedRef` / `gpuFailSaidRef` 从动画 effect 的局部变量**搬到组件级**：
    退避/占用必须两条路一起看（一边挂了另一边别急着再撞）。
- 判据：`web_probe.cjs durperf` 第 ④ 条 —— 停播 + 关实时后改 `nz1` 的 `scale`：
  **不发**宿主 `/run`（`POST /run=0`）且那张卡片当场变 `GPU` 徽标（实测过，探针最后把值改回原样）。
- 效果（`lava_flow` 实测）：改参数那条路**不再打宿主**，一帧 GPU 工作 ~20 ms（原来宿主 CPU 一轮 801 ms，
  含 `db_flow` 那个方向模糊）。

### 「一个 CPU 节点不再拖垮整条 GPU 腿」（2026-10-17）

用户口径：「我希望不能因为有一个节点是 cpu 就全部退化为 cpu」。以前 `canRunGpu()` 是**全有或全无**：
prompt 里只要有**一个**节点没迁，整段回 CPU（实测：`lava_flow` 上摆了个没接线的 `CallFunction`，
实时模式从 30 fps 掉到 **0.4 fps / 2633 ms 一帧** —— 而且**静默**）。

三件事一起做（`texGpu.js` + 编辑器）：

1. **只判"参与输出"的节点**（`liveNodesOf`）：从 sink 往上闭包。sink = `2d_view` / `Output` / `Preview` /
   `TexMaterial*` **＋ 没人消费的末端里能出图的那些**（`Tex*` / `SubGraph`）。图上角落没接线的
   `CallFunction` 不在这个集合里 ⇒ **既不参与判定、也不求值**（`runGpu` 只跳过"没参与输出**而且**自己或
   上游有没迁的"节点；能跑的照旧跑，探针的逐算子对拍靠这个）。
2. **混跑**（`gpuPlan()` → `ok` / `mixed` / `bake` / `dynamic`）：参与输出**但没迁**的节点，让**宿主算一次**
   （只把它们的上游打成内联 prompt 跑一轮），结果当常量喂进 GPU 图：
   · 图片输出 → `/render` 取 PNG → `gpuTextureFromBitmap`（`copyExternalImageToTexture` 进 `rgba8unorm`，
     再一趟 compute 转成 **`rgba32float`** —— 图里纹理格式写死了，不一致就是绑定组校验失败）；
   · 标量 → 原样。缓存按 `图#版本#用途#bake名单`，一版图只算一次。
   · `dynamic`（这些节点**依赖 `Time`**）⇒ 当不了常量 ⇒ 仍然整段回 CPU（徽标会写明）。
3. **`GetParam` → `Input` 的就地改写**（`gpuRewriteParamRefs`）：`Input` 已退休，图里入参是 `GetParam`
   （引擎侧它就是"读本图接口"）；GPU 那条腿只认 `Input` ⇒ 喂 GPU 的那份 prompt 里换回去（默认值取图级
   `params`，实参随后由 `injectParams` 覆盖）。**只有拿到默认值才换**，否则留给 CPU 明确报错。
   ⚠ 不换的话：**任何带参数的子图**都上不了 GPU（探针 `subgpu` 就是这么红的，2026-10-17）。

四个坑（都实测踩过，别再踩）：
- **预置失败要退避**（10s）+ **预置期间动画跳过这一拍**：宿主一次只跑一张图，CPU 帧会把预置自己的
  `/run` 挤成 **429**（实测 12 秒 287 次 `/run` 的风暴就是这么来的）；
- WGSL 里 `vec2f(gid.xy)` 这种 u32 向量构造会编译失败，而且 **"invalid due to a previous error" 会把后面
  的管线一起带崩** ⇒ 老老实实写 `vec2f(f32(gid.x), f32(gid.y))`；
- 模板字符串里**别出现反引号**（我在 WGSL 注释里写 `vec2f(gid.xy)`，直接让 vite 构建报 parse error）；
- `liveNodesOf` 的 sink 判据**不能只认 `SINK_KINDS`**：探针 `subgpu` 那种图没摆 `2d_view`，结果就是末端
  那个 `SubGraph` 节点 —— 只认显式 sink 会把整张图判成"没有输出"。

判据：`web_probe.cjs subgpu`（临时图里**故意塞一个没接线的 `CallFunction`**，必须仍走 GPU、**0 次 CPU
`/run`**）＋ 手工实测：`lava_flow` 实时模式 **30 张卡全 `GPU`、21~31 fps**（图里那个 `CallFunction` 卡片
干脆没有徽标 —— 它压根没跑）；混跑图（`TexNoise + TexLoadImage → TexBlend → 2d_view`）：
`ld` 标 `CPU`（悬停「混跑：ld 在宿主上算了一次」），其余 `GPU`，一帧 work 4~5 ms。

### `Input` 退休后的现状 + `Const` 上 GPU（2026-10-17）

用户口径：「`Input` 节点已经删除了，还有残余逻辑和代码么；如果不想建立入参，可以用常量节点替代（任意类型）」。

**产品面：已经没有任何入口能摆出 `Input` 节点** ——「⇄ 接口」的 **＋ 入参** 走 `addParam()`（加一条
**图级 `graph["params"]`**），**＋ 出参** 才摆 `Output` 节点；调色板另有 `PALETTE_HIDE` 兜底。
（刚删掉的残留：`addIoNode` 里 `kind === "Input"` 那一支 —— 它会 `addNode("Input")` 再设名字/`flags.show`，
而引擎已不注册这个类型 ⇒ 按出来是个"未注册类型"的空壳。）

⚠ **入参默认值要按类型归一**（2026-10-17 补的坑：`Input` 退休后这层没人做）：「⇄ 接口」里给 `int`
填 `7` 以前会以**字符串** `"7"` 存进 `params`，跑图时按字符串交给引擎（`CallFunction` 那类会炸/传错类型）。
现在两道：① `pySchema.normalizeParam` 在**读档/存档**时按 `type` 归一（标量→真标量、容器→仍存 JSON 文本、
`bool`→true/false、`None`→null）；② 输入框 **`onBlur`** 归一（⚠ 别放 `onChange`：边敲边转会把 `1.`
的句点吃掉，`Number("1.") === 1`）。自证 `web_probe types`（13/13：迁移 / 引用行 / 默认值存盘形态）。

**有意保留（不是残留 bug，删了老图就炸）**：
- 引擎：`graph._rewrite_legacy_inputs`（跑前把老 `Input` 就地换成 `GetParam`，`type` 一起带过去）+
  `interface_values()` 按**类名**认老 `Input` 取默认值；`migrate_input_nodes`（迁移工具）+
  前端 `pySchema.migrateInputNodes`（编辑器版同口径）；`validate` 把 `Input` 列进"内置类型"放行。
- **嵌套图的变量注入点**：`subgraph.inject_params` 只认 `Input`（按 `name` 灌实参；FX-Map 的
  `$index`/`$pos` 也走这条），`l_tex_nodes` 内置图案子图就是这么写的；GPU 侧 `texGpu.injectParams` 同口径。
- GPU 那条腿内部的 `Input` = **"可注入值槽"**（`gpuRewriteParamRefs` 把 `GetParam` 换成它，实参再由
  `injectParams` 灌），不是给用户摆的节点；卡片上那几条 `class_type === "Input"` 分支只为**老图显示正确**。

**常量 = 替代品，这次补齐了 GPU 那一半**：引擎里 `pygraph.Const` 本来就是"任意类型常量"
（`value` + `type` = python 类型表达式，走 `coerce_param`；`Input` 已删、它顶替"写死的值"），
但 **`texGpu.KINDS` 里没有 `Const`** ⇒ 图里放一个常量会被判"没迁"、落到混跑/CPU。现已加上
（`KINDS` + `SCALAR_KINDS` + 与 `Input` 同一份取值口径）。实测：`Const → TexTransform.translate_y` 的图，
改一个参数后 **`Const` 卡也是 `GPU`、0 次宿主 `/run`**；`gpuPlan` 判 `ok: true`（不再走混跑）。
口径提醒（也写在 `Const.DESCRIPTION` 里）：**想让调用方能覆盖那个值 ⇒ 那是入参**（图级 `params` +
`GetParam`），不是常量。

### 老图真迁掉：`l_mindmap_mmd_migrate_inputs`（2026-10-17）

用户口径：「`input` 节点完全清理了么，老图应该迁移」。**迁移工具**（默认 dry-run）：

```cmd
wuwor l_mindmap_mmd -- l_mindmap_mmd_migrate_inputs             :: 只看（要迁哪些、各改几个）
wuwor l_mindmap_mmd -- l_mindmap_mmd_migrate_inputs --apply     :: 真写（改过的旁边留 .bak-<时间戳>）
```

- 扫描范围：数据根 `<pygraphs>/*.json` ＋ `projects.json` 每个工程根，**递归**（图都在 `Functions/` /
  `demo/` / `Tools/` 子目录里 —— 第一版只 `glob` 顶层，扫出"要迁 2 张"的假绿）。
  `_renders` / `_sessions` / `_ui` / `_traces` / `media` / `models` 整枝跳过。
- 实现 = 引擎那份 `l_node_engine.pygraph.migrate_input_nodes`（`Input` → 图级 params + `GetParam`，
  **id 不变**、`dock`/连线/位置全保住；没人用的纯声明删节点）＋ 一条**补充**：没有 `name` 的老 `Input`
  是"**值槽**"不是入参（用户口径：「不想建立入参可以用常量节点」）⇒ 换成 **`Const`**（类型按 python 类型名
  沿用，控件名如 `image` 按值猜：bool/int/float/list/dict/string）。
- **实测（本机）**：23 张图、46 个 `Input` ⇒ **28 个改写成 `GetParam` / 18 个纯声明删掉 / 1 个值槽换
  `Const`**；迁完这批文件里**再没有 `Input`**（验证通过后 `.bak` 已删）。

⚠ **迁移逼出了两处配套修复**（不补的话"文件迁完，应用里全红"）：
1. **宿主** `server._graph_param_defaults`：以前 `interface=` 里那份"声明的默认值"是**引擎从 `Input`
   节点扫出来的**（`graph.interface_values`）—— 迁完没有 `Input` 了 ⇒ 现在宿主自己把
   `graph["params"]` 的默认值垫进 `interface=`（**调用方实参赢**）。漏了这条：迁移过的图一跑就是
   `GetParamError: 这张图没有声明入参 x`（`demo_use_subgraph` 实测）。
2. **量具** `tools/time_perf.py`：同样要传 `interface=`（`prompt_of()` 现在回 `(prompt, interface)`）——
   不然它会报一堆 `GetParamError`，看起来像"迁移把图弄坏了"。

**验证**（迁完逐张跑）：`Functions/demo_tex_basics` 70 ms、`demo_all_nodes` 1 ms、`Tools/demo_tool_add`
0 ms、`agent_lite_read` 10.7 s（AI 图，慢是正常）、`demo_fxmap_pattern` 6 ms、`demo_fxmap_fxmap` 0 ms、
`demo_bamboo_weave` 1753 ms、`Functions/123` 跑通（它的 AIImage/BaiduTTS 报错是**缺密钥/空问句**，与迁移无关）；
`web_probe.cjs fire` 8/8。
⚠ `Functions/demo_use_subgraph` 迁完后**顶层已通**，但**内层**还有个 `normalize_path_segments`
（"函数节点"插件产物）报 `Unknown node class` —— 那是**插件默认关**，与迁移无关。

**还剩哪些 `Input`**（有意留，见上一节）：引擎兼容层（`_rewrite_legacy_inputs` / `interface_values` /
`migrate_input_nodes` / `validate` 白名单）、嵌套注入点（`subgraph.inject_params` + `l_tex_nodes` 内置
FX 图案 `nodes.py:1103`）、GPU 内部"可注入值槽"（`texGpu`）、编辑器里"老图显示正确"那几处分支、
引擎 selftest 的兼容夹具。**磁盘上已经没有一张图带 `Input` 节点了。**

### 资产类型：文件里的 `tag` 字段（2026-10-17）

用户口径：「一个资产试一个 json 文件，通过什么来区分呢，其实没有本质区别对不」→「给每个资产 json
文件加 tag 字段吧，类型有 智能体 / 🖼 程序纹理 / 🎬 运动图形 / 🧰 工具 / 🤖 AI 调用 / 结构体 / 🧩 类」。

**结论**：确实没有本质区别 —— 图就是节点图，结构体/类/工具各是各的 `kind`。所以 `tag` 是**用途声明**：
**只影响界面怎么标/怎么筛，不改变执行**（执行语义仍在 `kind`：`fx` = FX 根/竖排，`tool` = 当工具暴露）。
同一份 JSON，靠 **`tag`（声明）+ 内容/引用（兜底）** 分成不同类型资产。

| `tag` | 界面 | 判定（`tools/tag_assets.py` 按优先级） |
|---|---|---|
| `agent` | 🧠 智能体 | 被 `agents/*.json` 的 `flow.graph` 引用 / 地址里有 `agent/` / 有 `HostStep` |
| `tool` | 🧰 工具 | `kind == "tool"` / 落在 `Tools/` |
| `texture` | 🖼 程序纹理 | 有 `Tex*`（`kind: "fx"` 也算这类） |
| `motion` | 🎬 运动图形 | 有 `Mg*`（`l_tex_nodes/motion_nodes.py` 那 19 个） |
| `ai` | 🤖 AI 调用 | 有 `AI*` / `Ask` |
| `struct` / `class` | 🧱 / 🧩 | `kind == "struct" / "class"` |
| `other` | 🗂 其他 | 都不像（`Calc`/`Preview`/子图/测速图/0 节点空图）—— 用户 2026-10-17：「**不加新类型**，无法确定的
设置为**其他**类型就行」 |

- 工具：`wuwor l_mindmap_mmd -- l_mindmap_mmd_tag_assets [--apply] [--force] [--root <目录>]`
  （默认 dry-run；扫数据根 + 各工程根，递归；`--force` 连已有 `tag` 的也重算）。
- 实测（本机 47 个资产，**每个都有 `tag`**）：**程序纹理 18 / 其他 19 / AI 调用 6 / 工具 2 / 智能体 1 /
  运动图形 1**。
- 宿主：`/api/pygraphs` 等列表里的 `tag` = **文件里的优先**、没有才用 `_graph_tag(内容)` 兜底。
- 编辑器：图标按 `tag`（`ASSET_ICON.graph.agent/texture/motion/tool/ai/misc`，老键 `tex/flow/sub/code` 仍在）；
  **列表视图每行挂一枚类型标签**（`.pyg-asset-type`，如 `程序纹理`）；格子视图按 2026-10-13 的口径
  "只留图标+名字"（类型走悬停 `title`）。
- ⚠ **`tag` 必须进 `pySchema` 的两处白名单**（`normalizeGraph` + `toStorage`）：少一处就是"编辑器里
  存一次、这个字段就没了"（`demo_mg_mandala` 实测丢过一次 —— 与 `kind`/`root`/`engine`/`keys` 同一个坑）。
- **还没做**（下一步，按需要）：类型**筛选**（内容浏览器工具栏）、「＋ 新建」先选类型、文案把"图"改叫"资产"。

### 右键「📄 查看资产源码」（2026-10-17）

用户口径：「类型浏览器任何资产，右键添加菜单，查看资产源码」。资产右键菜单（行 / 格子 / 收藏夹**同一份**
`assetMenu`）新增这一条：图 / 结构体 / 类 / 工具**一视同仁**（它们都是 json 文件）。

- 宿主端点：`GET /api/pygraphs/<资产地址>/source` → `{name, path, bytes, text}`。
  读的是**磁盘原文**（不是 `GET /api/pygraphs/<地址>` 那份规整过的图对象：缩进、字段顺序、临时手塞的键都在）。
  边界：**只读** + 只认 `_pygraph_path()` 解析得到的资产文件（与「在资源管理器里显示」同一条解析，
  **不接受绝对路径参数** —— 那等于开任意文件读的口子）；不存在 404。
- 编辑器：只读浮层（`.pyg-src`，等宽 `<pre>` 自己滚，头部显示地址/大小/磁盘路径 + 「📋 复制」「✕」）；
  **Esc / 点遮罩 / ✕** 都关（那段有专门的 keydown effect）。
- 实测：图 13.6 KB 原文 ✓（正文里能直接看到 `"nodes"`、`"tag"`）、工具 ✓、不存在 → 404 ✓；
  UI 上右键 → 菜单里那条在 → 点开浮层 → Esc 关 ✓。







### 动画帧的 React 更新做成增量的（2026-10-16）

**问题**：播放帧以前每帧走 `setNodes(整数组重建)` + `setLastOutputs` + `setViewerKey` + `setFrameStat`
四连 ⇒ **整个编辑器组件每帧重渲一遍**（22 张卡实测 ~20ms/帧），而同一帧真正变的往往只有 1 个 `Time`
节点（纹理那几路显示的是 `WebGPU(512x512)` 这类占位、帧帧一样）。

**做法**（`web/src/liveValues.js`，外部 store + `useSyncExternalStore`）：

| 每帧要变的东西 | 以前 | 现在 |
|---|---|---|
| 卡片上"输出端口本轮值" | `setNodes`（整数组重建 ⇒ 整棵重渲） | `liveValues.publish({节点id: […]})`，卡片**自己订阅自己**那一格 ⇒ 只有真变了的卡片重渲 |
| 卡片缩略图（「🖼 卡片动画」） | `setThumbLive`（state） | 同上，`publish(null, {节点id: dataURL})` |
| 取景器刷新 | `setViewerKey`（state） | `liveValues.commitFrame()` → 取景器订"又走完一帧" |
| 读数 / 面板「本轮值」/ 缩略图 epoch / 重建闸 | 每帧各一次 setState | **播放帧一个都不需要**（取景器由 `commitFrame` 驱动、缩略图走 store、卡片高度不随数值变）⇒ 只留面板值 **2Hz** |
| `▶ 动画` 每帧的"内联图" | 每帧 `S.toPrompt` + 子图内联重算 | 按 `图名#version` **缓存**（图一改/一存即失效） |

**实测**（`fire_time` 512²/18 节点，读数里的 `work`）：**27 ms → 15 ms → 11 ms → 8 ms**
（真·GPU 计算 ~3ms；剩下的是 `runGpu` 自己的 JS 规划/提交 —— 再往下的杠杆在 `texGpu` 的 plan 缓存，
不在 React 这层）。最后那 3ms 是从"低频 state"那批里拿掉的：`setViewerKey`/`setThumbEpoch`/`setVersion`
播放帧**一个都不需要**（它们一起落在同一帧上时那一拍要 **122ms** —— 探针 `durperf` 抓到的尖峰）。

⚠ 三条踩过的：
1. **两处共用一个节流闸 = 互相吃掉**：GPU 分支先刷面板把窗口用掉 → `frameResult()` 里的读数**永远发布不出来**
   （现象：读数整格消失、探针两条红）⇒ 读数**自己一个** `statTick`。
2. `useSyncExternalStore` 的 `getSnapshot` 必须返回**稳定引用**（store 里存的原样给出）—— 在这层做
   格式化/拷数组就等于"每帧都变了"，白干。
3. 探针那条「动画刷新保留缩放和平移」原来把 `scale(4.92344)` **写死**：取景器头部多一行读数就改变
   可用高度 ⇒ 适配出来的缩放变了 ⇒ **假红**。已改成比 `before === after` + 形状（本意就是"刷新别重置用户视角"）。
   同理 `fire` 那条读数判据要求"画布与取景器**同值**"（只查 `N FPS` 会误判成 rAF 那个 60）。

`动画 N fps` 旁边还给**这一帧的拆解**（2026-10-16 挪到**画布读数里直接显示**：`帧 1320 ms = 算 900 + 等 420`
—— 只写"帧 1320ms"会被读成"这一帧算了 1.3 秒"，而里面其实可能大半是节流在等）：
`算 X ms` = 从发出到结果回来（含宿主 `_cache_renders` 出图与 SSE；时间线那条腿 = 整张图的引擎时间），
含宿主 `_cache_renders` 出图与 SSE；Y = 间隔里剩下的，节流没到点 / 上一帧还在跑）。
⚠ 拆解里**不含**浏览器解码 + 上屏（几 ~ 十几 ms，要量得在 `TexViewer` 里挂钩子）⇒
`X+Y` 是下界，`fps`（起点到起点）才是含全部的真值。
⚠ 卡片上那个 `耗时` 徽标（`duration_ms`）同理：**只算引擎里这一步**，不等于"整帧"。
实现：`PyGraphEditor` 的 `frameStat`（`frameBegin`/`frameResult`/`frameReset`；播放的帧才进表 ——
`▶ 动画` 那一拍与时间线那一发，手动点「▶ 运行」不算），取景器通过 prop 拿到同一份。
自证：探针 `fire` 里两条 —— ① 读数必须是"动画 N fps"（1~30）且画布与取景器**同值**；
② 停播后读数里**不许再有** `动画 `。

⚠ **卡片上的「本轮值」怎么刷新**（`outVals` 在 `PyGraphEditor` 那个重建 effect 里算，它的依赖表里
**没有** `lastOutputs`，只有 `version` 等）：CPU 那档每轮末尾 `setVersion` 所以自然刷新；
**WebGPU 那档**（`texGpu.js` 的 `runGpu`）**只就地改跑到的那些节点的 `outVals`**（`setNodes` 里逐个比
`outVals` 相同就原样返回）—— 不再整张画布重建（一帧重建 2000 节点比一次纹理计算还贵）。
踩过的两个坑：① 第一版没刷 `outVals` ⇒ GPU 动画开着、2D 视图在动而 `Time` 卡片数字不动
（2026-10-15 用户报障）；② 补成 `setVersion++` 后卡片动了，但每帧整图重建 ⇒ 动画被 UI 拖慢。
另外动画"开着却一动不动"会**说一句原因**（图里有结构错误 / 图里没有 `Time` 节点，每张图各提示一次）。
**GPU 那条什么时候用**：只有"这一拍要跑的节点**全在** `texGpu.js` 的已迁移表里"才走，否则整拍回退 CPU
（半 GPU 半 CPU 的图不做）。失败**退避 5s 后自动重试**（2026-10-16 改：以前是"失败一次就永久退 CPU"
—— 而 WebGPU 初始化偶尔抖一下（实测同一张图两次跑，一次走 GPU 一次退 CPU）⇒ 表现是"动画突然掉到
个位数 fps，再也回不去"✗）。失败原因每次退避窗口说一次（`WebGPU 动画失败，这 5 秒先走 CPU（之后自动重试）`）。
清单与四条口径见 `Rez-Docs/Rez_pkg/l_tex_nodes.md` §9（现在 15 个纹理算子 + `Time`；材质打包 / 导出 / FX-Map 还没迁）。
另外动画"开着却一动不动"现在会**说一句原因**（图里有结构错误 / 图里没有 `Time` 节点，每张图各提示一次）。

**画布上看得见值**：卡片每个**输出端口**现在会显示该端口本轮的**短标量值**（`PyNode` 的
`.pyg-out-val` + `PyGraphEditor.shortScalar`；数字按 4 位小数截、**尾零剪掉**）—— 所以 `Time` 卡片上
`time / frame / phase / fps` 直接读得到（实测 `0.8333 / 20 / 0.8333 / 3.2`），不必跑时间线或翻属性面板。
⚠ `fps` 是**实测**的（`l_node_engine/graph.measured_fps`：**最近 2 秒窗口里跑了几拍**，开头几拍 0）：
它就是"这张图被重跑了多快"，与设置里那个 24 不是一回事 —— 图重则掉到个位数，一眼看出动画卡在图本身。
**为什么不是"上一拍间隔取倒数"**（2026-10-15 用户报"fps 很不稳定"）：动画按 fps 节流 + 上一轮没回来就跳拍，
真实间隔是帧周期的整数倍 ⇒ 速率天然在 24/12/8… 之间跳；单拍倒数会把抖动全暴露，指数平滑又会拖尾十几拍。
窗口平均 2~3 拍就稳（实测 25fps 档 24.0~25.5、10fps 档 9.8~10.1）。**WebGPU 那档同一口径**（`texGpu.js`），
切档不会突变。
图/长串/容器不上卡片（那是 👁 / 属性面板 / `RunValue` 的事）。
另外：图里有 `Time` 节点、又没人驱动时，日志会说一句「点工具栏「▶ 动画」它才会动」。

### 「🖼 卡片动画」（2026-10-16）—— 节点卡片上那张预览图也跟着动

用户口径：「网页顶部加一个开关，打开后节点的预览图也能播放动画」。工具栏 `▶ 动画` 右边加一个
**`🖼 卡片动画`**（浏览器偏好 `localStorage["pygraph.thumbanim"]`，**默认关**）——开着时，
`▶ 动画` / 时间线每跑完一帧，把**每个跑到的节点**的缩略图（`.pyg-node-thumb`）换成这一帧的图；
关掉就回到"上次运行那张静态图"。开着时角标是 **🎞**（与"缓存"那个 **🕗** 分开）。

两条来源（各自一套，别互相替代）：

| 那条腿 | 图从哪来 | 为什么不能换另一条 |
|---|---|---|
| **GPU**（`▶ 动画` 且整链已迁移） | 前端**从显存里画**：`texGpu.nodeThumbUrls()` 给每个节点扛一块小画布 + 一个 `createGpuPresenter`，画完 `onSubmittedWorkDone` 再 `toDataURL` | GPU 帧**根本不落盘**（整链留在显存），宿主手里没有这一帧的字节 |
| **CPU**（时间线每帧真跑图 / GPU 不可用） | 宿主**随结果事件内联**回来：`payload.thumbs=1` → `server.py::_thumbs_payload` 把 `_cache_renders` 刚写进内存那份小图 base64 进结果 | 逐节点打 `/render?thumb=1` 就是请求风暴（每秒十几帧 × N 个节点） |

⚠ 三条实测踩过的：
1. **GPU 那条的判据不能只看"缩略图在变"**：GPU 一失败就整拍退 CPU，缩略图会被 CPU 那条兜住
   ⇒ 看着绿、GPU 那根腿**整条没跑**。探针 `thumbanim` 因此**同时**要求
   `canvas[data-backend="webgpu"]`（第一版就是这么误判的，写下来别再犯）。
2. **顺手修掉一个老 bug**：`RADIAL_BODY` 里那句 `let dims=…` 与节点体前导声明的 `dims` **撞名**
   ⇒ WGSL `redeclaration of 'dims'` ⇒ 整条管线建不起来 ⇒ **`TexBlur(kind=radial)` 在 GPU 上从来
   就没编译过**，一直静默退 CPU（表现只有日志一行「WebGPU 动画失败，切回 CPU：TexBlur:171:
   redeclaration of 'dims'」）。改名 `rdims` 后对拍从 146/146 涨到 **151/151**。
   ⚠ 教训：**节点体是拼进同一个函数体的**，那句注释里的名字别跟前导声明重名；另外
   **WGSL 模板串里的注释不许出现反引号**（会把外层 JS 字符串截断，本轮踩了三次）。
3. **`thumbs` 只在开关开着时才带**（每帧多几毫秒编码 + 几十到几百 KB 的内联小图），
   关掉时结果事件里没有这个键 —— 前端的"关"那条判据就是"一个 `data:` 缩略图都不该有"。

自证：`node rez-package-source/l_mindmap_mmd/999.0/tools/web_probe.cjs thumbanim`（5/5，
三条判据见 `tools/web_probe.md`）；GPU 对拍回归 `webgpu`（151/151）、火焰两版 `fire`（5/5）。

### 时间线（`🎞 时间线`，2026-10-13）—— 画布面板底部那一条

用户口径：**画布面板底部一个时间线按钮**；**每个节点的任意属性**都能打关键帧；另有一个 `Time` 节点
（引擎侧，见 `l_node_engine.md` §3.0a），**时间线不动它的输出值也能驱动别的参数**；并且
**「⚡ 实时」开着时间流动才有意义**（⚠ **2026-10-16 起不再自动替你开**：点 ▶ 播放只提示一句
「实时是关的」，要每帧真跑图得点时间线里那颗「⚡ 实时没开（点这里打开）」—— 用户口径「实时按钮默认请关闭」）。

落点与形状：

- **位置**：`.pyg-timeline`（`web/src/PyTimeline.jsx`）= `.pyg-center` 网格的**第 2 行、跨满两列**
  （取景器开不开都在画布下面一整条）。收起时只有「🎞 时间线」一个按钮（**强调色填充**：第一版用暗底暗边，
  用户直接"没看到按钮"，见下面坑①）；展开 = **刻度尺** + 播放头 + **起止帧范围** + 时长/fps/循环 +
  读数 + 关键帧数 + 「🔥 烘焙」+ 关键帧轨（选中节点的每个参数一行 chip）。
- **刻度尺与范围**（2026-10-14 用户口径："时间线上得有刻度和帧开始结束范围"）：
  · 尺子 `.pyg-tl-ruler`：大格 = **1 秒**（带 `Ns` 标签，太密就隔几条写字）、小格 = **1 帧**
    （2026-10-14 用户口径"请每帧一个刻度"；范围 >1200 帧时按 `frameStep` **抽稀**并在 title 里说明 ——
    900px 的尺子画 3600 条只会糊成实线）、播放头 = 一条青色竖线 + 顶端小三角；
  · **起止帧**（`帧 [起始] – [结束] / 总帧数`，含端点、整数帧）：**播放/循环只在 `[起始, 结束]` 里走**，
    到头回**起始帧**（不是回 0）；总帧数 = `时长 × fps`，把 结束 调到总帧数就是整条。
    范围存在 `localStorage["pygraph.timeline.*"]`（浏览器偏好，不进图文件）。
  · ⚠ **几何只有一条数轴**：下面那条 `range` 的 `min/max` 就是范围两端，尺子按百分比画 ——
    所以行首给 `range` 留了 7px（≈滑块半径），不留就会出现"刻度跟滑块差半个滑块宽"（看着像刻度不准）。
  · 范围只影响**播放/循环/时钟**，不动已打的关键帧（范围外的键还在图上，范围调回来又能播到）。
- **时钟是"相对范围起点"的**：`timelineRun` 发 `time = t - 起始`、`frame = 帧号 - 起始帧`、
  `duration = 范围长度` → `Time.phase` 正好在范围里扫一遍 0..1（范围是整条时与"从 0 数"完全一样）。
- **关键帧存在图上**：`nodes[id].keys = {参数: [[t秒, 值], …]}`（升序、同 t 只留一条）。
  ⚠ `pySchema` 的 `toStorage` / `normalizeGraph` 都是**显式白名单** —— 加字段必须**两处都改**，
  否则"保存一次就没了"（与 `engine` 同一个坑，有单测钉着）。
- **打点的三个入口**（2026-10-14 用户口径："每个参数都能设置关键帧，参数上右键 set key"）：
  · 属性面板参数行上的 `⏱` —— **每个参数都有**（连线的也有）；
  · **参数行右键** = 关键帧菜单（`openContextMenu` 那套总线）：设置/删除这一帧的键、上一个/下一个关键帧、
    清掉这个参数的全部键、在此帧烘焙进输入。**画布节点卡片上的参数行也是同一个菜单**
    （`PyNode` 的 `PyRow` 早就留了 `onParamMenu` 口子，现在由编辑器把 `paramMenu` 递进去）；
  · 时间线轨上的 chip 点一下 = 把播放头挪过去，chip 上的 `×` = 删那一条。
  ⚠ 连线的参数**也能打**：那种值由上游决定，所以菜单/`⏱` 那一条写的是
  「✂ 断开连线并打关键帧」——点一下**先断线再打点**（日志里两条都记，别让人以为只打了个点）。
  ⚠ 右键落在 `input/textarea/select` 上时**不抢**菜单（复制/粘贴留给浏览器原生菜单）。
- **求值在编辑器**：`overlayAt(t)`（`sampleKeys` 线性插值；**非数值走步进**，所以字符串/布尔也能打点）
  → 覆盖到图的副本上 → 走 `doRun({prompt})` 那条**内联图**路。**一帧都不落盘** ——
  要固定某一帧的值按「🔥 烘焙」（那才 `mutate` + 落盘）。
  ⚠ 只覆盖**字面量**参数：连着上游的参数是 `[源id, 下标]` 引用，被时间覆盖没意义（要动就动上游 / 接 `Time`）。
- **每帧 ≤ 一轮跑**：rAF 循环走**墙钟**做播放头，**发射按 fps 节流**且"上一轮没回来就跳过"
  （`runRef.current` 那道闸）—— 服务端一次只跑一张图，不节流就会把队列堆满（还会吃 429）。
  ⚠ 节流时刻存 **ref**（`tlFire`）：那个 effect 每次重渲染都重启，局部计数会被重置 → 退化成
  "每次渲染发一轮"（实测一次播放打出去上千发请求）。
- **播放的帧不写留痕**：`server.py` 里带 `clock` 的那批 run **不写 `runs.jsonl` / `_traces/`**
  （24fps 会把盘与「历史/轨迹」面板刷爆）；同一批 run 也**不打日志**（前端 `quietRef` 只放行 `✗` 开头的行）。
- **`Time` 节点靠 run 正文里的时钟**：`time` / `frame` / `duration` 三个可选的数字键
  （`_run_clock` → `execute_graph(clock=…)`）。只对**本地引擎**有意义。
- 自证：`node tools/web_probe.cjs` 之外，另有一次性的浏览器检查（按钮/展开/打点/播放/保存落盘各一条）；
  离线部分是 `web/tests/pySchema.test.mjs` 里 `sampleKeys` / `setKeyframe` / 白名单那四条。
- ⚠ 踩过的两个坑：① 点「⏱」后**属性面板会掉选中**（那是编辑器既有的"面板里一改就清选中"行为，
  不是本功能引入的；探针里点完要**重新选节点**再断言）；② `⏱` 是**打/删开关**，同一个 t 上再点一次
  就是删 —— 夹具要从"没有关键帧的干净图"开始，否则一轮点下来全变成删。

  ⚠ 副作用：探针 `assets` 的夹具造在**默认工程（根）**下 → 那半现在**跳过**（要恢复得把夹具改成
  `projects/register` 登记的工程）。
  ⚠ **两次搬家要记住**：起初它是左栏「🧩 节点库」里的第一节 → 2026-10-13 拆成**独立标签**（左栏）
  → 同日又改名「📂 内容浏览器」并挪到**底栏**（`assets` 这个 key 一并改成 `browser`：
  **改 key 就是有意清掉旧布局**，否则老存档里它仍挂在左栏）；`normalizeTabOrder` 会把新键**按
  `TAB_DEFS` 顺序插到它该在的槽位**（不是一律追加到末尾），所以它稳定落在运行日志前面。
  要用搜索框 / 节点列表就切回 `🧩 节点库`（左栏）。
  - ⭐ **「🧩 节点库」那个搜索框现在同时给两张表**（2026-10-17 加的第二张）：`GET /api/pygraphs/search?q=…`
    → **「🧩 类」**（宿主已导入模块里的类，点一下落 `MakeInstance`）
    + **「ƒ 函数」**（用户口径 2026-10-17：「节点库面板可以搜索任意 Python 函数，比如 `os.path.getsize`，
    搜到的函数可以使用 callfunc 节点调用」）—— 点一条落 **`CallFunction`**，`func_path` 预填，
    端口按签名展开（走 `/api/pygraphs/ports`，与手打路径同一条链）。
    函数那半的实现与三条口径（**搜索不 import / 落节点不 import / 真 import 在跑图那一步**）
    见 `l_node_engine.md` 表里的 `functions.find_functions`；自证 `web_probe fnsearch`（7/7）。
    ⚠ **这条路的 URL 别再写回 `/api/pygraphs/classes`**：那个路径已经是**类资产列表**（`_CLASS_PICKER`
    与端口候选都用它），而同路径的 GET 在 FastAPI 里**先注册的那个赢** —— 搜索这条原来就挂在
    `/classes` 上，于是**从来没被调到过**（表现：调色板「🧩 类」一直搜不出东西；2026-10-17 做搜函数时
    才发现，顺手修了）。
  - ⭐ **资产图标：先分类型 → 图的角色 → 图的内容（2026-10-14 加中间那一层）**：
    用户口径"内容浏览器里**不同的资产要不同的类型**，`pro/agent/default_intelligent_agent`
    是智能体，应该用**大脑图标**"。
    - **第一层 `kind`**（前端贴的）：`graph`/`struct`(🧱)/`class`(🧩)/`tool`(🧰)；
    - **第二层 `role`**（宿主算，`server.py::_graph_role`）—— 只对图有意义，"**这是什么**"：

      | `role` | 判据 | 图标 |
      |---|---|---|
      | `agent` | **被智能体引用**（`:1250/api/agents` 里 `flow.graph` 指向它，30s 缓存；agent 没起时**按目录兜底**：地址里有一段 `agent`） | 🧠 智能体 |
      | `fx` | 顶层 `kind == "fx"`（FX-Map：根节点就是结果） | 🎨 |
      | `tool` | `kind == "tool"` 或地址落在 `Tools/`（引擎 `is_tool_addr`，那个判别只有一处） | 🧰 |
      | `""` | 其余 | 落到第三层 |

      ⚠ 为什么"智能体"要**问 agent 进程**：图文件里**没有**"我是智能体图"这种标记 —— 谁引用它
      只有 `agents/*.json` 的 `flow.graph` 知道（同 `/api/pygraphs/host-steps` 那套转一手的做法）。
    - **第三层 `tag`**（宿主算的**内容标签**，`server.py::_graph_tag`，判据 = **图上有什么节点类**，
      不读值，反正那行已经 parse 过整张图）：

      | `tag` | 判据（类名） | 图标 |
      |---|---|---|
      | `tex` | 有 `Tex*`（贴图算子） | 🖼 出图 |
      | `ai` | 有 `AI*` 或 `Ask` | 🤖 大模型/人环 |
      | `flow` | 有 `IfStart/ElseStart/IfEnd/LoopStart/LoopEnd` | ⚙ 控制流 |
      | `sub` | 有 `SubGraph` | 🔗 引用了别的图 |
      | `code` | 有 `PythonCode` | 🐍 |
      | `""` | 纯数据/编排（`Input`/`Calc`/`Preview`…） | 🕸 |

    `tag` 多标签时按**上表顺序**取第一个（"能出什么"比"怎么串"更值得一眼看到）。映射在前端
    `ASSET_ICON` + `ROLE_ICON` / `assetIcon()`（`PyGraphEditor.jsx` 顶部，**唯一一份**，
    搜资产下拉 `AssetSearch` 与 搜资产候选表都走它）。有**封面缩略图**的行/格子仍然优先显示缩略图
    （那比图标更准）；悬停 `title` 里会把角色与内容都写清（如 `🧠 智能体（被智能体配置引用） ·
    控制流（If/Loop）`）。
    - ⚠ **只给"图地址"的列表也别写死图标**（用户 2026-10-14："这里应该显示资产的真实图标"，
      截的是**顶栏下拉的第二组「最近打开的图」** —— 那里原来是写死的 🐍）：这类列表只有地址字符串、
      没有资产行，图标得**现查资产表** → 用 `iconOfAddr(addr)`（= `assetIcon({...assetByAddr.get(addr),
      kind:"graph"})`，查不到给中性的 🕸）。目前两处：**顶栏下拉的最近图**（`.lproject.recent`）
      与 **🔗 引用面板的两栏**（谁引用我 / 我引用了谁）。
  - ⭐ **cook（烤封面缩略图，2026-10-13 晚）**：入口**只有工具栏那个 `🍳N`**（批量，见下）。
    ⚠ 早先还加过"每行/每格一个单独的 🍳"，用户同日让**删掉**了（"每个缩略图单独的 cookie 请删除"）——
    现在**行与格子上都没有** per-asset 按钮；有缓存的行/格子只是**自动**挂一张封面 `<img>`。
    非图资产没有这一套（没有可烤的产出）。`cookAsset()` 留着给批量内部用。点它 =
    **① 先问 `/renders`（内存 + 磁盘）**：有 → 直接用（`🍳 …：用现成的缩略图（不重跑）`）；
    **② 没有 → `runGraph(<那张图>)` 跑一遍**（跑完宿主把逐节点缩略图落盘 `_renders/<图>/n<key>.t.png`）。
    ⚠ 跑的是**那张图自己的文件**（服务端按 `name` 读盘）—— **不用先打开它**、也不动当前正在看的图；
    ⚠ 它走服务端那把跑图锁（同一时刻只跑一张），排队时会先收到 `queued`。
    封面由**宿主**挑（`GET /api/pygraphs/asset-thumb`）—— 前端不抄这份"挑谁"的逻辑；一张都没有 → 404。
    ⚠ **口径 2026-10-16 改了**（用户："封面 = 图里 `2d_view` 接的那张"）：① 图里有 `2d_view` 节点
    就用**它**的小图（= 底栏「🖼 2D 视图」那张 = 这张图的最终纹理，见 `l_tex_nodes.md` §6.2b）；
    ② 没有（或那个节点还没跑出小图）才回退老口径 —— `meta.json` 顺序（上次跑的拓扑序）里第一个
    真有小图的节点。响应头带 **`X-Tex-Cover: 2d_view|first`**（一眼看出走的哪条口径，探针拿它判）。
    资产行带 **`thumb`** 标记（`/api/pygraphs` 顺手算的）：有才挂 `<img>`，否则 24 行会打 24 个 404。
    小图标视图也用它当图标（格子里那张大图）。跑完靠 `thumbV` **换 URL** 破缓存（响应是 `no-store`）。
    - **命令行版 cook**（同一套口径，给脚本/批量用）：`wuwor l_mindmap_mmd -- python
      <包>/999.0/tools/cook_assets.py --addr <工程/相对路径> [--addr …] [--dry|--force]`
      —— 只认 `X-Tex-Cover=2d_view` 才算烤好了（`first` 会重烤，否则那份老封面永远换不成新口径）。
  - ⭐ **批量 cook（工具栏 `🍳N`，2026-10-13 晚）**：烤**当前这一层**的图（`graphRowsInView`：
    `browserAt` 的工程 + 路径下的**直属**图；停在工程列表层 → 按钮禁用并提示"先进到文件夹"）。
    ⚠ **并行这件事分两半**：**查缓存**（`/renders`，纯 GET）**并行**发（一次 `Promise.all`）；
    **补跑**（真执行图）**只能串行** —— 服务端就一把跑图锁（"同一时刻只跑一张图"，`_run_lock` +
    `MAX_RUN_QUEUE`），并行提交只是排队、还会把日志搅乱。所以那边**顺序 await**，每张跑完再下一张，
    日志里能看到进度（`这一层 N 张图：X 张有缓存（跳过），Y 张要跑` → 逐张 → `cook 收工`）。
  - ⭐ **右键菜单（2026-10-13 补）**：资产行与**文件夹行**右键都有
    `📋 复制相对路径`（对外口径 `/工程/Functions/x`，**带工程前缀** —— 文件夹的 `it.path` 是工程内相对路径，
    少了前缀就会出现"复制出来打不开"的路径）与 `📋 复制绝对路径`（磁盘那份 `.json` 的绝对路径）；
    资产行多一条 `📁 在本地浏览器中显示`（宿主 `explorer /select,<文件>`）。
    复制走 `navigator.clipboard.writeText` —— 它要**用户手势 + 安全上下文**（https/localhost），
    拿不到就把内容**念进运行日志**（`📋 已复制 …` / `✗ 复制失败…`），别让人"点了没反应"。
  - **两种视图**（2026-10-13，同 UE 的 List / Tile）：标题行右侧 `☰` = **详情**（一行一条，名字 / 节点数 /
    引擎都给全）、`▦` = **缩略图**（网格方块，大图标 + 名字）。⚠ **缩略图模式不显示"最后一行文字"**
    （2026-10-13 用户口径）：图和文件夹的格子都只留 图标 + 名字 —— 那个 `N 节点 · 引擎` / `N 项` 的
    `<span class="pyg-tile-meta">` 已删；要细节切回 `☰`，或悬停看 `title`（资产详情还在）。选择落
    `localStorage["pygraph.browser.view"]` —— 那是**人的偏好**（不进图文件、不随工程走）。
  **悬停 = 资产详情**（同一份 `title`，两种视图都用）：**相对路径**（`/工程/Functions/x`）·
  **绝对路径**（磁盘上那份 `.json`，来自列表接口的 `path`）· **大小**（`size`，B/KB/MB）+
  按类型补的概况（图：`N 个节点 · 引擎 …`；结构体/类：字段 / 方法数）—— "磁盘上到底是哪个文件、
  多大"不用去翻目录。
  **项目路径显示**（2026-10-13）：内容浏览器标题行右侧常驻 `📁 …/<末两段>`（= 资产根目录，也就是
  "工程都在哪个磁盘目录下"），完整绝对路径在 tooltip 里，**点一下复制**到剪贴板（查磁盘 / 写脚本直接用）。
  **新建工程 + 一次性迁移历史资产**（2026-10-13）：标题行 `＋ 工程` → 内联输入框（不弹窗）填名字回车
  = 建出 `<工程>/project.json`（`{kind:"project", title:<名字>}`），随后**自动做一次迁移预检**：
  把"根上还有多少项资产可以搬进来"摆出来（黄条），点「迁过去」才真动文件（**两步确认**）。
  迁移走 `POST /api/pygraphs/projects/migrate`（`dry` 缺省 true = 只预检）：
  · 真跑 ① 先把要动的文件 `copy2` 到 `<图目录>/_backup/<时间戳>/`（**备份失败就不搬那个文件**），
    ② `os.replace` 逐个搬（**目标已存在 → 跳过并记 error，绝不覆盖**），
    ③ 连 `_sessions/<图名>` / `_renders/<图名>` 一起搬（它们按**图名**存，地址一变就等于
    "续不上会话 + 缓存全过期"）；`media` / `models`（素材）与 `_*`（内部）目录**不碰**。
  · ⚠ 迁完**图地址就变了**（`demo_x` → `<工程>/demo_x`）→ 顶栏"打开"下拉是另一份列表，
    刷新页面最稳；回退 = 从响应里的 `backup` 目录把文件拷回根上。
  ⚠ 探针只测**预检**（`dry`）—— 真迁移会把**别人的**根上资产一起搬走，回归测试里绝不点「迁过去」。
  **工程 = 一个磁盘目录（可以放在任何地方）**（2026-10-13 定；用户口径：「**没有"资产根"这个概念，
  只有"项目文件"的概念**」）：不是"资产根下挂工程"，而是每个工程自己就是一个目录 ——
  · **默认工程**（`name: ""`）= 老图目录 `pygraphs/`：历史平铺资产、`工程/资产` 两层写法**原样兼容**
    （解析时首段查注册表，查不到就落回它）；
  · 其余工程登记在 `<数据根>/l_mindmap_mmd/projects.json` = `[{name, root}]`（root 是**绝对路径**），
    每个工程在自己目录里放 `project.json`（`title` / `entry` / `deps`）；
  · 地址仍写 `工程/相对路径`（对外带前导 `/`），`POST /api/pygraphs/projects/register {name, root}`
    建 / 改指向（**已有 `project.json` 就只登记不动它**；默认工程里有同名目录 → 400 拒绝，
    免得"同首段的地址"指到新 root 造成"列得出来打不开"）；
  · 前端 `＋ 工程` = 输入**名字 + 工程目录**（留空 = 建在资产根下，老式）。
  **实测（2026-10-13）**：`lugwit_agent` → `E:\lugwit\lugwit_agent`，把默认工程根上 22 项历史资产
  迁过去（0 错误），备份在 `<数据根>/l_mindmap_mmd/_backup/20261013-124106/`；迁后地址 = `lugwit_agent/xxx`，
  `projects` 列表显示 `lugwit_agent(22)`。  **回退** = 把备份里的文件拷回 `pygraphs/`。
  **~~类型文件夹~~ 已删（2026-10-16 用户口径：「`.lproject` 不再需要 `dirs` 字段和相关逻辑」）**：
  以前那套是「`POST /api/pygraphs/projects/dirs` 建目录 + 把清单记进工程文件的 `dirs`」，
  为的是"空文件夹也画出来"（资产列表按文件扫，空目录本来看不见）。**现在整套去掉**：
  · 工程文件里**没有 `dirs` 字段**（`register` 新建工程也不再预建那五个目录，只写 `kind/title/deps`）；
  · 接口 `POST /api/pygraphs/projects/dirs` 与前端 `pyApi.ensureProjectDirs` **删除**；
  · 内容浏览器**按磁盘实际内容画文件夹**（空文件夹不画——要看空目录就直接在磁盘上建）。
  · `Tools/` 仍是"工具"那类的约定目录（`l_agent_tool` 扫它，见 `工具契约_ToolSpec.md`），
    但它现在只是**约定**、不在任何清单里。
  **`Tools/` 单列一类**（2026-10-13 用户拍板）：**工具**放 `Tools/` —— 节点网络工具（pygraph 图）
  与 python 类工具（资产里存 `module:Class` 路径）**都在这一层**；`Functions/` 只放普通函数图。
  **工程文件 = `*.lproject`（**按后缀认**，不跟目录名走）**（2026-10-13 用户定，同 UE 的
  `.uproject`）：
  · 读兼容老名 `project.json`（`_read_project_meta`），写一律写 `.lproject` 并**顺手删掉旧文件**
    （`_write_project_meta`）—— 根目录不会同时躺着两个工程文件；
  · 它**不是资产**（后缀不是 `.json`，扫描天然不碰）；读写只走 `projects/{register,entry}`；
    资产 PUT 进 `kind:"project"` 现在**直接 400 并指路**（免得又写回 `.json`）；
  · 入口用 `POST /api/pygraphs/projects/entry {project, entry}`（校验 entry 在工程自己目录里且图存在）；
  · 实测：`lugwit_agent` 的 22 项历史资产**全部迁进 `Functions/`**，根目录只剩
    `{Blueprints, Classes, Functions, Textures, Tools, lugwit_agent.lproject}`；地址随之变成
    `lugwit_agent/Functions/xxx`（连 `_sessions`/`_renders` 的键一起搬，续跑与缓存不失配）。
    备份：`<数据根>/l_mindmap_mmd/_backup/move-<时间戳>/`（回退 = 拷回工程根）。
  **内容浏览器 = 系统文件夹式导航**（2026-10-13 用户定的交互；**一次只看一层**）：
  · 默认**只显示"登记过的工程"**；默认工程（老图目录）与工程外的散目录 = 「**引擎内容**」，
    由标题行「👁 引擎内容」开关露出（对应 UE 的 `Show Engine Content`），选择记 `localStorage`；
  · **单击文件夹 = 选中**（高亮）· **双击 = 进入**（换当前层）；组头出现**面包屑**
    `⬆ 工程 › Functions › …`，点 ⬆ 逐层退回；进入某工程后只渲染它（`browserAt = {proj, path}`）；
  · **资产行/格子同样是"单击选中、双击才动"**（2026-10-18 用户口径「双击为打开图」）：
    双击 **图** = 打开它、双击 **类** = 进第一个方法图（之后顶栏 🏛 换方法）、
    双击 **工具/结构体** = 在画布中心落一个节点（它们没有"打开"这回事）。单击只打 `is-sel` 高亮，
    **不再落节点**（以前单击类/工具就会落一个，双击于是"多两个节点却没打开"）。
  · ⚠ **上层不显示下层资产**：当前层只列"子文件夹 + 这一层直接放的文件"，文件夹里的东西
    **进去才看得到**（2026-10-13 用户指出过一版"父层把 22 个文件全铺出来"的错 —— 那是把
    "<details> 折叠展开"的语义误套到系统文件夹模型上，现已去掉折叠概念）；
  · 行/格子只显示**文件名**（路径进 tooltip）、文件夹里缩进、空文件夹也画（`📁 Textures (0)`）、
    ▦ 视图里文件夹也是同尺寸格子。
  ⚠ **已知失败两条**（2026-10-13，未修完，别当成已交付）：
  ① ⌨ **Tab/空格 唤节点搜索**：**2026-10-14 结案 —— 不是产品的问题，是探针的探法错了**。
     真按 Tab 是好的（点画布后 `activeElement` = `.pyg-root` → 浮层出来、焦点进搜索框，实测）。
     当时那条 `keys` 红在**两处**：探针点了 `/W-260, H-200` 那一带（可能落在面板输入框上 → 焦点在
     input 里 → `onRootKey` **有意**不抢输入框的 Tab）＋ 它用**派发合成 keydown**（派发到 `.pyg-root`
     这种"容器本身"上时 React 找不到 fiber → 事件被丢；派发到容器内的元素才有效）。
     探针已改成"从 `.pyg-canvas` 真实矩形里取点 + 真按键"。**顺带**补了一处产品行为（真需求）：
     焦点跑到编辑器**外**（`body`）时（刚打开页面 / 点过页面别处）Tab 唤不出 —— 现在 window 级兜一层
     （只在焦点不在 `.pyg-root` 内、且不在输入框里时动手，见 `PyGraphEditor` 那个 keydown effect）；
  ② 🗂 **双击进入文件夹后，该层的文件没列出来**（`assets` 场景那条红：面包屑 `⬆ … › Functions`
     出现说明"层"切了，但里面的 `deep` 没渲染出来）。
  **还没做**：`kind:"tool"` 这类资产的校验 / 列表 / 🧰 图标 / "点一下落成 pygraph 里的工具节点"，
  以及"工具 UI 与 pygraph 节点 UI 用同一份 schema 渲染"。
  ⚠ 上面那句原来写的是「见**设计要点那节**」—— 那节**从来不存在**（悬空锚点，2026-10-13 发现）。
  口径已挪到**独立契约文档** `Rez-Docs/Rez_pkg/工具契约_ToolSpec.md`（S0：字段 / 权威类型词表 /
  容器 = 渲染器清单 / 落点 `Tools/` / 思考内容组件 = 内置工具 / S1–S4 判据 / 待拍板 6 条）。
  **工具表落地（S1，2026-10-13）**：`GET /api/pygraphs/tools` = **所有登记工程** `Tools/` 下的资产
  （判别走引擎 `l_node_engine.tool_spec.tool_spec_of`：**图工具 = 一张普通图**，参数由 `Input`/`Output`
  现算；**python 工具 = `kind:"tool"`** + `source.module` + `class`/`func`，参数可显式写、省略则**反射**
  并标 `inferred`）+ **内置工具**（含 `thinking` = 思考内容组件）。PUT `kind:"tool"` 走同一条校验
  （形状坏当场 400），且**不再被当图存**（`/api/pygraphs` 靠"有 `nodes` 才算图"自然挡住）。
  ⚠ 该端点必须声明在 `/api/pygraphs/{name:asset}` **之前**（`tools` 是单段，否则被当资产名接走）。
  实测：`demo_tool_add`（图工具，`a`/`b` 带**结构化默认 20/22** + `container:number`）、
  `describe_nodes`（python 工具，反射出 `module`/`allow_private`/`skip`/`skip_no_args`）、`thinking`；
  PUT → 200 之后图列表里查不到它。**端口类型表只剩一份**：原 `_SUBGRAPH_PORT_TYPES`（9 条）已并进
  引擎 `tool_spec`，宿主只剩 `_port_type_of()` 转发（引擎缺包回落 `*`）；`/api/pygraphs/ports` 实测 200。
  **agent 侧也接上了（S2，2026-10-13）**：`l_agent_tool` 的工具表现在 = **默认工程平铺图（历史资产）+ 每个
  登记工程的 `Tools/`**（判别同一份引擎实现）。⚠ **分类后果**：`Functions/` 里的图**不再是 agent 工具**
  （以前它们躺在默认目录里被顺手注册过）—— 要当工具就放进工程的 `Tools/`（或把它整支当图跑）。
  读/写与审批改看 `ToolSpec.flags`（`read_only` / `approval`），老的名字名单只做回落。
  **画布侧接上了（S3，2026-10-13）**：内容浏览器把 `<工程>/Tools/` 下的工具列成 `🧰` 行（`listTools()`
  → `assetTree` → `pyg-asset-tool`），**点一下就落一个 `Tool` 节点**（`addr` 填好）；`Tool` 节点的
  参数区不画笔布通用控件，而是挂**共享渲染器** `@pyshare`（= `l_py_ui` 包，与 l_agent 的工具卡
  **同一份组件**，契约 §6）；参数端口靠 `Tool.addr` 上的 `values.api` 被 `NAME_PORT_KINDS`
  认出来，自动问 `ports?kind=tool` 现算。**探针场景 `tools` 13/13 PASS**（进 `Tools/` → 点工具 → 落节点 → 端口请求 200 → `.pyg-tool-params` 带 `data-param`/`data-container`）。
  ⚠ **改了 `l_node_engine` 要重启本服务**（热更只覆盖自己那个包）：否则 `object_info` 还是旧元数据，
  前端认不出"这个端口引用工具"，**一步都不动**（实测：探针红在"没发 `ports?kind=tool`"，
  `POST /api/v1/services/l_mindmap_mmd/reload` 后 13/13）。
  **引擎增删 + 画布节点搜索**（2026-10-13）：
  · 🔌 引擎面板（右栏新标签 `pyTabs.TAB_DEFS.engines`，`EnginePanel.jsx`）：列清单（`local` 永远在、
    **不可删**）+ 在线点 + 切到它 + ✕ 移除；添加表单要 **`IP:端口`**（`192.168.1.5:8801` →
    `http://…`；**只有主机名时故意不猜端口** —— 猜错的表现是"远端连不上"，最难查）。
    写盘走已有 `PUT /api/engines`（整表覆盖；`GET` 会带上原始行含 token，所以写回不会把 token 弄丢）。
    ✚ **启停引擎**（2026-10-16 用户口径：「引擎面板要可以停止和运行引擎」）：本机地址的行上有
    **`▶ 启动` / `■ 停止`**（`POST /api/engines/start` / `/api/engines/stop`，两边共用 `_engine_card`
    那份解析）—— 按地址的 **IP:端口** 去主页控制面找**那张服务卡片**，`start` / `stop` 它
    （与 `wuwo svc start|stop` 同一套；`?dry=1` 只解析不动手）。三条边界 + 两道硬拦都在服务端：
    别的机器上的引擎本机点不动（那台的控制面我们连不上）→ 400；端口在卡片表里找不到 → 400
    （"手动起的进程"不是卡片）；**不许停本服务自己**（端口 = 本进程 → 400：停了它这个页面
    连这条响应一起断）；**不许停主页控制面 8090**（停了 `wuwo svc` 全不可用）。
    ⚠ 只有**本机托管的服务卡片**才点得动：手起的进程（比如 8795 上那个契约 v1 宿主，如果没用
    卡片托管）两个按钮都会给"卡片表里没有端口"。
    → **muse 那个引擎正是"手起"的**（用户口径：不由 wuwo 起）：双击
    `l_mindmap_mmd/999.0/tools/start_muse_engine.bat` 就起（走他们 D: 那套 `ru.bat` + python-3.11 货架，
    端口 8795 / token `muse-token`）。为什么要另走一路、设了哪些 env、实测数字与三个坑，见
    `Rez-Docs/Rez_pkg/远端引擎契约_v1.md` **§6.1**。
    ✚ **但引擎面板的 `▶` / `■` 现在也能起停它**（2026-10-18 用户口径「修复启动 muse 引擎的功能，
    详情弹窗显示启动命令」）：`engines.json` 里那台可以写
    `"start_cmd": "<...>\\start_muse_engine.bat"` / `"stop_cmd": "<...>\\stop_muse_engine.bat"`，
    `/api/engines/start|stop` 看到命令就**不再去找服务卡片**（它就是找不到卡才 400 的），
    直接 `CREATE_NEW_CONSOLE` 起一个窗口跑它（`cmd /c <命令>`；`start ""` 那版实测起不来，别改回去），
    并把 stdout/stderr **追加**到 `<数据目录>/engine_start.log`（停是 `engine_stop.log`）—— 起不来先看它。
    详情弹窗（ℹ）里会列出这两条命令；`PUT /api/engines` 整表覆盖时前端会把它们原样带回去（否则一改清单就没了）。
    ✚ **样例资产：`lugwit_agent_a/Functions/muse_upload_plans`（**类图**，2026-10-18 由普通图改过来）** ——
    字段 = 共享配置（`source_root` / `muse_path` / `user_email` / `organization_id` / `asset_type` /
    `update_if_same_name` / `dry_run` / `clean_local`），三个**内联方法**：
    `upload`（`make_muse_helper → build_upload_plans → LoopStart(Count) → GetItem → upload_plan → LoopEnd`）、
    `verify`（同前两跳 → `GetAttr` 取 `asset_name/category_path/source_path` → `verify_folder_against_server`
    → `LoopEnd`）、`clean`（同 verify，但 `write_keep_txt` / `delete_verified` 写死为 `true`，多一个
    `GetAttr(vver,"deleted")` 出参）。每个方法图**自己声明参数**（默认值与字段一致）——
    引擎的类机制里字段要靠 `GetItem(self,…)` 显式取，方法图各留一份参数才跑得起来（编辑器里也能直接 ▶）。
    ⚠ **方法体要在 muse 引擎上跑**：`CallGraphMethod` 是把方法图内联成 `sub`、由**调用方所在引擎**执行，
    所以调用它的节点也得在 `EngineStart(engine=muse)` 区内（跨引擎的嵌套子图 `_mixed_plan` 不切）。
    **整段用 `EngineStart(engine=muse)` … `EngineEnd` 括起来**（`start.out → gp_source_root.after`、
    `eng_end.body ← vle`），节点上**不写** `engine` —— 实测这段被切成 `local`(start) + `muse`(24 个) +
    `local`(3 个 Output)。
    ⚠ **启动器必须把用户货架放最前**（用户口径 2026-10-18「启动 muse 参考 `run_j_disc_backup_ui_5_26.bat`」）：
    他们的脚本是 `REZ_PACKAGES_PATH=C:\Users\%USERNAME%\packages;…` —— 少了这一节，
    `anim_upload_muse_tool` 会解析到 `repository\packages` 里**旧的 `0.4.1`**（266 个节点），
    而他们的开发构建在用户货架 `999.0.0`（337 个节点）。`tools/start_muse_engine.bat` 已按此修正。
    三处**他们包里的缺口**（为跑通这条路改了 `D:\TD_Depot\Wuzu_dev\anim_upload_muse_tool\src` 与
    用户货架 `999.0.0` 的**同一份**文件，不在我们仓里）：
    ① `muse_helper/muse_tool/upload_popo_artist_resources.py` 只有 `upload_plan(helper, plan…)`、
    没有"造 helper"的公开函数 → 补 `make_muse_helper(...)->object`（注解写 `object`：`MuseHelper`
    只在 `TYPE_CHECKING` 里，字符串注解解析不到）；
    ② `category_service.get_category_refs_by_path` 少了 `use_cache` 形参（`upload_popo_artist_resources`
    与 `upload_asset_precheck` 都在传）→ 补成兼容参数（传 False 也按 True）；
    ③ 新增 **`verify_folder_against_server(helper, folder, asset_name, category_path, …)`**（自动验证 +
    可选清理，返回**纯 JSON**）：逐文件按 `attachments[].md5` 对拍，`verified` 才给 `ok_paths`；
    `write_keep_txt` 先写凭据、写失败就不删；`delete_verified` 删前再核 size。
    ⚠ 两条实测坑：**`upload_plan` 把资产传在分类链第一节（根）**，按叶子分类查会 0 命中 → 验证先**全库按名**
    再按分类；`get_asset_by_id` / `search_assets` 在这个版本里可能回**原始 dict**（不是 `AssetInfo`）→
    文档里那两个 `_asset_*` 小工具统一读。凭据文件写到**上传目录的父目录**
    （`<父>/<目录名>.muse_verify.txt`）：留在目录里会被下次上传带走，而 `<目录名>.txt` 会被
    `build_upload_plans` 每轮开头删掉。
    ⚠ 另两条业务限制：`muse_path` **必须已存在**（这包只自动建它的下一级作者目录，父目录不存在直接
    `RuntimeError`）；`--dry_run` 那轮 `upload_plan` 返回 `None`，所以 `LoopEnd.items` 是空的
    （数轮数看 `count`）。
    ✚ **状态看得见**（2026-10-17 用户口径「引擎面板请显示引擎状态」）：每行改成两行 ——
    上排 `勾选框 + label + 状态徽标（🟢在线 / 🔴离线 / ⚪未探活）+ ▶ ■ ✕`，下排 `id · url · 节点数 ·
    python · 契约 v · 探活耗时 ms · 多久前探的`；离线那行还会把**错误原文**缩一行印在下面（悬停看全）。
    ⚠ 改前那个 7px 的点其实**没上过色**（CSS 只写了 `.is-off`，在线/未探过都是灰的）= 看不出状态；
    现在 `is-on` 绿 / `is-off` 红 / `is-unknown` 灰，文字跟着一起。数据侧 `engines.probe()` 多回
    `ms` 与 `checked_at`，`local` 那行也带 `node_count`/`python`/`contract`（`GET /api/engines?probe=1`）。
    探针判据：`web_probe engine` 的「状态徽标 / is-on / 状态细节 / local 也有细节」四条。
    探针已验：清单 / local 不可删 / 表单齐全（**不真加**，那会写 `engines.json`）+ `▶`/`■` 在、
    点下去真问服务端（假引擎 → 400 人话，见 `tools/web_probe.md` 的 `engine` 行）。
  · ⌨ 节点搜索浮层（`NodeSearch.jsx`）：画布上 **Tab 或空格** 唤出 → 打字过滤 `object_info`
    → ↑↓ 选、回车把节点**落在浮层弹出处**（`view.screenToFlow`，两种画布后端都对）、Esc 关。
    ⚠ **现状：探针里 Tab 唤不出**（`activeElement = BODY`，说明不是焦点/撞车 —— 监听挂在 `window`
    capture 上没吃到事件）→ 下一步改成**挂在编辑器根容器**（`onKeyDown` + 点画布时 `focus()`），
    别依赖全局 window。
  ⚠ **构建依赖**：并行会话新加的 `PyCodeEditor.jsx` 用了 `prismjs` 但没写进 `web/package.json`
    → 构建先挂在"Rollup failed to resolve import \"prismjs\""。已补 `"prismjs": "^1.29.0"` 并
    `npm install`（`web/` 目录下），构建恢复。
  **工程落点改到包内**（2026-10-13）：`lugwit_agent` 的目录从 `E:\lugwit\lugwit_agent` 搬到
  **`rez-package-source/l_agent_tool/999.0/src/lugwit_agent`**（与 `l_agent_tool` 包并列，
  方便 agent 侧直接按相对路径调用）；地址不变（`lugwit_agent/xxx`），只改了注册表里的 `root`
  （`register` 同名 = 改指向）→ 打开的图 URL 不用动。
  **工程改名 `lugwit_agent` → `pro`（= 只改挂载名）**（2026-10-13 用户口径：「把 `/lugwit_agent`
  修改为 `/pro`，相对路径统一使用 `/pro`」（**⚠ 这一条已被下面「别名只做图内引用」取代**：
  现在地址首段用**真工程名**、`pro` 只是图内引用的"本工程"别名，入口写它 = 400）。**只动注册表 `projects.json` 的 `name`**（外加探针里两条
  硬编码图名 `muse_demo` / `mix_demo`）—— 磁盘目录与工程文件名**没改**（还是
  `…\l_agent_tool\999.0\src\lugwit_agent` / `lugwit_agent.lproject`）：目录名、文件名叫什么与地址无关
  （见下条「虚拟挂载名」；用户随后确认「`/pro` 在任意项目都通用」）。迁后地址 = `pro/Functions/xxx`，
  URL = `/pygraph?name=pro%2FFunctions%2Fdefault_intelligent_agent&engine=muse`（⚠ 这一条是**当时的形态，已作废** —— 现在入口首段必须是真工程名，写它 = **400**；见下面「别名只做图内引用」）。
  实测：`GET /api/pygraphs/projects` → `name:"pro"` / `root` 指 `…\src\lugwit_agent` / 22 资产 /
  `dirs` 五项（`title` 也从 `.lproject` 里读出来了）；`GET /api/pygraphs/pro/Functions/default_intelligent_agent`
  → **200**（⚠ **当时**；现在这条 = **400**，要写 `project_name/Functions/default_intelligent_agent`），老地址 `lugwit_agent/…` → **404**（首段认不出工程 → 落回默认工程 → 那里没这个文件；
  ⚠ 此时磁盘上**确实有个叫 `lugwit_agent` 的目录** —— 它不影响解析，首段认的是挂载名不是目录名）。
  ⚠ 会话（`_sessions`）与渲染缓存的键里带旧前缀 → **老会话续不上**，没做数据迁移。
  **地址首段 = 虚拟挂载名（同 UE 的 `/Game`）**（2026-10-13 用户口径：「相对路径和工程在哪里无关，
  就像 UE 的 `/game` 一样」）：地址首段（`/pro`）是**挂载名**，由注册表 `projects.json` 映射到磁盘目录 ——
  **工程搬到哪、目录叫什么名字，都不影响任何地址**；换位置只改注册表那一行 `root`。实测过：目录从
  `E:\lugwit\lugwit_agent` 搬到包内 `l_agent_tool/999.0/src/lugwit_agent`，地址一个字没动。
  同理**工程文件按后缀认**（`_project_file` = `root.glob("*.lproject")`，不再由目录名拼串）——
  老实现把文件名绑死在目录名上，目录一改名就"元数据读不到"（`title`/`dirs`/`entry` 静默丢，
  内容浏览器里声明过的空文件夹全消失）。实测：把 `pro.lproject` 临时改名成 `zz_proj.lproject`，
  `GET /api/pygraphs/projects` 照样读出 `title:"pro"` + `dirs` 五项（改回原名后复测资产 200）。
  默认工程（老图目录 `pygraphs/`）的挂载名是**空串** = 「引擎内容」，位置上对应 UE 的 `/Engine`。
  ⚠ **挂载名就是地址里那一段**：改它 = 改地址（老图里的子图引用 / 结构体 / 类方法 / 会话键全按它写），
  不是"换个显示名"，别随手改。
  **固定别名 `/pro` = 本工程**（2026-10-13 用户口径：「`/pro` 在任意项目都通用」）：首段除了
  「注册表里的挂载名」还认一个**固定别名**（`server.ADDR_ROOT_ALIAS = "pro"`）—— 写 `pro/…`
  （对外 `/pro/…`）= 「**本工程**的内容根」（UE 的 `/Game`）。⚠⚠ **它只在"有引用方"的地方成立**：
  · **图内引用**（子图 `sub.flow` / 结构体 / 类方法 / `TexFxMap` 子图）：宿主解析引用时把"引用方在哪个工程"
    塞进 `ContextVar`（`_project_of_path` + `_run_prompt_of` 的 set/reset）→ 同一段
    `/pro/Functions/x` **写在哪个工程里就指哪个工程**。**这条是别名的正当用法**。
  · **URL / API** —— ⛔ **入口不许写别名**（2026-10-16 用户口径：「没有项目名为 `pro`，不存在不应该能访问」）：
    没有引用方 ⇒ 定不出它指谁 ⇒ **400**；首段必须是**真工程名**，或用 `?project=<真名>`（头
    `X-Lugwit-Project`，中间件 `_no_store_html` 读）显式指明。报错原文：
    `地址首段 pro/ 是图内引用用的别名（= 本工程，同 UE 的 /Game）—— 入口地址不能写它：…`
  · 实现要点：`_alias_project(explicit, *, guess=False)` —— **"唯一登记工程"的兜底只在 `guess=True` 时生效**
    （以前默认生效 ⇒ "不存在的工程名也能访问" ✗）。`guess=True` 的**唯一**调用点是 `_trace_graph_agg`
    （轨迹按图地址挑轮次，纯比较，猜错最多少列几轮）。
  · 实测（curl）：`GET /api/pygraphs/pro%2F…` → **400**；`project_name%2F…` → **200**；
    图内引用照旧（探针 `subgpu` 通过）。
  ⚠ 别名只在**首段**生效；`pro` **被列为保留名**（`register` 里 400 拦掉）—— 否则别名永久遮蔽一个真叫 `pro` 的工程。
  ⚠ **还没接**两处：① 前端不带 `project`（单工程不需要；多工程要在编辑器/内容浏览器里补）；
  ② `l_agent_tool` 的 `pygraph_<图名>` 工具仍只扫默认 `pygraphs/`（**看不到工程内资产**，
  与"agent 跑 pygraph"是同一个口子）。
  **右键菜单就是一份**（2026-10-17 用户口径：「**群有资产和目录应该共用右键代码**」）：
  `PyGraphEditor.assetMenu(e, {kind, row, addr, abs, proj, label, onEnter})` —— **资产行 / 文件夹行 /
  格子 / 收藏夹那一行**四处都调它，统一走 `openContextMenu`（标题「资产 / 文件夹」），旧的
  `.pyg-ctxmenu` 那套（`ctxMenu` state + 独立 JSX）已删。条目按目标不同：
  · 资产（有 `row`）：**打开这张图 / 放到画布上**（= 点一下，`pickAsset`）· **在资源管理器里显示**
    （`revealAsset`，只对文件）· **复制地址**（`/工程/子目录/名`）· **复制绝对路径**（`abs` 有才出）·
    **★ 收藏 / ☆ 取消收藏**（`toggleFav`）；
  · 文件夹（`kind:"dir"`）：**进入目录**（`onEnter` = `setBrowserAt`）· 两种路径（没有"定位"/"收藏"）；
  · 失效的收藏（`row` 为空、只有 `proj`+`addr`）：只给复制地址 + ☆ 取消收藏。
  ⚠ 原来那份"在本地浏览器中显示"的老菜单只为树行服务（`POST /api/pygraphs/reveal {name}` →
  宿主 `explorer /select,<文件>`；**只打开文件管理器**，不执行资产内容；只接受解析得到的**真实存在的
  文件**）。⚠ 探针只验"菜单出现 + 文案 + Esc 关闭"，**不点那一项**（会在真机弹窗口）。
  **注销工程**（2026-10-13）：`DELETE /api/pygraphs/projects/{name}` —— 只从 `projects.json` 里
  拿掉那条，**绝不动磁盘文件**（删工程 = 删一堆资产，太危险；要删文件自己去磁盘删）。
  ⚠ 它必须注册在裸 `DELETE /api/pygraphs/{name:asset}` 前面（`projects/x` 两段两者都能匹配，先注册者胜）。
  ⚠ **空工程也要成组**：`assetTree` 原来只按资产行建组 → 刚 `register` 出来的空工程**界面上看不见**
  （"我建了工程但列表里没有"）→ 现在按 `projectMeta` 把没资产的工程补成空组（实测踩到）。
  **还没做**：把已有资产**归类**到对应文件夹（迁移只搬进工程根，没做分类）、新建资产时选"存到哪个
  文件夹"、贴图文件（`Textures/*.png|exr`）的列出 / 预览 / 导入、结构体/类资产的字段表编辑器。
  **工程元数据 `project.json`**（P1，2026-10-13）：`工程/project.json` =
  `{"kind":"project","title":"…","entry":"/工程/Blueprints/main","deps":[…]}`，存它就是同一条 PUT
  （`kind=project` 分支：`entry` 统一成**带前导 `/`** 的地址，并**校验那张图真的存在** —— 指错当场 400，
  别等点开才发现）。它既不是图（没有 `nodes`）也不是结构体 / 类 → 三个列表接口都不把它当资产列出来，
  靠 `GET /api/pygraphs/projects` 露出（每行带 `title` / `entry` / `deps`；`assets` 计数**排除** project.json）。
  内容浏览器里：组头显示 `title`，已设 `entry` 就给 `⟶ <文件名>` 一键打开；没设给「⤓ 设为入口」
  把**当前打开的那张图**写进去（不属于该工程的不让设）。入口**必须显式** —— 「谁先建的谁是入口」
  那种约定一改名/一新建就漂（UE 的 `.uproject` + 默认关卡也是这个道理）。
  ⚠ **资产地址的统一口径**（2026-10-13 定；UE 味）：**对外一律带前导 `/`**（`/工程/Functions/x`，
  `/` = 图目录根），**内部一律不带**（拼文件路径 / 会话目录 / URL 参数 —— 前导 `/` 在 `Path` 拼接里
  会变成绝对路径、在 URL 里会被转义成 `%2F`）。解析只有一处：`strip_addr()`，显示 / 引用时 `"/" + name`。
  **段数不再限两层**：工程里可以再分层（UE 式 `Functions/` `Classes/` `Textures/` `Blueprints/`），
  代价与判据：
  · 裸路由的多段转换器**必须躲开动作名**（`_ACTION_SEGS`：`run`/`engines`/`validate`/… 见 `_ACTION_GUARD`），
    否则 `{name:…}` 会把 `/engines` 一起吞成地址（`/api/pygraphs/x/engines` 那条路由注册在后面，
    **先注册者胜** → 404）；反过来，**资产自己也不许叫这些名字**（`_ensure_pygraph_name` 直接 400，
    免得"建得出来打不开"）；
  · 列表接口**全深度扫描**（`_iter_asset_files`：`rglob("*.json")`，跳过 `_*` / 隐藏 / `models` / `media`
    —— 那些目录里也有 `.json`，被当资产列出来就是错的）。
  ⚠ **踩过的坑（真凶级）**：改 `_ensure_pygraph_name` 时文件里还留着**第二份**老实现（"最多一层"），
  Python **后定义静默覆盖**先定义 —— 症状是"代码明明改了，三层地址一路 400"，白查一轮。
  挪这类函数前先 `grep "def <名字>"`。
  ⚠ **资产地址带类型后缀**（`Task.struct` / `Counter.class`：文件命名
  约定 `<地址>.json`，同名结构体与图才不会撞）—— 落节点时写的名字**带后缀**，树里**标签藏后缀**（图标已表明类型）。
  probe：`tools/web_probe.cjs` 的 **`assets` 场景**（24 项，实测 PASS；含"底栏标签条上有它""排在第一个"
  "切过去后日志不在同一屏"）。⚠ 底栏**就一块面板**，日志正文与资产树不可能同时在屏上
  （各由 `isActive("bottom", …)` 守卫）—— 探针里读日志前要 `pinLog()`、读资产树前要切回浏览器标签。
  **节点的引擎字段与查询**（2026-10-13）：节点的引擎**一直是图文件里的 `nodes[id].engine`**
  （属性面板「引擎」可改，空 = 跟随默认引擎；宿主混引擎跑图就按它切片，见
  `run_pygraph_api` 的 `engine_of`）。这次把它**变成可查的**：
  · `GET /api/pygraphs` 每行带 `nodes`（节点数）与 `engines`（这些绑定值的并集，空 = 全跟随默认）；
  · `GET /api/pygraphs/{图}/engines?default=<引擎>` **只读、不执行**，给**逐节点生效引擎**：
    `{"default":"local","bound":{"2":"muse"},"nodes":{"1":"local","2":"muse"},"engines":[...],"mixed":true}`
    —— "空即默认"这条规则集中在这里，调用方不用自己实现（内容浏览器的「本机」就是它的读数）。
  ⚠ 路由顺序：它比裸路由 `GET /api/pygraphs/{name:asset}` **先注册**（两者都吃两段路径，先注册者胜）。
  **结构体 / 类的端口与下拉**（2026-10-13）：`/api/pygraphs/ports?kind=struct&target=<地址>` 按那份资产
  的字段表**出字段端口**（`MakeStruct` / `ConstructClass` 的动态输入 —— 能连线喂值了）；
  `_patch_graph_name_ports` 同时把 `STRUCT_NAME_PORT` 的端口补成**资产下拉**
  （`MakeStruct.struct` 列 `/structs`、`ConstructClass.class` 列 `/classes`）。
  前端**只认元数据**：`NAME_PORT_KINDS`（`values.api` → 端口预取的 `kind`，见 `PyGraphEditor.jsx`），
  **不点名任何节点类** —— 宿主再加一类资产只需多一行。
  实测：从资产树落下的 `MakeStruct` 上 handles = `struct/fields/n/out0`（字段端口 `n` 按结构体现算）、
  `struct` 是 select。
  **子图/方法的多输出端口**（P2，2026-10-13）：端口弹回来带 `outputs`（引擎 `list_graph_outputs` 的顺序 =
  运行期 `collect_outputs`），编辑器用 `pySchema.applyOutputPorts` **按节点覆盖输出元数据**
  （`toFlow(graph, objectInfo, portsByNode)` 第三参）→ 引用的图有 2 个 `Output`，画布上就有 2 个端口
  （实测 handles `["out0","out1"]`，名字 `a_first`/`b_second`）。⚠ 端口 effect 原先只存
  `input/methods/signature/doc`，**把 `outputs` 丢了**（丢了就只能按静态 `RETURN_TYPES` 画一个）。
  v1 边界：方法名是**静态**的（一个节点一个方法）、**不生成真 python 类**、方法图里再定义方法不支持。
  还没做：**结构体/类资产的字段表编辑器**（目前得手改 JSON）、`project.json`（工程设置/依赖）、
  按引擎筛选内容浏览器。
- **python 节点的编辑器侧（2026-10-13）**：引擎的 `PythonCode`（见 `l_node_engine.md` §3.1.1）在宿主这边
  只多三件事，**没有新端点**：
  · **端口不问服务端** —— 端口表就在节点输入里（`inputs.ports` 的 JSON），编辑器就地算：
    `pySchema.portsOfNode(info, node)` → `portsToNodePorts()` → 并进 `portsByNode`
    （形状与宿主那份一致，所以画布 / 面板 / canvas 后端三处都不用改：`allPorts`）；
    判据是**元数据**里的 `port_spec_input`，前端**不点名节点类**（同 `NAME_PORT_KINDS` 的纪律）。
  · **两个新容器**：`code` → 只读高亮代码块 + 「✎ 编辑代码」（`PyCodeEditor.jsx`：textarea + prism 叠层 +
    行号）；`hidden` → 这一行不画（`ports` 那行）；**双击节点开编码浮层**（`PyCodeOverlay.jsx`：
    左端口表 ＋输入/＋输出、右大编辑器；300ms 合并写回、关窗 flush）。
  · **canvas 后端**画不了代码块 → 标题后缀 `🐍 双击编辑代码`（`visibleRows()` 还要把 `hidden` 那行滤掉，
    否则端口行号/高度估算两端不一致）。
  ⚠ 端口名非法（不是标识符）浮层里给 ⚠ 但**不拦**：引擎按名字取值，非法名只是取不到。

- **flow→pygraph 镜像生成器（2026-10-12）**：`tools/flow_to_pygraph.py` —— 把 `l_agent_chat` 的**流程图**
  （goto 状态机）翻成 **pygraph**（拓扑求值 + 括号区），产出 `pygraphs/default_intelligent_agent.json`
  （标着【镜像】，21 节点：步循环 = `LoopStart/LoopEnd`、跨轮上下文 = `LoopEnd.state` 回灌、
  只读工具档 = `glob:glob`、收尾 = `LoopEnd.stop` → `AIText` → `Output`）。
  **两条不可平移**（背边 / 多路 branch 与 guard 无原语）逐条落成 `fx_*` 便签；
  而 `branch_calls`（有调用去 tools / 没有去收尾）**2026-10-12 起是真 if 区**（引擎放开了
  "循环体里的 if"）——`has_calls` + `if_tool/end_tool`，工具只在真需要时才跑。
  `--verify` 打「flow 节点 → 落点」覆盖表 + 跑结构校验（改源图后重跑一次即可）。
  实测：结构校验 errors/warnings 全空、覆盖 16/16；真跑 2 圈收敛（`stop` 生效）、答案正确、`node_errors` 空。
- **策略包（2026-10-12）**：把一张**节点图**打成可分享/可切换的 JSON —— 顶栏 `📦 包 → 导出/导入`。
  包结构 `{kind: "lugwit-pygraph-pack", version: 2, exported_at, title, graph, baseline?}`
  （与流程图那边的 `lugwit-flow-preset` 是**两种 kind**，互不混）；导入**也吃裸图 JSON**（只有图文件时不该被拦），
  **重名自动避让**（`demo` → `demo-2` → `demo-3`…）。纯函数在 `web/src/pygraphPack.js`，
  单测 `web/tests/pygraphPack.test.mjs`（`l_mindmap_mmd_web_test` 一起跑）。
  **`baseline` = 对照图**（v2 新增）：导出时若顶栏 `A/B` 里选中了候选，就把它一并打进包
  （内存候选直接用，命名图先取回来；取不回来只提示、不阻断）；导入时那份对照图**装进 A/B 候选**
  （不落盘）→ 紧接着点「跑 A/B」就能比新旧行为。对照图格式不对会**明说**（不静默丢）。
  实测：导出 → 真下载 `demo_objops.pygraph.json`（`version=2`、`baseline_name=demo_all_nodes`）；
  同一份导入两次 → `pack_probe` / `pack_probe-2`；导入带对照图的包 → `；对照图「对照(>=)」已装进 A/B 候选` ✅
- **ComfyUI ⇄ 我们的图（2026-10-15）**：顶栏 `🧩 ComfyUI → 导入/导出`，与上游**两种 JSON**互导
  （纯函数 `web/src/comfyFlow.js`，单测 `web/tests/comfyFlow.test.mjs`，`l_mindmap_mmd_web_test` 一起跑）：
  · **API 格式**（"Save (API Format)"，值是 `{class_type, inputs}`）= 我们的图去掉 `pos` → **精确**、零猜测
  （实测拿上游 `ComfyUI/tests/inference/graphs/default_graph_sdxl1_0.json` 走一圈：11 节点、逐节点/逐入参一模一样）；
  · **UI 工作流格式**（`nodes` 数组 + `links`）= 连线精确，**控件值（`widgets_values`）是位置数组**，
  靠 `object_info` 里"有控件的入参"顺序对齐 —— 所以**先开「🔌 插件」里的 ComfyUI 节点**再导，否则满屏"节点库没有这类节点"。
  对齐的两条硬规矩（都写在代码里）：① 上游 `seed` 类入参后面会多一格 `control_after_generate`（多一格时跳过它）；
  ② **控件/连线的分界靠"类型"**（`INT`/`FLOAT`/`STRING`/`BOOLEAN`/候选表 = 控件；`MODEL`/`CLIP`/`CONDITIONING`/
  `LATENT`/`IMAGE` … = 只能连线）—— **不是** `forceInput`（上游 `KSamplerAdvanced` 的 `model` 根本没有它，
  当初按它判，控件值整体串位一格；真数据对拍抓到的）。对不齐 / 类不在节点库里时**一律报出来**
  （`widgetMismatch` / `unknownTypes` / `droppedLinks`，进日志），绝不静默丢。
  实测：真 `object_info`（65 个核心节点）+ 上游那份样例 → 导出工作流 11 节点 / 15 连线 / 0 unknown →
  导回来 **mismatch 0、值一致、坐标一致**。导出只给**工作流**那一种（能在 ComfyUI 里 "Load" 打开）；
  API 格式是另一半（`toComfyApi`，现成但先不占按钮位）。
- **沙箱 A/B（2026-10-12）**：`POST /api/pygraphs/{name}/ab`，body
  `{"candidate": "<图名 或 图对象>", "scenarios": [{"name": "…", "inputs": {"<节点id>": {"<输入名>": 值}}}]}`
  （`scenarios` 省略 = 一个空场景）。A = 当前图，B = 候选图，**同一批场景各跑一遍**，
  比 `steps` / `skips` / `ifs`（分支决策）/ `errors`，返回 `{verdict: same|differs, scenarios: [{a, b, diff}], a_valid, b_valid}`。
  **安全**：只真跑纯节点（`_AB_PURE`：比较/逻辑/如果/循环/取值/状态/接口），其余（`CallFunction`/`CallMethod`
  任意代码、AI 联网、`SaveText` 写盘、`Ask` 等人、对象节点建实例）一律**演练**（不执行、产出 `None`，结果里
  列在 `stubbed`）→ 这个接口**不改磁盘、不联网、不等人**；比的是**控制流与结构**，不是真实副作用。
  实测（改一个比较运算符 `>` → `>=`）：默认场景两边一致；把 `b` 覆盖成等值后 A 走假分支、B 走真分支，
  `diff` 出 `skips` + `branches` ✅。前端入口：顶栏 `A/B`（候选图下拉 + 跑）、右侧「A/B 回放」面板、
  日志逐条差异；默认场景 = `默认` + **`边界(a==b)`**（把每个 `Compare` 的 `b` 覆盖成它的 `a` ——
  `>` 与 `>=` 这类差异只在等值处冒，只跑默认值会误判成"行为一致"）。
  **审计落盘**（v2 新增）：每次比对追加一行到 `pygraphs/_traces/<图名>.ab.jsonl`
  （`{ts, a, b, verdict, scenarios:[{name, diff:[种类]}], a_errors, b_errors}`，`b` 为内联图时记 `(内联图)`）；
  接口 `GET /api/pygraphs/{name}/ab-log?limit=N`、`DELETE .../ab-log`；前端面板多一段「A/B 审计」——
  改图留痕，半个月后还能追问"当时比出什么"。保留最近 200 行。
  另外**结论也写进图文件的 `note`**（一行 `【A/B <时间>】vs <候选>：…`，写前删同前缀旧行 → **只留最新一条**）；
  body 带 `{"note": false}` 可关掉；前端同步内存里那份 note（否则下次「保存」会把结论覆盖掉）。
  实测：`vs ab_demo_b：行为一致` → 改阈值再比 → 同一行被替换成 `1 个场景有差异（branches/skips）`，`ab_lines` 始终为 1 ✅
- **轨迹 / 回放（2026-10-12）**：每次跑**逐步过程**追加一行到 `pygraphs/_traces/<图名>.jsonl`
  （`{ts,run_id,name,session,dur,ok,steps:[{node,dur_ms,skip?,bypass?,mute?,if_taken?}],asks:[…],errors}`）——
  与历史（`runs.jsonl`，只记"成不成"）分工不同：轨迹回答"**为什么这么走**"（谁先谁后、哪步被
  `IfStart` 跳过、问过什么/答了什么）。接口：`GET /api/pygraphs/{name}/traces?limit=N`（最近在最前）、
  `DELETE .../traces`。前端右侧面板有「轨迹」段，点「回放」在画布上按步高亮走一遍
  （复用"正在执行"的流光高亮；被跳过的步骤在步骤条上标斜体）。保留最近 200 次，单次最多记 2000 步。

### 节点的**身份**、复制粘贴、贴给 AI 的一行引用（2026-10-15）

用户口径：「当前节点支持复制粘贴么，有唯一身份么，方便我发你图中的某个节点和你对话」。

- **身份 = 图地址 + 节点 id**：`project_name/Functions/demo_tex_basics#12 · Time（时间）`（图地址首段=**真工程名**；早先这里是 `lugwit_agent/…`，工程改名成 `project_name` 后一并更新）。
  图地址就是内容浏览器里的**规范地址**（`工程/子目录/图`，与编辑器 `?name=` 同口径）；节点 id 是图文件
  `nodes` 里的**键**（新建时 `S.nextNodeId` 递增给），随图存盘、重启不变。
  ⚠ id 只在**这一张图**里唯一 —— 跨图重名很正常，引用必须带图地址。
  **提一句话给我就能定位**：这条引用串是我（AI）认路的坐标（图文件里的节点键 + 图地址），
  比"那个噪波节点左上角那个"靠谱得多。
- **右键「📋 复制引用（贴给 AI 用）」**：把上面那行（含类型 / 显示名 / 前 6 个字面量参数 / 前 60 字注释）
  拷进剪贴板 —— 直接粘进对话即可。
- **Ctrl+C / Ctrl+V** 复制粘贴节点（与 `FlowEditor` 的键位口径一致）：复制**只带字面量参数 + 标志位 +
  注释 + 尺寸**，**不带连线**（粘到别处上游多半不在，带过去就是死线）；粘贴落在原节点右下 40px 并选中它。
  在输入框里打字、或画布里**选了文字**时不抢这两个键。右键「⧉ 复制节点」＝ 原地再放一个（同一个实现口径）。

### 🧰「工具卡」底部的输入框 + 参数行（一句话跑这张图，2026-10-15）
用户口径：「**工具卡面板底部加一个输入框按钮，参数使用 agent 图的入参**」→ 接着
「**输入框要可以修改给 agent 图的参数，参考 l_agent 的输入框**」。

- 落点：`ToolPane.jsx` 底部两块 —— 上面 **🎛 参数行**（`.pyg-toolpane-params`）+ 下面
  **一句话输入框**（`.pyg-toolpane-ask`：**共享件 `ComposerBox`**（= 聊天输入框那口框），
  框里是多行 `textarea.composer-input`（Enter 发送、Shift+Enter 换行）+ 右下 `▶ 跑`；
  跑着时禁用）—— 用户口径 2026-10-15「那排上面那个文本框…要连输入框本身也一模一样」→
  「自己截图对比一下…**要一模一样**」⇒ **尺寸也照聊天那口框**（不覆盖 `--cs-send-size` 等变量），
  那排控件放**框内右下**（`.composer-tools`，聊天同一个落点），其余旋钮（模型 / 图自己的参数 /
  服务端设置）收进框边「⚙」的浮层（聊天把它们放 `/panel`）。
  ⚠ 那口框与那排的**顺序/字面**也照聊天那排：模式 → 智能体 → 权限 → 思考 → 步数；没表态那档
  优先用聊天原话（`Agent` / `默认` / `步数默认`），没有原话的写「<中文名>（跟随）」。
- **参数行是自动生成的**（不写死）：扫本图的
  · `Input` 节点（= 图接口：`n` …；⚠ `prompt`/`question`/`ask` **不列** —— 那是下面输入框的事）；
  · `AgentParams` 里 **`source: "param"` 的行**（参数卡：`model`/`effort`/`mode`/`max_steps`/
    **`permission_mode`（审批 —— 卡里默认 `allow_all` 全放行，2026-10-15 用户口径）**…），
    连 `widget`/`options`/`min`/`max` 一起用（协议与引擎 `param_spec.ROW_KEYS` 同一份；
    `pySchema.parseParamSpec` / `paramRowsToText` 现在**原样带回来**，不然浮层里过一遍手就丢控件形态）。
    ⚠ **`literal` 行不列**：那是卡里写死的常量（`AgentParams.execute` 不读接口），列出来也改不动。
  · 值**按图**记在 `localStorage["pygraph.params.<图地址>"]`；空 = **不表态**（不塞进请求，回落链才生效）；
    旁边「↺ 清空」清掉这一图的覆盖。⚠ 载入时**按"现在还存在的行名"剪一遍**并擦掉脏键 ——
    老版本 UI 留下的键（删掉的行 / 改了名的键）会被继续发出去，实测踩到过
    （删了"思考开关"行，`thinking=true` 还在；选了"20 步"，却还发着旧的 `n=3`）。
  · **控件形态与 l_agent 输入框那排同源**（用户口径 2026-10-15「尽量一模一样，不要全是文本输入框」）：
    · **与聊天那排同名同序**（2026-10-15 追评「**并没有共用组件**」之后的对齐）：那一排与聊天输入框
      底部那排**同一个组件**（`@pyshare` 的 `OptionBar`/`TightSelect`），顺序也照聊天那排
      **模式 → 权限（图上的「审批」）→ 思考 → 步数**（智能体那格在 `left` 槽、位置也一致），
      其余的（模型 / 图自己的参数）跟在后面。
    · **不印英文参数名**（这是追评那条的根因）：能自解释的下拉一律不印名字 —— 选中的是中文档位名
      （`全放行`/`低`/`20 步`）；没表态那一档写成 **「<中文名>（跟随）」**（模式（跟随）/ 思考（跟随）/
      模型（跟随）…）⇒ 一排全是不重名的中文、**一个名字标签都不用印**，DOM 与聊天那排"只有下拉"同形。
      只有**复选框/自由输入**这种不自解释的才印中文名（行上的 `label`，没有才用行名）。
      身份另有 `data-param` 属性与悬停 `title`（自证按它们判，别拿可见文字当判据）。
    · **服务端设置单独一行**（`.pyg-toolpane-settings`，更淡、带「全端设置」小标）：权限（全局）/
      查错字 / 查错超时 —— 它们不是"这一轮的参数"，与上面那排挤一起就把"和聊天那排一样"弄没了；
      这三件**印中文短名**（两个「默认」不印名字分不出来）。
    · **静态候选**：行上的 `options` + 共享件 **`l_py_ui/paramLabels.js`** 的中文档位名
      （`effort: 低/高/最大/关闭`、`max_steps: 20/40/…/200 步`、`permission_mode` 有别名表…）；
      参数卡浮层（`PyNode.ParamControl`）同一份。图里那个默认值**直接当选中项显示**（`allow_all` → 全放行）,
      「↺ 清空」才是回到"不表态"的入口。
    · **动态候选**：行上的 **`values`**（与节点端口的 `opts.values` **同一份协议** `{api,pick,value,label}`，
      引擎 `param_spec.ROW_KEYS` 也加了它）→ 编辑器用 `valueSources.loadValues` 拉一次塞进 `paramChoices`；
      新端点 **`GET /api/pygraphs/agent-models`** 把 agent 的模型清单转回来（先 `available.models`、
      空则回落 `declared`；值 = `厂商/模型`）⇒ 那个「模型」下拉里是**这台机器当前真能用**的清单。
      模型的 `厂商/模型` 由 agent 侧拆开（显式 provider 优先），所以**没有单独的 provider 行**。
    · **步数那组去重**：卡里已有 `max_steps` 时不再单列图接口的 `Input(n)`（同一个旋钮的两个名字；
      宿主盖章时当一组别名）。
  · **这一排就是聊天那排本身**（2026-10-15 后半场，用户口径「工具卡的输入框请和 lagent 一模一样，
    **最好可以共用同一套代码**」）—— 组件与样式都搬进 `l_py_ui`，两处渲染**同一份**：
    · `OptionBar.jsx`（那一排的壳：`.opt-bar` + 每项一个 `.think-effort-inline` label + `left`/`right`
      两个插槽）、`TightSelect.jsx`（紧贴当前文字宽度的下拉，带可选 `className`）、
      `composerRow.css`（观感：值照抄聊天的 `.composer-tools`/`.think-effort-inline`/`.tight-select`，
      对主题变量留 `var(--x, fallback)` ⇒ 聊天那边逐像素不变、画布用回落值）。
    · 聊天 `NewApp` 那排改成 `<OptionBar className="composer-tools" ref={toolsRef}
      left={命令按钮组 ⏳/✨/🎤/＋附件/📷} items={模式/智能体/权限/思考/步数}>` ——
      **根类名仍是 `.composer-tools`**（位置、横向滚动、逐项收放
      `[data-collapsed~="mode"] .composer-tools .mode-pos` 全靠它，换名就散架）；
      本文件原来那份 `TightSelect` 实现已删（现在只有共享件那一份）。
    · 画布这排 = 同一个 `OptionBar`，只有两处宿主差异：`name`（控件前那点小字 —— 画布上光一个
      数字框看不出是哪个参数；**聊天不传** ⇒ 聊天 DOM 与以前逐字节一样）；`flex-wrap`（画布参数多，
      靠宿主 CSS `.pyg-toolpane-params` 换行；聊天的**不换行 + 横向滚动**是它的兜底，所以共享
      CSS 里刻意**不写** wrap）。
  · **智能体那一格**：聊天是"选一个命名智能体"，画布上没那个概念（**图就是智能体**）⇒ 换成
    **切图下拉**：候选 = 列表里 `…/agent/…` 的图（`listAssets()` 现取），选了就 `openGraph(那张)`；
    选完**不发任何 `params`**（换了图 = 换了整套控制流）。
  · **服务端设置**（权限 / 查错字 / 查错超时）也进了这排，但**不属于 `params`** —— 它们是 agent 的
    **服务端设置（全端生效）**，走宿主白名单代理 `GET|POST /api/pygraphs/agent-settings`
    （`server.py::AGENT_SETTING_KEYS` 只放这三件：agent 的 settings 里有明文 `lugwit_password`，
    原样透传就是泄露）。改一次 `say` 一条并把服务端回读值对账上。
  · **没搬的**：`模式`（ask/plan/goal/review）—— 那是**聊天那层**的对话模式，画布这条链没接；
    `multi` / `attachments` 是聊天专属。
- **提交**：文本 → `question`；控件那排 → `params: {名字: 值}`（`pyApi.runGraph` → 宿主 `payload["params"]`）。
  空则都不发（留空 = 用图里声明的那份）。
- 宿主怎么落地（`server.py`，**两条都做**）：
  · `_stamp_run_params(prompt, params)` —— 按名字**盖章进 prompt 的 `Input` 节点**（`Input.execute`
    回的是**自己的 `value`**、**不读**图接口 ⇒ 不盖章就是"发了参数、图里纹丝不动"，实测踩过：
    `params {"a":99}` 跑完 `Input a` 仍是 20；盖章后 99 / 下游 121）；
  · `interface=params` 照旧传（读接口的是 `AgentParams` 的 `param` 行与工具 schema）；
  · `question`/`n` 的回落链变成 `params > body > 图里 > 8`（`_run_steps_of`）。
  · 其余键原样随画布会话转给 agent（`_AgentStepScope(..., params=…)`）。
- agent 侧（`l_agent_chat` 的 `POST /api/graph-step/session`）：收 `params` → 落到
  `TurnContext(model / provider / reasoning_effort / thinking_on / max_steps)` —— 这几个字段**本来就有**，
  只是画布这条路以前**根本不读参数卡**（图里卡上写了 model/effort 也白写）。回显 `params`
  （只回声"真给了"的键，用于判据；`_AgentStepScope` 再把它塞进 SSE 的 `agent` 事件）。
- 只影响**这一轮**，不写图文件、不改入参默认值（要改默认值去「⇄ 接口」格 / 双击卡片）。
- 探针：`web_probe.cjs askbar`（**7/7**）—— ① 写的字进请求体 `question`；② 留空不发（用图里那句）；
  ③ **填 `a=99` 跑一次 → 画布上那个 `Input` 卡片的「本轮值」就是 99**（判据落画布上的值，
  不是"请求体里有没有 params"—— 只验请求体就漏掉"`Input` 不读接口"那个坑）。
  · **共享件的判据**（`panes` 场景，2026-10-15 后半场，**24/24**）：① 那口框 = 共享件 `ComposerBox`
    （`.pyg-toolpane-ask` 同时带 `.composer-box`、编辑器 `.composer-input`、**算出来圆角 16 + 有底色**）；
    ② `.pyg-toolpane-params` 同时带 `.opt-bar`（= 真用了那个组件，不是照抄样式）、项都是
    `label.think-effort-inline`；③ 下拉算出来 **20px 高**（聊天那排的实测值 ⇒ 观感同源，不是两套 CSS 碰巧像）；
    ④ `.pyg-tparam .opt-name` 看得见参数名；⑤ 有「智能体（切图）」下拉（`graphs` 接线生效）+
    「审批」行（`permission_mode`，提示里是**中文**默认值"全放行"）。
  · 聊天那侧的判据在 `l_agent_chat/web/e2e/newui_check.js`（同轮加的 7 项）：根同时带
    `.composer-tools` 与 `.opt-bar`、四个下拉都在（模式/权限/思考/步数）、每项是共享件的 label、
    命令按钮组（＋附件/🎤）仍在那一排、下拉宽度紧贴文字（`TightSelect` 生效，实测 60px）；
    那口框本身另由 `run_queue_e2e` **26/26** 守着（发送/排队/编辑/打断/停止 —— 真走那口框）。
  ⚠ 顺带修好的一个显示 bug：**单输出节点**的值在 SSE/盘上是"一层包"（`[99]`），原来直接喂给
  `shortScalar` 得 `null` ⇒ 卡片上永远空（`Time` 那种多输出的不受影响）；现在先脱一层再判标量。

### 💭 工具卡里的「思考」卡（2026-10-15）—— 与聊天那侧对齐的第一半

用户口径：「**工具卡和 lagent 还是显示不一致，比如没有思考组件**，思考组件现在也是一个工具么」。

- **答案（工具那半）**：是 —— 契约与注册表里它都是**内置工具**（`工具契约_ToolSpec.md` §7；
  `l_node_engine.tool_spec.builtin_tool_specs()` 第一条，`GET /api/pygraphs/tools` 里看得见
  `thinking`：无输入、一个 `text` 输出、`read_only`）。但**画布上的"思考节点"还没做**（S4 的事）。
- **补齐的那半（画布侧原来什么都没有）**：
  · 数据：`graph_turn.TurnContext.reasonings` 攒每一圈规划拿到的思考正文 `{t, text, secs, round}`
    （空思考不记）→ agent 的 `/api/pygraph-live` 多一个 `reasoning` 字段（宿主那个端点**整份透传**
    agent 的 JSON，所以不用改宿主）；
  · 组件：共享件 **`l_py_ui` 的 `ThinkingCard`**（`ThinkingCard.jsx` + `thinking.css`；正文走
    `ResultView` —— 契约原话"聊天里 = 现有那块思考内容 UI（同一个 `ResultView` 渲染）"）；
  · 落点：`ToolPane` 的卡片区**顶部**（过程块下面、工具卡上面），按时间顺序一张一段，
    `secs` 显示这次规划往返的耗时；跑着时最后一段默认展开。
- ⚠ **还没同源**：聊天那侧仍是它自己那块 `Reasoning`（带翻译 / 朗读 / 流式秒表 / 与消息状态联动的
  自动折叠）。要"一份渲染"得把那几件搬进 `ThinkingCard` 再让聊天改用它 —— **没做**。
- ⚠ **只覆盖"对话驱动"那一档**：`_LIVE_PYGRAPH` 由聊天驱动那条路填（`app.py` 的 pygraph 分支）；
  编辑器自己点 ▶（画布会话 `/api/graph-step/*`）那档目前**没有** reasoning 数据。
- 自证：真跑一轮（读 AGENTS.md 第一行）→ live `reasoning` 1 段（`secs 3.4`）→ 画布上
  `💭 思考过程 3.4s` 一张卡，展开就是那段原文；探针 `panes` **24/24**（判据里**画布上思考卡的数量
  = live 里思考段的数量**，没数据时两边都是 0 也算过；另 5 条是"参数排 = 共享件"，见上节）。

### 「工程与图路径分开写」的 URL（2026-10-16）

用户口径：「想把工程与图路径分开写」。**入口地址首段必须是真工程名**（别名只许出现在**图内引用**里）：

```
✅ ① 真名直接进图路径（**URL 与 API 都认**，最通用）：
   /pygraph?name=project_name%2Fdemo%2Fdemo_perf_stress&tab=viewer2d
✅ ② 工程名显式分开写（**编辑器那条 URL** 用；前端 `initNameFromUrl` 会把两者拼成完整地址再发 API）：
   /pygraph?project=project_name&name=demo%2Fdemo_perf_stress&tab=viewer2d
⛔ ③ 别名当入口：/pygraph?name=pro%2Fdemo%2F…  或  ?project=pro&… ⇒ **400**
```

⚠ **API 侧只认①**（`_pygraph_path` 是**单参**：地址自带真名首段）—— `?project=<真名>` / 头
`X-Lugwit-Project` 由**中间件**校验（值是别名 → 400），但它**不参与**"相对路径 + 工程名"的拼接：
那一步在**前端**（`initNameFromUrl`）。别在 curl 里指望 `?project=<真名>&name=<相对路径>` 能取到图。

⚠ **入口地址不许写别名 `pro`**（2026-10-16 用户口径：「**没有项目名为 `pro`，不存在不应该能访问**」）：
`pro` 是"本工程"（UE 的 `/Game` 那种**相对根**），它只在"**有引用方**"的地方成立 ——
**图内引用**（子图 `sub.flow` / 结构体 / 类方法 / `TexFxMap` 子图）由宿主按"引用方属于哪个工程"
确定地解析（`_resolve_addr` 里 `_addr_root.set(_project_of_path(p))`，**不是猜的**）；
而 URL / API 是**入口**，没有引用方 ⇒ 定不出它指谁 ⇒ 400（文案点明"没有工程名叫 pro"，并给两条出路：
写真名 / `project=<真名>`）。
服务端实现：`_alias_project(explicit, *, guess=False)` —— `guess` 默认 **False**（入口不猜）；
**唯一**传 `True` 的地方是 `_trace_graph_agg`（轨迹按图地址**挑轮次**，纯比较用途，猜错最多少列几轮）。

#### 顺着这条口径做了一次"同类根因"审计（2026-10-16，从根上修）

同族问题 = "**把别名当工程名用 / 拿别名当入口**"。审出 5 处：

| 落点 | 审计结论 |
|---|---|
| **`_project_root(name)`** | ✗ **别名盲**（只拿首段去注册表找同名工程 ⇒ `_project_root("pro/Functions/x")` 返回 `None`，调用方静默失败）⇒ **已修**：首段是 `pro` 时先 `_alias_project()` 解开 |
| **`_alias_project` 的"唯一登记工程"兜底** | ✗ **猜**（"反正只登记了一个就拿它当本工程" ⇒ `?name=pro%2F…` 也能访问 = 用户说的"不存在却能访问"）⇒ **已修**：兜底挪到 `guess=True`，默认**不猜**（只有轨迹比较那处传 True） |
| **注册工程** | ✗ 以前 `pro` **可以被注册成真工程名**（别名永久遮蔽它）⇒ **已修**：`register` 用**保留名**拦掉（400 + 出路：改工程名 = 改它根目录里 `<名>.lproject` 的文件名） |
| **写 `.lproject`**（`recent`/`favorites`/`links`） | ✅ **本来就对**：`POST /projects/{recent,favorites}` 写盘前都过 `_canon_recent_addr`（别名 → 派生名）—— 别以为它没归一 |
| **前端拼 URL**（`pushGraphRoute`） | ✗ 以前**自己造别名形态**（首屏/切图把地址写成 `name=pro%2F…`）⇒ **已修**：第 3 个参数收"当前图所属的**真名**工程"（`curProj`），能拆就写 `project=<真名>&name=<相对路径>`（首段是别名 / 真名都走这条）；定不下真名时**原样保留**地址 —— **绝不造别名**（造出来下一次刷新就是 400） |

实测（都进了探针）：
- `_project_root`：`project_name/demo/fire_time` → **工程目录** ✓／`demo/fire_time` → `None` ✓（首段不认识就别猜）／`''` → 默认根 ✓／
  `pro/demo/fire_time` → **只在"图内引用"上下文里**（`_addr_root` 已设）解成工程目录 ✓；**没有上下文时 → `None`** ✓
  （2026-10-16 起不再"唯一登记就自动认领" —— 那正是"工程名不存在也能访问"的来源）。
- 入口：`GET /api/pygraphs/pro%2Fdemo%2Fdemo_perf_stress` → **400** ✓／`project_name%2F…` → **200** ✓／
  `?project=pro` → **400**（中间件拦）✓／`project_name/…` + `?project=project_name` → 200 ✓（curl 实测）。
- 注册 `pro` → **400**（保留名）✓。
- **图内引用不受影响**（关键）：`_run_prompt_of` 是**先设上下文再解析**（`_addr_root.set(_project_of_path(p))`），所以 `sub.flow = "pro/Functions/x"` 照旧成立 —— 探针 `subgpu` 回归通过。
- 探针 `spliturl` **7/7**：真名分开写 ✓ / 首段真名 ✓ / **`?name=pro%2F…` API 400** ✓ / **`project=pro` API 400** ✓ / 页面上画不出图（0 节点）✓。
- 探针自身的入口地址**不再写 `pro/…`**：`TMP_NS` 运行时从 `GET /api/pygraphs/projects` 取第一条**有名字**的工程
  （第一条常是"默认根"、`name` 为空）⇒ **工程改名免疫**，代码里不再出现"工程叫什么"。
  ⚠ 探针临时图里的**子图引用**也要写完整地址（`probeAddr(ASSET)`）—— 裸名会被解析到"默认根"⇒ 内联 400
  （2026-10-16 踩过：`subgpu` 因此假红）。

- **读**：`initNameFromUrl()` 读 `project=`（也认 `?project_name=` 那种写法）→ 与 `name=` 拼成完整地址
  交给下游（`pyApi` / 引擎 / 缓存键**口径一点没变**，还是 `<工程名>/<相对路径>`）。
- **写**：`pushGraphRoute()` 在切图时，若地址本来就带工程名，就把工程名留在 `project=`、`name=` 只写
  **相对路径** ⇒ **工程改名时链接、书签、分享出去的那串都不用动**（这正是动机：2026-10-16 当天
  工程从 `lugwit_agent` 改名成 `project_name`，所有带完整地址的链接全废）。
- 自证：探针 `spliturl`（5/5）—— 两种形态各开一次，都要求"节点真加载、工程被正确解析"，
  且分开写那形态里地址栏 `name=` 保持相对路径。

## URL

| 路径 | 说明 |
|---|---|
| `/` | 编辑器主页（流列表 + 画布） |
| `/flow/{name}` | 打开指定流程图 |
| `/settings` | 兼容旧链接：进入编辑器并自动打开右侧「连线显示设置」面板 |

## 登录授权（lugwit_auth 统一闸门，2026-10-11 接入）

页面与 API 都走**统一登录**（`rez-package-source/l_mindmap_mmd/999.0/src/l_mindmap_mmd/gate.py`），
闸门原语来自 `lugwit_auth_client.gate`（**唯一实现**，本包只接线）。整体台账见 `各包登录接入台账.md`。

| 情形 | 行为 |
|---|---|
| 未登录 **页面**（`/`、`/classic`、`/settings`、`/flow/x`、`/pygraph/x`，或 `Accept: text/html`） | **302** `http://<入口>/login?next=<反代前缀+路径>` —— 登录页在 **nginx 入口根**（交给 lugwit_auth），**绝不能加前缀**（`/mindmap-mmd/login` 是 404 白页，网盘 / model_hub 都踩过） |
| 未登录 **API** | **401** `{"detail": …, "auth_required": true}` |
| 已登录（cookie `lugwit_token` / `Authorization: Bearer <token>` / env `LUGWIT_ACCESS_TOKEN`） | 照常；`GET /api/auth/me` 回 `{authed, user, role, role_label, local_direct}` |
| **本机直连**（peer 回环 **且** 无 `X-Forwarded-*`）的 **只读** 请求（GET/HEAD） | 免登录（本机 curl / 脚本 / 直接开 `:8110` 照旧）；**写类与 `/api/pygraphs/{name}/run` 仍要登录** |

- 开关：`L_MINDMAP_LOCAL_NO_AUTH=0` 关掉"本机直连只读免登录"；`LUGWIT_AUTH_URL` 指定 auth 地址
  （默认 `http://127.0.0.1:8080`，服务端调 auth 一律走回环 nginx 入口）。
- ⚠ **这条豁免要求 nginx 真带 `X-Forwarded-For`**：`/mindmap-mmd/` 与 `/mindmap-mmd-dev/` 两个
  location 一旦自己声明 `proxy_set_header`，http{} 级的同组头全部失继承 —— 少了 XFF，经反代进来的
  请求会被判成"本机直连"（2026-09 同类事故）。所以这两段**整组重声明**（见 `l_nginx/conf/routes.conf`）。
  连带加了 `proxy_redirect ~^/login…`（保住根级登录页）与 `proxy_buffering off`（SSE 端点别攒着吐）。
- 实测（2026-10-11）：未登录走反代 `/mindmap-mmd/pygraph/demo_all_nodes` → 302 → `…/login?next=/mindmap-mmd/pygraph/demo_all_nodes`；
  `/api/pygraphs` → 401；带 cookie 反代读写 → 200；直连 GET → 200、直连 PUT → 401、直连 PUT+Bearer → 通过；
  浏览器经网关打开编辑器且 `/api/auth/me` 回 `admin01 · 管理员`（`local_direct: false`）。
- 依赖：`requires` 里加了 `lugwit_auth_client`（轻依赖客户端包）。
- ⚠ 改了 `lugwit_auth_client`（共享包）后要 **reload 本服务** —— 热更只盯本包目录，共享包的改动
  只有重启进程才生效（同 `l_node_engine` 那条坑）。
- 自证怎么跑：改 `gate.py` → 重建无关，但**必须 reload 服务**，然后按上表逐条 curl
  （反代页面 302 / 反代 API 401 / 带 cookie 200 / 直连 GET 200 / 直连写 401 / 直连写+Bearer 通过
  + `GET /api/auth/me`）。⚠ 用 `curl` 而不是 `urllib` 看 302：`urllib` 默认**跟随跳转**，
  会把"302 到登录页"显示成 200 的登录页 HTML（我自己就先被它骗了一次）。

## 与环境、与 pygraph 的关系

`/flow/<名字>` 是 agent 流程图；**`/pygraph/<名字>` 是节点 python**（`l_node_engine` 引擎，
跑任意 python 对象/函数/类），用法/执行面开关/白名单/AI 节点后端见 **`l_node_engine.md`**。
AI 节点的地址与凭据也**不用环境变量**：`ai.json`（**数据目录，与 `pygraphs/` 平级** —— 别放图目录里，
那里任何 `*.json` 都会被当成一张图、还会跟用户的图名 `ai` 撞车；面板「AI 服务」可写，
`GET|PUT /api/pygraphs/ai-config`，凭据只回掩码）。
**两条"下拉候选"转调口**（都 60s 缓存、`?fresh=1` 跳过；凭据 = 引擎身份 → 退回浏览器会话）：
`GET /api/pygraphs/ai-models`（模型清单，`tags` 供前端按能力筛）与
`GET /api/pygraphs/video-providers`（**视频厂商**：转模型中心 `/v1/video/models`，把六家塞进
`models` 形状 —— `key` = provider id、`available` = 配没配钥匙；`AIVideo.provider` 的下拉读它）。

## pygraph 的 `models/` 读口与参数值口径

**两个路由**（`server.py`）：编辑器预览、以及"值到底存不存在"都看它们。值里存的是**文件名**，
光名字由引擎 `l_node_engine.model_path()` 补成 `models/` 下的绝对路径。

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | `/api/pygraphs/models/upload?name=<原始文件名>[&newdir=1][&dir=<子目录>]` | 上传模型；body = **原始二进制**（分块落盘，大文件不进内存）。落盘名由服务端清洗（`_safe_model_name`：只留字母数字/`._-`、限长、重名加 `-1` **不覆盖**）→ 回 `{name, dir, path, url, bytes}`，**写回参数值的就是 `name`**（相对名） |
| GET | `/api/pygraphs/models` | **列可用模型**（数据目录 + 包内内置）→ `{models:[{name, bytes, builtin}]}`。同名以**数据目录**那份为准（`setdefault`：先扫数据目录、再扫包内）；多文件 glTF 只在子目录里列主文件（`.gltf`/`.glb`）。编辑器把它当 `<datalist>` 用（`PyModelPicker`）—— 内置模型靠它被发现，**datalist 不是下拉框**：不占版面、手填照旧 |
| GET / HEAD | `/api/pygraphs/models/{name:path}` | 读 `models/` 里的文件：`<文件>` 或 **`<服务端子目录>/<文件>`**（多文件 glTF），其余形态（穿越 / 两层以上 / 反斜杠）一律 400。**落点顺序 = 数据目录 → 包内内置**（见下） |
| GET / HEAD | `/api/pygraphs/models-local?path=<绝对路径>` | **受控只读读口**：预览白名单根下的本机模型（见下） |
| GET | `/api/pygraphs/models-local/roots` | 白名单根清单（编辑器用它给"不在白名单根下"的明确提示） |

- **内置模型随包分发（2026-10-13）**：`<包>/src/l_mindmap_mmd/pygraphs/models/`（目前 `torus.obj`）——
  读口**数据目录优先、没有再回退包内**（`server.BUILTIN_PYGRAPHS_DIR`）。所以内置模型不用先上传一份就能用，
  用户自己传了同名文件就用自己那份（也能"覆盖"内置的观感），包内那份改了就立即生效（无需重启，读写都是现读）。
  值仍写 `torus.obj`（引擎 `model_path()` 补成数据目录下的绝对路径 —— ⚠ 那份**不存在**，
  所以服务端"按路径读模型文件"的代码**不能用** `model_path()`，得走同一个回退；
  目前没有这类代码：模型解析全在浏览器侧 `MeshViewer`）。
- ⭐ **3D 视图认谁：`3d_view` 节点说了算**（2026-10-17 用户：「我觉得统一为使用 `3d_view` 节点来显示结果」
  → 拍定「三个输入都接，如果某个端口没接就使用 3D 视图的设置」）。三路**各判各的**：
  `model`（模型名，可连线；连线时解析源节点的字面值，`GetParam` 去图级 `params` 查）/
  `mesh`（接一个出几何的节点 → 看它的几何）/ `material`（`MATERIAL` → `MeshViewer` 默认贴它，`preferMat`）；
  **没接的那路回落到这一格自己的下拉**，再退老口径（图级 `params` 里第一条 `type=model` → 未迁移的
  `Input(type=model)` 节点 / 第一个 `MATERIAL` 节点）。⚠ **判"图里填了什么模型"必须看 `graph.params`**：
  `Input` 节点类型已于同日退休（`Input` → 图级 `params` + `GetParam`，`migrateInputNodes` 打开就迁）。
  ⚠ `texGpu.js` 的 `KINDS`/`SINK_KINDS` 也要有 `3d_view`（少一个整段退回 CPU）。细节见
  `l_tex_nodes.md` §6.2c；自证 `web_probe view3d`（9/9，含真图竹编：模型 + 材质都按 `v3d` 走）。
- **`mesh` 那路谁来喂**（2026-10-20）：图里算出来的几何直接接 `3d_view.mesh` —— 老路子是
  `math_shapes`（`CallFunction` / `ShapeSurface` 出 `(顶点, 面)`），新的是插件 **`mesh3d`**
  （37 个 `Mesh*` 节点（含**绑定 MeshRig/MeshSkin/MeshPose**、**属性 AttrSet/AttrGet/AttrPromote**、**减面 MeshDecimate**、**UV 投影**、**OBJ 带 vt/MTL 与 PLY 带顶点色**、选择集/区域移动/Catmull-Clark/曲线+扫掠/删面/补洞/内缩/挤出/桥接/布尔/合并顶点），值 `{vertices, faces}`，见 `l_node_engine.md` §3.0d″；示例图
  `lugwit_agent_a/Functions/demo_mesh3d` 程序化山石、`demo_teapot` 茶壶、`demo_mesh_edit` 挤出+布尔+桥接、`demo_lion` 风格化狮子、`demo_mesh_refine` 选择→捏→曲面细分、`demo_mesh_curve` 曲线扫掠+内缩、`demo_geo_attrs` 属性 Cd 上色）。宿主 `mesh_preview`
  两种形状都认，链上任何一步都能看。
- ⭐ **🔥 热力 = 一格面板**（2026-10-17 用户：「**删除热力按钮，使用热力面板替换**」）：顶栏那颗
  `📈 热力` 删了，开关 / 轮次 / 步骤 / 本轮元信息（`ts · mode · steps · dur · end=`＋门拦）/ 全量汇总
  （轮·节点·边 + 最热节点标签）全在「🔥 热力」那一格（`put("heat", …)` → `.pyg-heat`，住**底栏**）；
  画布上只留叠加层（`TraceOverlay`：圆点数字 = **全量**走过次数，黄圈 = 「步」游标）。
  数据仍是 `GET /api/pygraph-heat`（按本图地址挑 `_traces/*.jsonl` 里 `mode=pygraph` 的轮次）。
  ⚠ **不许跟画布同格**：那格是"谁激活挂谁"，同格的话一开热力画布就被卸掉、叠加层一个圆点都看不到
  —— `pyLayout.dedupeAll` 里有一条**迁移规则**把它搬回底栏（第一次落地时 `home` 填错过，用户的存档
  已经长在画布那格；规则幂等，单测 `pyLayout.test.mjs` 钉住）。自证 `web_probe heat`（12/12）。
- ⚙ **设置也只有面板**（2026-10-17 用户：「设置顶部的设置按钮，有设置面板就行了」）：顶栏那颗
  「⚙ 设置」删了 —— 右栏那张「⚙ 设置」标签就是全部（与流程图**同一个组件、同一份 `settings.yaml`**）。
- ➕ **「＋ 分组框」按钮也删了**（2026-10-17 用户：「删除分组框按钮」）：要加框走**画布右键菜单
  →「新建分组框」**（`addGroupAtViewport` 仍在）；框本身的行为（拖框带成员 / 缩放 / 命名 / 属性面板
  删除 + Ctrl+Z）一点没变，自证仍是 `web_probe group`。
- 📦/🧩 **包与 ComfyUI 的「导出 / 导入」搬进「打开工程」下拉**（2026-10-17 用户：「是加到顶部
  **打开工程**那个下拉框中的搜索框的右侧」）：顶栏原来那两组（`📦 包 导出 导入` / `🧩 ComfyUI
  导出 导入`）删了，改到 `.pyg-projsel-pop > .pyg-projsel-tools` —— 那条行现在 = **🔍 搜索框 + 四颗按钮**
  （搜索框 `flex:1` 吃剩下的宽度，按钮在它右边；分组名降成 📦/🧩 图标 + `title`，面板才 340px 宽）。
  ⚠ **面板里那条搜索栏中间被删过一次**（当天早些的"删除这个组件"），用户随即报"搜索按钮不见了"
  ⇒ 已**恢复**：`projQ` + `projHits`，关面板即清空。**两条命中**（用户口径 2026-10-17 晚：
  「除了搜索本项目资产外，再加上能搜索其他项目的**项目路径**的功能，**不包括其他项目的资产**」）：
  ① **本项目的资产**（图 / 结构体 / 类 / 工具，`assetRowsAll` 里**只留 `project === curProj`**）；
  ② **别的工程的路径**（工程名或它的 root 命中 → 一行「📂 工程：名字（路径）」，点 = `pickProjectFromMenu`
  只切内容浏览器、不开图；当前工程不列）。资产排在前、工程在后。别再把它删了。
  ⚠ 判据里别用 `.pyg-ab` 找按钮：顶栏的 **A/B 那一组**也是 `.pyg-ab`（用 `.pyg-projsel-tools`）。
  🔀 **按节点搜图开关**（2026-10-19 用户口径：「搜索框添加一个开关组件，组件高亮就表示打开，打开后
  可以按照图内使用的节点搜索到对应的资产图」）：那颗 `.pyg-projsel-switch` 贴在搜索框**里面右侧**
  （输入框 `padding-right` 让位），**亮 = 开**（`.is-on`）；开的时候 `projHits` 除了名字/地址，还拿
  搜索词去比**图里用到的节点类名** —— 数据是服务端 `/api/pygraphs` 每条图新带的 **`types`**
  （`nodes[*].class_type` 去重排序；整张图本来就 parse 过，不多一次 I/O），命中行后面追加
  「· 含 KSampler」说明为什么被搜出来。⚠ 与按名字/地址搜是**并集**（不是替换），开关只对**图**有意义
  （结构体 / 类 / 工具没有节点表）；状态只在面板内存里、不落 `localStorage`。
  自证 `web_probe search`（31/31：顶栏/浏览器行没有、下拉里有搜索框 + 四颗按钮、按钮在搜索框右侧、
  打字出「🔍 搜索结果」、清空回到工程列表、**别的工程的资产搜不到**、**别的工程的路径搜得到且点了会切过去**、
  **开关默认关 → `KSampler` 搜不到 → 点亮后按节点搜到 `demo_comfy_txt2img` 且写「· 含 KSampler」→ 再点关掉恢复**）。
  ⚠ **搜索框被 `.pyg-bar .pyg-input` 钉成 150px**（那条是顶栏输入框的老规则：`max-width:150px; flex:0 1 150px`，
  优先级比 `.pyg-projsel-q` 高）⇒ 框只有 150 宽、右边一大块空白，📦/🧩 那组被推到面板最右
  （2026-10-19 用户报「对齐搜索框右侧」）。修法：`.pyg-bar .pyg-projsel-q, .pyg-projsel-tools .pyg-projsel-q
  { flex:1 1 auto; width:auto; max-width:none }`。实测（视口 1010）修前 input 150 / wrap 468（差 318px 空档），
  修后 input 满 468、按钮紧贴框右侧（4px gap）。
- **3D 视图面板那一行（2026-10-13 用户："合并这三行"）**：原来三行 = 提示（`模型来自 Input(model)`…）+
  `来源` 下拉 + ↻ + `MeshViewer` 自己的头部（名字 / 尺寸·顶点·面数 / 手感提示）。现在**并成一行**：
  **提示降成悬停**（chrome 上 `🧊 3D 视图` 那个 `title`）、**来源下拉与 ↻ 塞进 `.mshv-head`**
  （`MeshViewer` 新增 `headSlot`，只有停靠那处传）—— 标题 / 信息 / 来源 / ↻ / 手感提示同一行。
  空态（没有可看的模型 / 取几何失败）仍走 `Viewer3DPanel` 自己那行 `.pyg-log-head` 的 `bar`。
- **模型一律双面**（`MeshViewer.forceDoubleSide()`，2026-10-13 用户要求）：薄壳 / 单面网格（竹席那种平面、
  莫比乌斯带）从背面看是全空，看起来像"模型没加载"；glTF/FBX **自带**的材质（导出成 `doubleSided: false`）
  也一起改。⚠ three 是 effect 里 `await import("three")` 动态加载的 → 任何 new 出来的 helper
  **必须把 THREE 当参数传进去**（在模块作用域直接引用会 `THREE is not defined`，被加载流程的 catch
  吞成"载入失败"，实测踩过）。探针判据是根节点上的 `data-side`（`2` = DoubleSide，读**实际材质**的值）。

- **HEAD 必须显式声明**（`@app.api_route(..., methods=["GET","HEAD"])`）：FastAPI 的 `@app.get` 不带 HEAD，
  前端探 `Content-Length`（显示文件大小）会吃 **405**。
- **多文件 glTF**：一次选多个文件 → 第一个带 `?newdir=1`，服务端建一层 `<uuid8>/` 并回
  `name=<uuid8>/<主文件>`；伴随件（`.bin`/贴图/`.mtl`）带 `?dir=<uuid8>` 落到同一处。**子目录名只能由
  服务端生成**（读口按 `[0-9a-f]{8}` 校验），客户端只能回传。
- **边界**：扩展名白名单 `.glb/.gltf/.obj/.mtl/.bin/.stl/.ply/.fbx` + 贴图 `.png/.jpg/.jpeg/.webp`
  （贴图是多文件 glTF 的伴随件）、单文件上限 **200MB**、名字里带 `/` `\` 或 `..` 一律 400 ——
  服务端**绝不拼客户端给的路径**。
- **门槛**：执行面（`L_MINDMAP_ALLOW_RUN`）关着就不给上传 —— 往磁盘写任意字节本属执行面能力。

**本机路径预览（受控只读读口，`L_MINDMAP_MODEL_ROOTS`）**：四道闸 —— ① 白名单根（`;` 分隔的目录清单，
**未配 = 关着**，回 403 并点名这个变量）；② 只认 3D 扩展名（`.glb/.gltf/.obj/.stl/.ply/.fbx`）；
③ `resolve()` 后必须真在某个根下（这一条同时挡住 `..\..\`、绝对路径、盘符、UNC）；④ 必须是普通文件。
编辑器侧：`PyModelPicker` 对白名单根外的路径**不再静默置灰**，悬浮会说"不在白名单根下（当前白名单：…）"。
四道闸的逐条自证：`wuwor l_mindmap_mmd -- python <包>/999.0/tools/model_gates_check.py`。

**值口径**：`Input` 的值**标量存真标量、容器存 JSON 文本**，执行时由引擎 `coerce_param()` 转真对象。
为什么容器不能存数组：`Input.value` 端口是 `*`，而"连线"的存储约定正是 `[源id, 下标]` 数组 ——
存 `[1, 2]` 会被当成"连到节点 1 的第 2 个输出"，**值直接丢**且画布看着像有连线。
`type=image` / `type=model` 同理只存文件名。

## 多引擎（远端宿主）

编辑器可以**接多个后端引擎**：`local`（本进程，全功能）永远在，其余是登记的**远端宿主**
（别的进程里跑 `l_node_engine`，比如别的工具/Qt 应用）。契约与 bottle 宿主骨架见
`Rez-Docs/Rez_pkg/远端引擎契约_v1.md`。

- 配置：数据目录的 `engines.json`（`{engines:[{id,label,url,token,enabled}]}`）或 env `L_MINDMAP_ENGINES`（JSON 数组覆盖）。
- 接口：`GET /api/engines[?probe=1]`、`PUT /api/engines`、`GET /api/engines/<id>/instances`、
  `GET /api/engines/nodes-index`（**哪个引擎有哪些节点类**，60s 缓存；给"节点标引擎名"用）；
  `/api/pygraphs/object_info`、`/ports`、`/validate`、`/classes`、`POST /api/pygraphs/<名>/run` 都吃 `?engine=<id>`
  （**不带 = local**，老行为不变）。
- 口径：**图永远在编辑器这边**（换引擎不改图，引擎名只落 URL 与 localStorage）；
  远端跑图结果被**摊成与本地一模一样的 SSE 事件**（`started/executed/result/error`），前端零改动；
  宿主没在 `features` 里声明的能力一律降级（无权威校验 / 无缩略图 / 无 Ask / 无增量 / 无动态端口）。
- **一个图 = 一个引擎**（默认是这样）：图里混进"只有别的引擎才有"的东西时，要**标出引擎名**——
  ① **类型级**：那个 `class_type` 当前引擎没有、别的引擎有 → 卡片上 `🔗 muse`，左栏搜索单列一节
  「🔀 别的引擎才有的节点」（靠 `GET /api/engines/nodes-index`，60s 缓存）；
  ② **目标级**：`CallFunction.func_path` / `CallMethod.obj_path` 指向的函数/类**本地解析不了**、
  但别的引擎能解析（服务端在 400 文案里带上"可用引擎：…"，前端抓出来标同一个徽标）
  —— 解决"本地跑就报 `No module named 你们的包`，但不知道要切哪个引擎"。

### 混引擎：一张图同时用多个引擎

节点上带 `"engine": "<引擎id>"` 就是"**这个节点交给那台引擎跑**"（哪里来：属性面板的「引擎」下拉；
空/缺省 = 跟随工具栏的**默认引擎**）。跑图时的口径（`server.py::_mixed_plan` / `_mixed_run_stream`）：

- **按引擎切片**：段 = **同一引擎内用"同引擎边"连起来的极大连通块**（不是"每种引擎一整段"——
  否则 `local → muse → local` 会被误判成"跨引擎成环"，其实节点图是 DAG）；
- **段内不重造执行器**：整段交给那台引擎自己的 `execute_graph` 跑 —— 循环区 / if 分支 / bypass /
  缓存 / 护栏 / 人在环**全都不变**（这是"混用"能落地的前提）；
- **段间值交接**：跨引擎的边不能连线传（**对象不搬家**），改成"上一段跑完 → 取源节点输出 →
  **JSON 化** → 填成下一段目标输入的字面量"。JSON 不了就退成 `repr` 字符串；
- **`interface` 每段都带**（2026-10-18 修）：切段之后每段都是"另一张小图"，不把这一轮的**图入参值**
  传进去，段里任何 `GetParam` 都只看到空接口（症状：`GetParamError: 这张图没有声明入参 x`，
  或者编排层报"上游节点 xxx 没有产出"）。现在 `_run_one_segment(..., interface=…)` 本地段
  `execute_graph(interface=…)`、远端段随 `/engine/run` 正文一起发（契约 §2.1，可选键）；
  ⚠ 顺带一个画图习惯：一串只做取值的 `GetParam` 彼此**没有数据边**时会被切成 N 段（每段一个节点），
  用 `after` 把它们串起来就合成一段（`muse_upload_plans` 就是这么串的）；
- **`EngineStart` / `EngineEnd` 括号在跑图时也认**（2026-10-18 用户口径：「muse 的调用节点应该在引擎
  起始节点之间才对」）：`server.py::_engine_region_stamps` 在 `_mixed_plan` **之前**按括号给成员盖章
  （判据与 `web/src/pySchema.js::bracketRegions` 同一套：`start` 正向可达 ∩ 能到 `end`），
  括号盖的**赢过**逐节点 `nodes[id].engine` 缓存。所以图里只要画括号、**不必**给每个节点手填引擎；
  「刚加完括号没保存」的那次运行也是对的。画法上给区内首个节点接一条 `start.out → xxx.after`
  （或任意数据口）就够了 —— 区是靠**可达性**算的，`EngineStart` 本身是纯标记、不碰数据。
- **限制**（明确不支持，报错不猜）：跨引擎成环；循环区跨引擎（LoopStart/LoopEnd 在不同段）；
  段间对象身份/大对象（只能 JSON，二进制建议落盘传路径）；增量烹饪只在段内有效；
- 进度：`started` 带 `mixed:true` + 每段节点数，每段开始推一条 `executing{segment:true,engine,nodes}`，
  收尾仍是 `result{outputs,node_errors,order,mixed:true}` —— 前端零改动（多一句"▶ 段 local：N 个节点"）。
- 自证：`tools/engine_contract_check.py`（本地 vs 假宿主对拍）、`tools/engine_fake_host.py`（可跑的参照宿主）。

## 路径口径：**别写根绝对 `/api/...`**（踩过两次，2026-10-11）

页面可能挂在反代前缀下（`/mindmap-mmd/`、dev 的 `/mindmap-mmd-dev/`），所以请求前端一律走
**「`__API_BASE` 非空就拼前缀，否则用相对路径 `./api/...`」**这一套 —— 现有三处实现，改的时候必须一致：
`pyApi.js`、`flowApi.js`、`valueSources.js`（参数候选/值列表，`desc.api` 也按这套解析）。

- 写 `fetch("/api/xxx")` = **根绝对**：经前缀进来会逃出前缀、被别的 location 接走 → 404（dev 与生产都 404，
  控制台只留下一句 `Failed to load resource`，很容易被当成后端没这个接口 —— 其实直连 `:8110` 是 200）。
- **静态资源同理**：模板/页面里的 `static/...` 靠 `<base href>` 解析（`flow.html` 由服务端静态写入，
  dev 壳 `web/index.html` 用内联脚本归一化到应用根）。

**标签页图标**：唯一来源 `static/favicon.svg`（🧠），生产页（`templates/flow.html`）与 dev 壳（`web/index.html`）
**各引一行同一个文件**。dev 下这条相对路径由 Vite 的 `/static` 代理转给后端出图（见 `vite.config.js`）——
别把图标再复制一份到 `web/public/`（两份早晚不一样）。dev 壳少了那行 `<link rel="icon">` 时，
浏览器会去猜根路径的 `/favicon.ico`（那是主页的图标），表现就是"标签页没图标"。

## 核心功能

- **三种线型**：曲线（S 型双弯贝塞尔）/ 直角（smoothstep，圆角可调）/ 直线；`S` 键循环切换。
- **S 曲线细节**：
  - `curvature`（曲线弧度）控制弯幅，0 = 直线；
  - `handleWeight`（**直出直入手柄权重**）控制首/末控制柄沿各自 handle 方向伸出长度，
    让曲线先沿 handle「拉直」再转弯；两端方向**跟随 handle**（source 右→朝右、target 左→朝左），
    回边也不反向。
  - `handleScale`（**柄长系数**）：柄长 = `handleWeight` × 系数（0.2–2，默认 1.0；
    直角模式的「直出段长度」同样乘这个系数，所以三种线型（除直线）都能看到效果）。
- **回边**（target 在 source 左侧，或 gate 的 `on_trigger`）：**虚线 + 流动箭头**（可调速度/拖尾/箭头大小/虚线间隙/线宽透明度）。
- **布局模式**：「整理布局」下拉共 5 种：`dual`(双向) / `right`(向右) / `down`(纵向) / `radial`(放射) / `flow`(流程)。
  - **`flow` 是默认**（面向控制流图）：主链按「到 end 最短路径」水平排，上方分支横向展开、
    下方分支纵向展开，end 收敛到最右——与手工排好的 `default_intelligent_agent.json` 结构一致（唯一可见差异是
    `goal_nudge` 这类历史微调节点的 ±20px）。
  - 布局参数 `layoutCol / layoutRow / layoutX0 / layoutY0` 可在设置面板调。
  - 其余 4 种本质是树形脑图布局，有回边的控制流图用它们结构会乱，只适合当不同视角看。
- **连线智能避让**：默认路径穿过中间节点矩形时，自动 Catmull-Rom 平滑绕行（三种线型 + 回边都生效）。
- **下游跟随**：工具栏「⇁ 下游跟随」开关，开启后拖动上游节点，下游（沿正向边可达、跳过回边）实时跟随；
  状态存 `flow._ui.follow_move`，随文件保存。默认关。
- **框选（2026-10-13）**：左键从**空白处**拖 = 框选；判据是 **`Partial`** —— 框角**压到节点一点就算选中**，
  不要求整个框住（React Flow 的默认是 `Full`，大节点得把视图缩到很小才框得住）。**两套画布同一份口径**：
  本页 `/flow` 与 `/pygraph` 的 DOM 后端都传 `selectionMode="partial"`
  （`FlowEditor.jsx` 的 `<ReactFlow>` / `graphView.jsx` 的 `GraphSurface`），
  `/pygraph` 的 canvas 后端自己算**矩形相交**（`CanvasSurface.jsx` 的 marquee）—— **改一边必须改另一边**。
  自证：`web_probe.cjs select`（`MM_FLOW_NAME=<图名>` 时连本页 `/flow` 一起验）。
- **侧边设置面板**：`/flow` 那边由工具栏「⚙ 设置」打开主页**右侧占位列**（不悬浮、不遮挡画布/顶栏/节点检查栏）；
  `/pygraph` 这边**它就是一张普通标签**（`settings`，住右栏，能拖走/浮出去），
  ⚠ **2026-10-17 起顶栏不再有「⚙ 设置」按钮**（用户口径：「设置顶部的设置按钮，有设置面板就行了」）——
  要找设置就切到那一格；`SettingsPanel` 与流程图**同一个组件、同一份 `settings.yaml`**。
  拖动参数**画布实时预览**（边组件用 `useSyncExternalStore` 订阅）。
  - **页签（2026-10-17 用户口径「设置面板内部改为多标签页，因为设置项会不断增加」）**：
    一项设置只认两件事 —— `ITEMS[].group`（归哪页）+ `TAB_META`（页签标题）+ `TAB_ORDER`（顺序）。
    所以**加设置不用碰面板结构**：往 `ITEMS` 加一行（带 group）就有；**新开一个 group 自动多一个页签**
    （没在 `TAB_META` 登记的 group → 页签标题就用 group 名，排末尾）。现在 5 页：
    `🖼 画布`（网格/FPS/缩放补偿/流程布局）、`🪢 线型与箭头`、`🎯 手柄与曲线`、`🚧 碰撞与避让`、
    `🐍 Python 节点`。pygraph 传 `PYGRAPH_SETTING_KEYS` 过滤 → 只剩有项的那三页（空页签不出现）。
    **同屏只挂当前页的卡片**，页签上带项数徽标；当前页记 `localStorage["l_mindmap_mmd.settings.tab"]`
    （纯本机偏好，不进 `settings.yaml`）。⚠ 以前的**组头折叠**（`settings-group-head` + `GROUP_STATE_KEY`）
    被页签取代了 —— 别再加回来。
    自证：`web_probe.cjs settings`（15 项：页签数/标题/只画当前页/切页/切页不丢值/**探针把改动还原**/刷新记得
    + 下面那条「布局文件路径 + 👁 查看」四条）。
  - **界面布局文件：路径 + 👁 查看**（2026-10-17 用户口径「设置面板中我希望能看到界面布局文件的路径，
    并且添加一个查看按钮」）：面板顶栏下面常驻一行 `界面布局文件 | <等宽路径> | N 套 · 大小 | 👁 查看`。
    路径**由服务端给**（`GET /api/pygraphs/ui-layout` 多回 `path` / `saved` / `bytes` / `mtime`，
    见 `_ui_file_info()`）—— 前端不拼：数据目录按用户/实例变（`~/.lugwit/<实例>/l_mindmap_mmd/pygraphs/_ui/<用户>/__layout__.json`），
    拼必错。**没存过也显示路径**（告诉他"保存布局"会写到哪），后面挂「未存过」。
    「👁 查看」开**只读浮层**（壳子复用 `.pyg-src-*`，就是「📄 查看资产源码」那套）：标题 + 路径一行 +
    格式化的存档 JSON（**多套**：`{version:3, layouts:{名字: 树}}`）；点遮罩 / ✕ / Esc 关。
    这一行说的是**服务端那份**（权威、按用户）；本机**不存任何草稿**（2026-10-18「请不要使用草稿布局」）。
  - 卡片是三列：**标签+滑杆 | 效果预览 | 数值胶囊**。**效果预览列固定占卡片 1/5 宽**（2026-10-13 用户口径）：
    `.settings-mini-wrap { flex: 0 0 20% }`，里面的 `<svg class="settings-mini">` 按 `viewBox` **等比缩放**
    （`width:auto; height:auto`）。以前 svg 是**定宽 150px**（`SettingsPage.jsx` 的 `VW`）—— 在窄栏里吃掉
    四成宽，把标签挤成竖排（"背景网格"三个字占三行）。**数值胶囊是预览的兄弟节点**（在
    `.settings-mini-wrap` **外面**）：它有自己的自然宽度，不参与那 1/5 的分配 —— 挪进去就会把 svg 压扁。
- **智能体护栏**：start/end 节点显示由 `_ui.show_start_end` 控制（默认隐藏，仅流程语义）。
- **节点/图参数表单（2026-10-05 补）**：门节点（`gate_*`）的属性面板多一张「护栏参数」表，图属性面板多一张
  「图级参数」表 —— 写的就是节点 `params` 与图 `params`（拦几次 / 阈值 / 提示词、熔断与节奏等），
  存进 JSON 后运行端 `flow_engine` 直接读（改这些**改图即可**，不必动代码；详见
  `流程图智能体_图驱动控制流.md` §3.0）。表单键表在 `web/src/flowSchema.js` 的 `GUARD_PARAMS` / `FLOW_PARAMS`。
- **勾选引擎 = 可用集（2026-10-13）**：左栏「🔌 引擎」那张表里每行一个**勾选框**（`local` 恒勾、不可取消）。
  - ⚠ **没有"默认引擎"这个概念**（同日用户把整套删了，口径："只有自我引擎 + 远程链接第三方引擎，
    **默认引擎一定是 local**"）：**没标注 `engine` 的节点一律跑在本进程**；要交给远端就在节点上标
    `engine:<id>`（图文件 `nodes[id].engine`）。`?engine=` 服务端还认（给脚本/混引擎判定用），
    **界面没有入口**（工具栏那个下拉删了两轮；顶栏现在那个下拉是**选工程**，见下条）。
  - **两个东西拆开**：**节点表 = 看得见**（`loadInfo` 拉**登记表里所有引擎**的 `object_info` 并集；
    多台都有同一个类时 **local 那份赢**、来源记在 `info._engine`；**渐进合并** —— local 那份先上表，
    其余各拉各的，一台离线不该让整张表等它（远端返回空时**不再 set 一次**：表没变就不该重建节点视图，
    否则会白丢一次画布选中）。
    **为什么必须并集**：以前没勾的引擎不进表 → 那个类的**端口全没了** → React Flow 把缺 handle 的边
    **直接丢掉**（用户口径："节点端口要保持连接"）。并集之后连线与端口照旧，只是标 `✕ <引擎>`；
    「（这台没勾）」也进**告警**（不影响跑：那一段仍交给它）。canvas 后端同一条口径（红框 + 标题后缀 `✕ <引擎>`）。
    **可用集 = 能放/算不算可用**：节点库只列勾了的引擎的类。
  - 勾上某台 = 把它的**类/函数并进节点库**，从它那儿落下的节点**自动标 `engine`**（local 也有的不标）。
    勾选状态**只存 `localStorage["l_mindmap_mmd.engines"]`**（本机偏好）。
    ⚠ **不再进地址栏**（2026-10-16 用户口径：「把 url 参数会添加引擎字段的逻辑**完全删除**」）：
    `engines.js` 的 `selectedEngines()` 不再读 `?engines=`、`rememberEngines()` 不再 `replaceState` 写地址
    （以前是"URL 优先 + 每次勾选改写地址"）；切图时 `pushGraphRoute` 还会**把残留的 `engines=` 从地址删掉**
    （老书签里那个参数已经没人读了）。探针 `engine` 场景那条判据也跟着**反过来**：
    `勾选不写地址（没有 ?engines=）`。
  - 「＋ 添加」（加远端引擎）是**小弹窗**（id / 地址 / token 三个字段，Esc 或点遮罩关）——
    面板上**不常驻**那三个空输入框（低频操作，别占版面）。
  - **状态**（2026-10-16 补）：**打开这个标签就探活一次**，之后**每 10s 刷**（标签不活动时面板体不挂载 →
    定时器自动停；页面切后台也跳过）。以前状态**只在手点「⟳ 探活」那一下**更新（首屏故意不探活：
    每个远端 3s 超时会把首屏拖慢）。
  - **「▶ 启动」**（2026-10-16 补）：按端口找到**本机那张服务卡片**，走主页控制面
    `POST /api/v1/services/<卡>/start?trigger=cli`（与 `wuwo svc start` 同一套；口径抄
    `l_model_hub.local_models`）。⛔ **只对本机地址有效**（`127.0.0.1`/`localhost`）—— 别的机器上的引擎
    按钮置灰并写明"去那台启动"（那台的控制面我们连不上）；**不是本机卡片**的端口 → 400 + 人话。
    服务端 `POST /api/engines/start`（`?dry=1` 只解析卡片、不启动，给自证与"先看能不能启动"用）。
    ⚠ 两个坑：① `GET /api/v1/services/hosted` **必须带 `status=1`** —— 不带时只回极简索引
    （没有 `services` 行、**没有 `port`**）→ 一张卡都匹配不到（实测：报"读到 0 张卡"）；
    ② **本文件没有 `lprint`**（那是引擎/l_agent 的日志口）—— 写成 `lprint(...)` 会 NameError → 接口直接
    **500**（本次实测撞上；顺手把文件里另外 4 处同类隐患也改成了 `print(f"[mindmap] …")`）。
    自证：探针 `engine` 场景（本机行「▶ 启动」可用 + 点了日志里出现可读结论）。
  - **存盘 / 跑图前按归属引擎补 `engine`**（`PyGraphEditor.jsx` 的 `stampEngines` / `stampPrompt`）：
    哪个类"哪台有它"就标到节点上（**local 也有的不标**）。为什么必须补：`engine` 只跟节点自己走，
    而图上常有"没标的节点"——补一层之后"没标 = 跑 local"这条口径才与图上标注一致。
  - ⚠ **服务端顺序**：run 路由里**混引擎判定优先于"整张发给远端"**（`server.py` 的 `run_pygraph_api`；
    反着的话那条整张转发会**吞掉**所有跨引擎标注）。
  - 端口按**节点自己的引擎**问（`engineArg(node.engine || local)`）—— 标了 `engine` 的节点在别台进程里跑，
    它的函数/方法/资产只有那台能解析。
  - **跨引擎执行本来就有**（服务端 `server.py` 的 `_mixed_plan` / `_mixed_run_stream`：按 `nodes[id].engine`
    切段、跨引擎的边摘掉改成**值注入（JSON 交接）**、段间拓扑序、段间成环报错）；这里补的是**入口**。
  - ⚠ **JSON 只出现在"跨引擎"的边上，别以为段内也过 JSON**：段 = **同一引擎内用同引擎边连起来的极大
    连通块**（`server.py` 的并查集只沿"两端同引擎"的边合并），段内的边原样塞进那一段的 prompt，
    由那台引擎在自己进程里用**真对象**解 —— 对象不搬家。只有 `ra != rb`（两端不同引擎）的边才记
    `injections`、才 `_handoff()` JSON 化。HTTP 请求体是 JSON 没错，但那只是"图的形状"，
    **段内的值从来没变成过 JSON**。（全图一个跨引擎标注都没有时 `seg_ids <= 1` 直接走老路：整张发给那台，
    更是一段。）
  - ⚠ 别名：**"别的引擎才有的类"不再当结构错误**（`PyGraphEditor.jsx` 的 `validate()` 把它降成提示）——
    从前它会进 `errors`，把 ▶ 运行**拦死**（实测：混引擎图点运行一个 `/run` 请求都不发）。
- **值交接瘦身（2026-10-20，用户口径「要高性能」→「先治值交接」）**：几何（`{vertices, faces}` /
  `(顶点, 面)`）**不再随跑图结果走一遍**。实测同一张 demo 图（7.9k 顶点）：
  SSE `result` **2,292,268 B / 0.58s** → 瘦身后 **53 KB / 0.30s**；20 万顶点的图是几十 MB 的差距。
  - 判据与阈：`server._slim_outputs()`（`_SLIM_MIN_VERTS = 512`）—— 超过这个顶点数的几何换成
    `{kind, format, __mesh_ref__: 节点id, vertices: N, faces: M, note}`；**小网格原样发**（几百顶点几 KB，
    点开还能看见数字）。三元组里的元组 / `(顶点, 面)` 二元组 / `{"result": …}` 都会逐层换。
  - 用在哪：SSE `result`（本地 / 混引擎 / 类方法三条路）＋ `_values_payload`（`values.json` 也跟着瘦）。
    **只换送出去的那一份**：`_cache_renders` / `_cache_materials` / `_cache_meshes` 用的仍是原始 `outputs`
    （它们在这之前跑），所以 3D 视图 / 缩略图 / 材质**一点不受影响**。
  - 真几何怎么取：`GET /api/pygraphs/mesh?name=<图>&node=<节点>`（一直都有，`_MESH_SRC` 那份引用 +
    磁盘 `_renders/<图>/m*.mesh.json` 兜重启）。面板「本轮值」里那行摘要变成
    `mesh（7938 顶点 · 14400 面）· 几何按节点取（3D 视图见）`（`RunValue.valueSummary`）。
  - 顺带修的两处：`_cache_meshes` 现在**先数顶点**再决定要不要转 payload —— 超过
    `_MESH_DISK_MAX_VERTS`（60000）的本来就不落盘，以前却先白转一遍（几十万顶点要几百毫秒）；
    `_persist_mesh_node` 改成**先判再写**（以前先写 `m<key>.mesh.json` 再判节点数，超了直接 return，
    盘上留一堆没人引用的孤儿文件）。
  - ⚠ **落盘节点数 8 → 24**（2026-10-20 截图时踩到）：拼装链（茶壶：壶身→+盖→+钮→+嘴→+把）光中间件
    就 14 个，卡在 8 会把**最后那个整件**挤出去 ⇒ 重启后「🧊 3D 视图」跟着 `3d_view` 去看时，
    `Viewer3DPanel` 发现那个节点不在清单里就把来源回退成"没填"，面板一片空（看着像"几何丢了"）。
    现在另加**顶点总量预算** `_MESH_DISK_MAX_TOTAL_VERTS = 240000`（≈ 十几 MB JSON），节点放宽也不撑爆盘。
  - 自证：`_slim_outputs` 六种形状（大 dict / 小 dict / 二元组 / 已是摘要 / `{result, ui}` / 字符串与对象）
    + 真服务跑 demo 图（`POST …/demo_mesh3d/run` → 53 KB、`/api/pygraphs/meshes` 仍列 9 个几何节点、
    `/api/pygraphs/mesh?node=7` 仍回 670 KB 全量几何、`models/*.obj` 照旧生成）。
- **属性面板「本轮值」一览（2026-10-13）**：选中节点 → 右侧「🔧 属性」里那张表列出它**所有**入参/出参
  在本轮的值；**没跑过 = 全是 `?`**（面板会写"没跑过 → 全是 ?"）。短标量（数字/布尔/≤60 字无换行的串）
  **直接给原文**，容器与长串走可折叠的 `RunValue`（一行摘要 + ▸ 展开 + ⧉ 复制）。判据"这轮跑过没"=
  有输出 / 有耗时 / **或报过错**（失败也是跑过：字面量照样显示，只有出参是 `?` —— 那正是"它没产出"的信号）。
  节点**说明**（`DESCRIPTION`）不常驻：收在标题行那个 **❗ 图标**里（悬停给全文、点一下在面板里展开/收起，
  换节点自动收起）—— 几百字的说明会把面板挤掉半屏。⚠ 那个"换节点收起"的 effect **必须排在 `selected`
  声明之后**（依赖数组在渲染期求值，写在前面踩 TDZ → 整页白屏，实测踩过）。
- **运行日志收集报错原文（2026-10-13）**：一轮收尾时，除了 `✗ 完成（有 N 个节点报错）：4, 5, 6` 这行，
  再逐个报错节点打 `✗ 节点 <id>（<异常类型>）：<message 首行>` + 一行 `↳ <堆栈末行>`（SSE 的
  `result.node_errors`）。以前只给 id 列表，排错要先挨个点节点看右侧。
- **保存前校验会拦下引擎才会炸的坑**：谓词不在表里、`branch` 缺 `always` 兜底、guard 名没注册、
  指向装饰节点；`PUT /api/flows/{name}` 在能 import `l_agent_chat` 的环境里还会跑权威 `parse_flow`。

## 设置持久化（settings.yaml）

- **唯一设置源**：`~/.lugwit/l_mindmap_mmd/settings.yaml`（环境变量 `L_MINDMAP_MMD_SETTINGS` 可覆盖目录）。
- **前端不内置默认值**：默认值只在随包模板 `src/l_mindmap_mmd/settings.yaml`，首次访问自动复制到用户目录；
  服务器「模板补缺」会给旧文件自动补上新增键。
- **改动即时自动保存**：设置面板每改一项即 PUT 写 YAML；「恢复默认」= 后端用模板覆盖用户文件（`GET /api/settings?reset=1`）。
- 设置键：`handleWeight, handleScale, handleBias, handleAim, handleAimRef, showHandles`（手柄与曲线）、
  `avoidPad, softMargin, softRamp, softStep, showSamples, showAvoidBounds`（碰撞与避让）、
  `borderRadius, strokeWidth, backWidth, backOpacity, arrowDur, trailCount, arrowSize, dashGap,
  labelFontSize, showLabels`（线型与箭头）、`showGrid, showFps, zoomCompEdge, zoomCompNode, layoutCol, layoutRow, layoutX0,
  layoutY0`（画布与布局）、`allowAllPython`（Python 节点）。面板按这五组折叠（折叠态存浏览器 localStorage），数值胶囊点一下可手输数字
  （单位按参数自动补、越界自动夹紧）。
- **缩放补偿**（`zoomCompEdge` 连线 / `zoomCompNode` 节点特效，0..2，**默认 1**；2026-10-15 用户口径
  「视图缩到 0.25，线宽乘回 4，别缩得那么快」+「正在运行节点的特效**也**跟着这样」，并要求**两个旋钮分开调**）：
  屏幕尺寸 = `基础尺寸 × zoom^(1-补偿)` —— 1 = 缩到 25% 时尺寸仍和 100% 一样大（连线基础线宽 1.3px
  在 0.257 视图下实测屏幕仍是 1.30px），0 = 老样子（跟着视图缩），>1 = 缩远了反而更粗。
  · **连线** = 线宽（`.react-flow__edge-path`）＋ 热边光晕半径 ＋ 流动虚线那批；
  · **节点特效** = 运行中的呼吸光晕半径 ＋ 绕边框流光那条环的宽度（DOM `.pyg-node.is-running`、
    canvas 那圈脉冲）。
  落点：DOM 后端走画布容器上的两个 CSS 变量 `--pyg-k` / `--pyg-n`（= `zoom^(-对应补偿)`；
  `PyGraphEditor.applyZoomComp` 由 React Flow 的 `onMove` **命令式**写，不进 React state —— 平移/缩放
  每帧 setState 会把大图拖垮；连线的**基础线宽**另走 `--pyg-w`），canvas 后端自己在屏幕坐标系里乘同一因子
  （`CanvasSurface`）。两个数只影响观感，不落图文件、不进图 prompt。
  实测：zoom 0.257 下「连线」调 0.5 → `--pyg-k` = 1.9722（= `0.257^-0.5`）而 `--pyg-n` 不动；
  「节点特效」调 1.5 → `--pyg-n` = 7.6709（= `0.257^-1.5`）、`--pyg-k` 不动。
- `allowAllPython`（**默认 true** = "默认运行全部"）：python 节点图的导入白名单整表放行 —— 宿主在
  跑图 / 问端口 / 取节点元数据前给本进程设 `L_NODE_ENGINE_ALLOW_PKGS=*`（通配语义见 `l_node_engine.md` §2.4），
  **不动**用户的 `allow.json`；关掉则还原进程启动时的原值。⚠ 打开后任意 python 模块都能被导入/实例化。
- 后端读写是**扁平键值 YAML 子集**（`server.py` 的 `_yaml_load/_yaml_dump`），无第三方依赖；API 面走 JSON。

## 流程图文件与 API

- 流程图目录：**实例数据根**下 `<LUGWIT_DATA_ROOT>/l_agent_chat/flows`（默认
  `~/.lugwit/main/l_agent_chat/flows`；解析与运行端 `flow_spec.flows_dir` 同源，
  走 `l_app_ready.paths.pkg_data_dir`，`L_AGENT_CHAT_FLOWS_HOME` 可覆盖）——所以编辑器里改的图，
  `l_agent_chat` 运行端直接用。
  > ⚠️ 2026-10-04 修复：此前这里硬编码 `~/.lugwit/l_agent_chat/flows`，而引擎读数据根目录，
  > 两边分叉 → 脑图改的结构进不到 agent（还静默回退硬编码）。现已改为同源解析。
- API（`http://127.0.0.1:8110`）：

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/api/flows` | 流列表 |
| GET | `/api/flows/{name}` | 取流程图 JSON |
| PUT | `/api/flows/{name}` | 保存流程图（编辑器自动保存也会 PUT）；**校验不过也照写**，原因在响应的 `warning` 字段 |
| DELETE | `/api/flows/{name}` | 删除 |
| GET | `/api/settings` | 读设置（`?reset=1` 恢复模板默认） |
| PUT | `/api/settings` | 写设置（增量合并，落 YAML） |

> ⚠️ 编辑器有 1s 静默自动保存：在浏览器里点「整理布局」/拖动节点会 PUT 覆盖 `default.json`。
> 测试后若需还原，用 git / 备份恢复，或 reload 使内存与文件一致（无 dirty 时不写）。

## 与 l_agent_chat 的联动

`l_agent_chat` 默认智能体有三档「图驱动」（`config.py` 的 `flow_enabled` / `flow_full` / `flow_name`，
**档位也可以是智能体属性** —— `agents/<名>.json` 的 `flow:{enabled,full,name}`，优先级高于全局）：

| 档 | 谁决定控制流 | 说明 |
|---|---|---|
| `flow_enabled=1`（`flow_full=0`） | 图只决定**无调用时的收尾路由** | goal 判定 + `stall/verify/fact/confirm` 四道护栏门的顺序与去向 |
| `flow_enabled=1` + `flow_full=1` | 图决定**每一步路由** | 外加 `branch_calls`（去 tools / 走收尾）、`branch_steps`（继续 plan / 收尾 / `end_abort`）与 `gate_idle`（只看不改的提醒门）；`plan`、`tools` 两个动作节点由 app 步循环**托管执行** |
| `flow_enabled=0` | 纯硬编码 | 完全不读图 |

图默认名 = `flow_name`（空则智能体名），当前 `default` 与 `default_intelligent_agent` **共用**
`default_intelligent_agent` 这张图（存在 `<数据根>/l_agent_chat/flows/`，用户目录优先 → bundled
`default_flow()` 兜底）；图缺失 / 结构不满足 / 引擎报错 → 逐级降级（全流程 → 收尾路由 → 硬编码）
并打 `⚠️` mark。改图走本编辑器（:8110）可视化编辑，即改即生效（2s 内热生效）；
运行端权威状态看 `GET :1250/api/flow/status`。

## AI 节点的两种卡片形态（2026-10-19）

`AIVideo` / `AIImage` / `AIText` / `AIVision` / `AISpeech` / `AITranscribe` 这批发散型的 AI 节点，
卡片可以在**两种形态**之间切（用户口径「切换成这种类型的卡片」）：

| 形态 | 长什么样 | 数据 |
|---|---|---|
| 普通卡（默认） | 行式参数：端口 + 名字/类型 + 控件（**始终显示**，与是否选中无关） | 无额外字段 |
| **表单卡** | 宽卡 560px：大提示词、反向提示词、一排参数胶囊（模型/比例/时长/分辨率/种子/音频/数量/画质/帧率）、`⌄ 更多参数`（首帧/尾帧/参考图/多镜头/私有字段/重试）、右下 `▶ 生成`（= 运行到本节点） | `nodes[id].ui = "form"` |

⚠ **不要做成"选中才显示参数"**（2026-10-19 用户口径「不应该选中才显示参数」）：两种形态都始终显示参数，
选中只影响 React Flow 自己的高亮。

- 切换入口：卡片标题栏那颗 **`▤` / `▦`**（只有这批 AI 节点有）；写进**图文件**（`pySchema` 两处白名单），换机器打开还是这个形态
- 端口：两种形态都画在**左侧**（表单卡与收起卡共用同一个 `.pyg-cports` 侧边层，输出在右）；端点名默认隐藏，
  **鼠标靠近卡片 50px 内就全部浮出来**（`.pyg-node::before` 撑出的透明光环 → `.pyg-node:hover .pyg-portname`），
  单个端点悬停也显示；标签贴在**卡片外侧**：输入口在左（`translate(-100%,-50%)`）、输出口在右
- ⚠ 光环的 50px 是**画布坐标**（`inset:-50px`，跟着视图一起缩放）：2026-10-19 先按"屏幕 50px"
  补偿过一版（`--pyg-near = 50/zoom`），低缩放下那圈色带大到糊住画布，用户否掉
  （口径「视图缩小后这个区域也应该缩小」）→ 现在缩放 0.33 / 0.69 / 1.19 实测边界 16 / 34 / 58 屏幕像素，
  正好 `50 × zoom`；别再往这里加 `/zoom` 补偿。
  ⚠ 光环**不能**给 `z-index:-1` —— 会被画布 pane 压下去，命中测试轮不到它（等于没光环）；
  代价是卡片四周 50px 也算"卡片"（在那儿按下等于拖节点、不是框选/平移）。
- 这个 50px 边界**可见**（2026-10-19 用户要求「这个边界能半透明可视化么」）：`.pyg-node:hover::before`
  给光环铺 `rgba(56,132,255,.02)`、**只填色不描边** —— 平时全透明（否则整张图画布上一圈圈色带），
  **指针真落进光环那一下**才现形，正好是"判定生效"的同一刻。
  ⚠ 别用 `opacity` + `transition` 做这个显隐：Chromium 里伪元素的 hover 过渡有时不重绘
  （实测 `getComputedStyle(el,'::before').opacity` 恒为 0，光环画不出来）→ 直接切 `background` / `box-shadow`。
- ⚠ 实现坑（踩过）：表单卡的每个控件必须走 **`PyField`**（不是直接 `PyWidgetInput`）—— `model` / `provider`
  这类参数的候选是**接口描述对象**（`values:{api,pick,value,where}`），只有 `PyField` 会用 `useCandidates`
  去取；直接给 `PyWidgetInput` 的话下拉里只剩当前值一个（2026-10-19 用户报「aivideo 节点的表单模式
  为啥无法选择厂商,只有一个」）。修完实测：厂商 11 项（空 + 10 家）、模型 34 项
- ⚠ 实现坑（踩过）：`data` 里原来的 `ui` 键是**内联预览**（`lastUi`）→ 卡片形态另开 `cardMode` 传；`compactCard` 必须在 `formCard` 之后算，否则两套同时渲染、卡片底下多出一大块空白

## 环境变量

| 变量 | 作用 |
|---|---|
| `L_MINDMAP_MMD_RUNTIME` | 覆盖本包的 runtime 目录（`_runtime_dir()`：热更/守卫的运行时目录，默认 `~/.lugwit/l_mindmap_mmd/runtime`） |
| `L_AGENT_CHAT_FLOWS_HOME` | 覆盖 flows 目录（**与 l_agent_chat 运行端同源**，默认 `<实例数据根>/l_agent_chat/flows`，即 `~/.lugwit/main/l_agent_chat/flows`；`L_AGENT_MARKET_HOME` 亦可） |
| `L_MINDMAP_MMD_SETTINGS` | 覆盖 settings 目录（默认 `~/.lugwit/l_mindmap_mmd`） |
| `L_MINDMAP_LOCAL_NO_AUTH` | `0/false/no/off` = 关掉"本机直连只读免登录"（默认开） |
| `L_MINDMAP_MODEL_ROOTS` | 本机路径预览的**白名单根**（`;` 分隔，如 `E:/assets;D:/models`）；**不配 = 该功能关着**（读口回 403） |
| `L_MINDMAP_ENGINES` | 远端引擎清单（JSON 数组，**覆盖** `engines.json`）；条目 `{id,label,url,token,enabled}` |
| `L_MINDMAP_ENGINES_FILE` | 覆盖 `engines.json` 的位置（自动化测试用） |
| `LUGWIT_AUTH_URL` | 登录闸门验签用的 auth 地址（默认 `http://127.0.0.1:8080`，走回环 nginx 入口） |
| `L_MINDMAP_HOST` / 端口 env | 监听地址 / 端口（默认 8110，`PORT_ENV` 亦可） |
