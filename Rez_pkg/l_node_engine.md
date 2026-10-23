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
`l_mindmap_mmd_server` | **开**（2026-10-22 起自带 `L_MINDMAP_ALLOW_RUN=1`；之前是关的，跑图 403） | `127.0.0.1` |
`l_mindmap_mmd_server_net` | 开（同一端口，`0.0.0.0`，给别的托盘编辑这台） | `0.0.0.0` |

卡片（`l_homepage` 的 `l_mindmap_mmd_server`）就用日常别名；想**关**执行面就把别名命令里的 `set L_MINDMAP_ALLOW_RUN=1` 去掉（`wuwo svc` 用 `packages + run_args` 拼命令，`run_cmd` 只是展示）。

其它旋钮：`L_MINDMAP_RUN_TIMEOUT`（墙钟超时，默认 60s；**2026-10-12 起可被请求体的 `{"timeout": N}` 覆盖**）、留痕 `~/.lugwit/main/l_mindmap_mmd/pygraphs/runs.jsonl`。
**别在公网部署机上开这个开关，也别把 `--host` 改回 `0.0.0.0`。**

### 2.4 导入白名单（"能 import 哪些包"）

默认**空** = 只放行 `l_node_engine` 自身 + 已注册节点所在模块。放行外部包三条路（优先级从高到低）：

1. 环境变量 `L_NODE_ENGINE_ALLOW_PKGS=json;mypkg`（**设了就以它为准**）
2. 文件 `~/.lugwit/main/l_mindmap_mmd/pygraphs/allow.json`：`{"packages": ["json", "mypkg"]}`（卡片只能带参数带不了 env，才有这一档；改完**立即生效**，`allowed_packages()` 每次调用都重读）
3. 卡片启动参数：`run_args: ["l_mindmap_mmd_server", "--allow-import", "json;mypkg"]`

被拒绝时错误信息会**直接写出这两个办法**（含 `allow.json` 的绝对路径）—— 别把它当成"路径写错了"。

**结构校验（2026-10-12）**：`l_node_engine.validate_structure(prompt) -> {"errors": […], "warnings": […]}`
—— **不执行**就告诉调用方这张图跑起来会不会炸（未知节点 / 悬空连线 / 缺必填 / 括号配不上 / 真环 / 状态不成对）。
实现**直接调执行器那套计划器**（`_build_loop_plans` / `_build_if_plans` / `_topological_sort`），
所以不会出现"校验说没事、跑起来 GraphPlanError"。宿主入口见 `l_mindmap_mmd.md` 的 `/validate`。

**演练模式（dry-run，2026-10-12）**：`l_node_engine.set_stub(fn)` / `get_stub()` —— `fn(node_id,
class_type) -> bool`，判 True 的节点**不执行**、直接产出 `(None,)`（事件里带 `stubbed: true`）。
与 `guard` 的区别（别混）：guard 抛异常 = 该节点**失败**（进 `node_errors`）；演练是"这步没真做"，
图照样跑完 —— 于是能比的正是**控制流/路由**。用途：宿主做沙箱 A/B（`l_mindmap_mmd` 的 `/ab`）。

**通配 = 全部放行（2026-10-11）**：白名单里任一条目写成 `*`（或 `all`）即"允许任意 python 对象"
（`guard.ALLOW_ALL_TOKENS`；env / `allow.json` 都认）。`l_mindmap_mmd` 的编辑器把它做成了设置项
「允许所有 python 对象（导入白名单全放行）」——**默认开**（`settings.yaml` 的 `allowAllPython`，
宿主在跑图 / 问端口 / 取节点元数据前把它落到本进程 env 的 `L_NODE_ENGINE_ALLOW_PKGS=*`，
不动用户的 `allow.json`；关掉则还原启动时的原值）。

### 2.5 常用命令

