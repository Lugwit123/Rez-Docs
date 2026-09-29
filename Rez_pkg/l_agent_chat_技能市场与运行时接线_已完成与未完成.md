# l_agent_chat 技能市场 / 技能运行时：已完成与未完成（2026-09-29）

> 状态口径：**已完成** = 有代码 + 有测试 + 本地跑过；**未完成** = 只有锚点与方案，没有代码。
> 行号截至 **2026-09-29**，定位以**函数名**为准（行号会漂移）。
> 相关：市场部分见 `Rez_pkg/l_agent_chat_交接_已修与待办.md`、包内 `l_agent_market/999.0/README.md`。

---

## 0. 一览

| 块 | 内容 | 状态 |
|---|---|---|
| A | `l_agent_market` 包（技能/插件市场：源解析、清单归一、装/卸/已装） | ✅ 已完成（79 用例绿） |
| B | `l_agent_chat` 接线：requires / 设置项 / 3 个端点 / 设置页两个面板 | ✅ 已完成（端点用例绿） |
| C | **技能进 system 提示**（索引注入） | ✅ 已完成（§3.1，2026-09-29） |
| D | **`skill` 工具**（模型按需拉正文） | ✅ 已完成（§3.2，2026-09-29） |
| E | **`/技能名` 斜杠命令**（前端弹窗 + 正文展开） | ✅ 已完成（§3.3，2026-09-29） |
| F | 拍板项（注入策略 / 展开位置 / 扫描范围） | ✅ 已拍板（§4） |

一句话：市场把技能装进 `~/.lugwit/l_agent_chat/skills/`，运行时把"装好的技能"接上电 —— 智能体 / `/技能名` 决定**本会话有哪些技能可用**，`skill` 工具按需拉正文，索引只给名字 + 一行描述。

---

## 1. 已完成 A：`l_agent_market` 包（纯库，不起服务）

包根 `rez-package-source/l_agent_market/999.0/`，`requires = ["python-3.12+<3.13", "requests"]`。

| 文件 | 职责（关键函数） |
|---|---|
| `src/l_agent_market/paths.py` | 安装根。`home()` / `skills_dir()` / `plugins_dir()` / `kind_dir()` / `cache_file()`；根默认 `~/.lugwit/l_agent_chat/`，`L_AGENT_MARKET_HOME` 可整体替换 |
| `src/l_agent_market/sources.py` | 源读取层。`open_source()`（实例缓存）、`parse()` 规则、`join_rel()` / `_rel()` 路径归一；三个实现 `GhSource`（api.github.com tree+blob）、`HttpSource`（只读清单）、`FileSource`（本地目录/文件）。`Source.reset()` 丢目录缓存 |
| `src/l_agent_market/market.py` | 清单归一。`parse_frontmatter()`（SKILL.md 头部）、`list_items()`、`find_item(key)`、`refresh()`、`parse_sources()`；条目 key = `<源>|<kind>|<名字>` |
| `src/l_agent_market/store.py` | 装/卸。`install(item, force)`、`uninstall(kind, name, force)`、`installed(kind)`、`installed_names(kind)`、`safe_name()`；写 `.market.json`；`.tmp` 再改名；上限 `MAX_FILES=500` / `MAX_TOTAL_BYTES=50MB` / 单文件 5MB |
| `src/l_agent_market/cli.py` | `list / installed / install / uninstall / refresh`（alias `l_agent_market`） |
| `tests/` | `test_market.py` / `test_install.py` / `test_gh_source.py` / `fixtures.py` / `run_all.py`——**64 用例，不打网络** |

模块名注意：本地安装库叫 `store.py` **不能叫 `install.py`** —— 包 `__init__` 导出的 `install()` 函数会覆盖同名子模块，包内 `from . import install` 拿到的是函数而不是模块（踩过）。

**源写法**（设置项 `market_sources`，分号或逗号分隔，留空 = `gh:anthropics/skills`）：

| 写法 | 例 | 能力 |
|---|---|---|
| `gh:<owner>/<repo>[@<ref>][#<subdir>]` | `gh:anthropics/skills` | 浏览 + 安装。走 `api.github.com`（本机 `raw.githubusercontent.com` 不可达）；未鉴权 60 次/小时，`L_AGENT_MARKET_GITHUB_TOKEN` 提到 5000 |
| `http(s)://…/marketplace.json` | | 只浏览（无目录列举/下载能力，`install` 明确拒绝） |
| 本地路径 / `file://D:/market` | | 浏览 + 安装（离线、内网自托管） |

