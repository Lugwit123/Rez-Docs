# `l_node_engine`（节点 python 引擎）与它的编辑器 / agent 接线

> 更新：2026-10-11。面向"要用/要改这套东西的人"。**先读本文再动手** —— 「四、坑」那一节是踩出来的，不是推测。
> 本文以**引擎包 `l_node_engine`** 为主线（按 Rez-Docs 约定：包使用文档文件名与包名一致）；
> 编辑器属 `l_mindmap_mmd` 的 `/pygraph`，agent 接线属 `l_agent_tool` / `l_agent_chat`。

## 一、这是什么

三层结构，一层一件事：

| 层 | 包 / 文件 | 职责 |
|---|---|---|
**执行内核** | `rez-package-source/l_node_engine/999.0/` | 宿主无关的节点图引擎：节点基类与注册表、拓扑执行（含循环区、软失败）、任意对象/函数/类节点、护栏。**纯标准库** |
**编辑器** | `rez-package-source/l_mindmap_mmd/999.0/`（`/pygraph/<图名>`） | React Flow 画布 + `/api/pygraphs*`（CRUD / object_info / ports / run SSE）。图落在磁盘，跟 `l_agent_chat` 的流程图共用一个服务、**不同的目录与路由** |
**agent 工具** | `l_agent_tool/pygraph_tools.py` + `l_agent_chat/pygraph_tools.py` | 扫图目录 → 每张带 `Input`/`Output` 的图注册成 `pygraph_<图名>` 工具，在 agent 进程内执行 |

一句话数据流：**编辑器画图 → 存成 JSON → 引擎按拓扑执行 → `emit` 事件走 SSE 回编辑器（进度/结果）→ 同一张图也能被 agent 当工具调**。

## 二、怎么用

### 2.1 打开编辑器

```
http://127.0.0.1:8080/mindmap-mmd/pygraph/<图名>     # 经主页网关
http://127.0.0.1:8110/pygraph/<图名>                  # 直连服务
```
（`/mindmap-mmd/` 是主页反代；同一个服务上 `/flow/<名字>` 是 agent 流程图编辑器，`/pygraph/<名字>` 才是节点 python。）

### 2.2 起 / 重启服务

```cmd
wuwo svc status  l_mindmap_mmd_server
wuwo svc restart l_mindmap_mmd_server          # 最后手段，会中断在用的请求
```

⚠ **改了 `l_node_engine` 必须 `reload`**：mindmap 的源码热更只盯自己包目录，引擎在**包外** —— 光改引擎文件，跑的还是旧模块。

### 2.3 执行面开关（默认关闭）

跑图 = **执行任意 python**（RCE 面），所以是显式选择：

| 别名 | 执行面 | 绑定 |
|---|---|---|
`l_mindmap_mmd_server` | **关**（`POST /run` 返回 403） | `127.0.0.1` |
`l_mindmap_mmd_server_exec` | **开**（`L_MINDMAP_ALLOW_RUN=1`） | `127.0.0.1` |

卡片（`l_homepage` 的 `l_mindmap_mmd_server`）目前指向 `l_mindmap_mmd_server_exec`；想关就把 `run_args` 换回 `l_mindmap_mmd_server`（`wuwo svc` 用 `packages + run_args` 拼命令，`run_cmd` 只是展示）。

其它旋钮：`L_MINDMAP_RUN_TIMEOUT`（墙钟超时，默认 60s）、留痕 `~/.lugwit/main/l_mindmap_mmd/pygraphs/runs.jsonl`。
**别在公网部署机上开这个开关，也别把 `--host` 改回 `0.0.0.0`。**

### 2.4 导入白名单（"能 import 哪些包"）

默认**空** = 只放行 `l_node_engine` 自身 + 已注册节点所在模块。放行外部包三条路（优先级从高到低）：

1. 环境变量 `L_NODE_ENGINE_ALLOW_PKGS=json;mypkg`（**设了就以它为准**）
2. 文件 `~/.lugwit/main/l_mindmap_mmd/pygraphs/allow.json`：`{"packages": ["json", "mypkg"]}`（卡片只能带参数带不了 env，才有这一档；改完**立即生效**，`allowed_packages()` 每次调用都重读）
3. 卡片启动参数：`run_args: ["l_mindmap_mmd_server_exec", "--allow-import", "json;mypkg"]`