```cmd
wuwor l_node_engine  -- l_node_engine_selftest      :: 引擎自证（20 组，含 AI 路由表 / 标志位 / 类节点 / 增量烹饪）
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
`👁 2D 取景器` | 节点卡片头上的 👁（**只在输出含 `IMAGE` 端口的节点上出现**）→ 画布与属性面板之间多一栏看图：滚轮缩放、拖拽平移、双击适应、`1:1` 看 texel、**RGB/R/G/B/A 通道切换**、⛶ 铺满画布区、Esc 关。数据走宿主 `GET /api/pygraphs/render?name=&node=`（**上一次运行**的输出，内存里按需渲染，长边 ≤1024、RGBA；`X-Tex-Size`/`X-Tex-Mode` 在响应头）。输出是**文件路径**的节点（`TexExportImage`、AI 节点）render 会 404 → 取景器回退用它的 `ui.image` | 不落盘；只在内存留最近一次 |
`节点缩略图 / 算子库示例图` | 节点卡片底部显示**上一次运行**的输出小图（`?thumb=1`，≤128px；**没跑过不请求**）；左侧调色板里 `l_tex_nodes` 那 21 个算子带 **96px 示例图**（宿主用 `l_tex_nodes.samples` 现烤，`GET /api/pygraphs/palette/<类名>`，与跑图同门槛；没示例的 404 → 前端只显示名字） | 都是内存缓存（`_TEX_RENDER` / `_PALETTE_CACHE`） |
`🧩 增量：开/关` | **增量烹饪**：结构没变的节点复用上一轮结果，只重算脏的。实测 1024² 5 节点图：冷跑 2.8s → **参数没变 8ms**（5/5 复用）、**只改下游 0.27s**（上游重活全复用）、改上游 ~3s（脏按拓扑往下传）。复用过的节点在画布上挂 `🧩` 徽标、日志报"复用 N 个，只重算了 M 个"。**关掉 = 每次全量重算**（图里有取时间戳/随机数这类**非纯**节点时用它）。缓存键是**结构指纹**（`class_type + 字面量参数 + 上游指纹`，不做内容哈希），**括号节点（循环 + 如果）整段与区内节点** / `bypass` / `mute` 都不进缓存（见坑 16）；**删图会把它的缓存一起丢**（里面是数组级结果） | `localStorage.pygraph.incremental`；宿主 `_RunCache`（按图名，上限 120 节点） |
`鼠标手势` | 左键 = **点选 / 框选**（**不平移**）；中键 / 右键**拖** = 平移；右键**单击** = 右键菜单；滚轮 = 缩放。参数（`selectionOnDrag` + `panOnDrag={[1, 2]}`）**与流程图编辑器逐字一致** —— 两套画布手感必须一样；React Flow 默认是"左键拖 = 平移"，与节点编辑器的肌肉记忆相反（用户口径 2026-10-12）。**框选判据 = `Partial`（2026-10-13）**：框角**压到节点一点就算选中**，不要求整个框住（React Flow 默认是 `Full`，大节点得把视图缩到很小才框得住）；两个后端各有一份实现 —— DOM 走 `view.selectionMode`（`graphView.jsx` / `PyGraphEditor`）、canvas 自己在 marquee 里算**矩形相交**（`CanvasSurface`），所以**改一边必须改另一边** | — |
`运行日志` | **画布底部**的横带（横跨画布 + 属性面板整条底边，可「收起」成一行；收起后画布自动长高） | 运行时状态 |
`⚡ 实时` | **改动即跑**：结构改动（加删节点/连线/改参数/切标志位）后 700ms 防抖自动保存并运行；突发改动合并成一次。开开关时先跑一次。**默认关、不记、切图也不自动开**（2026-10-16 用户口径：「实时按钮默认请关闭」+「切换图时都不再自动开」—— 以前存 `localStorage["pygraph.live"]`，开过一次就每次打开都是开的、换图也照旧；现在三件事都堵上：① 进页面不读那个键（顺手把旧键清掉）② **换图（`name` 变）就把实时置回关** ③ 时间线 ▶ 播放**不再代开**，只有手点那颗按钮或时间线上那颗「⚡ 实时没开（点这里打开）」才开） | 曾经是 `localStorage.pygraph.live`（已废） |
`⊞ 吸附` | 拖拽吸附 15px 网格 | `localStorage.pygraph.snap` |
`原点标记（0,0）` | 画布坐标原点那一点上的**橙色圆点** + `0,0` 文字（跟着平移/缩放走；文字固定 11px，圆点 `10×zoom` 夹在 8–22px）。**不是**固定在屏幕中心 —— 它标的是**图坐标 (0,0)**，和上面标尺的 0 刻度对齐。纯展示（`pointer-events: none`）。想看屏幕中心那个点用鼠标读数（画布底部那条） | — |
`坐标点阵 + 坐标文字` | 画布上每隔一段距离一个**橙色小点**，每 **5** 个点（屏幕 ≥320px）在旁边写一次坐标 `x,y`（与原点的 `0,0` 同一种写法）。步长 = **底纹网格间距的整数倍**（`GRID.gap × k`，k 取到屏幕上 ≥64px），所以橙点**正好压在灰点上**。小点是**一个 CSS 平铺 div**（`radial-gradient` + `background-position` 锚到图坐标），**不是**几百个元素 —— 平移/缩放不占 React 重渲染预算（画布上有 FPS 读数）。**垫在节点下面**（靠 DOM 顺序：`<CanvasRulers />` 排在 `<ReactFlow>` 之前），所以点不会糊在卡片上 | 常量在 `CanvasRulers.jsx`（`GRID` / `GRID_OFFSET` / `LABEL_EVERY`）与 `.cnv-lattice` 的 CSS |
`（网格几何）` | ⚠ 底纹网格（`<Background>`）与点阵**必须共享同一份几何**：React Flow 把灰点画在 `i*gap + (gap+size)/2` 处（不在整数格点、原点处也没点），所以 `<Background>` 要带 `offset={(gap+size)/2}` 把它推回 `i*gap`，否则橙点永远落在灰点缝里（用户截图一眼看出来）。`offset` 的单位/方向**文档没写**，是实测反推的（8.5 → 差 1 个 flow 单位、7.5 → 差 2，斜率 -1 → 9.5 → 0） | `CanvasRulers.jsx` 的 `GRID` / `GRID_OFFSET`（`PyGraphEditor` 从这里 import；改一处两边都对） |
`连线配色 / 流动` | **每条连线的颜色 = 它源端口的颜色**（用户 2026-10-14 口径；判断走 `PyNode.outputPortColor`，与那个 Handle **同一份**，别在画线处另写一套"标准类型优先"）。选中的那条**不覆盖**颜色（仍走 CSS 的强调色）。**虚线流动只给"正在运行的节点之后"那条边**（`animated = animating && e.source === runNode`）—— 以前跑起来**全体**走虚线，图一大满屏乱闪；与"正在执行"的节点相连的边另加 `pyg-edge-hot`（呼吸 + 光晕，**颜色仍不变**）。canvas 后端同一套：**按颜色分批**描边、虚线相位自己推（`lineDashOffset`，canvas 没有 CSS 动画）、有流动边时**自续帧**（`flowOn`，跑完自然停） | — |
`＋ 分组框` | 画一个**视觉分区框**：可拖、可缩放（选中后拖右下角）、可命名（面板里改）。**拖框 = 整框搬运**：框里的节点跟着一起走（成员**起拖那一刻**定：卡片与框有重叠即算，拖到一半路过别的节点不会被顺走；2026-10-16 用户口径），节点自己拖不受框影响、也不被裁剪；**不参与执行** | 图文件顶层 `groups`（与 `nodes` 平级；`toPrompt()` 从不输出它）；成员判据 `pySchema.rectsOverlap` / `groupMembers`（两后端共用；手势分别在 `PyGraphEditor.onNodeDragStart` 与 `CanvasSurface.groupMembersOf`） |
**AI 服务 ▸** | AI 节点的后端配置（hub 基址 / lugwit 账号+密码 / 接入密钥 / STT 地址）→ 写 `ai.json`（**数据目录**，与 `pygraphs/` 平级）；**密钥与密码只显示"配了没"**，输入框永远从空开始；**清空某项 = 删掉它** | `ai.json`（见 §3.0.1；优先级 env > 文件 > 默认） |

**节点标志位（7 个，前三个有执行语义）**

| 标志位 | 语义 | 谁实现 |
|---|---|---|
`bypass`（⏭️ 跳过） | 不执行，把**第一个有值的输入**透传给下游（链路不断） | 引擎 |
`mute`（🔇 静音） | **不执行、输出空**；下游解析输入时**明确报错**（不让 `None` 静默算成 `NONE` 之类假结果） | 引擎 |
`debug`（🐛 调试） | 照常执行，把解析后的输入 + 返回值（**限长 300 字摘要**，载荷要过 SSE）带回宿主与日志 | 引擎 |
`pin`（📌 固定） | **整理布局时保持原位**（pin 的真实语义） | 编辑器 |
`lock_params`（🔒 锁定） | 面板里的**参数控件只读**（标志位那一行不锁，否则解不开） | 编辑器 |
`bookmark`（⭐ 收藏） | 纯标记 | — |
`autorun`（🔄 开图即跑） | **打开/切到这张图时自动「运行到本节点」**（它 + 上游，内联 prompt、不落盘）；每张图只跑一次、多个只取 id 最前那个 | 编辑器 |

标志位是**图的属性**（`node.flags`），随图存盘；`graph_prompt`/`toPrompt`/`toStorage` 都必须带它，
否则"编辑器点了跳过、跑起来没跳过"。`bypass`/`mute` 的节点**不参与必填校验**（它们不跑）。
`autorun` 是**纯宿主语义**（引擎不解释），所以放进图里也不影响 `graph_prompt` 之外的执行。

> **术语对齐 Houdini**（2026-10-22 用户口径「变量命名、注释请和 houdini 一致」）：UI 显示名与注释一律用
> Houdini 的叫法 —— `Bypass`（= **Bypass flag**，同名同义）、`Lock params`（= **parameter lock**，
> **不是** Lock flag，那面旗锁的是几何）；`Mute` / `Debug` / `Pin` / `Bookmark` / `Auto-run` 是 **Houdini 没有的旗标**，
> 每个的 `note` 里写明"Houdini 对应什么/为什么没有"。**键名不改**（`bypass` / `lock_params`… 是图文件里的 wire 格式，
> 老图里就写着它们）；改的只有 `label` / `note` / 菜单文案 / 运行日志文案（`node_base.NODE_FLAGS`、
> `PyNode.FLAG_EMOJI`、`PyGraphEditor` 的右键菜单与 `toggleFlag` 回执）。

**上游失败会逐级点名**：节点失败/静音/跳过且无可透传值时，它的下游会报
`上游节点 N 没有产出（失败 / 被静音 / 被跳过且无可透传值）：输入 'x' 拿不到值。先看那个节点的错误。`
—— 而不是像以前那样把**原始连线数组**当值传下去、报个莫名其妙的
`'list' object has no attribute 'name'`（实测踩到过）。执行器自己的记账路径用 `strict=False` 解析，
不会被这个异常打断（循环体失败时仍按软失败记 `node_errors`）。

**循环区包裹框（自动，Houdini 那种 block 框）**：`LoopStart`…`LoopEnd` **连两个括号一起**被圈住
（紫色多边形 + `↻ 循环区 · N 个节点`，N 只算"之间"的成员），**随里面节点（含括号）移动自动跟着变**。
  ⚠ 那个名字**压在线上**（2026-10-17 用户：「这个文字有办法直接显示在线上么」）：锚点 = 凸包**最上面那条边**
  的中点，标签以它为中心（DOM：`translate(-50%,-50%)` + 底色 `--fx-bg` 抹掉线的一段；canvas：
  `drawLabelOnLine`，同一套锚点 + 同一个底色 `CANVAS_BG`）—— **没有那个小边框药丸**了。两后端一起改，
  判据 `web_probe hull` ①′（标签中心与那条边中点的偏差 ≤2px、`border-top:0`）。

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

- **队列**：同一时刻**只跑一张图**，多的按 FIFO 排队（上限 `L_MINDMAP_RUN_QUEUE`，默认 **8**，1..64；满了 `429`。
  2026-10-20 之前是写死的 4 —— 长节点一占几分钟，4 格一撞就 429）。SSE 会先推 `queued`
  （带"前面还有几个"）再推 `started`。以前靠前端掐 SSE，但**服务端线程杀不掉**，两轮 python 会叠着跑。
  排队计数**只在跑完/放弃时减**（早减会让"正在跑"的那个不算数，后来者以为没人占槽）。
- **中止（2026-10-20）**：`POST /api/pygraphs/runs/<run_id>/cancel` 现在对**正在跑**的那次也有效 ——
  宿主置一枚 `threading.Event` 传给 `execute_graph(cancel=…)`，引擎在**节点边界**（每个节点前、
  循环每一轮/每一波前、并发批每一层前）检查，真中止就抛 `GraphCancelled`（**`BaseException` 子类**，
  不进 `node_errors`、不写"哪个节点报错"）。SSE 收尾事件是 `cancelled`（带 `steps_done`）。
  ⚠ 边界只在**节点之间**：正在执行的那个节点要跑完（Python 线程杀不掉）—— `AIVideo` 那种
  长节点点完停止要等它这一步；节点内部的超长循环想响应中止，自己轮询 `graph.cancel_requested()`。
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

> 分组框刻意**不是** React Flow 的父子节点（`parentId` + `extent`）：那要处理相对坐标、层次排序、成员进出判定——一堆坐标系坑，收益只是观感。成员按"拖的那一下"现算（`groupMembers`），不是常驻绑定 —— 所以没有"成员进出"这类状态要维护。

### 2.8 界面分区 = 可搬的标签条（2026-10-12）

`/pygraph` 的界面是**三块可搬的标签面板**：左栏（🧩 节点库）、右栏（🔧 属性 / 🤖 AI 服务 / ⚠ 问题 /
🔬 A/B / 🧾 审计 / 📈 轨迹 / 🕐 历史）、底栏（📂 内容浏览器 / 📋 运行日志 / 🧊 3D 视图 / 🖼 2D 视图）。引擎在
`web/src/pyTabs.js`（模型 + 落盘 + 尺寸拖拽 hook）与 `web/src/PyTabStrip.jsx`（标签条 + 面板壳）。

⚠ 加标签的两处必改：`pyTabs.TAB_DEFS` 登记（`home` 决定默认落哪个面板，旧存档会自动补进来）；
面板体里**每个标签都要 `tabs.isActive(panel, key)` 守卫**（面板的 `children` 是全量渲染的，
不守卫就会出现"切成 A 标签、B 标签还挂在下面"）。`🧊 3D 视图` 是这么加的（`Viewer3DPanel` →
`MeshViewer docked`，与节点卡片上的 👁 同一份渲染）。

⚠ 补进来的键**按 `TAB_DEFS` 的顺序插到它该在的槽位**（不是一律追加到末尾）—— 「📂 内容浏览器」
要压在「📋 运行日志」前面就是靠这条（`normalizeTabOrder`）。改标签的 `key` = **有意清掉老布局**
（老存档里那个 key 认不出来 → 按新 `home` 重新落位）。

**交互**（与 `lugwit_baidu_netdisk` 的 `web_depot.html`「标签引擎」同款 —— 那边是原生 JS 搬 DOM 节点，
这里是 React 按归属渲染）：

| 操作 | 效果 |
|---|---|
| 拖动标签 | 本面板内 = **排序**；拖到别的面板 = **连同内容搬过去**（落点有 3px 指示条；拖到边上自动横向滚） |
| 标签条末尾 `＋` | 列出**别的面板**的标签，选一个就搬到本面板末尾并切过去（等于"拖过来"的鼠标版） |
| **标签头右键** | 那一张标签的菜单：**「✖ 删除标签页」**（2026-10-16 用户口径「标签头右键菜单添加删除标签页功能」）。删掉 = `pyTabs.removeTab` → `pyLayout.removeTab`（**组空了顺带剪掉**；`?tab=` 指的正是它时一并清掉）；**钉住的画布不给这一项**，菜单里只写"钉住的标签，删不掉"。想拿回来：`＋` 菜单再拉一张、或把别处的标签拖进来 |
| 拖缝 | 左↔画布、画布↔右栏两条竖缝改宽，底栏上方横缝改高；**双击复位**（左 210 / 右 300 / 底 150） |
| 面板被搬空 | 自动收成 **26px**（标签条藏起来，只剩 `＋` 当落点 —— 搬回来即可复原） |
| 底栏「收起/展开」 | = 那个面板的折叠（只留标签条）；回放时会自动展开（不然回放条看不见） |
| 刷新 | 归属 / 每个面板的活动标签 / 尺寸 / 折叠 **全存 `localStorage["pygraph.ui"]`**（防抖 300ms + `pagehide` 补一记） |
| 工具栏 `↺` | 界面分区复位成**内置默认摆法**（只影响这一张图的存档） |
| `Ctrl+Z` / `Ctrl+Shift+Z`（= `Ctrl+Y`） | **撤销 / 重做**（2026-10-14 补；同 `FlowEditor` 的键位）。工具栏 `↶ ↷` 与之等价，灰=没得退/没得重做。**画布上的任何改动都退得掉**（拖节点/拖框/缩放框/整理布局/改参数/增删/连线…，2026-10-16 收口，见下面口径） |
| **点一下端口** | **引出连线**（2026-10-14 用户口径）：点输出 → 点输入 = 接上；再点同一个端口 / `Esc` / 点空白 = 取消。画布上给一条「🔗 从 x.out 引出中」提示条。⚠ 别指望 React Flow 的 `onConnectStart` —— **它只在真拖动时才发**（单击 down→up 连事件都没有，实测），所以这条挂在我们自己 `<Handle>` 的 `onClick` 上（`PyNode`），两种后端都归到 `PyGraphEditor.portClick` |
| **节点库拖到画布** | 在「🧩 节点库」里**按住一个节点类型拖到画布**，松手处落节点（点一下仍然是"再点画布落点"那套）。HTML5 DnD；⚠ 类型走模块级 `paletteDrag`（`dragover` 里 `dataTransfer.getData()` 读不到），落点走 `view.screenToFlow` ✓ 两种后端同一份 |

⚠ **撤销/重做的口径**（`PyGraphEditor.jsx` 里 `undoRef/redoRef/pushHistory/mutate/mutatePlace`）：快照 = **整张图**
（`nodes` / `groups` / `note` / `nodes[id].keys` 都在里面），上限 100 步；
**布局树与时间线的时长·范围·播放头不进历史**（那是浏览器偏好，撤销不该把面板搬回去）。
粒度 = **静默期合并**：450ms 内的一串 `mutate` 算**一步** —— `mutate` 每个拖动帧都会跑，
不合并的话"拖一下节点"就是几十步（等于没有撤销）。
**用户口径 2026-10-16：「我对画布的一切改动都请可以 ctrl+z」** ⇒ 图上任何改动都必须走
`mutate`（唯一入口）；**只改"摆法"的那几种**（拖节点 / 拖分组框 / 缩放分组框 / 整理布局）走
`mutatePlace` = `mutate(fn, {noLive:true, step:true})`：进撤销栈、**一次手势单独占一步**
（拖两下能撤两次），但**不置 `liveArmed`**（挪个位置不该触发实时重跑）。
⚠ 这两条以前是**裸 `setGraph`**（既不进栈也不 bump version）⇒ "拖完按 Ctrl+Z 一动不动、
撤掉的反而是上一步"，看着像坏的（同日的 `web_probe.cjs undo` 场景就是这条的判据）。
换图（`openGraph`）清历史。
焦点在 `input/textarea/select/contenteditable` 里时**不抢** Ctrl+Z（留给浏览器"撤销输入"）。

⚠ **"工作区"这个概念已经删掉**（2026-10-13 用户口径："删除这个按钮，另外删除工作区的概念"）：
原来工具栏那一组是 `🧩 下拉（内置 默认/宽画布/调试图/素材图 四套 + 用户"存为…"自己那几套）+ 存为… + 🗑`，
存档形状是 `{layouts: {工作区名 → 布局}, current}`。现在：**一份布局就是一份** ——
服务端 `_ui` 里只有 `layout` 一个键，前端只剩 `↺` 复位。老存档（`{layouts, current}`）在
**服务端读的时候**拍平成一份（`server.py` 的 `_ui_layout_of`，取 `current` 指的那份），
所以用户排过的摆法不会因为删概念就丢；代码侧的落点：`pyLayout.js`（删 `BUILTIN_PRESETS` /
`presetRoot` / `applyPreset` / 布局里的 `name` 字段）、`pyTabs.js`（删 `applyPreset` / `setName`）。
**别再加回来**：要几套摆法直接拖标签/拖缝摆就是了。

三条**有意**的分歧（别按 depot 改回去）：

1. **画布、顶栏、2D 取景器不参与**标签系统 —— 画布是 workspace，`.pyg-center` 的 `grid-area`
   绑定（`has-viewer` 那套）与 `CanvasRulers`/`overlay` 的 props 契约都绑死在它身上。
2. **「人在环」的询问条（Ask）挂在工具栏正下方**，不进任何标签体 —— 跑到一半要人回答时，那条要是
   被切到看不见的标签里（或底栏折着），图就一直挂着等人。
3. **切标签/搬面板 = 内容卸载重挂**（depot 是搬 DOM 节点、内容不重挂）。这里所有业务状态都在
   `PyGraphEditor` 的 `useState`（受控输入 / `aiDraft` / `log` / `history`…），所以重挂不丢数据；
   会丢的只有**非受控** DOM 状态（`<details>` 展开、滚动位置）。

⚠ 四个踩过的坑：① 存档键名要对上（写 `order`、读 `tabs` → **刷新后布局悄悄回默认**，最难从界面看出来）；② 标签条**不缩、不画滑块**（2026-10-13 用户口径："如果不够显示标签，那就不准缩小标签组宽度，
然后删除这个横线滚动滑块"）—— `.pyg-tabstrip` 是 `overflow:hidden` + 标签 `flex:0 0 auto`：
位置不够就**裁**。**别改回 `overflow-x:auto`**（那条滑块就是被点名删掉的）。
代价与对策（原来阻止这么改的理由，现在靠这两条兜住）：`overflow:hidden` 的盒子**照样能程序化滚动**，
所以 ① 末尾「＋」把标签叫过来（`headRef` 判"点的是不是自己这条"）；② 拖动到边缘（`edgeScroll` 直接改
`scrollLeft`）+ 活动标签 `scrollIntoView` 都还在 —— 真正失效的只有"鼠标滚轮手动滚"，那条没人要；
③ `.pyg-tabhead` **不能** `overflow:hidden`
（末尾「＋」的下拉是它的 `absolute + top:26px` 子元素，一裁就整块看不见 —— "点了没反应"，
而 DOM 里查得到：**用 `querySelector` 的测试会漏掉这类"渲染了但被裁掉"**，要么真点、要么查 `elementFromPoint`）；
④ 「＋」菜单的关闭**不能用 `stopPropagation` 拦**（一页三个 ＋，拦了就"上一块的菜单关不掉"、
两块同时开着）—— 改成"点的是不是自己这条标签条"（`headRef.contains(target)` 才不关）；
⑤ **别给 `.react-flow__viewport` 常挂 `will-change: transform`**（2026-10-14 用户报"放大后卡片上的文字发糊"）：
那会让 Chromium 把它当**合成层**、缩放后**复用缩放前的栅格**（旧位图拉大）→ 字软。
改法：只在**手势期间**加 `.is-gesturing`（`PyGraphEditor` 里用**捕获阶段**监听 `wheel`/指针 ——
React Flow 的 d3-zoom 会 `stopPropagation`，冒泡阶段收不到），停手 ~220ms 去掉 → 按当前缩放重栅格化一次。
⚠ 判据陷阱：这个现象在 **headless / 软件光栅化里复现不出来**（那种路径当场重画，实测 1× 与 3.4× 锐度持平），
真机 GPU 合成才会糊 —— 别拿 headless 的"没差别"当"没这回事"。同类风险点：`.pyg-viewer-canvas`（2D 取景器）。

## 三、节点清单（**83 种**，2026-10-20 核准）

> 分块：基础 4（`Output`/`Reroute`/`Preview`/`SaveText`）+ 数值/集合 19 + 控制流 11
> （`IfStart`/`IfEnd`/`ElseStart`/`LoopStart`/`LoopEnd`/`Branch`/`Select`/`Compare`/`And`/`Or`/`Not`）
> + 结构体 2 + 类图 10 + 对象操作 4（`CallFunction`/`GetAttr`/`GetItem`/`CallMethod`）+ python 1
> + 状态 2 + **遍历 5**（`Map`/`ForEach`/`ForEachConcurrent`/`ForEachThread`/`ForEachProcess`）
> + 子图/工具 2 + 入参/常量 3（`GetParam`/`SetParam`/`Const`）+ AI 6 + 百度 5 + agent 4
> + 引擎区 2（`EngineStart`/`EngineEnd`）+ 时间/问答/备注 3（`Time`/`Ask`/`Note`）= **83**。
>
> 核对方式（改完节点**顺手跑一次**，数字对不上就是文档该改了）：
> `wuwor l_node_engine -- python -c "import l_node_engine as e; print(len(e.NODE_CLASS_MAPPINGS))"`
> —— 自证跑起来会多出 5 个 `_` 开头的测试假节点（`AccNode`/`BoomNode`/`UpperNode`/`wrap_brackets`/`JoinPaths`），
> 那时打印 88，不算在这里。

> **算子不止这些**：图像处理那类（噪波/模糊/色阶/混合/法线…）在**包外**的 `l_tex_nodes` 里 ——
> 引擎刻意只用标准库，带依赖的算子库挂进来即可（见 `l_tex_nodes.md`，39 个 `Tex*` 算子 + `TexSbsar` + 4 个 FX `Fx*`，注册进画布共 **44** 个）。

### 3.0 agent 四件套（`HostStep` / `AgentTool` / `Gate` / `AgentParams`）—— 图也能"跑一轮对话"

引擎**不认识** plan / tools / 审批 / 提示词 / 护栏是什么，它只留钩子；实现在 **agent 进程**（`l_agent_chat`）。
四个节点分**三类**管三件事（都能在画布上看见、存进资产）：`HostStep`/`AgentTool` 是**动作**（改世界、
有返回值），`Gate` 是**决策**（不改世界），`AgentParams` 是**参数声明**（"这一轮
用哪个模型 / 思考多深 / 步数上限"这类成组的参数，一张卡全写上）。

> ⚠ **`AgentPrompt` 已删**（2026-10-17 用户口径：「**AgentPrompt 节点全部删除，统一使用获取参数节点**」）。
> 提示词注入现在是**图级入参**：`graph["params"]` 里叫 `system` / `user` / `tools_allow` / `tools_deny`
> 的那几行（值非空才生效；`system` 追加 / `user` 覆盖本轮问句 / `tools_*` 收窄工具表、黑名单优先）。
> 图上要看/连线就用 `GetParam` 引用它们；老图里残留的 `AgentPrompt` 节点仍能被 agent 侧读出来
> （`graph_turn.prompt_slots`，只读、无节点类型）。

| 节点 | 输入 | 谁实现 | 说明 |
|---|---|---|---|
| `HostStep` | `step`（注册名）+ 任意宿主参数 + `state`/`after`（`forceInput`，只做跨圈状态/排序） | 宿主进程注册的执行器（`register_host_step`） | **图决定控制流、宿主只当执行器**。⚠ `state`/`after` 必须声明，否则编辑器不画边、保存还会抹掉（踩过） || `AgentTool` | `tool` 下拉（来自 `GET /api/pygraphs/agent-tools`）+ 按该工具表现算的参数端口 | 宿主执行器 `tool_call` → **与聊天同一个分发器**（审批 / `fs_guard` / 规则引擎 / 痕迹） | 每个 agent 工具 = 画布上一个带 UI 的节点；它**不自己调工具** |
| `Gate` | `guard`（**门名下拉**，来自 `GET /api/pygraphs/guards`）+ 按该门 `params` 现算的参数端口 + `enabled`（`False`=直接放行）+ `state`/`after` | 宿主进程注册的**决策门**（`register_gate_guard`） | **具名决策门**：收尾/分支前让宿主判一句（`stall` 空手收工 / `fact` 先核实 / `verify` 写后未验证 / `confirm` 收尾确认 / `goal` 目标预算）。出口 `allow`=**放行** / `triggered`=**拦下**（同一答案的两面，接 `And` 链不用 `Not`）。阈值/文案（`max_hits`/`nudge`）落在**节点上** ⇒ "改图即改门"（agent 那张资产里这些值由**入参节点**连线喂进来 —— 2026-10-16 用户口径「先把哪些参数添加为入参，然后去作为小节点附着到需要这个参数的节点，**所有参数必须在入参中定义**」之后，那张图的参数卡已删、换成一堆 `Input` + **端口小节点**（见 §3.0f）；⚠ 门读的是 resolved inputs，**读不到图接口**，所以只能连线） |
| `AgentParams` | `params`（**行表** JSON，`hidden` 容器；浮层里增删行）+ `port` 行各带一个同名输入端口 | 引擎自己（`agent_node.AgentParams.execute`）+ `param_spec.py` | **参数卡**：一张卡 N 个参数（可无限加），每行**三态取值**（见下面 §3.0b）；行表推端口（每行一个输出、`port` 行多一个输入）；`required` = 必传，否则用行里的 `value` 当默认值 |

⚠ 这些都不是"引擎自己跑"：`HostStep`/`AgentTool`/`Gate` 没注册就**明确报错**（`Gate` 不静默放行）；
提示词槽（那四行入参）在跑图时什么都不做 —— 真正"注入"由 agent 读图后在**发请求之前**做
（见本仓 `AGENTS.md` 与 `l_agent_chat` 的 CHANGELOG）。所以别指望"把某个 `GetParam` 连进 `plan` 就生效"
——那只做排序/可读性。

⚠ **两个 `guard` 别混**：本表的 `Gate` 是**图上的决策门**（`l_node_engine/gate_nodes.py` 的
`register_gate_guard`，对齐 `l_agent_chat/flow_engine.py` 的 `register_guard`）；而
`l_node_engine.guard.set_guard` 是**节点执行前拦截钩子**（抛异常 = 该节点不跑，属引擎安全闸）。
同名不同物，互不可替代。

### 3.0c ComfyUI 节点适配（2026-10-14，用户口径「我们能够兼容 ComfyUI 的节点」）

**已经同源的**（本引擎就是从 `l_comfyui_backend` → ComfyUI 一路 fork 的，见 `node_base.py` 顶注）：
节点元数据 = ComfyUI `/object_info`；图格式 = ComfyUI prompt（`{id: {class_type, inputs}}`，
连线 `[源id, 下标]`）；`INPUT_TYPES {required, optional}`、`FUNCTION`、`RETURN_TYPES`、
`RETURN_NAMES`、`OUTPUT_NODE`、`CATEGORY`、`DESCRIPTION`、`DEPRECATED` 全认；
输出节点 `{"result", "ui"}` 同形；端口类型词表同一套（另有 `tool_spec.comfy_port_type()` 双向映射）。

**唯一的硬坎**：`registry.register_node` 要求 `issubclass(cls, BaseNode)`，而上游节点**不继承我们的
基类**（形状一样、基类不同）⇒ 直接注册不了。适配层在 **`comfy_compat.py`**：

| 能力 | 说明 |
|---|---|
| `is_node_like(cls)` | **鸭子类型**判据：有 `INPUT_TYPES` + `RETURN_TYPES` + `FUNCTION` 才算节点（不看基类） |
| `adapt_node_class(cls)` | 外资类 → `BaseNode` 子类：元数据**原样透出**（编辑器画出来与上游一致）、执行时**实例化上游类**并照它的 `FUNCTION` 调；`OUTPUT_NODE` 那种只回 `{"ui": …}` 的**补空 `result`**（否则下游会把 dict 当值） |
| `load_comfy_nodes(targets)` | 目标可以是**目录**也可以是**单个 `.py` 文件**；返回 `{loaded, skipped, errors}`，**不抛**。三处刻意保守：⚠ **撞名一律跳过**（本地实现优先 —— 上游包里叫 `Preview`/`Output` 的会把我们整套图顶掉）；⚠ 目标是**上游 ComfyUI 根目录**时**只取 `nodes.py`**（同层的 `main.py` 顶层就起服务器、`server.py`/`folder_paths.py` 是内核 —— 逐个 exec 等于起一份 ComfyUI）；⚠ **`package.py` 一律不 exec**（那是 rez 包的构建脚本，`early`/`this` 只在 rez 解析时存在，2026-10-15 踩过）。显示名取上游 `NODE_DISPLAY_NAME_MAPPINGS` |

**从上游货架真加载（2026-10-15，用户口径「从 `l_comfyui_ai` 加载执行节点」）**：

光有 `load_comfy_nodes` 还不够 —— `nodes.py` **顶层**就 `import torch` / `import comfy.sd` /
`import folder_paths`，宿主里没有这一整套，import 第一条就死，连**声明面**（节点名/端口/分类）都看不到。
`comfy_lite.py` 因此补两样（`prepare_comfy_env(root, lite=…)`）：

| 补什么 | 说明 |
|---|---|
| `sys.path` | 上游根（含 `comfy/`、`comfy_api/`、平铺的 `folder_paths.py`）进 `sys.path`；传 `l_comfyui` 的**包根**也行（自动补 `src/l_comfyui/ComfyUI`） |
| **假模块** | 上游要、**宿主不用**的重依赖换成"万能假对象"。`LITE_FAKE_PACKAGES` = torch 生态 8 个 + `comfy_aimdo/kitchen/angle` + `safetensors/einops/sympy/networkx/fsspec/blake3/av/scipy/transformers/tokenizers/sentencepiece/coloredlogs/ansi2html/tqdm` + `yaml/packaging`（后两个单列：宿主**将来**可能真用，那时该改去真装）。**只假"真没装"的**（`force=False`），默认关（进程级 `sys.modules` 是全局副作用） |

假对象上踩出来的四件事（都写在 `comfy_lite.py` 里，**别删**）：

| 坑 | 症状 → 修法 |
|---|---|
| `str()` 不能是 `<Lite …>` | 上游 `tuple(map(int, str(torch.version.cuda).split('.')))`（`comfy/quant_ops.py`）⇒ `ValueError: invalid literal for int()`。改：**`str()` 给 `"0"`**（数值化安全），完整名字留在 `repr()` |
| `torch.cuda` 必须是**真值** | `_, total = torch.cuda.mem_get_info(dev)`（`comfy/model_management.py`）解包 ⇒ `ValueError: not enough values to unpack`。改：`_FakeCUDA`（`mem_get_info`→`(0,0)`、`memory_stats`→"取什么都有的 dict"） |
| 假对象**当基类** | 没有 `__mro_entries__` ⇒ `TypeError: __mro_entries__ must return a tuple`；给 `object` 又撞 MRO（`comfy/ops.py` 的 `Linear(torch.nn.Linear, CastWeightBiasOp)`）。改：**每次新造一个** `_LiteFakeBase`（与任何同级基类都不同源 ⇒ 怎么排都排得出来） |
| 上游有的节点**不在这套判据里** | 近两年的 `comfy_extras/*` 几乎都是 **V3 语法**（`io.ComfyNode` + `define_schema`，节点表由异步 `comfy_entrypoint()` 产出）⇒ 只会落进 `skipped`（"形状不像节点"）。要支持得单独翻译那一套 |

实测（本机 `l_comfyui_ai/999.0`）：`nodes.py` **65 个节点**全部加载、零撞名、零错误，节点库 60 → **125**；
`CLIPTextEncode` 的显示名/端口/分类/`source=comfy` 都对；`EmptyLatentImage` 在假 torch 下能跑到上游代码
（值无意义）。**真出图仍要真依赖**（走 `comfyui` / `comfyui_lite` 那个包）。

宿主接线（四个环境变量，首次取 object_info 时生效，摘要进 `l_node_engine` 日志）。
⚠ **2026-10-15 起优先走编辑器右栏的「🔌 插件」面板**（`plugins.json` + `GET|PUT /api/pygraphs/plugins`，
见 §3.0d）——下面这四个 env 仍在，但只作**面板没填的字段**的回落：

| 变量 | 含义 |
|---|---|
| `L_MINDMAP_COMFY_ROOT` | 上游 ComfyUI 根（或 `l_comfyui` 包根）。**给了它才做环境准备** |
| `L_MINDMAP_COMFY_NODES` | 要加载的目录/`.py` 文件（分号/逗号分隔）。**留空 = 就用 `ROOT`**（只取 `nodes.py`） |
| `L_MINDMAP_COMFY_LITE` | `1` = 注入假模块（默认关） |
| `L_MINDMAP_COMFY_FAKE` | 额外要假掉的包（上游常年在补依赖，用它临时放行） |

⚠ 加载 = **跑那些模块的顶层代码**（与 `PythonCode` 同权限）⇒ 只有管理员**显式**给路径才读（引擎不自己扫任何默认目录）。

自证：`wuwor l_comfyui_ai -- l_node_engine_comfy_surface`
（或 `--root <包根>`，见 `tools/comfy_surface_check.py`；`--trace` 单独加载 `nodes.py` 并打完整栈、
`--fake a,b` 临时放行、`--extra <文件/通配>` 额外点 V3 模块 —— V3 那几条判据（`source=comfy_v3` /
`ComfyNotNode` 真跑 `True→False`）靠它）；离线那半在 `l_node_engine_selftest` 第 34 组末尾
（假模块 / 上游根 / **V3**：异步 entrypoint、schema→元数据、真跑、hidden 归一）。

**已补上（2026-10-14 同轮）**：

| 能力 | 落点 |
|---|---|
| `INPUT_TYPES["hidden"]` 档 | `get_node_info()` 照上游原样发 `input.hidden`（编辑器**不画**这一档 —— 那不是控件）；**执行器注入** 见 `graph._hidden_inputs`：`hidden` 是 `{参数名: 注入键}`，`UNIQUE_ID`→节点 id、`PROMPT`→整张图、`EXTRA_PNGINFO`→`{}`、`DYNPROMPT`/未知→`None`。⚠ **只注入类声明过的那几个**（本引擎自己的节点签名是严的，无脑塞会让整张图崩），且**图里给了同名输入就以图为准** |
| `VALIDATE_INPUTS` | `graph._validate_inputs`：调之前按签名过滤参数（带 `**kwargs` 就给全部）；**只有 `True` 算过**，字符串 = 给用户看的错误原文，其余 = 失败（按软失败记进 `node_errors`）。适配层会**转发**它（否则适配之后被吃掉 → 静默少一层校验） |
| `hidden` 也算"声明过的输入" | `validate.py`：导入来的图里带 `unique_id` 这类字面值，不再报"未声明的输入键" |

**还没做（不认的钩子会被静默忽略，移植时容易"看着能用、其实丢东西"）**：

| 缺 | 后果 |
|---|---|
| `IS_CHANGED` / `check_lazy_status` | 细粒度失效 / 懒算表达不了（本引擎用结构指纹缓存） |
| list 自动展开（`INPUT_IS_LIST` / `OUTPUT_IS_LIST`） | 批量语义不一致（得图内自己 `LoopStart/End`） |
| V3 的 `enable_expand`（子图展开）/ autogrow 动态输入 | 动态拓扑表达不了（本引擎的图是静态结构 + 指纹缓存）：`NodeOutput.expand` 会被忽略，autogrow 那几行只按静态那几路走 |

**V3 节点现已支持（2026-10-15 同一轮补的）** —— 上游近两年的 `comfy_extras/*` 几乎全是这套写法：

| 上游 | 我们 |
|---|---|
| 节点表 = **异步** `comfy_entrypoint()` → `ComfyExtension.get_node_list()`（**不在** `NODE_CLASS_MAPPINGS` 里） | `v3_nodes_of_module()`：照上游 `nodes.py::init_extra_nodes` 那几步走（`on_load()` 也要 await），`node_id` / `display_name` 取 `GET_SCHEMA()` |
| 元数据 = `io.Schema`（`define_schema()` 返回的东西） | **不重写映射**：`adapt_v3_node_class()` 直接取上游自己算好的 `GET_NODE_INFO_V1()`（`comfy_api/latest/_io.py`）当我们的 `INPUT_TYPES` / `RETURN_TYPES` / `RETURN_IS_LIST` / `CATEGORY` / `OUTPUT_NODE` —— 上游加新控件或新 io 类型，我们跟着变，不会漂"两份实现" |
| 执行 = `FUNCTION`（`EXECUTE_NORMALIZED` / `_ASYNC`）+ `PREPARE_CLASS_CLONE(v3_data)` | 照抄：**克隆类**（挂 `hidden` 持有者）再调它；返回的 `NodeOutput` → 有 `ui` 才给 `{"result","ui"}` 壳，没有就交普通元组（与 V1 那条路同形） |
| `hidden` 在 V1 元数据里是"**元组包着枚举**"（`(Hidden.PROMPT,)`） | 归一成 `"PROMPT"` 才算我们的 hidden 表；执行时按**注入键**（不是声明名）塞进 `v3_data["hidden_inputs"]`（`HiddenHolder.from_dict` 就是按 `Hidden` 那些**值**取的） |
| `NODE_CLASS_MAPPINGS` 与 `comfy_entrypoint` 同时存在 | **V1 优先、V3 兜底**（与上游 `elif` 同序） |

实测（本机 `l_comfyui_ai/999.0`，lite）：`comfy_extras/nodes_logic.py` + `nodes_string.py` + `nodes_primitive.py` +
`nodes_number_convert.py` + `nodes_math.py`(缺 `simpleeval`) + `nodes_replacements.py`(缺 `requests`) 一起点 →
**26 个 V3 节点进库**、2 条缺依赖错误（一条条列在 `errors` 里，不是静默）；`ComfyNotNode` 的
`out=['BOOLEAN'] / cat=utilities/logic / source=comfy_v3` 都对，**真跑** `True → False`（这一步**没有假依赖掺和** ——
`nodes_logic` 不碰 torch，是"V3 通了"的硬判据）。

宿主侧：`L_MINDMAP_COMFY_NODES` 现在**支持通配**（`<root>/comfy_extras/nodes_*.py`），
且逗号/分号分隔的文件目标与目录目标可以混着给。⚠ **默认仍然只收 `nodes.py`** —— `comfy_extras` 上百个文件、
不少要可选重依赖（mediapipe / moderngl / 各家 SDK），一次全进来会拖慢启动、把 `object_info` 撑大、
刷一屏"这个包没装"；要哪些就点名（缺依赖那几条会出现在日志的 `errors` 里）。

自证：`l_node_engine_selftest` 第 34 组（鸭子类型判据 / 元数据透出 / 真跑 / `OUTPUT_NODE` 补 result /
目录加载的"撞名跳过 + 坏包只记一笔" / **`hidden` 注入**（含"图里给了就以图为准"）/
**`VALIDATE_INPUTS`**（过 / 字符串错误 / 没产出值）/ **`Reroute`**（串两个还是原值））；
宿主那条接线另跑过真 env（`registered=True`、`run=(12,)`）。

**上游那 4 个"前端侧节点"的对照**（`l_comfyui_frontend/src/extensions/core/`，都是 `LiteGraph.registerNodeType`
的前端节点，**不参与服务端执行** —— 上游 `ComfyUI/nodes.py` 里搜不到它们）：

| 上游前端节点 | 我们这边 |
|---|---|
| `Reroute`（转接点） | **本轮补了**（`pygraph.Reroute`）—— 但**做成真节点**（上游是提交前被剪掉的前端节点；我们的图是资产，要能脱离编辑器跑）。判据：串两个转接点还是原值 |
| `Note` / `MarkdownNote` | 已有 `Note`（Markdown 卡片） |
| `PrimitiveNode`（把 widget 变成输入端口） | 语义已有：输入端口的 `forceInput` / `container: link`（"这一行只能连线"） |
| `widgetInputs`（同一件事的另一半） | 同上（`PyNode` 的"端口 vs 控件"判定） |

### 3.0d 插件面板（`plugins.py` + `comfy_plugin.py`，2026-10-15）

用户口径：「**给 pygraph 做一个插件面板，ComfyUI 节点的支持作为插件功能**」。所以 §3.0c 那套
从"环境变量"搬到**面板**上：一个插件 = 一格**可开关、可配置、可看状态**的能力，`comfy` 是第一个。

| 落点 | 是什么 |
|---|---|
| `l_node_engine/plugins.py` | **插件注册表 + 落盘**：`Plugin`（`fields()` / `status()` / `apply()` / `sources()`）、`describe()`（面板视图）、`config_path()`（`<包数据目录>/plugins.json`，`L_NODE_PLUGINS_CONFIG_DIR` 可覆盖）、`bootstrap()`（进程起来时把**已启用**的跑一遍） |
| `l_node_engine/comfy_plugin.py` | **「ComfyUI 节点」这个插件**：`apply()` = `prepare_comfy_env()` + `load_comfy_nodes()`，§3.0c 那四个环境变量成了四个字段 |
| 宿主 `l_mindmap_mmd` | `GET\|PUT /api/pygraphs/plugins[/{id}]` + `POST …/{id}/reload`（**写要执行面门槛**）；`_node_engine()` 改成 `bootstrap_plugins()` |
| 前端 `PluginPanel.jsx` | **不认识任何具体插件**：字段表与状态都从接口拿（加插件不用动前端）；布局树里是右栏标签「🔌 插件」 |

**通用性靠三份契约**（加插件只用这三样，别发明第四样）：

1. **字段类型词表**：`bool` / `text` / `path` / `list`（`plugins.FIELD_TYPES` 是唯一来源，前端按它渲染四种控件）；
2. **面板视图**：`{id, name, description, doc, runs_code, enabled, config, fields, defaults, status}`（`describe()`）；
3. **状态**：一个 dict，键随插件（面板按"标量 / 列表 / 错误"三种画法通用渲染，不认得的键也照画）。

**默认关**（`config_of()` 缺省 `enabled=False`）：启用 = 跑那个插件的顶层代码（与 `PythonCode` 同权限），
必须是显式选择。写入与重载都过 `L_MINDMAP_ALLOW_RUN` 门槛；`GET` 回一个 `allow_edit`，面板**事先**
就说清"只能看状态"，而不是让人点半天再吃 403。

**配置从哪来**：`plugins.json` **字段级覆盖**环境变量（面板没给的项才回落 `L_MINDMAP_COMFY_*`）——
env 是"机器级老通道"（卡片带不了环境变量），面板是"人的临时决定"；输入框的坑位显示的**就是 env 那份值**。
`GET` 的 `status.where` 直说现在吃的是哪一份（面板 / 环境变量 / 两者）。

**重载为什么必须先摘**：`load_comfy_nodes` 撞名即跳过（本地实现优先）⇒ 不先 `unregister_source`
把上一轮摘掉，第二次加载**一个都不进**（表现就是"点了重载没反应"）。`Plugin.sources()` 就干这个。

自证：`l_node_engine_selftest` 第 36 组（默认关 / 坏 JSON 当没配过 / 面板视图只用类型词表 /
启用→真加载 / **重载幂等** / 停用→摘净 / `bootstrap` 只跑已启用的）。
真服务那半（2026-10-15 实测，`l_mindmap_mmd_server`）：

```
GET  /api/pygraphs/plugins                      → comfy + 4 个字段 + allow_edit:true
PUT  /api/pygraphs/plugins/comfy {"enabled":true,
     "config":{"root":"<l_comfyui_ai>/999.0",
               "nodes":["…/comfy_extras/nodes_logic.py"], "lite":true}}
     → detail="加载 5 个节点" · status.where="面板" · root 归一化成 …/ComfyUI · 27 个假模块 · errors:[]
GET  /api/pygraphs/object_info                  → 出现 ComfyNotNode / CustomCombo（节点库当场变了）
PUT  /api/pygraphs/plugins/comfy {"enabled":false}
     → loaded:0 且 object_info 里那两个类消失、我们自己的节点一个没动
```

前端另有探针场景 `plugins`（15/15：接口面 / 面板把声明的字段一个不落画出来 / 状态表 / 开关 /
两张卡都被通用渲染画出来），判据表见 `l_mindmap_mmd/999.0/tools/web_probe.md`。

### 3.0d′ 「函数节点」插件（`functions_plugin.py`，2026-10-17）

用户口径（先问"现在所有 python 对象都能创建为节点了对么"，再问"建议接上么"→「接（插件方式，默认关）」）：
把**一个包的函数批量接成一等节点**（`from_functions.register_package_functions` 的插件壳）。

| 位 | 内容 |
|---|---|
| 插件 id | `functions`（面板名「函数节点」，`runs_code=True`） |
| 字段 | `packages`（list，一行一个包名）/ `category`（text，默认 `func`）/ `include`（list，只挂哪些子包，空=全包） |
| `apply()` | 先 `unregister_source("function")` 摘上一轮 → 逐包 `register_package_functions(pkg, category=…, include=…)` |
| 状态 | `at` / `packages` / `loaded`(节点数) / `modules` / `skipped` / `errors`（按包带原文） |
| 摘干净 | `sources()` = `["function"]`（`node_from_function` 给节点打的就是这个 `NODE_SOURCE`，面板据此标 `ƒ`） |

**三条边界**（不是"任意包全自动变节点"）：只扫**点名的包**；`_` 私有名 / **零参函数** / 读不出签名的
内置跳过（`iter_functions` 老口径）；同名函数跨模块退成 `<模块名>.<函数名>`。
⚠ `sources()` 按来源摘，所以停用会摘掉**所有** `function` 来源的节点 —— 目前只有本插件注册这种节点。

**接的时候撞出的两个真 bug（都修了，记着别再犯）**：
1. **插件启用时没同步导入白名单**：`apply_guard_setting()` 原来只挂在跑图/问端口/取元数据三处，
   插件 `apply` 走的 PUT 端点没调 ⇒ 即使设置里 `allowAllPython: true`，插件里每个包都报
   "不在导入白名单里"、**零节点**（2026-10-17 实测：`functions` 配 `json` 时 `loaded:0, skipped:5`）。
   修法：`_node_engine()` 的 bootstrap 前、以及 PUT/reload 两个端点里都先 `apply_guard_setting()`。
2. **`make_node_from_callable` 自己摊了一份端口表**（跟 `introspect.params_from_callable` 漂了）⇒
   漏了 ① 跳过 `_` 私有参数、② 默认值过 `safe_default`。`json.decoder.py_scanstring` / `JSONObject`
   的私有默认值是 `re.Pattern.match`（内建方法）⇒ 原样塞进端口表让**整张 `/object_info` 500**
   （浏览器那头表现是"节点库全空"）。修法：`make_node_from_callable` 改成直接调
   `params_from_callable`（全包唯一实现，顺带白捡 `std_type`）。

自证：`l_node_engine_selftest` 的"插件 functions"组（临时包 → 启用：端口按注解 / `source=function` /
私有与零参跳过 / **不可序列化默认值退 repr** / **节点真能跑**（`add(a=3)`→5）；停用 → 一个不剩）。
真服务（2026-10-17 实测 `l_mindmap_mmd_server`）：`PUT …/plugins/functions` + `json` 包 →
`1 个包 → 11 个节点（4 个模块）`、`object_info` 里 `category:"fnprobe"` 出现且 HTTP **200**；
停用 → `已停用（摘掉 11 个节点）`、`object_info` 里 `fnprobe` 消失。

### 3.0d″ 「3D 程序化建模」插件（`mesh3d_plugin.py` + `mesh3d.py`，2026-10-20）

> ⚠ **2026-10-21 起这些代码不在本包了**：3D 建模拆成独立 rez 包 **`l_mesh_nodes`**（插件形态 B：
> 宿主 `requires` 不含它，`plugins.load_external()` 按 `external_plugins.json` 解析 rez 路径再 import）。
> 下面这一节保留为**机制/口径的权威说明**；包结构、接入与自证见 `Rez-Docs/Rez_pkg/l_mesh_nodes.md`。

用户口径：「给 pygraph 加一个插件，可以做基础的 3d 程序化模型制作」→ 追加「可以参考开源库，**要高性能**」。
于是几何算法单独成 `mesh3d.py`（**numpy 向量化**，算法照 trimesh / Blender 那套：`np.add.at` 面法线累加、
`np.unique(axis=0)` 边中点去重、`einsum` 算有符号体积），节点壳在 `mesh3d_plugin.py`。

| 位 | 内容 |
|---|---|
| 插件 id | `mesh3d`（面板名「3D 程序化建模」，`runs_code=False` —— 只挂本包自己的纯函数） |
| 字段 | `category`（text，默认 `mesh3d`）/ `max_vertices`（text，默认 `200000`，**0 = 不限**）/ `save_dir`（path，空 = `<图目录>/models`）/ **`deps`（list，可选依赖目录：`manifold3d`、以及 torch / cupy）** / **`gpu`（`off`/`auto`/`on`）+ `gpu_backend`（`auto`/`torch`/`cupy`）+ `gpu_min`（元素数门槛，默认 `200000`）+ `numba`（`auto`/`off`/`on`，`remesh` 栅格化的 `@njit`）** |
| CPU JIT（2026-10-21） | `mesh3d_njit.py`：**`taubin`（numba → GPU → numpy 的第一顺位）** + `remesh` 的**逐三角形栅格化**。实测（RTX 4080 同机）：`taubin` 32.5k 顶点 numpy 175 → **numba 4.5 ms（39×）**、torch 40 ms（4.3×）；栅格化 48³ 658→**2 ms（426×）**、64³ 815→**1 ms（560×）**，整个 `remesh` 64³ 608→**328 ms（1.85×）**；**泛洪不用 numba**（numpy 切片 128³ 7.5 ms，numba 78 ms）。用不了 JIT 回 numpy 版（逐位一致），`detail.remesh.jit` 记这一轮走没走 |
| GPU（可选，2026-10-21 已实测） | 依赖挂在插件层（「依赖目录」填带 torch 的目录，本机 = comfy 那份 `torch 2.8.0+cu128`）→ `mesh3d_gpu.backend_for(元素数)` **torch 优先**、其次 cupy、都没有回 numpy；两个后端都带**冒烟自检**（import 成功≠能算），失败永久回落并在 `status().gpu` 写原因。**走 GPU 的只有 `taubin` 与 `remesh` 的泛洪/膨胀**；实测（RTX 4080）`taubin` 65k 顶点 462→96 ms（**4.8×**）、`remesh` 96³ 7.97→7.91 s（**1.01×**，瓶颈是逐三角形栅格化的 python 循环）。cupy 在无 CUDA toolkit 的机器上不可用（`Failed to find CUDA headers`，装 `[ctk]`+nvcc 也未解决）⇒ 用 torch。⚠ 插件跑在宿主进程里，后端加载即常驻 |
| 状态 | `loaded` / `nodes[]` / `max_vertices` / `save_dir` / `deps[]` / `errors[]` / **`numpy`**（版本）/ **`boolean`**（manifold3d 在不在） |
| 摘干净 | `sources()` = `["mesh3d"]`（48 个节点都带这个 `NODE_SOURCE`） |

**属性（Houdini 口径，2026-10-20 重构；值格式 `mesh3d/2`）**：

用户口径：「houdini 并没有每个属性一个端点，有一个属性面板可以查询所有属性，流动的是上游传下来的属性，
不一定要全部端点都连接」→「重构，不兼容老图」。**属性放值里、不占端口**：

    {"kind":"mesh","format":"mesh3d/2","vertices":[…],"faces":[…],        # vertices 就是 P
     "attrs":{"point":  {"Cd":[[r,g,b],…],"wear":[0.1,…]},               # 逐顶点（长度 = 顶点数）
              "vertex": {"uv":[[u,v],…]},                                # **面角**（长度 = 3×面数，三角面）
              "prim":   {"mat":[0,1,…]},                                 # 逐面（长度 = 面数）
              "detail": {"seed":3}}}                                     # 整体常量（一个值）

**四层口径**（Houdini 那套；用户口径 2026-10-21「点、面、顶点、detail 层级都应该有属性」）：
`point` = 顶点、`vertex` = **面角**（每个面的每个角，三角面 ⇒ 3×面数）、`prim` = 面、`detail` = 整体。
`const` 是 `detail` 的旧名（当别名认）。`AttrPromote` 四层之间可任意升：`point→vertex`（按面角复制）、
`vertex→prim`（每面聚三个角）、`vertex→point`（每个点聚相邻角）、`→detail`（整体聚合）、
`detail→任意`（广播）。

| 件 | 口径 |
|---|---|
| **长度必须对得上** | `point` = 顶点数、`vertex` = 3×面数、`prim` = 面数、`detail` = 1；对不上**直接报错**（不静默截断 —— 那是最难查的一类） |
| **搬运规则** | 老顶点照搬、细分/CC 的新点取平均、子面继承父面、拼接按顺序拼（缺的补零）、焊接按组平均；**`vertex` 类**不改拓扑的算子原样带、改拓扑的（细分/CC）从**同名 point 属性**按新面角重建、没有就丢（其余算子目前也丢，要带就 `AttrPromote` 再升一次） |
| **已搬运的算子** | `transform` / `mirror` / `array` / `join` / `MeshGroup` / `subdivide` / `MeshSubdivideSurface` / `weld` / `delete_faces` / `smooth` / `displace` / `orient_*` / **`extrude`**（顶面继承、侧墙零值）/ **`inset_faces`**（内圈取面均、边框继承父面）/ **`fill_holes`**（中心取环均）/ **`bridge`**（两份拼接、桥面零值）/ **`boolean`**（**走 manifold3d 的 `vert_properties` 第 4 列起**，CSG 会一起插值） |
| **目前不带的** | `MeshSweep` / `CurvePath` —— 它们**新造**几何（曲线上没有属性可继承）；要带就在 `CurvePath` 上加属性（下一步） |
| **几何表（属性面板）** | 两个入口，同一份数据（宿主内存 `_MESH_SRC`，重启后要重跑）：<br>①**底部「🧊 3D 视图」头部 `🧬 属性 N`** → 浮层汇总表 `类 / 名字 / 类型 / 条数 / min / max / mean`（`GET /api/pygraphs/attrs`），点任一行 → 逐行明细（`GET /api/pygraphs/attr-rows`，一页 100、`◀ ▶`、按值排序）；<br>②**独立标签页「🧬 几何表」（`geoattrs`，2026-10-22 用户口径「我要独立的属性扩展表单面板，可以查看点线面的属性，最基本的是 P 属性」）**：`GeoAttrsPanel.jsx`，域页签 **点 / 线（面角）/ 面 / 整体**（各自行数），整张表 `# + P + 该域所有属性`，分页 100 行、**点数值列表头**升→降→取消，数据走 `GET /api/pygraphs/geo-table?name=&node=&cls=&offset=&limit=&sort=&by=`。<br>**`P` 不在 `attrs` 里，按域现算**（`server._geo_domain_arrays`）：`point` = 顶点坐标、`vertex` = 面角坐标、`prim` = 面重心、`detail` 没有 —— `/attrs` 与 `/attr-rows` 也一起认 `P`；**`detail` 是"整张几何一份"**：`geo-table&cls=detail` 固定 **1 行**、每列就是那个常量（`AttrSet(cls="detail")` 写进来的），所以别拿它当"每元素一行"去取 `[i]`。就是 Houdini Geometry Spreadsheet 的对应物；**另外两口（2026-10-22）**：① **点一行 → 3D 视口里标出来**（那一行的 `P`：点 = 顶点 / 线 = 面角 / 面 = 重心；`MeshViewer` 在模型局部坐标里画红球 + 三根十字线，判据 = `canvas.dataset.hl`；再点同一行取消；**视图看的节点要和表里选的是同一个**才画）；② **`⤓ CSV`** 导出整张表（`geo-table?…&fmt=csv`，向量写成 `x y z`）。还没有的是"明细里直接改值" |

**绑定（骨架 / 蒙皮 / 姿势；2026-10-21 用户口径「添加绑定，让狮子动起来」）**：

| 件 | 口径 |
|---|---|
| `MeshRig` | `RIG` 端口的值 = `{kind:"rig", joints:[{name,parent,pos}]}`；文本每行 `名字, 父关节, x, y, z`（**绝对坐标**，父留空/`-`/`none` = 根；**父必须写在子前**，否则报错点名）。重名/未知父都直接报错 |
| `MeshSkin` | 每根骨段（父位置 → 自己位置）算点到线段的距离，权重 `1/(d+1e-4)^falloff`；每点只留 `max_bones` 根（默认 2）后**归一化**，写 `attrs.point['W:<关节名>']`。零配置，但和手工权重没法比 |
| `MeshPose` | LBS：`世界旋转[子] = 世界旋转[父] @ 局部旋转`，`M_i = T(P_i)·R_i·T(−P_i)`，`p' = Σ w_i·M_i·p`；`blend` 在静止与姿势之间插值 |
| **角度可以写 `t`** | `rotations` 里每一项是 `关节:rx,ry,rz`，三个分量都能写表达式 —— 白名单只有数字 / `t` / 四则 / 括号 / `sin cos tan abs min max sqrt floor round pi tau`，`eval` 时 `__builtins__` 为空。例：`tail2:0,sin(t*3)*26,0; head:sin(t*1.5)*6,0,0` |
| **怎么动起来** | `t` 接 `Time` 的 `time` → **时间线 ▶ 播放**时每帧重跑图，3D 视图逐帧更新（「⚡实时」默认关；拖动帧也会重跑）。实测：连跑两次，节点 102 的几何**逐次不同** |

⚠ 三个坑：① 关节坐标是**绝对**的（不是相对父的偏移），摆错一位整只狮子会散；② `pose` 没接 `MeshSkin`（没有 `W:` 权重）会明确报错；③ 权重用的是**到骨段的距离**，细长件（尾巴）建议把关节排密一点，否则一节会带不动下一节。

**录制时间线动画（2026-10-21 用户口径「添加录制时间线动画的功能」）——两条路**：

| 路子 | 怎么做 | 状态 |
|---|---|---|
| **面板录制**（`MeshViewer`） | 3D 视图头部多了 **`⏺ 录制`** 按钮：`canvas.captureStream(30)` + 浏览器自带 `MediaRecorder`（**不引第三方库**）→ 停止时自动下载 `.webm`；停止时把字节数写画布的 `data-rec-size`（探针用）。用法：点录制 → 时间线 ▶（或拖帧）/ 点画布 ▶ 跑一轮 → 点「⏹ 停止」 | **已实测**（2026-10-21）：录了 54s → 下载 `pygraph-_102-….webm`（**2.28 MB**，EBML 魔数 `1A 45 DF A3` 对），留在仓库根 `lion_record.webm` |
| **离线烤帧**（`tools/record_timeline.py`） | 按帧改一个输入（默认 `t`）→ 跑整张图 → 取某节点几何 → z-buffer 光栅 → GIF/WebP。不依赖 UI、可批处理。用法：`wuwor l_mindmap_mmd -- python tools/record_timeline.py --file <图.json> --node 102 --param t --frames 16 --period 2.09 --out lion.gif`（`--decimate` 降面预览、`--size` 尺寸、**`--static` = 不改任何输入只出一张**（图里没有 `t` 时看"这节点现在长啥样"）、**`--mat <槽:材质节点>` = 把它的 basecolor 按顶点 uv 采到顶点色上**（`--mat 309` = 整只；`--mat "0:309, 1:325"` = 按材质槽各贴各的，对应宿主 `3d_view.material` + `slot_mat`；给了它就不降面，因为降面会丢 uv/顶点色；**只换写了的那几个槽的顶点色**，其余槽保持纯色）） | **已验证**：`demo_lion` 16 帧 640×480 130ms/帧 → `lion_anim.gif`；`demo_lion4` `--static` 单帧 720×540 → 融球狮子外观复核；`demo_lion5 --mat 309` → 带毛皮贴图的静帧 |

⚠ 两个探针/排障的坑（2026-10-21 实测踩过）：
1. **`canvas` 个数 = 0 不代表编辑器坏了** —— ≤150 节点走 DOM/ReactFlow 后端（**SVG**），canvas 只在 >150 节点时才有。查"编辑器有没有渲染"看 `.react-flow` + `.react-flow__node` 的个数（本次 104 个 ✓）。
2. **3D 视图标签可能"被关掉"**：布局里 `layout.closed` 会记墓碑（右键删除标签页留下的），此时 `.pyg-tab[data-key=viewer3d]` 不在 DOM 里。用该面板标题栏的 **「＋」**（`.pyg-tab-plus`）→ 菜单里点「🧊 3D 视图」叫回来（本次存储的 `closed` 里就带着 `viewer3d`）。
| **非数值属性** | 不改拓扑的算子照搬；改拓扑的**丢弃**（字符串没法插值，绝不静默变成 0） |
| `AttrSet` | 写属性：`constant`（可三通道）/ `position`（包围盒归一化 → 拿来当 `Cd`）/ `noise` / `radial`，可只写选择集 |
| `AttrGet` | 按名字读：`values` 全量、`mean/min/max` 一个数；**读不到会列出这一类现有的属性名** |
| `AttrPromote` | `point → prim`（每面取三点聚合）/ `prim → point`（每点取相邻面聚合），`mean/min/max/first` |
| `MeshDecimate` | **顶点聚类减面**：按 `cell` 格子吸顶点、丢退化/重复面（`mean` 取质心 / `first` 取一个）。⚠ 不做二次误差度量，`cell` 要跟特征尺寸同量级（单位方块的 CC 结果：0.3 不动、**0.45 → 26v/48f**、0.6 → 22v/40f，0.9 会拍没——会报错） |
| `MeshSave(fmt=ply)` | **PLY 才存得下 `Cd`**（OBJ 没有顶点色）：头里 `property uchar red/green/blue`，0..1 或 0..255 都吃 |
| `MeshProjectUV` | `planar`（XZ：`u=x, v=z`）/ `cylindrical`（绕 Y：`u` 一圈、`v` 归一化高度），写 `attrs.point.uv`；⚠ **逐顶点** UV（整件投影，接缝处会拉伸） |
| **`MeshUnwrap`（2026-10-21，P1.2）** | **分岛 + 打包**：二面角 ≥ `angle` 的边当接缝 → 连通块分岛 → 每岛取**面积加权平均法线**做切平面投影（岛内不拉伸）→ 等比缩放 + 货架式排进 0..1（`margin` 留白）→ 写 `attrs.point.uv`，**沿缝分裂顶点**（`(顶点, 岛)` 去重）。岛数落 `detail.islands`、口径落 `detail.unwrap`。单位方块 `angle=60` → **6 岛**；圆柱 → 3 岛（侧面 + 两盖）；`angle=180` → 1 岛 |
| OBJ 导出带 UV/材质 | 有 `uv` → 写 `vt` 且面写 `f v/vt`；有 `Cd` → 顺手写 `<stem>.mtl`（`Kd` = `Cd` 均值）并在 OBJ 里 `mtllib`/`usemtl`；**有 `prim.mat` → 多材质**：`.mtl` 每槽一条 `newmtl mat<号>`（色 = 该槽面角均色，没 `Cd` 用调色板）、OBJ 逐段 `usemtl`；`MeshSave` 的 `texture` / `bump` 字段写进 `map_Kd` / `map_Bump`（**文件自己放同目录**，不搬） |
| **多材质槽（2026-10-21，P1.1）** | 口径：槽号 = `prim.mat` 整数，**用 `AttrSet(mat, prim)` 打标**（不需要额外的"材质清单"节点）。宿主 `mesh_preview` 把它翻成 `groups`（`[[start,count,slot],…]`，铺平索引的连续段）+ `mats`（每槽色），`MeshViewer` 用 `geometry.addGroup` + **材质数组**画 ⇒ 3D 视图里一眼分得清（`canvas.dataset.matslots` = 槽数，状态行也写「N 个材质槽」）。⚠ `attrs.prim.mat` 经 `subdivide` 等改拓扑的算子照样跟着走（每父面 4 个子面） |
| 生成几何贴图 | payload 带 `uv` → `MeshViewer` 把生成网格也交给"贴图那套"（`meshesRef`），`l_tex_nodes` 的材质通道就能贴到图里算出来的网格上 |
| 3D 视图 | 有 `attrs.point.Cd` 就**按顶点色画**（`mesh_preview._point_colors` → payload `colors` → `MeshViewer.vertexColors`；>1 的按最大值归一） |

自证（`mesh3d` 的「属性」两组）：`Cd` 长度/归一化、`stats` 对得上、读不存在的属性报错带清单、
长度不对报错；`transform` 原样、`mirror`/`array`/`join` 复制拼接、`subdivide` 中点=两端平均、
CC 条数 = 新顶点数、`weld` 接缝取平均、`delete` 跟着剩点走、面属性 ×4（细分）、非数值属性该丢的丢；
**挤出/内缩/封洞/桥接/布尔** 五个也都断言"属性条数 = 新几何的顶点数/面数"。
示例图 `demo_geo_attrs`（方块 → CC → `AttrSet(Cd, position)` + `AttrSet(wear, noise)` → `AttrGet(Cd, mean)`），
3D 视图按 `Cd` 上色、`🧬 属性 2` 那张表就是截图左下角（`Cd vec3 38` / `wear float 38`）。


**P0/P1 + 属性 + 绑定 + 雕刻/晶格 + UV 展开 + P3 建模工具 + 抄 three.js `examples/jsm` 那批 + 图元补遗/数据进出共 72 个节点**（命名一律 ASCII；中文只进说明 / 悬停 / 搜索别名）：

| 组 | 节点 |
|---|---|
| **P3 收尾（2026-10-21）** | **`CurveEdit`**（改/插/删控制点：`3 = x,y,z` / `+3 x,y,z` / `-1`）/ **`MeshScatter`**（按面积撒点，`align` normal/up/random + `tilt`/`jitter`/`scale_var`；**`bias`/`bias_amount`/`bias_falloff` = 方向偏置**（2026-10-23：毛往后倒，纯随机 tilt 是炸毛球）；⚠ v1 是真几何不是 instancer）/ **`MeshRemesh`**（体素重拓扑：栅格化 → 填实心 → 表面方块面 → Taubin 平滑 → 沿法线收回 1.2 格；**丢属性**、闭合无自交） |
| **P3 建模工具（2026-10-21）** | **`MeshBevel`**（边倒角：二面角 ≥ `angle` 的硬边或接 `MeshSelectEdge` 的边 → 面组退 `width` + 边缝补四边形 + 顶点洞补扇形 ⇒ 仍闭合）/ **`MeshLoft`**（多截面放样：`|` 分隔的若干**同点数**环 / 一串 CURVE）；**选择增强**：`MeshSelectEdge`（`angle`/`all`，输出带 `edges`）+ `MeshSelectGrow`（按相邻面扩散 / `shrink` 收一圈）；`MeshSculpt` 加 `symmetry=x/y/z`（镜像处同雕一笔）、`MeshSweep` 加 `roll` / `twist`（截面转角 / 累积扭转） |
| **雕刻 / 变形 / 折痕（2026-10-21）** | **`MeshCrease`**（二面角 > 阈值 → 标折痕，接 CC 保棱角）/ **`MeshSculpt`**（球刷：inflate/pinch/flatten/grab，中心+半径+衰减+强度，可只雕选择集；`symmetry` 对称雕）/ **`MeshLattice`**（晶格 FFD：`nx×ny×nz` 格 + 控制点偏移表，三线性插值） | 肌肉/眉骨/鼻梁靠雕，胸肩收腰靠晶格 |
| **绑定（2026-10-21）** | **`MeshRig`**（骨架 `RIG` 端口：`名字, 父关节, x, y, z`）/ **`MeshSkin`**（按到骨段距离算权重 → `point.W:<关节>`）/ **`MeshPose`**（LBS 摆姿势，角度可写 `t` 的表达式 ⇒ 动画） |
| **属性** | **`AttrSet`**（写：constant / position / noise / radial，可只写选择集）/ **`AttrGet`**（按名字读：全量或 mean/min/max）/ **`AttrPromote`**（point↔prim 升降，mean/min/max/first） |
| **UV / 减面 / 导出** | **`MeshProjectUV`**（planar / cylindrical 逐顶点 UV → `attrs.point.uv`）/ **`MeshUnwrap`**（分岛展开 + 打包，缝上分裂顶点 ⇒ 不拉伸）/ **`MeshDecimate`**（顶点聚类减面）/ `MeshSave`（`fmt=obj` 或 **`ply`（带 `Cd` 顶点色）**；`texture`/`bump` 写 MTL 的 `map_Kd`/`map_Bump`；有 `prim.mat` 就写多材质 MTL） |
| **抄 three.js `examples/jsm`（2026-10-23）** |**`MeshSimplify`**（QEM 二次误差边折叠减面，`ratio`/`target`/`passes`；区别于 `MeshDecimate` 的聚类）/ **`MeshConvexHull`**（QuickHull 三维凸包）/ **`MeshOBB`**（最小体积定向包围盒 + `align` 摆正；`detail.obb` 给三边）/ **`MeshCurveDeform`**（沿曲线弯曲整件，`twist` 累计扭）/ **`CurveNurbs`**（真 NURBS 曲线，`weights` = 有理）/ **`MeshNurbs`**（控制网格 → 全四边形 `ngons`）/ **`MeshTessellate`**（保形加密：只劈最长边、**原顶点不动**）/ **`MeshEdgeSplit`**（按二面角分平滑组 + 按组复制顶点 = `toCreasedNormals`）；另 `MeshDisplace.noise=simplex`、`MeshSave(fmt=glb).compress`（八面体法线）。口径与实测见 `l_mesh_nodes.md` §8.1f |
| **图元补遗 + 数据进出（2026-10-23）** | **`MeshCapsule`**（圆柱+两半球）/ **`MeshCircle`**（圆盘·扇形）/ **`MeshRing`**（内外半径之间的**四边形**环）/ **`MeshShape`**（任意简单多边形填充片，不挤出）/ **`MeshRoundedBox`**（SDF 圆角盒，一步出光滑圆角）/ **`MeshParametric`**（`u,v → x,y,z` 三条表达式 → 四边网格，沙箱 eval）/ **`MeshMetaball`**（融合球：四面体版 MarchingCubes 抽等值面）/ **`MeshImport`**（OBJ/PLY/STL/glb 读进图，带 uv·Cd·多材质槽；`MeshSave` 多一档 `fmt=stl`）。口径与实测见 `l_mesh_nodes.md` §8.1g |
| 图元 | `MeshBox` / `MeshSphere` / `MeshCylinder` / `MeshCone` / `MeshTorus` / `MeshPlane` / `MeshLathe`（轮廓绕 Y 转一圈） |
| 变换 | `MeshTransform`（缩放→旋转 X→Y→Z→平移）/ `MeshArray`（阵列：偏移 + 每份旋转/缩放）/ `MeshMirror` |
| 细节 | `MeshSubdivide`（线性细分，形状不变）/ `MeshSmooth`（Laplacian，可只磨选择集）/ `MeshDisplace`（**沿法线**：默认分形噪声；接 `image`（PNG 路径 / `{width,height,pixels}` / 二维表）就**按 `uv` 采样贴图置换**，`repeat` 平铺、中灰 0.5 = 不动；可只推选择集；没 `uv` 明确报错不退回噪声） |
| **曲线 / 扫掠（P1）** | **`CurvePath`**（点列 → Catmull-Rom 曲线，`CURVE` 端口）/ **`CurveEdit`**（改/插/删控制点）/ **`MeshSweep`**（圆截面沿曲线扫成管，起半径 + `taper` 收放 + `roll` 截面转角 / `twist` 累积扭转）/ **`MeshLoft`**（多截面放样） |
| **开洞与收边（P1）** | **`MeshDelete`**（按选择集删面 = 开洞）/ **`MeshFillHole`**（中心点扇形封洞）/ **`MeshInset`**（面内缩 = 面板线/边框） |
| **选择与细化（P0）** | **`MeshSelect`**（选择集：`normal`/`box`/`sphere`/`all` + `invert`）/ **`MeshSelectEdge`**（选边：`angle` 硬边 / `all`；输出带 `edges`）/ **`MeshSelectGrow`**（圈扩散 / `shrink`）/ **`MeshMove`**（只捏选中的那一块：平移/旋转/缩放，绕选择集质心）/ **`MeshSubdivideSurface`**（Catmull-Clark 曲面细分） |
| **多边形编辑** | **`MeshExtrude`**（区域挤出）/ **`MeshBridge`**（开放边界环对接）/ **`MeshBoolean`**（并差交，manifold3d）/ `MeshWeld`（**合并顶点** = 按距离焊接） |
| 组织 | `MeshGroup`（**可变口** `mesh` / `mesh.2`…，最多 16；老 `more` 仍可用）/ `MeshJoin`（吃列表）/ `MeshInfo`（顶点/面/尺寸/中心） |
| 落地 | `MeshSave`（写 `.obj` 到 `<图目录>/models`，**透传 mesh** + 出 path/url） |

**P1 的口径**（曲线 / 扫掠 / 开洞 / 内缩，2026-10-20）：

| 件 | 口径 | 坑 |
|---|---|---|
| `CurvePath` | 点列 → `{kind:"curve", points:[…]}`；`linear` 折线 / `smooth` = **Catmull-Rom**（每段 `samples` 份，首尾过控制点） | 少于 2 点报错；`smooth` 在两端做镜像延伸，别指望它保持切线方向 |
| `MeshSweep` | 圆截面沿点列扫：`radius` 起半径、`taper` 末半径倍数、`sides` 截面段数、`caps` 封盖；**法线走平行移动**（Rodrigues 旋转） | ① 环不能复用 `_band_faces`（它按"哪圈 y 高"定绕向 ⇒ 管子朝任意方向会翻错），统一绕向后 `orient_outward`；② 连续重合点会算不出切线 → 先剔掉 |
| `MeshDelete` | 按选择集删面，孤立顶点一起清（索引重排）；全删光报错 | 删完是**开口件**：布尔/CC 都会拒收，用前先想好补不补 |
| `MeshFillHole` | 每个开放边界环 → 中心点扇形；`max_edges>0` 只补小洞 | 补面按边界环的**反向**走（边界边是"面内走向"），否则补片内翻 |
| `MeshInset` | 逐面独立内缩（Blender 的 Inset Individual）：每个选中面往自己中心收 `amount`，边上一圈补边框 | ⚠ **真正的倒角（bevel）没做**：那要半边网格重构（顶点分裂 + 顶点多边形补片），风险远大于收益；视觉上先用 `MeshInset` + `MeshExtrude(inset=…)` 这一档 |

**P0「选择 → 捏 → 收曲面」的口径**（用户口径 2026-10-20：「如果要细化模型，还得做哪些工具」→「开工」）：

| 件 | 口径 | 三个坑 |
|---|---|---|
| `MeshSelect` | 输出 `SELECT` 端口的值 = **面索引表** `{kind:"select", faces:[…], verts:[…], count, note}`；`normal`（方向+角度）/`box`（中心±尺寸/2）/`sphere`（球心+半径）/`picked`（**3D 视图里点出来的面号**，`picked` 文本 `"12, 40, 90-94"`，部分越界静默丢掉）/`attr`（**按属性**：`attr`/`attr_op`/`attr_value`/`component`，Houdini 的 `@Cd > 0.5`）/`all`，`invert` 取反 | 索引是对**那一刻的网格**说的：中间插了细分/焊接/布尔就**要重选**（全越界会明确报错，不静默） |
| `MeshMove` | 只动**选择集覆盖的顶点**：绕选择集质心先缩放、再旋转（X→Y→Z）、最后平移；空选择集直接报错 | 整件变换用 `MeshTransform`；⚠ 未焊的网格（`MeshBox`）选顶面再动会**把面片拽开**（接缝处顶点不共享）⇒ 先 `MeshWeld` |
| `MeshSubdivideSurface` | Catmull-Clark：每层每三角面 → 3 个四边形（6 个三角面），面数 **×6/层**；默认先 1e-6 焊一遍 | v1 **只吃闭合网格**（边界边的 crease 规则没做），遇到边界直接报数字数量 |

**实测数字**（自证里钉住的）：球 `0.5088 → CC 一层 0.502`（只收 1.3%，说明不是"整件缩水"）；
方块 `1.0 → 0.621`（一层的圆角方块）；`38v/72f`（1 层）→ `218v/432f`（2 层）。
⚠ 踩过的坑：`_cc_once` 里"半边 → 面号"曾用 `np.repeat(arange(F), 3)`，而半边是**同类型聚在一起**
（`[所有面(0,1)] + [所有面(1,2)] + [所有面(2,0)]`）⇒ 必须 `np.tile`。错了的表现很直观：
边点飞到球另一侧、45% 的面片翻面、体积只剩一半（0.27）。

**多边形编辑那四个的口径**（2026-10-20 用户口径：「没看到基本的多边形编辑节点，比如挤出，合并顶点，
桥接，布尔」）：

| 动作 | 怎么选 | 三个坑 |
|---|---|---|
| 挤出 `MeshExtrude` | 面法线 · `direction` ≥ cos(`angle`)；区域沿**平均法线**推 `distance`，`inset` 让顶面往中心收 | ① 侧墙绕向必须走 `(a→b)`（原来写反 ⇒ 体积 1.5 变 0.83）；② **先焊**再挤，`box` 每面自带顶点，不焊就留缝（节点默认 `weld=true`） |
| 桥接 `MeshBridge` | 两件各自的**开放边界环**（`caps=false` 那种），`loop_a`/`loop_b` 选第几个环 | 两环**段数必须一样**（不一样直接报错并写出两个数量）；对齐自动做（转位 + 翻向取距离最小） |
| 布尔 `MeshBoolean` | `union` / `difference`（A 减 B）/ `intersect` | ① 要 **`manifold3d`**（可选依赖：`pip install --target <目录> manifold3d` → 面板「依赖目录」填它 → 重载；本机默认装在 `<包数据目录>/mesh_deps`）；② 输入要**一致绕向 + 闭合**，不满足直接报 `NotManifold`（不糊弄）；③ 输入自动 `weld` + `orient_consistent` |
| 合并顶点 `MeshWeld` | `tolerance` 格子 | 太大（比模型细节还大）会把形状焊塌 |

⚠ **新加的 `orient_consistent()`**（`mesh3d.py`）：让相邻面沿公共边**反向**走 —— CSG 和"顶点法线求平均"
都要求这个。快路径是向量化查冲突（没有就只做一次体积定号，`sphere(240×120)` 57ms 不受影响），
有冲突才走 BFS（O(F) python 循环）。实测：球面的**极盖与腰带原来不同向**（82 条边同向），
`manifold3d` 直接 `NotManifold`；修好后 `genus 0 / NoError`。

⚠ **为什么要有 `MeshGroup`**（2026-10-20 做茶壶时补的）：图里**没有"造列表"的原语**，`MeshJoin` 只吃列表 ——
所以"壶身 + 壶盖 + 壶钮 + 壶嘴 + 壶把"这种多零件拼装在图上是接不起来的。
**2026-10-22 起它和 `MeshBoolean` 都是「可变口」**（用户口径「merge 节点，布尔节点应该支持多个端口」）：

| 件 | 口 | 语义 |
|---|---|---|
| `MeshGroup` | `mesh` / `mesh.2` … `mesh.16`（＋ 老口径 `more`） | 接几路并几路，不接的自动跳过；`mesh.16` = `opts.multi_max` |
| `MeshBoolean` | `a` / `a.2`…（并起来当 A）+ `b` / `b.2`…（依次叠） | `a.2+` 先并进 A；`b` 那几路按顺序依次 `union/difference/intersect` |

机制（三处一起，缺一处就"能画不能跑"）：
① **引擎**：类上 `DYNAMIC_PORTS = True`（`validate` 才不把 `mesh.2` 报成 `extra_input`）+
`INPUT_TYPES` 里给那行写 `{"multi": True, "multi_max": 16}`；`execute(self, mesh=None, …, **rest)` 用
`_multi_keys(rest, "mesh")` 按**序号**排（不是字典序：`mesh.10` 排在 `mesh.2` 后）；
② **图**：连线仍是 `inputs["mesh.2"] = [源, 0]`（一对一存法没变）；卡片额外记 `nodes[id].multi_ports = {mesh: 4}`
（`＋ 接一路` 加它；`pySchema.toStorage` 白名单里必须有这个键，否则存一次就丢）；
③ **前端**：`PyNode` 的 `rows` 把 `opts.multi` 展开成 `名字` / `名字.2`…（路数 = max(2, 已接最大序号 + 1,
`multi_ports[名字]`)，上限 `multi_max`，**永远留一个空口**方便直接拖线）；表单卡脚上是「＋ 接一路」
（`data.onAddPort` → 编辑器 `addNodePort`）。canvas 后端读同一份 `rows`，不用另写。
⚠ `data.linked` 才是"已接哪些口"的真来源（`data.widgets` 只收字面量，拿它数会数成 0 → 端口少画 → 边被 RF 丢掉）。

自证：`wuwor l_mesh_nodes -- l_mesh_nodes_plugin_selftest`（49 节点 OK）+ 直接跑
`MeshGroup().execute(mesh=b1, **{"mesh.2": b2, "mesh.3": b3})` → 3 个 box = 72 顶点 / 36 面；
`MeshBoolean().execute(a=…, b=…, **{"b.2": …})` 也能跑。

**参数编辑面板（Houdini parm pane 那味，2026-10-22）**：右栏「🔧 属性」的**参数区**（`pygraph/NodePropPanel.jsx`）：
- 顶上一条 **filter**（搜 名字 / 中文名 / 组名；参数 ≤6 个时不占位）、右边显示命中数；
- **组可折叠**（`opts.group` → `.pyg-parm-ghead`，▾/▸ 记住折叠状态；无组名的排最前、不画组头）；
- 每行**两列**（`.pyg-prow`：左 = 中文名 + `类型 · 单位 · 可选`，右 = 控件）；
- 行上有 **↺ 恢复默认**（写回类里声明的 `default`；没默认值不显示）、**⧉ 复制值** 与 **⏱ 关键帧**（时间线，右键给全套）；
- **改动标记**：值 ≠ `default` 时行加 `is-changed`、名字后一个**琥珀小圆点**（Houdini 那种一眼看出改过哪儿）；
  ⚠ 接线行不参与（值由上游决定）；
- **整组恢复默认**：组头右侧一枚 `↺`（悬停才亮），只重置本组里有 `default` 且**没接线**的行；
- **同行参数**（`opts.row` / `opts.row_label`，Houdini 的 inline parms：`tx ty tz` 一行）：同一 `row` 名的一串参数
  并成**一行**，左 = 总标签（`row_label`），右按个数均分格子、每格自带小标题 + `↺`/`⏱` + 控件
  （`.pyg-inline` / `.pyg-inline-cell`）。已接线的格子显示 `← 源.槽` + 断开。例：`MeshBox` 的 `宽/高/深` = `row:"size"`；
- **表达式显隐**：就是 `opts.visible_when`（受限 DSL，禁 eval、fail-open、只影响显示不影响执行）——
  过滤发生在**归堆之前**，所以藏掉的参数**不占格子**（同行参数里藏一格，剩下几格自动重分宽度）。
  可写 `{"all": [...]}` / `{"any": [...]}` 组合，前端镜像在 `pyConditions.js`；
- `visible_when` 过滤、`disabled_when` 置灰、`opts.tab` 页签、`opts.order` 排序口径**不变**；
- 搜索词与折叠都是**纯界面状态**（不写进图）；节点换人时自动按新节点的组重算。

**📝 定义模式（参数定义覆盖层，2026-10-22）**：面板顶上多一档 `值 | 📝 定义`。
- **值** = 这张图里这个节点的参数值（随图存盘）；**定义** = 这个**类**的参数声明；
- 定义模式里逐参数编辑 `label / group / order / row / row_label / tab / unit / default / min / max / step /
  visible_when / disabled_when / tooltip / card / multi`，**空着 = 跟随源码**；
- 保存落 `<pygraphs>/_node_param_overrides.json`（`{类名: {参数名: {opts 补丁}}}`，原子写），
  `/object_info` 出表时由 `_patch_param_ovr()` 并进那行的 opts ⇒ **卡片 / 属性面板 / 节点帮助 / 连线控件一起变**；
- 端点：`GET /api/pygraphs/param-overrides`（读全表）、`PUT`（`{class, inputs:{参数:补丁}}` 或 `{class, reset:true}`）；
  保存后清 `_OBJECT_INFO_CACHE` 自动重建；面板随即 `loadInfo()` 刷新元数据（`onSavedDef`）；
- ⚠ **只覆盖 `opts`，不动端口与类型**：`type`/名字/端口集合归引擎（改了会让图里的连线与值对不上）；
  真要加参数/改类型 = 改类或 `.nlib` 作者源再 reload 插件；
- ⚠ `PUT` 要登录态（本机直连只读，写会被 401）—— 界面上操作不用管，脚本里要带 cookie/令牌。

**分区（美化，2026-10-22 用户口径「属性面板太乱」）**：值档下整栏折成三节（`<Section>`，可折叠、记住的是组件内状态）：
「**节点**」注释/引擎/标志位 → 「**参数**」方法下拉/提升到接口/搜索/页签/组/行 → 「**高级**」（默认收起）端口表入口/参数卡/本轮值；
危险动作（🗑 删除节点）单独放最底下、一条红线隔开。密度口径：面板里 `.pyg-field` 行高 24px、标签 10px、注释框 1 行起。



**数据契约**（`MESH` 端口，值就是这一个 dict —— 能 `json.dumps`，混引擎段间交接不会坏）：

    {"kind": "mesh", "format": "mesh3d/2",
     "vertices": [[x, y, z], ...], "faces": [[i, j, k], ...], "name": "",
     "attrs": {"point": {...}, "vertex": {...}, "prim": {...}, "detail": {...}}}

> ⚠ `mesh3d/1` 是**历史格式**（只有几何、没有属性）：`mesh_of()` 仍能读老缓存里的值（当作没有属性），
> 但新值一律写 `mesh3d/2`；**不兼容老图**是用户明确要求。

面**一律三角**（多边形先扇形三角化）；`mesh_of()` 还收 `(顶点, 面)` 二元组（`math_shapes` 出的就是它）/
`.vertices + .faces` 鸭式对象（trimesh / Open3D）/ 纯顶点表（当点云）——**老节点能直接接进这条链**。

**3D 预览不用落盘**：宿主 `mesh_preview.mesh_payload()` 本来就认 `{vertices, faces}`（契约 ③），
所以链上**任何一步**都能在 👁 / 底栏「🧊 3D 视图」里看；`MeshSave` 只是为了给外部工具一份文件。
⚠ **几何不随"本轮值"走**（2026-10-20 宿主侧瘦身，见 `l_mindmap_mmd.md` 的「值交接瘦身」）：
超过 512 顶点的网格在 SSE / `values.json` 里只是摘要（`__mesh_ref__` + 顶点/面数），真几何按节点取
（`/api/pygraphs/mesh`）。所以夜里看"本轮值"只有数字是**正常**，不是值丢了。

**实测性能**（本机，`wuwor l_mindmap_mmd -- python -m l_node_engine.mesh3d` 的表）：

| 操作（2.9 万顶点 / 5.7 万面） | 耗时 |
|---|---|
| `transform` / `weld` | 30 / 38 ms |
| `smooth`（1 遍）/ `displace`（3 层噪声） | 55 / 91 ms |
| `subdivide`（→ 11.4 万顶点 / 22.8 万面） | 181 ms |
| `obj_text`（写文本） | 134 ms |
| 六节点链（球→置换→平滑→阵列→变换→导出，7.9 万顶点） | **821 ms** |

**示例图**：
- `lugwit_agent_a/Functions/demo_mesh3d`（方块 → 细分 → 置换 → 平滑 → 阵列 = 程序化山石；
  另一支 `MeshLathe` 花瓶；两个 `MeshSave` 落 `procedural_rocks.obj` / `procedural_vase.obj`）。
  真服务实测 2026-10-20：`POST /api/pygraphs/lugwit_agent_a%2FFunctions%2Fdemo_mesh3d/run` → HTTP 200、
  `models/` 下两份 OBJ 都生成（石头 7938 顶点 / 14400 面 / 501KB）。
- **`lugwit_agent_a/Functions/demo_teapot`（2026-10-20，用户口径「能否做一个茶壶的案例」；2026-10-21 形状精修）**：
  壶身 = `MeshLathe`（10 点轮廓：底收 0 → 肚子 0.56 → 收颈 0.29 → 壶口 0.31，segments 64）、
  壶盖 = `MeshLathe` 小穹顶（盖沿 0.33 压住壶口，`MeshTransform` 抬到 y=0.885 → 盖沿啃进壶口不再飘）、
  壶钮 = `MeshSphere(0.05)` 抬到 1.12、壶嘴 = 锥形 `MeshLathe`（5 点轮廓，底半径 **0.17**、`cap=false` 开口管）
  `rz=-46°` + 挪到 (0.35, 0.33)（**底环整个埋在壶身里**，接缝不再露出一圈切口）、
  壶把 = `MeshTorus(0.30, 0.055)` `rx=90°`（环面转到 XY 平面）+ `sy=1.12` 拉成椭圆握把，挪到 (−0.50, 0.56)；
  然后 `MeshGroup` **串四层** → `MeshSave('teapot')` + `3d_view`。
  实测：整只壶 **1878 顶点 / 3664 面**（**边界环 2 = 壶嘴开口，闭合件其余部分无洞**），
  尺寸 1.75×1.17×1.10；3D 视图「来源」下拉里 14 个中间件都能单独看（壶嘴 200v / 壶把 640v / 逐层组合 772→1038→1238→1878v）。
  ⚠ 两个坑记在这儿：**绕 Z 转正角度是往 −X 倒**（壶嘴想朝 +X 得给负角度）；
  **圆环默认在 XZ 平面**（绕 Y 轴），要竖起来当把手得 `rx=90`。
- **`lugwit_agent_a/Functions/demo_mesh_curve`（2026-10-20，P1）**：① `CurvePath`（4 个控制点、
  `smooth`、每段 8 份 ⇒ 25 个点）→ `MeshSweep(radius .18, taper .35, sides 20)` = 一根收尖的弯管
  （**502 顶点 / 1000 面**，2.52×2.10×0.55）；② 方块 → 焊 → 选顶面 → `MeshInset(0.25)` = 面板线。
  实跑：`swept_horn.obj` + `inset_panel.obj` 都生成。
- **`lugwit_agent_a/Functions/demo_mesh_refine`（2026-10-20，P0）**：方块 → 焊 → `MeshSelect(+y)` →
  `MeshMove(ty .55, scale 1.35)` → `MeshSubdivideSurface(1)` ⇒ **38v/72f** 的圆角锥台（截图那张）。
- **`lugwit_agent_a/Functions/demo_lion`（2026-10-20/21；「能否做一个狮子」→「重做一个细致的狮子」）**：
  **98 个节点**、**16310 顶点 / 32496 面**（2.72×2.22×1.24）。做法：
  躯干 = 三个球 **`MeshBoolean(union)`** 并成一件 → `MeshSubdivideSurface(1)` 收圆；
  脖子 / 四条腿 / 尾巴 = `CurvePath`（Catmull-Rom）→ `MeshSweep`（起半径 + `taper` 收放）；
  头 / 吻 / 鼻 = 球；**鬃毛 = 12 簇锥体绕头一圈**（按角度绕 X 排成径向）+ 一圈托
  （2026-10-21 精修：锥 **0.15×0.54**、托圈 **0.38/0.17** ⇒ 鬃毛成圈包住头，不再是一排"莫西干"；
  吻部 0.16→**0.14** 并前移到 (1.22, 1.45)、鼻子贴到吻部表面）；眼睛/耳朵镜像成对。
  颜色按部位用 `AttrSet(Cd, constant)`（土黄 / 深棕鬃毛 / 奶油吻部与四蹄 / 近黑鼻眼），
  `MeshSave(fmt=ply)` 带顶点色、另存一份 `.obj`。
  ⚠ **顶点色是反照率**：3D 视图里一旦有 `Cd`，`MeshViewer` 会把材质改成哑光（metalness 0 / roughness .85）
  并把 **`scene.environmentIntensity` 压到 0.35** —— 否则 HDR 环境会把 0.5 的底色照成白色（实测踩过）。
  ⚠ 这仍是**顶点色材质**，不是贴图材质：没有 UV 展开（只有平面/柱面投影）、没有多材质槽。
- **`lugwit_agent_a/Functions/demo_mesh_edit`（2026-10-20）**：① `MeshBox` → `MeshExtrude(+y, 0.5)` →
  `MeshBoolean(difference, 球)` = 长高一截的方块顶上挖个球孔（453 顶点 / 902 面，1.00×1.50×1.00）；
  ② 两根 `MeshCylinder(caps=false, segments=24)` → `MeshBridge(loop_a=1, loop_b=0)` = 一根长管。
  真服务实测：`POST …/demo_mesh_edit/run` → HTTP 200、`models/drilled_block.obj` + `bridged_tube.obj` 都生成、
  3D 视图「来源」里 6 号（布尔结果）就是截图那张。

**四条口径（改之前先看）**：
1. **numpy 是"调用时才要"**：`l_node_engine` 内核仍是纯标准库（别的宿主 import 得起来），
   缺 numpy 时节点**明确报错**（不是静默变慢）。所以引擎那个 env 跑 selftest 时几何段会**跳过并说明**，
   真几何要跑 `wuwor l_mindmap_mmd -- …`（那个 env 有 numpy）。
2. **盖的绕向由调用方说清**（`_fan_faces(..., up=True/False)`）：顶盖 / 底盖的圆心和它那一圈**同一个 y**，
   靠"谁更高"推不出来（踩过：圆柱/圆锥体积算成 0 = 一半面内翻）。
3. **细分前先 `MeshWeld`**：`box` 的 6 个面各自带顶点，不焊的话接缝处中点不会复用（面数一样、会出现缝）；
   自证里就是 `weld(box)` → 8 顶点 → 细分 2 级 = 98 顶点（8 + 18 + 72）。
4. **上限在 `LIMITS` 里**（面板 `max_vertices` → 引擎侧 `max_faces = 2×`）：超了**报错带数量**；
   纯 python 时代几十万面会把整轮卡住，"静默慢"比"明确拒绝"难查。

**不引 trimesh / manifold3d**：图元 / 变换 / 细分 / 平滑 / 置换 / IO 这一组已经覆盖，它们真正独有的是
**CSG 布尔**（`manifold3d`）—— 那是"要不要为它加一个运行时依赖"的单独决定，不塞进"基础"里。
将来要布尔：新增一个可选 `import manifold3d` 的 `boolean()`（接口与 `MeshJoin` 同形）即可。

自证：`wuwor l_mindmap_mmd -- python -m l_node_engine.mesh3d`（38 项：图元数量/体积对得上、
四边面三角化、细分中点复用、噪声确定性、weld、OBJ 行数、上限报错 + 上面那张性能表）
＋ `l_node_engine_selftest` 的「插件 mesh3d」组（注册 / 分类 / `MESH` 入参是**连线**不是控件 /
真跑图 / 停用摘净）+ 真服务（2026-10-20 实测）：

```
PUT /api/pygraphs/plugins/mesh3d {"enabled":true} → "分类 `mesh3d` → 72 个节点（顶点上限 200000）"
GET /api/pygraphs/object_info                    → MeshBox / MeshLathe / MeshSave… 都在
```

### 3.0e 节点译名（中文名 / 说明 / 入参悬停，`labels.py`，2026-10-15）

节点库里全是英文类名（`CLIPTextEncode` / `CheckpointLoaderSimple`）时，"看得见"和"用得顺"是两回事。
上游 ComfyUI 前端自带一份 **`locales/zh/nodeDefs.json`**（键 = 节点类名 → `display_name` /
`description` / 每个入参的 `name` + `tooltip`）——**翻译好的现成品**，不该再翻一遍。这一层就是套它。
⚠ **节点名一律 ASCII**（用户口径 2026-10-17：「所有节点名称中不能有中文」）⇒ `comfy_plugin` 只取它的
**入参悬停 / 说明**，`display_name` 丢掉（`KSampler` 仍叫 `KSampler`，不叫「K采样器」）；
`locale` 留空/删掉时也会**整层清一次**（`apply_labels({}, layer="plugin")`），否则上一轮的中文名会残留。

**两层来源、一套形状，本地赢**：

| 层 | 谁写 | 怎么用 |
|---|---|---|
| `plugin` | 插件（「ComfyUI 节点」的 **`locale` 字段**，指向上游那份 locale） | 面板上填路径即可；只作用于**自己那批**节点（`sources`）+ |
| `local` | `<包数据目录>/node_labels.json`（`GET\|PUT /api/pygraphs/labels`） | **盖过 plugin 层**；也用来给**我们自己的**节点起中文名（`{"Preview": {"display_name": "预览"}}`） |

`apply_labels(table, sources=?, layer=?)` 换掉一层之后**重算**：显示名 = 本地 > 插件 > **注册时那个默认名**
（`_BASE_DISPLAY`，由 `register_node` 记一笔）。所以"删掉某条译名"能**回到原来的名字** ——
只写不管删就会"删了还在"（实测踩过，那是个真 bug）。

**三件事分别落在哪**：

  · **显示名** → `NODE_DISPLAY_NAME_MAPPINGS`（节点库标题 / 搜索 / 画布卡片都认它）；
    ⚠ **包自带的显示名一律 ASCII**（用户口径 2026-10-17：「所有节点名称中不能有中文」）——
    中文只许进 `DESCRIPTION` / `SEARCH_ALIASES` / 入参 tooltip；`local` 译名表是**用户自选**覆盖，不在此列。
    自证：`l_node_engine -- l_node_engine_selftest`（"节点显示名：N 个全是 ASCII"）+
    `l_tex_nodes -- l_tex_nodes_selftest`（同一个判据，覆盖贴图/运动图形那 71 个）。
  · **硬盘位置**（2026-10-18）：`get_node_info()` 多带 `source_file` / `source_line` / `package_root`
    （往上找到含 `package.py` 的那层）。编辑器「右键节点 → 查看节点详情」直接显示
    `…\ComfyUI\nodes.py:1575` 这种真地址；comfy 适配的壳类靠 `NODE_SOURCE_FILE/LINE`
    （`comfy_compat` 从上游类取——**上游模块要登记进 `sys.modules`**，否则 `inspect` 只看到壳文件）。
  · **说明 + 入参悬停** → `LABELS`，由 **`node_base.get_node_info()` 拼元数据那一刻套上**（`label_overlay()`）。
    为什么不就地改节点类的 `input.opts`：那些 dict 可能是类属性、也可能来自 `@classmethod INPUT_TYPES()`
    （每次现算）—— 改要么改到别人身上，要么改完就丢；`get_node_info()` 是所有节点元数据的唯一出口，挂那儿一次全吃到。
  · **悬停文案** = `中文名（英文键） · 说明`：带键是为了对着文档/上游查；译名表没给说明就**沿用节点原本的提示**
    （只给中文名不带说明时，别把原来的提示弄丢）；两边都有时**译名的赢**（那就是同一个提示的译文）。

⚠ **入参名不改**：中文入参名（`种子`）**只进悬停**，端口键仍是英文 —— 键就是连线协议，改了等于把已存的图打断。

### 3.0f 入参中文 label（`param_labels.py`，2026-10-19）

用户口径：「给每个节点的每个参数添加 label 属性，label 的值为中文」。

`get_node_info()` 在拼元数据时给**每个入参**的 `opts` 加一个 **`label`**（中文），前端直接读
`spec.opts.label`（画布行名 / 表单卡标签 / 提示），**不用再在前端写死译名表**。

来源优先级（`param_labels.label_of()`）：

| 顺序 | 来源 | 说明 |
|---|---|---|
| 1 | 节点自己写在 `INPUT_TYPES` 的 `opts["label"]` | 动态节点 / 函数派生节点自己说了算 |
| 2 | `PARAM_LABELS[类名][键]` | **本节点专属**（`Branch.to1` = "条件 1 的去向"） |
| 3 | `EXTRA` | 别的包 `register()` 进来的（目前是 `l_tex_nodes.param_labels`，导入 `l_tex_nodes.nodes` 时注册） |
| 4 | `COMMON[键]` | 跨节点同义的键（`a`/`digits`/`items`/`seed`…） |
| — | 都没有 | **不发 `label`**（前端回落显示端口键，不会画错，只是那块是英文） |

⚠ 端口键 / 类型 / tooltip 一个都不动（`label` 只是附加字段）；⚠ 覆盖口径是**每个节点的每个入参**：
`l_node_engine_selftest` 第 38 组会逐节点逐参点出缺的那个（"这些入参没有中文 label：{类名: [键…]}"），
`l_tex_nodes_selftest` 末尾同一条判据（覆盖贴图 / FX / MG / SBSAR 那 73 个）。
comfy 上游节点不在其内（它们的译名走 §3.0e 的 locale 层）。

> ⚠ **`AIVideo` 已退休**（2026-10-20 用户口径「从并列推进到替换内置」）：`ai_nodes.NODE_CLASS_MAPPINGS` 里
> 不再注册它，**节点由节点库提供**（`<工程>/nodes/ai_video.nlib`，`exec: class → l_node_engine.ai_nodes:AIVideo`）。
> 类本身保留（执行体 + `nodelib_export` 的导出源）。**删了那个库就没有 AIVideo 节点** —— 这是预期，不是 bug。

### 3.0g 节点库文件 `.nlib`（一份 SQLite 装 N 个节点，2026-10-19）

节点声明除了写在 `.py` 里，还能放在**库文件**里：`.nlib` 就是**一份合法 SQLite**
（`PRAGMA application_id = 0x4C4E4C42 ('LNLB')`、`user_version` = 结构版本），
表 `meta` / `nodes(id,version,category,label,search,enc,body,sha256)` / `blobs` / `assets`。
实现在 `l_node_engine/nodelib.py`（**只用标准库**：`sqlite3` / `json` / `hashlib`）。

| 入口 | 干什么 |
|---|---|
| `nodelib.pack(src, out)` | 作者源目录（`library.json` + `nodes/*.json` + `assets/`）→ `.nlib`；先写临时文件再 `os.replace`，并 `VACUUM` |
| `nodelib.read_catalog(path)` | **只读目录**（`id/version/category/label/search`，**不需要 key**） |
| `nodelib.load_library(path)` | **整库原子**注册：先建完所有类再动注册表；节点多带 `versions` / `latest` / `library` |
| `nodelib.node_versions` / `get_versioned` | 版本表：注册表只挂 `latest`，旧版按需取；**未知版本报错不回落** |
| `nodelib.discover()` | 扫 `L_NODE_ENGINE_NODELIBS` + `<包数据目录>/nodes` 里的 `*.nlib` **和作者源目录**（含 `library.json` + `nodes/` 的那种，`load_source_dir`）；坏库只记一条 |
| CLI | `l_node_engine_nodelib_pack` / `_inspect`（`--verify` / `--diff` / `--assets-dir` / `--with-key` 解密密文 / `--sql` 只读查询）/ `_export`（`tools/nodelib_cli.py`） |

⚠ 声明里 `exec` 现在**三种**：`{"kind":"callable","target":"pkg.mod:fn"}`（普通函数）／
`{"kind":"class","target":"pkg.mod:Class"}`（**借另一个节点类的 `FUNCTION`** —— 把内置节点迁成文件节点就靠它）／
`{"kind":"pygraph","graph":"pro/Tools/x"}`（**指向一张图**：参数 = 图入参、输出 = 图输出，走 `run_pygraph`）。
`exec.target` / `exec.graph` 在**加载时**就解析：目标/图地址不存在 → 整库拒收并点名。
⚠ `pygraph` 的地址解析**宿主注入**（`nodelib.set_graph_resolver`，`l_mindmap_mmd` 用 `_pygraph_path`）——
引擎不认"工程/别名"；**必须在 `discover()` 之前注册**，否则加载期解析会走默认 loader（只认 pygraphs 根下的名字/绝对路径）。
⚠ **加密**（P4c）：`pack(encrypt=True, keyslot=…, lib_key=…)` 把 `body` / blob 用 **AES-256-GCM** 加密，\n目录列仍明文（**无 key 也能列清单**）；key 走 `L_NODE_ENGINE_LIB_KEY=<slot>:<hex>` 或 `<包数据目录>/license.json`，\nAAD 绑 `库|id|版本`（换行即失败）。签名两档：`ed25519`（推荐）/ `hmac-sha256`。\n都靠 `cryptography` —— **这是 `requires` 里唯一破例的依赖**（2026-10-19 用户口径「可以按需添加依赖」）。
⚠ **`HELP`（帮助文档指针，2026-10-20）**：节点类写 `HELP = "<名字>"`（库声明写 `"help": "<名字>"`）→ `get_node_info()` 发 `help`，编辑器「❓ 节点帮助」给「📖 打开帮助」（`/api/pygraphs/help/<名字>`）；**没写 `HELP` 也发** `node/<类名>` —— 宿主按元数据现生成节点帮助页（`/api/pygraphs/help/node/<名字>`，目录 `/api/pygraphs/help/nodes`），所以每个节点都有帮助页，且永不与实现漂移。
⚠ **`source_file` 对文件节点指向库文件本身**（`.nlib`），实现体在 `info.exec`（`{kind, target}`）\n—— 属性面板那一行「定义」显示的正是它（用户 2026-10-19 口径）。\n⚠ **不引 aiohttp**（同轮追问）：执行模型是**同步拓扑**，节点接口 `execute(**kwargs)` 同步；\n换 async 要贯穿图执行 / 宿主事件循环 / GUI，收益只是HTTP 客户端，而长任务（视频）早就是\n"建任务 + 轮询"、并发靠 `ForEachThread/Concurrent` + `PARALLEL` 节点。要现代 HTTP 客户端用 `httpx`（可选）即可。\n⚠ 样例：`nodelib_samples/ai_video/`（`AIVideoFile` = 内置 `AIVideo` 的迁移版，19 个入参一致）；
自证：`l_node_engine_selftest` 第 39 组。

### 3.0h 参数条件显隐（`visible_when` / `disabled_when`，2026-10-20）

节点在 `INPUT_TYPES` 的 opts 里写（`conditions.py`）：

```python
"image_tail": (..., {"visible_when":  {"multi_shot": True}}),
"quality":    (..., {"disabled_when": {"provider": ["agnes", "vidu"]}}),
```

DSL（**受限，禁 eval**）：`{参数: 字面量}`（等值，数字/布尔宽松）、`{参数: [值…]}`（属于集合）、
`{"all": […]}` / `{"any": […]}`（组合）。

口径：
- **只影响显示与可编辑**：隐藏/置灰的参数**照旧参与执行**（值还在），必填判据仍看 `spec.required`；
- **fail-open**：规则写坏 / 引用的参数不存在 / 值类型不认识 → 当"没有这条规则"（宁可多显示一个控件）；
- 前端 `l_mindmap_mmd/web/src/pyConditions.js` 是**同一份语义的镜像**（即时显隐），权威在引擎；
  卡片行 / 表单卡 / 属性面板三处都过滤；`disabled_when` 在面板**置灰不隐藏**；
- ⚠ **端口不隐藏**（只藏行/控件）：端口没了会把已有连线变成悬空；
- 自证：`l_node_engine_selftest` 第 40 组。

**卡片上只摆少量参数**（2026-10-22 用户口径：「节点上仅仅显示少量参数，完整的参数在属性面板…节点上显示的参数要布局美观，和输入输出端口无关」）：
建模表单卡（`.pyg-form-mesh`）只画 **≤ 4 个**参数（`PyNode.jsx` 的 `CARD_MAX`）—— 先取类里标了
`{"card": True}` 的行（作者点名的关键参数），不够按声明顺序补；剩下的**全在「🔧 属性」面板**，
卡片底部只留一句 `＋N 个参数 · 属性面板`。参数行是**两列网格**（标签左 / 控件右，76px + 1fr），
`.pyg-form-key`/`.pyg-form-type` 在卡上不显示（悬停 title 与面板里有）—— **与端口无关**：
端口照旧只贴卡片左右两端（左 = 真数据口、右 = 输出），大多数建模节点就是 **1 进 1 出**。

### 3.0i 节点钩子（`HOOKS`，2026-10-20）

节点类声明（`hooks.py`）：

```python
HOOKS = {
    "on_param_change": {"param": "provider", "call": "mypkg.hooks:refresh_models"},
    "on_create":       {"call": "mypkg.hooks:init_defaults"},
    "on_loaded":       {"call": "mypkg.hooks:after_load"},
}
```

钩子函数：`def fn(values=None, param=None, node=None, dry_run=False) -> dict|None`，
返回 `{"opts": {参数名: opts 补丁}, "ui": {…}, "warn": "一句话"}`（**其它键丢弃**）。

口径：
- **只读**：不许改图结构、不许写盘；`call` 走 `load_callable` ⇒ **导入白名单**同 PythonCode；
- **异常降级**：抛异常 / 被白名单拦 / 返回值不是 dict → `ok:false + warn`，**永不抛**、不阻断执行；
- `get_node_info()` 发 `hooks` 摘要（`on_param_change` 带 `param`），**不发 `call`**；
- 宿主 `POST /api/pygraphs/node_hook {name,hook,param,values}`（本机写接口要 Bearer）；前端改参数时自动问，
  把返回的 `opts` 只补**那一行**（内存态，不落盘）；
- ⚠ `.nlib` / 作者源声明里也能写 `hooks`（`_normalize_decl` 打包时校验钩子名与 `call`）；\n
- 自证：`l_node_engine_selftest` 第 41 组。

**依赖留在插件层**（用户口径 2026-10-17：「comfyui 作为插件就行，我们的 pygraph 不直接依赖那些库，
但是可以通过插件去添加这些依赖」）：

- 宿主包 `l_mindmap_mmd` 的 `requires` 里**永远不该出现** `torch` / `torchvision` / `torchaudio` / `torchsde`
  —— pygraph 关掉插件也照常工作。
- **`rez` 通道（推荐）**：comfy 插件填 `rez = <包名>`（如 `l_comfyui_ai`）。点「重载」时插件执行
  `wuwor <包> -- python <临时脚本>` 打印那个 rez 环境的 `sys.path` → 只挑 **rez 包 payload**
  （`torch/2.9.0/python-3.12/python` 这种按变体分目录的也算 —— **rez-pip 装的包不在 `site-packages`
  里**，只认 `site-packages` 会把 torch 整套漏掉）、`site-packages` 与 ComfyUI 目录，追加到 `sys.path`
  （**缺的依赖由 wuwo 自动下载安装**，首次很久，之后秒回）；`rez-package-3rd/python/…` 那个运行时
  标准库**排除**（搬进来会顶掉宿主自己的模块）。结果缓存在包数据目录的 `comfy_rez_env.json`；
  **开机那次不解析**（`plugins.is_bootstrapping()`，保持秒起、先走声明面 lite），解析成功即不再注入假模块。
  ⚠ 子进程必须**清掉宿主带下去的 `REZ_*` / `PYTHONPATH`**（宿主自己就在 rez env 里，原样传会让
  `wuwor` 解析不到任何包 —— 实测 `missing: arch, l_app_ready, l_comfyui_ai, …`）。
  ⚠ **解析必须串行**：两次 `wuwor` 同时跑会抢 pip 的暂存目录、把对方的包清成空目录（实测：
  `torchaudio` 只剩一个空版本目录 ⇒ `import torchaudio` 直接 ModuleNotFound）；插件里加了模块级锁，
  锁内复查缓存。**缓存也要验活**：路径不存在就当过期、下次重载重新解析（不然缓存会一直撒谎）。
- 也可以手填 **「插件依赖目录」**（`deps`，多值；env `L_MINDMAP_COMFY_DEPS`）：`pip install --target <dir>`
  的产物或某个 venv 的 `Lib/site-packages`，同样追加到 `sys.path` 末尾；
  存在/缺失都进 `status.deps` / `status.deps_missing`，`status.rez_state` 记 `boot-skip / ok / error`。
- **模型目录（`models` 字段）**：填你自己的模型根（下面按 `checkpoints/`、`vae/`、`loras/` 组织），
  插件把它们挂进上游 `folder_paths`（**不复制文件**），挂完清文件列表缓存 ⇒ `CheckpointLoaderSimple`
  等下拉立刻能选。**换/删目录时先摘掉上一轮加的路径**（`_apply_models_paths(..., previous=…)`）——
  不摘的话旧路径还在，`get_filename_list` 会因为"目录不存在"整条报错、那个节点直接从节点库消失（实测）。
- 流程：装依赖到该目录 → 插件面板把「注入假模块」关掉 → 重载插件。仍是插件内部的事，
  换机器/卸载 = 删目录/删缓存，pygraph 一行不改。
- ⚠ **torch 三件套要同代**：`l_comfyui_ai` 的 heavy requires 钉成 `torch-2.8+<2.9` /
  `torchvision-0.23+<0.24` / `torchaudio-2.8+<2.9`（区间要容得下货架上的 `.cu128` 本地版本标签，
  写 `torch-2.8` 会被 wuwo 的 reuse 检查判成"只认 2.8.0"⇒ 触发一次 3.4GB 的重复安装）。
  不钉时 wuwo 装最新 torchaudio（2.11）与 torch 2.8 的 DLL 不匹配 ⇒ `import torchaudio`
  `OSError: [WinError 127]`。`sympy` 不再单列：它随 torch 装进 `torch/<ver>/python/sympy`，
  torch 包把 `{root}/python` 放进 PYTHONPATH ⇒ 照常 `import sympy`（单列会卡在 wuwo/rez-pip 的
  RECORD 相对路径 bug 上）。
- ⚠ **torchaudio 要新式布局**：wuwo 启动时会把 `x.x.x-pyX.Y` 旧式目录当历史残留清掉 ⇒ torchaudio
  必须走 **rez-pip**（`torchaudio/2.8.0/platform-windows/arch-AMD64/python-3.12/python`）。
  它的依赖（`torch==2.8.0`）由手工 `torch/2.8.0.cu128` 满足；装它时要带 `--no-deps`，
  否则 rez-pip 会去解 torch 的依赖（sympy）又撞 RECORD bug。
- ⚠ **执行要套 `torch.inference_mode()`**：ComfyUI 主程序在 no-grad 里跑节点；我们独立执行时
  `VAEDecode` 出来的张量带 `requires_grad=True`，`SaveImage`/`PreviewImage` 调 `image.numpy()`
  直接 `RuntimeError: Can't call numpy() on Tensor that requires grad`（实测，2026-10-18）。
  适配层 `comfy_compat._call_node()` 统一套（没装 torch / lite 假模块时原样调用）。

实测（2026-10-15，真服务）：

```
PUT /api/pygraphs/plugins/comfy {enabled:true, root:<l_comfyui_ai>/999.0,
     locale:<l_comfyui_frontend>/…/locales/zh/nodeDefs.json, …}
  → detail="加载 5 个节点，中文说明 5 个" · status.labels=0（节点名仍是类名，只译入参）
  → /object_info：ComfyNotNode.display_name = "非"、description 中文、入参 value 悬停 = "值（value）"
PUT /api/pygraphs/labels {"labels":{"Preview":{"display_name":"预览","inputs":{"value":{"name":"值","tooltip":"要看的那个东西"}}}}}
  → Preview.display_name = "预览"、悬停 = "值（value） · 要看的那个东西"（**我们自己的节点也吃**）
PUT /api/pygraphs/labels {"labels":{"Preview":null}}          # 删掉那条
  → Preview 回到 "Preview"、悬停回到节点自己写的（**重算**，不用重启）
```

自证：`l_node_engine_selftest` 第 37 组（归一 / 两层 / 删了回落 / `sources` 过滤 /
悬停三态：译名赢、兜底、不带键重复）；探针那半不用动（插件面板的字段表是接口驱动的，
新加的 `locale` 字段自动出现在面板上）。

### 3.0f 节点图标（`BaseNode.ICON`，2026-10-16）

用户口径：「**功能性节点使用 svg**，svg 的内容**直接写在节点的 json 文件**中」→ 落成"**写在节点类里**"
（`ICON = "<svg …>…</svg>"`，随类一起走；没有图标文件、没有额外 json 表 —— 图标的"手工维护 / 一改就过期"
正是调色板"现烤示例图"当初要躲的坑，所以这里也不引入文件）。

- **形状与链路**：`BaseNode.ICON`（字符串，默认 `""`）→ `get_node_info()` 发 `info["icon"]`
  （**只有非空才发这个键**，与其他可选键一致）→ 宿主 `/object_info` 直通（宿主零改动，远端引擎契约 v1
  的 `nodes()` 也照带）→ 编辑器调色板那一格：**有现烤示例图就画图，没有才画图标**
  （`web/src/PyGraphEditor.jsx` 的 `PaletteThumb` / `NodeIcon`）。图类节点**不该**带图标 —— 它们画的是
  真实输出，信息量更大。
- **分工（关键，别只做一半）**：引擎侧 `icons.py` 只做**粗筛**（不是 svg 开头 / 没 `</svg>` 结尾 /
  带 `<!DOCTYPE·<!ENTITY` / 超 8KB → 一律当"没有图标"），**消毒在渲染端**
  （`web/src/nodeIcon.js`：DOMPurify 的 svg 档，与 Markdown 那条链同一个信任边界）。
  为什么不在引擎消毒：那是件"漏一处就白做"的事，两份不同口径的白名单只会让人不知道该信哪份；
  而那格 `dangerouslySetInnerHTML` 前必须有且只有一道净化。图标串**可能来自远端引擎**，别当自家代码。
- **现在谁在用**（2026-10-16 第二轮，用户口径「**主要是功能节点要加 svg 图标**，噪波节点就应该显示
  噪波本身的形状」）：
  · **引擎全部功能节点**（2026-10-17 时 77 个，数字随节点增加而长）都带了 `ICON`（`Input`/`Output`/
    `If*`/`Loop*`/`State*`/`GetParam`·`SetParam`/`Agent*`/`Gate`/
    `HostStep`/`Tool`/`SubGraph`/`PythonCode`/结构体/类图/数值集合/数学/AI/百度…）—— 图标就写在各自
    的节点类里（一个类一份，就近看）；
  · `l_tex_nodes` 里**不产图 / 要外部文件 / 出矢量场景**的那 29 个（`FxQuadrant`/`FxIterate`/`FxSwitch`/
    `FxTile`/`TexSbsar`/`TexLoadImage`/`TexInfo`/`TexExportImage`/`TexMaterial`/`TexExportMaterial`
    + `Mg*` 运动图形那 19 个）——
    ⚠ 那批**别用点名单**判（`motion_nodes` 19 个就是这么漏过一次）：`l_tex_nodes_selftest` 现在查的是
    **互补不变量**「有图标 ⟺ 没有示例图」；
  · **产图的算子一个都不带**：它们画的是**现烤示例图**（噪波节点显示的就是噪波本身的图案 ——
    为了让"一看就是噪波"，`TexNoise` 示例改成了 `value, scale 12`，见 `l_tex_nodes.md` §6.2）。
  · **上游 ComfyUI 节点**（本机 125 个，`nodes.py` 那批 + `comfy_extras/*` 的 V3）：**也给图标**，
    但**不能写在类里**（那不是我们的代码）—— 由**适配层贴**：`comfy_compat.adapt_node_class` /
    `adapt_v3_node_class` 包壳时 `ICON = comfy_icons.icon_for(名字, CATEGORY)`。三层口径在
    `l_node_engine/comfy_icons.py`：**名字表**（点名的常用节点，38 枚）→ **category 族表**（16 族，
    `advanced/loaders` 逐级回退）→ **通用兜底** ⇒ **任何一个上游节点都不会没图标**（上游改了名字/加了
    节点也不会空）。⚠ 改这张表要**重新加载插件**（或重启宿主）才贴到已适配的壳上。
- **加图标时最容易踩的坑**（我踩过）：把 `ICON = …` 插进**多行 docstring 中间** —— 它就成了文档字符串的
  一段文字、类上并没有这个属性，表现是"那几个节点静默没图标"。判据就是下面那两组。
- **判据**：`l_node_engine_selftest`（粗筛五档 + 元数据只发声明过的 + **功能节点点名抽查**；外加
  「上游节点图标」一组：两张表逐个过粗筛 + `icon_for` 三层 = 名字 / 类别回退 / 兜底 + **真过适配层**
  拿 `get_node_info()["icon"]`）+ `l_tex_nodes_selftest` 的「调色板图标」（**互补不变量**：
  有图标 ⟺ 没有示例图）+ 实测（2026-10-17）：`/object_info` 149 个节点里 **114 个带 `icon`**，
  没有图标的**恰好是那 35 个有现烤示例图的贴图算子**（`NOICON 非贴图类 = []`）。

### 3.0b 参数卡（`AgentParams`：一张卡 N 个参数 × 三态，2026-10-14）
用户口径：**"一个节点能提供多个参数、参数可以无限增加；每个参数可以选择用硬编码 / 输入端点控制 /
链接到图的入参的某个参数"** —— 那些"成组出现"的参数（模型选择、思考强度、最大步数、权限模式…，
也就是聊天输入框那一排）以前只能一个 `Input` 节点对一个参数，或者干脆没法在图里表达。

**行表**（存在节点输入 `params` 里的一份 JSON；`BaseNode.PARAM_SPEC_INPUT`，形状与解析在
`param_spec.py`，前端那份在 `l_mindmap_mmd/web/src/pySchema.js` 的 `parseParamSpec` —— **同一份协议**）：

```json
{"params": [
  {"name": "model",  "type": "str", "source": "literal", "value": "gate::x"},
  {"name": "effort", "type": "str", "source": "port", "value": "low"},
  {"name": "steps",  "type": "int", "source": "param", "param": "n", "required": true}
]}
```

| `source` | 值哪来 | 端口（`rows_to_ports`） |
|---|---|---|
| `literal` | **写死**在行里的 `value` | 只出输出端口 |
| `port` | **由输入端口控制**：连了就用连线值（`None`/没连 → 落 `value`），行表另给它一个**同名输入端口** | 输出 + 输入 |
| `param` | **图的入参**：按名字取这次执行的**图接口值**（`graph.active_interface()`）。**不画线** | 只出输出端口 |

- **每行一个输出端口**（名字 = 行名、顺序 = 行序）→ 下游照常连线消费（`HostStep.plan` 的入参、
  `Preview`、别的节点都行）。
- `required` = **必传**：解析出来是空（`None`/全空白）就报错**点名这一行**（不静默给 `None`）；
  不勾的用行里的 `value` 当默认值。
- **图接口值（`interface`）从哪来**：`execute_graph(prompt, …, interface=…)` —— 不传就**从 prompt 里的
  接口节点扫一份**（类上声明了 `INTERFACE_INPUT = True` 的，也就是 `pygraph.Input`：`name` → `value`），
  传进来的**覆盖同名项**。`run_pygraph` 走这条（它把"默认值 + 实参"合并好交进去）—— 于是
  **"调用方传的实参"与"图的入参"在这一层是同一件事**，`param` 行两种都能取到。
- **进图接口**：`graph_tool_spec` 也扫这些行 —— 只有 `source="param"` 且**名字没被 `Input` 声明过**的行
  进 `parameters`/`defaults` 与 `param_rows`（那张卡就是它的声明处，实参走 `interface=` 注入）；
  指向已有 `Input` 名字的行**不重复声明**（调用方本来就按那个名字传）。所以 `run_pygraph` 的
  "未知参数 / 缺必传参数"两道闸对**卡里声明的参数一样管用**。
- 这个节点 `ALWAYS_RUN = True`：它的值取决于"这次谁在调、传了什么"，那不在结构指纹里 ——
  增量缓存会把上一轮的实参冻住（同 `Time` 节点那条理由）。
- 编辑器：卡片上一条 `.pyg-params` 摘要（每行写清"值哪来"）、双击卡片开**参数卡浮层**
  （`PyParamsOverlay`：增删行 / 类型 / 来路 / 值 / 必传）；canvas 后端把同一份信息缀在行名后面。
  端口**照常由 `rows_to_ports` 推**，所以画布/面板/两个后端都不需要为它另写一套逻辑。

**`step` 是下拉、端口按步算（2026-10-14）**：`HostStep` 声明 `STEP_NAME_PORT = "step"` —— 步骤名只有
**注册方（agent 进程）** 知道，所以宿主把候选补成下拉（候选接口 `GET /api/pygraphs/host-steps`
→ agent `:1250/api/host-steps`，数据来自 `l_agent_chat.graph_turn.STEP_META`），并按
`/ports?kind=hoststep&target=<步>` 把**那一步**的入参/出参画成端口（`result: bool/str`、
`tool_call` 的 `tool` 参数）。**编辑器侧没有一行点名 `HostStep` 的代码** —— 全靠元数据里的
`values.api`（`NAME_PORT_KINDS` 那张表加一行就通了）。⚠ 别把"值来自接口的端口"当成"引用另一张图"：
「↗ 子图 / 双击进内层图」只认 `kind === "subgraph"`。

### 3.0g 入参的**引用 / 设置**（`GetParam` / `SetParam`，2026-10-16；照 UE 蓝图）

用户口径：「参考 ue 蓝图，`…/agent/default_intelligent_agent` 的入参添加**不必全都要在图中添加一个
input 节点，孤立的节点没啥意义**；另外入参可以**左键按住入参不放拖动到视图中**，拖动过来后显示
**引用还是设置**，也是参考 ue」。

- **声明 vs 引用**：`Input` 节点 = **声明**（`interface_values()` 扫它；只在「⇄ 接口」面板里列）；
  图上出现的是**引用它的节点** —— `GetParam`（读，UE 的 `Get <变量>`）/ `SetParam`（写，`Set <变量>`）。
  ⚠ 一个 `Input` 节点**没人引用**时**不画卡片**（`pySchema.isDeclaredOnly`；`flags.show` 是例外）。
- **拖拽**：在「⇄ 接口」里**左键按住那一行拖进画布** → 松手弹「引用 / 设置」；拖到**端口上**时第一条
  写成「引用（贴到 x.y）」= 落一个 `GetParam` 并**贴成端点胶囊**（**endpointChip**，见 §4 端点胶囊口径）。
  📛 **术语（2026-10-17 定）**：端口上那枚写着参数名的小节点正名 **endpointChip（端点胶囊）** ——
  语义 = **参数引用**（Parameter / Variable Reference，UE 的 `Get <变量>`）；形态 = 端口胶囊
  （port chip / pill）；交互 = **dock（收成胶囊）↔ unfold（展开成卡片，按 `D`）**；
  ⚠ **图文件里的键仍叫 `nodes[id].dock`**（wire 名，老图/资产零迁移，`gen_agent_asset` 与
  `AssetGuardTest` 比的也是它）；代码里一律叫 chip：`pySchema.endpointChipsOf / attachEndpointChip /
  detachEndpointChip / unfoldEndpointChip / isEndpointChipped`、CSS `.pyg-epchip`。
- **取值**：两者都走 `graph.active_interface()`（与参数卡的 `param` 行同一份）。所以
  `GetParam` **认调用方实参**（`Input` 节点自己不认 —— `Input.execute` 只返回自己的 `value`，老坑）；
  `SetParam` 写的是**这一轮**那份 ⇒ 它下游的 `GetParam` / 参数卡的 `param` 行读到新值，
  **agent 侧（发请求之前按名读图文件）看不到图内的 Set**（要真改 `model`/`permission_mode` 那类，
  改的是**声明**）。
- ⚠ **顺序**：本引擎没有 UE 那种 exec 线，顺序只由**数据依赖**定；`Set(p)` 与 `Get(p)` 之间没有数据依赖
  ⇒ 想让 Set 先跑就接一根 `get.after <- set`（`after` 是 `forceInput` 的"顺序端口"，口径同 `HostStep`）。
- **卡片上要写清楚"引用的是哪个参数 + 默认值"**（2026-10-17 用户口径：「展开后为啥参数端点没有显示
  参数名……并且要显示默认值」）：节点的 `param` 只是**一个名字**，默认值/类型/可空在图级声明
  `graph["params"]` 里 —— 编辑器把两者合起来显示（`pySchema.paramRefOf` → `data.paramRef`）：
  · 那一行 `param` 的**显示名换成参数名**（`spec.display`；端口键仍是 `param` —— 改名等于把所有连线打断）；
  · 卡片顶上一行 `📥 引用 <名> · 默认 <值> · <type> · 可空`（没声明 → 黄色警示，没选 → 「⚠ 还没选入参」）。
  canvas 后端同一口径只改**端口显示名**（那行文字画不进卡片，见 `CanvasSurface` 的行名绘制）。
  ⚠ 引擎给它声明的 `values.api = "/api/pygraphs/graph-params"` 在**本站主里还没有这条路由**
  （编辑器走静态候选 `withParamChoices`，所以编辑器不受影响）—— 别的宿主接的时候记得提供它，否则
  `param` 下拉是空的。
- ⚠ 两个都 `ALWAYS_RUN`（同 `Time` 的理由）：`Get` 的值取决于"这一轮被谁 Set 过"、`Set` 有副作用，
  被增量缓存跳过就会"上游 Set 没跑、下游 Get 读到旧值"。
- 自证：引擎侧小脚本（读声明 / 认实参 / Set 生效 / 拼错名字明确报错）+ 探针 `dock` 的 ⑨a/⑨b/⑨c
  （拖到空白落 `Get`/`Set`、拖到端口贴成小节点）+ `agentgraph`（只声明的入参不画卡片、接口面板里列着）。

### 3.0f 参数 = 图入参 + 端口小节点（2026-10-16 用户口径）—— 智能体那张图的参数卡已退休

用户口径：「**先把哪些参数添加为入参，然后去作为小节点附着到需要这个参数的节点，所有参数必须在
入参中定义**」。落点（`l_agent_chat/graph_turn.default_turn_graph()` 是唯一声明处）：

- **`PARAMS` 表** → 一张表 24 行，每行 = 图级 `graph["params"]` 里**一条声明**（`name` / `type` / `value` /
  `optional` + **控件元数据** `widget/options/values/label`）。⚠ **2026-10-17 起声明不再各占一个
  `Input` 节点**（那个类型删了；用户口径「请删除 input 这个节点类型」）；"所有参数必须在入参中定义"
  这条仍然成立 —— 入参 = **图级 `params`**。
  另有一张 **`DOCKS` 表**（参数名 → `[(宿主节点, 端口), …]`）写"它喂哪儿"。
- **生成时一次做两件事**：① 每个参数声明进 `graph["params"]`（调用方实参的落点 / 工具 schema 的来源）；
  ② **真喂端口的那些**（`DOCKS` 表里的 15 个）各起一个 **`GetParam` 引用节点**（`gp_<参数名>`，
  带上声明 `type` ⇒ 值按类型转真对象）：
  连上那几根线（门/步骤执行时真读它）**并且**给 **Get 节点**记 `dock: [{node, port}, …]`（端点胶囊标记）
  ⇒ 画布上不画卡片，改在那些端口旁边贴成**端点胶囊**（`pySchema.isEndpointChipped` / `endpointChipsOf`；
  前端 DOM 是 `.pyg-epchip`、canvas 后端自己画一枚胶囊）。
  **一个参数喂多处 = 多处各一枚**（`gate_hits` 贴 4 道门、`read_repeat_limit` 贴 3 个 `tools_*`）。
  ⚠ **为什么是 Get 而不是 `Input` 节点**：只有 `GetParam` 读 `active_interface()` ⇒ **调用方传的实参**
  才改得到端口上那几道门。
  ⚠ 换载体时**别留陈旧 `dock`**：`gen_agent_asset.py` 的合并只 `update` 不删键 ⇒ 生成器里去掉 `dock` 时
  要显式 `keep.pop("dock")`（已加），且 `shape()`（生成器与 `AssetGuardTest` 两处）**要一起比 `dock` 与图级
  `params`**，否则这种漂移看不见（实测：一个参数被算两遍、探针计数打架）。
- **子图的参数**（2026-10-17）：内联时把内层 `graph["params"]` 一起带过去（`SubGraph` 的 **hidden 输入**
  `sub_params`，不在端口上画），`run_nested` 再把 `{声明默认值, 端口实参}` 并成嵌套执行的
  `interface=` —— 于是子图里的 `GetParam` 读得到调用方灌进来的实参（老图里 `Input` 节点的
  `inject_params` 那条老路仍然保留，双轨）。
- **`dock` 是纯编辑器标记**：引擎不认、`toPrompt` 不发它 ⇒ 删掉它图照样跑，只是卡片画回画布上；
  `removeNode` 会把它清掉（宿主被删的那几条），免得新节点**复用同一个 id** 时莫名贴上。
- **读取**：`graph_turn.agent_params(graph, overrides)` 先扫 `Input` 节点（**调用方传的按名覆盖**），
  再回落老图里的参数卡（`literal`/`param` 两态；`port` 态跳过）—— 同名以入参节点为准。
- **那排控件**：工具卡/agent 那排从 `Input` 的控件元数据来（`paramRows`）；**已经被线引出的入参不进那一排**
  （它们是图内旋钮，端口旁已经写着名字了）。
- 自证：`tests/test_graph_turn.py`（84 项，含 `AssetGuardTest`「资产 ≡ 生成器」）＋
  `wuwor l_agent_chat -- l_agent_chat_graph_smoke`（4/4：真图 + 真模型 + 真服务）＋
  `node l_mindmap_mmd/999.0/tools/web_probe.cjs agentgraph`（8/8：节点/边数、**入参卡在可视区**、
  贴上去的不画卡片、**小节点数量 = `dock` 条目数**）。

### 3.0a `Time`（时间，2026-10-13；2026-10-14 加自走帧、10-15 加 `fps`）—— **时钟**变成一根线

`time_nodes.py`，节点类就叫 `Time`（`display_name` 时间）。**零输入、四个输出**：

| 输出 | 含义 |
|---|---|
| `time` | 秒（`clock["time"]`） |
| `frame` | 帧号（`clock["frame"]`） |
| `phase` | `time / duration` 的**余数**（0..1；`duration<=0` 给 0）—— 驱动 0..1 参数最顺手 |
| `fps` | **实测帧率**：这张图被**重跑了多快**（`graph.measured_fps`；开头几拍是 0） |

**引擎自己不走时间**：它只读 `graph.active_clock()`（宿主在 `execute_graph(clock={...})` 里塞的
ContextVar，与 `emit`/`guard` 同一套"引擎只留钩子"的分工）。但**没人喂时它给自走的那份**
（2026-10-14 用户口径「Time 不依赖时间线，而是引擎时间」）：每次顶层 `execute_graph` 帧号 +1
（`_CLOCK_TICK`）、`time = 帧 / ENGINE_FPS`（24）、`duration = 1.0`
⇒ `phase` 每 24 帧扫一遍 0..1，编辑器那个 `▶ 动画` 一开就动。

- 谁在喂：编辑器底部「🎞 时间线」播放时**每帧把 `time/frame/duration` 带进 `POST …/run` 的正文**
  （宿主 `server.py` 的 `_run_clock` → `clock=`）。所以"时间驱动参数"就是**一句普通连线**：
  `Time.phase → 某个 0..1 参数`，不用给那个参数打关键帧、引擎也不需要知道什么叫"帧"。
  ⚠ 三个数是**相对时间线[起止帧范围]起点**的（`time` 从 0 数、`duration` = 范围长度）——
  范围是整条时与"从 0 数"完全一样；把范围设成 24..72 帧，`phase` 就正好扫过那两秒。
  **喂了以喂的为准**（`time`/`frame` 用宿主那份）；`fps` 永远是**实测**的（宿主那份里没有 fps 字段）。
- `fps` 量的是"**图被重跑了多快**"，与标称的 24 **不是一回事**：它是**最近 2 秒窗口里跑了几拍**
  （`graph._FPS_STATE` / `_tick_fps` / `measured_fps`）—— 重图一轮 6s ⇒ `fps ≈ 0.17`，动画慢就慢在图本身
  （想快：尺寸设**相对父级** / 把「⇄ 接口」基准尺寸调小）。
  ⚠ **为什么是窗口平均而不是"上一拍间隔取倒数 + 指数平滑"**（2026-10-15 用户先说"显示自己的 fps"、
  随后报"很不稳定"）：动画是"按 fps 节流 + 上一轮没回来就跳拍"，真实间隔本来就是帧周期
  （1/24 = 41.7ms）的**整数倍** ⇒ 速率在 24 / 12 / 8 … 之间跳；单拍倒数把抖动全暴露，
  指数平滑（alpha 0.4）又要十几拍才收敛、中间一直在"爬"。窗口平均 2~3 拍就到位：
  实测 25fps 档稳定落在 **24.0~25.5**、10fps 档落在 **9.8~10.1**（旧实现里同一段是 21.4→9→5.8→…→1 的一条长尾）。
  ⚠ **只有最外层那次执行计一拍**（`_RUN_DEPTH`）：子图 / FX 图跑在外层执行**里面**，计重了帧率会算成假高值。
  样本不足（刚开动画）报 0；停超过两个窗口就把旧样本清掉（不拿"停那几秒"算出一个假低值）。
  宿主喂了时钟也照样量。卡片上这个值直接显示（`PyNode` 的**输出端口本轮值**，见 `l_mindmap_mmd.md`）。
  ⚠ **WebGPU 那档（编辑器动画走 `runGpu` 时）用同一套口径**（2 秒窗口 + 1 位小数，见 `texGpu.js`）
  —— 两边不一致的话，切档时数字风格会突变。
- `ALWAYS_RUN = True`（`BaseNode` 上的标记）：**绝不进宿主那套增量烹饪缓存** —— 它的值取决于
  执行时钟，而时钟不在结构指纹里；不标的话缓存一命中，"时间"就永远停在上一轮那一秒。
  ⚠ 副产品（这正是我们要的）：`_execute_plan` 不给它记指纹 → **它的下游也一律不进缓存**
  （`_node_fingerprint` 里"上游指纹未知 → 自己也别缓存"那条），所以接出去的那条链每帧都会真算。
- 缩放 / 偏移：接 `Calc`（刻意不做 `speed` 参数，节点越薄越没歧义）。
- 只对**本地引擎**有意义：远端宿主没有"现在几秒"，混引擎的远端段里 `Time` 输出 0。
- 自证：`wuwor l_node_engine -- l_node_engine_selftest` 第 35 组（自走帧、实测 fps、嵌套不重复计拍、
  宿主时钟优先）；浏览器探针见 `l_mindmap_mmd` 的 `web_probe` 与 `Rez-Docs/Rez_pkg/l_mindmap_mmd.md` 的「时间线」一节。

### 3.0 AI 节点（6 种，输入输出只有 文字 / 图片 / 视频 / 音频）

| 节点 | 入 → 出 | 默认后端 |
|---|---|---|
| `AIText` | 文字 → 文字（LLM） | hub `POST /v1/chat/completions` |
| `AIVision` | 图片 + 文字 → 文字（VLM，图片走 data URL 塞进多模态消息） | 同上（`model` **要点名带 vision 的那只**，`auto` 的降级链可能先撞上纯文本模型） |
| `AIImage` | 文字 → 图片（**输出图片文件路径**） | hub `POST /image`（`provider` 默认 `volcengine`） |
| `AIVideo` | 文字（可带首帧图）→ 视频（输出路径） | hub **异步两面**：`POST /video/task` 建任务 → `GET /video/task/{id}` 轮询（`provider` 留空 = 自动挑一家配了钥匙的；`resolution` 留空 = 该家默认档） |
| `AISpeech` | 文字 → 音频（TTS，输出路径） | hub `POST /v1/audio/speech`（`model` 默认 `indextts2` = 本机 IndexTTS2） |
| `AITranscribe` | 音频 → 文字（STT） | **hub 没有这条路由** → 必须自己配 `L_NODE_AI_STT_URL` |

⚠ **`HUB_PATHS` 是"hub 真实路由表"，不是"OpenAI 兼容面"** —— 2026-10-11 核对过：
`sst` 故意**不在表里**（hub 未提供），image/video 是 hub 原生路由（不是 `/v1/images/generations`
那种 OpenAI 面，那两条在 hub 上 **404**）。改这张表之前先看图：hub 侧的权威是
`GET :8462/openapi.json`（匿名可读）+ `server.py` 的 `@app.post` 清单。
⚠ 表里的 `/video` 现在只用来**推导基址**：节点实际打的是同族的 `/video/task` + `/video/task/{id}`
（异步两段；`/video` 那个同步面留给脚本/一次性出片）。

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

### 3.1 基础节点（18 种）

| 分类 | 节点 | 用途 / 端口要点 |
|---|---|---|
| `l_node_engine/flow` | `LoopStart` / `LoopEnd` | 循环括号。**不要画回边**：`LoopStart → 循环体 → LoopEnd`，环由执行器接管。`LoopEnd.items` 是逐轮收集，`state` 回灌下一轮。轮数上限 `L_NODE_ENGINE_MAX_ITERATIONS`（兼容旧名 `L_COMFYUI_MAX_ITERATIONS`）。**`until_stop=True` = while 循环**（2026-10-13）：忽略 `iterations`、跑到 `LoopEnd.stop` 为真；⚠ 必须真接 `stop`，否则明确报错（防跑到上限）。**循环可嵌套**（2026-10-13 起；内层括号并入外层区，跨边界仍拒） |
| | `Compare` / `Not` / `And` / `Or` | **值级判断**（2026-10-12）：`Compare(a, op, b)` 出真/假（两边都能当数字就按数字比）；`Not` / `And` / `Or` 逻辑组合。喂给 `IfStart` 用 |
| | **`Select`（多路值选择，2026-10-13）** | `Select(c1,v1,c2,v2,c3,v3,default)`：按 **c1→c2→c3** 取第一个为真的 `v`，都不真给 `default`（语义对齐流程图那边的 `branch`：**按顺序、第一个命中的赢**）。多路**控制流**见下面的 `Branch` |
| | **`Branch`（多路 goto，2026-10-14）** | `Branch(c1,to1,c2,to2,c3,to3,default)`：**执行级路由**（flow 的 `branch` 对等物）—— 按 **c1→c2→c3** 取第一个为真的，**跳到**它对应的 `toK` 节点（`toK`/`default` 填**节点 id**）；**只跑命中的那条臂，其余臂整段跳过**（区内写 `(None,)`）。出口 `arm` = 选中的臂序号（default 臂=3、都没命中=-1）。区靠"从各目标**可达**、且从别的目标**到不了**"切出来（各目标在图上**不能互相嵌套**；各路汇合点不属于任何臂 → 按外层顺序无条件跑）。⚠ v1：`Branch` 不能放进 if/循环区里、臂与 if/循环区只能"**整段包含**"不能部分重叠 —— 都**明确报错**不静默跑错。自证第 32 组 |
| | `IfStart` … `IfEnd`（+ 可选 `ElseStart`） | **条件分支括号**（2026-10-12；**`else` 2026-10-16 补**）：条件真 → 执行括号之间的节点；假 → **整段跳过**（区内节点输出写 `(None,)`，下游读它拿到 `None`、不会报「上游没有产出」）。区怎么认：**IfEnd 的祖先 − IfStart 的祖先**（所以 `IfEnd` 至少要接一条区内的线）；`IfEnd.taken` 告诉你这一支走没走。**`else` 直写**：夹一个 `ElseStart` 变成三件套 `IfStart … ElseStart … IfEnd` —— **then 区末节点接 `ElseStart.x`**、**`ElseStart.taken` 接进 else 区**（else 区末节点接 `IfEnd.x`），条件真跑 then 区、假跑 else 区（三个括号一起进循环区，跨边界仍拒）。⚠ 两处口径（刻意，别当 bug）：`IfEnd.taken` 仍是"**if 支走没走**"（走 else 时 = False）、`ElseStart.taken` 才是"else 支走没走"；else 区**必须真接 `taken`**，孤儿 `ElseStart` **明确报错**。else-if 链仍推荐 `Not` + **嵌套 if**（2026-10-13 起支持嵌套）；多路值选择用 `Select`、多路跳转用 `Branch`；**`if` 放进循环体也支持**（2026-10-12） |
| `l_node_engine/class` | `ConstructClass` / `CallGraphMethod` | **类图**（P3 v1，2026-10-13）：**用图定义类** —— **实例 = 普通 dict**（`{"__class__": "工程/类名", 字段…}`）、**方法 = 一张图**（`Input` 是参数、`Output` 是返回，与 `SubGraph` 同一套契约）。`ConstructClass`：类名 + 各字段（动态端口，字段表由宿主按类资产内联，**与结构体共用 `STRUCT_NAME_PORT`/`resolve_struct_refs`**）→ 实例；`CallGraphMethod`：实例 + `class` + `method`（宿主按类资产的 `methods = {方法名: 图地址}` 把那张图内联成 `sub`，`METHOD_PORT`）→ 跑方法图，**`self` 按名字注入**、方法图里 `GetItem(self,"n")` 就读字段，返回它的 `Output`（多输出照旧按下标）。读字段 `GetItem`、改字段 `SetField`（值语义）。**不生成 python 类**（纯 JSON 资产、可热更、无动态类序列化坑）；v1 边界：方法名是**静态**（一个节点一个方法，同蓝图）、方法图里**再定义方法**不支持 |
| `l_node_engine/struct` | `MakeStruct` / `SetField` | **结构体**（P1，2026-10-13）：字段表 → dict / 改字段返**新** dict。**运行期就是普通 dict**（不引入新类型：落盘、增量指纹、节点间传递、预览全不用改）；"类型"只体现在**端口**（`MakeStruct` 按字段表出**动态输入端口**，没接的用字段 `default`）与**资产**（宿主存 `{"kind":"struct","fields":[…]}`，见 `l_mindmap_mmd.md`）。字段表三种写法：`[{"name","type","default"}]` / `["a","b"]` / JSON 文本；**重名当场报错**。取值用现成的 `GetItem`（`key` 填字段名）；**按名字引用**（v2，2026-10-13）：节点类声明 `STRUCT_NAME_PORT = ("struct","fields")` → 宿主的 loader 读资产、引擎 `resolve_struct_refs` 把字段表**内联成 JSON 文本**（⚠ 别内联成列表：长度 ≥2 的 list 会被当**连线**，校验报 `dangling`；字符串没这问题，指纹也稳）；引用不到 → `StructNameError`（带节点，宿主翻 400） |
| `l_node_engine/data` | `Calc` / `Count` / `Sum` / `GroupCount` / `ToText` | **数值与集合**（2026-10-13 补；原先"图算不了数"）。`Calc(op,a,b)` 四则（`+ - * / // % **`，解析不动**报错**不当 0）；`Count(items)` / `Sum(items[,digits])`（`Sum` 遇 `[[键,个数],…]` 形状自动对**第 2 项**求和）；**`GroupCount(items[,key,key_sep])`** 分组计数 → `[[键,个数],…]`（**按个数降序、同数按键升序**，与输入顺序无关；`key_sep="/"` 时对字符串取**第 1 段** —— 配 `glob('*/*.py')` 就是"每个子包几个文件"）；`ToText(value[,sep])` 成文本（不想为串一下就拉 `json` 模块时用它） |
`…/objects` | `MakeInstance` | 造实例。`target`（路径/已注册键）**或** `class_ref`（类对象，优先）；`init_args` 是构造 kwargs 的 JSON；`singleton` 决定复用 |
| | `CallMethod` | 调对象方法。固定三项 `obj_path/method_name/call_args` + `class_ref` 可选；**方法参数端口按节点问**（见坑 2） |
| | `GetAttr` / `GetItem` | 取属性 / 取下标（纯数字当整数下标；可取不到给 `fallback`） |
| | `CallFunction` | 按 `module:func` 调任意函数（也支持 `module.Class.method`，走导入白名单），端口按签名展开；固定输入 `func_path`/`func_args`（**任意**函数参数名都能直接上端口，含 `path`）；**位置专用参数**（`len(obj, /)` / `operator.sub(a, b, /)`）也展开，执行时按签名顺序绑到 `*args` |
`…/classes` | `LoadClass` | 路径 → **类对象**（蓝图 Class Reference 引脚），另给路径字符串 |
| | `ClassOf` | 对象 → 所属类 |
| | `ClassInfo` | 类 → name / module / qualname / bases / doc |
| | `CastTo` | `ok` + `value` 两路；不匹配 `ok=False, value=None`（Cast Failed 引脚），`strict=True` 则报错 |
| | `IsChildOf` | 类是不是某基类的子类（给实例则等价 `isinstance`） |
| | `SubclassesOf` | 类 → 子类列表（`classes` 是真实类对象列表，可递归；`paths` 是路径字符串，便于打印） |
| | `FindClasses` | **按关键词/模块/基类搜类**（`keyword` / `module` / `base` / `limit` → `classes` 真类对象列表 + `paths` + `count`）。⚠ **只搜已经导入的模块**（`sys.modules`），不 import 新东西 —— 要搜磁盘上的包用宿主侧的 `introspect.discover_classes`。编辑器用它给"节点搜索框"加一节「🧩 类」（点一下落 `MakeInstance`），契约里的 `/engine/classes` 也是同一个实现 |
| 💡 | `functions.find_functions`（**不是节点**，是宿主调的函数） | **搜函数**（2026-10-17）：`keyword` / `module` / `limit` → `[{path: "模块:函数", name, module, signature, doc}]`，编辑器那节点搜索框用它加一节「ƒ 函数」，点一下落 **`CallFunction`**（`func_path` 预填），**真 import 在跑图那一步**（`CallFunction` → `load_callable` 过白名单）。三条口径（用户原文"搜索 0 副作用，加入图中也不要导入，执行图的时候由节点引擎导入"）：① **不 import**（只看 `sys.modules`；`selftest` 拿"装得到但没导入"的模块当探针数 `sys.modules` 增量）；② **不按 `__module__` 过滤、不按对象去重**（`os.path.getsize` 的 `__module__` 是 `genericpath`，丢掉就搜不到用户手打的那条；去重又会被更短的 `ntpath:getsize` 顶掉 ⇒ 靠**排序**把"最像你打的那条"排最前：路径整条相等 > 前缀 > 函数名前缀 > 其余，再按公开/短/字典序）；③ 排掉 **`typing` 别名**（`Callable[...]` 可调用但不是函数）、类、模块。宿主口径见 `l_mindmap_mmd.md` 的「⌨ 节点搜索」一节（搜索框一次同时给类/函数两张表）。**补：显式反射搜索**（2026-10-18 用户口径「搜索时试试通过反射获取子类和方法」）= `functions.reflect_module` / `POST /api/pygraphs/reflect`：**用户点按钮才 import**（跑顶层代码，过 `check_import_allowed` + 执行面闸），列出公开的**函数 / 类 / 方法 / 已加载子类**（`__subclasses__()`）；`method` 带 `owner`（落 `CallMethod` 的 `obj_path`）。编辑器入口只在搜索词带模块前缀时出现（`cv2.imread` → 模块 `cv2`），不点仍走"只搜已导入"那条 |
`…/io` | `Const` | **常量**（任意类型，2026-10-17 新增）：`value` + `type`（python 类型表达式，编辑器按它给控件、`execute` 按它把值转成真对象）。⚠ **`Input` 节点类型同轮退休** —— 入参的**声明**搬到了**图级 `graph["params"]`**（「⇄ 接口」面板里增删改；形状 = `{name, type, value, optional, description, label, widget, options, values}`），图上要用某个入参走 `GetParam`（引用）/ `SetParam`（设置），写死的值用 `Const`。老图里的 `Input` 节点：**读**这一侧由 `graph_params()` 双读、**跑**这一侧由 `graph._rewrite_legacy_inputs` 就地重写成 `GetParam`（连 `type` 一起带，值仍由接口那份提供）⇒ 老图不用手改就能跑。**值怎么存见 §3.3** |
| | `Output` | 图的**返回值出口**（= 工具返回）。按 ComfyUI 约定返回 `{"result": …, "ui": …}`，`ui` 供画布内联预览 |
| | `Preview` | 显示任意结果：**原样透传**（插在链中间不断流），同时把值交给画布渲染。`mode` = `auto`（默认）/`text`/`json`/`repr`/`csv`；`rows`（默认 50）控表格行数。**`auto` 会认表**：`.csv`/`.tsv` 文件、多行且分隔符一致的文本、`list[list]`、`list[dict]` → 给 `ui.table`（画布画成真表格，首行当表头，最多 12 列），认得不像就照旧当文本/JSON（要文本显式写 `mode="text"`，别误伤多行日志）。编辑器里表格预览的折叠条右侧有个 `⤢`：**悬浮放大窗**里是同一张表（12px 字、不设尺寸上限；`Esc` 或点空白关） |
| `…/note` | `Note` | **画布注释**（2026-10-12）：`text` 是 Markdown，画布上渲染成便签、**双击就地编辑**（`Esc` 放弃）。**不参与执行** —— `execute` 返回空 tuple，跑图时就是个空操作（但必须注册：没注册的 class_type 执行器会报 `Unknown node class`）。前端渲染见 `l_mindmap_mmd` 的 `PyNote.jsx` + `markdown.js`：**marked 解析 + DOMPurify 消毒**（两步都要 —— marked 不做消毒，注释里的 `<script>`/`onerror`/`javascript:` 全靠 DOMPurify 删；实测探针通过）。**2026-10-20 起也认 HTML**：`text` 以标签开头 → 卡片里走 **sandbox iframe**（`allow-scripts`，**不给** same-origin），**主题跟卡片底色**（纸白→亮 / 其余→暗；注入 `color-scheme` + `--note-*`），标题栏 **🪟** 打开的是**固定网址** `/api/pygraphs/note_window?graph=<图>&node=<id>&theme=<light|dark>`（可刷新/收藏/分享；同名窗口复用同一标签页）；示例图 `<工程>/demo/demo_nodelib_features`（暗/亮各一张）；**也能指向帮助文档**：`text` 只写一行 `help:<名字>` → 卡片渲染包里 `l_mindmap_mmd/help/<名字>.html`（固定网址 `/api/pygraphs/help/<名字>`，改文件即更新） |
| `…/io` | `Ask` | **人在环**（2026-10-12）：跑图到此**暂停**问人（`prompt` / `options` 逗号候选 / `default`），答完继续、`answer` 往下流。实现 = 引擎钩子 `set_ask(fn)`（与 `set_guard` 同形状）+ 宿主通道：`l_mindmap_mmd` 把问题推成 SSE 的 `ask` 事件，浏览器在运行日志上方的询问条作答 → `POST /api/pygraphs/runs/{run_id}/input` 唤醒线程。**宿主没挂钩子时明确报错**（绝不静默返回空串 —— 审批被跳过却继续跑比报错危险）。超时 `L_MINDMAP_ASK_TIMEOUT`（默认 180s） |
| | `StateIn` / `StateOut` | **跨轮状态**（2026-10-12，会话续跑）：`StateIn(name, default)` 读上次存的值（`default` 是 **JSON 文本**，读不动退化成普通字符串）；`StateOut(name, value)` 原样透传给下游、**跑完由宿主落盘**。引擎**不碰磁盘**（它不知道"会话"是什么）：宿主跑前把会话里 `name` 的值灌进 `StateIn.default`，跑后把 `StateOut` 的输出写回 —— `l_mindmap_mmd` 落成 `pygraphs/_sessions/<图名>/<会话id>.json`，跑图带 `{"session": "<id>"}` 才启用 | `…/subgraph` | `SubGraph` | **子图**（2026-10-12）：把**另一张图**当一步跑。契约 = 嵌套图的 `Input`（= 本节点参数端口，按 `name` 注入）/ `Output`（= 返回值，**2026-10-13 起可多个**：全按 `name` 排序返回、下游按下标取）。`sub` 是**内联 prompt、不是图名** —— 引擎宿主无关（图名解析是宿主的活），且增量指纹必须随内容变（只放名字会"内容改了还复用旧结果"）。深度上限 8、**按内容哈希**判成环、护栏接力（`active_guard()`，显式传入的 guard 不能被子图绕过）；`sub` 的指纹走**整体 sha1**（`_safe_repr` 会截断 → 会撞，见坑 14）；失败用 `SubGraphError` **原样上抛不套娃**（见坑 15）；子图内**不接**增量缓存（key 是节点 id，会撞外层）。**图名 → 内联 prompt** 走 `subgraph.resolve_graph_names`（节点类用 `GRAPH_NAME_PORT` 声明"哪个输入是图名、内联写进哪个键"，引擎仍不认目录 —— loader 是宿主给的；递归解开子图的子图，按**名字**拦环 / 拦深度，报错带节点路径 `GraphNameError`）。宿主侧（图名→内联 / 参数端口 / 编辑器 UI）与它的第一个用家 `TexFxMap` **已做**（2026-10-12）：逐条清单与实测判据见 `子图与FX-Map_计划.md`。**双击进内层图**（Substance 那种）：节点类声明 `GRAPH_SKELETON`（图文件模板）→ `get_node_info` 带成 `graph_skeleton`，宿主"双击没接图的节点"时照它建一张并接上（`SubGraph` 的模板是直通 `Input → Output`）。**图级 `root`**（2026-10-14，"根节点就是这张图的输出"，SD 的 FX-Map 那套）：节点类用 `ROOT_WRAP = (收口节点类名, (要搬过去的字面量参数…))` 声明"多出怎么收成一个值"（如 `FxQuadrant` → `("FxTile", ("width","height"))`）；`pygraph.graph_prompt` 里的 `_apply_root` 把它收成一个 `Output`（**单出**直接接第 0 个出，多出才用收口节点；`carry` 的字面量照搬、其余口按 0..N 接上）—— 于是"根"在引擎里仍然只是 `Input/Output` 那套，跑图 / 校验 / 子图内联 / A-B **全都自洽**，引擎**不认识任何具体节点类名**。`kind:"fx"` 的图**必须有 root**；坏 root（缺 / 指向不存在的节点 / 收口不了 / 类没注册）抛 `GraphRootError`（宿主映射成 400；自证第 29 组）。 |
| `…/code` | `PythonCode` | **python 节点**（2026-10-13）：在节点里写代码、**端口自己定**。取值 `inputs["a"]`、出值 `outputs["y"] = …`；端口表存 `ports`（JSON）。细节见下面的 §3.1.1 |


