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
  **只绑本机**：`/api/pygraphs/{name}/run` 是任意 python 执行面，不许对外暴露）与
  `l_mindmap_mmd_server_exec`（同上但 `L_MINDMAP_ALLOW_RUN=1`，**打开执行面**；卡片当前指这条）。
- 带 `.dev_mod`（`L_DEV_MOD`）：改 `src/**` 会热更自重启；改 `web/src/**` 需 `npm run build`
  （产物进 `src/l_mindmap_mmd/static/dist`）后浏览器**硬刷新**——`main.js` 无版本哈希，易被缓存。
- 前端开发：`wuwo svc` 之外的 `l_mindmap_mmd_web_dev` 别名起 Vite :5175，`/api` 代理到 :8110。
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
- **权威结构校验（2026-10-12）**：`POST /api/pygraphs/{name}/validate`（body 可带 `{"prompt": …}`，
  可传**图文件形状**或引擎 prompt；不带就读盘上的图）→ `{ok, errors, warnings}`。
  实现是引擎的 `validate_structure`：**与执行器共用同一套计划器**（括号配对 / 嵌套限制 / 真环都问
  `_build_loop_plans` / `_build_if_plans` / `_topological_sort`），所以"校验过了 = 真能跑"，
  不会出现"校验说没事、跑起来 GraphPlanError"。编辑器「校验」按钮 = 本地快查（立刻出行内标记）
  + 这条权威校验（日志会分别报两边的条数）。脏图（非对象节点等）不会 500 —— 兜成一条可读 error。
- **轨迹里的分支决策**：`IfStart` 会发一条 `executed{node, if_taken}` —— 轨迹的 `steps` 里就有
  `{"node": "2", "if_taken": true}`（回放时步骤条上悬停能看到"条件为真/假"，运行日志里是
  `🔀 如果 2：条件为真 → 执行区内`）。
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

## pygraph 的 `models/` 读口与参数值口径

**两个路由**（`server.py`）：编辑器预览、以及"值到底存不存在"都看它们。值里存的是**文件名**，
光名字由引擎 `l_node_engine.model_path()` 补成 `models/` 下的绝对路径。

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | `/api/pygraphs/models/upload?name=<原始文件名>[&newdir=1][&dir=<子目录>]` | 上传模型；body = **原始二进制**（分块落盘，大文件不进内存）。落盘名由服务端清洗（`_safe_model_name`：只留字母数字/`._-`、限长、重名加 `-1` **不覆盖**）→ 回 `{name, dir, path, url, bytes}`，**写回参数值的就是 `name`**（相对名） |
| GET / HEAD | `/api/pygraphs/models/{name:path}` | 读 `models/` 里的文件：`<文件>` 或 **`<服务端子目录>/<文件>`**（多文件 glTF），其余形态（穿越 / 两层以上 / 反斜杠）一律 400 |
| GET / HEAD | `/api/pygraphs/models-local?path=<绝对路径>` | **受控只读读口**：预览白名单根下的本机模型（见下） |
| GET | `/api/pygraphs/models-local/roots` | 白名单根清单（编辑器用它给"不在白名单根下"的明确提示） |

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
- **侧边设置面板**：工具栏「⚙ 设置」打开主页**右侧占位列**（不悬浮、不遮挡画布/顶栏/节点检查栏），
  拖动参数**画布实时预览**（边组件用 `useSyncExternalStore` 订阅）。
- **智能体护栏**：start/end 节点显示由 `_ui.show_start_end` 控制（默认隐藏，仅流程语义）。
- **节点/图参数表单（2026-10-05 补）**：门节点（`gate_*`）的属性面板多一张「护栏参数」表，图属性面板多一张
  「图级参数」表 —— 写的就是节点 `params` 与图 `params`（拦几次 / 阈值 / 提示词、熔断与节奏等），
  存进 JSON 后运行端 `flow_engine` 直接读（改这些**改图即可**，不必动代码；详见
  `流程图智能体_图驱动控制流.md` §3.0）。表单键表在 `web/src/flowSchema.js` 的 `GUARD_PARAMS` / `FLOW_PARAMS`。
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
  labelFontSize, showLabels`（线型与箭头）、`showGrid, showFps, layoutCol, layoutRow, layoutX0,
  layoutY0`（画布与布局）、`allowAllPython`（Python 节点）。面板按这五组折叠（折叠态存浏览器 localStorage），数值胶囊点一下可手输数字
  （单位按参数自动补、越界自动夹紧）。
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