被拒绝时错误信息会**直接写出这两个办法**（含 `allow.json` 的绝对路径）—— 别把它当成"路径写错了"。

**通配 = 全部放行（2026-10-11）**：白名单里任一条目写成 `*`（或 `all`）即"允许任意 python 对象"
（`guard.ALLOW_ALL_TOKENS`；env / `allow.json` 都认）。`l_mindmap_mmd` 的编辑器把它做成了设置项
「允许所有 python 对象（导入白名单全放行）」——**默认开**（`settings.yaml` 的 `allowAllPython`，
宿主在跑图 / 问端口 / 取节点元数据前把它落到本进程 env 的 `L_NODE_ENGINE_ALLOW_PKGS=*`，
不动用户的 `allow.json`；关掉则还原启动时的原值）。

### 2.5 常用命令

```cmd
wuwor l_node_engine  -- l_node_engine_selftest      :: 引擎自证（18 组，含 AI 路由表 / 标志位 / 类节点）
wuwor l_mindmap_mmd  -- l_mindmap_mmd_web_test      :: 前端协议层单测（30 项，node --test）
wuwor l_mindmap_mmd  -- l_mindmap_mmd_web_build     :: 重建静态产物（→ src/l_mindmap_mmd/static/dist）
wuwor l_mindmap_mmd  -- l_mindmap_mmd_web_dev       :: Vite 开发服务器（:5175，/api 代理到 8110）
```

### 2.6 在 agent 里当工具

```python
from l_agent_tool import pygraph_tools
pygraph_tools.register_pygraph_tools()      # 扫目录 → 注册 pygraph_<图名>
pygraph_tools.register_pygraph_reload_tool()# 注册 pygraph_reload（模型自己重扫，免重启）
```
- `l_agent_chat` 侧由 `src/l_agent_chat/pygraph_tools.py` 在导入时完成（`agent_client.py` 一行导入），**失败只记日志**，不拖垮 agent 启动。
- 开关：`L_AGENT_PYGRAPH_OFF=1` 整体关；`L_AGENT_PYGRAPH_DIR` 换目录；`L_AGENT_PYGRAPH_PREFIX` 改工具名前缀（默认 `pygraph_`）。
- 工具参数 = 图里 `Input` 节点的**静态快照**（注册时固化），返回值取 `Output` 节点。
- **演练模式**（`in_dry_run()`）会拒：这张图能跑任意 python，`DRY_RUN_BLOCKED` 那份按工具名匹配的名单里不可能有它，所以拦在工具实现里。

### 2.7 画布上的开关与工具（都是浏览器偏好或纯视觉，不进执行）

