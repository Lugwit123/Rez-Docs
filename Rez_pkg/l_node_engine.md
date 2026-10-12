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

其它旋钮：`L_MINDMAP_RUN_TIMEOUT`（墙钟超时，默认 60s；**2026-10-12 起可被请求体的 `{"timeout": N}` 覆盖**）、留痕 `~/.lugwit/main/l_mindmap_mmd/pygraphs/runs.jsonl`。
**别在公网部署机上开这个开关，也别把 `--host` 改回 `0.0.0.0`。**

### 2.4 导入白名单（"能 import 哪些包"）

默认**空** = 只放行 `l_node_engine` 自身 + 已注册节点所在模块。放行外部包三条路（优先级从高到低）：

1. 环境变量 `L_NODE_ENGINE_ALLOW_PKGS=json;mypkg`（**设了就以它为准**）
2. 文件 `~/.lugwit/main/l_mindmap_mmd/pygraphs/allow.json`：`{"packages": ["json", "mypkg"]}`（卡片只能带参数带不了 env，才有这一档；改完**立即生效**，`allowed_packages()` 每次调用都重读）
3. 卡片启动参数：`run_args: ["l_mindmap_mmd_server_exec", "--allow-import", "json;mypkg"]`

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
`鼠标手势` | 左键 = **点选 / 框选**（**不平移**）；中键 / 右键**拖** = 平移；右键**单击** = 右键菜单；滚轮 = 缩放。参数（`selectionOnDrag` + `panOnDrag={[1, 2]}`）**与流程图编辑器逐字一致** —— 两套画布手感必须一样；React Flow 默认是"左键拖 = 平移"，与节点编辑器的肌肉记忆相反（用户口径 2026-10-12） | — |
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

### 2.8 界面分区 = 可搬的标签条（2026-10-12）

`/pygraph` 的界面是**三块可搬的标签面板**：左栏（🧩 节点库）、右栏（🔧 属性 / 🤖 AI 服务 / ⚠ 问题 /
🔬 A/B / 🧾 审计 / 📈 轨迹 / 🕐 历史）、底栏（📋 运行日志 / 🧊 3D 视图）。引擎在
`web/src/pyTabs.js`（模型 + 落盘 + 尺寸拖拽 hook）与 `web/src/PyTabStrip.jsx`（标签条 + 面板壳）。

⚠ 加标签的两处必改：`pyTabs.TAB_DEFS` 登记（`home` 决定默认落哪个面板，旧存档会自动补进来）；
面板体里**每个标签都要 `tabs.isActive(panel, key)` 守卫**（面板的 `children` 是全量渲染的，
不守卫就会出现"切成 A 标签、B 标签还挂在下面"）。`🧊 3D 视图` 是这么加的（`Viewer3DPanel` →
`MeshViewer docked`，与节点卡片上的 👁 同一份渲染）。

**交互**（与 `lugwit_baidu_netdisk` 的 `web_depot.html`「标签引擎」同款 —— 那边是原生 JS 搬 DOM 节点，
这里是 React 按归属渲染）：

| 操作 | 效果 |
|---|---|
| 拖动标签 | 本面板内 = **排序**；拖到别的面板 = **连同内容搬过去**（落点有 3px 指示条；拖到边上自动横向滚） |
| 标签条末尾 `＋` | 列出**别的面板**的标签，选一个就搬到本面板末尾并切过去（等于"拖过来"的鼠标版） |
| 拖缝 | 左↔画布、画布↔右栏两条竖缝改宽，底栏上方横缝改高；**双击复位**（左 210 / 右 300 / 底 150） |
| 面板被搬空 | 自动收成 **26px**（标签条藏起来，只剩 `＋` 当落点 —— 搬回来即可复原） |
| 底栏「收起/展开」 | = 那个面板的折叠（只留标签条）；回放时会自动展开（不然回放条看不见） |
| 刷新 | 归属 / 每个面板的活动标签 / 尺寸 / 折叠 **全存 `localStorage["pygraph.ui"]`**（防抖 300ms + `pagehide` 补一记） |

三条**有意**的分歧（别按 depot 改回去）：

1. **画布、顶栏、2D 取景器不参与**标签系统 —— 画布是 workspace，`.pyg-center` 的 `grid-area`
   绑定（`has-viewer` 那套）与 `CanvasRulers`/`overlay` 的 props 契约都绑死在它身上。
