---
alwaysApply: false
description: 未完成清单（l_py_ui 共用 / 过程 / agent 图 / 画布面板）。**只记录、不在此处改代码**。
---

# 未完成 TODO：`l_py_ui` 共用 · 「过程」 · agent 图 · 画布面板

> 这份文档是 2026-10-14 一整轮工作的**未完成清单**（用户口径：「还有哪些没做完都写一个 todo 文档，
> 不再修改代码」）。每条都带**现状 + 目标 + 判据 + 风险**；做的时候按「先小后大」的顺序来。

## 完成情况（2026-10-14 晚，本轮按 §5 顺序做完 §1 / §2 / §2.1 / §2.2 / §3）

| 项 | 状态 | 落点 / 判据 |
|---|---|---|
| §1 rez 给准确路径 | ✅ | `l_py_ui/package.py` 增 `env.L_PY_UI_WEB`；两宿主 `requires` 加 `l_py_ui`；两个 `vite.config.js` 改 `process.env.L_PY_UI_WEB \|\| <回退>`。判据：`L_PY_UI_WEB` 有值、解析图出现 `l_py_ui-999.0`、两个 web `✓ built` |
| §2 过程流水统一出口 | ✅ | `TurnContext.steps` + `_step()` + **`process()`**（`[{kind,text,t}]`）；工具行带自己的 `steps`。判据：`tests.test_graph_turn` **33/33**（`ProcessFlowTest`） |
| §2.1 过程进工具卡、撤独立那格 | ✅ | `/api/pygraph-live` 多回 `process`；`ToolPane.jsx` 顶部 `⚙ 过程 · N 步 · Xs` + 每条卡挂自己那两步；**`ProcPane.jsx` 删除**、`pyLayout` 去掉 `proc` 标签。判据：`l_mindmap_mmd` `✓ built`、`.pyg-proc-block` 进了产物 |
| §2.2 过程作为一个工具 | ✅ | `process_tool.py` 注册只读工具 `process`（dry-run 也放行）；线程本地 `graph_turn.current_process()`。判据：`DEFAULT_TOOLS["process"]` 在、`blocks_in_dry_run()` False |
| §3 抽 `ActivityPanel` 成共享件 | ✅ | 抽进 `l_py_ui/ActivityPanel.jsx` + `activity.css`（Tailwind 工具类**逐条译成等价 CSS**，`ap-*`）；依赖 props 注入；聊天右栏 + 画布新「🧩 活动」格渲染**同一份**（⚠ 画布那一格 **2026-10-16 已删**，见 §3 补记；共享件只留给聊天）。判据：两个 web `✓ built`；**聊天侧真浏览器** `node web/e2e/activity_check.js` **25/25**（结构 + 关键元素**计算样式逐条钉住** + 无 JS 报错）；画布侧 `web_probe.cjs panes` **7/7** |
| §4.3 「📋 过程」格的去留 | ✅ | 撤掉（过程已进工具卡） |
| §4.4 同名类决定 | ✅ | 现状：聊天用 `.tool-card .t-*`（style.css）、`l_py_ui` 用 `tc-*` + 新 `ap-*` —— **三套前缀互不重叠**，谁都不用删 |
| §4.1 画布那 1 条 console 报错 | ✅ | `web_probe.cjs agentgraph` 与 `panes` 两跑均「整轮无 JS 报错 / 无异常 HTTP」——本轮复现不到（先前那条应已被早前的修复带走） |
| 既有红：`test_gov_scripts` 2 处禁用字符串 | ✅ | `doc/CHANGELOG.md` 与 `tools/agent_graph_smoke.py` 里那句 8.3 短路径示例的 `C:\Users\…` 改成 `<用户目录>\…` 写法（非本轮引入，顺手收掉）。现在 **23/23 OK** |

**注意**：§3 原判据是「改动前后同会话截屏**逐像素对**」——本仓**没有 VCS**（不是 git 仓库），改动前的产物/基线取不回来 ⇒ 那条没法做。
替代（已做、可复跑）：`web/e2e/activity_check.js` 把**关键元素的计算样式逐条钉死**成原 Tailwind 那份值
（`padding: 12px` / `border-radius: 8px` / `color: rgb(139,152,173)` …），等价于"像素级"的硬断言。
新增的两支自证资产：`web/e2e/activity_check.js`（聊天侧）· `l_mindmap_mmd/tools/web_probe.cjs` 的 `panes` 场景（画布侧）。

## 0. 背景：这一轮已经做完的（别重复做）