### 3.1.1 python 节点（`PythonCode`，2026-10-13）

**在图上直接写一小段 python** —— 别的节点都是"某个函数/方法 → 节点"（端口 = 签名），而一张图里
最常缺的恰恰是**胶水逻辑**（算个数、洗数据、拼字符串、试试写法）。为它建个包再注册一个函数，
代价远高于在节点里敲三行。

**三条契约**（改它们 = 换协议）：

1. **端口表面板定**：输入 `ports` 存一份 JSON（`{"inputs":[{"name","type"}],"outputs":[…]}`，
   短写法 `"a"` / `{"name":"a","type":"FLOAT"}` 都认）。引擎**不认类型名、不做转换** ——
   类型只给编辑器画端口/挑控件/上色用（图里流动的本来就是 python 对象）。
   节点类用 `PORT_SPEC_INPUT = "ports"` 声明"端口表存在哪个输入上"→ `get_node_info` 带出
   `port_spec_input`，编辑器据此把它当**动态端口**处理（前端不点名任何节点类）。
2. **`inputs` 读、`outputs` 写**：`inputs["a"]` 取输入端口，`outputs["y"] = …` 写输出端口。
   **刻意不用裸变量**：端口名是用户随手起的，塞进命名空间会吃掉 `print`/`json` 这类名字，
   也会让"这个变量哪来的"不可判。返回 tuple 按**端口表里 outputs 的顺序**拼（顺序错 = 下游取错值）。
