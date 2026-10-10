# l_agent_chat 使用指南

## 概述

`l_agent_chat` 是本地 AI 编码 Agent 聊天服务：FastAPI 提供 Web UI + SSE 流式对话，
可调用本地工具集（文件/命令/Git/HTTP）辅助编码，支持对话存储、远程工具服务、
上下文压缩与 token 统计。模型走 OpenAI 兼容 Chat Completions 接口，支持多个供应商
（厂商），默认火山方舟 `volcengine` + `deepseek-v4-flash`；模型目录与 API Key
复用 `l_model_hub` 的统一注册表（`models.json`）与密钥（权威存储 = lugwit_auth 中心密钥存储，经 hub 取）。

> **模型 id 必须是 `l_model_hub` 清单里真实存在的**（`GET {hub}/v1/models`）。写了不存在的名字
> （例：早期的 `DeepSeek-V4.1-Flash`，清单里只有 `deepseek-v4-flash`）时，网关会把它当
> **未知模型**走 auto 兜底：按 `priority.json` 的 global 顺序（volcengine → deepseek →
> siliconflow → **minimax** → zhipu → …）**每个厂商各取一个代表模型**依次试，实际可能落到
> 完全另一家。页脚会如实标出上游回报的实际模型（`served`，如「实际 MiniMax-M2.5」）；
> 各家内联工具调用模板（`seed:tool_call` / `minimax:tool_call`）就是「谁接上的」指纹。

## 包信息

| 属性 | 值 |
|---|---|
| 包名 | `l_agent_chat` |
| 版本 | `999.0` |
| 作者 | Lugwit Team |
| 依赖 | `python-3.12+<3.13`, `fastapi`, `uvicorn`, `jinja2`, `requests`, `psutil`, `l_agent_tool` |
| 默认端口 | `1250` |
| 默认供应商 / 模型 | 火山方舟 `volcengine` / `deepseek-v4-flash` |

## 启动

进入 rez 环境并启动服务：

```bat
wuwo rez env l_agent_chat -- l_agent_chat
```

或调用包 alias（相当于上面的命令）：

```bat
wuwo rez env l_agent_chat -- l_agent_chat -y
```

`-y`：端口被占用时自动结束占用进程，不询问。

带热重载（推荐开发用）：热重载**不是默认开的**，它由 `.dev_mod` **硬门控**
（wuwo 注入 `L_DEV_MOD=1`）；带 `.dev_mod` 启动时才起 `SrcWatchService`
（还可被 `L_SRC_WATCH=0` 关掉）。

⚠ **带 `.dev_mod` 时改 `.py` 会打断正在跑的那一轮**：热重启是**进程级**的（不是热替换模块），
SSE 流随之断开。留下的是实时保存兜底（残留 `live` 快照定格成带 `interrupted` 的正式消息，
半截回答不丢），**丢掉的是**这一轮的规划/工具进度，以及**等审批 / 等用户回答的 future**
（`approval_futures` / `ask_futures` 在内存里）。另：反复保存会撞重启锁/熔断（暂时拒重启 →
表现为"改了不生效"）。所以要**边优化边测**（尤其 A/B，两臂必须跑同一份代码）时，
请另起端口 + **不带** `.dev_mod`，见 `Rez_pkg/l_agent_chat_隔离实例A_B实测.md`。

```bat
wuwo rez env l_agent_chat .dev_mod -- l_agent_chat
```

⚠ **不带 `.dev_mod` 时改 `.py` 不生效**：服务是单进程 `uvicorn.run(app)`，
**没有** `uvicorn --reload`。改完必须手动重启，别盲杀进程：

```bat
wuwor l_agent_chat -- python -m l_agent_chat.app restart_self_cli
```

模板 `.html` 是例外：走 Jinja `auto_reload`，刷新页面即变。
**实测踩过**：改了 `config.py` 而进程还在跑旧配置 → 表现为「市场源读取失败」，
排错时先比「进程启动时间 vs 文件修改时间」（见《src_hot_reload 源码热重载与主页常驻》）。

`.dev_mod` 这类修饰符必须跟在包名后（`wuwo rez env <包> .dev_mod -- <alias>`）。

访问：浏览器打开 `http://127.0.0.1:1250`。

### 前端构建与开发（`web/`）

网页前端在 `web/`（React 19 + assistant-ui + Vite + Tailwind），**构建产物**直接落进
Python 包的静态目录 `src/l_agent_chat/static/dist/`（见 `web/vite.config.js` 的 `outDir`），
运行时由 FastAPI 托管，**不需要 Node**。因此改了 `web/src` 必须重新构建才会在页面生效：

```bat
cd web
npm install          rem 首次（node_modules 已存在则跳过）
npm run build        rem 一次性生产构建
npm run watch        rem 开发常驻：保存即增量重建 dist，刷新页面即可（推荐）
npm run dev:server   rem Vite dev server（127.0.0.1:5176，HMR，/api 代理到 :1250）
                     rem ⚠ 端口以 web/vite.config.js 的 DEV_PORT 为准（5173→5174→5176 挪过两次）
```

- **推荐 `npm run watch`**：与生产走同一路径（后端 URL、Jinja 模板注入、设置页、VS Code 内嵌 shim 都在），只是保存自动重建。
- `npm run dev:server` 只有 HMR 快这一优势，但走的是 `web/index.html`，**不经 Jinja**：没有设置页、
  `__UI_ENTRY`、`__TOOL_CONTENT_EXPANDED`、favicon 与 VS Code shim，只在纯新版 UI 调样式时临时用。
- 改 `templates/*.html`（含设置页）与 Python 源码**不需要前端构建**；Python 源码由 uvicorn 热重载。
- 交付 / VS Code 扩展内嵌用构建产物，验完后照常 `npm run build` 固化 `static/dist`。

## 主要功能

### 聊天（SSE 流式）

`POST /api/chat`，请求体：

```json
{
  "messages": [{"role": "user", "content": "帮我写一个函数..."}],
  "model": "deepseek-v4-flash",
  "temperature": 0.1,
  "stream": true,
  "max_tool_steps": 3,
  "thinking": true,
  "reasoning_effort": "low",
  "show_reasoning": true
}
```

> 注：`thinking` 开关思考；`reasoning_effort` 是**思考强度**（`low` / `high` / `max`）。
> **前端默认档是「低」**（2026-10-07 起）—— 输入框底栏那个下拉去掉了旧的「默认（不指定）」档，
> 改成**始终下发最低档**（`off` = 关思考，由前端翻成 `thinking:false` 且不带该字段）。
> 历史上「`thinking` + `reasoning_effort` **同发**会被网关秒回空流」的坑，在当前网关/模型
> （volcengine / deepseek-v4-1-flash）上**已不复现**（9 次采样全部 200 正常返回）；`stream_chat`
> 与 `chat()` 两条路的**空流降级重试保留作保险**（依次摘 `reasoning_effort`、`thinking` 再试）。
> `show_reasoning` **只作记录**，
> 后端不再据此裁剪 `reasoning` 事件（见下表），渲染与否交给前端开关。
> `mode` 是对话模式（`agent` 默认 / `ask` / `plan` / `review`，见「新版 UI 要点」），
> 缺省或未知值按 `agent` 处理。
> `persist: false` = **无状态调用**（不写会话文件、不进侧栏），脚本化/端到端自测用；
> 缺省 `true`（照常落盘）。

两个**特殊负载**（都只带容器字段，历史以服务端会话文件为准，见「消息操作与发送队列」）：

| 字段 | 说明 |
|---|---|
| `regen: {mid, guide?}` | **重新生成**那条回复：结果作为**新版本**追加（= 同层新分支），不新开消息；`guide` 是这次的临时要求（只进本轮 wire，不落盘） |
| `edit: {mid? \| index?, original?, content}` | **重新编辑**某条提问：定位（`mid` → `index` → 按 `original` 原文从后往前找）→ 建树 → 当前分支截到那条之前 → 落盘时新提问成为旧提问的**兄弟节点**。定位失败 / 原文对不上 → 报错**不下笔**。两个都不能与 `persist: false` 同用 |

SSE 事件类型：

| 事件 | 说明 |
|---|---|
| `tool_start` | 开始执行工具，附 `tool`/`path`/`command` |
| `tool_result` | 工具结果文本 |
| `tool_error` | 工具执行异常 |
| `tool_rejected` | 用户拒绝了需人工审批的工具 |
| `approval_required` | 请求人工审批，附 `approval_id`/`tool`/`args`/`diff`（写文件时含 diff 预览） |
| `reasoning` | 思考过程，**始终透出**（`show_reasoning` 仅作记录；是否渲染由前端「显示思考」开关决定） |
| `delta` | 回复增量 |
| `done` | 结束，附完整 `reply` |
| `error` | 出错信息 |

人工审批：需要确认的调用发 `approval_required` 事件，前端确认后回调
`POST /api/tool-approval`（`{approval_id, approved}`）。**哪些调用要问**由「权限规则 +
权限模式」共同决定，见「审批与权限模式」；规则可在设置页「🔑 权限规则」增删。

### 新版 UI 要点

默认路由（`/`、`/chat`）是新版（assistant-ui 两栏 + 会话列表），`/classic` 为旧版；
`?ui=classic` / `?ui=chat` 可临时覆盖（见 `web/src/main.jsx`）。侧栏品牌区文字为 `l_agent_chat`。

- **思考过程**：回答进行中（消息 `running`）思考块**实时展开**，整条回答结束后**自动折叠**，点标题可手动展开/收起。
  判据是**消息级** `status` 而非单个 part 的 `status`——part 流完会先变 `complete`，用它会导致正文还没吐完就折叠。
  **思考按真实位置分段显示**（不再全堆在过程块顶部）：每段思考落在它那次调用之后，`reasoning` part 带 `startedAt`，
  与工具卡/正文按时间线交错；**每段思考与每张工具卡都显示耗时 chip**（思考=阶段起点到该段结束，工具=客户端 `tool_start`→`tool_result` 实测耗时）。
- **消息底部的身份 / 提问 / 耗时**（2026-10-07，同一行）：`第 3 条回复 · deepseek::… · ↑2.1k ↓430 ·
  提问：看一下 a.txt ▸ · ⏱ 12.4s · 完成 14:23:05`
  - **耗时/时钟**：跑的过程中**实时**（`⏱ 12s · 现在 14:23:05`，本地起算的秒表 250ms 一跳），跑完变
    统计值 `⏱ 12.4s · 完成 14:23:05`（悬停看「起 → 止 · ↑输入 ↓输出」）；报错/被掐断也会冻住。
    刷新后用后端 `steps` 最后一条的 `t` 当总耗时、消息 `at` 当结束时刻，与实时是同一个量纲。
    **不报步数**（客户端行数与服务端落盘行数不是一个口径）。
    **悬停这串 ↑↓** 会弹「本轮 Tokens 消耗详情」——与消息**顶部**那颗是**同一张卡**（同一组件、
    同一份 `run-meta` 数据、同样的观感；都是 fixed 定位、按各自那一行算位置，移开即收起）。
  - **第 N 条回复** = 本会话里 assistant 消息的序号；**提问**按钮（原「提示词」）= 从这条往前找
    最近的 user 消息（点开看全文，本轮 system 提示词收在面板里的二级折叠中）。
    两者都从 `thread.messages` 推（只返回原始值的 selector，避免流式时每条消息重渲），
    所以**实时与刷新后一致**，且天然是**当前分支**那一问。
  - 实现：`runtime.js` 的 `withMeta`（页脚 part 不进 `parts`、每次 yield 拼上，保下标又保证在最末尾）
    + `NewApp.jsx` 的 `MsgClock` / `useTurnOrdinal` / `useTurnQuestion`；e2e 见 `web/e2e/run_clock_e2e.py`。
- **「过程」流水的进度是"真实内容"口径**（2026-10-10）：头部按服务端 `status` 事件的字段显示
  `phase`（`waiting` 等上游首内容 / `received` 只收到协议帧 / `reasoning` / `response` /
  `tool_args` 收工具参数尚未执行 / `tool` 工具执行 / `approval`、`ask_user` 等用户等待）、
  累计字符数（`reasoning_chars` / `response_chars` / `tool_args_chars` / `data_events`）、
  `silent_seconds`（距最近**真实内容**多久）与 `elapsed`（本次模型请求耗时）。
  - **心跳与协议帧（role、usage、空 delta）不算内容进展**：既不刷新静默基准，也不冒充"模型在思考"；
    上游一直不吐内容就如实写「等待上游内容（尚无内容）」。
  - **收工具参数 ≠ 工具已执行**（`tool_args` 阶段写明"尚未执行"）；**等待用户**（审批 / 问答 /
    并答配置）是**独立阶段**，不会报成"模型无新进度"。
  - 整轮耗时按采样时刻续表，内容静默单独计时 —— 不再用"步骤身份有没有变化"当进度
    （那会把同一步持续输出误报成停滞）。