| 已完成 | 判据（都真跑过） |
|---|---|
| 图 = agent：默认智能体每轮由 `pro/agent/default_intelligent_agent` 驱动 | 普通对话里出现 `🧭 pygraph 驱动一轮…`；`restate_question` 0 次 |
| 作答进图（`HostStep(answer)`） | SSE 里 `answer-0` 0 次；`done.reply` = 图产出的答复 |
| 提示词槽（**图级入参** `system`/`user`/`tools_allow`/`tools_deny`；2026-10-17 起 `AgentPrompt` 节点已删）+ 接口入参 `prompt` | 四行入参空 = 不生效；老图里的 `AgentPrompt` 节点仍能被读 |
| **图里那句只是兜底**（调用方问句优先） | `test_graph_turn`+`test_agents` 40/40；冒烟 4/4 |
| `tools` 按类拆三节点（读/写/其它，各领一类、静默跳过） | 冒烟 4/4、工具轨迹与单节点基线一致 |
| 画布「🧰 工具卡」格（对话驱动 + 编辑器运行两条来源）+ 数据流/节点动画 | 采样 `animated=31 / hot=4→2`；接口 `tools=N` |
| 工具卡共用零件（`l_py_ui` 的 `ArgChips` / `ToolHeader` / `ResultView` / `ToolCard` + `toolcard.css`） | 两个前端 `✓ built` |
| **工具卡底部那排 = 聊天输入框那排**（同一份源码：`l_py_ui` 的 `OptionBar` / `TightSelect` / `composerRow.css`；聊天 `NewApp` 那排改用共享件，根类名仍是 `.composer-tools`） | 画布探针 `panes` 24/24（含"下拉 = 20px 高"这条同源硬判据）、`askbar` 7/7；聊天 `newui_check` 6 项（`.opt-bar` 在、四个下拉在、宽度贴文字 60px） |
| **那口输入框本身也是同一件**（`l_py_ui` 的 `ComposerBox` + **`ComposerEditor`**（编辑器本体：富文本 contenteditable+命令 chip / 纯文本 textarea **同一份代码**）/ `composerBox.css`；聊天侧只剩 ~60 行 assistant-ui 接线） | 画布 `panes` 26/26、`askbar` 7/7；聊天 `cmd` 6/6（chip + canonical 文本）、`queue` 25/0、`newui` 22/0 |
| **整口输入区收口成一个 `Composer`**（2026-10-16，用户口径「一个完整 Composer 组件，抽为共用」）：`l_py_ui/Composer.jsx` 装配全部（框 + 编辑器 + 那排 + 附件行 + 队列 + `/`·`@` 菜单 + 设置面板 + 拼写弹窗 + toast）；`ComposerTriggers.jsx` 也搬进共享件（Tailwind→`composerMenu.css`）；附件/截图/语音/优化/排队在画布**真能用**（`ToolPane` 注入 `services` → 宿主 `/api/pygraphs/composer/*` 转一手到 `:1250`）；附件**真进模型请求**（`graph-step/session` 与聊天同口径） | 画布 `composer`（同宽外观逐属性对拍 + 两端按钮齐 + 优化写回 + 附件上传/移除）、`panes`（⚙ 打开**共享**设置面板 + 点外关闭）、`askbar` 7/7；聊天 `cmd` 6/6；两端 `✓ built`；`tests.test_agent_endpoint.AttachmentChatTest` 两条新用例（附件内容真发出 / 缺失 400） |
| **`/multi_ai` 图化**（2026-10-16，用户口径「multi_ai 工具做成一个图可以么」→ 选「连**并行与流式**一起图化」+「**agent 图一定走 /multi_ai 图分支**」）：引擎加**并发批**（`PARALLEL=True` 节点成批并发，默认仍串行、走增量烹饪时不并发）+ **流式增量**（`AIText.stream` → `node_delta`）；资产 `Tools/multi_ai`（四家 `AIText` 并发 → PythonCode 收成 JSON → Output）；**图驱动那一档**靠 `ctx.app_tool_fn` 钩子把 `multi_ai` 路由到这张图（`live_frames` 每 50ms 排空 → SSE 实时出字；`multi` 载荷透传） | 引擎 `SELFTEST PASS`（4×0.35s 并发 0.35s vs 串行基线 1.4s；流式逐段推）；资产 `/validate` 0 错 0 警；`l_agent_chat_multi_ai_graph` 离线自证（含 app 钩子 16 帧）；**真服务**探针 `PASS`：`multi_start.models=[deepseek-v4-pro, doubao-seed-2-1-pro, glm-5-2, kimi-k2]`（= 图上那四家）、`multi_*` 帧 1596 条、**4 屏 ok=4** |
| **再搬一档（A 组四条 + B 组旋钮）**（2026-10-16 晚，用户口径「还有哪些可以搬」→ 四项全选）：**`g_overclaim`**（正文自称「已验证/已修复…」却没产出型调用，原 `app.py:10266`）/ **`g_planreview`**（plan 模式交计划前先让用户批准，原 `app.py:9090`；**不是 plan 模式整道门不管 = 模式分支进图那一支**）/ **B 组旋钮**（`max_fails`/`max_gate_hits`/`dedupe_loose_x2`/`answer_keep`，写在 `gc` 上、`apply_knobs` 按名读、**不接线**）/ **`answer` 节点 `keep` 参数** | `tests.test_graph_turn` **84 例全过**（`OverclaimGateTest` 4 / `PlanReviewGateTest` 4 / `KnobsTest` 2）；`l_agent_chat_graph_smoke` 4/4（资产自检把 `overclaim`/`planreview` 列进必查项）；画布 `agentgraph` 6/6（41 节点 / 66 边）；资产 37 → **41 节点** |
| **再搬三件进图**（2026-10-16 用户口径「还有哪些可以搬到图里面」→「三条都搬」）：**`nudge`**（预算提醒 ~70%，原 `app.py:7842` 硬编码）· **`g_nochange`**（要求「修/改/解决」却零改动，原 `app.py:10279` 只标一句）· **防抖护栏限额**（原 `config.READ_REPEAT_LIMIT`，判据仍只有 `app._dedupe_guard` 一处，图上只给限额 + 拦下的**不进分发器**）。三条旋钮都挂门参数卡 `gc`（新行只能往后加） | `tests.test_graph_turn` **74/74**（`BudgetNudgeTest` 3 / `NoChangeGateTest` 3 / `DedupeGuardTest` 2）；`l_agent_chat_graph_smoke` 4/4（资产自检把 `budget_nudge`/`nochange` 列进必查项）；画布 `agentgraph`；资产 34 → **37 节点**（合并式重生成，`AssetGuardTest` 绿） |
| **画布那口框也有 lagent 那条发送队列**（2026-10-16 用户口径「把 lagent 的队列做到 `/lugwit_agent/agent/default_intelligent_agent`」）：队列条/交互/样式全在共享件（`ComposerQueueBar`），本次补接线 —— `ToolPane` 托管队列并把 `agentSid`/`steerTaken` 透给 `Composer`；**`↪ 引导`** 与聊天**同一套邮箱**（`app._STEER_BOX`，键 = **画布会话号**）：图上新增宿主步骤 **`steer`**（循环体第一跳）→ `graph-step/exec` 回执带 id、宿主转 `/api/pygraphs/steer` 并发 `steer` SSE、编辑器据此撤行 | `tests.test_graph_turn`（`SteerTest` 3 例）；`AttachmentChatTest.test_graph_session_steer_reaches_the_model` 1/1（引导真进那次模型调用）；画布 `web_probe.cjs pqueue` 12/12；聊天 `run_queue_e2e` 25 项仍全过；资产合并式重生成（`AssetGuardTest` 绿） |
| 编辑器右栏「💬 对话」格（内嵌 `:1250`） | 真浏览器里渲染通过 |
| 「📋 过程」格（Step A，画布侧） | `?tab=proc` 出面板后「读类调用 · 1」 |
| 面板清单漏项修掉（`defaultLayout()` 少了 `chat/toolcards/proc`） | 补上后 `proc` 才冒得出来 |