**安装落点**（本产品自己的目录，不碰 CodeMaker / Claude Code）：

- 技能 `~/.lugwit/l_agent_chat/skills/<技能名>/`
- 插件 `~/.lugwit/l_agent_chat/plugins/<插件名>/`
- 每个安装目录内 `.market.json`：`{key, kind, name, plugin, source, path, files, installed_at}`
- 卸载只删带清单的目录（手放的目录先拒绝，`force=True` 才删）

清单缓存 600 秒（内存 + `<安装根>/market_cache.json`），`?force=1` / `refresh()` 连源内目录树一起重拉。

---

## 2. 已完成 B：`l_agent_chat` 侧接线

| 改动 | 位置 |
|---|---|
| 依赖 | `package.py` 的 `requires` 加 `l_agent_market` |
| 设置项 | `config.py`：`MARKET_SOURCES = _v("market_sources", "AGENT_CHAT_MARKET_SOURCES", "gh:anthropics/skills")` + `_SCHEMA["market_sources"]`；设置页 `FIELDS` 加 `market_sources` |
| 端点 | `app.py`：`market_list()` → `GET /api/market`（`?kind=skill\|plugin`、`?force=1`；返回 `sources/notes/items/skills/plugins/installed`）、`market_install()` → `POST /api/market/install`（按 key 重新解析条目，不信前端回传路径）、`market_uninstall()` → `DELETE /api/market/install?kind=&name=&force=` |
| 设置页 | `templates/settings.html`：两个卡片「🧩 技能市场」「🧩 插件市场」（源输入框 + 已安装列表 + 市场列表 + 刷新远程 + 装/卸）；JS 侧 `MARKET_KINDS` / `loadMarket()` / `renderMarketList()` / `renderMarketInstalled()` |
| 测试 | `tests/test_agent_endpoint.py::MarketApiTest` 3 用例（本地市场夹具 + `env_extra` 指 `AGENT_CHAT_MARKET_SOURCES` 与 `L_AGENT_MARKET_HOME`；含"设置页含市场面板"断言） |

**验证记录（2026-09-29，A/B 轮）**

| 验证 | 结果 |
|---|---|
| `wuwor l_agent_market -- l_agent_market_test` | 64 项 OK（C/D/E 后扩到 79，见 §9） |
| `MarketApiTest`（真起 uvicorn，3 项） | OK |
| `settings.html` 内联 JS 语法 | `node --check`（vm.Script 解析每段 `<script>`）通过 |
| `l_agent_chat` 默认批（713 项） | 4 失败 + 3 错误，**基线复跑同样失败**（`code_mode` / `terminal` / `workspace_file`，本机 temp/子进程环境问题） |
| `l_agent_chat` 端点批（82 项） | 5 失败，**基线复跑同样失败**（`ApprovalTest` / `EndpointTest` / `ManualCompactionTest` ×2 / `RecorderReplayTest`，与市场无关）；`MarketApiTest` 3 项全过 |
| `wuwo doc_pkg` | 已刷新自动块；顺带刷新了 5 个本就有漂移的包（`conemu` / `l_agent_tool` / `l_model_hub` / `l_notepad_server` / `l_tray`） |
| 真机拉公网市场 | 未鉴权配额已耗尽 → 优雅降级成一行 note（源标 ⚠），不影响内置/本地源 |

---

## 3. 已完成：运行时接线（C / D / E）

**已完成（2026-09-29）。** 三块互不依赖，按 §6 顺序 1~4 分别落地；下面的"方案"段保留为设计依据，每节末尾的 **已实施** 段是真实落点与验收。

### 3.1 C：技能进 system 提示（索引注入）

**锚点（现状）**

| 事实 | 位置 |
|---|---|
| 直接回答路径的 system 拼装 | `agent_client._system_parts(turns, mode, history_summary)`，返回有序 dict：`base → history → vscode → preference → rules → specs → memory → mode`；由 `_build_messages()` 合并成单条 `role=system` |
| 工具循环（planner）路径的 system | `_planner_messages()`：常量 `_PLANNER_SYSTEM` 打底，再按 `rules → specs → … → mode` 依次追加 |
| 现成先例（抄它） | `rules.prompt_block(workspace)`（三层 scope：`always` 全文 / `index` 名+描述+路径 / `skip`）、`specs.prompt_block(workspace)`（**只给索引**，末尾一句"要正文自己用文件工具读"）；两者都有 `config.*_ENABLED` 开关、`*_MAX_CHARS` 上限、`*_TTL` 秒级缓存与 `clear_cache()` |
| 记忆块的开关形态 | `memory.prompt_block(query)` + `agent_client._memory_block()` 里再判 `MEMORY_ENABLED` / `MEMORY_INJECT` |

