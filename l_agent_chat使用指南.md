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
|------|-----|
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
npm run dev:server   rem Vite dev server（127.0.0.1:5174，HMR，/api 代理到 :1250）
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
  "reasoning_effort": "high",
  "show_reasoning": true
}
```

> 注：`thinking` / `reasoning_effort` 控制是否启用思考；`show_reasoning` **只作记录**，
> 后端不再据此裁剪 `reasoning` 事件（见下表），渲染与否交给前端开关。
> `mode` 是对话模式（`agent` 默认 / `ask` / `plan` / `review`，见「新版 UI 要点」），
> 缺省或未知值按 `agent` 处理。

SSE 事件类型：

| 事件 | 说明 |
|------|------|
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
- **翻译**：思考块展开后右上角有「🌐 翻译」，调 `POST /api/translate` 翻译整段思考（免费后端优先、失败回退 AI），再点收起译文。
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
- **设置页窄屏**：宽 ≤860px 时左侧「设置分组」侧栏变顶部横排，可**按住拖动**横向滚动（桌面仍为竖排）。
- **手机端与老内核兼容**：Tailwind v4 把工具类全塞进 `@layer`，而「荣耀自带浏览器」这类老 Chromium
  内核（<99）**不认 `@layer`** —— 未知 at-rule 整块丢弃，工具类一个不剩（侧栏 `hidden`、`truncate`
  全失效，页面塌成无样式单列）。构建期用 `@csstools/postcss-cascade-layers` 把层拍平，且**必须跑在
  `generateBundle`**（见 `web/vite.config.js` 的 `cascadeLayersPlugin`）：逐 CSS 模块跑时它看不见
  无层那份 style.css，会把无层/层内优先级算反，反而压掉旧 UI 的 `.btn`。
  手机端输入框高度也在移动端媒体查询里调大（`.composer-input { min-height: 64px }`）。

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
- **`ask_user`（向用户提问）**（`ask_user.py`）：信息不足、需要用户拍板（选哪个目录 / 哪个方案 /
  要不要删）时用它**真问一句并等回答**，最长 `ask_timeout`（默认 600s）；用户的回答就是该次工具
  结果，planner 拿着它继续。作答期走 SSE `ask_required` + `POST /api/ask-answer`（与工具审批同一
  条 Future 通道，实现见 `app.py` 工具循环里那段拦截）。界面渲染成**问答卡**（选项按钮 + 自定义
  输入框），答完点 🔄 可改选 —— 改选是**发一条新一轮用户消息**（已发生的那次工具结果改不了），
  走 `thread.append` 而不是 `aui.composer`（问答卡不在 composer 子树里，那边的桥没注册）。
  只读模式（ask/plan/review）放行它；**子 agent 一律拿不到**（在 `subagents.DENY_ALWAYS` 里，
  无人值守只会白等到超时）。提问与回答随工具痕迹落盘（`trace` 的 `ask` 字段），刷新后卡片仍在。

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
|------|------|------|-----|------|
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
不补的话模型不知道上一轮自己问过什么 —— 用户点「🔄 重新回答」发出「关于上面的问题「…」」时，
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
|------|----------------|
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
|------|------|
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
|------|------|------|
| `/api/session/current` | GET | 当前会话 + 消息 + 会话列表 + token 统计 |
| `/api/session/new` | POST | 新建会话 |
| `/api/session/switch` | POST | 切换会话 |
| `/api/session/delete` | POST | 删除会话 |
| `/api/session/rename` | POST | 重命名会话 |

会话文件存储于 `<工作区>/.l_agent_ws/sessions/session_<id>.json`。

### 输入框命令（`/` 与 `@`）

两版 UI 都有，命令表**同源** `web/src/slashCommands.js`（改一处两版都变）：

| 命令 | 行为 |
|------|------|
| `/tools` | 打开工具选择器（列各工具服务发现的工具），选一个 → 填 `/tools <名称>` |
| `/ls` | 打开**目录浏览面板**（可逐级下钻、★ 收藏）→ 选目录 → 填 `/ls <路径>` |
| `/rez` | 同上，但从 **rez 包名**起（包 → 版本 → 子目录）→ 填 `/ls <路径>` |
| `/read` `/ws` `/workspace` | 直接把命令名填进输入框，参数自己接着打 |

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
|------|------|
| `GET /api/tool-services` | 列出服务（含本机自动发现的 `builtin` 服务） |
| `POST /api/tool-services` | 创建服务 |
| `POST /api/tool-services/activate` | 激活服务 |
| `POST /api/tool-services/test` | 测试连通（`/tools` 不可用时自动用 `/execute` 发现） |
| `POST /api/tool-services/invoke` | 调用远端服务的一个工具 |
| `POST/DELETE /api/tool-services/<sid>` | 更新/删除（内置服务不支持删除） |

### 其他端点

| 端点 | 说明 |
|------|------|
| `GET /` | 聊天网页 |
| `GET /health` | 健康检查 |
| `GET /api/models` | 模型列表 |
| `GET/POST /api/settings` | 读取/更新运行配置 |
| `GET /api/workspace` | 工作区信息（含 `workspace_file`：agent 自己那份 `.code-workspace`） |
| `POST /api/workspace` | 设置工作区 |
| `GET/POST /api/workspace/root` | 工作区根目录管理 |
| `POST /api/workspace/root/activate` | 激活根目录（= 把该项移到 `folders` 首位） |
| `POST /api/ask-answer` | 回答 `ask_user` 的提问（SSE `ask_required` 之后调用；`{ask_id, answer}`） |
| `GET /api/browse` | 浏览目录 |
| `GET /api/browse_rez` | 多级浏览 rez 包仓库 |
| `POST /api/translate` | AI / 免费翻译 |
| `POST /api/prompt/optimize` | 提示词优化（草稿 → 更清晰的提示词；独立小模型，见 `prompt_optimizer.py`） |
| `GET /api/sandbox/status` | 终端沙盒能力探测（仅 Windows，AppContainer） |
| `GET/POST/DELETE /api/permission-rules` | 权限规则（saved 层）列表 / 追加 / 删除 |

## 回归测试

```bat
wuwor l_agent_chat -- l_agent_chat_test              rem 跑 tests/ 全部用例，退出码非 0 即失败
wuwor l_agent_chat -- python tests/run_all.py -v      rem 逐条看用例名
```

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
- **hooks**：matcher 语义、可阻断集合、决策合并（deny 只在可阻断生效 / ask 不覆盖 deny /
  `hookEventName` 不符丢弃）、**真起子进程**走 stdin-stdout 的端到端、Claude Code 四层设置来源
- **planner 工具循环**：请求形状、工具往返、HTTP 错误 fail-open
- **端点级**（真起 uvicorn + 假 provider）：SSE 流式回复、工具调用往返、hook deny、
  审批**放行 / 拒绝 / 超时**、`.codemakerignore` 拦读、工具失败也必须发事件
- **权限模式**：三档合成（`deny` 永远拦 / `allow_all` 全放行 / `autopilot` 保受保护路径的
  `ask`）、`normalize_mode` 回退（`test_permissions.py`）
- **对话模式**：受限模式裁掉写 / 执行工具、与权限模式正交（`test_chat_modes.py`）
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
|--------|----------|------|------|
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

## 依赖该包的包

| 包名 | 用途 |
|------|------|
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
