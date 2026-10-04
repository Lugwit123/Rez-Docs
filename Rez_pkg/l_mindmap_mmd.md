# l_mindmap_mmd —— 脑图 / 流程图编辑器（mm.js 迁移 · React Flow）

> 面向 AI 助手的包文档。编辑、排障本包功能前先读本文件；涉及 `.solo` 守卫 / 热重载 / 反代的
> 机制细节见 `solo_单实例守卫模式.md` / `src_hot_reload_源码热重载与主页常驻.md` / `Nginx反向代理机制.md`。

## 这是什么

`l_mindmap_mmd`（端口 **8110**）是本仓库的脑图 / 流程图可视化编辑器，也是
`l_agent_chat` 默认智能体流程图（`default_intelligent_agent.json`，2026-10-04 由
`default.json` 改名）的**可视化编辑入口**：
图和运行端（`l_agent_chat/flow_spec.py` / `flow_engine.py`）共用**同一份 flows 目录**，这里改完即生效。

前端基于 **React Flow**（`@xyflow/react`），构建产物在 `web/dist`，由 FastAPI（`src/l_mindmap_mmd/server.py`）托管。

## 启动 / 服务

```cmd
wuwo svc status l_mindmap_mmd
wuwo svc log l_mindmap_mmd -f
wuwo svc open l_mindmap_mmd
```

- 服务别名：`l_mindmap_mmd_server`（= `python -m l_mindmap_mmd.server --host 0.0.0.0 --port 8110`）
- 带 `.dev_mod`（`L_DEV_MOD`）：改 `src/**` 会热更自重启；改 `web/src/**` 需 `npm run build`
  （产物进 `src/l_mindmap_mmd/static/dist`）后浏览器**硬刷新**——`main.js` 无版本哈希，易被缓存。
- 前端开发：`wuwo svc` 之外的 `l_mindmap_mmd_web_dev` 别名起 Vite :5175，`/api` 代理到 :8110。

## URL

| 路径 | 说明 |
|------|------|
| `/` | 编辑器主页（流列表 + 画布） |
| `/flow/{name}` | 打开指定流程图 |
| `/settings` | 兼容旧链接：进入编辑器并自动打开右侧「连线显示设置」面板 |

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

## 设置持久化（settings.yaml）

- **唯一设置源**：`~/.lugwit/l_mindmap_mmd/settings.yaml`（环境变量 `L_MINDMAP_MMD_SETTINGS` 可覆盖目录）。
- **前端不内置默认值**：默认值只在随包模板 `src/l_mindmap_mmd/settings.yaml`，首次访问自动复制到用户目录；
  服务器「模板补缺」会给旧文件自动补上新增键。
- **改动即时自动保存**：设置面板每改一项即 PUT 写 YAML；「恢复默认」= 后端用模板覆盖用户文件（`GET /api/settings?reset=1`）。
- 设置键：`handleWeight, handleScale, handleBias, handleAim, handleAimRef, showHandles`（手柄与曲线）、
  `avoidPad, softMargin, softRamp, softStep, showSamples, showAvoidBounds`（碰撞与避让）、
  `borderRadius, strokeWidth, backWidth, backOpacity, arrowDur, trailCount, arrowSize, dashGap,
  labelFontSize, showLabels`（线型与箭头）、`showGrid, showFps, layoutCol, layoutRow, layoutX0,
  layoutY0`（画布与布局）。面板按这四组折叠（折叠态存浏览器 localStorage），数值胶囊点一下可手输数字
  （单位按参数自动补、越界自动夹紧）。
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
|------|------|------|
| GET | `/api/flows` | 流列表 |
| GET | `/api/flows/{name}` | 取流程图 JSON |
| PUT | `/api/flows/{name}` | 保存流程图（编辑器自动保存也会 PUT） |
| DELETE | `/api/flows/{name}` | 删除 |
| GET | `/api/settings` | 读设置（`?reset=1` 恢复模板默认） |
| PUT | `/api/settings` | 写设置（增量合并，落 YAML） |

> ⚠️ 编辑器有 1s 静默自动保存：在浏览器里点「整理布局」/拖动节点会 PUT 覆盖 `default.json`。
> 测试后若需还原，用 git / 备份恢复，或 reload 使内存与文件一致（无 dirty 时不写）。

## 与 l_agent_chat 的联动

`l_agent_chat` 默认智能体有两档「图驱动」（`config.py` 的 `flow_enabled` / `flow_full`）：

| 档 | 谁决定控制流 | 说明 |
|---|---|---|
| `flow_enabled=1`（`flow_full=0`） | 图只决定**无调用时的收尾路由** | goal 判定 + `stall/verify/fact/confirm` 四道护栏门的顺序与去向 |
| `flow_enabled=1` + `flow_full=1` | 图决定**每一步路由** | 外加 `branch_calls`（去 tools / 走收尾）与 `branch_steps`（继续 plan / 收尾）；`plan`、`tools` 两个动作节点由 app 步循环**托管执行** |
| `flow_enabled=0` | 纯硬编码 | 完全不读图 |

图默认名 = `config.FLOW_NAME`（空则智能体名），当前工作区为 `default_intelligent_agent`
（存在 `<数据根>/l_agent_chat/flows/`，用户目录优先 → bundled `default_flow()` 兜底）；
图缺失 / 结构不满足 / 引擎报错 → 逐级降级（全流程 → 收尾路由 → 硬编码）并打 `⚠️` mark。
改图走本编辑器（:8110）可视化编辑，即改即生效。

## 环境变量

| 变量 | 作用 |
|------|------|
| `L_MINDMAP_MMD_RUNTIME` | 覆盖 flows 目录（默认 `~/.lugwit/l_agent_chat/flows`） |
| `L_MINDMAP_MMD_SETTINGS` | 覆盖 settings 目录（默认 `~/.lugwit/l_mindmap_mmd`） |
| `L_MINDMAP_HOST` / 端口 env | 监听地址 / 端口（默认 8110，`PORT_ENV` 亦可） |