**方案（建议）**

1. `l_agent_market` 增 `skills.py`：`discover(include_plugins=True)` → `[{name, description, path, root, kind("standalone"\|"plugin"), plugin, source}]`；`read(name)` → `(ok, text, path, err)`（带上限）。扫描范围（默认）：`skills_dir()/*/SKILL.md` + `plugins_dir()/*/**/SKILL.md`（跳过 `.git`，深度限 3，单个 `SKILL.md` ≤ 上限）。名字取 frontmatter `name`，退目录名；重名规则：**独立安装优先于插件内**，同层先到先得 + note。
2. `l_agent_chat` 增 `skills.py`（胶水，对齐 `mcp.py` 的定位）：`prompt_block(workspace)` 产出索引文本（受 `SKILLS_MAX_CHARS` 限制、`SKILLS_TTL` 缓存）+ `skills_enabled` 开关；**两条 system 路径都要挂**（漏 planner 路径等于常态没索引）。
3. `config.py`：`SKILLS_ENABLED` / `SKILLS_MAX_CHARS` / `SKILLS_BODY_MAX_CHARS` / `SKILLS_TTL`（+ 可选 `SKILLS_EXTRA_ROOTS`，默认空），三处登记见 §5 的模式。

**索引文本草案**

```markdown
## 可用技能（SKILL.md；要正文时调用 skill 工具，参数 name）
- alpha：干活的（standalone）
- xlsx：做表格（插件 document-skills）
提示：技能正文按需加载，不要凭空猜测其内容；用户消息以 `/技能名` 开头时正文已展开。
```

**验收判据**

- 装 1 个技能 → 新会话 system 里出现该技能的"名字 + 描述"；一条都没装时**不出现空标题**；
- `SKILLS_ENABLED=0` → 不出现；技能很多时被 `SKILLS_MAX_CHARS` 截断（有"已省略 N 个"提示）；
- 单测（新 `tests/test_skills.py`，默认批自动发现）：索引文本形状、上限截断、重名规则、TTL 缓存命中、插件内技能可被发现；
- **两条路径都要有用例**（直接回答路径 + planner 路径各一），否则容易只挂一半。

**已实施（2026-09-29）**

| 落点 | 说明 |
|---|---|
| `l_agent_market/src/l_agent_market/skills.py`（新） | `discover(force=False)` / `find(name)` / `read(name, max_chars)` / `scripts(item)` / `describe(item)` / `clear_cache()`；扫 `skills_dir()/*/SKILL.md` + `plugins_dir()/*/**/SKILL.md`（深度 3、跳 `.git`/隐藏目录、单文件 ≤ 512KB），独立安装优先于插件内，TTL **5s** |
| `l_agent_chat/src/l_agent_chat/skills.py`（新，胶水） | `prompt_block()`（索引，受 `SKILLS_MAX_CHARS` 截断 + 「已省略 N 个」）、`agent_block()`（智能体额外提示）、`system_block()`（两者拼接）、`clear_cache()`；缓存键 = `(智能体, 激活名字元组)`，TTL 取 `SKILLS_TTL` |
| 直接回答路径 | `agent_client._system_parts` 的返回 dict 新增 `"skills"` 键（在 `specs` 之后、`memory` 之前）：`"skills": f"\n\n{skills_text}" if skills_text else ""` |
| planner 路径 | `agent_client._planner_messages` 在 `specs` 之后追加 `skills.system_block()`（R1：两条都挂） |
| 设置项 | `config.py`：`SKILLS_ENABLED` / `SKILLS_MAX_CHARS`(4000) / `SKILLS_BODY_MAX_CHARS`(12000) / `SKILLS_TTL`(5) + `_SCHEMA` 四条 + `settings.html` 的 `FIELDS`/控件 |
| 上下文面板 | `agent_client._BLOCK_MAP` 把 `skills` 并进 `rules` 块（多对一，与 `specs` 一致） |
| 语义 | **只列「本会话激活 ∪ 当前智能体预设」**；默认智能体预设为空 → 只出一句「已装 N 个技能，用 `/技能名` 添加」（不列名字，对齐 §4 Q1 拍板） |