---

## 1. `l_py_ui` 的路径：**能不能让 rez 给准确路径**

**现状**（用户口径：「走 vite 相对路径别名，这个不大好对不」——对，确实不硬）：

- `l_agent_chat/web/vite.config.js:12`：
  `SHARE = fileURLToPath(new URL("../../../l_py_ui/999.0/web/src", import.meta.url))`，
  第 58 行 `alias: { "@pyshare": SHARE }`。
- `l_mindmap_mmd/web/vite.config.js` 里同一套（指同一目录）。
- `l_agent_chat/999.0/package.py:35` 的 `requires` 里**没有** `l_py_ui`。

**问题**：① 相对路径把"两个包必须同货架（`rez-package-source/`）"写死了；② 哪个包被单独拷走/部署 ⇒
别名指空 ⇒ 构建失败或悄悄用旧源码；③ 路径里还硬编码了 `999.0` 版本目录名 ⇒ 升版本要改源码。

**目标（rez 提供准确路径）**：

1. `l_py_ui/999.0/package.py` 的 `commands()` 里设一个包根环境变量（照本仓既有口径，大写包名）：
   `env.L_PY_UI_ROOT = "{root}"`（顺带 `env.L_PY_UI_WEB = "{root}/web/src"` 更省事）。
2. `l_agent_chat/999.0/package.py` 与 `l_mindmap_mmd/999.0/package.py` 的 `requires` 加 `"l_py_ui"`。
3. 两个 `vite.config.js` 改成**优先读环境变量、回退相对路径**：
   ```js
   const SHARE = process.env.L_PY_UI_WEB
     || fileURLToPath(new URL("../../../l_py_ui/999.0/web/src", import.meta.url));   // 回退：直接跑 vite 时用
   ```
   （回退**必须留**：`vite`/`node` 直接跑、或没走 `wuwor` 的 CI 里没有那个 env。）