3. **输出端口没赋值 → `None` + 一条 WARNING**（不抛）：条件分支里只给一支赋值是常态，
   抛异常会把能跑的图变成必炸；静默 None 又会被下游当真结果 —— 所以值给 None、日志里明说。

**看得见结果的两条**（都可不用）：代码里设 `ui = {"text": …}` → 节点卡片上出现「执行结果预览」
（与 `Preview` 同口径：`text`/`json`/`repr`/`image`/`audio`/`video`/`table`）；**`print` 会被收进
`ui.text`**（仅当没自己设 `ui`）—— 不然打印只落在服务端 stdout，编辑器里什么都看不见
（实测这条正是"我 print 了怎么没反应"的来源）。`ui` 里过不了 JSON 的值会退回 `str()` 截断
（要过 SSE，不能让一次打印把流冲爆）。

**编辑器侧**（`l_mindmap_mmd`）：卡片上给**只读高亮代码块**（prism 叠层，见 `PyCodeEditor.jsx`）+
「✎ 编辑代码」；**双击节点开编码浮层**（`PyCodeOverlay.jsx`：左边端口表 ＋输入/＋输出、右边大编辑器）。
`hidden` 容器的输入（`ports` 那行）画布与面板都**不画**、`code` 容器渲染成代码块 ——
这两条走 `container`，前端不写死字段名。canvas 后端画不了代码块，标题后缀标 `🐍 双击编辑代码`。
端口表节点**不问宿主 `/api/pygraphs/ports`**：端口表就在节点输入里，编辑器就地算
（`pySchema.portsOfNode` → 并进 `portsByNode`，形状与宿主那份一致）。