| 控件 | 作用 | 存在哪 |
|---|---|---|
`▶ 运行` | 手动跑一次（SSE 推进度/结果） | — |
`运行日志` | **画布底部**的横带（横跨画布 + 属性面板整条底边，可「收起」成一行；收起后画布自动长高） | 运行时状态 |
`⚡ 实时` | **改动即跑**：结构改动（加删节点/连线/改参数/切标志位）后 700ms 防抖自动保存并运行；突发改动合并成一次。开开关时先跑一次 | `localStorage.pygraph.live`（不进图文件） |
`⊞ 吸附` | 拖拽吸附 15px 网格 | `localStorage.pygraph.snap` |
`原点标记（0,0）` | 画布坐标原点那一点上的**橙色圆点** + `0,0` 文字（跟着平移/缩放走；文字固定 11px，圆点 `10×zoom` 夹在 8–22px）。**不是**固定在屏幕中心 —— 它标的是**图坐标 (0,0)**，和上面标尺的 0 刻度对齐。纯展示（`pointer-events: none`）。想看屏幕中心那个点用鼠标读数（画布底部那条） | — |
`坐标点阵 + 坐标文字` | 画布上每隔一段距离一个**橙色小点**，每 **5** 个点（屏幕 ≥320px）在旁边写一次坐标 `x,y`（与原点的 `0,0` 同一种写法）。步长 = **底纹网格间距的整数倍**（`GRID.gap × k`，k 取到屏幕上 ≥64px），所以橙点**正好压在灰点上**。小点是**一个 CSS 平铺 div**（`radial-gradient` + `background-position` 锚到图坐标），**不是**几百个元素 —— 平移/缩放不占 React 重渲染预算（画布上有 FPS 读数）。**垫在节点下面**（靠 DOM 顺序：`<CanvasRulers />` 排在 `<ReactFlow>` 之前），所以点不会糊在卡片上 | 常量在 `CanvasRulers.jsx`（`GRID` / `GRID_OFFSET` / `LABEL_EVERY`）与 `.cnv-lattice` 的 CSS |
`（网格几何）` | ⚠ 底纹网格（`<Background>`）与点阵**必须共享同一份几何**：React Flow 把灰点画在 `i*gap + (gap+size)/2` 处（不在整数格点、原点处也没点），所以 `<Background>` 要带 `offset={(gap+size)/2}` 把它推回 `i*gap`，否则橙点永远落在灰点缝里（用户截图一眼看出来）。`offset` 的单位/方向**文档没写**，是实测反推的（8.5 → 差 1 个 flow 单位、7.5 → 差 2，斜率 -1 → 9.5 → 0） | `CanvasRulers.jsx` 的 `GRID` / `GRID_OFFSET`（`PyGraphEditor` 从这里 import；改一处两边都对） |
`＋ 分组框` | 画一个**视觉分区框**：可拖、可缩放（选中后拖右下角）、可命名（面板里改）。**成员节点独立**（拖框不带成员、不被裁剪），**不参与执行** | 图文件顶层 `groups`（与 `nodes` 平级；`toPrompt()` 从不输出它） |
**AI 服务 ▸** | AI 节点的后端配置（hub 基址 / lugwit 账号+密码 / 接入密钥 / STT 地址）→ 写 `ai.json`（**数据目录**，与 `pygraphs/` 平级）；**密钥与密码只显示"配了没"**，输入框永远从空开始；**清空某项 = 删掉它** | `ai.json`（见 §3.0.1；优先级 env > 文件 > 默认） |

**节点标志位（6 个，前三个有执行语义）**

| 标志位 | 语义 | 谁实现 |
|---|---|---|
`bypass`（⏭️ 跳过） | 不执行，把**第一个有值的输入**透传给下游（链路不断） | 引擎 |
`mute`（🔇 静音） | **不执行、输出空**；下游解析输入时**明确报错**（不让 `None` 静默算成 `NONE` 之类假结果） | 引擎 |
`debug`（🐛 调试） | 照常执行，把解析后的输入 + 返回值（**限长 300 字摘要**，载荷要过 SSE）带回宿主与日志 | 引擎 |
`pin`（📌 固定） | **整理布局时保持原位**（pin 的真实语义） | 编辑器 |
`lock_params`（🔒 锁定） | 面板里的**参数控件只读**（标志位那一行不锁，否则解不开） | 编辑器 |
`bookmark`（⭐ 收藏） | 纯标记 | — |

标志位是**图的属性**（`node.flags`），随图存盘；`graph_prompt`/`toPrompt`/`toStorage` 都必须带它，
否则"编辑器点了跳过、跑起来没跳过"。`bypass`/`mute` 的节点**不参与必填校验**（它们不跑）。

**上游失败会逐级点名**：节点失败/静音/跳过且无可透传值时，它的下游会报
`上游节点 N 没有产出（失败 / 被静音 / 被跳过且无可透传值）：输入 'x' 拿不到值。先看那个节点的错误。`
—— 而不是像以前那样把**原始连线数组**当值传下去、报个莫名其妙的
`'list' object has no attribute 'name'`（实测踩到过）。执行器自己的记账路径用 `strict=False` 解析，
不会被这个异常打断（循环体失败时仍按软失败记 `node_errors`）。

**循环区包裹框（自动，Houdini 那种 block 框）**：`LoopStart`…`LoopEnd` **连两个括号一起**被圈住
（紫色多边形 + 左上角 `↻ 循环区 · N 个节点`，N 只算"之间"的成员），**随里面节点（含括号）移动自动跟着变**。