- **翻译**：思考块展开后右上角有「🌐 翻译」，调 `POST /api/translate`（带 `lines=true`）**一行英语一行中文**逐行对照，
  **每行右侧带 🔊 播放**（`POST /api/voice/tts`，`voice` 参数指定音色）；按钮旁的下拉可切语音（`GET /api/voice/voices` 取音色表），
  翻译失败/朗读失败会显式标 ⚠，不静默。行的原文/译文也可用「读全部原文 / 译文」整段朗读。
- **语音播报**：思考与回复的 🔊 都走**微软 Edge-TTS 免费服务**（`l_agent_chat.voice`，`/api/voice/tts` / `/api/voice/voices`），
  不需要额外密钥；音色可选（`EDGE_VOICE_DEFAULT` 中文 / `EDGE_VOICE_DEFAULT_EN` 英文），同一文本+音色走内存缓存。
- **源码查看弹窗**：消息里的文件路径/行号可点开 `GET /api/source`（读文件 + 上下若干行上下文，单次上限 `SOURCE_MODAL_MAX_LINES=400`），
  弹窗内可直接「在编辑器中打开」（`web/src/new/sourceView.js`，检测 vscode 可用时按 `path:line` 定位）。
- **对话模式**（输入框右下角「模式」下拉，全局设置存 `localStorage`）：`agent`=完整能力（默认）；
  `ask`=只读问答；`plan`=只规划不执行；`review`=代码审查。随 `POST /api/chat` 的 `mode` 传后端，
  由 `chat_modes.py` 同时做两件事：注入对应 system 指令 + 按**只读工具白名单**裁剪可用工具
  （`write_file`/`edit_file`/`run_command`/`run_background`/`execute`/`task` 等一律不给，未列入的
  远程/MCP 工具也拦掉）。受限模式会跳过 `/ls`、`/read`、`/ws` 快速路径（含切工作区等副作用），
  改由 planner 用只读工具处理。
- **权限模式**（输入框右下角「权限」下拉，与「模式」并列）：`default`=默认权限（规则说了算）；
  `allow_all`=全放行（除 deny 外一律不问）；`autopilot`=自动巡航（普通工具自动放行，改配置 /
  受保护路径仍确认一次）。它是**服务端设置**（`permission_mode`，切换即时生效），与对话模式
  是两条正交的轴，详见「审批与权限模式」。
- **提示词优化**：输入框右下角「✨」把草稿交 `POST /api/prompt/optimize`（实现
  `prompt_optimizer.py`）改写成更清晰的提示词并写回输入框；请求中按钮显示七彩环形 spinner。
  模型**独立于聊天默认模型**（设置页「提示词优化」的 `prompt_optimize_provider` /
  `prompt_optimize_model`，缺省 zhipu / `glm-4-flash` 免费档），不占主力模型额度。
- **发送前拼写检查**（2026-10-09）：Enter / 点「发送」之前，把草稿交 `POST /api/spell/check`
  （实现 `spell_check.py`）过一遍独立小模型；**查出错字才弹窗**（`SpellCheckDialog`），让你选
  「纠正后发送 / 原样发送 / 取消（留在输入框自己改）」。模型同样独立于聊天默认模型
  （`spell_check_provider` / `spell_check_model`，缺省 zhipu / `glm-4-flash` 免费档）。
  默认**开**，开关与超时在 `/panel`（「发消息设置」）或设置页「发送前拼写检查」：
  `spell_check_enabled` / `spell_check_timeout`（秒，默认 12 —— **实测定的**：免费档
  glm-4-flash 上短草稿 1.6~4s、长草稿 5~10s，按"0.5~1.5s"估的 8 秒会经常踩超时）。
  **延迟怎么藏**：停手 800ms 就**先查一遍并缓存**（"边打字边查"），按 Enter / 点「发送」时命中缓存
  → 零等待、不多花一次调用；只有"打完字立刻发"才会真的等那 1.6~4s。
  三条硬口径：① **超时 / 出错一律按
  "没查出问题"原样发送**（检查失败绝不堵住输入框）；② 三个发送口（Enter、排队 ⏳、「发送」）
  走同一个收口，拦一处等于没拦；③ `/命令`、`@引用`、文件名 / 路径 / 标识符禁改 —— 提示词点名之外，
  `spell_check.py` 落地前还有两道**确定性**校验（`_tokens` 少一个 token、`_undo_identifier_edits`
  撤销"猜着改"的文件名，撤不干净就整条作废）。**质量预期**：免费档英文较稳但**召回不稳**
  （同一句有时只抓一处；句子里带文件名时更容易漏），中文错字一般（`地止`/`文当` 时对时错）
  —— 所以弹窗逐条列 `from → to` 让人过目是必须的，不做静默替换；要更准就在设置页把
  `spell_check_model` 换成更强的档（走的是独立小模型，不占聊天模型）。
- **纠错经验库**（2026-10-09，接上一条）：弹窗里的**「纠正后」那个框可以直接改** —— 不认同 AI 的写法
  就在里面改，发出去之后把它记成经验，**下次同一处直接照你的写法改**；点「原样发送」则是反向教学：
  那几条记成"别改这个词"（累计 2 次才生效，防赶时间误点）。学什么是**算出来的**：后端 diff
  「原文 → 你最终发出去的文本」（不是猜），落盘 `<存储根>/.l_agent_ws/spell_experience.json`
  （与「记忆」同根），注入时**只挑出现在这句草稿里的**条目（没命中一个字符都不加 —— 免费档提示词
  越长越乱，实测过）。端点：`GET /api/spell/experience`（看）/ `POST`（记）/ `DELETE`（清，
  `?from=<词>` 清单条）；总开关 `spell_learn_enabled`（设置页「记住我自己改过的写法」）。
  ⚠ 护栏：单字规则（如 `止 → 址`）**不进库** —— 注入是按子串命中，单字会误伤一整片，所以不足 2 字
  就借上下文或丢弃。
- **设置页窄屏**：宽 ≤860px 时左侧「设置分组」侧栏变顶部横排，可**按住拖动**横向滚动（桌面仍为竖排）。
  手机端**模型设置卡片**改为**水平紧凑布局**（`templates/settings.html` 的 `.card.model-compact`：键值对同行、压缩内边距，
  避免一屏只放得下一项）。
- **手机端与老内核兼容**：Tailwind v4 把工具类全塞进 `@layer`，而「荣耀自带浏览器」这类老 Chromium
  内核（<99）**不认 `@layer`** —— 未知 at-rule 整块丢弃，工具类一个不剩（侧栏 `hidden`、`truncate`
  全失效，页面塌成无样式单列）。构建期用 `@csstools/postcss-cascade-layers` 把层拍平，且**必须跑在
  `generateBundle`**（见 `web/vite.config.js` 的 `cascadeLayersPlugin`）：逐 CSS 模块跑时它看不见
  无层那份 style.css，会把无层/层内优先级算反，反而压掉旧 UI 的 `.btn`。
  手机端输入框高度也在移动端媒体查询里调大（`.composer-input { min-height: 64px }`）。

### 输入框上方的「运行 / 改动」面板（2026-10-07）

输入框**上方**常驻一条可收展的摘要条（`▸ 运行 / 改动 · ⚙ 运行中 N · 📝 改动 M`），点开是两个标签：

| 标签 | 内容 | 数据源 |
|---|---|---|
| 正在运行的命令 | 在跑的**置顶**（绿点 + 已运行秒数 + `📄 日志 / ↻ 重启 / ⏹ 停止`），本会话跑过的其它命令标灰排后面 | `GET /api/proc`（权威运行态）+ 前端 activity 里的命令历史，按命令文本合并 |
| 文件修改记录 | 按文件聚合：时间 · `新建`/`已删除`/`修改` · 相对路径 · `×N`（改过几次）· `+N −Y ~Z`（纯增 / 纯删 / 修改行）· 工具名；**点一行 → diff 视图**（该文件历次改动可逐轮切换，增行绿删行红） | `GET /api/session/checkpoints?stats=1`（落盘、跨服务重启还在；行数与状态由后端拿「改动前内容 vs 当前盘上文件」现算：`_file_changes` / `_diff_stats3`）+ 单文件 diff 走 `GET /api/session/checkpoint/diff?id=&path=` |

- 组件 `web/src/new/RunPanel.jsx`，挂在 Composer 的 `<footer>` 里（附件条与输入框之间），所以窄屏也在输入框上方。
- 摘要条右端有 **`↻ 刷新`**（一次重拉进程表 + 改动记录；自动轮询是展开看命令 3s / 其它 15s）与 **`✓N`**（已同意条数）。
- 「同意」过的条目会挪到第三个标签 **「已同意」**（纯本地状态、刷新页面即清；那里把按钮换成「↩ 撤回」放回来）。
- 文件那栏**每行都有「同意 / 拒绝」**：同意 = 本地确认（改动已落盘，点过变「已同意」，刷新回到未标记）；
  拒绝 = 真回滚到**这次改动之前**（二次确认 → `POST /api/session/revert`），diff 弹窗底部同两个按钮、
  作用于当前查看的那一轮（键 = `checkpoint|路径`，同一文件多轮各自标记）。
- 轮询只在**需要**时走：展开且停在命令标签 3s、其它 15s（只为摘要条上的数字）；有活进程时 1s 心跳续算时长。
- 命令列表会**过滤掉**"参数 JSON 被塞进 command 字段"的历史噪音（以 `{` / `[` 开头的跳过）。

### 消息操作与发送队列（2026-10-06）

**每条消息底部的操作**：助手回复 = 继续 / 复制 / 重新生成 / 讲解 / 朗读 / 复盘；用户提问 = 复制 / 重新编辑。
「重新生成」「重新编辑」都不是覆盖，而是**开分支**：

- **继续（`▶`）**：**续写这条回复**（不是发新消息、也不是重新作答）—— 服务端把这条的正文喂回去
  让模型从断点接着写（`regen: {mid, mode: "continue"}`），落盘**拼成同一条消息的新版本**
  （原正文 + 续写；旧的那半截留作上一版可切回）。live 期间正文也带着旧内容，所以看到的是
  这一条在变长。

|  | 重新生成 | 重新编辑 |
|---|---|---|
| 触发 | 回复底部 `↻`（可附一句本次要求） | 提问底部 `✎ 重新编辑`（气泡原地变编辑框，Ctrl+Enter 发送 / Esc 取消） |
| 结果 | 那条回复多一个版本，旧版本**带着它自己的后续**留着 | 从那题起开新分支重新回答，旧提问 + 后续整段留在隔壁 |
| 请求 | `POST /api/chat` 带 `regen:{mid, guide?}` | `POST /api/chat` 带 `edit:{mid?/index?, original?, content}`；**问答卡的「重新回答」**用 `edit:{ask:{ask_id?, question, answer}}`（提问原文不动，只把"这次的选择改了"补在后面 → 重走那一轮；`ask_id` 是定位依据，卡片自己没 mid 时也认得出） |
| 切回 | 底部 `◀ 第 i/N 版 ▶`（= `POST /api/session/version/final`，**切的是分支**，下面的对话跟着换） | 同左（提问版本走同一套控件） |

- **跑的过程中是"原地重写"**（2026-10-08）：点下去这条卡**立刻清空**（像一条**新消息**一样），
  新答案从头流出来（页脚标 `重新生成中 · 下面 N 条属旧版分支`），**不会**在列表底部另开一条
  影子回复、更不会"边跑边给你看上一版的内容"（旧版留在 `◀ i/N ▶` 后面，切回去才看得到）。
  为什么必须这样：服务端把"正在跑的那一轮"接在历史末尾
  （`/api/session/current` 的 `messages + live`），而重新生成是**同一条回复的另一个版本** →
  影子回复长在视口外、你点的那条卡零反馈、跑完影子消失而上面那条悄悄换版 —— 三处都对不上
  （ChatGPT / Claude / LibreChat 都是原地重写）。现在 `app._attach_live` 按 live 消息上的
  `regen_of` 并回原位置。