2. **「人在环」的询问条（Ask）挂在工具栏正下方**，不进任何标签体 —— 跑到一半要人回答时，那条要是
   被切到看不见的标签里（或底栏折着），图就一直挂着等人。
3. **切标签/搬面板 = 内容卸载重挂**（depot 是搬 DOM 节点、内容不重挂）。这里所有业务状态都在
   `PyGraphEditor` 的 `useState`（受控输入 / `aiDraft` / `log` / `history`…），所以重挂不丢数据；
   会丢的只有**非受控** DOM 状态（`<details>` 展开、滚动位置）。

⚠ 四个踩过的坑：① 存档键名要对上（写 `order`、读 `tabs` → **刷新后布局悄悄回默认**，最难从界面看出来）；
② 标签条**必须能横向滚**（右栏 300px 塞 7 个标签总宽 ~470px，`overflow:hidden` 时后面几个
**鼠标够不到**、拖不动 —— 实测拖 `history` 拖起来的是别的标签）；③ `.pyg-tabhead` **不能** `overflow:hidden`
（末尾「＋」的下拉是它的 `absolute + top:26px` 子元素，一裁就整块看不见 —— "点了没反应"，
而 DOM 里查得到：**用 `querySelector` 的测试会漏掉这类"渲染了但被裁掉"**，要么真点、要么查 `elementFromPoint`）；
④ 「＋」菜单的关闭**不能用 `stopPropagation` 拦**（一页三个 ＋，拦了就"上一块的菜单关不掉"、
两块同时开着）—— 改成"点的是不是自己这条标签条"（`headRef.contains(target)` 才不关）。

## 三、节点清单（17 种基础 + 6 种 AI = 23）

> **算子不止这些**：图像处理那类（噪波/模糊/色阶/混合/法线…）在**包外**的 `l_tex_nodes` 里 ——
> 引擎刻意只用标准库，带依赖的算子库挂进来即可（见 `l_tex_nodes.md`，38 个 `Tex*` 节点）。

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

### 3.1 基础节点（17 种）