**安全**：**与宿主同权限**（完整 builtins、可 import、可读写文件）—— 与脚本编辑器 `:8764`、
`CallFunction` 同口径。**打开别人的图 = 跑别人的代码**（用户 2026-10-13 拍板；执行面本身还另有一道
`L_MINDMAP_ALLOW_RUN` 闸）。

### 3.2 3D 模型参数（`Input(type=model)`，2026-10-12）

值语义与 `image` **完全同构**（值是**路径**：上传后是服务端路径，手填本机路径也认）。四处配套：

| 环节 | 在哪 | 要点 |
|---|---|---|
| **函数 → 节点（批量）** | `l_node_engine/999.0/src/l_node_engine/from_functions.py` | 宿主侧那一半：`register_package_functions("我的包")` 一行把包里的公开函数全挂成节点 —— 只扫本模块定义的（不把 `os.path.join`/`lprint` 这类 import 进来的也包上）、端口按注解（没注解没默认值 → `*`+forceInput）、跳过私有/零参/denylist、**重名退成 `<模块名>.<函数名>` 不覆盖**、重扫幂等、过导入白名单、`async def` 也能跑；`describe_module_functions()` 只看不注册。自证 `tools/from_functions_check.py` |
| 目录 | `l_node_engine.pygraph.models_dir()` | `<图目录>/models`（与 `media/` 平级）；`model_url(path)` 出宿主相对 URL（`models/` 下 → 读口；本机路径 → 受控读口 `models-local?path=`）；**`model_path(value)` 给节点用** —— 绝对路径原样、相对名（含多文件 glTF 的 `<子目录>/<文件>`）补成 `models/` 下绝对路径 |
| 上传 | `l_mindmap_mmd/server.py` | `POST /api/pygraphs/models/upload?name=x.glb[&newdir=1\|&dir=<uuid8>]`：**原始 body 流式落盘**（不是 base64 —— 模型几十上百 MB，base64 再撑大 1/3 且全进内存）；白名单 `glb/gltf/obj/mtl/bin/fbx/stl/ply` + 贴图 `png/jpg/jpeg/webp`，上限 200MB，走 `L_MINDMAP_ALLOW_RUN` 门槛；**多文件**（glTF + `.bin` + 贴图）→ 服务端建一层 `<uuid8>/`，值取主文件的相对名 |
| 读取 | 同上 | `GET` **和 `HEAD`** `/api/pygraphs/models/{name:path}`（认 `<文件>` 与 `<服务端子目录>/<文件>`，其余形态 400、绝不拼客户端路径；HEAD 给前端探文件大小）；**落点顺序 = 数据目录 → 包内内置**（`<包>/src/l_mindmap_mmd/pygraphs/models/`，2026-10-13：内置模型随包分发，如 `torus.obj`）；**本机路径**走受控只读读口 `GET\|HEAD /api/pygraphs/models-local?path=<绝对路径>`（白名单根 `L_MINDMAP_MODEL_ROOTS` + 只认 3D 扩展名 + `resolve()` 后在根内 + 普通文件） |
| 清单 | 同上 | `GET /api/pygraphs/models` → `{models:[{name, bytes, builtin}]}`（数据目录 + 包内内置，同名数据目录优先）。编辑器当 `<datalist>` 用 —— 内置模型靠它被发现，不是下拉框 |
| 编辑器 | `l_mindmap_mmd` 的 `PyModelPicker.jsx` + `MeshViewer.jsx` | 值输入框（带 `<datalist>` 候选）+ 📁 上传（**可多选**）+ 👁 预览；预览是 three.js（**懒加载**，主包只 +24KB），认 glb/gltf/obj/**fbx**/stl/ply，OrbitControls 拖拽/滚轮/右键平移；**模型一律双面**（`forceDoubleSide`：单面薄壳从背面看是全空） |