- 会话文件里就是一棵**分支树**（`session_branches`：`nodes` / `path` / `roots`，`messages` 是当前分支的扁平投影）；
  没分叉过的会话不写 `nodes`（**懒建**，第一次重新生成 / 重新编辑才建树）。
- 会话**标题**跟着当前分支的第一条提问走（`session_store.title_from`）：改了第一条再切回原版，标题也回去。
- 本轮还在跑时「重新编辑」禁用（那题还没落盘）；服务端 `session_busy` 也会兜底。
- **开分支后会检查「文件改动要不要留」**（2026-10-07，`branch_changes.py`）：**任何**开分支的动作
  （重新生成 / 重新编辑 / 问答卡重答）之后，被放弃的那条线改过的文件**还在磁盘上**（agent 不会自动
  回退）→ 弹窗列出「从这条分支点往后改过哪些文件、各自 +N/−M 行」，点文件名看 diff，底部
  「全部保留 / 全部撤销」，也可以勾选后只撤勾选的。
  - 怎么算：分支点 = 那一轮的**用户消息发送时刻**；晚于它的 `checkpoints` 就是被放弃那条线的改动
    （一个工具调用一个快照）；行数/diff 用「运行 / 改动」面板同一套口径（快照内容 vs 当前文件）。
  - 撤销 = 每个文件取**最早**那个快照写回（= 那些改动之前的样子）。⚠ 破坏性：新分支之后若又改过
    同一个文件，那些新改动也会一起丢（弹窗里写明了）。
  - 没改过文件的分支**不弹**（不留标记）；标记落在会话文件的 `branch_check` 键上，`/api/session/current`
    带回，拍板端点 `POST /api/session/branch-check {action: keep|undo, files?}`。
- **问答卡（`ask_user`）的「重新回答」**（2026-10-07）：本轮**还在跑**时按选项/输入 = 直接回答案
  （进那次工具结果，同一轮继续）；本轮**已结束**时改答案 = **开新分支重走那一轮**（回到那条提问，
  把「这次的选择改了」补在后面重跑；旧那条对话与它已经落盘的改动都留着，可 `◀ ▶` 切回）。
  只有**本轮被刷新/断线掐掉**的「补答」才作为新一轮消息往下聊（那一轮没跑完，没有"重走"可言）。
  ⚠ 旧分支**已经落盘的文件改动不会回退**（agent 不自动撤销）：要退用回滚点 —— 输入框上方
  「运行 / 改动」面板的**文件**标签：每行一个「同意 / 拒绝」（拒绝 = 只回滚**这一个文件**），
  标签栏右侧「↩ 回滚最近一轮」（= 整轮所有文件，都不传参走 `POST /api/session/revert`）。
  面板只列本包写工具（`write_file` / `edit_file` / `apply_patch`）的快照，保留最近 20 轮；
  `run_command` / 外部服务 / MCP 改的文件**不在**快照范围，>512KB 的文件只记标记不回滚。

**发送队列**（`web/src/new/NewApp.jsx` 的 `QueueBar` + `unstable_enableMessageQueue`）：
agent 还在跑的时候继续发消息 → 先进输入框顶部那条队列，本轮结束**依次自动发出**（不是丢弃、也不挡人）。

| 按钮 | 语义 |
|---|---|
| `↑` `↓` | 只调顺序（`queueItem.move`），**不打断当前轮** |
| `↪` 引导 | 登记「下一处模型调用时插进去」（`POST /api/session/steer`；ON 后服务端广播 `steer`，那条随即从队列撤下）；再点撤回 |
| `⚡` | **立即发送**：打断当前轮，马上发它。库里 `move/steer` 只做到「下个发」，所以实现是「先 `POST /api/session/interrupt` 掐断，再把这条直接发出去」 |
| `✎` | 库里没有就地改队列项的接口 → 撤下 + 文本塞回输入框，改完再发 |
| `×` | 从队列删除 |

- **停止**（`ComposerPrimitive.Cancel`）= `POST /api/session/interrupt`：**服务端**也真的停 —— 规划步循环、
  工具循环、作答循环三处都有检查点，停下后照常收尾落盘（若还没进作答，落一句「已停止」说明，不留空回复）。
- 按过停止之后队列会**暂停**：点某条的 `⚡` 或再发一条消息即恢复。
- **队列非空时不重挂对话**：队列活在 assistant-ui 的运行时里，重挂 = 换运行时 = 队列清空 + 当前轮被 abort
  （`runtime.readQueuePending`，见 `session_changed` / `workspace_changed` 的处理）。
- 端到端自测：`wuwor l_agent_chat -- python web/e2e/run_queue_e2e.py [--headed] [--delay 0.8]`
  （真服务 + 慢速假 provider + Playwright，验排队/排序/引导/编辑/删除/停止后恢复）。

**「🧠 复盘」**（每条消息底部，用户消息也有）= 拿**这一轮**的客观证据找问题，并给出**待确认**的改动
（`session_review.py`；与模型自己调的 `optimize_agent` 共用落地层）：

| 环节 | 做什么 |
|---|---|
| 取证 | 从会话文件里那条助手消息取：工具（慢 / 失败 / **同名连调** / **同路径反复读**）、步数、墙钟、工具耗时占比、token、控制流路线与护栏命中（`steps` 里带 `flow` 那条）、图结构与门参数、知识类工具、用问题反向检索知识库（0 命中 = 知识缺口）；另加两块 —— **是否解决**（被中断 / 有没有动手改 / 回答是不是开场白或把球踢回用户 / **结束后用户接着说了什么**）、**验证成本**（测试类命令几条、合计几秒、占墙钟多少、跑没跑整包、带没带 `--headed`、同一条重跑几次） |
| 结论 | 先出**规则式**结论（不调模型也能用，`use_llm=false`），再让模型补写总评与「下次怎么更快更准」（`faster`）；两类结论合并展示。问题分 `repeated_calls / time_waste / amnesia / guard / **unresolved** / **verification** / other` |
| **维度** | 每条问题挂一个 `aspect`（"这问题和哪些方面有关"）：`提示词/技能`、`工具与参数`、`知识库/记忆`、`控制流（脑图）`、**`控制流（表达边界）`**（根因在图上**表达不出来**：缺参数化谓词 / 按次数分叉 / 图内变量 / 通用动作等原语）、`验证方式`、`任务设定`、`模型能力`、`会话与存储`；面板顶部给「维度分布」（`提示词/技能×4 验证方式×2 …`） |
| 待确认改动 | 模型只提**小改动**（`flow_params`：图级/门参数；`notes`；`memory`），由代码拼成合法图 → 过权威校验 + **沙箱路由回放 A/B** → 面板给出**参数级 diff**（原值 → 新值）与 A/B 结论，用户点「应用这些改动」才落地（回放有回归时默认拒收，`force` 才放行，均留痕） |

- 端点：`POST /api/review-turn`（`{mid? | index?, session_id?, use_llm?, provider?, model?}`，同步一次模型往返）、
  `POST /api/review-apply`（`{edits, why, force?, flow_name?, flow_full?}`）。
- 「失忆」= 客观计数：同一路径重复读 / 同一句提问在会话里问过 / 知识库 0 命中；「时间浪费」= 步数、工具耗时占比、护栏催了几次。
- **「提问解决了没有」没有 ground truth**，用四件事拼：① 被中断/中止；② 像要改动却一个字没改；
  ③ 回答是「没说完的开场白」或把球踢回用户（复用护栏 `stall` 那套判据）；④ **结束后用户接着说了什么**
  （追问/报错/不满 → 上一轮多半没解决；这条最实在）。
- **「验证是否臃肿」**：测试类命令（`pytest` / `run_all.py` / `*_e2e` / `--headed` …）几条、合计几秒、
  占墙钟多少、有没有跑整包、有没有带 `--headed`、同一条重跑几次 —— 建议先跑改动相关的子集。
- 图改动前自动备份到 `flows/_versions/<图名>/`；日志记 `_traces/_applied.jsonl`。

### 默认智能体（流程图驱动）

> **2026-10-08：默认智能体的名字就是 `default_intelligent_agent`。** 原来那个内置 `default`
> 让位了（要求原话「任何会话的智能体默认选择 default_intelligent_agent」）——下拉不选就是它、
> 后端不带 `agent` 也回落到它；用户目录里若还留着 `default.json`，那现在只是一个普通智能体
> （可改名可删）。设置页与 `/api/agents` 的 `default` 字段是**默认名的唯一来源**，别在前端写死。

默认智能体「无调用收尾」路由由**流程图**驱动（`flow_engine.run_flow`），不再只靠硬编码护栏：

- **图来源**：`<数据根>/l_agent_chat/flows/default_intelligent_agent.json`（2026-10-04 由
  `default.json` 改名；用户目录优先）→ 内置 `default_flow()` 兜底。
- **可视化编辑**：用 `l_mindmap_mmd`（:8110）打开 `/flow/default_intelligent_agent` 改图，保存即对
  `l_agent_chat` 生效（两边共用同一份 flows 目录，见 `Rez_pkg/l_mindmap_mmd.md`）。
- **开关（三档，可切换）**：
  - `flow_enabled`（默认开）+ `flow_full`（默认关）= **只驱动「无调用收尾路由」**（旧行为）；
  - `flow_enabled=1` + **`flow_full=1`** = **图驱动整个工具循环**：每一步路由都由图决定
    （`branch_calls` 去 tools 还是走收尾路由、`branch_steps` 继续 plan 还是收尾、
    `branch_goal` + 四道护栏门决定收尾去向）；`plan` / `tools` 两个动作节点仍由 app 的
    步循环托管执行（规划是 LLM 往返、工具执行要审批/流式/痕迹，引擎侧没给它们注册 handler）。
  - `flow_enabled=0` = **纯硬编码**（完全不读图）。
  - 图缺失 / 结构不满足（缺 `has_calls`、`steps_exhausted`）/ 引擎报错 → 逐级自动降级：
    全流程 → 收尾路由 → 硬编码，并打一条 `⚠️` mark 说明原因。
- **范围**：不管哪一档，规划与工具**执行**都在 app 里（图决定的是控制流）。
- **兜底**：图加载/执行失败会**自动回退硬编码护栏级联**（`app.py` 包了 try/except），不会卡死对话，
  所以旧硬编码护栏仍在（作为兜底而非默认路径）。
- 默认图结构：`start → plan → branch_calls → branch_goal →（goal_nudge / gate_stall→gate_verify→
  gate_fact→gate_confirm）→ end`，各 gate `on_pass` 沿链推进、`on_trigger` 回 `plan`（回边虚线显示）。

**在对话里怎么看它的专属特性**（2026-10-05 补齐的四层，否则"图驱动"在界面上看不出来）：

| 层 | 在哪 | 内容 |
|---|---|---|
| 路线条 | 每条回答的「过程」流水里，`🧭 控制流` 那一行 | **这一轮实际走了哪些节点**（`plan → branch_calls → gate_stall⁴ → …`），被护栏拦的门**标红**（带次数）；点标题展开全路径，点 `脑图 ↗` 直接开编辑器那张图。**数据落盘**，刷新后还在 |
| 常驻徽标 | 顶栏第二行 `🧭 <图名> · <档位> · N 门` | 当前智能体用哪张图、哪一档；纯硬编码时显示 `🧭 硬编码`（标黄）。切智能体即变 |
| `/flow` 面板 | 点顶栏徽标，或输入框 `/flow` | 档位 + 说明、命中文件（**用户目录 / 包内置**）、节点数、门清单、一键开脑图编辑器；并提示 `/youhua_agent` 可做沙箱回放自查 |
| 策略建议 | 收尾前可能有 `🔧 策略建议（改图后下一轮生效）：…` | 规则式诊断（`flow_spec.diagnose`），只在有建议时出现 |

实现要点（改这块先读）：引擎调用是 `emit_nodes=True`，节点事件 `flow_node` / 护栏事件
`flow_gate` 直接进 SSE；轮末再发一条带 `flow={name,path,gates}` 的**终版** status ——
它落进 steps，前端按同 key 原地更新那一行，所以**刷新/换端后路线条仍能重建**。
后端在 `app.py`：引擎吐 dict 而生成器吐 `_sse(...)` 字符串，转发处必须包装，别退回去 `yield _ev`。

**改图之前怎么验**（2026-10-07 补齐第 ⑤ 层，四层机制见
`Rez_pkg/流程图智能体_图驱动控制流.md` §7.3）：