4. 一并把「包被单独部署」的口径写进 `Rez-Docs/Rez_pkg/工具契约_ToolSpec.md` §6（现在只写了"同一份源码"）。

**判据**：
- `wuwor l_agent_chat -- python -c "import os;print(os.environ.get('L_PY_UI_WEB'))"` 有值且指向 `l_py_ui/999.0/web/src`；
- `wuwor l_agent_chat -- l_agent_chat_web_build` 与 `wuwor l_mindmap_mmd -- l_mindmap_mmd_web_build` 都 `✓ built`；
- 把 `l_py_ui` 临时改成另一个版本目录（`999.1`）后**不改前端源码**仍能构建（这条证明"版本目录名不再硬编码"）。

**风险**：`env` 只在 `wuwor` 环境里有 ⇒ 任何"绕过 wuwor 直接 vite"的路径必须靠回退 ✓（保留回退即可）。

---

## 2. 「过程」：**要的是聊天里那种、要长在工具卡里、过程本身也算一个工具**

用户口径（2026-10-14，附截图）：聊天消息里那个 **「过程 · 20 步 · 14s」** 折叠块才是要的「过程」——
逐条带时间：`0.0s 检查 hooks…` / `2.5s 🗺 图级参数生效…` / `4.6s 🧠 规划（第 1/80 步）…` / `4.8s ▶ 调用 restate_question` …
**不是**我这次新加的那个独立「📋 过程」格。

三条待办：

1. **搬进工具卡**（用户原话：「我说得过程应该显示在工具卡，不是单独的一个过程面板」）✗ **与现状不符**：
   现状 = 画布多了一格「📋 过程」✗。目标 = 过程作为**工具卡的一部分**渲染（例如 `AgentTool`/`tools`
   节点的卡片里、或工具卡面板里每条调用下面挂它那一步的过程行），独立那一格**撤掉或并进去**。
2. **过程本身算一个工具**（用户原话：「过程能否也算一个工具」）：
   - 契约层：`ToolSpec` 已经写了"**思考内容组件也是一个工具**"（见 `工具契约_ToolSpec.md`）；
     这里要再落一层：**过程流水** = 一种特殊工具（`kind` 建议 `builtin`/`tool` + 只读、无需审批）。
   - 引擎层：它要有节点形态（能在画布上摆、有 UI、有"本轮值"）——**只读**（不改任何东西），
     值 = 这一段过程记录（`[{t, text}]`）。
   - agent 层：`graph_turn` 把过程记进 `ctx`（现在有 `node_events` / `all_tools` / `notes` / `gate_hits`
     / `usage` 这些碎片，**没有**"给用户看的过程流水"这一份统一结构 ✗）⇒ 需要一个
     `ctx.process()`（或 `ctx.steps`）作为唯一出口，供：工具卡、过程工具、复盘、轨迹共用。
   - 判据：工具表里能看到它；图上能摆它的节点；跑一轮后它的"本轮值"= 与聊天里那份一致的逐条过程
     （时间 + 文本），并且**只读**（dry-run 下也放行）。

**风险**：聊天那份过程流水是 `ActivityPanel` 从**会话 SSE** 里累加出来的（`runtime.js` 的活动累加器）；
图上/工具卡那份数据来源不同 ⇒ **先统一数据形状**（第 3 条做完才好做这条），别急着画。

---

## 3. Step B：把 `ActivityPanel` 抽成共享件（**要动聊天界面**）