- 工具 schema 里就是 `{"name": …, "type": "model"}`（`graph_tool_spec`）—— agent 看到"这是个 3D 模型参数"。
- **多文件 glTF**：外部 `.bin` / 贴图按**同目录相对路径**解析（GLTFLoader 以 `.gltf` 的 URL 为基准），
  所以主文件与伴随件必须在同一层（服务端就是这么放的）。
- 本机路径的 👁 不再是"置灰"：走受控读口，白名单根**外**时按钮 title 会明说"不在白名单根下（当前白名单：…）"；
  没配 `L_MINDMAP_MODEL_ROOTS` 时读口回 403（功能关着）。
- `.usd` 明确不做（要 USDZLoader + wasm，收益/成本不划算）；`.fbx` 已加（FBXLoader 是 three 自带 examples）。
- **把引擎跑在别的进程里（当宿主）**：契约 v1 + bottle 骨架见 `Rez-Docs/Rez_pkg/远端引擎契约_v1.md`
  —— 脑图编辑器（l_mindmap_mmd）可同时接**多个**远端引擎；图随编辑器走，宿主只提供 schema + 执行，
  对象留在宿主进程（图里用 `MakeInstance`/`CallMethod` 那族节点操作它）。
  本包的 `requires` 是 **`python-3.11+<3.13`**（刻意放宽）：宿主常常是别的解释器/别的 rez 环境
  （比如 py3.11 的 Qt 工具），内核零三方依赖，只 require 这一个包就够。
  要再加格式 = 上传白名单 + `MeshViewer` 里一个加载器分支（两处）。