```bat
:: 先用 recorder_mode=record 录一轮真实对话（设置页「模型调用录制回放」，或 env AGENT_CHAT_RECORDER）
:: 再让现状图与候选图各跑一遍同一盘录像
wuwor l_agent_chat -- l_agent_chat_turn_replay ^
    --cassette <录像名> --ask "<录像里那句问题>" ^
    --flow-a <现状图目录> --flow-b <候选图目录> [--strict] [--live-tools]
:: 等价写法：python -m l_agent_chat.turn_replay（alias 见 package.py）
```

- 报告给的是**相对结论**：候选是否报错 / 收不了尾 / 让护栏一次都不响 / 多绕几步；命中情况
  （严格 / 位次对齐 / 未命中）与"哪一段输入变了"逐条落 `<cassette 目录>/_replay.jsonl`。
- **严格 vs 容错**：指纹要求输入一字不差。改了门上的 `params.prompt`、或工具结果与录像不同
  （工作区变了 / 演练模式把写类拦了）→ 默认 `recorder_loose=1` 按**位次对齐**跑完并标 drift
  （结论只对路由类改动有意义）；`--strict` 则直接报错 —— 宁可失败，不假装验过。
- 回放**不落会话**（`persist: False`）、工具默认走进程级演练模式（`L_AGENT_TOOL_DRY_RUN=1`，只读不改）。

### 工具调用

`l_agent_chat` 复用 `l_agent_tool` 的工具集（文件/命令/Git/HTTP），流程分两种：

- **快速路径**：`run_fast_tools` 直接解析用户消息命中简单工具（读文件/列目录等），
  不触发危险工具。
- **Planner 路径**：`_planner_step` 走**多轮工具循环**——每轮把
  `assistant(tool_calls)` + `role=tool` 的结果追加进 planner 对话（带 `tool_call_id`），
  模型看得到自己已调过什么、结果是什么，因此能不重复地逐步推进；一轮里模型返回多个
  `tool_calls` 时会**并行执行多个工具**。轮数上限 `max_tool_steps`（内置默认 6，设置页可改）。
  命中 `APPROVAL_TOOLS`（`write_file` / `edit_file` / `apply_patch` / `run_command` /
  `run_background` / `kill_port` / `git_push` / `vscode_apply_edit` / `vscode_run_command`）
  时先审批（具体要不要问还受**权限模式**影响，见下节）。工具失败也会把失败结果回填给模型，
  让它换别的工具继续。

- **工具按需声明**（`tool_router.py`）：默认只把**核心集**（读/写/改/搜/命令/后台/`execute`/
  `task`/`ask_user`/待办）声明给模型，其余靠 `find_tools` / `describe_tool` 按需拉出来
  （过程流水里写「工具清单就绪：声明 N/M 个」）。目的是省掉每轮上万 token 的工具 schema。
- **工具清单缓存：过期不阻塞**（`agent_client.available_tools_cached`）：清单 = 本地工具 + 各服务
  探测（并发度 8）+ MCP 握手，每轮重建就是每句先空等几秒，所以进程内缓存（`tools_cache_ttl`，
  默认 300s）。流水里写「缓存命中」还是「现场构建 Xs」就看它。**TTL 一到只后台重建**、本轮先用
  旧清单（显示「缓存命中（后台刷新中）」）—— 隔十几分钟回来问第一句不再先白等：实测冷建 5.56s →
  过期后再取 **0.004s**。只有**指纹变了**（工作区 / 服务表 / MCP 配置）才同步重建，那时旧清单是错的。
- **`ask_user`（向用户提问）**（`ask_user.py`）：信息不足、需要用户拍板（选哪个目录 / 哪个方案 /
  要不要删）时用它**真问一句并等回答**，最长 `ask_timeout`（默认 600s）；用户的回答就是该次工具
  结果，planner 拿着它继续。作答期走 SSE `ask_required` + `POST /api/ask-answer`（与工具审批同一
  条 Future 通道，实现见 `app.py` 工具循环里那段拦截）。界面渲染成**问答卡**：默认（单选）点选项
  **即答**；带 `multi=true` 时（2026-10-07 起）候选变成**可勾选条目** —— 点候选**只切换**、要按
  「确认（N）」才提交，多选取值以 `；` 连成一条文本回来（模型侧拿到的仍是单条文本；此时手填的
  补充会作为最后一条一起提交）。多选标记随 SSE `ask_required`、落盘 `trace.ask` 与 live 轮次都带
  上，刷新 / 多端重取后卡片仍是多选形态。答完点 🔄 可改选 —— 本轮**已结束**时改选 = **开新分支重走那一轮**（回到那条提问、
  把"这次的选择改了"补在后面重跑；旧对话与它已落盘的改动留着，见「消息操作与发送队列」里的
  问答卡与「开分支后会检查文件改动」两段），只有**本轮被刷新/断线掐掉**的「补答」才走
  `thread.append` 发一条新一轮消息（用 `thread.append` 而不是 `aui.composer`：问答卡不在
  composer 子树里，那边的桥没注册）。
  只读模式（ask/plan/review）放行它；**子 agent 一律拿不到**（在 `subagents.DENY_ALWAYS` 里，
  无人值守只会白等到超时）。提问与回答随工具痕迹落盘（`trace` 的 `ask` 字段），刷新后卡片仍在。
- **`restate_question`（先复述用户问题）**（`agent_client.py` + `app.py`）：**合成元工具**，与
  `find_tools` / `describe_tool` 一起**始终声明**；要求 planner 在**第 0 步第一个**调用，用模型
  自己的话把用户问题复述清楚（要什么 / 涉及哪些对象或路径 / 约束与验收点 / 还不确定的地方）。
  **由 harness 保证、不靠提示词照做**：只认第一个调用（排在别的工具之后不算）、内容为空也算没做
  → 追一条纠正消息（`RESTATE_RETRY_NUDGE`）**重试一次** → 仍没有就用 `fallback_restate()`
  从用户原话本地兜底（SSE 事件带 `fallback:true`）。复述经 `compose_reply()` 拼成**答复第一段**
  （`**问题复述**：…`，函数幂等、重试不叠两遍）；**工具卡也显示复述全文**（后端 `_tool_args_view`
  对 `restate` 不按 160 字截断，前端用 `.t-restate` 换行整段渲染）。机制与判据见包内
  `src/l_agent_chat/doc/CHANGELOG.md` 的「先复述用户的问题」条目。

### 审批与权限模式（`permissions.py` / `chat_modes.py`）

**两条正交的轴，别混为一谈**：

- **对话模式**（`agent` / `ask` / `plan` / `review`，见「新版 UI 要点」）= **能力边界**：
  受限模式先把写盘 / 执行命令的工具从可选清单里移除，因此那些模式下基本不会触发审批。
- **权限模式**（`permission_mode`）= **放行方式**：在工具已经可用的前提下，决定「直接跑」
  还是「问一次」。

顺序是**先按对话模式裁工具，再对将要执行的那个调用做权限求值**；权限模式永远不能把对话
模式裁掉的工具放回来。所以两者不会冲突，也不需要谁去重复实现谁。

规则求值 `permissions.evaluate`（对齐 l_kilocode 的 Ruleset）：按顺序找**最后命中**的一条，
`deny` 拦 / `ask` 问 / `allow` 放行；都没命中才是 `none`。最终效果由
`permissions.effective(decision, mode)` 合成：

| 模式 | 含义 | deny | ask | none |
|---|---|---|---|---|
| `default`（默认） | 默认权限：规则说了算 | 拦 | 问 | 回落到 `AUTO_APPROVE` + `APPROVAL_TOOLS`（见下） |
| `allow_all` | 全部允许：除 deny 外一律不问 | 拦 | 放行 | 放行 |
| `autopilot` | 自动巡航（预览）：普通工具自动放行 | 拦 | 受保护路径仍问，其余放行 | 放行 |

- **受保护路径**（改 `settings.json` / `permission_rules.json` / `AGENTS.md` 等写操作）在
  `autopilot` 下**仍会问一次**；只有 `allow_all` 才跳过它。
- `AUTO_APPROVE`（设置页「工具自动批准」，默认开）+ `APPROVAL_TOOLS` 是**旧回落**，仅在
  `default` 模式且规则未命中时生效；`allow_all` / `autopilot` 会覆盖它。
- **口径一致**：聊天审批、WebSocket 终端（`terminal_ws`）、后台进程（`procs`）、文本兜底
  执行都走 `permissions.effective`，不会各判一套。子 agent（`task`）的硬顶只继承父级
  **deny** 规则，任何模式都不放宽。
- 规则分三层（后命中者胜）：内置默认（MCP 工具默认 ask、受保护路径 ask）→ 配置层
  （`permission_rules`）→ saved 层（`<工作区>/.l_agent_ws/permission_rules.json`，审批卡选
  「总是允许」写入）。读写经 `GET / POST / DELETE /api/permission-rules`。

### 终端沙盒化（仅 Windows，`sandbox.py`）

在 Windows 上用 **AppContainer + Job Object** 隔离命令执行（对齐 Kilo 的「终端沙盒化」，
但只做 Windows 原生实现；上游 `kilo-sandbox` 只有 bubblewrap / seatbelt，没有 Windows 后端）。

- 开关 `terminal_sandbox`（默认**关**）。打开后 `run_command`、WebSocket 终端、后台进程
  （`procs`）、CodeMode `execute` 都在 AppContainer 内跑。
- **可写范围**：当前工作区 + 沙盒临时目录（`<工作区状态目录>/sandbox/tmp`）+
  `sandbox_extra_dirs`；其余位置只读（能读，写不了）。默认**禁网**，要联网把
  `sandbox_network` 打开（授予 AppContainer `internetClient` 能力）。
- **不静默降级**：开关开着但沙盒不可用（非 Windows / API 缺失 / 授权失败）时，命令返回
  明确错误，绝不悄悄改成非沙盒执行。
- 能力探测：`GET /api/sandbox/status`；设置页「WebSocket 终端」卡内也会显示可用性。
- 实测限制：
  - 授权走 `icacls … (OI)(CI)M` 写**可继承 ACE**，会传播到目录已有子项并在 ACL 上留痕；
    **大目录首次授权可能较久**（幂等，已存在则跳过）。
  - 执行 cwd 必须在容器可**遍历**的路径下：用户配置目录的父路径不可遍历，PowerShell 会
    `Set-Location` 失败；工作区 / 临时目录应放在容器可访问的位置。
  - 容器很严格，个别工具可能因缺注册表 / 用户目录访问而失败——这是隔离的代价。
  - AppContainer 里 PowerShell 启动会打一条 `InitializeDefaultDrives … 访问被拒绝` 的非致命
    告警：`run_capture` 已从 stderr 剥掉，终端模式按行过滤。

### 改代码与 diff

- `edit_file(path, old_string, new_string, count=1)`：**修改代码首选**——精确替换，比 `write_file`
  整文件重写安全省 token；`count=1` 只换第一处，`0` 换全部。工具在 `l_agent_chat/agent_tools.py`
  内注册进 `l_agent_tool.DEFAULT_TOOLS`（与 `rez_knowledge` 同一套路）。
- **审批预检**：`edit_file` 审批前用 `preview_edit_file` 干跑（不写盘）——审批卡片直接显示 diff，
  且 `old_string` 找不到 / 出现次数不足这类错误提前回填给模型，不必让用户批准一个注定失败的改动。
- **diff 展示**：审批卡片与执行结果都用 `buildDiffBlock` 渲染彩色 diff（`+` 绿 / `-` 红 / `@@` 蓝，
  头部带 `+N -M` 统计）；`tool_result` 事件新增 `diffs: [{path, diff}]` 字段供前端渲染。
- 连续改多个文件时 Planner 不会在第一个 `edit_file` 后停止（只有 `run_command` / `write_file` 成功才停）。

### 上下文压缩

当估算 token 超过 `CONTEXT_LIMIT_TOKENS * COMPRESS_THRESHOLD` 时，把早期对话历史
交给 LLM 压缩成中文摘要，保留最近 `COMPRESS_KEEP_RECENT` 轮，避免上下文超限。

**历史回放只带 role + 正文**（`app.py` 构造 `turns` 处）：工具调用与工具结果**不进** prompt，
只给「这一轮调用过哪些工具」补一行 `[本轮已调用工具：ask_user]`（`_replay_tool_note`）。
不补的话模型不知道上一轮自己问过什么 —— 用户改了那次选择、重走那一轮时（见上面「问答卡」），
它在上下文里找不到那次提问，只能反过来再问一遍。