| 分类 | 节点 | 用途 / 端口要点 |
|---|---|---|
`l_node_engine/flow` | `LoopStart` / `LoopEnd` | 循环括号。**不要画回边**：`LoopStart → 循环体 → LoopEnd`，环由执行器接管。`LoopEnd.items` 是逐轮收集，`state` 回灌下一轮。轮数上限 `L_NODE_ENGINE_MAX_ITERATIONS`（兼容旧名 `L_COMFYUI_MAX_ITERATIONS`） |
| | `Compare` / `Not` / `And` / `Or` | **值级判断**（2026-10-12）：`Compare(a, op, b)` 出真/假（两边都能当数字就按数字比）；`Not` / `And` / `Or` 逻辑组合。喂给 `IfStart` 用 |
| | `IfStart` … `IfEnd` | **条件分支括号**（2026-10-12）：条件真 → 执行括号之间的节点；假 → **整段跳过**（区内节点输出写 `(None,)`，下游读它拿到 `None`、不会报「上游没有产出」）。区怎么认：**IfEnd 的祖先 − IfStart 的祖先**（所以 `IfEnd` 至少要接一条区内的线）；`IfEnd.taken` 告诉你这一支走没走。⚠ 第一版边界：**没有 else**、不支持 if 套 if、不支持把 if 放进循环体（循环放进 if 可以）；违反 → `GraphPlanError` 明说 |
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
`…/io` | `Input` | 图的**参数入口**（= agent 工具的形参）：`name` / `value` / `type` / `description`。`type` = **任意 python 类型表达式**（`str`/`int`/`float`/`bool`/`list`/`tuple`/`dict`/`set`/`bytes`/`None`/`Path`…，也能写 `list[int]`/`dict[str, int]`），只影响 ①给模型看的 schema ②编辑器挑控件 ③`Input.execute` 按它把值转成真对象；空/不像类型 → 退回 `string`。`image`/`model` 是语义标签（文件路径参数）。**值怎么存见 §3.3** |
| | `Output` | 图的**返回值出口**（= 工具返回）。按 ComfyUI 约定返回 `{"result": …, "ui": …}`，`ui` 供画布内联预览 |
| | `Preview` | 显示任意结果：**原样透传**（插在链中间不断流），同时把值交给画布渲染（`mode` = auto/text/json/repr） |
| `…/note` | `Note` | **画布注释**（2026-10-12）：`text` 是 Markdown，画布上渲染成便签、**双击就地编辑**（`Esc` 放弃）。**不参与执行** —— `execute` 返回空 tuple，跑图时就是个空操作（但必须注册：没注册的 class_type 执行器会报 `Unknown node class`）。前端渲染见 `l_mindmap_mmd` 的 `PyNote.jsx` + `markdown.js`：**marked 解析 + DOMPurify 消毒**（两步都要 —— marked 不做消毒，注释里的 `<script>`/`onerror`/`javascript:` 全靠 DOMPurify 删；实测探针通过） |
| `…/io` | `Ask` | **人在环**（2026-10-12）：跑图到此**暂停**问人（`prompt` / `options` 逗号候选 / `default`），答完继续、`answer` 往下流。实现 = 引擎钩子 `set_ask(fn)`（与 `set_guard` 同形状）+ 宿主通道：`l_mindmap_mmd` 把问题推成 SSE 的 `ask` 事件，浏览器在运行日志上方的询问条作答 → `POST /api/pygraphs/runs/{run_id}/input` 唤醒线程。**宿主没挂钩子时明确报错**（绝不静默返回空串 —— 审批被跳过却继续跑比报错危险）。超时 `L_MINDMAP_ASK_TIMEOUT`（默认 180s） |
| | `StateIn` / `StateOut` | **跨轮状态**（2026-10-12，会话续跑）：`StateIn(name, default)` 读上次存的值（`default` 是 **JSON 文本**，读不动退化成普通字符串）；`StateOut(name, value)` 原样透传给下游、**跑完由宿主落盘**。引擎**不碰磁盘**（它不知道"会话"是什么）：宿主跑前把会话里 `name` 的值灌进 `StateIn.default`，跑后把 `StateOut` 的输出写回 —— `l_mindmap_mmd` 落成 `pygraphs/_sessions/<图名>/<会话id>.json`，跑图带 `{"session": "<id>"}` 才启用 |
`…/subgraph` | `SubGraph` | **子图**（2026-10-12）：把**另一张图**当一步跑。契约 = 嵌套图的 `Input`（= 本节点参数端口，按 `name` 注入）/ `Output`（= 返回值，v1 只取一个，其余进 `ui.text`）。`sub` 是**内联 prompt、不是图名** —— 引擎宿主无关（图名解析是宿主的活），且增量指纹必须随内容变（只放名字会"内容改了还复用旧结果"）。深度上限 8、**按内容哈希**判成环、护栏接力（`active_guard()`，显式传入的 guard 不能被子图绕过）；`sub` 的指纹走**整体 sha1**（`_safe_repr` 会截断 → 会撞，见坑 14）；失败用 `SubGraphError` **原样上抛不套娃**（见坑 15）；子图内**不接**增量缓存（key 是节点 id，会撞外层）。**图名 → 内联 prompt** 走 `subgraph.resolve_graph_names`（节点类用 `GRAPH_NAME_PORT` 声明"哪个输入是图名、内联写进哪个键"，引擎仍不认目录 —— loader 是宿主给的；递归解开子图的子图，按**名字**拦环 / 拦深度，报错带节点路径 `GraphNameError`）。宿主侧（图名→内联 / 参数端口 / 编辑器 UI）与它的第一个用家 `TexFxMap` **已做**（2026-10-12）：逐条清单与实测判据见 `子图与FX-Map_计划.md`。**双击进内层图**（Substance 那种）：节点类声明 `GRAPH_SKELETON`（图文件模板）→ `get_node_info` 带成 `graph_skeleton`，宿主"双击没接图的节点"时照它建一张并接上（`SubGraph` 的模板是直通 `Input → Output`） |

### 3.2 3D 模型参数（`Input(type=model)`，2026-10-12）
值语义与 `image` **完全同构**（值是**路径**：上传后是服务端路径，手填本机路径也认）。四处配套：