**验收**：`tests/test_skills.py` 的 `TestPromptBlock`（关闭/无安装/默认智能体只给提示/预设列名字/激活技能进索引/预设未装跳过/超限省略）+ `TestAgentBlock` + `TestSystemInjection`（**直接路径 + planner 路径各一条**，直接路径另有一条"无可选集时不出块"）；`test_compress_retry.py` 的 `_system_parts` key 序契约同步加 `"skills"`。

### 3.2 D：`skill` 工具（模型按需拉正文）

**锚点（现状）**

| 事实 | 位置 |
|---|---|
| 工具注册表 | `l_agent_tool.agent_tools.ToolRegistry` / 全局 `DEFAULT_TOOLS`；`AgentTool(name, func, description, parameters)`；装饰器 `register_default_tool(...)` |
| "导入即注册"的先例 | `rez_knowledge.py` 顶部 `from l_agent_tool import DEFAULT_TOOLS, register_default_tool`，9 个 `@register_default_tool(...)`；`agent_client` 顶部有一段副作用 import（`rez_knowledge` / `subagents` / `code_mode` / `procs` / `ask_user`）——**新模块必须加进这段，否则工具不进清单** |
| 合并进模型清单 | `agent_client._local_tools_schema()` 迭代 `DEFAULT_TOOLS.list()`；`_available_tools()` 再拼远程服务与 MCP；`tool_router.select_tools()` 按 `CORE_NAMES` / `HINTS` 做相关性筛选 |
| 分发 | `app.py` planner 循环按 `entry["q"]` 前缀分派：`local:` → `agent_client._local_call()` → `DEFAULT_TOOLS.get(name)` |
| 审批 | `permissions.evaluate(tool, args, qualified, ws_root)`（`DEFAULT_RULES` + `config.PERMISSION_RULES` + 工作区 `permission_rules.json`）；写操作名单 `config.APPROVAL_TOOLS`。**只读工具不在名单里 → 无匹配规则时 `effect=none` → `AUTO_APPROVE=1`（默认）下自动放行** |

**方案（建议）**

- 工具名 `skill`，参数 `name`（string，必填）；返回：`SKILL.md` 正文 + 来源路径 +（若有）`scripts/` 文件名列表——**只读，不执行任何脚本**。
- 正文超 `SKILLS_BODY_MAX_CHARS` 截断，并明确告诉模型"已截断，可用 read_file 读全文（路径给出）"。
- 未知名字：返回错误 + **可用候选列表**（避免模型瞎猜）。
- 注册落点：`l_agent_chat/src/l_agent_chat/skills.py`（`DEFAULT_TOOLS.has("skill")` 守卫 + `register`，与 `agent_tools._register_edit_tools()` 同风格），并在 `agent_client` 的副作用 import 段加一行。
- 相关性：`tool_router.CORE_NAMES` **不要**把 `skill` 当核心工具（避免每轮都占清单），在 `HINTS` 里加"技能/技能名/SKILL"之类触发词即可（具体见 `tool_router.select_tools`）。
- 审批：保持不在 `APPROVAL_TOOLS`（读的是本机已装技能，无副作用）。要更严的话加默认规则 `skill:* → allow`（显式 allow，便于用户改成 ask）。

**验收判据**

- 让模型对着一个已装技能发起调用 → 工具返回正文头部 + 路径；`tool_start/tool_result` 事件成对；
- 未知名字 → 返回候选列表、不抛异常、整轮不中断；
- 超长正文 → 截断 + 提示路径；
- 不弹审批（default + `AUTO_APPROVE=1`）；把 `skill` 加进 `APPROVAL_TOOLS` 后才弹（负向用例）；
- 单测：`DEFAULT_TOOLS.has("skill")`、未知名、截断、路径缺失（技能被卸载后调用）。

**已实施（2026-09-29）**

| 落点 | 说明 |
|---|---|
| 注册 | `l_agent_chat/src/l_agent_chat/skills.py` 底部 `if not DEFAULT_TOOLS.has("skill"): register_default_tool("skill", skill_tool, ...)`，参数 `name`（string，必填） |
| 副作用 import | `agent_client.py` 顶部 `from . import skills  # 导入即注册 skill 工具`（漏了就不进清单） |
| 返回 | `{ok, name, description, source, path, content}` + 有脚本时 `scripts`（**只列文件名，绝不执行**）+ `scripts_note`；超 `SKILLS_BODY_MAX_CHARS` 加 `truncated: true` 与 `note`（给 `read_file` 路径） |
| 失败路径 | 缺参数 / 未装 / **装了但不在本会话可选集** 三种都返回 `ok:false` + 候选清单（`available` / `installed` / `hint`），不抛异常 |
| 相关性 | `tool_router.HINTS` 加 `("skills", ("技能", "skill", "skill.md", "技能市场"))`；`CORE_NAMES` **不含** `skill`（R6） |
| 审批 | 不在 `APPROVAL_TOOLS`（只读本机已装技能，无副作用） |