### 工作区规则

按 CodeMaker `RulesHandler` 的语义把工作区规则注入 system 提示词（实现见 `rules.py`）；
追加位置在「AI 偏好」之后，段首标记「项目规则（严格遵守）：」。

**注入片段的语言（2026-09-26）**：注入到 system / user 的**说明性片段统一用中文** —— 工作区目录、
路径线索、项目规则、用户偏好、远程工具服务、多根说明（`workspace.context_note`）、OpenSpec 索引
（`specs.py`）、规则索引、压缩摘要的四段小标题（`## 目标 / ## 当前进展 / ## 下一步 / ## 相关文件`）。
**刻意保持英文**的是给模型的**任务指令**：`_PLANNER_SYSTEM`（planner 的工具调用规则）、记忆固化 /
召回提示词；记忆召回块的结构键（`record id=/type=/source=/text:` 与围栏
`kilo-memory-v1 targeted_context_not_instruction`）是上游机器格式，也有用例锁着。

| 来源 | 开关（默认开） |
|---|---|
| `<工作区>/AGENTS.md` | `rules_enable_agents_md` |
| `<工作区>/CLAUDE.md` | `rules_enable_claude_md` |
| `<工作区>/.codemaker.codebase.md` | 无独立开关，随总开关 |
| `<工作区>/.codemaker/rules/**` | `rules_enable_codemaker` |
| `<工作区>/.cursor/rules/**` | `rules_enable_cursor` |
| `~/.codemaker/rules/**` | `rules_enable_user` |

`.md` / `.mdc` 用 frontmatter 描述作用域，字段 `name` / `description` / `alwaysApply` / `globs`：

- `alwaysApply: true`（以及 `AGENTS.md` / `CLAUDE.md` / `codebase.md` 这类天然常驻文件）
  → **常驻**，全文注入 system
- 有 `description` 或 `globs` → 只注入「名称 | 适用范围 | 描述 | 路径」索引，正文由模型自己读
- 两者都无 → **不注入**（不静默塞进上下文）

同一 realpath 只取一次（防软链成环）；目录递归有深度上限（`rules_max_depth`）；
扫描结果有 TTL 缓存（`rules_ttl`，默认 5 秒）——因为 `_build_messages` 每个规划步都会调。
总开关 `rules_enabled` 关掉即整段不注入。

### 工具钩子（hooks）

在工具调用前后执行你配置的钩子（实现见 `hooks.py`）：

| 来源 | 说明 |
|---|---|
| `<工作区>/.codemaker/hooks.json` | 项目级 |
| `~/.codemaker/hooks.json` | 用户级 |
| Claude Code 各层 `settings.json` | 仅当 `hooks_sync_cc` 打开，或项目/用户配置里写了 `syncCcHooksConfigs: true` |

- 已接线事件：`PreToolUse` / `PostToolUse`；其余 34 个事件在枚举内但当前不会触发
- handler 四型：`command`（JSON 载荷走 stdin）/ `http`（受 `allowedHttpHookUrls` 白名单门控）/
  `prompt`（需显式写 `model`，LLM 求值由 `app.py` 注入）/ `mcp_tool`（依赖 MCP 客户端）
- `PreToolUse` 返回 `decision: "deny"` → 该工具不执行。判定**在审批之前**：
  规则说不行的，不必再打扰你
- 返回的 `additionalContext` / `updatedToolOutput` 随工具结果回填给模型
- 设置门：`disableAllHooks` / `allowManagedHooksOnly`（后者只有 managed 源算数）；
  `allowedHttpHookUrls` / `httpHookAllowedEnvVars` 取并集
- **全部 fail-open**：钩子自身出错、超时、输出非 JSON 都只写日志，不阻断主流程
- 未实现：hook 信任台账 `hook-trust.json`（本仓无插件体系，没有「不可信来源」的概念）

### MCP（外部工具服务器）

配置：`~/.codemaker/mcps.json`（用户级）+ `<工作区>/.codemaker/mcps.json`（项目级，
按 server 名逐个覆盖）。

```json
{ "mcpServers": {
    "fs": { "command": "npx", "args": ["-y", "@modelcontextprotocol/server-filesystem", "D:/x"] }
} }
```

- 传输：`stdio`（有 `command` 即默认）/ `streamableHttp`（`url`）
- 字段白名单：stdio 认 `type,command,args,env,cwd,timeout,user,token,disabled,autoApprove,autoApproveTools`；
  远程认 `type,url,headers,timeout,disabled,autoApprove,autoApproveTools`。
  未知字段**只警告**；未知传输类型、缺必填、`disabled` 非布尔才是硬错误
- `autoApprove: true` → 该 server 全部工具免审批；`autoApproveTools: [...]` → 只放行列出的
- 自愈：断连先重连再重试一次；`spawn ENOENT` 刷一遍 PATH 后重试；
  stderr 尾巴（UTF-8 乱码时回退 gb18030）会附在错误里
- 逐个 server 隔离失败：某个连不上只跳过它，不影响其它 server 的工具表
- 未实现：`sse` 传输、心跳、stdio 发送停滞看门狗、`resources`/`prompts` 能力探测
- 开关 `mcp_enabled`

#### 市场（环境页 → MCP 服务 → 市场）

内置 7 条精选（filesystem / fetch / git / time / memory / sequential-thinking / playwright）
固定在前，其余从**远程源**拉。

- **默认源是官方 MCP Registry**（`https://registry.modelcontextprotocol.io/v0/servers`，免 key）。
  设置 `mcp_market_url` 可换/加源：**逗号或换行分隔多源**（跨源按 id 去重）；
  填 `file://D:/market.json` 或绝对路径则读**本地清单**（内网离线走这条）。
  留空 = 只用内置精选
- **schema 适配**（`mcp.py`）：`server.name` 作 id、`server.title` 作显示名；分类/标签取
  `server._meta` 的 `publisher-provided`（`categories` / `keywords`）；`remotes[]` → `streamableHttp`
  （`headers[]` 转成安装时要填的参数，`isRequired` 决定必填）；`packages[]` 按 `registryType` 定命令
  （npm → `npx -y <identifier>`、pypi → `uvx <identifier>`、oci → `docker`，`environmentVariables`
  同样转安装参数）。同名多版本按 `_meta` 的 `official.isLatest` **只留最新**
- 翻页**每页 100、最多 6 页**，结果截 300 条。⚠ 官方对 `limit>100` **直接返空**（不是截断），
  所以页大小只能是 100
- **缓存三层**：进程内 5 分钟（`MARKET_TTL`）→ 磁盘 `<存储根>/.l_agent_ws/mcp_market.json`
  → 冷拉。磁盘命中时**先返回旧数据、后台线程刷新**（stale-while-revalidate），所以重启或久置后
  首次打开也是瞬时的；界面「刷新远程」= `?force=1`，跳过前两层同步拉完再返回
- 安装写 `<工作区>/.codemaker/mcps.json`，必填参数在卡片里就地填
- 实测：冷拉 ~2.9s。**HTTP 连接复用**是关键 —— 每页新建连接要多付 ~1.2s 握手（6 页 8.2s → 2.4s）

### 工具行为开关（l_agent_tool）

ignore 治理 / 注册表 PATH 补齐 / rg 后端 这三项属于 `l_agent_tool`（`l_script_editor` 也在用它），
所以**存储不在本包**，而在本机 `~/.lugwit/l_agent_tool/settings.json`
（整份路径可用环境变量 `L_AGENT_TOOL_SETTINGS` 覆盖）。

优先级：**运行时 `set()` > 环境变量 > 该 JSON > 默认值**。
设置页「🧰 工具行为」卡片改完**立即生效**（经 `config.py` 的 `_EXTERNAL` 映射转发），无需重启。

| 设置页字段 | 环境变量 | 默认 | 作用 |
|---|---|---|---|
| `tool_ignore_enabled` | `AGENT_TOOL_IGNORE` | `1` | 启用 `.codemakerignore` 治理 |
| `tool_ignore_root` | `AGENT_TOOL_IGNORE_ROOT` | 空 | 强制指定 ignore 根（空=从目标路径向上找） |
| `tool_shell_env_enabled` | `AGENT_TOOL_SHELL_ENV` | `1` | 子进程并入注册表 Machine + User PATH |
| `tool_shell_env_ttl` | `AGENT_TOOL_SHELL_ENV_TTL` | `60` | 注册表 PATH 缓存秒数 |
| `tool_rg_path` | `AGENT_TOOL_RG` | 空 | rg 可执行文件（空=按 PATH 探测；填 `0` 禁用） |

`l_agent_tool` 是被复用的库，不能反向依赖 `l_agent_chat`，所以它自带这一份最小设置层
（`l_agent_tool/settings.py`），由本包的 `_EXTERNAL` 映射桥接。

### Token 统计

每次 LLM 调用的 `usage`（prompt/completion/total tokens）累计到当前会话 `stats`
字段，可通过 `/api/session/current` 读取，含每次调用明细（`calls`）。

### 会话管理

| 端点 | 方法 | 说明 |
|---|---|---|
| `/api/session/current` | GET | 当前会话 + 消息 + 会话列表 + token 统计；`?session=<id>` = **只读**要那一条（不动"当前会话"指针，多端各看各的用） |
| `/api/session/new` | POST | 新建会话 |
| `/api/session/switch` | POST | 切换会话 |
| `/api/session/delete` | POST | 删除会话 |
| `/api/session/rename` | POST | 重命名会话 |
| `/api/session/interrupt` | POST | **停止本轮**（置中断标记，服务端在规划 / 工具 / 作答三处检查点退出并收尾落盘） |
| `/api/chat/attach` | GET | 附着到**正在跑的那一轮**（刷新 / 重挂后把续写实时接回来；`_live_turn` 现拼在 `/api/session/current` 里） |
| `/api/session/steer` | POST | 「引导」：登记一条消息，在**下一处模型调用**时插进本轮（`/api/session/steer/cancel` 撤回） |
| `/api/session/version` | GET | 取某条回复 / 提问某个版本的正文快照（切换前预览） |
| `/api/session/version/final` | POST | **切分支**：把某版本设为当前落点（它下面的对话随之换成那条分支） |
| `/api/session/version/delete` | POST | 删掉某个版本（连同它的子树） |

会话文件存储于 `<存储根>/.l_agent_ws/sessions/<工作区 key>/session_<id>.json`。

**对话按 `.code-workspace` 工作区隔离**（2026-10-05）：

|  |  |
|---|---|
| 隔离键 | **工作区身份**（`workspace.workspace_identity()`）：**归属自动判定** —— 请求带 `X-Lugwit-Host: vscode`（扩展 webview）用 VS Code 窗口报的文件夹；`browser` / 没声明（浏览器、托盘、脚本）用 agent 自己的 `.code-workspace` 工作区文件。key = `<标签>-<路径哈希8>`（如 `l_rez_src_ws-7201b69c`），直接做目录名。**没有"切换来源"这回事**（2026-10-05 删了两个按钮）—— 扩展就老实做扩展、独立就用自己的。⚠ 判据是**请求声明**，不是"服务端能否连上 VS Code 桥"：网页与扩展共用同一个服务，只看桥会让浏览器也被判成扩展（会话列表凭空换一套） |
| 目录 | `sessions/<工作区 key>/` 一套会话 + 一份 `current.txt`（"当前会话"指针）；换工作区 = 换一整套列表，互不可见 |
| 不隔离的 | 存储根不变：**记忆 / 附件 / 待办 / 检查点 / 事件 / cassette 仍共用一套**（它们按会话 id 或全局落盘） |
| 不等于 | **会话基准（base_folder）**：那是"每个对话各自的视野根"（环境 → 工作区），改它**不会**换列表 |
| 旧数据 | 升级时把平铺在 `sessions/` 下的会话**搬进当前工作区**（一次性、幂等）—— 旧版本所有工作区共用一份，无从分辨归属，只能归到迁移那一刻的工作区 |
| 云同步 | 老条目（`sessions/session_x.json`）拉回来会落到当前工作区；**别的工作区**的云端会话不会出现在当前工作区的"仅云端"里 |
| 界面 | 顶栏 `🗂 <工作区名>` 标出当前在哪个工作区；`/api/session/list` 返回 `workspace:{key,label,source}`，`/api/workspace` 返回 `identity` 与 `sessions_dir` |