| 环节 | 在哪 | 要点 |
|---|---|---|
| **函数 → 节点（批量）** | `l_node_engine/999.0/src/l_node_engine/from_functions.py` | 宿主侧那一半：`register_package_functions("我的包")` 一行把包里的公开函数全挂成节点 —— 只扫本模块定义的（不把 `os.path.join`/`lprint` 这类 import 进来的也包上）、端口按注解（没注解没默认值 → `*`+forceInput）、跳过私有/零参/denylist、**重名退成 `<模块名>.<函数名>` 不覆盖**、重扫幂等、过导入白名单、`async def` 也能跑；`describe_module_functions()` 只看不注册。自证 `tools/from_functions_check.py` |
| 目录 | `l_node_engine.pygraph.models_dir()` | `<图目录>/models`（与 `media/` 平级）；`model_url(path)` 出宿主相对 URL（`models/` 下 → 读口；本机路径 → 受控读口 `models-local?path=`）；**`model_path(value)` 给节点用** —— 绝对路径原样、相对名（含多文件 glTF 的 `<子目录>/<文件>`）补成 `models/` 下绝对路径 |
| 上传 | `l_mindmap_mmd/server.py` | `POST /api/pygraphs/models/upload?name=x.glb[&newdir=1\|&dir=<uuid8>]`：**原始 body 流式落盘**（不是 base64 —— 模型几十上百 MB，base64 再撑大 1/3 且全进内存）；白名单 `glb/gltf/obj/mtl/bin/fbx/stl/ply` + 贴图 `png/jpg/jpeg/webp`，上限 200MB，走 `L_MINDMAP_ALLOW_RUN` 门槛；**多文件**（glTF + `.bin` + 贴图）→ 服务端建一层 `<uuid8>/`，值取主文件的相对名 |
| 读取 | 同上 | `GET` **和 `HEAD`** `/api/pygraphs/models/{name:path}`（认 `<文件>` 与 `<服务端子目录>/<文件>`，其余形态 400、绝不拼客户端路径；HEAD 给前端探文件大小）；**本机路径**走受控只读读口 `GET\|HEAD /api/pygraphs/models-local?path=<绝对路径>`（白名单根 `L_MINDMAP_MODEL_ROOTS` + 只认 3D 扩展名 + `resolve()` 后在根内 + 普通文件） |
| 编辑器 | `l_mindmap_mmd` 的 `PyModelPicker.jsx` + `MeshViewer.jsx` | 值输入框 + 📁 上传（**可多选**）+ 👁 预览；预览是 three.js（**懒加载**，主包只 +24KB），认 glb/gltf/obj/**fbx**/stl/ply，OrbitControls 拖拽/滚轮/右键平移 |

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
   WARNING handler，能在 `wuwo svc log l_mindmap_mmd_server_exec` 里看到。
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

## 五、怎么自证（改完跑什么）

| 改了什么 | 跑什么 |
|---|---|
`l_node_engine/**` | `wuwor l_node_engine -- l_node_engine_selftest`（**20 组**；新增能力请加组） |
`web/src/pySchema.js`、校验/映射口径 | `wuwor l_mindmap_mmd -- l_mindmap_mmd_web_test`（30 项） |
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

## 六、已知未做 / 限制

- **跑图的墙钟超时**：默认 = 服务端 env `L_MINDMAP_RUN_TIMEOUT`（**60s**）；
  **2026-10-12 起支持 per-request 覆盖** —— 请求体给 `{"timeout": N}`（夹 1..3600）就用它
  （`_run_timeout`）。要跑长的**别改服务 env**（那是全服务口径），在这一次请求里给就行。
  撞上超时的表现：宿主发一条 `error`（`执行超时（Ns）。工作线程无法强杀…`）—— 脚本/前端**很容易漏看**，
  现场就只剩"耗时正好 N 秒 + 输出空"（实测踩过）。⚠ 另：**超时常常是症状不是病**：
  把整棵树的路径都塞进提示词那种"题目太大"，调到 300s 也回不来（实测）——先看提示词有多大。
- **引擎没有数值/排序原语**：`Compare`/`Not`/`And`/`Or` 只出真假，`CallFunction` 能借 stdlib
  做**单个**运算，但"计数 + 分组 + 排序"这种**汇总**在图上表达不出来（要么宿主办、要么模型办）。
  做"把智能体控制流搬进图"时，这是与"`if` 不能进循环体"并列的第二个硬缺口。