- **形状是最小凸包，不是外接矩形**：节点排布常是斜的/不齐的，一律套矩形会大出好几圈。
  点集 = 每个节点矩形**外扩 padding**（顶部再多留 headroom 给标签）后的四角 → `convexHull()`（Andrew 单调链）
  → 承载它的节点尺寸 = 凸包包围盒，多边形点集在 `data.hull`（`PyLoopWrap` 里用 SVG `<polygon>` 画；外层 div 透明）。
- 范围判定：`从 LoopStart 可达 ∩ 能到 LoopEnd 可达`，扣掉两个括号本身（分支也兜得住）；`LoopEnd` 按 id 顺序
  与尚未配对的 `LoopStart` **一对一**配对。
- 尺寸：按**括号 + 成员**的**实测**尺寸；尺寸没量全就先不画（只圈住一半会看着像算错），下一帧自然出现。
- **不落盘、不进 prompt**：它是纯推导物（存了反而和成员位置打架）；也不可选中/拖拽，`pointer-events: none`
  （点它穿透到画布）。想调整范围就动里面的节点。
- 判定与包围盒都在 `pySchema.loopRegions()` / `loopWrapNodes()`，**有单测**。

**运行队列 + 历史**（服务端）：

- **队列**：同一时刻**只跑一张图**，多的按 FIFO 排队（`MAX_RUN_QUEUE = 4` 满则 `429`）。SSE 会先推 `queued`
  （带"前面还有几个"）再推 `started`。以前靠前端掐 SSE，但**服务端线程杀不掉**，两轮 python 会叠着跑。
  排队计数**只在跑完/放弃时减**（早减会让"正在跑"的那个不算数，后来者以为没人占槽）。
- **历史**：`GET /api/pygraphs/history?name=<图名>&limit=50` 读 `pygraphs/runs.jsonl`（最近在前，可只取某张图）。
  面板底部「历史」一节显示每次运行的时间 / ✔✗ / 时长 / 摘要，失败的能展开看**每个节点的首条原因**
  （点即定位到该节点）；跑完自动刷新。留痕条目里新增 `reasons`（每节点首条原因，截 200 字，**不塞整份堆栈**）。
  历史行损坏会被跳过（写日志时进程被杀会留半行）。
- **保留策略**：`runs.jsonl` 只追加到 `MAX_HISTORY_LINES = 2000` 行，之后**留最近 N 行**重写（每追加 200 次检查一次，
  原子替换）；面板「清空」走 `DELETE /api/pygraphs/history?name=`（`name` 空 = 全清）；
  **删图时会连它那张图的历史一起删**（别留"图没了历史还在"）。

**问题面板（按节点分组的错误卡）**：右侧面板底部常驻一节「问题」，把**校验错误/警告**与**上一次跑图的运行错误**放一起：

- 运行错误**按节点分组**（`节点 2 · CallMethod（3 条）`），点标题即选中并居中到该节点；首条直接展开，
  其余收进「还有 N 条」，每条都带可折叠的「详细堆栈」（引擎给了 `details`）。
- **一张图里同一个节点可以报多条**：循环区每轮失败都会追加一条（带 `[第 N 轮]` 前缀）——
  引擎侧是**追加**不是覆盖（覆盖会把前几轮的失败吞掉）。
- 画布上节点只显示首条，多条时后缀 `＋N 条（右侧面板看全部）`，`title` 里挂全部消息。
- 这一节**不在"选中节点"分支里** —— 否则一选中节点错误列表就消失（以前的毛病）。

> 分组逻辑在 `pySchema.groupNodeErrors()`（纯函数，有单测）。

> 分组框刻意**不是** React Flow 的父子节点（`parentId` + `extent`）：那要处理相对坐标、层次排序、拖框带动成员、成员进出判定——一堆坐标系坑，收益只是观感。

## 三、节点清单（16 种基础 + 6 种 AI = 22）

### 3.0 AI 节点（6 种，输入输出只有 文字 / 图片 / 视频 / 音频）