**实时保存 / 多端各看各的 / 同名会话**（2026-10-06）：

| 事 | 规则 |
|---|---|
| **实时保存** | 整轮跑完才写一次的老行为改了：提问一发出就把「提问 + 目前攒到的回复/过程」写进会话文件（独立键 `live`，**不进 `messages`** —— 半截助手进了 messages 会在正式落盘时长成多余的分支版本），跑的过程中每 ~3s 节流更新（`LIVE_SAVE_INTERVAL`，工具步之间 + 作答阶段的半截正文）。收尾时清掉快照、写正式消息对 |
| **被打断也留得下** | 进程重启 / 被强杀留下的 `live`：刷新时（`/api/session/current`）超过 `LIVE_STALE_SEC=30s` 没更新就**定格**成正式消息（提问 + 半截回复 + `interrupted` 标记）；30s 内的当"还在跑"照常回给前端（同工作区可能另有一个进程在跑） |
| **多端各发各的** | 落盘一律按请求带来的 `session_id`（`save_messages(sid=...)`），**不再**因为"当前会话指针被别人挪走"就报 `session_switched`「会话已在别处切换」（那是老代码的误报，用户报障"无法继续回答"）。指针只用于"这条工作区当前在看哪条会话"，**不影响**谁往哪条会话写。目标会话已不在 → 报 `session_missing`（明确） |
| **深链只读** | `/chat/<id>` 打开时走 `/api/session/current?session=<id>`（不动指针）：多开窗口时 B 端切走会话，A 端刷新后仍留在自己那条 |
| **同名会话** | 标题 = 第一句提问，问同样的话开两条会重名 → `list_sessions` 给同名的第 2 条起标 `dup`（1 起），界面显示成「标题 (2)」（侧栏 / 标签 / 更多下拉）。只改显示，落盘标题与云端元数据不动 |
| **id 不再撞车** | 会话 id 从"毫秒时间戳"改成"毫秒 + 4 位随机后缀 + 存在性检查"：同一毫秒连建两条（删当前会话会自动新建一条）以前会互相覆盖 |

### 多模型并答（`/multi_ai` 卡片）

> **行为契约（规格）**：`openspec/specs/multi-ai-card/spec.md` —— 以**卡片为操作单位**的不变量
> （之上保留 / 之下清空 / 不新增消息 / 配置不动）、首次阻塞调用、卡片级重跑、同名多路与验收清单。
> 改这块先读它，并同步更新（含文末「规格测试缺口」T2/T5/T8）。

`/multi_ai 你的问题` **不在前端做实现**：后端把这条斜杠展开成"立刻调用 `multi_ai` 工具"的明确指令
（弱模型经常不调工具，只靠 planner 提示词会变成"只有一份回答"，实测 2026-10-08），于是
`multi_ai` 就是一次**普通工具调用**，结果落在**工具痕迹**里（刷新/历史天然可见），前端把那张工具卡
渲染成**并答面板**。

- **首次调用是阻塞的**：先出**配置卡**（选哪几家 / 各家思考档位 / 直接提问），用户点「⚖ 开始并答」
  才真去问（`POST /api/multi/config`；`config.ASK_TIMEOUT` 默认 600s 等不到就按卡里默认跑）。
  真实调用走网关 `POST /v1/chat/completions/multi/stream`（各家并行、**钉死不降级**），
  SSE 帧 `multi_start`（带 models/keys/levels）→ `multi_provider`（各家开始，含实际上游厂商/模型）
  → `multi_delta` / `multi_reasoning` → `multi_done` → `multi_all`。
- **「⚖ 并答」= 只重跑这张卡**（2026-10-10，`POST /api/multi/rerun`）：**不跑规划、不重做复述、
  不再出配置卡**，只按卡上这份「模型 + 各方档位 + 问题」重新问各家（名单留空 = 服务端挑默认几家）。
  落盘时**卡片之上**（过程流水、复述、之前的工具卡、思考）**原样搬过去**，**卡片之下**（这张卡
  之后的工具痕迹与正文）**清空**；结果是这条回复的**新版本**（旧方案连同它的后续留在
  `◀ i/N ▶` 隔壁，可切回）。卡片自己消费那条 SSE 就地刷新各方方案，跑完重载会话换成落盘那一版。
- **「↺ 重置」**：把这张卡与它**之后的后续对话**重置成新分支（走 `session_branches.reset_multi`，
  口径同上：之上保留、卡片方案清空、之下清空）；**模型名单 / 各方档位 / 提问框都保留**
  —— 配置在浏览器 localStorage `lac_multi_ai`（按工作区），重置/并答都不动它。
- **同一个模型可以加两次**（2026-10-10）：名单是**有序、可重复**的；每一路有**实例键**
  （`厂商/模型` / `厂商/模型#2`…，`multi_ai.instance_keys`，前端 `runtime.js` 的 `instKeys` 同一套规则），
  档位按实例键存 —— 同一模型两路可以一个关思考、一个高思考。⚠ 网关按 `request` = spec 认领事件，
  同名两路会互相盖，所以**同名会被拆成两次请求**（各家的第一次出现合并成一次，第 2..N 次各自一次）；
  每路把自己的档位放进该次请求的 `options`（`multi_ai.options_for`）。
- **「✅ 用这个方案」**：把这一格的方案当**指令**交给 agent 执行（工具本身仍只出方案）。
  可点判据 = 这一格**正文或思考有内容**（2026-10-10 修：原来只看正文，而阿里云/火山那几个在「高档」下
  把方案写进**思考**里、正文为空 → 按钮永远点不动，用户报障「点击用这个方案后为啥没用」）。
  **默认 = 在原来那条回复里接着答**（卡上的复选框「就在这条回复里接着答（不新发消息）」默认勾上）：
  不新发用户消息，走「重新生成」通道的 `mode:"plan"` —— 卡片及其之上原样留着、这一轮的运行接在
  下面，结果是这条回复的**新版本**（`◀ i/N ▶` 可切回）。取消勾选则另发一条用户消息。
- **档位**：`off` / `low` / `high` / `max`（与「思考强度」四档同名），逐路可不同；卡头把"本轮实际用的
  档位"和"下次用哪档"分开显示（证据 + 选择）。
- 其他：模型清单来自 hub `/models/available` 的"探过且可用"清单；`multi_ai` 属**写类工具**，
  演练模式（`flow_evolve.dry_run`）下会被拒；同一轮重复调同一个调用签名会被护栏跳过。

### 输入框命令（`/` 与 `@`）

两版 UI 都有，命令表**同源** `web/src/slashCommands.js`（改一处两版都变）：

| 命令 | 行为 |
|---|---|
| `/tools` | 打开**工具选择器弹窗**：数据源 `GET /api/tools`（含本机/远程服务来源前缀），按**分组与服务**分节、**带搜索框**（输入即过滤）、↑↓/Enter/Esc 键盘操作；弹窗 **portal 到 `body` 并底部对齐**（输入框在带 `backdrop-filter` 的 footer 里，`position: fixed` 会被当成新包含块 → 必须 portal）。选中 → 填 `/tools <名称>` |
| `/ls` | 打开**目录浏览面板**（可逐级下钻、★ 收藏）→ 选目录 → 填 `/ls <路径>` |
| `/rez` | 同上，但从 **rez 包名**起（包 → 版本 → 子目录）；选中后插入的是 **`rez_pkg <包名>` 文本**（不是 `/ls <路径>`）—— `/rez` 只是"挑一个包发给 AI"，路由由 `rez_pkg` 指令解释 |
| `/read` `/ws` `/workspace` | 直接把命令名填进输入框，参数自己接着打 |

- **tag/chip 只影响显示**：`/tools <名称>`、`rez_pkg <包名>`、`/ls <路径>` 这类 token 在输入框与消息气泡里渲染成**标签（chip）**，
  **发送时仍原样保留文本** —— AI 收到的就是真实命令，UI 只是把命令名与参数显示成 chip（`web/src/new/rezTag.jsx`：
  `renderTags()` + `chipLabel()`，`/rez` 与 `rez_pkg` 同样识别）。
- **拼写检查**：输入框开 `spellCheck` + `lang="en"`（浏览器原生红波浪线；中文习惯环境误报可忽略，故意不开第三方词库 —— 廉价、零请求）。
- 触发：输入 `/` 弹命令菜单（↑↓ 选、Enter/Tab 确认、Esc 关闭）；输入 `@` 弹**文件补全**
  （异步查后端、防抖），选中插入 `@相对路径 `。两者的**指令语义都在后端** ——
  例如 `/ls` 由 `agent_tools.py` 解释，客户端只负责补全与插入，不做实现。
- 新版 `#/new` 用的是 assistant-ui 自带的 `ComposerPrimitive.Unstable_TriggerPopover`
  + `unstable_useSlashCommandAdapter` / `unstable_useLiveCompletionAdapter`
  （见 `web/src/new/ComposerTriggers.jsx`）；旧版 `/classic` 是自研菜单（`CommandMenu.jsx`）。
  浏览面板与工具选择器**两版复用同两个组件**（`components/BrowsePanel.jsx` / `ToolPicker.jsx`），
  收藏目录也共用同一份 localStorage（`l_agent_chat.favs`）。
- 匹配是**模糊子序列**：`/ls` 也会命中 `/tools`（`t-o-o-l-s` 含 `l`,`s`），与旧版行为一致 ——
  想精确选就多打几个字符或用 ↑↓。

**新版实现踩过的三个坑**（都在 `ComposerTriggers.jsx` 的注释里，别踩回去）：

1. **TriggerItem 必须带 `type` 字段**：`{id, type, label}`。缺 `type` 时菜单能出来、鼠标点也能插入，
   但 **Enter 只会把菜单关掉、不插入**。
2. **必须自定义 `formatter`**：默认的 `unstable_defaultDirectiveFormatter` 插的是
   `:command[/read]{name=read}` 这种 directive chip，**后端不认**。我们要的是纯文本
   `/read ` 与 `@路径 `，所以 `serialize` 原样返回命令名/路径、`parse` 只当纯文本。
3. **`/` 按钮不能用 `setText`**：库靠真实的输入/光标事件判定触发器，programmatic 改 text
   只会在框里出现 `/`、菜单不开（且回车会把裸 `/` 当消息发出去）。要 `el.focus()` +
   `document.execCommand("insertText", false, "/")`。

### 工具服务（含本机自动发现）

工具调用发生在**聊天服务所在那台机器**上，与用户从哪台设备打开网页无关；服务清单存在
`<存储根>/.l_agent_ws/tool_services.json`（每个工作区一份）。

- **本机自动发现**：读 `~/.Lugwit/run/<service>.json`（可用 `LUGWIT_RUN_DIR` 覆盖），
  没有发现文件时兜底探测 `127.0.0.1:${SCRIPT_EDITOR_HTTP_PORT:-8764}`；命中即作为内置
  服务（`builtin: true`，名称后缀「（本机自动发现）」，不可编辑/删除，只能选用/测试）。
  因此**从手机浏览器打开**访问服务器实例时，用的就是服务器自己那台机器的脚本编辑器。
- **手工服务**：在设置页添加外部同类服务（如对端 PC 的 `http://<ip>:8764`）后，Agent 也能调。
- **调用协议**：优先 `POST {url}/call_tool`；服务未实现（404/405）时自动退回
  `POST {url}/execute` + 执行环境里的 `agent` 命名空间
  （`agent.list_tools()` / `agent.call_tool(name, **args)`）—— 现网 l_script_editor 属于后者。
- 模型看到的工具描述带来源前缀：远程为 `[远程服务 <名称> @ <url>]`，本机为 `[本机服务 <名称>]`；
  本机服务与本地内置工具同名的会被去重（同一台机器同一套实现）。

| 端点 | 说明 |
|---|---|
| `GET /api/tool-services` | 列出服务（含本机自动发现的 `builtin` 服务） |
| `POST /api/tool-services` | 创建服务 |
| `POST /api/tool-services/activate` | 激活服务 |
| `POST /api/tool-services/test` | 测试连通（`/tools` 不可用时自动用 `/execute` 发现） |
| `POST /api/tool-services/invoke` | 调用远端服务的一个工具 |
| `POST/DELETE /api/tool-services/<sid>` | 更新/删除（内置服务不支持删除） |

### 其他端点