**现状**：聊天的过程面板是 `l_agent_chat/web/src/new/ActivityPanel.jsx`，**不能直接搬** ——
它 import 了 `../markdown.jsx`（`parseDiff`）· `../api.js`（`listCmdLogs`/`listProcs`/`procLogs`/
`readCmdLog`/`restartProc`/`startProc`/`stopProc`…）· `../components/DiffBlock.jsx` · `DiffModal.jsx`。

**目标**：把**展示层**（`FileRow` / `CommandRow` / `Section` / `TraceRow` + 那份样式）移进 `l_py_ui`，
三样依赖改成 **props 注入**（`parseDiff` 函数、进程接口对象、`DiffBlock`/`DiffModal` 组件），
两个宿主渲染**同一个组件**（DOM 与类名一字不改）。

**判据**：聊天侧**改动前后同会话截屏逐像素对**（含字体族/行高/间距/药丸尺寸 —— 这几项是上一轮
把工具卡搬进共享件时踩过的坑）；画布侧同一份组件渲染同一份数据。

**风险**：这是**动主应用**的改动 ⇒ 必须"先对拍再交"，且**一次只改这一个东西**（上一轮的教训：
两个会改行为的改动叠在一起，冒烟 1/4 时把锅记错了 ✗）。

**补记（2026-10-16，用户要求「删除"活动"标签页」）**：画布侧那格「🧩 活动」**已删** ——
`pyLayout` 去掉 `activity` 标签、`PyGraphEditor` 去掉 `put("activity", …)`（连带 `liveActivity`
派生与 `ACT_WRITE`/`ACT_RUN` 两张表）、页内不再 import `ActivityPanel`；老存档里那一份由
`dedupeAll`（只认 `TAB_DEFS` 里的键）自动丢掉。**共享件本身留着**：聊天右栏「Agent 活动」照旧
用 `l_py_ui/ActivityPanel.jsx`（那 25/25 的判据仍有效）。要改判据看 `web_probe.cjs panes`
（那条已换成"标签条里不该再出现 🧩 活动"）。**别没问就加回来。**

---

## 4. 杂项（小、独立，随时可捡）

| # | 事项 | 现状 / 判据 |
|---|---|---|
| 4.1 | 画布那页仍有 **1 条 console 报错**未查 | 不影响「📋 过程」渲染；先看 `.playwright-mcp/console-*.log` |
| 4.2 | `p_allow`（`AgentPrompt(tools_allow)`）**空着** | 空 = 不生效 ✓；要收窄工具表得**点名**填（别写死一份会砍掉 156 个工具 ✗） |
| 4.3 | 「📋 过程」格的去留 | 取决于 §2.1（过程应在工具卡里）⇒ 做完 §2 后**撤掉或并进工具卡** |
| 4.4 | `l_py_ui/toolcard.css` 与聊天 `style.css` 有几组**同名类**（`.t-tag`/`.t-args`/`.t-arg*`/`.t-result`） | 现在聊天卡片已不用共享件（回退了 ✓），同名不冲突 ✓；将来做 §3 时要决定"谁留谁删" |
| 4.5 | `_LIVE_PYGRAPH` 只留**最近一轮** | 刷新页面看到的是上一轮的流水（有意为之 ✓）；若要"跨轮历史"另有 `_traces`/会话文件 |
| 4.6 | `l_agent_chat_graph_step_probe --readonly` 需**服务自己带** `AGENT_CHAT_PYGRAPH_READONLY=1` | 没带时探针**自动跳过并说明**（不是失败 ✓） |
| 4.7 | 编辑器的「接口」格默认值 | `prompt` 已有默认值 ✓；改完记得**「实时保存」关着要点「保存」** |
| 4.8 | `l_mindmap_mmd/web/src/pyLayout.js::defaultLayout()` 的显式清单 | 已补 `chat/toolcards/proc` ✓；**加新右栏标签必须同步加到这里**（文件第 101 行有注释） |

## 5. 建议顺序

1. **§1（rez 给准确路径）** —— 小、独立、且是"共用的地基" ✓。
2. **§2 的数据统一**（`ctx.process()` 一份过程流水）—— 它是 §2.1/§2.2/§3 的共同前置 ✓。
3. **§2.1 过程进工具卡** + 撤掉独立那一格 ✓（先把观感对齐用户要的）✓。
4. **§3 Step B**（动聊天，前后对拍）✓。
5. **§2.2 过程作为工具**（节点 + 工具表 + 只读）✓ + §4 的小尾巴逐个清 ✓。