| 节点 | 入 → 出 | 默认后端 |
|---|---|---|
| `AIText` | 文字 → 文字（LLM） | hub `POST /v1/chat/completions` |
| `AIVision` | 图片 + 文字 → 文字（VLM，图片走 data URL 塞进多模态消息） | 同上（`model` **要点名带 vision 的那只**，`auto` 的降级链可能先撞上纯文本模型） |
| `AIImage` | 文字 → 图片（**输出图片文件路径**） | hub `POST /image`（`provider` 默认 `volcengine`） |
| `AIVideo` | 文字（可带首帧图）→ 视频（输出路径） | hub `POST /video`（`duration` 整秒、首帧走 `image_url`） |
| `AISpeech` | 文字 → 音频（TTS，输出路径） | hub `POST /v1/audio/speech`（`model` 默认 `indextts2` = 本机 IndexTTS2） |
| `AITranscribe` | 音频 → 文字（STT） | **hub 没有这条路由** → 必须自己配 `L_NODE_AI_STT_URL` |

⚠ **`HUB_PATHS` 是"hub 真实路由表"，不是"OpenAI 兼容面"** —— 2026-10-11 核对过：
`sst` 故意**不在表里**（hub 未提供），image/video 是 hub 原生路由（不是 `/v1/images/generations`
那种 OpenAI 面，那两条在 hub 上 **404**）。改这张表之前先看图：hub 侧的权威是
`GET :8462/openapi.json`（匿名可读）+ `server.py` 的 `@app.post` 清单。

**四条契约**（改之前先读）：

1. **媒体一律用文件路径**：产物写进 `<pygraphs 目录>/media`（`L_NODE_MEDIA_DIR` 可覆盖）；
   节点的 `ui` 里给的是宿主相对 URL `api/pygraphs/media/<文件名>`，前端按 `ui.image/audio/video` 渲染。
2. **地址与凭据走三条通道，优先级 env > `ai.json` > 默认**（见 §3.0.1）。默认后端就是 **`l_model_hub`**
   （`L_NODE_AI_HUB_URL`，默认 `http://127.0.0.1:8462`）—— 所以"什么都不配"也能用：hub + 该模态路由。
   单个模态想接别的服务就设 `L_NODE_AI_{LLM,IMAGE,VIDEO,TTS,STT}_URL`（如视频指向 `l_video_hub` / `l_ltxvideo`）。
   载荷**按目标分流**：目标是 hub 就发它认的原生字段（`provider` / `duration` / `image_url` / `model=indextts2`），
   是别的服务就只发通用字段 —— 免得给对方塞它不认识的键（严格的服务会 400）。
3. **两种回法都接**：服务直出二进制（Content-Type 不是 JSON）→ 按 mime 定后缀落盘
   （hub 的 TTS 就是这种）；回 JSON → 认 `path` / `b64`（含 `image_base64` 等同族键）/ `url`，
   LLM 那条另认 `choices[0].message.content`。
4. **凭据：接入密钥不是必须的**（2026-10-11 实测）。hub 只要求 `Authorization: Bearer <lugwit 登录 token>`，
   所以三档任选，**优先级**：`L_NODE_AI_API_KEY`（ai.json 的 `key`）> env `LUGWIT_ACCESS_TOKEN` >
   **`user`+`password` 自己登录换 token**（`ai_auth.py`，进程内缓存按 JWT exp、401 时作废重登一次）。
   实测：只填 `admin01`/`666`（`POST 127.0.0.1:1027/api/v1/auth/login` 拿 574 字符 token）→
   `AIText` 经 hub 拿到真答案（`node_errors == {}`），**零接入密钥**。
   ⚠ hub 的 `/v1/*` 与原生路由都要鉴权（`gate.py`：不区分回环；`gate.py` 会打一行 `[gate] 401 … 凭据头=[…]` 便于排错）。