| 端点 | 说明 |
|---|---|
| `GET /` | 聊天网页 |
| `GET /health` | 健康检查 |
| `GET /api/models` | 模型列表 |
| `GET/POST /api/settings` | 读取/更新运行配置 |
| `GET /api/workspace` | 工作区信息（`workspace_file` = agent 自己那份 `.code-workspace`；`source` 是**自动判定**的，`source_auto: true`；`identity` / `sessions_dir` 见「会话管理」） |
| `POST /api/workspace` | 设置工作区文件根（`POST /api/workspace/source` 已随"来源自动判定"删除） |
| `GET/POST /api/workspace/root` | 工作区根目录管理 |
| `POST /api/workspace/root/activate` | 激活根目录（= 把该项移到 `folders` 首位） |
| `POST /api/ask-answer` | 回答 `ask_user` 的提问（SSE `ask_required` 之后调用）。两种入参取到哪个用哪个：`{ask_id, answer}`（单选 / 自由输入）或 `{ask_id, answers:[…]}`（**多选**，用 `；` 连成一条） |
| `GET/POST /__dev__/prompt-flags` | 两条**提示词实验片段**的运行期开关（`planner_batch_hint` / `answer_write_tight`；POST 传 `null` = 清覆盖、回退 config）。默认都关，A/B 与排障用，只影响本进程 |
| `GET /api/browse` | 浏览目录 |
| `GET /api/browse_rez` | 多级浏览 rez 包仓库 |
| `GET /api/tools` | 工具清单（工具选择器数据源：本地内置 + 各工具服务的工具，带来源分组） |
| `POST /api/translate` | 免费翻译优先、失败回退 AI；`lines: true` → **逐行对照**（返回 `{pairs, source, translated, total}`，只翻含字母的行） |
| `POST /api/voice/tts` | Edge-TTS 合成（`voice` 指定音色；响应体是音频字节） |
| `GET /api/voice/voices` | Edge-TTS 音色表（可按 locale 过滤，供语音下拉） |
| `GET /api/source` | 读源码上下文（`path`/`start`/`end`/`context`，上限 400 行）供源码弹窗 |
| `POST /api/prompt/optimize` | 提示词优化（草稿 → 更清晰的提示词；独立小模型，见 `prompt_optimizer.py`） |
| `POST /api/spell/check` | **发送前拼写检查**（`{text}` → `{changed, issues[], corrected}`；独立小模型，开关/超时是设置 `spell_check_*`，见 `spell_check.py`） |
| `GET/POST/DELETE /api/spell/experience` | **纠错经验库**（看 / 记 / 清）：`POST {action:"corrected",original,final}` 记正向偏好（后端 diff 出 `from → to`）、`{action:"as-is",issues}` 记"别改这个词"（累计 2 次生效）；`DELETE ?from=<词>` 清单条。见 `spell_experience.py` |
| `POST /api/review-turn` | **按消息复盘**（消息底部「🧠 复盘」）：取证这一轮 → 结论 → 待确认改动 + 沙箱 A/B（见「消息操作与发送队列」） |
| `POST /api/session/branch-check` | **分支后的文件改动**拍板：`{action: "keep" | "undo", files?}` —— 保留只清标记；撤销用那些改动**之前**的快照写回（不给 `files` = 全撤，见 `branch_changes.py`） |
| `POST /api/review-apply` | 落地复盘里**用户确认过**的改动（图参数 / 笔记 / 记忆；改图先备份，见 `flow_evolve.apply_edits`） |
| `GET /api/sandbox/status` | 终端沙盒能力探测（仅 Windows，AppContainer） |
| `GET/POST/DELETE /api/permission-rules` | 权限规则（saved 层）列表 / 追加 / 删除 |

## 回归测试

```bat
wuwor l_agent_chat -- l_agent_chat_test              rem 跑 tests/ 全部用例，退出码非 0 即失败
wuwor l_agent_chat -- python tests/run_all.py -v      rem 逐条看用例名
```

> ⚠ `run_all.py` 是**相对路径**写法：它按 `__file__` 定位包根，但 `tests/run_all.py` 这个**入口路径**
> 要在**包目录**下执行才找得到 —— 从仓库根或别处跑请给绝对路径
> （`wuwor l_agent_chat -- python "<包>/999.0/tests/run_all.py" --endpoint --jobs 6`）。

**改什么跑什么**（用户口径 2026-10-10：「**只跑点名类 + 相关 e2e**」，日常**不跑整批**；
同样写在 `.cursor/rules/l_agent_chat-改什么跑什么.mdc`）：

| 改了什么 | 跑什么（点名为主） |
|---|---|
| `web/src/**`（JSX / JS / CSS） | Python 批**测不到它** → `l_agent_chat_web_build` + 相关 e2e + 浏览器点一遍 |
| 只改注释 / 文档 | **不跑** |
| 单个 `src/l_agent_chat/<模块>.py` | 点名覆盖它的类（`loadTestsFromNames([...])`，几秒） |
| `app.py` 的 SSE / 审批 / 路由 / 会话流 / 工具循环 | 点名 `test_agent_endpoint` 里**相关的类** + 相关 e2e |
| `tools/**` 治理脚本 | `tests/test_gov_scripts.py`（它扫全包，别跳） |
| 测试夹具（`fake_llm.py` / `agent_server.py`） | 点名受影响的类 + 相关 e2e |

**整批**（默认批 572 项 / ~45s；端点批 94 项 / ~95s）**只在这两种时候跑**：
① 改动触及**共享面**（测试夹具、`session_store`/`session_branches` 会话层、`app.py` 的 SSE 管道 /
会话锁 / 工具循环主干）；② **发版/交付前**。理由：整批的价值在"你没想到的耦合"，而它测不到前端、
也验不了 prompt/阈值类改动 —— 把它留给真正有耦合风险的时候。

> ⚠ **两层硬拦，默认拒整批**：① **`wuwo` 启动器层**（`wuwo/py_modules/wuwo_rez.py`
> `_guard_full_batch`）—— 唯一的公共咽喉，agent 与人都必经 `wuwor`，命中即拒（退出码 2）；
> ② **`l_agent_chat` 工具层**（`l_agent_tool.agent_tools.guard_full_test_batch`，挂在 `run_command` /
> `execute` / `execute_sync`）。判据窄：解释器后跟 `run_all.py`，或整词 `l_agent_chat_test`；
> `grep`/`type`/`git log` 只是提到文件名 → 放行。**要跑整批必须显式加 `--full-ok`**（会被摘掉再执行），
> 或设 `LAC_ALLOW_FULL_TESTS=1`。用例：`tests/test_batch_guard.py`。




**写端点用例的两条契约**（2026-10-10 清陈旧用例时总结，踩过就别再踩）：

1. **每一轮回复列表以 `fake_llm.restate_call()` 开头**：harness 强制「第 0 步第一个调用必须是
   `restate_question`」，模型没做会**追一次重试**，脚本化回复整体错位一条 —— 断言会拿到兜底文本
   `done`，看着像实现坏了（当年 `EndpointTest` / `ManualCompactionTest` 那批就是这么假失败的）。
   于是"一轮 = 复述步 + 判断步 + 作答"**三次**模型调用，别按两次写死。
2. **别按请求下标认身份**（`fake.requests[1]` 这类）：复述步会让下标整体漂一位 —— 按**内容**认
   （如"含任务描述且不含父会话暗号的那一次 = 子 agent 的请求"）。会话文件也别拼老路径
   `sessions/session_<id>.json`，2026-10-05 起按**工作区 key** 分子目录，用 `rglob` 找。
   环境依赖同理：`LUGWIT_USER` 来自实例数据目录/包内 `.env`（开发机登录过就有值），
   要"未登录"的场景在用例里显式传 `LUGWIT_USER=""`。

覆盖范围：

- **工作区规则 / OpenSpec 索引注入**：frontmatter 四字段、always·index·skip 分档、来源开关、
  两条路径（直连问答 `_build_messages` 与 **planner 工具循环** `_planner_messages`）
- **工作区文件（`.code-workspace`）**：`tests/test_workspace_file.py` —— 默认位置 / 相对路径解析 /
  `folders[0]` 基准 / 写回保留未知键与显式 `name` / 切基准与删除的顺序语义 / 不再产出 `config.json`
- **内联工具调用兜底**：`tests/test_inline_tool_calls.py` —— `seed:tool_call` 与 `minimax:tool_call`
  两种模板解析、幻觉工具名丢弃、流式过滤器逐字符切块边界、planner 兜底、`model_cb` 上报实际模型
- **`ask_user`**：`tests/test_ask_user.py`（注册与参数、只读模式放行、子 agent 被 `DENY_ALWAYS` 拦掉、
  选项规范化）+ `tests/test_agent_endpoint.py` 的 `AskUserTest`（真起服务：SSE `ask_required` → 回答
  进工具结果；超时给 `NO_ANSWER` 仍能正常收尾）—— 端点批要 `python tests/run_all.py --endpoint`
- **检索缓存 / 工具失败显示 / 提示词实验开关**：`tests/test_notepad_cache.py`（检索同参数只打一次
  HTTP、带 kb 时忽略 sources、TTL 过期重查、缓存有上限、预热失败静默）、
  `tests/test_tool_result_errors.py`（失败结果渲染成「执行失败」而不是误导的「(空文件)」；
  `_dedupe_guard` 的 dup / 同目标重复读 / **已读区间提示**）、`tests/test_prompt_flags.py`
  （两条实验片段**默认关**、可运行期开、未知开关报错）
- **hooks**：matcher 语义、可阻断集合、决策合并（deny 只在可阻断生效 / ask 不覆盖 deny /
  `hookEventName` 不符丢弃）、**真起子进程**走 stdin-stdout 的端到端、Claude Code 四层设置来源
- **planner 工具循环**：请求形状、工具往返、HTTP 错误 fail-open
- **端点级**（真起 uvicorn + 假 provider）：SSE 流式回复、工具调用往返、hook deny、
  审批**放行 / 拒绝 / 超时**、`.codemakerignore` 拦读、工具失败也必须发事件
- **权限模式**：三档合成（`deny` 永远拦 / `allow_all` 全放行 / `autopilot` 保受保护路径的
  `ask`）、`normalize_mode` 回退（`test_permissions.py`）
- **对话模式**：受限模式裁掉写 / 执行工具、与权限模式正交（`test_chat_modes.py`）
- **消息版本分支 / 重新编辑**：`tests/test_message_versions.py`、`tests/test_regenerate_turn.py`（重新生成
  = 追加版本 + 切回旧分支）、`tests/test_edit_user_message.py`（改提问 = 开新分支，旧提问与后续都在；
  拒绝错目标时会话不动；标题跟随当前分支）；浏览器端到端见 `web/e2e/run_queue_e2e.py`
- **会话隔离 / 工作区归属**：`tests/test_workspace_isolation.py`、`tests/test_workspace_file.py`、
  `tests/test_depot_workspace.py`（会话按 `.code-workspace` 工作区隔离、云端老条目落当前工作区）
- **实时保存 / 多端 / 同名会话**：`tests/test_session_realtime.py`（边跑边存 + 收尾清快照、残留快照定格、
  往非当前会话发不报错且落对文件、会话不存在报 `session_missing`、同名 `dup`、连建不撞 id）
- **按消息复盘**：`tests/test_session_review.py`（证据与规则式结论、模型报告 + 参数级 diff + 沙箱 A/B、
  确认后落地、定位不到就明确报错；用例**隔离 flows 目录**，别改到开发机真实那份图）
- **分支后的文件改动检查**：`tests/test_branch_changes.py`（开分支列出被放弃那条线改过的文件 + 行数、
  保留只清标记、撤销把新建文件删掉、没改文件不弹、重新生成也记）
- **整轮 LLM 回放**：`tests/test_turn_replay.py`（容错取条目"严格 → 位次对齐 → 用尽"三级、
  回放日志、游标按轮重置、输入指纹分段、进程级演练开关 `L_AGENT_TOOL_DRY_RUN`；端到端**真录一轮**
  再用同一盘录像让两版图各跑一遍：同工作区**严格命中**、换工作区**严格模式报错 / 容错模式跑完并记 drift**）
- **终端沙盒**（仅 Windows；默认批**跳过**，`set LAC_SANDBOX_TESTS=1` 才跑真跑用例）：
  AppContainer 内执行、授权目录可写、**未授权路径写入被拒**、超时可杀、管道收发
  （`test_sandbox.py`）

设计要点：