### 3.3 `Input` 的值怎么存（全 python 类型，2026-10-12）

**标量存真标量，容器存 JSON 文本** —— 后者是被逼的，原因写死在这里免得后人改回去：

| 类型 | 存进图里的形状 | 编辑器控件 | 节点里拿到的 |
|---|---|---|---|
| `int` / `float` / `number` | JSON 数字（`3`） | 数字框 | `int` / `float` |
| `bool` | JSON 布尔（`true`） | 勾选框 | `bool` |
| `None` | `null` | 只读提示 | `None` |
| `str` / `Path` / 自由类型 | 字符串 | 文本框 | `str`（`Path` 交给下游自己 `Path()`） |
| `list` / `tuple` / `set` / `dict` / `object`（含 `list[int]`） | **JSON 文本字符串**（`'[1, 2]'`） | **结构化行**（`list` 类：一行一个元素；`dict`：键值行；可切回 JSON 文本视图；非法 JSON 退文本视图红框，但**照样写得进去**） | 真 `list`/`tuple`/`set`/`dict`（`Input.execute` → `coerce_param` 转） |
| `bytes` / `bytearray` | 字符串 | 文本框 + 📄 选文件（文件 → `base64:<…>`） | `b"…"`（文本按 utf-8；`base64:` 前缀**解码**成字节） |
| `image` / `model` | 路径字符串 | 选文件 / 上传（见 §3.2） | 路径字符串 |

⚠ **容器为什么不能直接存数组**：`Input.value` 的端口是 `*`，而"连线"的存储约定正是
`[源id, 下标]` 数组（`toPrompt` / `_resolve_inputs` 都按这个认）。存 `[1, 2]` 会被当成
"连到节点 1 的第 2 个输出" —— **值直接丢了**，画布上还看着像有连线。所以容器值以 JSON
文本入库、执行时 `coerce_param()` 再 parse 回真对象（图文件里一眼看是字符串、节点里是真对象，
这个不对称是刻意的）。

- 类型字段是 COMBO：下拉给常见类型，**点右边 ✎ 可手写任意表达式**（`list[int]` / `np.ndarray`）。
- 前端那张表在 `l_mindmap_mmd/web/src/pySchema.js`：`paramEditorOf` / `paramValueFromText` /
  `paramTextOf` / `paramJsonError` —— **有 node 单测**（`web/tests/pySchema.test.mjs`），改口径先改测试。

### 3.4 宿主侧的四个配套（引擎只保证"图能跑"，这四件全在宿主）

| 配套 | 引擎侧给了什么 | 宿主（`l_mindmap_mmd`）在哪 |
|---|---|---|
| **校验** | `validate_structure(prompt) -> {"errors", "warnings"}`，与执行器**同一套判据**（括号配对 / 嵌套限制 / 真环 / 缺必填 / 悬空连线） | `POST /api/pygraphs/{name}/validate`；编辑器「校验」按钮 = 本地快查 + 这条权威（结果进「问题面板」）。端到端盯着这条链的是探针场景 **`runflow`**（见 `tools/web_probe.md`） |
| **轨迹 / 回放 / 历史** | 跑图事件 `executed` / `executing` / `failed`（带 `if_taken` / `loop.iteration` / `cached`） | `runs.jsonl`（每次运行一行摘要）+ `_traces/<图名>.jsonl`（逐步）+ 编辑器「📈 轨迹 / 🕐 历史」两个面板 + **画布回放**（按步高亮，复用"正在执行"视觉） |
| **A/B 沙箱** | 同一张图喂两套 `Input` 值各跑一遍，只比**控制流**（值级比较没意义） | `POST /api/pygraphs/ab`；编辑器 🔬 A/B + 🧾 审计（`_traces/_applied.jsonl`） |
| **策略包** | 无（纯宿主资产） | 编辑器 `📦 包 → 导出/导入`：`{kind: "lugwit-pygraph-pack", version: 2, exported_at, title, graph, baseline?}`；导入**也吃裸图 JSON**、重名自动避让。纯函数 `web/src/pygraphPack.js`（+ `web/tests/pygraphPack.test.mjs`，`l_mindmap_mmd_web_test` 一起跑） |
| **工程（Project）** | 无（引擎只认"图名"，命名规则由宿主定） | **一个工程 = 图目录下一个大目录**，地址 `工程/资产`（P0，2026-10-13）：见 `l_mindmap_mmd.md`；引擎侧不受影响（`resolve_graph_names` 的 loader 仍由宿主给，地址怎么解析是宿主的事） |

### 3.5 端口元数据里的**标准类型**（`std_type` / `output_std`，2026-10-14）

**要解决的问题**：端口类型（ComfyUI 那套 `INT` / `STRING` / `*`…）**分不出容器种类** ——
按 `工具契约_ToolSpec.md` §5.2，`list` / `tuple` / `set` / `dict` / `object` 在端口上**全是 `*`**。
而"这到底是数组还是元组"编辑器要用来**给端口配色 + 画形状**（竖条 / 九点 / 花括号 / 双环 / 虚线圈…）：
**权威说明见契约 §5.4**（表格 + 每个环节的字段名）。引擎侧就三件事：

| 引擎侧 | 在哪 | 说明 |
|---|---|---|
| `stdlib` 的词表 | `introspect.std_of_annotation(ann) -> (标准类型, 可空)` | **唯一实现**：注解 → python 表达式；`Optional[X]`/`X \| None` 剥成 `X`；认不出回 `""`（= 不知道，不发这一格） |
| **输入**端口 | `introspect.params_from_callable()` 往 `opts` 里加 `std_type` | 按**参数注解**；函数 / 方法 / 类生成的端口都走这一条 |
| **输出**端口 | `callables._return_std()` → 生成类的 `RETURN_STD_TYPES` → `node_base.get_node_info()` 的 `output_std` | 按**返回注解**；`return_spec(func)` / `callmethod.method_return_spec(target, method)` 是给宿主 `/ports` 用的同源封装（带 `name`/`port`） |

⚠ 两条纪律：
1. **别在别处再写一份"注解 → 标准类型"的表**（原来 `tool_spec._std_of_annotation` 有一份，已并掉）；
2. **两处画端口的代码都要按它画**：DOM 后端 `PyNode.jsx`（`.pyg-handle.is-<shape>` + `style.css` 的 CSS 图案）
   与 canvas 后端 `CanvasSurface.jsx`（`drawPortMark()` 手绘）—— 只加一边等于没加。
   类型 → 颜色/形状的唯一表在 **`l_mindmap_mmd/web/src/portStyle.js`**。

自证：`l_node_engine_selftest`（输入 `std_type` / 返回 `return_spec` / 方法 `method_return_spec` 各组）；
端到端（**真浏览器看端口上的形状类**）目前**没有固定探针场景** —— 改完照 `l_mindmap_mmd/tools/web_probe.cjs`
那套（headless Edge + 独立 profile）自己写一段即可（口径见 `tools/web_probe.md`）。

## 四、坑（按"会咬人"排序）
1. **服务是 `.dev_mod` 但引擎不在它眼里**：改 `l_node_engine/*.py` 后，`l_mindmap_mmd` 不会自重启 —— 必须 `wuwo svc reload l_mindmap_mmd_server`，否则现象是"代码明明改了却没生效"。
2. **动态端口按节点问，不许改回"上次上下文"**：`CallMethod`/`CallFunction` 的 `INPUT_TYPES` 是**确定性**的（只有固定几项）。曾用"上一次执行过的 (target, method)"全局上下文展开端口，结果同一张图在不同会话/时刻画出的行都不一样。现在由编辑器按节点问 `GET /api/pygraphs/ports?target=&method=&kind=`（`kind=function` 走函数）。
3. **缺 handle 的边会被 React Flow 静默丢掉**：所以编辑器**预取所有** `CallMethod`/`CallFunction` 的端口（`portsKey` 只含 id+目标+方法，拖拽不重取），画布与面板都用 `mergedInputSpecs()` **同一份合并结果**。症状：打开保存过的图，"连线不见了"，点一下节点又回来。
4. **标志位是图的属性，不是节点类的**：`{"flags": {"bypass": true}}` 存在图里；`graph_prompt` / `toPrompt` / `toStorage` 都必须带 `flags`，否则"编辑器点了跳过、跑起来没跳过"。**bypass 有执行语义**：跳过该节点并把「按类声明顺序的第一个有值输入」透传给下游（无值给 `None` —— 空 tuple 会让下游按下标取时越界、报错更难懂）。bypass 的节点**不参与必填校验**。
5. **实例键语义**：`MakeInstance(target=裸路径, init_args={})` 注册的是**头部键**；带 `init_args` 注册的是**参数化键**（`pkg:Cls(tag="a")`）。`CallMethod(target=裸路径)` 优先命中头部键，否则取"最近登记的那个"。要确定性就**用确切的键**或走 `class_ref`。
6. **`instantiate(target, kwargs=…)` 不许吞构造参数**：`target` 恰好也是注册键时，旧实现直接返回旧对象、把 kwargs 丢掉（已修：只有"光秃秃要这个键"才复用）。`instantiate_from_class` 与它**同一套键算法** —— 否则 `class_ref` 与路径写法会各造一个对象。
7. **演练模式要在"执行点"拦**：按工具名匹配的名单（`DRY_RUN_BLOCKED`）拦不住运行时注册的工具，也拦不住图里的节点。pygraph 工具自己判 `in_dry_run()`。
8. **前端产物没有内容 hash**：模板里用 `?v={{ asset_ver() }}`（`main.js` 的 mtime）做 cache-busting。**症状：改了前端、重建了，页面却像没变 → 先怀疑浏览器缓存**（我自己被坑过一次）。
9. **超时只是"放弃等待"**：Python 杀不掉线程，节点里死循环会继续跑（超时事件文案里写明了）。SSE `abort` 只是前端不再读。
   ｜**2026-10-16 起**：执行默认仍串行，但标了 `PARALLEL = True` 的节点会被**成批并发**跑
   （连续的一批；批内按真实依赖分层、同层并发；`L_NODE_ENGINE_PARALLEL_MAX` 默认 8；
   走增量烹饪时不并发）。第一个用家是 `AIText`（fan-out 问多家模型）；worker 里用
   `contextvars.copy_context()` 跑，所以 `active_guard` / `active_emit` / `active_clock` 照旧可用。
10. **`Output` 的 `{"result","ui"}` 形状**：`ui` 是给宿主做**节点内联预览**的（前端按 ui 的**键名**选渲染器：`text`/`json`/`repr`/`value`）。`run_pygraph` 取 `result`；SSE 汇总也按 `result` 呈现，别把整只 dict 丢给前端。
11. **换函数/方法后，旧参数键会留在图的 `inputs` 里**：`_resolve_inputs`（`graph.py`）不按签名过滤，
   `CallFunction`/`CallMethod` 又把端口传来的键当 kwargs 打下去 —— 于是把 `path` 从
   `fs_nodes:list_dir` 改成 `os.path.exists` 会以 `takes at most 1 keyword argument (3 given)` 失败
   （残留的 `root`/`pattern`/`detail`），而画布上**看不见**它们（端口是按**新**签名展开的）。
12. **写类接口在前端只能"带会话"用**：闸门对本机直连只放行 `GET/HEAD`（见 `gate.py`）——
   页面免登录**不等于**接口免登录。自动化里要测上传，别在页面里模拟登录表单（本机直连的页面
   **不弹**登录框），正确做法是 `POST 127.0.0.1:1027/api/v1/auth/login` 拿 token，再给浏览器
   种 `lugwit_token` cookie（域 = 目标站）。
13. **`MeshViewer` 的加载器必须写成字面量 `import("…")` 分支**：用变量路径（`import(SPEC[kind])`，
   哪怕加 `@vite-ignore`）Vite 无法静态分析 → 运行时按 URL 去请求裸模块名 → **404**，症状是预览
   永远停在"载入中"。同理 FastAPI 的 `@app.get` **不带 HEAD**（`HEAD` 探大小会吃 405）——
   要 HEAD 就用 `@app.api_route(..., methods=["GET", "HEAD"])`。
   2026-10-12 修：两个节点执行前用 `introspect.keyword_names()` 只收**当前签名认的**键
   （带 `**kwargs` / 签名读不出 → 照旧全收），忽略的键打 WARNING —— 宿主挂了 `l_node_engine` 的
   WARNING handler，能在 `wuwo svc log l_mindmap_mmd_server` 里看到。
   ⚠ 副作用：参数名**打错**也只会被忽略 + 一条警告（不再报错），排错要看日志。
   另一条相关口径（**2026-10-12 已修**）：函数自己的参数名以前会撞节点的**固定输入名**
   （`path`/`kwargs_json`；`CallMethod` 还有 `target`/`method`），撞上就永远拿不到端口、只能走 `kwargs_json`
   —— 像 `os.listdir(path=…)` / `os.path.isdir(path=…)` 这类根本没法动态传参。现在固定输入改名为
   `func_path`/`func_args`（CallFunction）与 `obj_path`/`method_name`/`call_args`（CallMethod），
   **任意参数名都照原样展开**。老图不需要手工迁移：读图 / 跑图 / 存图都会把旧键映射成新键
   （`server.py` 的 `_migrate_legacy_inputs` + 引擎两个 `execute` 里的 `LEGACY_INPUTS` 兜底）。
   相关的第二条（**2026-10-12 一起补的**）：**位置专用参数**照常展开成端口，执行时按签名顺序绑到
   `*args`（`introspect.positional_names` + `objops`/`callmethod` 的 `func(*args, **merged)`）。
   没有这条时 `len(obj, /)` / `operator.sub(a, b, /)` / `glob` 之外的多数内置函数"端口有但调不动"
   （`takes no keyword arguments`），图里就**没法数长度、没法做算术、没法拼字符串** ——
   只能回头用引擎自己的 kwargs 友好函数。
   ⚠ 仍有表达不出来的：`os.path.join(a, *p)` 这类 **VAR_POSITIONAL**、`str.join` 这种要先有实例的
   （`self` 被跳过）、以及 f-string 式的字符串拼接（没有可用的 kwargs 版格式化）。

14. **节点自己拿不到宿主的 `emit`**（2026-10-12，做 `SubGraph` 时踩的）：`emit` 由执行器一路传给
   `_safe_emit`，`execute()` 里**根本没有这个名字**。第一版在 `SubGraph.execute` 里写了个引用
   `emit` 的闭包 → 被 `_safe_emit` 的 try/except **静默吞掉**：图跑完了、结果也对，日志里却一条
   子图进度都没有（最难查的一类）。要给节点一条"自己推事件"的路，用 `graph.active_emit()`
   （与 `active_guard()` 同形，都是按执行上下文放的 ContextVar）。
15. **`_safe_repr` 对 dict 会截断（20 项 × 40 字符）**：它是给 debug 载荷限长的，不是给指纹用的 ——
   一份**嵌套 prompt** 会被压成前几十个字符，**两份不同的子图会算出同一个指纹**，于是"改了子图却
   复用上一轮结果"。这类"整体语义"的字面量必须整体 sha1：`graph._HASH_LITERAL_INPUTS` +
   `_literal_fingerprint`（自证里有一条专门盯它：两份只差在 300 字之后的子图，指纹必须不同）。
   顺带：`SubGraphError` 往上传时**别层层包裹**（8 层嵌套各套一句"子图里有 N 个节点失败"，
   最外层就只剩一团乱码）—— 约定是子图自己的错**原样上抛**。
16. **增量的"括号节点不进缓存"漏了 `IfStart`（2026-10-12 修）**：第一版只把 `LoopStart/LoopEnd`
   排除了，`IfStart` 照旧命中缓存 —— 而"命中"只是把括号自己的旧值塞回来，**"跑区内节点 / 整段跳过"
   这一步根本没发生** → 区内节点既没产出、也没被写成 `(None,)` → **下游**读区内节点的那些节点
   连环报「上游节点 X 没有产出」（现象是整图 0.06s 秒退、答案空，报错全落在下游，根因在括号上）。
   现在 `bracket_nodes = 循环括号 ∪ if 括号` 一律不进缓存（区内节点本来就不在外层 `order` 里、
   没有指纹，所以等价于"这一区每轮真跑"）。self-test：`_check_incremental` 里那张带 if 的图
   **连跑两次**（真/假两支都测）。教训与坑 14 同类：**判据要落在"这一轮到底执行了什么"上**，
   只看"输出对不对"会把这种"该跑没跑"的错放过去。