- **`if` 区放进循环体：** ~~不支持~~ → **2026-10-12 起支持**（智能体的循环体里必然有分支，
  这条不放开，"把控制流搬进图"就堵死）。实现要点（改这块先看这三条，都踩过）：
  ① **规划顺序** —— if 配对要先算，循环区靠它把括号 + **条件那一段**补进区里
  （只看"可达 ∩ 能到 LoopEnd"，`IfStart` 与它的 `Compare` 会漏在区外 → 假支会被当成真支）；
  ② `_run_loop` 每轮要 **dispatch 括号**，`executed` 集每轮新开并预置循环括号/区外上游
  （不预置会 `_run_if → upstream → LoopStart → _run_loop` **递归爆栈**）；
  ③ 区内的普通节点**要等它的括号跑完**再走（拓扑序里它可能排在括号前面）。
  仍拒：**`if` 套 `if`**、**if 与循环区跨边界**（一个括号在里一个在外）。自证：第 22 组。
- **多路分支 / 护栏（guard）在 pygraph 侧没有原语**：flow 的 `branch`（多 case）与 `gate`（guard 注册表）
  只能靠条件节点 + 逐条 if 展开（且 if 不能嵌套）；把智能体控制流搬进图时，这两层要么补齐原语，要么交给宿主。
- **子图（引擎 + 宿主都已做，2026-10-12）**：`SubGraph` 能跑（自证第 21 组）；宿主 `l_mindmap_mmd`
  在跑图 / 校验前把 `{graph: "图名"}` 内联成 `sub`（`_inline_subgraphs` → 引擎的 `resolve_graph_names`，
  解析不了直接 400）、`GET /api/pygraphs/ports?kind=subgraph&target=<图名>` 给参数端口、
  `/object_info` 把 `GRAPH_NAME_PORT` 那个输入换成**图名下拉**、编辑器卡片上有「↗ 子图」跳转。
  **仍然只取一个 `Output`**（多输出要动编辑器的端口声明 —— 前端按静态 `RETURN_TYPES` 画端口，留到"子图多输出"那步）。
- **子图内不接增量缓存**（缓存的 key 是节点 id，嵌套 id 会和外层撞）：改子图里任意一处 →
  整个子图重跑。子图**内容**变了外层那个 `SubGraph` 节点的指纹会跟着变，所以不会拿到旧结果。
- 队列只有"深度 4 的 FIFO"（没有优先级 / 取消排队；正在跑的那个杀不掉 —— 同"超时只能放弃等待"）。
- 分组框是**视觉框**（不带动成员、不做父子绑定）；要"整块搬走"得自己多选拖。循环区包裹框同理（自动生成、不可交互）。
- **hub 的图片/视频两条原生路由没真跑过**（未测：出图/出片能不能成、首帧 data URL 火山认不认）。
  LLM 那条**已验证**（只用 lugwit 账号密码，见 §3.0 第 4 条）。
- **STT 没有标准形状**：hub 不提供，节点发的是"任意本机服务的通用形状"（`audio_b64` + `filename`）——
  接自己的服务时可能要改。
- 图表 / PDF 等其余预览类型没做（图片/音频/视频已做：产物落媒体目录 + `ui.{image,audio,video}` + 宿主只有
  单文件名的媒体路由，别放宽那个安全边界）。**3D 模型预览是宿主的事**（`l_mindmap_mmd` 的 `MeshViewer`，
  2026-10-12 起还会把节点图产出的 PBR 材质贴上去，见 `l_tex_nodes.md` §6.3）—— 引擎侧不掺和。
- 折叠 / pin 的其余交互（节点标题栏折叠）没做；`bookmark` 是纯标记（没做筛选/跳转）。
- 前端只有协议层单测（`pySchema`），组件层没有自动化。
- `l_comfyui_backend` 里还躺着一份同源但更旧的执行内核（`node_base`/`node_registry`/`prompt_executor`），尚未接到 `l_node_engine`，两处口径可能漂。
- ⚠ **写前端单测的坑**：`web/src` 里相对导入**不带扩展名**（Vite 能解析，Node ESM 不能）。所以被测试的模块若引了同级文件，必须写成 `"./x.js"`（如 `pySchema.js` 引 `flowSchema.js`）——否则 `node --test` 报 `ERR_MODULE_NOT_FOUND`。