⚠ **安全边界**：宿主只暴露 `GET /api/pygraphs/media/<单个文件名>`，只从媒体目录读、拒绝任何带 `/` `\` `..` 的名字
（实测：正常文件 200，穿越尝试 404/400）。别把它改成"按路径取文件"。

### 3.0.1 面板上的「AI 服务」入口（`ai.json`）

服务卡片只能带**启动参数**，带不了环境变量；而 AI 节点要用的地址与凭据都是环境变量 —— 所以有一份落盘配置
**`<pkg_data_dir>/ai.json`**（即 `~/.lugwit/main/l_mindmap_mmd/ai.json`，**与 `pygraphs/` 平级**）。

```json
{ "hub": "http://127.0.0.1:8462", "key": "sk-lmh-…",
  "user": "admin01", "password": "666", "auth_url": "http://127.0.0.1:1027",
  "urls": { "llm": "", "image": "", "video": "", "tts": "", "stt": "" },
  "media_dir": "", "timeout": 120 }
```

- **优先级 env > 文件 > 默认**；每次读取都**重读文件**（改完立即生效，不用重启服务）。
  想换位置设 `L_NODE_AI_CONFIG_DIR`（测试/隔离实例用；⚠ 它**不跟着** `L_MINDMAP_PYGRAPH_HOME` 走
  —— 图目录能被指到任意位置，"上一层"就不是数据目录了）。
- **凭据二选一**：`key`（接入密钥，优先）或 `user`+`password`（lugwit 账号，自己换 token）。
  账号密码是**明文落盘**（与 `l_agent_chat` 的 `settings.json` 同口径；本机数据目录、不入库），
  不想留就把 `key` 发给用户 / 让 `LUGWIT_ACCESS_TOKEN` 进环境。
- 界面上：编辑器右侧面板底部「**AI 服务 ▸**」（hub / 账号 / 密码 / 接入密钥 / STT 地址）；
  脚本化：`GET|PUT /api/pygraphs/ai-config`。
  **空值 = 删掉该项**（回到 env/默认），不是"留空不改" —— 界面与端点都是这个语义，别记反。
- `GET` **只回掩码**（`has_key` / `key_masked` / `has_password`），明文密钥与密码永不回前端；
  `PUT` 与执行面同门槛（`L_MINDMAP_ALLOW_RUN` 关着就 403），因为这里能写**凭据**与后端地址。
- ⚠ 路由必须声明在 `/api/pygraphs/{name}` **之前**，否则会被当成图名接走（同 `object_info` / `media`）。
- ⚠ **别把这份配置放回 `pygraphs/`**：那个目录里任何 `*.json` 都会被 `GET /api/pygraphs` 当图列出来，
  而且**用户给自己的图起名 `ai` 就会与它撞同一个文件**（2026-10-11 实测踩到：面板里写完配置，
  "打开图"下拉里冒出一个叫 `ai` 的条目）。现在文件在上一层，宿主也会把"不像图"的 json 过滤掉
  （见 `list_pygraphs_api` 的形状检查）。

### 3.1 基础节点（16 种）

| 分类 | 节点 | 用途 / 端口要点 |
|---|---|---|
`l_node_engine/flow` | `LoopStart` / `LoopEnd` | 循环括号。**不要画回边**：`LoopStart → 循环体 → LoopEnd`，环由执行器接管。`LoopEnd.items` 是逐轮收集，`state` 回灌下一轮。轮数上限 `L_NODE_ENGINE_MAX_ITERATIONS`（兼容旧名 `L_COMFYUI_MAX_ITERATIONS`） |
`…/objects` | `MakeInstance` | 造实例。`target`（路径/已注册键）**或** `class_ref`（类对象，优先）；`init_args` 是构造 kwargs 的 JSON；`singleton` 决定复用 |
| | `CallMethod` | 调对象方法。固定三项 `target/method/kwargs_json` + `class_ref` 可选；**方法参数端口按节点问**（见坑 2） |
| | `GetAttr` / `GetItem` | 取属性 / 取下标（纯数字当整数下标；可取不到给 `fallback`） |
| | `CallFunction` | 按 `module:func` 调任意函数（也支持 `module.Class.method`），端口按签名展开；保留名 `path`/`kwargs_json` |
`…/classes` | `LoadClass` | 路径 → **类对象**（蓝图 Class Reference 引脚），另给路径字符串 |
| | `ClassOf` | 对象 → 所属类 |
| | `ClassInfo` | 类 → name / module / qualname / bases / doc |
| | `CastTo` | `ok` + `value` 两路；不匹配 `ok=False, value=None`（Cast Failed 引脚），`strict=True` 则报错 |
| | `IsChildOf` | 类是不是某基类的子类（给实例则等价 `isinstance`） |
| | `SubclassesOf` | 类 → 子类列表（`classes` 是真实类对象列表，可递归；`paths` 是路径字符串，便于打印） |
`…/io` | `Input` | 图的**参数入口**（= agent 工具的形参）：`name` / `value`（默认值）/ `type` / `description` |
| | `Output` | 图的**返回值出口**（= 工具返回）。按 ComfyUI 约定返回 `{"result": …, "ui": …}`，`ui` 供画布内联预览 |
| | `Preview` | 显示任意结果：**原样透传**（插在链中间不断流），同时把值交给画布渲染（`mode` = auto/text/json/repr） |

## 四、坑（按"会咬人"排序）

1. **服务是 `.dev_mod` 但引擎不在它眼里**：改 `l_node_engine/*.py` 后，`l_mindmap_mmd` 不会自重启 —— 必须 `wuwo svc reload l_mindmap_mmd_server`，否则现象是"代码明明改了却没生效"。
2. **动态端口按节点问，不许改回"上次上下文"**：`CallMethod`/`CallFunction` 的 `INPUT_TYPES` 是**确定性**的（只有固定几项）。曾用"上一次执行过的 (target, method)"全局上下文展开端口，结果同一张图在不同会话/时刻画出的行都不一样。现在由编辑器按节点问 `GET /api/pygraphs/ports?target=&method=&kind=`（`kind=function` 走函数）。
3. **缺 handle 的边会被 React Flow 静默丢掉**：所以编辑器**预取所有** `CallMethod`/`CallFunction` 的端口（`portsKey` 只含 id+目标+方法，拖拽不重取），画布与面板都用 `mergedInputSpecs()` **同一份合并结果**。症状：打开保存过的图，"连线不见了"，点一下节点又回来。
4. **标志位是图的属性，不是节点类的**：`{"flags": {"bypass": true}}` 存在图里；`graph_prompt` / `toPrompt` / `toStorage` 都必须带 `flags`，否则"编辑器点了跳过、跑起来没跳过"。**bypass 有执行语义**：跳过该节点并把「按类声明顺序的第一个有值输入」透传给下游（无值给 `None` —— 空 tuple 会让下游按下标取时越界、报错更难懂）。bypass 的节点**不参与必填校验**。
5. **实例键语义**：`MakeInstance(target=裸路径, init_args={})` 注册的是**头部键**；带 `init_args` 注册的是**参数化键**（`pkg:Cls(tag="a")`）。`CallMethod(target=裸路径)` 优先命中头部键，否则取"最近登记的那个"。要确定性就**用确切的键**或走 `class_ref`。
6. **`instantiate(target, kwargs=…)` 不许吞构造参数**：`target` 恰好也是注册键时，旧实现直接返回旧对象、把 kwargs 丢掉（已修：只有"光秃秃要这个键"才复用）。`instantiate_from_class` 与它**同一套键算法** —— 否则 `class_ref` 与路径写法会各造一个对象。
7. **演练模式要在"执行点"拦**：按工具名匹配的名单（`DRY_RUN_BLOCKED`）拦不住运行时注册的工具，也拦不住图里的节点。pygraph 工具自己判 `in_dry_run()`。
8. **前端产物没有内容 hash**：模板里用 `?v={{ asset_ver() }}`（`main.js` 的 mtime）做 cache-busting。**症状：改了前端、重建了，页面却像没变 → 先怀疑浏览器缓存**（我自己被坑过一次）。
9. **超时只是"放弃等待"**：Python 杀不掉线程，节点里死循环会继续跑（超时事件文案里写明了）。执行也是单线程串行，SSE `abort` 只是前端不再读。
10. **`Output` 的 `{"result","ui"}` 形状**：`ui` 是给宿主做**节点内联预览**的（前端按 ui 的**键名**选渲染器：`text`/`json`/`repr`/`value`）。`run_pygraph` 取 `result`；SSE 汇总也按 `result` 呈现，别把整只 dict 丢给前端。

## 五、怎么自证（改完跑什么）

| 改了什么 | 跑什么 |
|---|---|
`l_node_engine/**` | `wuwor l_node_engine -- l_node_engine_selftest`（18 组；新增能力请加组） |
`web/src/pySchema.js`、校验/映射口径 | `wuwor l_mindmap_mmd -- l_mindmap_mmd_web_test`（30 项） |
`l_node_engine/ai_nodes.py`、AI 路由/载荷/凭据 | selftest 的 AI 组（验路由表 + 载荷 + 落盘 + ai.json 通道 + 凭据三档/过期重登/401 换一枚重试一次）；**再核对一次 hub 的 `GET :8462/openapi.json`** —— 路由表是抄出来的，不是推出来的；带真凭据的端到端见 §3.0 第 4 条（本机 dev 账号，别拿去公网机） |
`web/src/*.jsx`（画布/面板/预览/框/错误卡/历史/AI 配置） | 重建 + 浏览器点一遍（FPS 读数、预览、标志位、实时模式、分组框、循环区包裹框、问题面板、历史、AI 服务） |
`l_agent_tool/pygraph_tools.py`、agent 接线 | `wuwor l_agent_chat -- python -c "...loadTestsFromNames(['tests.test_pygraph_tools'])"`（14 项） |
改图/新增 demo | `curl -X PUT …/api/pygraphs/<图名>` + `POST …/<图名>/run` 看 `node_errors == {}` |

现成示例图（`~/.lugwit/main/l_mindmap_mmd/pygraphs/`）：**`demo_all_nodes`**（用到全部 16 种节点 + 一个 bypass + 4 个 Preview）、**`demo_ai_image`**（**生图样板**：中文需求 → `AIText` 改写成英文提示词 → `AIImage` 出图 → 两个 `Preview`（提示词 / 图）+ `Output`；`provider` 是个 `Input`，默认 `zhipu` = 智谱 **CogView-3-Flash（免费模型）**，改成 `free` 只用免费档、留空走自动链 —— 免费档与坑见 `l_model_hub.md`「图片生成」）、`demo_objops`、`demo_tool_add`、`t2`。

> ⚠ 跑任何**带 AI 节点**的图之前，先在右侧「AI 服务 ▸」里配好模型中心凭据（lugwit 账号密码，或接入密钥）：
> 节点是在**服务进程里**打模型中心的，拿不到浏览器的登录态（见 §3.0.1）。

## 六、已知未做 / 限制

- 队列只有"深度 4 的 FIFO"（没有优先级 / 取消排队；正在跑的那个杀不掉 —— 同"超时只能放弃等待"）。
- 分组框是**视觉框**（不带动成员、不做父子绑定）；要"整块搬走"得自己多选拖。循环区包裹框同理（自动生成、不可交互）。
- **hub 的图片/视频两条原生路由没真跑过**（未测：出图/出片能不能成、首帧 data URL 火山认不认）。
  LLM 那条**已验证**（只用 lugwit 账号密码，见 §3.0 第 4 条）。
- **STT 没有标准形状**：hub 不提供，节点发的是"任意本机服务的通用形状"（`audio_b64` + `filename`）——
  接自己的服务时可能要改。
- 图表 / 3D / PDF 等其余预览类型没做（图片/音频/视频已做：产物落媒体目录 + `ui.{image,audio,video}` + 宿主只有
  单文件名的媒体路由，别放宽那个安全边界）。
- 折叠 / pin 的其余交互（节点标题栏折叠）没做；`bookmark` 是纯标记（没做筛选/跳转）。
- 前端只有协议层单测（`pySchema`），组件层没有自动化。
- `l_comfyui_backend` 里还躺着一份同源但更旧的执行内核（`node_base`/`node_registry`/`prompt_executor`），尚未接到 `l_node_engine`，两处口径可能漂。
- ⚠ **写前端单测的坑**：`web/src` 里相对导入**不带扩展名**（Vite 能解析，Node ESM 不能）。所以被测试的模块若引了同级文件，必须写成 `"./x.js"`（如 `pySchema.js` 引 `flowSchema.js`）——否则 `node --test` 报 `ERR_MODULE_NOT_FOUND`。