**验收**：`tests/test_skills.py` 的 `TestToolRegistered`（含 `test_skill_tool_is_not_gated_by_approval`）、`TestRouterHint`（是 hint 不是 core、中英文触发词都命中）、`TestSkillTool`（缺参数 / 未装 / 已装未激活 / 激活返回正文 + scripts / 截断打标）；**端到端**（真起 uvicorn + 假模型）`tests/test_agent_endpoint.py::SkillsRuntimeTest` 的 `test_skill_tool_roundtrip`（`tool_start`/`tool_result` 成对、正文进结果）、`test_skill_tool_unknown_name_does_not_break_round`（未知名 → 返回候选、整轮不中断）、`test_skill_is_not_gated_by_approval`（关掉 `auto_approve` 也不弹审批）。

### 3.3 E：`/技能名` 斜杠命令

**锚点（现状）**

| 事实 | 位置 |
|---|---|
| 命令表是**前端硬编码** | `web/src/slashCommands.js`：`export const SLASH_CMDS`，6 条（`/tools` `/ls` `/rez` `/read` `/ws` `/workspace`），形状 `{name, desc, arg, browse?, tools?}`；**没有任何端点提供命令列表** |
| 弹窗与选中行为 | `web/src/new/ComposerTriggers.jsx`：`unstable_useSlashCommandAdapter({commands: SLASH_CMDS.map(...)})`；选中后 `slashFormatter.serialize` 只把**纯文本**（如 `/tools name`）写回输入框（`browse` / `tools` 两类命令打开面板、不写字）；**不自动发送、不调后端** |
| 命令语义 | 由服务端 agent 解释（如 `/ls` → 工具）；不存在 `/compact` `/clear` `/help` |
| 前端产物 | 改 `web/src/**` 后要 `l_agent_chat_web_build`（`npm run build`）重建 `src/l_agent_chat/static/dist`，包内提交；浏览器还需强刷（dist 防缓存中间件曾按前缀匹配漏掉 `/chat/static/dist/…`，已修） |

**两个方案（§4 已拍板：选 E1 服务端展开）**

- **E1 服务端展开（推荐）**：`/名字 剩余参数` 在服务端（`app.py` 建 turns 之前）匹配"已发现技能名"→ 展开成"技能正文 + 剩余参数"，其余消息原样。优点：所有 UI/API 客户端（含经典页 / VS Code 扩展 / 脚本调用）一次生效，权限与开关都在服务端统一。缺点：需要新增"消息预处理"这一步。
- **E2 前端展开**：弹窗里选中技能 → 前端 `GET /api/skills/<name>` 拉正文 → 拼进输入框再发送。优点：不动后端消息路径、用户能看见实际发送的内容。缺点：正文进输入框（很长、体验差），且经典页 / API 客户端拿不到。

**配套（两方案都要）**

- 后端加 `GET /api/skills`：`{items: [{name, description, source, plugin, path, bytes}], enabled, roots}`——供前端弹窗动态合并命令（`SLASH_CMDS` 静态项 + 动态技能项，命名前缀 `/`）。
- 前端把 `SLASH_CMDS` 从常量改成"静态 + 异步拉取"，`ComposerTriggers.jsx` 的 adapter 支持后到列表（assistant-ui 的 `commands` 变更要能触发重渲染，需实测）。
- `skills_enabled=0` 时端点返回空列表（弹窗里不出现技能项）。
- **不要在技能名上做前缀猜测**（如 `/s-`），直接用技能名；与现有 6 条命令重名时以静态命令优先。

**验收判据**

- `/` 弹窗里能看到已装技能（含描述），选中 → 发送后服务端确实把正文纳入该轮（用 planner trace / 假模型收到的 messages 断言）；
- 与 `/ls` 等既有命令并存不冲突；未装技能时弹窗只有 6 条静态命令；
- `npm run build` 后强刷可见；`GET /api/skills` 有端点用例。

**已实施（2026-09-29，E1 服务端展开）**