- 全部用标准库 `unittest`，**不引入新依赖**。环境里没有 `httpx`（`TestClient` 用不了），
  所以端点级走**真 uvicorn 子进程 + `requests`**，顺带把 SSE 分帧、审批回传这些真 socket 行为一起测到
- `tests/fake_llm.py` 是假的 OpenAI 兼容服务（含 SSE 分支）；provider 用
  `L_MODEL_HUB_USER_JSON` 把 `deepseek` 的 `api_base` 重定向过去 → 结果确定、不花钱、不依赖外网
- `tests/agent_server.py` 用 `AGENT_CHAT_WORKSPACE` 指向临时目录，**不碰本机的工作区 / 设置 / 会话**
- 注意：一句用户输入会打两次模型（先 planner 判断要不要工具，再由最终答复走流式），
  写假回复脚本时要留两条

## 配置项

设置页写入的配置存于 `<工作区>/.l_agent_ws/settings.json`，重启后仍生效。
优先级：环境变量 > 设置文件 > 内置默认。

| 配置键 | 环境变量 | 默认 | 说明 |
|---|---|---|---|
| `ai_provider` | `AGENT_CHAT_PROVIDER` | `volcengine` | 默认供应商（siliconflow/minimax/zhipu/deepseek/volcengine/aliyun/wuzu） |
| `<供应商>_model` | `AGENT_CHAT_MODEL` | 各供应商默认模型 | 各供应商模型 ID（火山方舟默认 `deepseek-v4-flash`） |
| — | `<供应商>_API_KEY` | 中心密钥存储 | API 密钥统一存 **lugwit_auth 的中心密钥存储**（命名空间 `model_hub`，PG 密文；本机无明文密钥文件）。本包经 hub 的 `GET /v1/keys` 取（回环），不落 settings.json |
| — | `AGENT_CHAT_API_URL` | 按供应商推导 | 全局 API 地址覆盖（调试用） |
| `host` | `AGENT_CHAT_HOST` | `127.0.0.1` | 监听地址（局域网设 `0.0.0.0`） |
| `port` | `AGENT_CHAT_PORT` | `1250` | 服务端口 |
| `temperature` | — | `0.1` | 采样温度 |
| `timeout` | — | `120` | 请求超时（秒） |
| `max_history_turns` | — | `20` | 携带历史轮数（只带 role+正文，助手轮附「本轮已调用工具」一行） |
| `max_tool_steps` | — | `6` | 工具规划最大步数 |
| `context_limit_tokens` | — | `6000` | 上下文压缩阈值（估算 token） |
| `compress_threshold` | — | `0.85` | 达到阈值比例触发压缩 |
| `compress_keep_recent` | — | `6` | 压缩时保留最近轮数 |
| `translator_backend` | `AGENT_CHAT_TRANSLATOR` | `baidu` | 翻译后端（baidu/mymemory/ai） |
| `prompt_optimize_provider` | — | `zhipu` | 提示词优化供应商（**独立于聊天默认模型**，缺省智谱） |
| `prompt_optimize_model` | — | `glm-4-flash` | 提示词优化模型（免费档，不占主力额度） |
| `spell_check_enabled` | — | `1` | **发送前拼写检查**总开关（关掉 = 发送不再多一次往返；AI/脚本经 `POST /api/settings` 也能改） |
| `spell_check_timeout` | — | `12` | 拼写检查那次模型调用的硬上限（秒）；**超时按「没查出问题」放行** |
| `spell_check_provider` | — | `zhipu` | 拼写检查供应商（独立于聊天默认模型） |
| `spell_check_model` | — | `glm-4-flash` | 拼写检查模型（免费档） |
| `spell_learn_enabled` | — | `1` | **纠错经验库**总开关（记 + 注入；关掉 = 不学也不用） |
| `permission_mode` | — | `default` | 权限模式：`default` / `allow_all` / `autopilot`（见「审批与权限模式」） |
| `auto_approve` | — | `1` | 工具自动批准（**仅 `default` 模式**的旧回落；`allow_all`/`autopilot` 覆盖它） |
| `mobile_msg_height` | — | `66` | 移动端单条消息气泡限高（屏幕高度百分比，`0` = 不限；≤860px 生效） |
| `rez_roots` | `AGENT_CHAT_REZ_ROOTS` | 源码+3rd 仓库 | rez 包仓库根（`;` 分隔） |
| `rules_enabled` | `AGENT_CHAT_RULES_ENABLED` | `1` | 工作区规则注入总开关 |
| `rules_enable_agents_md` | — | `1` | 读工作区根 `AGENTS.md` |
| `rules_enable_claude_md` | — | `1` | 读工作区根 `CLAUDE.md` |
| `rules_enable_codemaker` | — | `1` | 递归读 `<工作区>/.codemaker/rules` |
| `rules_enable_cursor` | — | `1` | 递归读 `<工作区>/.cursor/rules` |
| `rules_enable_user` | — | `1` | 读 `~/.codemaker/rules` |
| `rules_max_chars` | — | `20000` | 常驻规则正文上限（字符） |
| `rules_index_chars` | — | `4000` | 按需规则索引上限（字符） |
| `rules_max_depth` | — | `4` | 规则目录递归深度上限 |
| `rules_ttl` | — | `5` | 规则扫描缓存秒数 |
| `specs_enabled` | — | `1` | OpenSpec 索引开关 |
| `specs_max_chars` | — | `4000` | OpenSpec 索引字符上限 |
| `specs_ttl` | — | `5` | OpenSpec 扫描缓存秒数 |
| `hooks_enabled` | — | `1` | 工具钩子总开关 |
| `hooks_timeout` | — | `10` | 单个 hook 超时秒数 |
| `hooks_sync_cc` | — | `0` | 并入 Claude Code 各层 `settings.json` 的 hooks（默认关，读别的产品的配置该显式选择） |
| `mcp_enabled` | — | `1` | MCP 客户端开关 |
| `mcp_market_url` | — | 官方 MCP Registry | MCP 市场源（逗号/换行分隔多源；`file://` 或绝对路径 = 本地清单；留空 = 只用内置精选） |
| `terminal_enabled` | `AGENT_CHAT_TERMINAL_ENABLED` | `0` | WebSocket 终端（`ws://…/ws/terminal`）开关 |
| `terminal_sandbox` | `AGENT_CHAT_TERMINAL_SANDBOX` | `0` | 终端沙盒化（**仅 Windows**，AppContainer；见「终端沙盒化」） |
| `sandbox_network` | — | `0` | 沙盒内允许联网（授予 AppContainer `internetClient` 能力） |
| `sandbox_extra_dirs` | — | 空 | 沙盒额外可写目录（分号串 / 列表；工作区与沙盒临时目录之外） |
| `tool_content_expanded` | — | `0` | 工具生成内容默认是否展开显示 |
| `recorder_mode` | `AGENT_CHAT_RECORDER` | `off` | 模型调用录制回放：`off` / `record` / `replay`（replay 不联网，缺条目直接报错） |
| `recorder_cassette` | `AGENT_CHAT_RECORDER_CASSETTE` | `default` | 录像名（`<名>.jsonl`） |
| `recorder_dir` | `AGENT_CHAT_RECORDER_DIR` | 空=工作区 `.l_agent_ws/cassettes` | 录像目录（录制/回放两个进程共享时用绝对路径） |
| `recorder_loose` | `AGENT_CHAT_RECORDER_LOOSE` | `0` | **容错回放**：严格 miss 时按「同一轮对话 + 文件顺序」位次对齐跑完，记 drift（结论打折） |
| `recorder_run` | `AGENT_CHAT_RECORDER_RUN` | 空=auto | 回放标记（写进 `_replay.jsonl` 的 `run`，整轮 A/B 按它筛本次条目） |
| `planner_batch_hint` | — | `0` | 规划 system 那段「互不依赖的调用同一步发完」。**默认关**：受控 A/B（4 臂 × 2 题 × 2 样本）实测**没减步数、反而更慢**（Q1 66.7s→88.7s、`path:line` 12.5→6），故关掉。运行期可开：`POST /__dev__/prompt-flags` |
| `answer_write_tight` | — | `0` | 作答 system 那段「证据/未验证项一行一条、别粘工具结果原文」。**默认关**：受控 A/B 里字数/引用变化全落在噪声内（早期 2 个**非受控**样本看着"更有料"，没能复现） |

## 依赖该包的包

| 包名 | 用途 |
|---|---|
| `start_multi_app` | 多应用启动（引入 `l_agent_chat`） |

## 端口占用处理

启动器启动前会检查端口：被占用时打印 PID/进程名/exe 路径并询问是否结束；
`-y` 直接结束。会剔除 netstat/psutil 中的僵尸 LISTENING 残留，并清理
历史遗留的 `uvicorn --reload` supervisor + worker 双进程树（本包现已不用那条路径）。

Windows 独占绑定防共享（治本）：

- uvicorn 默认给监听 socket 设 `SO_REUSEADDR`，Windows 下允许多进程共享绑定
  同一端口——旧进程残留时请求会被路由到旧代码（"改了源码不生效"的元凶）。
- 启动器在调用 `uvicorn.run` 前打补丁（`_patch_uvicorn_exclusive_bind`），
  把监听 socket 改为 `SO_EXCLUSIVEADDRUSE` 独占绑定：端口被占用时直接报错，
  绝不静默共享。**单进程**运行，socket 由本进程自己创建，所以在这里补即可生效
  （重启不涉及 socket 交接，没有重新绑定竞态）。
- 端口探测 `_port_bindable` 同样用独占绑定，避免 `SO_REUSEADDR` 误判"已释放"。
- 僵尸 socket（netstat 显示 LISTENING 但 PID 已死，句柄被后代继承持有）：
  自动找出死 PID 的存活后代并结束（`_clear_zombie_holders`）。
- 启动自检：启动时生成 `boot_id` 注入环境变量，后台线程轮询 `/health`
  核对回显的 `boot_id`，确认应答者确为本次启动的进程。

## 常见问题

- **改了源码不生效**：先看**是不是 `.py`**。`.py` / 启动期配置**不保证**热重载 ——
  热重载要带 `.dev_mod` 启动（硬门控）且 `L_SRC_WATCH` 未被关掉；开发机实测
  「watchfiles 首次重载后可能停摆」。不确定就直接重启：
  `wuwor l_agent_chat -- python -m l_agent_chat.app restart_self_cli`（停旧起新）。
  另一类原因是**旧进程残留抢端口**：独占绑定下会直接报错而非静默抢请求，
  用 `-y` 重启即可自动清理（含僵尸 socket）。
- **API key 报错**：密钥统一在 **lugwit_auth 的中心密钥存储**（命名空间 `model_hub`，PG 密文；
  环境变量 `<供应商>_API_KEY` 仍最高优先）；本包经 hub 的 `GET /v1/keys` 取，hub/auth 不通会
  明确报错而不是回退本地文件。设置页只读展示各供应商密钥状态。
- **公网/手机访问时看不到流式、审批总是被拒（user_rejected）**：nginx 默认会缓冲上游响应，
  SSE 会攒到请求结束才一次性下发（人工审批 120s 超时即判为拒绝）。应用侧已在 SSE 响应加
  `X-Accel-Buffering: no` 关掉本响应的缓冲；若仍被缓冲，可在 nginx 的 `location /agent_chat/`
  里加 `proxy_buffering off; proxy_cache off;`（`/v1/chat/completions` 段已有同样配置）。
- **改代码后整个文件都变成已修改（git diff 全红）**：`edit_file` 会按原文件换行风格写回
  （CRLF 文件仍 CRLF、LF 文件仍 LF），如遇到请确认是原文件本身混用了换行。
- **启动报"端口仍不可用（僵尸 socket 或无权限）"**： zombie 持有者不在本用户
  进程树内（如系统服务占用），手动 `netstat -ano | findstr :1250` 排查或重启系统。
- **反代子路径下页面跳错地方（如设置页「返回聊天」跳到 `/homepage`）**：模板里写了**绝对路径** `/…`。
  页面挂在 `/agent_chat/` 下时 `/` 是 nginx 根（托盘主页），不是聊天页。正确做法是按前缀拼：
  `react.html` 用注入的 `window.__API_BASE`；`settings.html` 用 `(location.pathname||'/').replace(/\/[^\/]*$/,'')`
  剥掉最后一段当前缀（两者算法一致；直连 `:1250` 时前缀为空）。新增模板/按钮时别写 `href="/…"`。