17. **`INPUT_TYPES` 两种写法都要接得住（2026-10-12 修）**：引擎**支持**「类方法」与「类属性 dict」两种
   （`node_base.get_node_info` 就是这么读的；`LoopStart/LoopEnd`、自检里的假节点用的都是 dict），
   但 `validate.py` 原来写死了 `cls.INPUT_TYPES()` → 循环括号一律吃 `bad_spec`
   （`'dict' object is not callable` 警告），**顺带把它们的必填检查整段跳过** —— 而"循环区接线对不对"
   正是最该被校验的一类（做 `flow→pygraph` 镜像时踩到）。已改成两种都读。
   校验与执行器**同判据**是这张表的既定原则，别再分叉：self-test `_check_validate` 里加了一张带循环的图，
   断言 `errors` 与 `warnings` **都为空**。
18. **`StateIn.default` / `Input.value` 是"JSON 文本"口径**（2026-10-12 写跨轮图时踩到）：
   `StateIn` 把 `default` **按 JSON 解析**（`parse_state_text`），所以写 `[]` / `{}` 得到 list / dict
   （多数场合正是本意），但下游若是**只吃字符串**的节点（`operator:concat`、`json:loads` 以外的文本拼接）
   就会当场炸 —— 实测 `can only concatenate str (not "list") to str`（报错落在**下游**节点上，
   根因在 `StateIn` 的默认值）。要"空字符串"就写 `""`，要"空列表"就写 `[]` 并让它只喂给吃容器的节点。
   `Input` 的容器值同理（存 JSON 文本、执行时 parse 回真对象，见 §3.3）。

19. **python 节点的三处"想当然"**（2026-10-13 做 `PythonCode` 时逐个踩/提前堵）：
   ① **代码与端口表必须进整体 sha1**：两者都是长文本，而 `_safe_repr` 只留前 60 字符 ——
   不登记到 `_HASH_LITERAL_INPUTS` 的话，"改了两行同前缀的代码"指纹不变 → **跑出来还是旧的（缓存）结果**
   （同坑 15 的子图那条；自证里专门有一段"前 60 字符完全相同的两段代码必须算出不同指纹"）；
   ② **`ports` 读不动要当场报错，不能当"没有端口"**：半截 JSON 读成空表会让所有输出静默变 None
   （跑得通、结果全错）。所以 `parse_port_spec` 对**非空但坏**的值抛 `ValueError`，校验器与执行器
   **共用这一份**（`validate_structure` 报 `bad_ports`，跑图时是节点错误）；编辑器写回前先校验 JSON，
   所以只有"手改图文件"才会看到它；
   ③ **`contextlib.redirect_stdout` 换的是进程级 `sys.stdout`**：宿主若并行跑多张图，几段 `print`
   可能互相串一点（窗口只有 exec 那段时间）。换取的是"节点里的 `print` 在卡片上看得见"——值这个价；
   真要干净的输出就别并行跑同一进程（或让节点用 `ui` 自己给预览）。

## 五、怎么自证（改完跑什么）

| 改了什么 | 跑什么 |
|---|---|
`l_node_engine/**` | `wuwor l_node_engine -- l_node_engine_selftest`（**27 组**；新增能力请加组） |
`PythonCode` / 端口表（`node_base.PORT_SPEC_INPUT`、`code_nodes.py`） | selftest 第 27 组（端口表解析/坏值、多输出顺序、未赋值端口给 None、print→`ui`、**改代码触发重算**、校验器同判据）；编辑器侧 `wuwor l_mindmap_mmd -- node rez-package-source/l_mindmap_mmd/999.0/tools/web_probe.cjs pynode`（15 项：卡片高亮 / `hidden` 不画 / 双击浮层 / ＋端口与改码**落盘** / 跑出 `← 2: [35]`） |
`web/src/pySchema.js`、校验/映射口径 | `wuwor l_mindmap_mmd -- l_mindmap_mmd_web_test`（62 项） |
`l_node_engine/ai_nodes.py`、AI 路由/载荷/凭据 | selftest 的 AI 组（验路由表 + 载荷 + 落盘 + ai.json 通道 + 凭据三档/过期重登/401 换一枚重试一次）；**再核对一次 hub 的 `GET :8462/openapi.json`** —— 路由表是抄出来的，不是推出来的；带真凭据的端到端见 §3.0 第 4 条（本机 dev 账号，别拿去公网机） |
`web/src/*.jsx`（画布/面板/预览/框/错误卡/历史/AI 配置） | 重建 + 浏览器点一遍（FPS 读数、预览、标志位、实时模式、分组框、循环区包裹框、问题面板、历史、AI 服务） |
`l_agent_tool/pygraph_tools.py`、agent 接线 | `wuwor l_agent_chat -- python -c "...loadTestsFromNames(['tests.test_pygraph_tools'])"`（14 项） |
改图/新增 demo | `curl -X PUT …/api/pygraphs/<图名>` + `POST …/<图名>/run` 看 `node_errors == {}` |

现成示例图（`~/.lugwit/main/l_mindmap_mmd/pygraphs/`）：**`demo_all_nodes`**（用到全部 16 种节点 + 一个 bypass + 4 个 Preview）、**`demo_ai_image`**（**生图样板**：中文需求 → `AIText` 改写成英文提示词 → `AIImage` 出图 → 两个 `Preview`（提示词 / 图）+ `Output`；`provider` 是个 `Input`，默认 `zhipu` = 智谱 **CogView-3-Flash（免费模型）**，改成 `free` 只用免费档、留空走自动链 —— 免费档与坑见 `l_model_hub.md`「图片生成」）、`demo_objops`、`demo_tool_add`、`t2`。

> ⚠ 跑任何**带 AI 节点**的图之前，先在右侧「AI 服务 ▸」里配好模型中心凭据（lugwit 账号密码，或接入密钥）：
> 节点是在**服务进程里**打模型中心的，拿不到浏览器的登录态（见 §3.0.1）。

**AI 节点的「模型」下拉**（候选 = 模型中心清单里**该能力**的那批，见 `ai_nodes.model_spec`）：

- 候选**全部来自接口**（`/api/pygraphs/ai-models`，按 `tags` 筛能力），文生图节点里不会列出只会聊天的模型；
- ⛔ **别在节点声明里塞"默认模型"当候选**：控件会把"当前值"补进选项列表，于是界面上冒出一个
  **接口从没返回过**的取值，看着就像候选是硬编码的（`default="auto"` / `default="whisper-1"` 都踩过，
  2026-10-12 已改成**空默认**）。要 `auto` 就走下拉右侧的 ✎ 自由输入。
- 选中的值会真的发给模型中心（`厂商/模型`）；**留空 = 不指定**，由上游自己挑（模型中心走降级链，
  各家出图/语音用自家默认常量）。
- ⚠ **「（不可用）」徽章 = 探过且确实不可用**（`probed && !available`），**不是**"没探过"：
  生图 / 视频 / 语音这几类**根本不做探活**（探一次要真出一张图/一段音频），永远是
  `available=false, probed=false`。只看 `available` 会把整张 image 列表染红 —— 2026-10-12 用户
  当场问"为啥全部标注不可用"（而那几个模型刚跑出过图）。所以宿主 `/api/pygraphs/ai-models`
  必须把 `probed` 一起透出去（`server.py`），前端在 `valueSources.js` 里只按 `probed && !available` 加后缀；
  上游没给 `probed` 就**不标**（宁可不评价，也别把没验证过的说成坏的）。
- 示例图 `demo_ai_image` 已经把两个模型都**显式钉住**：`zhipu/glm-4-flash` + `zhipu/cogview-3-flash`（都免费）。
- **`AIVideo.provider` 也吃同一套**（2026-10-15）：声明见 `ai_nodes.provider_spec()`，候选来自
  宿主 `/api/pygraphs/video-providers`（转模型中心 `/v1/video/models`）—— 那个接口**把六家厂商
  塞进 `models` 形状**（`key` = provider id、`tags:["video"]`、`available` = 配没配钥匙），
  于是 `pick/value/where` 这套前端机制（`valueSources.js`）**一行都不用改**就渲染成下拉。
  同样**不许有默认值**（自证里与模型那条一起钉住）。

## 六、已知未做 / 限制

- **跑图的墙钟超时**：默认 = 服务端 env `L_MINDMAP_RUN_TIMEOUT`（**60s**）；
  **2026-10-12 起支持 per-request 覆盖** —— 请求体给 `{"timeout": N}`（夹 1..3600）就用它
  （`_run_timeout`）。要跑长的**别改服务 env**（那是全服务口径），在这一次请求里给就行。
  撞上超时的表现：宿主发一条 `error`（`执行超时（Ns）。工作线程无法强杀…`）—— 脚本/前端**很容易漏看**，
  现场就只剩"耗时正好 N 秒 + 输出空"（实测踩过）。⚠ 另：**超时常常是症状不是病**：
  把整棵树的路径都塞进提示词那种"题目太大"，调到 300s 也回不来（实测）——先看提示词有多大。
- **map / 遍历：2026-10-18 起有了**（这条原先写"没有"，是旧口径）：`Map`（跑**子图**模板，
  `items/template/item_param/shared/parallel` → `results/errors/ok_count`）、`ForEach`（串行）、
  `ForEachConcurrent`（线程池，默认开满 8）、`ForEachThread`（**每项一个新线程**，
  上限 `L_NODE_ENGINE_THREAD_MAX` 默认 64）、`ForEachProcess`（**每项一个子进程**，
  `L_NODE_ENGINE_PROCESS_MAX` 默认 8；跨进程只收结果、参数必须可 pickle）。
  另有 `Calc`/`Count`/`Sum`/`GroupCount`/`ToText`、"跑到条件满足"的 `LoopStart(until_stop=True)`、
  多路**值**选择 `Select`、多路**控制流** `Branch`、`else` 三件套。
  **两个静默跑歪的防线（2026-10-20）**：
  ① `item_param` 打错字（内层 `GetParam` 读不到就是 `None`，每项结果全空且不报错）→ 现在
     内层**真声明过参数**却对不上时当场 `ValueError`（报错里列出它声明了哪些名字）；
     内层一个参数都没声明的（不需要 item）照旧放行。
  ② **全局线程预算** `L_NODE_ENGINE_THREAD_TOTAL`（默认 32，`thread_budget.py`）：三处并发
     —— `Map`/foreach 的逐项线程、`LoopStart.parallel` 的波次、并发批的同层 worker ——
     **共用一本账**（以前各守自己上限，乘起来能到 8×8=64）。拿不到名额就**就地串行**
     （安全优先）；`map_node.active_threads()` / `l_node_engine.active_threads()` 可看占用。
     自证：预算掐到 1 时循环波次与并发批都降级串行、结果正确、跑完归零。
  **仍没有**：`filter`（拿 `Map` + `IfStart` 筛，或 `PythonCode` 一行 list comprehension）、
  一等"自定义函数节点"（单节点胶水用 `PythonCode`；要复用就抽子图，或用「函数节点」插件
  `functions_plugin.py` 把某个包的函数批量接成节点）。
- **循环 / if 的嵌套边界（2026-10-20 复核）**：`if` 套 `if`、`if` 在循环体里、**循环套循环都支持**
  （自证里是 2×3、三层 2×2×2、外层 `parallel=2` 里套内层）；**深度没有上限**（`_run_loop` 递归、
  区解析是扁平的）。**仍明确拒**：if 与循环区**跨边界**（一个括号在里一个在外）、
  内层两个循环括号不都在外层区里、`LoopEnd.state` 与 `LoopStart.parallel>1` 同用 —— 全都报
  `GraphPlanError` / 明确错误，不静默跑错。
  ⚠ 接线坑（踩过两次）：内层 `LoopStart.state` 接外层 `state` 时，**外层自己也要有 `state` 来源**，
  否则内层首轮拿到 `None`（`Calc` 会明确报"别拿它当 0"，不静默算 0）。
- **`node_errors` 的并发记账（2026-10-20 修）**：并行轮次里同一个节点失败，旧写法是"先 `get` 再 `set`"
  —— 两个线程都看到 `None` 就各建一份，后写的把前一份丢了。现在统一走 `_record_error()`
  （模块级锁 + **追加**），失败条数不丢（判据：`parallel=2` 里同一个节点两轮都失败 → 至少 2 条）。
- **`if` 区放进循环体：** ~~不支持~~ → **2026-10-12 起支持**（智能体的循环体里必然有分支，
  这条不放开，"把控制流搬进图"就堵死）。实现要点（改这块先看这三条，都踩过）：
  ① **规划顺序** —— if 配对要先算，循环区靠它把括号 + **条件那一段**补进区里
  （只看"可达 ∩ 能到 LoopEnd"，`IfStart` 与它的 `Compare` 会漏在区外 → 假支会被当成真支）；
  ② `_run_loop` 每轮要 **dispatch 括号**，`executed` 集每轮新开并预置循环括号/区外上游
  （不预置会 `_run_if → upstream → LoopStart → _run_loop` **递归爆栈**）；
  ③ 区内的普通节点**要等它的括号跑完**再走（拓扑序里它可能排在括号前面）。
  嵌套自 2026-10-13 起支持（`if` 套 `if`、`if` 在循环体里都行）；**仍拒：if 与循环区跨边界**
  （一个括号在里一个在外）。自证：第 22 / 24 组。
- **多路分支 / 护栏（guard）—— 2026-10-14 两块原语都补齐了**：flow 的 `branch`（多 case）= 图上的
  `Branch`（N 个 `{条件→目标}`，只跑命中的那条臂）；flow 的 `gate`（具名 guard）= 图上的 `Gate`
  （引擎出注册表 `register_gate_guard`、判据由宿主 `l_agent_chat` 注册，`allow`/`triggered` 两个出口）。
  这两块是"把智能体控制流搬进图"的最后两个硬缺口（原先只能条件节点 + 逐条 if 展开、护栏只能留在宿主）。
- **子图（引擎 + 宿主都已做，2026-10-12）**：`SubGraph` 能跑（自证第 21 组）；宿主 `l_mindmap_mmd`
  在跑图 / 校验前把 `{graph: "图名"}` 内联成 `sub`（`_inline_subgraphs` → 引擎的 `resolve_graph_names`，
  解析不了直接 400）、`GET /api/pygraphs/ports?kind=subgraph&target=<图名>` 给参数端口、
  `/object_info` 把 `GRAPH_NAME_PORT` 那个输入换成**图名下拉**、编辑器卡片上有「↗ 子图」跳转。
- **python 节点（`PythonCode`，2026-10-13 已做）**的边界（见 §3.1.1）：代码在**宿主进程**里跑
  （与宿主同权限）；**canvas 后端只给标题后缀**（画不了高亮代码块，DOM 后端才有代码块）；
  代码块内没有自动补全/跳定义（就是 textarea + prism 高亮 + 行号 + Tab/回车缩进）；
  端口名要能当 `inputs["名字"]` 用（标识符），浮层里对非法名给了⚠但**不拦**（引擎侧按名字取值，
  名字非法只会取不到）。（"自己写个函数节点"这条硬缺口由它就**部分**补上了：单节点的胶水逻辑够用，
  复用仍要子图/宿主函数 —— 见上一条。）
  **多输出：2026-10-13 起支持** —— 嵌套图的 `Output` **全部按 `name` 排序返回**（无 `name` 用 `out<节点id>`；
  排序同 `collect_outputs`：先 `name`、再 id 的**数字**大小），下游按**下标**取（`["sub", 1]` = 第二个）；
  宿主/编辑器要列端口用 `list_graph_outputs(prompt)`（**不跑图**，顺序与运行期一致），
  宿主的 `/api/pygraphs/ports?kind=subgraph&target=<图>` 已带结构化 `outputs`。
  **画布早就是多端口了**（2026-10-20 复核：`pySchema.outputHandles` 按 `info.output` 逐个画
  `out{i}` 句柄，端口表节点用 `portsOfNode` 的 `outputs` 覆盖；DOM 与 canvas 两个后端都走它）——
  以前这里写"编辑器只画一个端口"是旧口径，已作废。
- **代码节点准入门（2026-10-20，优化清单 #11）**：`PythonCode` 与宿主**同权限**（能 import、能读写文件），
  所以至少给了两道闸：`L_NODE_ENGINE_ALLOW_CODE=0` 直接关掉这一类节点（明确报错，不静默跳过）；
  宿主可以用 `set_code_guard(fn)` 挂审批钩子（`fn(code)` 抛异常 = 拒这段代码）。
  **默认全放**（老图/老行为不变）—— 这是"能不能拦"，不是"默认拦"。⚠ 它**不是沙箱**。
- **旧内核边界（2026-10-20，优化清单 #7）**：`l_comfyui_backend/prompt_executor.py` 与我们同源但更旧，
  已在它的模块头标 **LEGACY**：引擎语义（拓扑/循环/缓存/并发/事件）只改 `l_node_engine`，
  那边只接 ComfyUI 上游兼容。自证里有一条 `_check_no_legacy_import` 断言**引擎源码里不许出现
  `comfyui_backend` 的 import**（防止两套语义被反向绑死）。
- **子图接增量缓存（2026-10-20）**：以前整段不缓存（缓存键是节点 id，内外层都叫 "1" 会撞）。
  现在 `graph.active_cache()` 把宿主那份递进节点，`subgraph._ScopedCache` 给内层键加命名空间：
  `sub:<外层节点id>:<内层内容哈希>/<内层节点id>` —— 同一个调用点内容没变才命中，子图一改前缀就变。
  ⚠ 外层那个 `SubGraph` 节点自己也缓存 ⇒ 它命中时内层根本不会跑（正常现象，别拿"没看到内层 cached
  事件"当"内层没接上"）。自证：键带前缀 / 代理四条方法透传 / 内容一改不命中。
- 队列只有 FIFO（没有优先级）；**中止**：`/runs/<id>/cancel` 能在**节点边界**停（`GraphCancelled`，
  2026-10-20），但正在执行的那个节点打断不了 —— 节点内部的长循环要自己轮询 `graph.cancel_requested()`。
- **断点 / 单步（2026-10-20）**：引擎侧只有 `execute_graph(pause=…)`（每个节点执行前调一次
  `pause(node_id)`；钩子想停就阻塞、想中止就抛 `GraphCancelled`，钩子自己抛别的只记日志）。
  "断点"是**宿主的语义**：`POST /run` 带 `pause_mode`（`breakpoints` = 只停挂了
  `nodes[id].flags.breakpoint` 的节点 / `every` = 每个节点都停），命中了发 SSE `paused`
  并阻塞，前端用 `POST /runs/<id>/resume {"action": "continue|step|stop"}` 放行。
  ⚠ 命中缓存的节点原本不执行（`_run_node` 走不到）→ 缓存命中那一支也补了一次 `_maybe_pause`，
  否则"断点在那儿但图被增量缓存跳过"= 永远不停。
  ⚠ 暂停期间**流不判超时**（收到 `paused` 就把墙钟预算重置一轮），否则人还在看断点就被判超时。
- **规划缓存（2026-10-20）**：`_plan_graph` 按**结构哈希**（id + `class_type` + 连线引用）缓存
  规划结果（LRU 32），实时模式/时间线那种"同图反复跑"不再每次重算 4 遍全图；
  改字面量参数不换键（规划只吃拓扑），换连线才重算。`register_node` 会顺手清缓存（括号角色会变），
  `L_NODE_ENGINE_PLAN_CACHE=0` 可关，`plan_cache_info()` 看现状。
- 分组框是**视觉框**（不做父子绑定、成员不被裁剪），但**拖框会带上框里的节点**（成员起拖时现算；2026-10-16 起）。循环区包裹框仍是自动生成、不可交互。
- **hub 的视频那条已真跑（2026-10-15）；图片那条仍未测**。LLM 那条**已验证**（只用 lugwit 账号密码，见 §3.0 第 4 条）。
  视频实测：`AIVideo`（真 hub、真上游）→
  产物 2.38 MB MP4 落 `<pygraphs>/media/`，`ui.note` 记了 `minimax · 6s · minimax:<task_id>`；
  走的是 **MiniMax 海螺**（hub 侧自动挑中"有钥匙且真能出片"的那家，见 `l_model_hub.md`「视频」）。
  仍未测：出图能不能成、首帧 data URL 上游认不认（海螺的首帧要**公网 URL**，data URL 大概率被拒）。
- **`AIVideo` 是异步的（2026-10-15 改）**：提交 `/video/task` 拿 `task_id` → 每 5s 轮 `/video/task/{id}` →
  按 `succeeded/failed` 收尾（上限 `max(单次超时×5, 900s)`）。为什么：出片 1~3 分钟，
  原先"一个 POST 死等 300s"既会把整张图卡死，又在中途给不出任何信号。
  **进度**走 `graph.active_emit()` 推 `executing` + `note`（宿主把 emit 原样转发进 SSE；事件名必须是
  宿主/前端本来认的那套，自定义名推了也没人渲染 —— 与 Ask 节点同一通道、同一 `current_node()` 取 id）。
  前端（`l_mindmap_mmd/web/src/PyGraphEditor.jsx`）把它渲染成卡片上一行 `⏳ …（已等 Ns）`
  （`.pyg-node-runnote`），流水里只写**变化**的那条；对话驱动那一档经 `GET /api/pygraph-live`
  （agent 侧白名单已含 `note` / `progress`）。
  ⚠ 两个配套：**宿主墙钟超时**默认 60s，出片必被掐 —— 编辑器现在按图里的慢节点算
  `runTimeoutFor()`（`AIVideo` 420s/个）随请求带上，见 `l_mindmap_mmd.md`；
  节点内**仍是同步阻塞**（一次 `execute` 不能中途 yield），所以"真正的百分比进度"没有
  （上游不给百分比，只有状态）—— 现在能看到的是状态 + 已等待秒数。
  配了 `L_NODE_AI_VIDEO_URL`（自建服务）时退回**同步**路径（那边没有 `/video/task`）。
  **建任务有原地重试**（2026-10-15 补，**2026-10-19 改成"次数是节点参数"**）：免费档常态是
  `503 队列已满` / `429 速率限制`，撞上这类"上游忙"的错误时自己重试；**最多试几次 = 节点入参
  `retries`**（INT，默认 **50**，可调 1~200 —— 用户口径「节点本身就能无限重试，最多 50 次，
  并且把重试次数作为这个节点的参数」）。间隔为 `CREATE_RETRY_DELAYS = (20, 60, 120)` 秒、
  **第 4 次起都按 120s 封顶**；**上游给了 `Retry-After` 就按它睡**（夹 5~180s，来源写进进度行：
  `… （7/50 · 上游 Retry-After）` / `… （2/50 · 指数退避）`）。
  为什么这样：固定 20s 会自己撞进 429 的分钟窗，越撞越久（2026-10-19 实测 4 拍只成 2 拍）。
  `Retry-After` / `X-RateLimit-*` 由 hub `/video/task` 从**上游响应头**转进错误 JSON
  （`retry_after` / `rate_headers`；`rate_headers` 只进日志、不参与调度）；
  **只对"上游忙"重试** —— 配置错（没钥匙/参数错）重试一万次也一样，直接抛。
  ⚠ 节点里是**同步睡**：50 次 × 120s 最坏约 1.6 小时，**宿主墙钟超时会先到**
  （编辑器按 `retries` 把 `AIVideo` 预算加到 `900 + (retries-1)*125` 秒，仍被宿主 3600s 夹住）。
  **横向参数（2026-10-19 第一批）**：`aspect_ratio` / `negative_prompt` / `seed`（-1 = 不表态）/
  `fps`（0 = 不表态）/ `count`（1 = 不表态，目前没有一家认 —— Agnes 只收 `n=1`）/ `with_audio` /
  `enhance_prompt`。**默认全不表态** ⇒ 只有填了才写进 `/video/task`；认不认由各适配器决定
  （映射表见 `l_model_hub.md` 视频那节）。
  **第二批（同日）**：`image_tail`（尾帧，IMAGE）/ `reference_images`（参考图，每行一个 URL 或本机路径）/
  `quality`（`std`/`pro`/`4k`，可灵用）/ `provider_options`（**厂商私有字段 JSON**，原样并进上游请求体 ——
  多镜头、动作控制、参考视频这类没做成一等参数的走这里；坏 JSON 直接报错，不静默丢）
  / `multi_shot`（**多镜头分镜**，每行 `秒数|提示词`，最多 6 段、总时长 3~15s；只 kling-v3 认，
  别家明确报错；设了它根提示词失效）。
  **置灰（2026-10-19 用户口径「切模型时不支持的参数自动隐藏么」→ 选"置灰"）**：卡片按
  hub `supports` 能力表把**当前 model/provider 不认**的那几行标灰（`web/src/videoCaps.js`，
  行加 `is-unsupported`、控件加 `is-muted`、悬停写"当前模型不认这个参数"）。**不隐藏也不禁用**：
  值照旧能填、端口与接线不动（切回认它的模型就生效）。只对**能力参数**生效
  （`CAP_KEYS`），`prompt`/`model`/`provider`/`image`/`seconds`/`resolution`/`retries` 永远不灰。
  自证：`l_node_engine_selftest` 的 AI 节点组（建任务载荷 / 轮询 URL 编码 / 落盘 / provider·resolution 透传 /
  非 hub 走同步）。
- **HTTP 错误现在带响应体**（2026-10-15 改）：以前 `urllib` 只给一句 `HTTP Error 502: Bad Gateway`，
  而真正的原因在 body 里（模型中心会把上游原文转出来，如
  `Agnes 创建任务失败：HTTP 503 视频队列已满，请稍后重试`）—— 画布上只看到"Bad Gateway"时
  排障得翻服务日志。现在 `_post_raw` 把 body 里的 `detail`/`error`/`message` 提出来拼进异常
  （`HTTP 502：…`，截断 400 字）。**所有 AI 节点都受益**（LLM / 图 / 视频 / 语音同一条出口）。
- **STT 没有标准形状**：hub 不提供，节点发的是"任意本机服务的通用形状"（`audio_b64` + `filename`）——
  接自己的服务时可能要改。
- 图表 / PDF 等其余预览类型没做（图片/音频/视频已做：产物落媒体目录 + `ui.{image,audio,video}` + 宿主只有
  单文件名的媒体路由，别放宽那个安全边界）。**3D 模型预览是宿主的事**（`l_mindmap_mmd` 的 `MeshViewer`，
  2026-10-12 起还会把节点图产出的 PBR 材质贴上去，见 `l_tex_nodes.md` §6.3）—— 引擎侧不掺和。
- 折叠 / pin 的其余交互（节点标题栏折叠）没做；`bookmark` 是纯标记（没做筛选/跳转）。
- 前端只有协议层单测（`pySchema`），组件层没有自动化。
- `l_comfyui_backend` 里还躺着一份同源但更旧的执行内核（`node_base`/`node_registry`/`prompt_executor`），尚未接到 `l_node_engine`，两处口径可能漂。
- ⚠ **写前端单测的坑**：`web/src` 里相对导入**不带扩展名**（Vite 能解析，Node ESM 不能）。所以被测试的模块若引了同级文件，必须写成 `"./x.js"`（如 `pySchema.js` 引 `flowSchema.js`）——否则 `node --test` 报 `ERR_MODULE_NOT_FOUND`。