| 落点 | 说明 |
|---|---|
| 端点 | `app.py`：`GET /api/skills`（`{items, enabled, reserved}`，`skills_enabled=0` 时 `items=[]`；`reserved` = 静态命令名，重名以静态优先）；`GET/POST/DELETE /api/agents`（智能体增删改，写盘走 `agents.py`，`default` 不可删） |
| 服务端展开 | `app.py` 建 turns 前 `skills.set_context(agent, session_skills, sid)`；随后对**最后一条 user turn** 调 `skills.expand()`：`/名字 其它` → 正文 + 其它（并入本会话激活集），`/名字 -` → 移除；保留命令 / 未装名字**原样不动**。**只改 `turns`，不动落盘的 `messages`**（否则每轮重发正文烧 token） |
| 会话激活集 | `session_store` 的会话 meta 存 `skills` 列表（`get_skills` / `set_skills`；变更只 bump rev、不碰 messages） |
| 会话级语义 | `skill` 工具**只在可选集里找**（= 会话激活 ∪ 智能体预设），未激活返回"怎么加"而非硬失败 |
| 前端 | `web/src/slashCommands.js`（**只有**静态 6 条命令表）、`web/src/new/ComposerTriggers.jsx`（拉 `GET /api/skills` → `skillCmds`，与静态表合并成 `slashCommands` 交给 `unstable_useSlashCommandAdapter`；`commands` 后到列表触发重渲染；保留名/同名技能被过滤，静态优先）、`web/src/new/NewApp.jsx`（「智能体」下拉，与「模式」下拉并排）、`web/src/new/runtime.js`（`readAgent`/`writeAgent`）、`web/src/api.js`（`getSkills`/`getAgents`）、`settings.html`「🧠 智能体与技能接线」面板；改完 `npm run build` 重建 `static/dist` |
| 缓存失效 | 装 / 卸 / 改智能体后统一调 `skills.clear_cache()`（不等 5s TTL） |

**验收**：`tests/test_skills.py` 的 `TestExpand`（普通消息/保留命令/未装名字不动；`/名字` 加正文；`-` 移除；`-` 保留尾随文本；前导空白仍匹配；截断提示）+ `TestSessionSkills`（往返/去重/trim，空列表清键，缺文件，改激活集只 bump rev 不碰 messages）+ 端点用例 `SkillsApiTest`（`/api/agents` CRUD、`/api/skills` 列表、`skills_enabled=0` 返回空）+ **端到端** `tests/test_agent_endpoint.py::SkillsRuntimeTest.test_slash_skill_expands_into_this_turn_only`（`/alpha 帮我看看` → 假模型收到的请求里含展开的正文；落盘的**消息 content** 仍为原始 `/alpha 帮我看看`（不随历史重发），激活名 `alpha` 随会话落盘）。

---

## 4. 已拍板（3 问）

| # | 问题 | 结论（2026-09-29 拍板） |
|---|---|---|
| Q1 | 技能怎么进 system | **① 索引注入 + `skill` 工具**（对齐 `specs.py`）。且**默认智能体不自动注入索引** —— 只给一句「已装 N 个，用 `/技能名` 添加」；要"开箱带技能"就切到预设了技能的智能体 |
| Q2 | `/技能名` 在哪展开 | **① 服务端**（所有客户端统一，含 API / VS Code 扩展；权限与开关都在服务端） |
| Q3 | 扫描范围 | **① 只扫 `<安装根>/skills/` + `<安装根>/plugins/*`**（与"只归 l_agent_chat"一致；`SKILLS_EXTRA_ROOTS` 未实现，留作后续） |

补充拍板（实现前确认）：**智能体来源 = 独立 agents 配置**（`~/.lugwit/l_agent_chat/agents/<名>.json`，内置 `default`，设置页管理，输入框「模式」下拉旁加「智能体」下拉）；**`/技能名` 语义 = 会话级添加**（`skill` 工具也只在可选集里找，`/技能名 -` 移除）。

---

## 5. 新增设置项的三处登记（别漏）

以 `specs_enabled` 为例的固定套路：

| 处 | 写法 |
|---|---|
| `config.py` 模块常量 | `SPECS_ENABLED = _bool_setting("specs_enabled", "", "1")`（int/float 用 `_v("键", "环境变量名", 默认值, int)`） |
| `config.py` 的 `_SCHEMA` | `"specs_enabled": ("SPECS_ENABLED", lambda v: str(v).strip().lower() in ("1","true","yes","on"))` |
| `templates/settings.html` 的 `FIELDS` | 数组里加键名（控件 `id` 必须等于键名；数组型还要进 `ARRAY_FIELDS`） |

---

## 6. 建议实施顺序

1. **`l_agent_market.skills`**（发现 + 读正文）＋ 包内单测 —— 不依赖 chat 侧，先跑通。
2. **C：system 索引注入**（`l_agent_chat/skills.py` + 两条 system 路径 + 4 个设置项 + 单测）—— 先有"模型知道有哪些技能"。
3. **D：`skill` 工具**（注册 + 副作用 import + `tool_router` 提示词 + 单测 + 端点/循环用例）。
4. **E：斜杠命令**（`GET /api/skills` + 前端动态合并 + `npm run build` + 端点用例）。

每步跑：`wuwor l_agent_chat -- l_agent_chat_test`（改了 `app.py` 路由再加 `--endpoint`）；跨包改动补 `wuwor l_agent_market -- l_agent_market_test`。服务在跑时用 `wuwo svc reload l_agent_chat` 生效。

---

## 7. 已知风险与坑

| # | 风险 | 说明 / 对策 |
|---|---|---|
| R1 | **两条 system 路径只挂一条** | 工具循环（planner）是常态路径；只改 `_system_parts` 会出现"有时有索引、有时没有"。两条都要加 + 各一条用例 |
| R2 | 上下文膨胀 | 索引也会涨（`specs.py` 同样问题）：只给名字 + 一行描述、给上限、超限写"已省略 N 个" |
| R3 | 重名 | 独立安装 vs 插件内 vs 前端命令三条命名空间要写死优先级（建议：静态命令 > 独立技能 > 插件内技能，同层先到先得） |
| R4 | 扫描 TTL 与市场缓存 TTL 不同 | 市场清单缓存 600s、技能索引扫描建议 5s（对齐 `rules`/`specs`），否则"装完看不见" |
| R5 | 前端 dist 缓存 | 改 `web/src/**` 必须重建 dist 并强刷；别把"没生效"误判成后端问题 |
| R6 | `skill` 工具污染工具清单 | 别放进 `tool_router.CORE_NAMES`；只在 `HINTS` 里给触发词 |
| R7 | 公网技能正文 = 第三方指令 | 见下方安全提示 |

**安全提示（R7，务必看）**：技能正文是**来自市场的第三方文本**，"装第三方技能"等价于"信任它写进模型上下文的一切"，正文里可以夹带指令（提示注入）。因此：

- 索引里**标出来源**（独立 / 插件名 / 源仓库），让用户与模型都能看到这是外来内容；
- SKILL.md 附带的 `scripts/` **只在工具返回里列文件名，绝不自动执行**；
- 需要的话给 `SKILLS_TRUSTED_ONLY`（仅装本机白名单源）之类开关，默认关；

## 8. 锚点速查（相对本节正文，行号截至 2026-09-29）

| 用途 | 锚点 |
|---|---|
| 直接路径 system | `agent_client._system_parts` / `_build_messages` |
| planner 路径 system | `agent_client._planner_messages`（`_PLANNER_SYSTEM` 打底） |
| 注入先例 | `rules.prompt_block` / `specs.prompt_block` / `memory.prompt_block` / `chat_modes.prompt_block` |
| 工具注册 | `l_agent_tool.agent_tools.ToolRegistry` / `DEFAULT_TOOLS` / `register_default_tool`；`rez_knowledge.py` 的副作用注册 + `agent_client` 顶部 import 段 |
| 工具清单/筛选 | `agent_client._local_tools_schema` / `_available_tools` / `available_tools_cached`；`tool_router.select_tools`（`CORE_NAMES` / `HINTS`） |
| 工具分发 | `app.py` 按 `entry["q"]` 前缀分派 → `agent_client._local_call` |
| 审批 | `permissions.evaluate` / `active_rules` / `effective`；`config.APPROVAL_TOOLS`；`app.py` 的 `needs_approval` 段 |
| 斜杠命令 | `web/src/slashCommands.js`（`SLASH_CMDS`）；`web/src/new/ComposerTriggers.jsx`（adapter + `slashFormatter`） |
| 市场侧 | `l_agent_market.paths.skills_dir/plugins_dir`、`store.MANIFEST`、`market.parse_frontmatter` |
| 运行时侧（新） | `l_agent_market.skills.discover/find/read/scripts/describe`；`l_agent_chat.skills.prompt_block/agent_block/system_block/expand/skill_tool/list_items/set_context`；`l_agent_chat.agents.list_agents/get_agent/save_agent/delete_agent` |

---

## 9. 验证记录（C/D/E 通电，2026-09-29）

| 验证 | 命令 | 结果 |
|---|---|---|
| 市场包单测（含新 `test_skills.py`） | `wuwor l_agent_market -- l_agent_market_test` | **79 用例 OK**（退出码 0） |
| 技能单测（含新增免审批用例） | `wuwor l_agent_chat -- python -m unittest tests.test_skills -v` → `s12_skills_unit_out.txt` | **33 用例 OK**（含 `TestToolRegistered.test_skill_tool_is_not_gated_by_approval`） |
| chat 默认批（`--quick`，跳过 proc / endpoint） | `wuwor l_agent_chat -- python tests/run_all.py --quick` → `s12_chat_quick_out.txt` | **757 用例：失败 4 / 错误 3**，与基线一致（`test_p2_batch.TestCodeMode` 3E；`TestTerminalPolicy.test_ask_rule_rejected_no_channel` 1F；`test_workspace_file` 3F，均为本机 temp/子进程环境问题）。**本次改动零新增回归** |
| 回归修复 | `wuwor l_agent_chat -- python -m unittest tests.test_compress_retry` | 26 OK（`_system_parts` key 序契约补 `"skills"`） |
| 端点用例（真起 uvicorn） | `SkillsApiTest`（`test_agents_crud` / `test_list_skills` / `test_list_skills_disabled`） | **3 OK** |
| **C/D/E 端到端通电**（真起 uvicorn + 假模型） | `wuwor l_agent_chat -- python -m unittest tests.test_agent_endpoint.SkillsRuntimeTest -v` → `s12_skills_runtime_out.txt` | **4 用例 OK**（§3.3 `/技能名` 展开进本轮且不落历史；§3.2 `skill` 工具往返 + 未知名不中断 + 免审批） |
| **端点整批回归复验**（真起 uvicorn） | `wuwor l_agent_chat -- python tests/run_all.py --endpoint` → `s12_chat_endpoint_out.txt` | **89 用例：失败 6**，零新增回归。5 条为 §2 记录在案的基线（`ApprovalTest.test_edit_preview_failure_emits_result` / `EndpointTest.test_plain_reply_streams_and_done` / `ManualCompactionTest` ×2 / `RecorderReplayTest.test_record_then_replay_without_llm`）；第 6 条 `McpApiTest.test_market_install_list_delete` 为**环境性超时**（`GET /api/mcp/market` 要外网拉 `registry.modelcontextprotocol.io`，整批并发下 15s 读超时；单跑 `McpApiTest` 类 8 用例全过，与技能接线无关） |
| 陈旧判据修复（随本次端点批复验发现） | `wuwor l_agent_chat -- python -m unittest tests.test_agent_endpoint.PartModelTest -v` → `s12_iso_two.txt` | `test_new_file_write_has_no_patch` 断言的是**功能落地前**的旧契约（新建文件不出 patch）。该契约已被「工具卡显示 VS Code 风格 diff」功能推翻（`app.py`「整文件写入的 diff」：`write_file` 写盘前一律补 diff 预览，`_diff_preview` 语义即「新文件标记为全部新增」）——**改的是用例而非实现**：重命名为 `test_new_file_write_emits_all_add_patch`，断言改为「产出 patch 且 `adds≥1 / dels=0`」。修后 `PartModelTest` + `McpApiTest` 合跑 **8 用例 OK** |
| G6 全量直传（四包对齐远端） | `wuwor l_agent_chat -- python d:\Temp\g6_push_align.py` → `g6_push_align_console.txt`（17:01，最新）/ `g6_push_align_out.txt`（16:48，前一轮） | 退出码 0；**最新**（`g6_push_align_console.txt`）`written=1 skipped=265 errors=0`（唯一改动的 `tests/test_agent_endpoint.py`）；前一轮（`g6_push_align_out.txt`）为 `written=2`：`tests/test_agent_endpoint.py` + `tests/test_skills.py`；C/D/E 落地轮为 `written=23` |
| 远端重启（红线⑤：`.py` 不热重载） | `wuwor l_agent_chat -- python d:\Temp\g6_remote_restart.py` → `g6_remote_restart_out.txt` | `RESTART_RC 0`；`HTTP 1250 200` / `HTTP 8765 200` |
| G6 终检 | `wuwor l_agent_chat -- python d:\Temp\g6_final_check.py` → `g6_final_console.txt`（17:01，最新）/ `g6_final_out.txt`（16:49） | `SVC_OK True`；`l_agent_tool 20/20`、`l_agent_chat 155/155`、`l_notepad_server 76/76`、`l_agent_market 15/15` 全部 `bad=0 missing=0`；远端 `HTTP 1250 200` / `HTTP 8765 200`；末行 **`汇总 = G6 PASS`** |
